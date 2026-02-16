# Account Delete and Reset Endpoints – Developer Reference

This document describes the **account deletion** and **account reset** API endpoints and related implementation. Use it when integrating with these APIs, debugging, or onboarding new developers.

---

## 1. Overview

| Endpoint | Purpose | User record | All other data |
|----------|---------|-------------|-----------------|
| **DELETE /api/account** | Full account deletion (e.g. GDPR/compliance) | **Deleted** | Deleted |
| **DELETE /api/account/reset** | Reset account data; user can log in again and start fresh | **Kept** | Deleted |

Both require an **authenticated** user (Bearer token). Only the **current user** can delete or reset their own account.

---

## 2. DELETE /api/account (Full Account Deletion)

### 2.1 When to use

- User requests permanent account removal (e.g. “Delete my account” in settings).
- Compliance requirements (e.g. right to erasure).
- The user will **not** be able to log in again; the identity is removed.

### 2.2 Request

| Method | Path | Auth |
|--------|------|------|
| `DELETE` | `/api/account` | Required (Bearer JWT) |

No request body. The user is identified from the JWT (current user).

### 2.3 Response

All responses use the standard `ObjectResponse` shape:

```json
{
  "success": true | false,
  "message": "string",
  "data": null
}
```

| HTTP status | Condition | `success` | `message` (example) |
|-------------|------------|------------|----------------------|
| **200 OK** | Deletion succeeded | `true` | `"Your account has been deleted successfully."` |
| **404 Not Found** | User not found (e.g. already deleted) | `false` | From exception (e.g. "User not found") |
| **500 Internal Server Error** | Unexpected error during deletion | `false` | `"Account deletion failed. Please try again or contact support."` |

If the user is not authenticated, the security layer returns **401** (no `ObjectResponse` body).

### 2.4 What gets deleted

The service deletes **all** data associated with the user, then deletes the **user** row (and thus **user_role** via JPA). Order of deletion (conceptually):

1. Email verification tokens, password reset tokens, user device tokens  
2. Doctor–patient links  
3. Reminders (deliveries then reminders)  
4. Notifications (deliveries, app statuses, notifications)  
5. Medication chain (intakes → schedule history → schedules → patient_medication) and custom medications  
6. Nutrient intakes, meal menus (with nutritions and menu items), meal logs  
7. Custom food items (food_nutrient then food_item)  
8. Heart risk metrics (history and current)  
9. Weight/height logs (history and current)  
10. Symptom track and symptoms  
11. Patient notifications and patient_audit_log  
12. Patient profile, doctor profile  
13. **User** (and user_role)

**Implementation:** `UserDeletionService.deleteUserAndAllData(Long userId)`  
**Controller:** `AccountResource.deleteAccount(HttpServletRequest request)`  
**Audit:** One HIPAA audit log with action `ACCOUNT_DELETE`, outcome SUCCESS or FAILURE.

---

## 3. DELETE /api/account/reset (Reset Account Data)

### 3.1 When to use

- User wants to “start over” but keep the same login (email/password).
- All profile, health, and activity data is removed; **user** and **user_role** are kept so the user can sign in again and go through onboarding or use the app from scratch.

### 3.2 Request

| Method | Path | Auth |
|--------|------|------|
| `DELETE` | `/api/account/reset` | Required (Bearer JWT) |

No request body. The user is identified from the JWT.

### 3.3 Response

Same `ObjectResponse` shape as delete:

| HTTP status | Condition | `success` | `message` (example) |
|-------------|------------|------------|----------------------|
| **200 OK** | Reset succeeded | `true` | `"Your account data has been reset successfully. You can continue using the app with the same login."` |
| **404 Not Found** | User not found | `false` | From exception |
| **500 Internal Server Error** | Unexpected error | `false` | `"Account reset failed. Please try again or contact support."` |

### 3.4 What gets deleted (and what is kept)

- **Deleted:** Same list as in §2.4 (steps 1–12). All tokens, reminders, notifications, medications, meals, profiles, etc., for that user.
- **Kept:** **user** and **user_role** only. No other tables are left on purpose for “reset.”

**Implementation:** `UserDeletionService.resetUserData(Long userId)`  
**Controller:** `AccountResource.resetAccount(HttpServletRequest request)`  
**Audit:** One HIPAA audit log with action `ACCOUNT_RESET`, outcome SUCCESS or FAILURE.

After a successful reset, the user has no patient_profile or doctor_profile; the app should treat them as needing onboarding or profile creation again (same as a new user with an existing login).

---

## 4. Implementation Details (for developers)

### 4.1 Where the logic lives

| Layer | Class / interface | Responsibility |
|-------|-------------------|----------------|
| REST | `AccountResource` | Resolve current user, call service, return `ObjectResponse`, write HIPAA audit (ACCOUNT_DELETE / ACCOUNT_RESET) |
| Service interface | `UserDeletionService` | `deleteUserAndAllData(Long userId)`, `resetUserData(Long userId)` |
| Service impl | `UserDeletionServiceImpl` | Private `deleteAllUserDataExceptUser(userId, email)` for steps 1–15; full delete = that + delete user; reset = that only |

### 4.2 Shared deletion logic

Both flows use the same “delete all user data except user” steps (tokens, reminders, notifications, medications, meals, profiles, etc.). Full delete **additionally** calls `userRepository.delete(user)`. So:

- **Full delete:** `deleteAllUserDataExceptUser(...)` then `userRepository.delete(user)`  
- **Reset:** `deleteAllUserDataExceptUser(...)` only  

Do not duplicate the deletion steps; keep them in one place (`deleteAllUserDataExceptUser`).

### 4.3 Constants (messages)

Defined in `Constants.java`:

- `ACCOUNT_DELETED_SUCCESS_MSG`, `ACCOUNT_DELETION_FAILED_MSG` (full delete)
- `ACCOUNT_RESET_SUCCESS_MSG`, `ACCOUNT_RESET_FAILED_MSG` (reset)

Use these in the controller so message changes are centralized.

### 4.4 Security

- Both endpoints are protected and require the same permission as other account operations (e.g. `VIEW_PERMISSION`).
- Configured in `SecurityConfiguration` (e.g. `requirePermission` for `DELETE /api/account` and `DELETE /api/account/reset`).
- Only the authenticated user can delete or reset their own account (controller uses `SecurityUtils.getCurrentUserLogin()` and `userRepository.findOneByLogin(login)`).

---

## 5. Manual SQL Reset (by user id)

For operations or support scenarios where you need to reset a user by **user id** (e.g. from DB or admin tool), use the provided SQL script. It performs the same data deletions as the reset endpoint but does **not** remove the user or user_role.

**Script location:** `src/main/resources/sql/reset_user_data_by_user_id.sql`

**Usage:**

1. Open the script and set the user id at the top:
   ```sql
   SET @user_id = 1;   -- replace with the target user's id
   ```
2. Run the script in your MySQL/MariaDB client against the correct database.

The script deletes data in the same order as the Java reset logic to respect foreign keys. It does **not** delete from `user` or `user_role`. For full account deletion (including user), use the API or implement a separate, carefully controlled process; the script is for **reset only**.

---

## 6. Quick reference

| Item | Full delete | Reset |
|------|-------------|--------|
| **Endpoint** | `DELETE /api/account` | `DELETE /api/account/reset` |
| **Service method** | `deleteUserAndAllData(userId)` | `resetUserData(userId)` |
| **User + user_role** | Removed | Kept |
| **All other user data** | Removed | Removed |
| **Audit action** | `ACCOUNT_DELETE` | `ACCOUNT_RESET` |
| **SQL script** | N/A (use API) | `reset_user_data_by_user_id.sql` (reset by user id only) |

---

*Last updated for HeartThrive backend; adjust paths or class names if your project structure differs.*
