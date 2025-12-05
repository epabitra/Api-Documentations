# Heart Thrive Notification & Reminder System Documentation

**Version:** 1.0  
**Last Updated:** 2024  
**Author:** Development Team

---

## Table of Contents

1. [System Overview](#system-overview)
2. [API Endpoints](#api-endpoints)
3. [Notification Features](#notification-features)
4. [Reminder System](#reminder-system)
5. [Scenarios & Use Cases](#scenarios--use-cases)
6. [Technical Implementation Details](#technical-implementation-details)

---

## System Overview

The Heart Thrive notification system is a comprehensive, multi-channel notification platform that handles:
- **4 Notification Features**: High Sodium Intake, Medication Reminder, Missed Medication Check, Weight Increase Notification
- **4 Delivery Channels**: Email, SMS, Push, In-App
- **Scenario-based Templates**: Weight Increase notifications support multiple scenarios (HIGH_SODIUM, MISSING_MEDICINE, BOTH, UNKNOWN)
- **Reminder System**: Automated medication reminder scheduling with RRULE support
- **Idempotency**: Prevents duplicate notifications using idempotency keys

### Architecture Flow

```
User Action/System Event
    ↓
Notification Trigger (NotificationService)
    ↓
Create Notification Entity + Trigger Record
    ↓
Get Active Channels (EMAIL, SMS, PUSH, IN_APP)
    ↓
Select Template (Feature + Channel + Scenario)
    ↓
Process Template Variables
    ↓
Create Delivery Records (one per channel)
    ↓
Schedule Delivery (async after transaction commit)
    ↓
Delivery Processor (Email/Push/SMS)
    ↓
Update Delivery Status (SENT/FAILED/RETRYING)
```

---

## API Endpoints

### Base URL
All endpoints are prefixed with `/api/notifications`

### Authentication
All endpoints require Bearer token authentication (JWT).

---

### 1. Get User Notifications

**Endpoint:** `GET /api/notifications`

**Description:** Retrieves paginated list of notifications for the current authenticated user.

**Query Parameters:**
- `includeSeen` (Boolean, optional, default: `true`) - Whether to include seen notifications
- `page` (Integer, optional, default: `0`) - Page number
- `size` (Integer, optional, default: `20`) - Page size
- `sort` (String, optional) - Sort criteria (e.g., `createdAt,desc`)

**Request Example:**
```http
GET /api/notifications?includeSeen=true&page=0&size=20&sort=createdAt,desc
Authorization: Bearer <token>
```

**Response:** `200 OK`
```json
[
  {
    "uuid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "userId": 123,
    "seen": false,
    "seenAt": null,
    "active": true,
    "createdBy": "system",
    "lastModifiedBy": "system",
    "createdAt": "2024-01-15T10:30:00Z",
    "lastModifiedAt": "2024-01-15T10:30:00Z",
    "title": "Sodium Intake Alert",
    "message": "Your sodium intake (2500 mg) has exceeded your daily limit (2000 mg). Small changes today can make a big difference for your heart health. Explore healthier meal options in Heart Thrive!"
  },
  {
    "uuid": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
    "userId": 123,
    "seen": true,
    "seenAt": "2024-01-15T11:00:00Z",
    "active": true,
    "createdBy": "system",
    "lastModifiedBy": "system",
    "createdAt": "2024-01-15T09:00:00Z",
    "lastModifiedAt": "2024-01-15T11:00:00Z",
    "title": "Medication Reminder",
    "message": "It's time for your Morning medication! Take Aspirin 100mg now to stay on track with your heart health journey. Log it in Heart Thrive when done."
  }
]
```

**Response Headers:**
- `X-Total-Count`: Total number of notifications
- `Link`: Pagination links (first, prev, next, last)

**Notes:**
- Returns only notifications for the authenticated user
- Notifications are ordered by `createdAt` descending (newest first)
- When `includeSeen=false`, only unseen notifications are returned

---

### 2. Get Notification by UUID

**Endpoint:** `GET /api/notifications/{uuid}`

**Description:** Retrieves a specific notification by its UUID for the current authenticated user.

**Path Parameters:**
- `uuid` (String, required) - Notification UUID

**Request Example:**
```http
GET /api/notifications/a1b2c3d4-e5f6-7890-abcd-ef1234567890
Authorization: Bearer <token>
```

**Response:** `200 OK`
```json
{
  "uuid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "userId": 123,
  "seen": false,
  "seenAt": null,
  "active": true,
  "createdBy": "system",
  "lastModifiedBy": "system",
  "createdAt": "2024-01-15T10:30:00Z",
  "lastModifiedAt": "2024-01-15T10:30:00Z",
  "title": "Sodium Intake Alert",
  "message": "Your sodium intake (2500 mg) has exceeded your daily limit (2000 mg). Small changes today can make a big difference for your heart health. Explore healthier meal options in Heart Thrive!"
}
```

**Error Responses:**
- `404 Not Found` - Notification not found or doesn't belong to user

---

### 3. Mark Notifications as Seen

**Endpoint:** `PUT /api/notifications/mark-seen`

**Description:** Marks multiple notifications as seen for the current authenticated user.

**Request Body:**
```json
{
  "notificationUuids": [
    "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "b2c3d4e5-f6a7-8901-bcde-f12345678901"
  ]
}
```

**Request Example:**
```http
PUT /api/notifications/mark-seen
Authorization: Bearer <token>
Content-Type: application/json

{
  "notificationUuids": [
    "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "b2c3d4e5-f6a7-8901-bcde-f12345678901"
  ]
}
```

**Response:** `200 OK` (Empty body)

**Validation:**
- `notificationUuids` must not be null or empty
- All UUIDs must belong to the authenticated user
- At least one UUID is required

**Error Responses:**
- `400 Bad Request` - Invalid request (null/empty list, invalid UUIDs, or UUIDs don't belong to user)

**Notes:**
- Sets `seen=true` and `seenAt=current timestamp` for all specified notifications
- All notifications must belong to the authenticated user

---

### 4. Mark Notifications as Unseen

**Endpoint:** `PUT /api/notifications/mark-unseen`

**Description:** Marks multiple notifications as unseen for the current authenticated user.

**Request Body:**
```json
{
  "notificationUuids": [
    "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "b2c3d4e5-f6a7-8901-bcde-f12345678901"
  ]
}
```

**Request Example:**
```http
PUT /api/notifications/mark-unseen
Authorization: Bearer <token>
Content-Type: application/json

{
  "notificationUuids": [
    "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
  ]
}
```

**Response:** `200 OK` (Empty body)

**Validation:**
- Same validation rules as mark-seen endpoint

**Notes:**
- Sets `seen=false` and `seenAt=null` for all specified notifications

---

## Notification Features

The system supports 4 notification features, each with specific triggers and template variables.

### Feature 1: HIGH_SODIUM_INTAKE

**Feature Code:** `HIGH_SODIUM_INTAKE`

**Trigger:** When user's daily sodium intake exceeds their maximum limit.

**Template Variables:**
- `user_name` - User's first name or login
- `app_name` - "Heart Thrive"
- `current_sodium` - Current sodium intake in mg (e.g., "2500")
- `max_sodium` - Maximum allowed sodium in mg (e.g., "2000")
- `excess_sodium` - Excess sodium amount in mg (e.g., "500")

**Channels:** EMAIL, SMS, PUSH, IN_APP

**Example Notification:**
```
Title: "Sodium Intake Alert"
Message: "Your sodium intake (2500 mg) has exceeded your daily limit (2000 mg). Small changes today can make a big difference for your heart health. Explore healthier meal options in Heart Thrive!"
```

**Trigger Method:**
```java
notificationService.triggerHighSodiumIntakeNotification(
    user, 
    new BigDecimal("2500"),  // currentSodiumIntake
    new BigDecimal("2000")   // maxSodiumLimit
);
```

---

### Feature 2: MEDICATION_REMINDER

**Feature Code:** `MEDICATION_REMINDER`

**Trigger:** Scheduled medication reminder (triggered by ReminderSchedulerService).

**Template Variables:**
- `user_name` - User's first name or login
- `app_name` - "Heart Thrive"
- `medicationName` - Medication name (e.g., "Aspirin")
- `doseDescription` - Dose description (e.g., "100mg")
- `timeSlot` - Time slot (Morning/Afternoon/Evening)
- `scheduledTime` - Scheduled time (HH:mm format)
- `date` - Scheduled date (YYYY-MM-DD format)

**Batched Template Variables** (when multiple medications):
- `medicationNames` - Comma-separated medication names
- `medicationCount` - Number of medications
- `medicationName` - Formatted list (e.g., "Aspirin, Metformin, and Lisinopril")

**Channels:** EMAIL, SMS, PUSH, IN_APP

**Example Notification:**
```
Title: "Medication Reminder"
Message: "It's time for your Morning medication! Take Aspirin 100mg now to stay on track with your heart health journey. Log it in Heart Thrive when done."
```

**Batched Example:**
```
Title: "Medication Reminder"
Message: "It's time for your Morning medications! Take Aspirin 100mg, Metformin 500mg, and Lisinopril 10mg now to stay on track with your heart health journey. Log them in Heart Thrive when done."
```

**Trigger Method:**
```java
Map<String, String> templateVariables = new HashMap<>();
templateVariables.put("medicationName", "Aspirin");
templateVariables.put("doseDescription", "100mg");
templateVariables.put("timeSlot", "Morning");
templateVariables.put("scheduledTime", "07:00");
templateVariables.put("date", "2024-01-15");

notificationService.triggerMedicationReminderNotification(
    user,
    templateVariables,
    LocalDate.parse("2024-01-15"),
    LocalTime.parse("07:00")
);
```

**Notes:**
- Triggered automatically by ReminderSchedulerService when reminder_delivery.runTimeUtc is reached
- Checks if medication is already taken before sending (skips if taken)
- Supports batched notifications when multiple medications are due at the same time

---

### Feature 3: MISSED_MEDICATION_CHECK

**Feature Code:** `MISSED_MEDICATION_CHECK`

**Trigger:** When medication scheduled time has passed and medication intake is not logged.

**Template Variables:**
- `user_name` - User's first name or login
- `app_name` - "Heart Thrive"
- `medicationName` - Medication name
- `doseDescription` - Dose description
- `timeSlot` - Time slot (Morning/Afternoon/Evening)
- `scheduledTime` - Scheduled time (HH:mm format)
- `date` - Scheduled date (YYYY-MM-DD format)
- `missedDuration` - Duration since scheduled time (e.g., "30 minutes")

**Batched Template Variables** (when multiple medications missed):
- Same as MEDICATION_REMINDER batched variables
- `missedDuration` - Duration since scheduled time

**Channels:** EMAIL, SMS, PUSH, IN_APP

**Example Notification:**
```
Title: "Missed Medication Alert"
Message: "We noticed Aspirin scheduled for Morning hasn't been logged yet (about 30 minutes ago). If you've already taken it, please log it now. If not, take it as soon as possible to stay on track with your heart health."
```

**Trigger Method:**
```java
Map<String, String> templateVariables = new HashMap<>();
templateVariables.put("medicationName", "Aspirin");
templateVariables.put("doseDescription", "100mg");
templateVariables.put("timeSlot", "Morning");
templateVariables.put("scheduledTime", "07:00");
templateVariables.put("date", "2024-01-15");
templateVariables.put("missedDuration", "30 minutes");

notificationService.triggerMissedMedicationNotification(
    user,
    templateVariables,
    LocalDate.parse("2024-01-15"),
    LocalTime.parse("07:00")
);
```

**Notes:**
- Triggered automatically by ReminderSchedulerService after scheduled time + check delay
- Checks PatientMedicationIntake table to verify if medication was logged
- Only triggers if medication intake is not found for the scheduled time slot
- Supports idempotency to prevent duplicate notifications

---

### Feature 4: WEIGHT_INCREASE_NOTIFICATION

**Feature Code:** `WEIGHT_INCREASE_NOTIFICATION`

**Trigger:** When user's weight increases by threshold amount (default: 2.0 lbs) compared to baseline.

**Scenarios:** This feature supports scenario-based templates:
- `HIGH_SODIUM` - Weight increase due to high sodium intake
- `MISSING_MEDICINE` - Weight increase due to missed medications
- `BOTH` - Weight increase due to both factors
- `UNKNOWN` - Weight increase with unknown cause

**Template Variables:**
- `user_name` - User's first name or login
- `app_name` - "Heart Thrive"
- `current_weight_label` - Formatted current weight (e.g., "91 kg" or "200.5 lbs")
- `baseline_weight_label` - Formatted baseline weight (e.g., "89 kg" or "196.2 lbs")
- `weight_difference_label` - Formatted weight difference (e.g., "+2 kg" or "+4.3 lbs")
- `weight_unit` - Weight unit (kg or lbs)
- `current_weight` - Current weight numeric value (no unit)
- `baseline_weight` - Baseline weight numeric value (no unit)
- `weight_difference` - Weight difference numeric value (no unit)
- `scenario_code` - Scenario code (HIGH_SODIUM, MISSING_MEDICINE, BOTH, UNKNOWN)

**Channels:** EMAIL, SMS, PUSH, IN_APP

**Example Notifications:**

**HIGH_SODIUM Scenario:**
```
Title: "Weight Change Alert"
Message: "We noticed a +2 kg change in your weight (currently 91 kg, previously 89 kg). Your recent sodium intake has been higher than recommended. Consider choosing lower-sodium meal options over the next few days—small changes can make a meaningful difference for your heart health."
```

**MISSING_MEDICINE Scenario:**
```
Title: "Weight Change Alert"
Message: "We noticed a +2 kg change in your weight (currently 91 kg, previously 89 kg). Some medication doses may have been missed or delayed recently. Consistent medication adherence helps support your heart health—please try to resume your prescribed schedule and log each dose."
```

**BOTH Scenario:**
```
Title: "Weight Change Alert"
Message: "We noticed a +2 kg change in your weight (currently 91 kg, previously 89 kg). Our system detected both higher sodium intake and some missed medication doses recently. Focusing on both areas—choosing lower-sodium meals and resuming your medication schedule—will help stabilize your weight more effectively. We're here to support you every step of the way."
```

**UNKNOWN Scenario:**
```
Title: "Weight Change Detected"
Message: "We noticed a +2 kg change in your weight (currently 91 kg, previously 89 kg). We couldn't identify a specific contributing factor from your recent logs. Please take a moment to review your meals and medications—if you have questions or concerns, your care team is here to help."
```

**Trigger Method:**
```java
notificationService.triggerWeightIncreaseNotification(
    user,
    new BigDecimal("91"),      // currentWeight (normalized to display unit)
    new BigDecimal("89"),      // baselineWeight (normalized to display unit)
    new BigDecimal("2"),       // weightDifference (normalized to display unit)
    "kg",                      // weightUnit
    "HIGH_SODIUM"              // scenarioCode
);
```

**Notes:**
- Weight threshold: 2.0 lbs (configurable via `Constants.WEIGHT_INCREASE_THRESHOLD_LBS`)
- Analysis period: Last 7 days (configurable via `Constants.WEIGHT_INCREASE_ANALYSIS_DAYS`)
- Scenario determination:
  - HIGH_SODIUM: If sodium exceeded threshold for at least 1 day in last 7 days
  - MISSING_MEDICINE: If at least 1 medication dose missed in last 7 days
  - BOTH: If both conditions are met
  - UNKNOWN: If neither condition is met or analysis fails

---

## Reminder System

The reminder system handles automated medication reminders using a scheduling-based approach.

### Architecture

```
PatientMedicationSchedule
    ↓
ReminderService.createOrUpdateReminderFromSchedule()
    ↓
Create Reminder Entity (with config_json containing RRULE)
    ↓
ReminderDeliveryGenerationService.generateDeliveriesForReminder()
    ↓
Create ReminderDelivery Records (one per occurrence)
    ↓
ReminderSchedulerService (runs every minute)
    ↓
Find Pending Deliveries (runTimeUtc <= now)
    ↓
Process by Subtype:
  - MEDICATION_REMINDER → ReminderNotificationService.sendMedicationReminder()
  - MISSED_MEDICATION_CHECK → ReminderNotificationService.checkMissedMedication()
    ↓
Trigger Notification via NotificationService
    ↓
Mark ReminderDelivery as SENT
```

### Reminder Entity Structure

**Table:** `reminder`

**Key Fields:**
- `user_id` - User who owns the reminder
- `entity_type` - "MEDICATION_SCHEDULE"
- `entity_id` - PatientMedicationSchedule ID (as string)
- `reminder_type` - "MEDICATION_REMINDER"
- `timezone` - User's timezone (IANA format, e.g., "Asia/Kolkata")
- `config_json` - JSON containing RRULE and timings
  ```json
  {
    "rrule": "FREQ=WEEKLY;BYDAY=MO,WE,FR",
    "timings": ["07:00", "19:00"],
    "additional": {}
  }
  ```
- `start_date` - When reminder schedule starts
- `end_date` - When reminder schedule ends (NULL = no end)
- `metadata` - JSON containing medication details
  ```json
  {
    "medication_name": "Aspirin",
    "dose_description": "100mg"
  }
  ```
- `status` - "active" | "paused" | "deleted" | "completed"

### ReminderDelivery Entity Structure

**Table:** `reminder_delivery`

**Key Fields:**
- `reminder_id` - Foreign key to reminder
- `subtype` - "MEDICATION_REMINDER" | "MISSED_MEDICATION_CHECK"
- `run_date` - Date for this delivery (YYYY-MM-DD)
- `time_slot` - Time slot (HH:mm format, e.g., "07:00")
- `run_time_utc` - UTC timestamp when delivery should be processed
- `status` - "PENDING" | "SENT" | "SKIPPED" | "FAILED"

### Reminder Lifecycle

1. **Creation:** When PatientMedicationSchedule is created/updated, ReminderService creates/updates Reminder
2. **Delivery Generation:** ReminderDeliveryGenerationService generates ReminderDelivery records for next N occurrences
3. **Scheduling:** ReminderSchedulerService runs every minute, finds pending deliveries, and processes them
4. **Execution:**
   - **MEDICATION_REMINDER:** Check if medication already taken → Skip if taken, else send reminder notification
   - **MISSED_MEDICATION_CHECK:** Check if medication logged → Send missed notification if not logged
5. **Completion:** Mark ReminderDelivery as SENT after successful notification

### Reminder Configuration

**RRULE Examples:**
- Daily: `FREQ=DAILY`
- Weekly (Mon, Wed, Fri): `FREQ=WEEKLY;BYDAY=MO,WE,FR`
- Every weekday: `FREQ=WEEKLY;BYDAY=MO,TU,WE,TH,FR`

**Timings:** Array of time strings in HH:mm format (e.g., `["07:00", "14:00", "19:00"]`)

**Lead Offsets:** Optional array of minutes before main reminder (e.g., `[15, 10, 5]` for advance notifications)

---

## Scenarios & Use Cases

### Scenario 1: High Sodium Intake Detection

**Trigger:** Daily meal log processing detects sodium intake exceeds limit.

**Flow:**
1. User logs meals throughout the day
2. System calculates total daily sodium intake
3. If `currentSodium > maxSodium`, trigger HIGH_SODIUM_INTAKE notification
4. Notification sent via all active channels (EMAIL, SMS, PUSH, IN_APP)

**Example:**
- User's max sodium: 2000 mg
- Current intake: 2500 mg
- System triggers notification with excess amount (500 mg)

**Handled By:**
- `HighSodiumNotificationService` (checks daily sodium)
- `NotificationService.triggerHighSodiumIntakeNotification()`

---

### Scenario 2: Medication Reminder (Single Medication)

**Trigger:** ReminderSchedulerService finds ReminderDelivery with subtype=MEDICATION_REMINDER and runTimeUtc <= now.

**Flow:**
1. ReminderSchedulerService runs every minute
2. Finds pending ReminderDelivery records
3. For each delivery:
   - Check if medication already taken (via PatientMedicationIntake)
   - If taken → Mark as SENT (skip notification)
   - If not taken → Send MEDICATION_REMINDER notification
4. Mark ReminderDelivery as SENT

**Example:**
- Medication: Aspirin 100mg
- Scheduled: 2024-01-15 07:00 (Morning)
- User hasn't logged intake yet
- System sends reminder notification

**Handled By:**
- `ReminderSchedulerService` (scheduler)
- `ReminderNotificationService.sendMedicationReminder()`
- `NotificationService.triggerMedicationReminderNotification()`

---

### Scenario 3: Medication Reminder (Batched - Multiple Medications)

**Trigger:** Multiple ReminderDelivery records have same runTimeUtc, subtype=MEDICATION_REMINDER, same userId.

**Flow:**
1. ReminderSchedulerService groups deliveries by (runTimeUtc, subtype, userId)
2. If group size > 1, process as batch
3. Collect all medication names from reminders
4. Send single batched notification with all medication names
5. Mark all ReminderDelivery records as SENT

**Example:**
- User has 3 medications scheduled for 07:00:
  - Aspirin 100mg
  - Metformin 500mg
  - Lisinopril 10mg
- System sends single notification: "Take Aspirin 100mg, Metformin 500mg, and Lisinopril 10mg"

**Handled By:**
- `ReminderSchedulerService` (batches deliveries)
- `ReminderNotificationService.sendBatchedMedicationReminder()`
- `NotificationService.triggerBatchedMedicationReminderNotification()`

---

### Scenario 4: Missed Medication Check

**Trigger:** ReminderSchedulerService finds ReminderDelivery with subtype=MISSED_MEDICATION_CHECK and runTimeUtc <= now.

**Flow:**
1. ReminderSchedulerService runs every minute
2. Finds pending ReminderDelivery records with MISSED_MEDICATION_CHECK subtype
3. For each delivery:
   - Check PatientMedicationIntake table
   - If medication logged → Mark as SENT (no notification)
   - If medication not logged → Send MISSED_MEDICATION_CHECK notification
4. Mark ReminderDelivery as SENT

**Example:**
- Medication: Aspirin 100mg
- Scheduled: 2024-01-15 07:00
- Check time: 2024-01-15 07:30 (30 minutes after scheduled time)
- User hasn't logged intake
- System sends missed medication notification

**Handled By:**
- `ReminderSchedulerService` (scheduler)
- `ReminderNotificationService.checkMissedMedication()`
- `ReminderNotificationService.sendMissedMedicationNotification()`
- `NotificationService.triggerMissedMedicationNotification()`

**Idempotency:**
- Uses idempotency key: `user_{userId}_feature_{featureId}_{date}_{time}_{timestamp}`
- Prevents duplicate notifications if scheduler runs multiple times

---

### Scenario 5: Weight Increase Detection (HIGH_SODIUM Scenario)

**Trigger:** User logs new weight entry, system detects increase >= threshold.

**Flow:**
1. User logs weight: 91 kg (previous: 89 kg)
2. System calculates difference: +2 kg (>= 2.0 lbs threshold)
3. System analyzes last 7 days:
   - Checks sodium intake (exceeded threshold for >= 1 day)
   - Checks medication adherence
4. Determines scenario: HIGH_SODIUM (sodium exceeded, medications OK)
5. Triggers WEIGHT_INCREASE_NOTIFICATION with scenario=HIGH_SODIUM
6. Uses scenario-specific template

**Example:**
- Current weight: 91 kg
- Baseline weight: 89 kg
- Difference: +2 kg
- Last 7 days: Sodium exceeded on 3 days
- Scenario: HIGH_SODIUM
- Notification: "We noticed a +2 kg change... Your recent sodium intake has been higher than recommended..."

**Handled By:**
- `WeightIncreaseAnalysisService` (analyzes causes)
- `NotificationService.triggerWeightIncreaseNotification()`

---

### Scenario 6: Weight Increase Detection (MISSING_MEDICINE Scenario)

**Flow:** Similar to Scenario 5, but analysis finds missed medications instead of high sodium.

**Example:**
- Current weight: 91 kg
- Baseline weight: 89 kg
- Difference: +2 kg
- Last 7 days: 2 medication doses missed
- Scenario: MISSING_MEDICINE
- Notification: "We noticed a +2 kg change... Some medication doses may have been missed..."

---

### Scenario 7: Weight Increase Detection (BOTH Scenario)

**Flow:** Analysis finds both high sodium and missed medications.

**Example:**
- Current weight: 91 kg
- Baseline weight: 89 kg
- Difference: +2 kg
- Last 7 days: Sodium exceeded AND medications missed
- Scenario: BOTH
- Notification: "We noticed a +2 kg change... Our system detected both higher sodium intake and some missed medication doses..."

---

### Scenario 8: Weight Increase Detection (UNKNOWN Scenario)

**Flow:** Analysis cannot determine specific cause (neither high sodium nor missed medications detected).

**Example:**
- Current weight: 91 kg
- Baseline weight: 89 kg
- Difference: +2 kg
- Last 7 days: No clear contributing factors
- Scenario: UNKNOWN
- Notification: "We noticed a +2 kg change... We couldn't identify a specific contributing factor..."

---

### Scenario 9: Medication Already Taken (Skip Reminder)

**Flow:**
1. ReminderSchedulerService finds pending MEDICATION_REMINDER delivery
2. Checks PatientMedicationIntake table
3. Finds intake record exists for scheduled date and time slot
4. Skips notification (medication already taken)
5. Marks ReminderDelivery as SENT

**Example:**
- Medication: Aspirin 100mg
- Scheduled: 2024-01-15 07:00
- User logged intake at 06:55 (before reminder)
- System skips reminder notification

**Handled By:**
- `ReminderNotificationService.sendMedicationReminder()` (checks intake before sending)

---

### Scenario 10: Duplicate Notification Prevention (Idempotency)

**Flow:**
1. System attempts to create notification
2. Checks idempotency_key uniqueness
3. If duplicate key exists:
   - For MISSED_MEDICATION_CHECK: Treat as success (notification already sent)
   - For other features: Throw exception (should not happen)

**Idempotency Key Patterns:**
- Default: `user_{userId}_feature_{featureId}_{timestamp}`
- Missed Medication: `user_{userId}_feature_{featureId}_{date}_{time}_{timestamp}`

**Handled By:**
- `NotificationServiceImpl.triggerNotification()` (generates idempotency key)
- Database unique constraint on `notification.idempotency_key`

---

## Technical Implementation Details

### Notification Channels

**Supported Channels:**
1. **EMAIL** - SendGrid integration
2. **SMS** - (Future implementation)
3. **PUSH** - Firebase Cloud Messaging (FCM)
4. **IN_APP** - Stored in NotificationAppStatus table

### Template Processing

**Template Variables:**
- Variables are replaced using `{{variable_name}}` syntax
- Processed by `TemplateService.processTemplate()`
- Variables are shared across all channels for same notification

**Template Selection:**
- Simple features (HIGH_SODIUM_INTAKE, MEDICATION_REMINDER, MISSED_MEDICATION_CHECK):
  - Select template by (feature_id, channel_id, language_code)
- Scenario-based features (WEIGHT_INCREASE_NOTIFICATION):
  - Select template by (feature_id, channel_id, scenario_id, language_code)
  - Falls back to generic template if scenario-specific template not found

### Delivery Processing

**Email Delivery:**
- Processed asynchronously after transaction commit
- Uses `EmailDeliveryService` and `SendGridEmailService`
- Rate limiting: 30 emails/second (configurable)
- Retry logic: 5 retries with exponential backoff

**Push Delivery:**
- Processed asynchronously after transaction commit
- Uses `PushNotificationProcessor` and FCM
- Rate limiting: 15 pushes/second (configurable)
- Retry logic: 3 retries with exponential backoff

**In-App Delivery:**
- Created synchronously in same transaction
- Stored in `NotificationAppStatus` table
- Available immediately via GET /api/notifications endpoint

### Database Schema

**Key Tables:**
- `feature` - Notification features (HIGH_SODIUM_INTAKE, etc.)
- `channel` - Delivery channels (EMAIL, SMS, PUSH, IN_APP)
- `feature_channel` - Active channels per feature
- `feature_scenario` - Scenarios for scenario-based features
- `template` - Notification templates (title, message)
- `feature_channel_template` - Template mapping (feature + channel + scenario)
- `notif_trigger` - Notification trigger records
- `notification` - Notification master records
- `notif_delivery` - Delivery records (one per channel)
- `notification_app_status` - In-app notification status
- `reminder` - Reminder schedules
- `reminder_delivery` - Reminder delivery records

### Error Handling

**Notification Creation Failures:**
- Logged but don't fail parent transaction
- Hibernate session cleared to prevent cascading failures
- Duplicate key violations handled gracefully (idempotency)

**Delivery Failures:**
- Marked as FAILED with error code and reason
- Retry logic with exponential backoff
- Maximum retries: 5 (email), 3 (push)

**Template Not Found:**
- Logs warning
- Skips delivery for that channel
- Continues with other channels

### Configuration

**Application Properties:**
```yaml
sendgrid:
  api-key: <key>
  from-email: epabitra@vidyayug.com
  from-name: Heart Thrive

email:
  rate-limit:
    max-per-second: 30
  retry:
    max-retries: 5
    initial-delay-minutes: 1
    backoff-multiplier: 2

notification:
  push:
    rate:
      max-per-second: 15
    retry:
      max-retries: 3
      initial-delay-seconds: 60
      backoff-multiplier: 2
```

**Constants:**
- `WEIGHT_INCREASE_THRESHOLD_LBS`: 2.0
- `WEIGHT_INCREASE_ANALYSIS_DAYS`: 7
- `SODIUM_EXCEEDED_DAYS_THRESHOLD`: 1
- `MEDICATION_MISSED_THRESHOLD`: 1
- `MISSED_MEDICATION_CHECK_MINUTES`: 30 (check delay after scheduled time)

---

## Future Enhancements

1. **SMS Channel:** Full SMS integration with provider
2. **Notification Preferences:** User-configurable channel preferences per feature
3. **Notification History:** Extended history and analytics
4. **Advanced Reminders:** Support for advance notifications (lead offsets)
5. **Multi-language Support:** Templates in multiple languages
6. **Notification Templates Management:** Admin UI for template management

---

## Notes for Developers

1. **Idempotency:** Always use idempotency keys to prevent duplicate notifications
2. **Transaction Management:** Delivery processing happens after transaction commit to avoid blocking
3. **Template Variables:** Ensure all required variables are provided for each feature
4. **Scenario Selection:** Weight increase notifications require scenario code for proper template selection
5. **Reminder Lifecycle:** Reminders are automatically created/updated when medication schedules change
6. **Batch Processing:** Multiple medications at same time are batched into single notification
7. **Error Handling:** Always handle DataIntegrityViolationException for duplicate key scenarios gracefully

---

**End of Documentation**
