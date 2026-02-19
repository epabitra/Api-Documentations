# Plan: Fix Duplicate Custom Medications (Production)

## 1. Add Medication Flow Summary

### Endpoint
- **POST /api/medications/add** → `MedicationResource.addMedication()` → `MedicationServiceImpl.addMedication()`

### Flow
1. **Resolve or create medication** (`resolveOrCreateMedication`)
   - If `medicationUuid` is provided → fetch existing medication by UUID (application or custom).
   - If `medicationUuid` is **not** provided → **create new medication** with `source = 'CUSTOM'`, `sourceId = currentUser.getId()`.
2. **Get or create patient_medication** (`getOrCreatePatientMedication`)
   - Finds or creates a row in `patient_medication` linking `patient_id` (user) to `medication_id`.
3. **Create schedule** (`createMedicationSchedule`)
   - Inserts into `patient_medication_schedule` (and optionally archives/soft-deletes previous schedule for that patient_medication).

### Tables Affected by Add (when medicationUuid is missing)
| Table | Effect |
|-------|--------|
| **medication** | **INSERT** – new row with `source = 'CUSTOM'` |
| **patient_medication** | INSERT or reuse – links user to medication |
| **patient_medication_schedule** | INSERT – one row per add; may soft-delete previous schedule for same patient_medication |
| **patient_medication_schedule_history** | INSERT – when replacing a schedule (archive old) |
| **reminder** | DELETE then INSERT – when schedule is replaced; `entity_id` = `patient_medication_schedule.id` |
| **patient_medication_intake** | No direct insert on add; exists for tracking intakes against schedules |

### Key columns for matching
- **medication**: `id`, `uuid`, `name`, `brand`, `strength`, `form`, `source`, `source_id`
- Application-defined medication: `source IS NULL` or `source <> 'CUSTOM'` (master data has `source` NULL).
- Custom medication: `source = 'CUSTOM'`.

---

## 2. Problem

- Client did not pass `medicationUuid` when adding medications that already exist in the app.
- Result: New **custom** medications were created with the same name/brand/strength as application medications.
- Users see duplicate names in lists and cannot distinguish application vs custom.

---

## 3. Fix Strategy

1. **Identify duplicate custom medications**
   - Custom: `medication.source = 'CUSTOM'`.
   - For each such row, find an **application** medication (`source IS NULL OR source <> 'CUSTOM'`) with the same:
     - `name`, `brand`, `strength`, `form` (null-safe comparison).
   - If a match exists → treat custom as duplicate; we will replace its usage with the application medication.

2. **Replace usage of custom medication with application medication**
   - For each `patient_medication` row that points to the custom medication:
     - **Case A – Same user already has a patient_medication for the app medication**
       - **Merge**: Move all `patient_medication_schedule` rows from the custom `patient_medication` to the existing app `patient_medication` (UPDATE `patient_medication_schedule.patient_medication_id`).
       - Update `patient_medication_schedule_history.patient_medication_id` for those schedules so history still references a valid patient_medication.
       - Delete the custom `patient_medication` row (schedules now belong to app patient_medication).
     - **Case B – Same user does not have a patient_medication for the app medication**
       - **Redirect**: UPDATE `patient_medication SET medication_id = app_medication_id` so this row now points to the app medication.

3. **Remove duplicate custom medications**
   - After all references are updated, DELETE from `medication` where `id` = custom medication id.

### Tables updated by fix (no new tables)
- **patient_medication**: `medication_id` updated (Case B), or row deleted after merge (Case A).
- **patient_medication_schedule**: `patient_medication_id` updated when merging (Case A).
- **patient_medication_schedule_history**: `patient_medication_id` updated when merging (Case A) so history still points to a valid patient_medication.
- **medication**: DELETE duplicate custom rows.

### What we do **not** change
- **patient_medication_intake**: References `patient_medication_schedule_id`; schedule IDs do not change when we only update `patient_medication_id` on schedules.
- **reminder**: Uses `entity_id` = `patient_medication_schedule.id`; schedule IDs are unchanged, so no update needed.
- Application medications and their data: read-only.

---

## 4. Safety and Rollback

- **Backup**: Take a full backup (or at least backup `medication`, `patient_medication`, `patient_medication_schedule`, `patient_medication_schedule_history`) before running.
- **Dry run**: Script includes a DRY RUN that only SELECTs and reports:
  - Pairs (custom_med_id, app_med_id) and counts of affected patient_medication / schedules.
  - No UPDATE/DELETE in dry run.
- **Execution**: Run in a transaction; commit only after verifying row counts.
- **Rollback**: If not committed, rollback. If committed, restore from backup and re-evaluate if needed.

---

## 5. Matching Rule (when to consider custom “duplicate”)

Only treat a custom medication as duplicate of an application medication when **all** of the following match (null-safe):

- `name`
- `brand`
- `strength`
- `form`

If multiple application medications match, we pick one (e.g. `MIN(id)`) consistently. This keeps behavior deterministic and avoids breaking references.

---

## 6. Execution Order (in script)

1. Create temporary mapping table: custom_med_id → app_med_id (only for custom meds that have an exact match).
2. For each custom medication in the mapping (e.g. loop or cursor in script, or batch in application):
   - For each `patient_medication` with `medication_id = custom_med_id`:
     - If exists `patient_medication` with same `patient_id` and `medication_id = app_med_id`:
       - Update all schedules: `patient_medication_schedule.patient_medication_id` → app patient_medication id.
       - Update history: `patient_medication_schedule_history.patient_medication_id` for those schedule ids → app patient_medication id.
       - Delete the current (custom) `patient_medication` row.
     - Else:
       - Update `patient_medication SET medication_id = app_med_id`.
   - Delete `medication` where `id = custom_med_id`.
3. Drop temp table, commit (or rollback).

---

## 7. Verification After Fix

- Count of medications with `source = 'CUSTOM'` should decrease by the number of merged/redirected duplicates.
- No `patient_medication.medication_id` should point to a deleted medication id.
- For any (patient_id, app_medication_id), there should be at most one `patient_medication` row (we merged duplicates).
- Spot-check a few users: their schedules and intakes should still show the same medications (now under the application medication name).
