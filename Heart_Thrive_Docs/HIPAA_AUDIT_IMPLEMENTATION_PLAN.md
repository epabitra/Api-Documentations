# HIPAA Audit Controls – Research & Implementation Plan

**Document purpose:** Research summary and implementation plan for HIPAA §164.312(b) Audit Controls in the HeartThrive health application (USA). No code changes are implemented in this phase; this is planning only.

**This revision** explicitly addresses: (1) whether existing History tables replace the need for `audit_logs`; (2) POST-as-READ endpoints (sensitive data via POST) so they are not mis-audited as CREATE; (3) composite/multi-table operations (e.g. create-meal, medications/add) with one record per logical API call; (4) exactly how to collect each audit field in production; (5) production-safe rollout for a live project.

**Audience:** Senior backend engineers, compliance, and deployment leads.

---

## 1. HIPAA Rules Relevant to Auditing (USA)

### 1.1 Legal basis

- **45 CFR §164.312(b) – Audit Controls (Technical Safeguards)**  
  Covered entities and business associates must implement hardware, software, and/or procedural mechanisms that **record and examine activity** in information systems that contain or use **electronic Protected Health Information (ePHI)**.

- **Retention:** HIPAA requires retention of audit records for **six (6) years**. Stricter state laws apply if they require longer retention.

- **Scope:** Risk-based. All systems that contain or use ePHI must have appropriate audit controls. The rule does not list every data element; guidance and industry practice define what to log.

### 1.2 What must be recorded (industry / guidance)

| Category | What to record |
|----------|----------------|
| **Application audit trails** | User activities: open/close data, **create, read, update, delete** ePHI records. |
| **System-level** | **Login attempts (success and failure)**, login ID, date/time, device/app where possible. |
| **User audit trails** | User-initiated actions, authentication events, **access to ePHI** (which resource, which patient). |

**Recommended per-event data (aligns with FHIR AuditEvent and common practice):**

- **User identification** – who performed the action (e.g. login/user id).
- **Patient identifier** – which patient’s ePHI was involved (when applicable).
- **Resource / data accessed** – entity type and ID (e.g. `PatientProfile`, `PatientMedication`, id).
- **Action type** – CREATE, READ, UPDATE, DELETE, or other (e.g. LOGIN_SUCCESS, LOGIN_FAILURE).
- **Timestamp** – when the action occurred.
- **Outcome** – success vs failure (important for security and investigations).
- **Request context** – IP, user-agent, or similar where appropriate (already partially present).

---

## 2. Current State in HeartThrive

### 2.1 Existing audit-related components

| Component | Purpose | HIPAA relevance |
|-----------|---------|------------------|
| **`AuditLog`** (entity) | New table `audit_logs`: id, userId, action, entityName, description(2000), ipAddress, createdAt. | Intended for access/activity audit but **not yet wired to ePHI endpoints** and missing fields (see below). |
| **`AuditAspect`** | AOP `@Around("@annotation(auditable)")` – runs only on methods annotated with `@Auditable`. | Only logs **after** method success; no failure logging; no patient/resource ID. |
| **`@Auditable`** | Method-level annotation: `action`, `entity`. | **Only one use:** `FoodItemQueryService.countByCriteria` (food_item, COUNT). FoodItem is reference data, not ePHI. |
| **`AuditLogRepository`** | JPA repository for `AuditLog`. | Exists; no retention or indexing strategy defined. |
| **`PatientAuditLog`** | Business entity: intakeDate, auditTime, details, patient. | **Different concept** – patient-facing or operational “audit” (e.g. intake events), **not** the HIPAA access/activity log. Do not repurpose for §164.312(b). |
| **`AbstractAuditingEntity`** | createdBy, createdDate, lastModifiedBy, lastModifiedDate on entities. | **Data provenance** (who created/updated the row). Valuable but **not a substitute** for an access/activity audit trail. |
| **`AuditConfig`** | `AuditorAware<String>` for JPA auditing. | Supports createdBy/lastModifiedBy only. |

### 2.2 Gaps vs HIPAA audit requirements

1. **Login / authentication events**
   - **Success:** Not explicitly written to any audit log (only JWT issued in `AuthenticateController.authorize()`).
   - **Failure:** Not logged. Failed attempts are important for §164.312(b) and security.

2. **ePHI access not audited**
   - Almost all **Patient\*** and **Doctor\*** REST endpoints that read/write ePHI have **no** `@Auditable` (and no other audit).
   - Example ePHI resources: PatientProfile, PatientMedication, PatientMedicationSchedule, PatientMedicationIntake, PatientMealLog, PatientMealMenu, PatientHeartRiskMetric, PatientWeightHeightLog, PatientSymptom*, PatientNutrientIntake, PatientNotification, DoctorProfile, DoctorPatient, etc.
   - Account (GET/DELETE) and User (when used for patient/doctor data) also touch ePHI or identity.

3. **Audit record content**
   - Missing: **patient identifier**, **resource ID**, **outcome** (success/failure), **HTTP method** or action type at API level.
   - **description** is rarely set (annotation has no description attribute; aspect doesn’t set it).

4. **Failure and exceptions**
   - Aspect runs only on **successful** completion. If the method throws (e.g. 403/404), no audit record is written. HIPAA and security best practice require logging failed access attempts.

5. **Database**
   - **`audit_logs` table:** There is **no Liquibase changelog** for it. With `ddl-auto: none`, the table is **not** created in production. Must be added via Liquibase.

6. **Retention**
   - No documented or enforced 6-year retention policy for audit logs (configuration, archival, or purge policy).

7. **Annotation placement**
   - Today: only **method-level** `@Auditable`, and only on one non-ePHI method. No class-level or controller-level strategy; no consistent coverage of ePHI endpoints.

---

## 3. History Tables vs Access Audit – Do You Still Need `audit_logs`?

### 3.1 What your existing History tables do

You already have **History** tables that track *data changes*:

| Table | Purpose |
|-------|--------|
| **patient_weight_height_log_history** | Snapshot of PatientWeightHeightLog on INSERT/UPDATE/DELETE (operation_type, original_id, full row copy). |
| **patient_medication_schedule_history** | Snapshot of PatientMedicationSchedule at each change (original_schedule_id, operation_type, field copies). |
| **patient_heart_risk_metric_history** | Similar versioning for PatientHeartRiskMetric. |

These answer: **“What changed in the data?”** (data versioning, point-in-time state). They do **not** record who *accessed* the data (which user called which API, when, from where, and whether it succeeded or failed).

### 3.2 What HIPAA §164.312(b) requires

The **Audit Controls** standard requires recording and examining **activity** in systems that contain ePHI – i.e. **access and usage**, not only data changes:

- Who (user) did what (action: read, create, update, delete, login).
- When, from where (IP), and whether it succeeded or failed.
- Which resource/patient was involved.

So:

- **History tables** = *what* changed in the data (versioning).
- **Access audit (e.g. `audit_logs`)** = *who* accessed or used ePHI, *when*, and *how* (action + outcome).

**Conclusion:** You **do** need a dedicated **access/activity audit** store. Your History tables are complementary; they are **not** a substitute for `audit_logs` (or an equivalent access-audit table). Keep History tables as-is; add and use **one** access-audit table for HIPAA.

---

## 4. Do You Need Additional Tables?

### 4.1 Recommendation: one dedicated HIPAA access-audit table

- **Keep a single access/activity audit table** (current `audit_logs` concept), extended with the fields below.
- **Do not** use `PatientAuditLog` for HIPAA access auditing; keep it for its current business meaning (e.g. intake/event log per patient).
- **Do not** repurpose History tables for access audit; they serve a different purpose (see §3).

### 4.2 Suggested schema for `audit_logs` (Liquibase)

Add or extend the table so each row represents one auditable event (e.g. one API call or one login attempt):

| Column | Type | Purpose |
|--------|------|--------|
| id | BIGINT PK | Surrogate key. |
| event_time | TIMESTAMP (UTC) | When the event occurred. |
| user_id / username | VARCHAR | Who (login or user id). Null for unauthenticated (e.g. failed login). |
| action | VARCHAR | CREATE, READ, UPDATE, DELETE, LOGIN_SUCCESS, LOGIN_FAILURE, etc. |
| resource_type | VARCHAR | Entity/API resource (e.g. PatientProfile, PatientMedication, authenticate). |
| resource_id | VARCHAR | ID of the resource (e.g. entity id). Null when N/A (e.g. list, login). |
| patient_id | VARCHAR/BIGINT | Patient user/id when the resource is tied to a patient. Null when N/A. |
| outcome | VARCHAR | SUCCESS, FAILURE (and optionally DENIED, ERROR). |
| request_uri | VARCHAR | e.g. /api/patient-profiles/123. Optional but useful. |
| http_method | VARCHAR | GET, POST, PUT, PATCH, DELETE. Optional. |
| ip_address | VARCHAR | Client IP. |
| user_agent | VARCHAR(500) | Optional; for device/app. |
| description | VARCHAR(2000) | Optional free text (e.g. reason, error summary). |

- Indexes: `event_time`, `user_id`, `resource_type`, `patient_id` (and optionally composite) for 6-year query and retention management.
- **Retention:** Document that this table (or its exported/archived data) is retained for at least 6 years; implement archival/purge policy later if needed.

### 4.3 Other tables

- **No additional tables are strictly required** for basic HIPAA audit controls if the single `audit_logs` table is implemented and populated as above.
- Optional later: separate **login_audit** or **auth_events** table if you want to isolate auth events for reporting; functionally the same can be achieved with `resource_type = 'authenticate'` (or similar) in `audit_logs`.

---

## 5. POST-as-READ: Sensitive Data via POST (Critical for Correct Action)

### 5.1 The problem

Many of your endpoints that **return** sensitive/ePHI data use **POST** (so filters can pass criteria in the body). If the audit layer infers action **only** from HTTP method, every POST would be logged as **CREATE**, which is wrong for read-only operations and would create a serious compliance and investigation issue.

### 5.2 Endpoints that must be treated as READ (not CREATE)

The following **POST** endpoints are **read/query** operations (they return data; they do not create new ePHI records). They **must** be audited as **action = READ**:

| Path (pattern or literal) | Resource type (suggested) | Notes |
|---------------------------|---------------------------|--------|
| POST `/api/medications/list` | medication (list) | List medications with filters. |
| POST `/api/medications/my-medications` | patient_medication | Get current user’s medications (ePHI). |
| POST `/api/medications/schedule-list` | patient_medication_schedule | Get schedule list. |
| POST `/api/medications/intake-stats` | patient_medication_intake | Intake statistics. |
| POST `/api/medications/intake-count-summary` | patient_medication_intake | Intake count summary. |
| POST `/api/medications/daily-intake-stats` | patient_medication_intake | Daily intake stats. |
| POST `/api/medications/upcoming-schedules` | patient_medication_schedule | Upcoming schedules. |
| POST `/api/medications/missed-schedules` | patient_medication_schedule | Missed schedules. |
| POST `/api/medications/schedule-overview` | patient_medication_schedule | Schedule overview. |
| POST `/api/patient-heart-risk-metrics/riskMetricDashboard` | patient_heart_risk_metric | Dashboard data (ePHI). |
| POST `/api/patient-heart-risk-metrics/heart-risk-summary` | patient_heart_risk_metric | Heart risk summary. |
| POST `/api/patient-heart-risk-metric-histories/heart-risk-summary` | patient_heart_risk_metric_history | Heart risk summary (history). |

### 5.3 Rule: never infer action from HTTP method alone for POST

- **Implementation:** Maintain an **explicit path → action (and optionally resource_type) mapping** for all ePHI endpoints that use POST.
- For any request: first check if the **exact path or path pattern** is in this mapping; if **yes**, use the mapped **action** (e.g. READ) and **resource_type**. If **no**, then fall back to HTTP method (POST → CREATE, etc.).
- Add new POST-as-READ endpoints to this mapping as they are introduced. This avoids ever logging a “sensitive GET via POST” as CREATE.

---

## 6. Composite / Multi-Table Operations (e.g. create-meal, medications/add)

### 6.1 Examples in your app

- **POST `/api/patient-meal-logs/create-meal`**  
  One API call can create/update: **PatientMealLog**, **PatientNutrientIntake** (multiple rows), **PatientMealMenu** / **PatientMealMenuItem**, and possibly **FoodItem** / **FoodNutrient**.  
- **POST `/api/medications/add`**  
  Creates/links: PatientMedication (or reuse), PatientMedicationSchedule, and related.  
- **POST `/api/medications/create`**  
  Creates medication details plus schedules (multiple tables).

### 6.2 How to audit: one record per API request (logical operation)

- **Do not** write one audit record per database table or per entity. That would duplicate events and make “one user action = one audit event” unclear.
- **Do** write **one** audit record per **API request** (one per logical operation):
  - **action:** CREATE (or UPDATE if the API is an update).
  - **resource_type:** A single logical name, e.g. `create_meal`, `add_medication`, or `patient_meal_log` (primary resource). Choose one and use it consistently.
  - **resource_id:** Leave **null** for composite creates, or set to the primary created ID if you capture it (e.g. new meal log id) without adding complexity in v1.
  - **patient_id:** Current user id (for patient-scoped APIs). Source: SecurityContext after authentication.
- This satisfies HIPAA (“user activity … create, read, update, delete”) at the **operation** level without tying the audit schema to your internal table layout.

---

## 7. How to Collect Each Audit Field (Production)

Use these **exact sources** so implementation is consistent and safe for a live system.

| Field | Source | Notes |
|-------|--------|------|
| **event_time** | Server clock at time of writing the log (UTC). | Use `Instant.now()` or equivalent; store in DB as UTC. |
| **user_id** | `SecurityContextHolder.getContext().getAuthentication().getName()` (after the request is processed). | Null for unauthenticated (e.g. failed login). Use login/email consistently. |
| **action** | From **path + method mapping** (§5) first; if not in map, then GET→READ, POST→CREATE, PUT/PATCH→UPDATE, DELETE→DELETE. Override from `@Auditable` if present on the controller method. | Never infer action from POST alone for paths in the POST-as-READ list. |
| **resource_type** | From same mapping as action (path pattern → resource_type), or from path segment (e.g. `/api/patient-profiles` → patient_profiles). Override from `@Auditable.entity` if present. | Normalize to a short, stable string (e.g. patient_meal_log, create_meal). |
| **resource_id** | Path variable `id` (or `uuid`) when the request path contains it (e.g. `/api/patient-profiles/123` → 123). Null for list/create endpoints or composite operations unless you explicitly set it. | Do not parse request body in filter for IDs (avoid body read/stream issues). |
| **patient_id** | For patient-scoped APIs: **current user’s id** from SecurityContext (same user as actor). Optional: request attribute `auditPatientId` set by controller for doctor-viewing-patient flows. | Prefer current user id when the API is “current user’s data.” |
| **outcome** | After request completes: if response status 2xx → SUCCESS; 4xx (e.g. 403/404) → FAILURE or DENIED; 5xx or unhandled exception → FAILURE or ERROR. | Filter or interceptor must run in “after” phase and inspect status or exception. |
| **request_uri** | `HttpServletRequest.getRequestURI()`. | Do not include query string if it can contain sensitive params, or truncate. |
| **http_method** | `HttpServletRequest.getMethod()`. | GET, POST, PUT, PATCH, DELETE. |
| **ip_address** | `HttpServletRequest.getRemoteAddr()` (or X-Forwarded-For if behind proxy, with policy for trust). | Already used in your current AuditAspect. |
| **user_agent** | `HttpServletRequest.getHeader("User-Agent")`. | Truncate to 500 chars if needed. |
| **description** | Optional. Only from `@Auditable` override or from login failure (e.g. “invalid credentials”). | **Never** log request/response body or ePHI. |

Important for production:

- **Do not** read or log request/response body in the audit layer (no ePHI in audit records; avoids stream consumption and PII exposure).
- **Do not** depend on path variables that are only available inside the controller; in a filter you have the path string and can parse path templates (e.g. `/api/patient-profiles/{id}`) to extract id if your framework exposes it, or use a request attribute set once in a thin layer that has access to path variables.

---

## 8. Where to Apply Auditing: Method vs Endpoint vs Class vs Global

### 8.1 Options

| Approach | Pros | Cons |
|----------|------|------|
| **Method-level `@Auditable`** (current) | Explicit, per use-case control. | Easy to miss endpoints; many annotations; resource/patient ID must be passed or inferred. |
| **Class-level** (e.g. on `@RestController`) | One annotation per resource class. | Less granular (same action for all methods unless overridden); resource/patient ID still need to be derived. |
| **Endpoint-level** (on each `@GetMapping` etc.) | Same as method-level; aligns with “per endpoint” thinking. | In your stack, “endpoint” is the method; so same as method-level. |
| **Global (filter / interceptor / aspect)** | One place; can’t forget an endpoint. | Must decide which paths are ePHI; must derive resource/patient from path or context; can be noisier. |

### 8.2 Recommended hybrid (senior-engineer approach)

1. **Use a single, centralized mechanism** so behavior and schema are consistent and easy to review for compliance:
   - **Option A – HTTP-layer (recommended):** A **filter or interceptor** that runs for (at least) all `/api/**` requests (or a curated list of ePHI paths). It:
     - Resolves **action** and **resource_type** from the **path + method mapping** first (§5); only if the path is not in the mapping, infer from HTTP method (GET→READ, POST→CREATE, etc.).
     - Extracts **resource_id** from path (e.g. `/api/patient-profiles/{id}`) when present; **patient_id** = current user id for patient-scoped APIs, or from a request attribute for doctor-viewing-patient.
     - Logs **after** request (success or failure) with **outcome** from response status or exception.
   - **Option B – Aspect on controllers:** Same semantics; aspect has access to method and can use the same mapping keyed by path + method.

2. **Keep `@Auditable` as an optional override** for custom action/entity/description (e.g. composite operations like create-meal where resource_type = `create_meal`). Prefer **method-level** for overrides.

3. **Explicit login audit** in **AuthenticateController**: on success and on failure (LOGIN_SUCCESS / LOGIN_FAILURE), with outcome and no body logging.

4. **Exclude** non-ePHI and noisy paths (public APIs, health, swagger, static).

This gives you correct **READ** for POST-as-read endpoints, **one record per logical operation** for composite calls, and a single place to maintain mapping and overrides.

---

## 9. Production Safety and Rollout (Live Project)

Because this is a **live** application and you will move to production directly, the following minimizes risk.

### 9.1 Principles

- **No change to business logic:** Auditing is additive only. No modification of existing service/controller behavior for correctness; only adding logging and (optionally) request attributes for audit.
- **Fail-open for audit:** If writing an audit record fails (DB error, etc.), **do not** fail the user’s request. Log the error and continue. Optionally use async write so audit persistence cannot block or crash the request thread.
- **Feature flag (optional but recommended):** Use a property (e.g. `app.audit.hipaa.enabled`) to turn the ePHI audit filter on/off. Deploy with it off, verify in staging, then enable in production.

### 9.2 Rollout order

1. **Schema only:** Add Liquibase changelog for `audit_logs` and run it. No code that writes to the table yet. Ensures table exists and is indexed.
2. **AuditLogService:** Implement the service that writes one record (sync or async). Test it in isolation (e.g. unit tests or a small admin-only test endpoint). Do not attach to request path yet.
3. **Path mapping and filter:** Implement the path→action/resource_type mapping (§5) and the filter/interceptor that collects fields as in §7 and calls AuditLogService. Register the filter but **exclude all paths** (or enable only for a single test path) so no production traffic is audited yet.
4. **Login audit:** Add login success/failure audit in AuthenticateController; call AuditLogService directly. Verify logs appear for login only.
5. **Enable ePHI paths gradually:** Add ePHI path patterns to the filter’s “audit” list (or turn on the feature flag). Prefer enabling by path list so you can start with a small set (e.g. account, patient-profiles) and then expand.
6. **Monitor:** Watch for errors in audit write (log and metrics). Ensure no increase in request failures or latency. If using async, monitor queue depth and failure rate.

### 9.3 What to avoid

- **Do not** read request body in a filter for audit (stream consumption, performance, and PII risk). Use only path, method, headers, SecurityContext, and response status.
- **Do not** log passwords, tokens, or ePHI in the description or any audit field.
- **Do not** block the request on audit DB write in the same thread if you can avoid it (use async or fire-and-forget with error logging).

---

## 10. Implementation Plan (Checklist)

Use this as the order of work; no code is implemented in this phase.

### Phase 1 – Schema and core audit service

- [ ] **1.1** Add Liquibase changelog for `audit_logs` with columns: id, event_time (UTC), user_id, action, resource_type, resource_id, patient_id, outcome, request_uri, http_method, ip_address, user_agent, description; add indexes (event_time, user_id, resource_type, patient_id).
- [ ] **1.2** Update `AuditLog` entity to match (or create a new entity, e.g. `AccessAuditLog`, and keep `AuditLog` only if you still need the old shape elsewhere). Use a single table for all access/activity events.
- [ ] **1.3** Introduce an **AuditLogService** (or similar) that takes structured parameters (userId, action, resourceType, resourceId, patientId, outcome, requestUri, httpMethod, ip, userAgent, description) and persists one record. Use it from both HTTP-layer and login flow.
- [ ] **1.4** Ensure **event_time** is stored in UTC and that retention (6 years) is documented; no purge of data older than 6 years without a written policy.

### Phase 2 – Path mapping and request-scoped audit

- [ ] **2.0** Implement **path + method → action, resource_type mapping** (§5): Maintain a config or in-code map for all POST-as-READ and any other special paths. For each such path, store action (e.g. READ) and resource_type. All other paths use default: GET→READ, POST→CREATE, PUT/PATCH→UPDATE, DELETE→DELETE; resource_type from path segment.
- [ ] **2.1** Implement **request-scoped audit** (filter or interceptor):
  - Apply to ePHI path prefixes (or all `/api/**` with exclusions for public and management). Use the mapping from 2.0 to resolve **action** and **resource_type** (never infer action from POST alone for mapped paths).
  - Derive **resource_id** from path variable when present; **patient_id** = current user id for patient-scoped APIs, or request attribute for doctor-viewing-patient.
  - For **composite** operations (§6): one audit record per request; resource_type = logical name (e.g. `create_meal`, `add_medication`); resource_id can be null in v1.
  - On completion (success or exception): set **outcome** (SUCCESS / FAILURE / DENIED), get user from SecurityContext, IP and User-Agent from request; call AuditLogService. Collect fields only as in §7 (no body read).
- [ ] **2.2** **Login success:** In `AuthenticateController.authorize()` after successful authentication, call AuditLogService (LOGIN_SUCCESS, resource_type=authenticate, user_id, outcome=SUCCESS, ip, user_agent).
- [ ] **2.3** **Login failure:** Add handling for failed authentication. Log LOGIN_FAILURE, outcome=FAILURE, no user_id, ip, optional description; do not log password.
- [ ] **2.4** **Account deletion:** When `AccountResource.deleteAccount()` is called, log ACCOUNT_DELETE (or equivalent) with user_id and outcome.

### Phase 3 – ePHI coverage and overrides

- [ ] **3.1** Maintain the **ePHI path list** and the **POST-as-READ mapping** (§5). Include all paths that touch ePHI and all POST endpoints that are reads (medications/list, my-medications, schedule-list, intake-stats, etc.; patient-heart-risk-metrics/riskMetricDashboard, heart-risk-summary; etc.).
- [ ] **3.2** **patient_id:** Use current user id for patient-scoped APIs; optionally support request attribute `auditPatientId` for doctor-viewing-patient. Do not read request body in filter.
- [ ] **3.3** **Method-level `@Auditable`:** Use only as override for action/entity/description when present. Document that primary coverage is the global mechanism + mapping.

### Phase 4 – Security and ops

- [ ] **4.1** Restrict **read access** to `audit_logs` (and any export): only authorized roles (e.g. admin, compliance) and from secure channels; no exposure via public or patient-facing APIs.
- [ ] **4.2** Do **not** log request/response bodies (to avoid logging ePHI inside the audit trail). Log only identifiers (ids), action, resource type, outcome, and request metadata (URI, method, IP, user-agent).
- [ ] **4.3** Document the 6-year retention policy and how it will be enforced (e.g. no delete in app for audit table; archival job and purge only after 6 years with approval).

### Phase 5 – Optional improvements (post–go-live)

- [ ] **5.1** Add a simple **admin-only** API or report to query audit logs (by user, date range, resource type, patient) for compliance reviews.
- [ ] **5.2** Consider **async writing** (e.g. queue or async method) so audit logging does not block the request path; ensure no loss under normal load.
- [ ] **5.3** If you need to support “viewed by” or “export” semantics, add actions like READ_EXPORT or READ_PRINT and log them from the same AuditLogService.

---

## 11. Summary: Your Questions Answered

- **We already have History tables; do we need `audit_logs` or other new tables?**  
  **Yes, you still need one access-audit table.** History tables answer *what changed in the data* (versioning). HIPAA §164.312(b) requires *who accessed or used ePHI, when, and with what outcome* (activity/access audit). Those are different. Keep your History tables; add **one** `audit_logs` (or equivalent) table for access/activity only. No other new tables are required.

- **Some GET-like endpoints use POST (sensitive params in body); won’t they be treated as CREATE?**  
  **Not if you implement the path mapping.** Maintain an explicit **path + method → action (and resource_type) mapping**. All POST-as-READ endpoints listed in §5 must be in this mapping with **action = READ**. The filter uses the mapping first; only unmapped paths fall back to HTTP method (POST→CREATE). So POST `/api/medications/my-medications` is audited as READ, not CREATE.

- **Endpoints like create-meal write to multiple tables; how do we handle them?**  
  **One audit record per API request (logical operation).** Use a single **resource_type** (e.g. `create_meal` or `patient_meal_log`). Do **not** log one record per DB table. **resource_id** can be null for composite creates in v1, or the primary created ID if you capture it without adding complexity. **patient_id** = current user id. This satisfies HIPAA at the operation level.

- **How do we collect all the suggested audit fields?**  
  Use the **exact sources** in §7: user_id from SecurityContext; action/resource_type from path+mapping (and @Auditable override); resource_id from path variable; patient_id from current user or request attribute; outcome from response/exception; request_uri, http_method, ip, user_agent from `HttpServletRequest`. **Do not** read or log request/response body. Event time = server UTC when writing the log.

- **This is a live project; how do we avoid serious issues before production?**  
  Follow **§9 Production Safety and Rollout**: (1) No change to business logic; (2) fail-open for audit (if audit write fails, do not fail the user request); (3) optional feature flag to enable audit; (4) rollout order: schema → AuditLogService → filter with all paths excluded → login audit only → enable ePHI paths gradually; (5) do not read body in filter; do not log passwords or ePHI; prefer async audit write.

- **Do you need to handle the annotation anywhere else?**  
  **Yes.** Use a **global** filter/interceptor (with path→action mapping) for coverage. Use **method-level `@Auditable`** only as an **override** for custom action/entity/description. Handle **login success and failure** explicitly in the authentication flow.

After implementing the plan above, you will have a single access-audit trail that correctly records READ vs CREATE for POST-as-READ endpoints, one record per logical operation for composite calls, and safe collection of fields without touching request body, and you can roll out to production in a controlled way.
