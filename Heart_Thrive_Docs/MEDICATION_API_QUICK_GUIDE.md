# Medication API - Quick Reference Guide

## 🚀 Thirteen Powerful APIs

### 1. Search → 2. Add → 3. View List → 4. View Schedule → 5. Track Intake → 6. View Stats → 7. Upcoming → 8. Missed → 9. Get for Edit → 10. Update Schedule → 11. Delete Schedule → 12. Intake Summary → 13. Schedule Overview

---

## 1️⃣ Search Medications

**What it does:** Find medications in the database before adding them to your profile.

**URL:** `POST /api/medications/list`  
**Auth:** ❌ Not required  

**Input:**
```json
{
  "name": "aspirin",
  "brand": "bayer",
  "form": "tablet"
}
```

**Output:**
```json
[
  {
    "uuid": "550e8400-...",
    "name": "Aspirin",
    "brand": "Bayer",
    "strength": "500mg",
    "form": "Tablet"
  }
]
```

**Use when:** You want to find an existing medication before adding it.

---

## 2️⃣ Add Medication

**What it does:** Add a medication to your schedule with dosing times.

**URL:** `POST /api/medications/add`  
**Auth:** ✅ Required  

**Input (Using Existing Medication):**
```json
{
  "medicationUuid": "550e8400-...",
  "startDate": "2025-10-12",
  "endDate": "2025-11-12",
  "morning": true,
  "evening": true,
  "morningTime": "08:00:00",
  "eveningTime": "20:00:00",
  "isAfterMeal": true,
  "doseDescription": "500mg",
  "dosageFrequency": "Twice Daily",
  "daysOfWeek": ["Mon", "Wed", "Fri"]
}
```

**Input (Creating New Medication):**
```json
{
  "medicationUuid": null,
  "medicationName": "My Custom Medicine",
  "medicationBrand": "Generic",
  "startDate": "2025-10-12",
  "endDate": "2025-11-12",
  "morning": true,
  "morningTime": "08:00:00",
  "isAfterMeal": true,
  "doseDescription": "1 tablet",
  "dosageFrequency": "Once Daily",
  "daysOfWeek": ["Mon", "Tue", "Wed", "Thu", "Fri"]
}
```

**Output:**
```json
{
  "success": true,
  "message": "Medication added successfully with schedule",
  "data": {
    "uuid": "770e8400-...",
    "doseTime": "Morning, Evening",
    "startDate": "2025-10-12",
    "endDate": "2025-11-12",
    "isMorning": true,
    "isEvening": true,
    ...
  }
}
```

**Use when:** You want to add a medication to your schedule.

---

## 3️⃣ My Medications

**What it does:** View all your medications with current schedules. Can filter by name, time slots, or date range.

**URL:** `POST /api/medications/my-medications`  
**Auth:** ✅ Required  

**Input (No Filters - Show All):**
```json
{}
```

**Input (With Filters):**
```json
{
  "medicationName": "aspirin",
  "isMorning": true,
  "fromDate": "2025-10-15",
  "toDate": "2025-10-25"
}
```

**Output:**
```json
{
  "success": true,
  "message": "Found 1 medication(s)",
  "data": [
    {
      "medicationName": "Aspirin",
      "medicationBrand": "Bayer",
      "medicationForm": "Tablet",
      "scheduleUuid": "550e8400-...",
      "doseDescription": "500mg",
      "dosageFrequency": "Twice Daily",
      "daysOfWeek": "mon,wed,fri",
      "isMorning": true,
      "isAfterNoon": false,
      "isEvening": true,
      "morningTime": "08:00:00",
      "afternoonTime": null,
      "eveningTime": "20:00:00",
      "startDate": "2025-10-12",
      "endDate": "2025-11-12",
      "isAfterMeal": true,
      "active": true
    }
  ]
}
```

**Use when:** You want to see your medication list or find specific medications.

---

## 4️⃣ Medication Schedule List

**What it does:** Get detailed schedules for a date range, expanded by date and time slot. Shows what to take and when, with intake tracking.

**URL:** `POST /api/medications/schedule-list`  
**Auth:** ✅ Required  

**Input:**
```json
{
  "fromDate": "10-10-2025",
  "toDate": "15-10-2025",
  "timezone": "Asia/Kolkata",
  "medicationName": "Aspirin",
  "isMorning": true
}
```

**Output:**
```json
{
  "success": true,
  "message": "Found 6 schedule(s) for 1 medication(s)",
  "data": {
    "fromDate": "2025-10-10",
    "toDate": "2025-10-15",
    "totalSchedules": 6,
    "totalMedications": 1,
    "schedulesByDate": {
      "2025-10-10": [
        {
          "date": "2025-10-10",
          "dayOfWeek": "FRIDAY",
          "medicationName": "Aspirin",
          "timeSlot": "MORNING",
          "scheduledTime": "08:00:00",
          "isTaken": false
        }
      ]
    },
    "allSchedules": [ ... ]
  }
}
```

**Use when:** You want to see your medication schedule for specific dates with intake status.

---

## 5️⃣ Track Medication Intake

**What it does:** Mark a medication dose as taken or not taken. Records when you take your medications.

**URL:** `POST /api/medications/track-intake`  
**Auth:** ✅ Required  

**Input:**
```json
{
  "scheduleUuid": "550e8400-...",
  "date": "14-10-2025",
  "timeSlot": "MORNING",
  "isTaken": true,
  "timezone": "Asia/Kolkata"
}
```

**Output:**
```json
{
  "success": true,
  "message": "Created morning medication for Aspirin on 2025-10-14 - marked as taken",
  "data": {
    "intakeUuid": "intake-uuid-123",
    "intakeDate": "2025-10-14",
    "medicationName": "Aspirin",
    "timeSlot": "MORNING",
    "isTaken": true,
    "takenAtTime": "2025-10-14T03:30:00Z",
    "operation": "CREATED"
  }
}
```

**Use when:** User clicks "taken" or "not taken" on a medication dose.

---

## 6️⃣ Medication Intake Statistics

**What it does:** Get adherence statistics showing how well you're following your medication schedule.

**URL:** `POST /api/medications/intake-stats`  
**Auth:** ✅ Required  

**Input (Simple - Overall Stats):**
```json
{
  "fromDate": "01-10-2025",
  "toDate": "07-10-2025",
  "timezone": "Asia/Kolkata"
}
```

**Input (Detailed - With Medication Breakdown):**
```json
{
  "fromDate": "01-10-2025",
  "toDate": "07-10-2025",
  "timezone": "Asia/Kolkata",
  "isMorning": true,
  "isMedicationBreakdownRequired": true
}
```

**Input (Advanced - With Slot-Wise Breakdown):**
```json
{
  "fromDate": "01-10-2025",
  "toDate": "07-10-2025",
  "timezone": "Asia/Kolkata",
  "isSlotWiseBreakDownRequired": true
}
```

**Input (Complete Analysis - Both Breakdowns):**
```json
{
  "fromDate": "01-10-2025",
  "toDate": "07-10-2025",
  "timezone": "Asia/Kolkata",
  "isMedicationBreakdownRequired": true,
  "isSlotWiseBreakDownRequired": true
}
```

**Output (Simple):**
```json
{
  "success": true,
  "message": "Statistics for 7 day(s): 42 scheduled, 36 taken, 4 missed, 85.71% adherence",
  "data": {
    "totalDays": 7,
    "totalScheduled": 42,
    "totalTaken": 36,
    "totalNotTaken": 6,
    "totalMissed": 4,
    "overallAdherencePercentage": 85.71,
    "medicationBreakdown": null,
    "slotWiseBreakdown": null
  }
}
```

**Output (With Breakdowns):**
```json
{
  "success": true,
  "message": "Statistics for 7 day(s): 42 scheduled, 36 taken, 4 missed, 85.71% adherence",
  "data": {
    "totalScheduled": 42,
    "totalTaken": 36,
    "medicationBreakdown": [
      {"medicationName": "Aspirin", "adherencePercentage": 92.86, ...},
      {"medicationName": "Metformin", "adherencePercentage": 85.71, ...}
    ],
    "slotWiseBreakdown": [
      {"timeSlot": "AFTERNOON", "adherencePercentage": 71.43, ...},
      {"timeSlot": "EVENING", "adherencePercentage": 90.48, ...},
      {"timeSlot": "MORNING", "adherencePercentage": 85.71, ...}
    ]
  }
}
```

**Use when:** 
- ✅ See overall adherence reports
- ✅ Identify which medications need attention (medication breakdown)
- ✅ Identify which time slots have problems (slot breakdown)
- ✅ Complete adherence analysis (both breakdowns)

---

## 7️⃣ Upcoming Medication Schedules

**What it does:** Get future medication schedules (after current time). Perfect for "What's Next?" widgets or upcoming medication views.

**URL:** `POST /api/medications/upcoming-schedules`  
**Auth:** ✅ Required  

**Input (Next Slot Only):**
```json
{
  "fromDate": "15-10-2025",
  "toDate": "22-10-2025",
  "timezone": "Asia/Kolkata",
  "isAllSchedulesReq": false
}
```

**Input (All Upcoming):**
```json
{
  "fromDate": "15-10-2025",
  "toDate": "22-10-2025",
  "timezone": "Asia/Kolkata",
  "isAllSchedulesReq": true,
  "isMorning": true
}
```

**Output (Next Slot):**
```json
{
  "success": true,
  "message": "Next time slot: EVENING on 2025-10-15 - 3 schedule(s)",
  "data": {
    "fromDate": "2025-10-15",
    "toDate": "2025-10-22",
    "timezone": "Asia/Kolkata",
    "currentDateTime": "2025-10-15T14:30:00",
    "totalUpcomingSchedules": 3,
    "totalMedications": 3,
    "isAllSchedulesReq": false,
    "nextTimeSlot": "EVENING",
    "nextSlotDateTime": "2025-10-15T20:00:00",
    "allSchedules": [
      {
        "date": "2025-10-15",
        "timeSlot": "EVENING",
        "medicationName": "Aspirin",
        "scheduledTime": "20:00",
        "isTaken": false
      }
    ]
  }
}
```

**Use when:** 
- Dashboard "What's Next?" widget (`isAllSchedulesReq=false`)
- "Upcoming This Week" page (`isAllSchedulesReq=true`)
- Next dose reminder notifications
- Weekly medication planner

---

## 8️⃣ Missed Medication Schedules

**What it does:** Get detailed list of missed medication schedules. Unlike stats which returns counts, this returns the actual schedule entries.

**URL:** `POST /api/medications/missed-schedules`  
**Auth:** ✅ Required  

**Input:**
```json
{
  "fromDate": "01-10-2025",
  "toDate": "14-10-2025",
  "timezone": "Asia/Kolkata",
  "isMorning": true
}
```

**Output:**
```json
{
  "success": true,
  "message": "Found 8 missed schedule(s) for 3 medication(s) - Most missed: Aspirin (4 times)",
  "data": {
    "fromDate": "2025-10-01",
    "toDate": "2025-10-14",
    "timezone": "Asia/Kolkata",
    "currentDateTime": "2025-10-15T14:30:00",
    "totalMissedSchedules": 8,
    "totalMedications": 3,
    "mostMissedMedication": "Aspirin",
    "mostMissedCount": 4,
    "missedSchedulesByDate": {
      "2025-10-10": [
        {
          "date": "2025-10-10",
          "timeSlot": "MORNING",
          "medicationName": "Aspirin",
          "scheduledTime": "08:00",
          "isTaken": false
        }
      ]
    },
    "allMissedSchedules": [ ... ]
  }
}
```

**Use when:**
- "Missed Medications" report page
- Alert user about recent missed doses
- Historical adherence analysis
- Identify problematic medications

---

## 9️⃣ Get Schedule for Editing

**What it does:** Retrieve existing schedule details to pre-fill an edit form.

**URL:** `GET /api/medications/schedule/{scheduleUuid}`  
**Auth:** ✅ Required  

**Input:**
```
GET /api/medications/schedule/550e8400-e29b-41d4-a716-446655440000
```

**Output:**
```json
{
  "success": true,
  "message": "Schedule retrieved successfully",
  "data": {
    "scheduleUuid": "550e8400-...",
    "medicationName": "Aspirin",
    "medicationBrand": "Bayer",
    "medicationForm": "Tablet",
    "medicationStrength": "500mg",
    "startDate": "2025-10-01",
    "endDate": "2025-12-31",
    "morning": true,
    "afternoon": false,
    "evening": true,
    "morningTime": "08:00:00",
    "afternoonTime": null,
    "eveningTime": "20:00:00",
    "isAfterMeal": true,
    "doseDescription": "500 mg",
    "dosageFrequency": "Twice Daily",
    "daysOfWeek": ["Mon", "Wed", "Fri"],
    "active": true
  }
}
```

**Use when:** User clicks "Edit" on a schedule to pre-fill the edit form with current values.

**Key Features:**
- Returns medication info (name, brand) for display only - **cannot be edited**
- Returns all schedule fields with current values - **can be edited**
- Days of week auto-converted from "mon,wed,fri" to ["Mon", "Wed", "Fri"]
- Only schedule owner can retrieve it

---

## 🔟 Update Medication Schedule

**What it does:** Update an existing medication schedule (dates, times, frequency, days, etc.)

**URL:** `PUT /api/medications/edit-schedule`  
**Auth:** ✅ Required  

**Input:**
```json
{
  "scheduleUuid": "550e8400-...",
  "startDate": "2025-10-15",
  "endDate": "2026-03-31",
  "morning": true,
  "afternoon": false,
  "evening": true,
  "morningTime": "08:00:00",
  "eveningTime": "20:00:00",
  "isAfterMeal": true,
  "doseDescription": "500 mg",
  "dosageFrequency": "Twice Daily",
  "daysOfWeek": ["Mon", "Wed", "Fri"]
}
```

**Output:**
```json
{
  "success": true,
  "message": "Medication schedule updated successfully",
  "data": {
    "scheduleUuid": "550e8400-...",
    "medicationName": "Aspirin",
    "medicationBrand": "Bayer",
    "startDate": "2025-10-15",
    "endDate": "2026-03-31",
    "doseDescription": "500 mg",
    "dosageFrequency": "Twice Daily",
    "daysOfWeek": "mon,wed,fri",
    "isAfterMeal": true,
    "isMorning": true,
    "isAfterNoon": false,
    "isEvening": true,
    "morningTime": "08:00:00",
    "afternoonTime": null,
    "eveningTime": "20:00:00",
    "lastModifiedBy": "user@example.com",
    "lastModifiedDate": "2025-10-15T10:30:00Z",
    "message": "Successfully updated medication schedule for Aspirin"
  }
}
```

**Use when:** User modifies schedule and clicks "Save" to update it.

**Can Update:**
- ✅ Time period (startDate, endDate)
- ✅ Days of week
- ✅ Time slots (morning, afternoon, evening)
- ✅ Time values for each slot
- ✅ Meal timing (before/after)
- ✅ Dosage frequency
- ✅ Dose description

**Cannot Update:**
- ❌ Medication name
- ❌ Medication brand
- ℹ️ To change medication, create a new schedule

---

## 1️⃣1️⃣ Delete Medication Schedule

**What it does:** Delete (soft delete) a medication schedule. Always allowed. Preserves intake history.

**URL:** `DELETE /api/medications/schedule/{scheduleUuid}`  
**Auth:** ✅ Required  

**Input:**
```
DELETE /api/medications/schedule/550e8400-e29b-41d4-a716-446655440000
```

**Output (Success):**
```json
{
  "success": true,
  "message": "Medication schedule deleted successfully",
  "data": {
    "scheduleUuid": "550e8400-...",
    "medicationName": "Aspirin",
    "medicationBrand": "Bayer",
    "deleted": true,
    "deletedBy": "user@example.com",
    "deletedAt": "2025-10-16T10:30:00Z",
    "message": "Medication schedule for 'Aspirin' has been successfully deleted"
  }
}
```

**Use when:** 
- ✅ You added a schedule by mistake
- ✅ Want to stop taking a medication
- ✅ Doctor discontinued the medication
- ✅ Want to clean up old schedules
- ✅ Need to end medication early

**What happens:**
- ✅ Schedule becomes inactive (active=false)
- ✅ Removed from active medication lists
- ✅ **Intake history preserved** for medical records
- ✅ No future doses scheduled

**Important Rules:**
- **Soft delete only:** Sets active=false (data preserved)
- **Always allowed:** No restrictions based on intake history
- **Preserves history:** All intake records remain for medical records
- **Immediate:** Schedule removed from active lists right away

---

## 1️⃣2️⃣ Medication Intake Count Summary

**What it does:** Get combined adherence statistics, upcoming doses, and missed doses in one call. Optimized for dashboards.

**URL:** `POST /api/medications/intake-count-summary`  
**Auth:** ✅ Required  

**Input:**
```json
{
  "fromDate": "10-10-2025",
  "toDate": "15-10-2025",
  "timezone": "Asia/Kolkata",
  "isMorning": true,
  "isAfterNoon": true,
  "isEvening": true
}
```

**Output:**
```json
{
  "success": true,
  "message": "Intake Summary: 86.67% adherence, 21 upcoming doses, 2 missed doses",
  "data": {
    "totalMedications": 3,
    "totalScheduled": 36,
    "totalTaken": 13,
    "totalNotTaken": 23,
    "totalMissed": 2,
    "overallAdherencePercentage": 86.67,
    "totalUpcomingDoses": 21,
    "upcomingMedications": [...],
    "statusMessages": [
      "Next Dose: 8:00 pm - Aspirin",
      "Next Dose: 8:00 pm - Metformin",
      "Next Dose: 8:00 pm - Vitamin D"
    ],
    "totalMissedDoses": 2,
    "missedMedications": [...],
    "missedStatusMessages": [
      "Missed Dose: 8:00 pm - Vitamin D",
      "Missed Dose: 8:00 am - Aspirin"
    ],
    "mostMissedMedication": "Aspirin",
    "mostMissedCount": 1,
    "fromDate": "2025-10-10",
    "toDate": "2025-10-15",
    "timezone": "Asia/Kolkata"
  }
}
```

**Numbers Explained:**
- 📊 **36 total** doses scheduled (6 days × 3 medications × 2 doses/day)
- ✅ **13 taken** out of 15 past doses (86.67% adherence)
- ❌ **2 missed** past doses
- ⏰ **21 upcoming** future doses
- 📈 Math check: 13 taken + 23 not taken = 36 total ✓
```

**Use when:** 
- ✅ Building medication dashboard
- ✅ Need adherence stats + upcoming + missed doses
- ✅ Want status messages for display
- ✅ Optimize API calls (1 instead of 3)

**Benefits:**
- **60% less data** than separate calls
- **Single API call** instead of three
- **Status messages** ready for display
- **All dashboard data** in one response

---

## 1️⃣3️⃣ Medication Schedule Overview

**What it does:** Get complete schedule overview with all, upcoming, and missed schedules in one call.

**URL:** `POST /api/medications/schedule-overview`  
**Auth:** ✅ Required  

**Input:**
```json
{
  "fromDate": "10-10-2025",
  "toDate": "15-10-2025",
  "timezone": "Asia/Kolkata",
  "medicationName": "Aspirin",
  "isMorning": true,
  "isAfterNoon": true,
  "isEvening": true
}
```

**Output:**
```json
{
  "success": true,
  "message": "Schedule Overview: 18 total, 12 upcoming, 3 missed (15 taken)",
  "data": {
    "totalSchedules": 18,
    "allSchedules": [...],
    "totalUpcoming": 12,
    "upcomingSchedules": [...],
    "upcomingStatusMessages": [...],
    "totalMissed": 3,
    "missedSchedules": [...],
    "mostMissedMedication": "Aspirin",
    "mostMissedCount": 2,
    "totalTaken": 15,
    "totalMedications": 3,
    "fromDate": "2025-10-10",
    "toDate": "2025-10-15",
    "timezone": "Asia/Kolkata"
  }
}
```

**Use when:** 
- ✅ Need complete schedule overview
- ✅ Want all schedules categorized
- ✅ Building comprehensive schedule view
- ✅ Optimize API calls (1 instead of 3)

**Benefits:**
- **60% less data** than separate calls
- **Single API call** instead of three
- **Categorized schedules** (all, upcoming, missed)
- **Essential fields only** (7 vs 17 fields)

---

## 📋 Quick Comparison

| Feature | Search | Add | My Meds | Schedule | Track | Stats | Upcoming | Missed | Get Edit | Update | Delete | Intake Summary | Schedule Overview |
|---------|--------|-----|---------|----------|-------|-------|----------|--------|----------|--------|--------|----------------|------------------|
| **Purpose** | Find meds | Add schedule | View active | Daily doses | Mark taken | Adherence | Next doses | Missed list | Prep edit | Save changes | Remove | Combined stats+upcoming | All schedules categorized |
| **Auth** | ❌ No | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes |
| **Method** | POST | POST | POST | POST | POST | POST | POST | POST | GET | PUT | DELETE | POST | POST |
| **Output** | Search results | Schedule | Active meds | Doses+tracking | Intake log | Stats | Next list | Missed list | Schedule data | Updated | Success | Combined response | Categorized schedules |
| **Use For** | Discovery | Create | Current view | Daily tracking | Record | Reports | What's next | Adherence | Form pre-fill | Edit schedule | Stop med | Dashboard | Complete overview |

---

## 🎯 Common Scenarios

### Scenario 1: Add Aspirin (Existing Medication)
```
1. Search: POST /list {"name": "aspirin"}
2. Get UUID: "550e8400-..."
3. Add: POST /add {"medicationUuid": "550e8400-...", ...schedule...}
4. Done!
```

### Scenario 2: Add Custom Vitamin
```
1. Add: POST /add {"medicationName": "My Vitamin", ...schedule...}
2. Done! (No search needed)
```

### Scenario 3: View All My Medications
```
1. View: POST /my-medications {}
2. See list!
```

### Scenario 4: Find Morning Medications for Next Week
```
1. View: POST /my-medications {
     "isMorning": true,
     "fromDate": "2025-10-20",
     "toDate": "2025-10-27"
   }
2. See filtered list!
```

### Scenario 5: Track Daily Medications
```
1. Get today's schedule: POST /schedule-list {
     "fromDate": "14-10-2025",
     "toDate": "14-10-2025",
     "timezone": "Asia/Kolkata"
   }
2. See: Aspirin - MORNING - Not Taken
3. Take medication
4. Mark as taken: POST /track-intake {
     "scheduleUuid": "550e8400-...",
     "date": "14-10-2025",
     "timeSlot": "MORNING",
     "isTaken": true,
     "timezone": "Asia/Kolkata"
   }
5. Done! Next schedule list shows "Taken"
```

### Scenario 6: Weekly Adherence Report
```
1. Get stats: POST /intake-stats {
     "fromDate": "01-10-2025",
     "toDate": "07-10-2025",
     "timezone": "Asia/Kolkata",
     "isMedicationBreakdownRequired": true
   }
2. See: "85.71% adherence - 36 of 42 doses taken"
3. See breakdown: Aspirin 92%, Metformin 85%, Vitamin D3 71%
4. Identify: Vitamin D3 needs attention!
```

### Scenario 7: What's My Next Dose?
```
1. Get next dose: POST /upcoming-schedules {
     "fromDate": "15-10-2025",
     "toDate": "22-10-2025",
     "timezone": "Asia/Kolkata",
     "isAllSchedulesReq": false
   }
2. See: "Next: EVENING at 8:00 PM - Aspirin, Metformin, Vitamin D3"
3. Display on dashboard: "3 medications due at 8:00 PM"
```

### Scenario 8: View All Upcoming Week
```
1. Get all upcoming: POST /upcoming-schedules {
     "fromDate": "15-10-2025",
     "toDate": "22-10-2025",
     "timezone": "Asia/Kolkata",
     "isAllSchedulesReq": true
   }
2. See: 42 upcoming doses for the week
3. Plan ahead: Know what's coming
```

### Scenario 9: Review Missed Medications
```
1. Get missed list: POST /missed-schedules {
     "fromDate": "01-10-2025",
     "toDate": "14-10-2025",
     "timezone": "Asia/Kolkata"
   }
2. See: 8 missed doses over 2 weeks
3. See: "Most missed: Aspirin (4 times)"
4. Action: Set reminder for Aspirin doses
```

### Scenario 10: Edit Medication Schedule
```
1. User clicks "Edit" on Aspirin schedule
2. Get schedule data: GET /schedule/550e8400-...
3. Form pre-fills: Start: Oct 1, End: Dec 31, Days: Mon/Wed/Fri
4. User changes: End date to Mar 31, 2026, Days to Every Day
5. Save changes: PUT /edit-schedule {
     "scheduleUuid": "550e8400-...",
     "startDate": "2025-10-01",
     "endDate": "2026-03-31",
     "daysOfWeek": ["Mon", "Tue", "Wed", "Thu", "Fri", "Sat", "Sun"],
     ... (other fields)
   }
6. See: "Medication schedule updated successfully"
7. Schedule now runs daily until March 2026
```

### Scenario 11: Delete Schedule Added by Mistake
```
1. User adds "Aspirin" but meant to add "Ibuprofen"
2. Delete: DELETE /schedule/550e8400-...
3. See: "Medication schedule deleted successfully"
4. Schedule removed from list, intake history preserved
5. User can now add correct medication
```

### Scenario 12: Stop Active Medication
```
1. User has been taking "Metformin" daily for 2 weeks
2. Doctor discontinues medication
3. Delete: DELETE /schedule/abc-123-...
4. See: "Medication schedule deleted successfully"
5. Schedule removed from active lists
6. Past intake records preserved for medical history
7. No more future doses scheduled
```

---

## 🔑 Key Fields Explained

### medicationUuid
- Unique ID for a medication in the system
- Get it from the search endpoint
- Use it to add existing medications

### Time Slots (morning, afternoon, evening)
- Set to `true` for times you need to take medication
- Set to `false` if not needed
- Example: Morning and evening = breakfast and dinner time

### Time Values (morningTime, afternoonTime, eveningTime)
- Format: "HH:mm:ss" (24-hour)
- Examples: "08:00:00" (8 AM), "14:00:00" (2 PM), "20:00:00" (8 PM)
- Only needed if corresponding time slot is true

### Days of Week
- Array of days: ["Mon", "Tue", "Wed", "Thu", "Fri", "Sat", "Sun"]
- Everyday: Include all 7 days
- Specific days: ["Mon", "Wed", "Fri"]

### isAfterMeal
- `true` = Take after eating
- `false` = Take before eating or anytime

### Date Range (fromDate, toDate)
- Format: "YYYY-MM-DD" or "dd-MM-yyyy"
- Examples: "2025-10-12" or "12-10-2025"
- Used for when to start and stop taking medication

### scheduleUuid
- Unique ID for a medication schedule
- Get it from /schedule-list response
- Use it to track intake with /track-intake

### timeSlot
- Values: "MORNING", "AFTERNOON", or "EVENING"
- Case-sensitive (use uppercase)
- Used in /track-intake to specify which dose

### timezone
- IANA timezone identifier
- Example: "Asia/Kolkata" for IST
- Required for date-based endpoints
- Ensures dates are interpreted correctly

### isMedicationBreakdownRequired
- Used in /intake-stats
- true = Get per-medication statistics
- false/null = Get overall statistics only
- Controls what backend calculates

### isAllSchedulesReq
- Used in /upcoming-schedules
- true = Get ALL upcoming schedules in date range
- false = Get only NEXT immediate time slot
- Perfect for "What's Next?" vs "Weekly View"

### isAddToMyMedication
- Used in /add endpoint
- Reserved for future use (currently not functional)
- Planned feature: Save medication configuration for quick reference

---

## ⚠️ Important Notes

### For Add Medication:
- Either provide `medicationUuid` (existing) OR `medicationName` (new)
- `startDate` and `endDate` are required
- If time slot is enabled (true), provide the time
- Days of week should include at least one day

### For My Medications:
- All filters are optional
- Empty body `{}` shows everything
- Date filter needs both fromDate and toDate
- Time slot filters use AND logic (all selected must match)

### General:
- All POST requests need `Content-Type: application/json` header
- Authenticated endpoints need `Authorization: Bearer <token>` header
- Dates use format: YYYY-MM-DD
- Times use format: HH:mm:ss (24-hour)

---

## 🆘 Troubleshooting

**Problem:** "Medication not found with UUID"  
**Solution:** The UUID doesn't exist. Search again or create a new medication.

**Problem:** "Start date is required"  
**Solution:** Add `startDate` and `endDate` to your request.

**Problem:** "Unauthorized"  
**Solution:** Check your authentication token is valid and included in headers.

**Problem:** No results when filtering  
**Solution:** Your filters might be too restrictive. Try removing some filters.

**Problem:** Date range not working  
**Solution:** Make sure both `fromDate` and `toDate` are provided.

**Problem:** "Invalid time slot: MIDDAY"  
**Solution:** Use MORNING, AFTERNOON, or EVENING (uppercase).

**Problem:** "Medication schedule not found with UUID"  
**Solution:** The scheduleUuid doesn't exist or is inactive. Get fresh UUID from /schedule-list.

**Problem:** Statistics show 0 scheduled  
**Solution:** Check if you have medications added and date range overlaps with schedule dates.

**Problem:** Can't mark as taken  
**Solution:** Verify scheduleUuid is correct and date is within schedule period.

**Problem:** Want to stop a medication I'm taking  
**Solution:** Use DELETE `/schedule/{uuid}` to remove it. Your intake history will be preserved, and the schedule will become inactive immediately.

**Problem:** Deleted schedule by mistake  
**Solution:** Contact system administrator. API doesn't support restoration, but admin may be able to reactivate the schedule from the database.

---

## 📞 Quick Help

**Need to:**
- Find a medication? → Use `/list`
- Add medication to schedule? → Use `/add`
- See your medications? → Use `/my-medications`
- See daily schedule? → Use `/schedule-list`
- Mark medication as taken? → Use `/track-intake`
- View adherence stats? → Use `/intake-stats`
- See what's next to take? → Use `/upcoming-schedules`
- Review missed medications? → Use `/missed-schedules`
- Edit a medication schedule? → Use `/schedule/{uuid}` then `/edit-schedule`
- Delete/stop a medication? → Use `/schedule/{uuid}` DELETE (preserves intake history)

---

## 🔥 Pro Tips

### For Schedule List:
- Use same fromDate and toDate for single day (optimized response)
- Add filters (medicationName, time slots) to narrow results
- Use schedulesByDate for grouped view or allSchedules for flat list

### For Track Intake:
- Get scheduleUuid from /schedule-list response
- Can toggle - mark as taken, then unmark if mistake
- Each time slot tracked independently

### For Intake Stats:
- No filters = overall adherence for all time slots
- Add time slot filters to focus on specific times (morning only, etc.)
- Set isMedicationBreakdownRequired=true for detailed per-medication stats
- Perfect for weekly/monthly reports

### For Upcoming Schedules:
- Use isAllSchedulesReq=false for dashboard widgets (shows next dose only)
- Use isAllSchedulesReq=true for weekly planner views
- Add time slot filters to see only morning/evening upcoming doses
- Perfect for push notifications and reminders

### For Missed Schedules:
- Returns actual schedule entries, not just counts
- Shows "most missed medication" for quick insights
- Use time slot filters to see morning-only missed doses
- Perfect for adherence reports and intervention alerts

### For Schedule Editing:
- GET `/schedule/{uuid}` first to get current values
- Medication name/brand shown but **cannot be edited**
- All schedule fields can be edited
- PUT `/edit-schedule` with **all fields** (not partial update)
- Use to extend/reduce period, change days, add/remove time slots

### For Schedule Deletion:
- DELETE is **always allowed** regardless of intake history
- Performs soft delete (sets active=false)
- **All intake records preserved** for medical history
- Schedule removed from active lists immediately
- Cannot be restored via API (contact admin if needed)
- Use DELETE or UPDATE endDate - both preserve history, choose based on preference

---

## 🎯 Complete Workflow Example

**Monday Morning:**
1. Add Aspirin: POST /add (set morning + evening, daily)
2. View schedule: POST /schedule-list (see what to take)

**Daily:**
3. Check today: POST /schedule-list (fromDate = toDate = today)
4. Take morning dose: POST /track-intake (mark as taken)
5. Take evening dose: POST /track-intake (mark as taken)

**Sunday Evening:**
6. Weekly report: POST /intake-stats (see 7-day adherence)
7. Review: "95% adherence - Great week!"

**Need to Extend Medication:**
8. GET /schedule/{uuid} (retrieve current values)
9. Form shows: End date Dec 31, 2025
10. Change to: End date Mar 31, 2026
11. PUT /edit-schedule (save changes)
12. Done! Medication extended for 3 more months

**Added Wrong Medication:**
13. Oops! Added "Aspirin" but need "Ibuprofen"
14. DELETE /schedule/{uuid}
15. See: "Medication schedule deleted successfully"
16. Add correct medication: POST /add (Ibuprofen)
17. Done! Wrong schedule removed, intake history preserved, correct one added

**Need to Stop Active Medication:**
18. Doctor says stop taking Metformin (been taking for 2 weeks)
19. DELETE /schedule/{uuid}
20. See: "Medication schedule deleted successfully"
21. Schedule removed, intake history preserved
22. Done! No more future doses, medical records intact

---

**Version:** 2.5  
**Last Updated:** October 21, 2025  
**New in 2.5:** Added `/intake-count-summary` and `/schedule-overview` endpoints; Combined endpoints for optimized dashboard views with 59-60% data reduction  
**New in 2.4:** Removed My Medication Menu List feature (deprecated); `isAddToMyMedication` field reserved for future use  
**New in 2.2:** Added DELETE `/schedule/{uuid}` endpoint  
**New in 2.1:** Added schedule editing endpoints - GET `/schedule/{uuid}` and PUT `/edit-schedule`  
**New in 2.0:** Added Upcoming Schedules and Missed Schedules endpoints

For detailed documentation, see `MEDICATION_API_DOCUMENTATION.md`

