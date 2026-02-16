# HIPAA Audit Implementation – Developer Reference

This document explains **how** the HIPAA §164.312(b) Audit Controls are implemented in the HeartThrive backend. Use it when maintaining, extending, or debugging the audit trail, or when onboarding developers who need to work with it.

For the **planning** document (requirements, rationale, rollout), see `docs/HIPAA_AUDIT_IMPLEMENTATION_PLAN.md`.

---

## 1. What This Implementation Does

- **Records access/activity** for systems that contain ePHI: who did what, when, and whether it succeeded or failed.
- **One row per auditable event** in the `audit_logs` table (e.g. one API call or one login attempt).
- **Does not** log request/response bodies or ePHI; only identifiers, action, resource type, outcome, and request metadata (URI, method, IP, user-agent).
- **Fail-open:** if writing an audit record fails, the user’s request is not failed; the error is logged.

It does **not** replace:
- **History tables** (e.g. patient_medication_schedule_history) – those record *what changed* in data; audit_logs record *who accessed or used* ePHI and with what outcome.
- **PatientAuditLog** – that is a different business concept (e.g. intake/event log per patient), not the HIPAA access/activity log.

---

## 2. Where Everything Lives

| What | Location |
|------|----------|
| **Database table** | `audit_logs` (schema via SQL scripts, not Liquibase) |
| **Entity** | `com.ht.audit.AuditLog` |
| **Repository** | `com.ht.repository.AuditLogRepository` |
| **Service** | `com.ht.service.AccessAuditService` → `AccessAuditServiceImpl` |
| **Path → action/resource mapping** | `com.ht.audit.HipaaAuditPathMapping` |
| **Request-scoped audit (filter)** | `com.ht.audit.HipaaAuditFilter` |
| **@Auditable override (request attributes)** | `com.ht.audit.AuditAspect` |
| **Login success/failure audit** | `AuthenticateController.authorize(...)` |
| **Account delete/reset audit** | `AccountResource.deleteAccount(...)`, `AccountResource.resetAccount(...)` |
| **Feature flag** | `app.audit.hipaa.enabled` (e.g. in `application.yml`) |
| **Filter registration** | `SecurityConfiguration` (bean `hipaaAuditFilter()`, added after `BearerTokenAuthenticationFilter`) |
| **SQL scripts** | `src/main/resources/sql/create_audit_logs_hipaa.sql`, `alter_audit_logs_hipaa.sql` |

---

## 3. Database: `audit_logs` Table

### 3.1 Schema (conceptual)

| Column | Type | Purpose |
|--------|------|---------|
| id | BIGINT PK | Surrogate key |
| event_time | TIMESTAMP(6) | When the event occurred (UTC) |
| user_id | VARCHAR(255) | Who performed the action (null for e.g. failed login) |
| action | VARCHAR(100) | CREATE, READ, UPDATE, DELETE, LOGIN_SUCCESS, LOGIN_FAILURE, ACCOUNT_DELETE, ACCOUNT_RESET |
| resource_type | VARCHAR(255) | Entity/API resource (e.g. PatientProfile, authenticate) |
| resource_id | VARCHAR(255) | ID of the resource (null for list/login/composite) |
| patient_id | VARCHAR(255) | Patient user/id when applicable |
| outcome | VARCHAR(50) | SUCCESS, FAILURE, DENIED, ERROR |
| request_uri | VARCHAR(2000) | e.g. /api/patient-profiles/123 (no query string) |
| http_method | VARCHAR(10) | GET, POST, PUT, PATCH, DELETE |
| ip_address | VARCHAR(100) | Client IP |
| user_agent | VARCHAR(500) | Device/app (truncated) |
| description | VARCHAR(2000) | Optional; never ePHI or passwords |

Indexes are on `event_time`, `user_id`, `resource_type`, `patient_id` for querying and retention management.

### 3.2 Scripts (no Liquibase)

- **New environment / table does not exist:** run `sql/create_audit_logs_hipaa.sql` (creates table and indexes).
- **Table already exists** (e.g. from JPA with old columns): run `sql/alter_audit_logs_hipaa.sql` (adds new columns and renames as needed).

Do **not** add Liquibase changelogs for this table; schema is maintained with these manual SQL scripts.

### 3.3 Retention

HIPAA requires retention of audit records for **at least 6 years**. The application does not purge `audit_logs`; archival or purge (e.g. after 6 years) should be done via a documented process outside the app (e.g. DB job or export then delete with approval).

---

## 4. How an API Request Gets Audited

### 4.1 Flow (high level)

1. Request hits the security filter chain (JWT resolved, SecurityContext set).
2. Request is processed (controller, service, etc.).
3. **HipaaAuditFilter** runs in a `finally` block after the chain, so it runs for both success and failure (e.g. 403, 404).
4. If the path is in scope and the feature flag is on, the filter builds one audit record and calls `AccessAuditService.log(...)`.
5. **AccessAuditServiceImpl** persists one `AuditLog` row (async when `@Async` is available). Exceptions are caught and logged; they do not fail the request.

### 4.2 Which paths are audited

The filter uses a **regex** to include most `/api/` paths and **excludes**:

- `/api/authenticate`
- `/api/password/`
- `/api/email-verification/`
- `/api/users/create`
- `GET /api/roles` (exact)
- `/api/app-param-values/get-gender`, `/api/app-param-values/get-meal-types`

So: public/auth and a few public reference endpoints are not audited; everything else under `/api/` that matches the pattern is.

### 4.3 How action and resource_type are chosen

- **First:** Check **path + method** in `HipaaAuditPathMapping`. If the request matches (e.g. `POST /api/medications/my-medications`), use the **mapped** action and resource_type (e.g. READ, patient_medication). This is what makes “POST-as-READ” endpoints get audited as READ, not CREATE.
- **Override:** If the controller method has **@Auditable**, the aspect sets request attributes before the method runs; the filter reads those and uses them instead of the mapping (so one record per request with the overridden action/entity/description).
- **Default:** If there is no mapping and no @Auditable, action is inferred from HTTP method (GET→READ, POST→CREATE, PUT/PATCH→UPDATE, DELETE→DELETE), and resource_type from the first path segment after `/api/` (e.g. patient_profiles).

**Important:** For any POST that returns sensitive data (list/query), add it to `HipaaAuditPathMapping` with action READ and the right resource_type so it is never logged as CREATE.

### 4.4 Resource ID from path

`resource_id` is the **first** path segment after `/api/` that looks like an id:

- **Numeric** (e.g. `123`), or  
- **UUID** (8-4-4-4-12 hex).

Examples:

- `/api/patient-profiles/123` → resource_id `123`
- `/api/patient-profiles/123/info` → resource_id `123` (not `info`)
- `/api/patient-profiles/create` → resource_id null

Logic lives in `HipaaAuditFilter.extractResourceIdFromPath(String path)`.

### 4.5 Outcome

Set after the request completes:

- **SUCCESS** – HTTP 2xx  
- **DENIED** – 401 or 403  
- **FAILURE** – 4xx (other)  
- **ERROR** – 5xx or response not committed (e.g. exception)

No request or response body is read; only status (and whether the response was committed) is used.

---

## 5. POST-as-READ Mapping (critical for compliance)

Many endpoints that **return** ePHI use **POST** (so criteria can be in the body). If we inferred action only from HTTP method, every such POST would be logged as CREATE, which is wrong.

**Solution:** Maintain an explicit **path + method → action, resource_type** map in `HipaaAuditPathMapping`. All known “POST that is really a read” endpoints are in that map with **action = READ**.

Current POST-as-READ entries (conceptually) include:

- Medications: list, my-medications, schedule-list, intake-stats, intake-count-summary, daily-intake-stats, upcoming-schedules, missed-schedules, schedule-overview  
- Patient heart risk: riskMetricDashboard, heart-risk-summary (and history heart-risk-summary)

Composite operations (one logical create per request) are also mapped, e.g.:

- `POST /api/patient-meal-logs/create-meal` → CREATE, create_meal  
- `POST /api/medications/add` → CREATE, add_medication  
- `POST /api/medications/create` → CREATE, patient_medication  

When you add a **new** POST endpoint that returns data (read) or that is a composite create, update `HipaaAuditPathMapping` accordingly so the audit stays correct.

---

## 6. Login and Account Events (explicit audit)

These are **not** left to the filter; they are written explicitly in the code so that success and failure are both logged with the right action and no ePHI.

| Event | Where | Action | resource_type | user_id on failure |
|-------|--------|--------|----------------|--------------------|
| Login success | `AuthenticateController.authorize(...)` | LOGIN_SUCCESS | authenticate | current user |
| Login failure | Same (catch block) | LOGIN_FAILURE | authenticate | null |
| Account delete | `AccountResource.deleteAccount(...)` | ACCOUNT_DELETE | account | current user |
| Account reset | `AccountResource.resetAccount(...)` | ACCOUNT_RESET | account | current user |

In all cases we pass request metadata (URI, method, IP, user-agent) and outcome; we never log passwords or ePHI in the description.

---

## 7. @Auditable Override

- **AuditAspect** runs around methods annotated with **@Auditable**.
- It does **not** write to the database. It sets **request attributes**: `hipaa.audit.action`, `hipaa.audit.resourceType`, `hipaa.audit.description`.
- **HipaaAuditFilter** reads these attributes when building the audit record. If they are set, they override the path-based action/resource_type and set the optional description.
- So: **global** behavior is path mapping + HTTP method; **per-method** override is @Auditable. Use @Auditable only when you need a custom action/entity/description (e.g. composite operations); the main coverage is the filter + mapping.

---

## 8. Feature Flag and Enabling Audit

- **Property:** `app.audit.hipaa.enabled` (boolean).
- **Default:** `false` in the main `application.yml` so that enabling is explicit (e.g. per profile or environment).
- **Effect:** When `false`, the filter does not write any records (path checks and mapping are skipped). Login and account-delete/reset audit are still called from the controller, but the filter will not add duplicate records for those paths if you exclude them; in practice login and account endpoints are audited from the controller.
- **Where set:** e.g. in `application.yml` under `app.audit.hipaa.enabled`, or in profile-specific files (e.g. `application-dev.yml`, `application-prod.yml`).

**Rollout:** Deploy with the flag off, run the SQL script(s) so `audit_logs` exists, then enable the flag (e.g. in staging first, then production). Monitor for audit write errors and latency.

---

## 9. AccessAuditService and Async

- **Interface:** `AccessAuditService` with a single method: `log(userId, action, resourceType, resourceId, patientId, outcome, requestUri, httpMethod, ipAddress, userAgent, description)`.
- **Implementation:** `AccessAuditServiceImpl` builds an `AuditLog` entity and saves it via `AuditLogRepository`. The method is **@Async** so the audit write does not block the request thread. If the save throws, the exception is caught and logged (fail-open); the method does not rethrow.

Any new caller that needs to write to the same audit trail (e.g. a new explicit event type) should use this service and the same field semantics (no ePHI in description, etc.).

---

## 10. What Not to Do

- **Do not** read the request or response body in the audit layer (no ePHI, no stream consumption).
- **Do not** log passwords, tokens, or ePHI in `description` or any audit field.
- **Do not** add POST-as-READ endpoints without adding them to `HipaaAuditPathMapping` with action READ.
- **Do not** rely on “last path segment” for resource_id if the path has a sub-resource (e.g. `/api/patient-profiles/123/info`); the implementation uses the **first** id-like segment (numeric or UUID).
- **Do not** fail the user request when audit write fails; keep the design fail-open.
- **Do not** add Liquibase changelogs for `audit_logs`; use the SQL scripts in `src/main/resources/sql/` only.

---

## 11. Quick Reference: Key Classes and Files

| Purpose | Class / file |
|---------|----------------------|
| Entity | `com.ht.audit.AuditLog` |
| Save one record | `AccessAuditService` / `AccessAuditServiceImpl` |
| Path → action/resource | `com.ht.audit.HipaaAuditPathMapping` |
| Filter (per-request audit) | `com.ht.audit.HipaaAuditFilter` |
| @Auditable → request attrs | `com.ht.audit.AuditAspect` |
| Login audit | `AuthenticateController.authorize` |
| Account delete/reset audit | `AccountResource.deleteAccount`, `AccountResource.resetAccount` |
| Create table | `sql/create_audit_logs_hipaa.sql` |
| Alter table | `sql/alter_audit_logs_hipaa.sql` |
| Enable/disable | `app.audit.hipaa.enabled` |

---

*Last updated for HeartThrive backend. For requirements and rationale, see `docs/HIPAA_AUDIT_IMPLEMENTATION_PLAN.md`.*
