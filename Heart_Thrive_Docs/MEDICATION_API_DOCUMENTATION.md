# Medication API Documentation

## Overview

This document describes eleven endpoints for managing medications in the HeartThrive application. These APIs allow users to search for medications, add medications to their profile, view their medication schedules, track daily medication intake with detailed schedule listings, record intake tracking, analyze adherence statistics, view upcoming doses, review missed medications, retrieve schedule details for editing, update existing schedules, and delete medication schedules.

---

## Table of Contents

1. [Search Medications](#1-search-medications) - *Find medications in the database*
2. [Add Medication](#2-add-medication) - *Create a medication schedule*
3. [My Medications](#3-my-medications) - *View all your active medications*
4. [Medication Schedule List](#4-medication-schedule-list) - *Get detailed daily schedule with intake status*
5. [Track Medication Intake](#5-track-medication-intake) - *Mark doses as taken/not taken*
6. [Medication Intake Statistics](#6-medication-intake-statistics) - *View adherence reports and compliance*
7. [Upcoming Medication Schedules](#7-upcoming-medication-schedules) - *See future doses and what's next*
8. [Missed Medication Schedules](#8-missed-medication-schedules) - *Review missed doses*
9. [Get Schedule for Editing](#9-get-schedule-for-editing) - *Retrieve schedule data for editing*
10. [Update Medication Schedule](#10-update-medication-schedule) - *Modify existing schedule*
11. [Delete Medication Schedule](#11-delete-medication-schedule) - *Remove/stop medication schedule*

---

## 1. Search Medications

Search for available medications in the system by name, brand, or form.

### Endpoint Details

- **URL:** `/api/medications/list`
- **Method:** `POST`
- **Authentication:** Not required (Public API)
- **Content-Type:** `application/json`

### Purpose

Search the medication database to find medications before adding them to your profile. Returns basic medication information including UUID, name, brand, strength, and form.

### Request Body

All fields are optional. If no filters provided, returns all active medications.

| Field | Type | Required | Description | Example |
|-------|------|----------|-------------|---------|
| name | String | No | Medication name (partial match) | "aspirin" |
| brand | String | No | Brand name (partial match) | "bayer" |
| form | String | No | Medication form (partial match) | "tablet" |

### Query Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| page | Integer | No | 0 | Page number (starts from 0) |
| size | Integer | No | 20 | Number of results per page |

### Response

Returns a list of medications matching the search criteria.

| Field | Type | Description |
|-------|------|-------------|
| uuid | String | Unique identifier for the medication |
| name | String | Medication name |
| brand | String | Brand/manufacturer name |
| strength | String | Medication strength (e.g., "500mg") |
| form | String | Medication form (e.g., "Tablet", "Capsule") |

### Example Request

```http
POST /api/medications/list?page=0&size=10
Content-Type: application/json

{
  "name": "aspirin",
  "brand": "bayer"
}
```

### Example Response

```json
[
  {
    "uuid": "550e8400-e29b-41d4-a716-446655440000",
    "name": "Aspirin",
    "brand": "Bayer",
    "strength": "500mg",
    "form": "Tablet"
  },
  {
    "uuid": "650e8400-e29b-41d4-a716-446655440001",
    "name": "Aspirin",
    "brand": "Bayer",
    "strength": "325mg",
    "form": "Tablet"
  }
]
```

### Response Headers

- `X-Total-Count`: Total number of matching medications
- `Link`: Pagination links (first, last, next, prev)

### Notes

- Search is case-insensitive
- Partial matches are supported (e.g., "asp" will match "Aspirin")
- Only active medications are returned
- No internal IDs are exposed for security

---

## 2. Add Medication

Add a medication to your medication schedule with dosing times and frequency.

### Endpoint Details

- **URL:** `/api/medications/add`
- **Method:** `POST`
- **Authentication:** Required (Bearer token)
- **Content-Type:** `application/json`

### Purpose

Add a medication to your profile with a detailed schedule including:
- Which days to take it
- What times of day (morning, afternoon, evening)
- Dosage information
- Whether to take before or after meals

You can either:
1. Use an existing medication (provide `medicationUuid` from search)
2. Create a custom medication (provide `medicationName`)

### Request Body

| Field | Type | Required | Description | Example |
|-------|------|----------|-------------|---------|
| medicationUuid | String | No* | UUID of existing medication | "550e8400-..." |
| medicationName | String | No* | Name for new medication | "Aspirin" |
| medicationBrand | String | No | Brand name | "Bayer" |
| startDate | Date | Yes | When to start taking | "2025-10-12" |
| endDate | Date | Yes | When to stop taking | "2025-11-12" |
| morning | Boolean | No | Take in morning? | true |
| afternoon | Boolean | No | Take in afternoon? | false |
| evening | Boolean | No | Take in evening? | true |
| morningTime | Time | No** | Morning dose time | "08:00:00" |
| afternoonTime | Time | No** | Afternoon dose time | "14:00:00" |
| eveningTime | Time | No** | Evening dose time | "20:00:00" |
| isAfterMeal | Boolean | No | Take after meal? | true |
| doseDescription | String | No | Dosage amount | "500mg" |
| dosageFrequency | String | No | How often to take | "Twice Daily" |
| daysOfWeek | Array | No | Which days to take | ["Mon", "Wed", "Fri"] |
| isAddToMyMedication | Boolean | No | Save to menu list for quick reference | true |

**Notes:**
- \* Either `medicationUuid` OR `medicationName` must be provided
- \*\* If time slot is enabled (morning/afternoon/evening = true), corresponding time should be provided
- **New Feature:** Set `isAddToMyMedication: true` to save this medication configuration to your personal "My Medication Menu List" for easy reference in the future

### Response

Returns the created medication schedule details wrapped in a success response.

| Field | Type | Description |
|-------|------|-------------|
| success | Boolean | Always true for successful requests |
| message | String | Success message |
| data | Object | Created schedule details |

The `data` object contains the full schedule information including all time slots and dates.

### Example Request 1: Add Existing Medication

```http
POST /api/medications/add
Authorization: Bearer eyJhbGciOiJIUzUxMiJ9...
Content-Type: application/json

{
  "medicationUuid": "550e8400-e29b-41d4-a716-446655440000",
  "startDate": "2025-10-12",
  "endDate": "2025-11-12",
  "morning": true,
  "afternoon": false,
  "evening": true,
  "morningTime": "08:00:00",
  "eveningTime": "20:00:00",
  "isAfterMeal": true,
  "doseDescription": "500mg",
  "dosageFrequency": "Weekly Twice",
  "daysOfWeek": ["Mon", "Wed"],
  "isAddToMyMedication": true
}
```

**Note:** Setting `isAddToMyMedication: true` will save this complete configuration to your "My Medication Menu List" for future reference.

### Example Request 2: Add Custom Medication

```http
POST /api/medications/add
Authorization: Bearer eyJhbGciOiJIUzUxMiJ9...
Content-Type: application/json

{
  "medicationUuid": null,
  "medicationName": "My Custom Supplement",
  "medicationBrand": "Generic",
  "startDate": "2025-10-12",
  "endDate": "2026-10-12",
  "morning": true,
  "afternoon": false,
  "evening": false,
  "morningTime": "09:00:00",
  "isAfterMeal": true,
  "doseDescription": "1 capsule",
  "dosageFrequency": "Daily",
  "daysOfWeek": ["Mon", "Tue", "Wed", "Thu", "Fri", "Sat", "Sun"]
}
```

### Example Response

```json
{
  "success": true,
  "message": "Medication added successfully with schedule",
  "data": {
    "id": 123,
    "uuid": "770e8400-e29b-41d4-a716-446655440002",
    "doseTime": "Morning, Evening",
    "doseDescription": "500mg",
    "intakePattern": "Twice Daily",
    "daysOfWeek": "mon,wed,fri",
    "afterMeal": true,
    "startDate": "2025-10-12",
    "endDate": "2025-11-12",
    "isMorning": true,
    "isAfterNoon": false,
    "isEvening": true,
    "morningTime": "08:00:00",
    "afternoonTime": null,
    "eveningTime": "20:00:00",
    "active": true,
    "createdBy": "user@example.com",
    "createdDate": "2025-10-12T10:30:00Z"
  }
}
```

### Error Responses

#### Missing Required Fields
```json
{
  "status": 400,
  "message": "Validation failed",
  "errors": [
    {
      "field": "startDate",
      "message": "Start date is required"
    }
  ]
}
```

#### Medication Not Found
```json
{
  "status": 400,
  "message": "Medication not found with UUID: 550e8400-..."
}
```

#### Unauthorized
```json
{
  "status": 401,
  "message": "Unauthorized"
}
```

### Notes

- If `medicationUuid` is provided, the system uses the existing medication and ignores `medicationName` and `medicationBrand` from the request
- Days of week are stored as lowercase comma-separated values (e.g., "mon,wed,fri")
- At least one time slot (morning, afternoon, or evening) should be enabled
- The schedule is specific to the authenticated user

---

## 3. My Medications

Get all your medications with their current schedules. Supports filtering by medication name, time slots, and date range.

### Endpoint Details

- **URL:** `/api/medications/my-medications`
- **Method:** `POST`
- **Authentication:** Required (Bearer token)
- **Content-Type:** `application/json`

### Purpose

Retrieve all medications you've added to your profile along with their latest schedules. You can filter the results by:
- Medication name
- Time of day (morning, afternoon, evening)
- Date range

This is useful for viewing your current medication schedule or finding specific medications.

### Request Body

All fields are optional. If no filters provided, returns all your medications.

| Field | Type | Required | Description | Example |
|-------|------|----------|-------------|---------|
| medicationName | String | No | Filter by name (partial match) | "aspirin" |
| isMorning | Boolean | No | Show only morning medications | true |
| isAfterNoon | Boolean | No | Show only afternoon medications | true |
| isEvening | Boolean | No | Show only evening medications | true |
| fromDate | Date | No | Start of date range filter | "2025-10-15" |
| toDate | Date | No | End of date range filter | "2025-10-25" |

### Response

Returns your medications with schedule details wrapped in a success response.

| Field | Type | Description |
|-------|------|-------------|
| success | Boolean | Always true for successful requests |
| message | String | Summary message with count |
| data | Array | List of medications with schedules |

### Response Data Fields

Each medication in the `data` array contains:

| Field | Type | Description |
|-------|------|-------------|
| medicationName | String | Name of the medication |
| medicationBrand | String | Brand name |
| medicationForm | String | Form (Tablet, Capsule, etc.) |
| scheduleUuid | String | Unique identifier for this schedule |
| doseDescription | String | Dosage amount (e.g., "500mg") |
| dosageFrequency | String | How often (e.g., "Weekly Twice") |
| daysOfWeek | String | Days to take (e.g., "mon,wed,fri") |
| isMorning | Boolean | Morning dose enabled? |
| isAfterNoon | Boolean | Afternoon dose enabled? |
| isEvening | Boolean | Evening dose enabled? |
| morningTime | Time | Morning dose time (HH:mm:ss) |
| afternoonTime | Time | Afternoon dose time (HH:mm:ss) |
| eveningTime | Time | Evening dose time (HH:mm:ss) |
| startDate | Date | Schedule start date |
| endDate | Date | Schedule end date |
| isAfterMeal | Boolean | Take after meal? |
| active | Boolean | Schedule is active? |

### Example Request 1: No Filters (All Medications)

```http
POST /api/medications/my-medications
Authorization: Bearer eyJhbGciOiJIUzUxMiJ9...
Content-Type: application/json

{}
```

### Example Request 2: Filter by Morning Medications

```http
POST /api/medications/my-medications
Authorization: Bearer eyJhbGciOiJIUzUxMiJ9...
Content-Type: application/json

{
  "isMorning": true
}
```

### Example Request 3: Filter by Medication Name

```http
POST /api/medications/my-medications
Authorization: Bearer eyJhbGciOiJIUzUxMiJ9...
Content-Type: application/json

{
  "medicationName": "aspirin"
}
```

### Example Request 4: Filter by Date Range

```http
POST /api/medications/my-medications
Authorization: Bearer eyJhbGciOiJIUzUxMiJ9...
Content-Type: application/json

{
  "fromDate": "2025-10-15",
  "toDate": "2025-10-25"
}
```

### Example Request 5: Multiple Filters

```http
POST /api/medications/my-medications
Authorization: Bearer eyJhbGciOiJIUzUxMiJ9...
Content-Type: application/json

{
  "medicationName": "aspirin",
  "isMorning": true,
  "fromDate": "2025-10-12",
  "toDate": "2025-11-12"
}
```

### Example Response

```json
{
  "success": true,
  "message": "Found 3 medication(s)",
  "data": [
    {
      "medicationName": "Aspirin",
      "medicationBrand": "Bayer",
      "medicationForm": "Tablet",
      "scheduleUuid": "550e8400-e29b-41d4-a716-446655440000",
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
    },
    {
      "medicationName": "Metformin",
      "medicationBrand": "Glucophage",
      "medicationForm": "Tablet",
      "scheduleUuid": "650e8400-e29b-41d4-a716-446655440001",
      "doseDescription": "850mg",
      "dosageFrequency": "Twice Daily",
      "daysOfWeek": "mon,tue,wed,thu,fri,sat,sun",
      "isMorning": true,
      "isAfterNoon": false,
      "isEvening": true,
      "morningTime": "08:30:00",
      "afternoonTime": null,
      "eveningTime": "20:30:00",
      "startDate": "2025-10-12",
      "endDate": "2026-10-12",
      "isAfterMeal": true,
      "active": true
    },
    {
      "medicationName": "Vitamin D3",
      "medicationBrand": "Nature Made",
      "medicationForm": "Capsule",
      "scheduleUuid": "750e8400-e29b-41d4-a716-446655440002",
      "doseDescription": "2000 IU",
      "dosageFrequency": "Weekly Twice",
      "daysOfWeek": "sat,sun",
      "isMorning": true,
      "isAfterNoon": false,
      "isEvening": false,
      "morningTime": "09:00:00",
      "afternoonTime": null,
      "eveningTime": null,
      "startDate": "2025-10-12",
      "endDate": "2026-04-12",
      "isAfterMeal": true,
      "active": true
    }
  ]
}
```

### Error Responses

#### Unauthorized
```json
{
  "status": 401,
  "message": "Unauthorized"
}
```

### Filter Behavior

#### Medication Name Filter
- Case-insensitive search
- Partial matches (e.g., "asp" matches "Aspirin")
- Searches in medication name field

#### Time Slot Filters (isMorning, isAfterNoon, isEvening)
- `true`: Only show medications with that time slot enabled
- `false` or `null`: Show all medications (no filter applied)
- Multiple slots can be selected together (uses AND logic)

**Example:**
- `isMorning: true` → Only medications with morning dose
- `isMorning: true, isEvening: true` → Only medications with BOTH morning AND evening doses

#### Date Range Filter (fromDate and toDate)
- Both dates must be provided for filtering to work
- Shows medications whose schedule overlaps with your filter date range
- **Overlap means:** Any part of the medication schedule falls within your specified date range

**Date Range Examples:**

Your medication schedule: **October 10, 2025 to October 30, 2025**

| Filter From | Filter To | Will Show? | Reason |
|-------------|-----------|------------|--------|
| Oct 1, 2025 | Oct 9, 2025 | ❌ No | Filter ends before schedule starts |
| Oct 15, 2025 | Oct 20, 2025 | ✅ Yes | Filter is inside schedule period |
| Oct 20, 2025 | Nov 10, 2025 | ✅ Yes | Filter overlaps end of schedule |
| Oct 31, 2025 | Nov 10, 2025 | ❌ No | Filter starts after schedule ends |
| Oct 5, 2025 | Nov 5, 2025 | ✅ Yes | Filter covers entire schedule |

### Notes

- Returns only the **most recent schedule** for each medication
- Only active schedules are included
- No internal database IDs are exposed
- Empty filter object `{}` returns all medications

---

## 4. Medication Schedule List

Get detailed medication schedules for a specific date range, expanded by date and time slot (morning, afternoon, evening). This endpoint shows what medications to take on which dates and times, including intake tracking status.

### Endpoint Details

- **URL:** `/api/medications/schedule-list`
- **Method:** `POST`
- **Authentication:** Required (Bearer token)
- **Content-Type:** `application/json`

### Purpose

Retrieve a comprehensive schedule list for your medications within a specific date range. Unlike `/my-medications` which shows medication summaries, this endpoint:

- **Expands schedules** by individual dates and time slots
- **Tracks intake status** - shows which doses were taken and which were missed
- **Respects medication patterns** - Handles both DAILY and CUSTOM (weekly) schedules
- **Supports filtering** - Filter by medication name and/or specific time slots
- **Handles timezones** - Properly converts dates from your local timezone (e.g., IST)

**Example:** If you take Aspirin twice daily (morning & evening) for 5 days, this returns 10 entries (5 days × 2 time slots).

### Request Body

|| Field | Type | Required | Description | Example |
||-------|------|----------|-------------|---------|
|| fromDate | String | Yes | Start date in dd-MM-yyyy or yyyy-MM-dd format | "10-10-2025" |
|| toDate | String | Yes | End date in dd-MM-yyyy or yyyy-MM-dd format | "15-10-2025" |
|| timezone | String | Yes | User timezone (IANA timezone) | "Asia/Kolkata" |
|| medicationName | String | No | Filter by medication name (partial match, case-insensitive) | "Aspirin" |
|| isMorning | Boolean | No | Filter for morning time slot only | true |
|| isAfternoon | Boolean | No | Filter for afternoon time slot only | true |
|| isEvening | Boolean | No | Filter for evening time slot only | true |

**Filter Behavior:**
- **Time Slots:** If NO time slot filters provided → Returns ALL time slots. If ANY filter provided → Returns ONLY matching time slots.
- **Medication Name:** Case-insensitive partial match. Searches in medication name field.

### Response Structure

|| Field | Type | Description |
||-------|------|-------------|
|| success | Boolean | Always true for successful requests |
|| message | String | Summary (e.g., "Found 18 schedule(s) for 3 medication(s)") |
|| data | Object | Schedule list response object |

### Response Data Object

|| Field | Type | Description |
||-------|------|-------------|
|| fromDate | Date | Parsed start date (YYYY-MM-DD) |
|| toDate | Date | Parsed end date (YYYY-MM-DD) |
|| timezone | String | User timezone from request |
|| totalSchedules | Integer | Total number of schedule entries |
|| totalMedications | Integer | Number of unique medications |
|| schedulesByDate | Object | Map of dates to schedule entries (null if single day) |
|| allSchedules | Array | Flat list of all schedule entries |

### Schedule Entry Object

Each entry in `allSchedules` array contains:

|| Field | Type | Description |
||-------|------|-------------|
|| date | Date | Date for this schedule entry |
|| dayOfWeek | String | Day name (e.g., "MONDAY", "FRIDAY") |
|| patientMedicationUuid | String | Unique identifier for patient medication |
|| medicationName | String | Medication name |
|| medicationBrand | String | Brand name |
|| medicationForm | String | Form (Tablet, Capsule, etc.) |
|| medicationStrength | String | Strength (e.g., "500mg") |
|| scheduleUuid | String | Schedule unique identifier |
|| doseDescription | String | Dosage amount |
|| quantity | String | Quantity (same as doseDescription) |
|| isAfterMeal | Boolean | Take after meal? |
|| intakePattern | String | Pattern (DAILY, WEEKLY_ONCE, etc.) |
|| daysOfWeek | String | Days to take (e.g., "mon,wed,fri") |
|| timeSlot | String | Time slot (MORNING, AFTERNOON, EVENING) |
|| scheduledTime | Time | Scheduled time for this slot (HH:mm:ss) |
|| isTaken | Boolean | Was this dose taken? |
|| intakeUuid | String | UUID of intake record (if taken) |

### Example Request 1: Basic Date Range (No Filters)

```http
POST /api/medications/schedule-list
Authorization: Bearer eyJhbGciOiJIUzUxMiJ9...
Content-Type: application/json

{
  "fromDate": "10-10-2025",
  "toDate": "15-10-2025",
  "timezone": "Asia/Kolkata"
}
```

**Result:** Returns ALL medications, ALL time slots for 6 days

### Example Request 2: Filter by Medication Name

```http
POST /api/medications/schedule-list
Authorization: Bearer eyJhbGciOiJIUzUxMiJ9...
Content-Type: application/json

{
  "fromDate": "10-10-2025",
  "toDate": "15-10-2025",
  "timezone": "Asia/Kolkata",
  "medicationName": "Aspirin"
}
```

**Result:** Returns only schedules for medications containing "Aspirin"

### Example Request 3: Filter by Time Slots (Morning + Evening)

```http
POST /api/medications/schedule-list
Authorization: Bearer eyJhbGciOiJIUzUxMiJ9...
Content-Type: application/json

{
  "fromDate": "10-10-2025",
  "toDate": "15-10-2025",
  "timezone": "Asia/Kolkata",
  "isMorning": true,
  "isEvening": true
}
```

**Result:** Returns ONLY morning and evening schedules (afternoon excluded)

### Example Request 4: Combined Filters

```http
POST /api/medications/schedule-list
Authorization: Bearer eyJhbGciOiJIUzUxMiJ9...
Content-Type: application/json

{
  "fromDate": "10-10-2025",
  "toDate": "15-10-2025",
  "timezone": "Asia/Kolkata",
  "medicationName": "Aspirin",
  "isMorning": true
}
```

**Result:** Returns only morning schedules for medications containing "Aspirin"

### Example Request 5: Single Day Query

```http
POST /api/medications/schedule-list
Authorization: Bearer eyJhbGciOiJIUzUxMiJ9...
Content-Type: application/json

{
  "fromDate": "10-10-2025",
  "toDate": "10-10-2025",
  "timezone": "Asia/Kolkata"
}
```

**Result:** Single day schedules with optimized response (schedulesByDate = null)

### Example Response (Multi-Day Query)

```json
{
  "success": true,
  "message": "Found 18 schedule(s) for 3 medication(s)",
  "data": {
    "fromDate": "2025-10-10",
    "toDate": "2025-10-15",
    "timezone": "Asia/Kolkata",
    "totalSchedules": 18,
    "totalMedications": 3,
    "schedulesByDate": {
      "2025-10-10": [
        {
          "date": "2025-10-10",
          "dayOfWeek": "FRIDAY",
          "patientMedicationUuid": "abc-123-uuid",
          "medicationName": "Aspirin",
          "medicationBrand": "Bayer",
          "medicationForm": "Tablet",
          "medicationStrength": "500mg",
          "scheduleUuid": "schedule-uuid-1",
          "doseDescription": "500mg",
          "quantity": "500mg",
          "isAfterMeal": true,
          "intakePattern": "DAILY",
          "daysOfWeek": "sun,mon,tue,wed,thu,fri,sat",
          "timeSlot": "MORNING",
          "scheduledTime": "08:00:00",
          "isTaken": true,
          "intakeUuid": "intake-uuid-1"
        },
        {
          "date": "2025-10-10",
          "dayOfWeek": "FRIDAY",
          "patientMedicationUuid": "abc-123-uuid",
          "medicationName": "Aspirin",
          "medicationBrand": "Bayer",
          "medicationForm": "Tablet",
          "medicationStrength": "500mg",
          "scheduleUuid": "schedule-uuid-1",
          "doseDescription": "500mg",
          "quantity": "500mg",
          "isAfterMeal": true,
          "intakePattern": "DAILY",
          "daysOfWeek": "sun,mon,tue,wed,thu,fri,sat",
          "timeSlot": "EVENING",
          "scheduledTime": "20:00:00",
          "isTaken": false,
          "intakeUuid": null
        },
        {
          "date": "2025-10-10",
          "dayOfWeek": "FRIDAY",
          "patientMedicationUuid": "def-456-uuid",
          "medicationName": "Metformin",
          "medicationBrand": "Glucophage",
          "medicationForm": "Tablet",
          "medicationStrength": "850mg",
          "scheduleUuid": "schedule-uuid-2",
          "doseDescription": "850mg",
          "quantity": "850mg",
          "isAfterMeal": true,
          "intakePattern": "DAILY",
          "daysOfWeek": "sun,mon,tue,wed,thu,fri,sat",
          "timeSlot": "MORNING",
          "scheduledTime": "08:30:00",
          "isTaken": true,
          "intakeUuid": "intake-uuid-2"
        }
      ],
      "2025-10-11": [
        // Schedules for Oct 11...
      ],
      "2025-10-12": [
        // Schedules for Oct 12...
      ]
      // ... more dates
    },
    "allSchedules": [
      // Flat list of all 18 schedule entries
    ]
  }
}
```

### Example Response (Single Day Query - Optimized)

```json
{
  "success": true,
  "message": "Found 6 schedule(s) for 3 medication(s)",
  "data": {
    "fromDate": "2025-10-10",
    "toDate": "2025-10-10",
    "timezone": "Asia/Kolkata",
    "totalSchedules": 6,
    "totalMedications": 3,
    "schedulesByDate": null,
    "allSchedules": [
      {
        "date": "2025-10-10",
        "dayOfWeek": "FRIDAY",
        "medicationName": "Aspirin",
        "timeSlot": "MORNING",
        "scheduledTime": "08:00:00",
        "isTaken": true,
        "intakeUuid": "intake-uuid-1"
      },
      {
        "date": "2025-10-10",
        "dayOfWeek": "FRIDAY",
        "medicationName": "Aspirin",
        "timeSlot": "EVENING",
        "scheduledTime": "20:00:00",
        "isTaken": false,
        "intakeUuid": null
      }
      // ... 4 more entries
    ]
  }
}
```

### Error Responses

#### Missing Required Fields
```json
{
  "status": 400,
  "message": "Validation failed",
  "errors": [
    {
      "field": "fromDate",
      "message": "fromDate is required"
    }
  ]
}
```

#### Invalid Date Format
```json
{
  "status": 400,
  "message": "Invalid date format: 2025/10/10. Expected dd-MM-yyyy or yyyy-MM-dd"
}
```

#### Invalid Timezone
```json
{
  "status": 400,
  "message": "Invalid timezone: InvalidZone"
}
```

#### Date Range Error
```json
{
  "status": 400,
  "message": "fromDate cannot be after toDate"
}
```

#### Unauthorized
```json
{
  "status": 401,
  "message": "Unauthorized"
}
```

### Schedule Generation Logic

#### DAILY Pattern
- **What it does:** Generates ALL dates between fromDate and toDate
- **Example:** Oct 10 to Oct 15 → 6 entries per time slot

#### CUSTOM Patterns (WEEKLY_ONCE, WEEKLY_TWICE, WEEKLY_THRICE)
- **What it does:** Generates ONLY dates that match the `daysOfWeek` field
- **Example:** Oct 10 to Oct 15, daysOfWeek="mon,wed,fri" → Only Mon, Wed, Fri dates

#### Date Range Overlap
The endpoint respects **both** date ranges:
1. **Request range:** Your fromDate to toDate
2. **Schedule range:** Medication's own startDate to endDate

**Example:**
- Request: Oct 10 to Oct 20
- Medication schedule: Oct 15 to Oct 30
- Result: Generates schedules for Oct 15 to Oct 20 (the overlap)

### Time Slot Expansion

Each medication schedule can have multiple time slots enabled:
- If **isMorning=true**, creates a MORNING entry for each date
- If **isAfternoon=true**, creates an AFTERNOON entry for each date
- If **isEvening=true**, creates an EVENING entry for each date

**Example:**
- Medication: Aspirin
- Schedule: Oct 10-15 (6 days), DAILY pattern
- Time slots: Morning + Evening (2 slots)
- **Result:** 6 days × 2 slots = **12 schedule entries**

### Intake Status Tracking

For each schedule entry, the system checks the `patient_medication_intake` table:

|| Scenario | isTaken | intakeUuid |
||----------|---------|------------|
|| No intake record exists | `false` | `null` |
|| Intake record exists, time slot flag = true | `true` | UUID string |
|| Intake record exists, time slot flag = false | `false` | `null` |

**Example:** You took the morning dose but not the evening dose:
- Morning entry: `isTaken: true, intakeUuid: "abc-123"`
- Evening entry: `isTaken: false, intakeUuid: null`

### Timezone Handling

This endpoint properly handles timezone conversion:

1. **Frontend sends:** Dates in local timezone (e.g., IST)
   - Example: `"fromDate": "10-10-2025"` (IST date)

2. **Backend parses:** As LocalDate (timezone-agnostic)
   - Stored as: `2025-10-10`

3. **Database stores:** LocalDate (no timezone conversion needed)
   - Dates are calendar dates, not timestamps

**Why this works:**
- We're dealing with **dates** (Oct 10), not **timestamps** (Oct 10 at 8:00 AM)
- LocalDate represents a calendar date independent of timezone
- No conversion needed because "October 10" is the same date regardless of timezone

### Filter Behavior

#### Time Slot Filters

| isMorning | isAfternoon | isEvening | Result |
|-----------|-------------|-----------|--------|
| null | null | null | **ALL** time slots returned |
| true | null | null | **ONLY** morning schedules |
| true | true | null | **ONLY** morning + afternoon |
| null | true | true | **ONLY** afternoon + evening |
| true | true | true | All three time slots |
| false | false | false | Treated as no filter (ALL) |

#### Medication Name Filter
- **Matching:** Case-insensitive, partial match
- **Examples:**
  - "asp" matches "Aspirin", "Asparagus Med"
  - "ASPIRIN" matches "Aspirin"
  - "rin" matches "Aspirin", "Loratadine"
- **Empty/Null:** No filtering applied

### Response Optimization

#### Single Day Query (fromDate == toDate)
- `schedulesByDate`: Set to `null` (not needed)
- `allSchedules`: Contains all entries
- **Benefit:** Reduces response size by ~30-40%

#### Multi-Day Query (fromDate != toDate)
- `schedulesByDate`: Populated with grouped data
- `allSchedules`: Contains same data in flat structure
- **Benefit:** Frontend can use either structure based on UI needs

### Notes

- **Performance:** Uses bulk database queries for optimal performance
- **Active schedules only:** Only returns active medication schedules
- **Latest schedule:** For each medication, uses the most recent schedule record
- **No IDs exposed:** Only UUIDs are returned, no internal database IDs
- **Empty result:** Returns empty arrays/null map if no schedules found
- **Comprehensive logging:** All filtering steps are logged for debugging

### Common Use Cases

#### Use Case 1: Today's Medication Schedule
```json
{
  "fromDate": "14-10-2025",
  "toDate": "14-10-2025",
  "timezone": "Asia/Kolkata"
}
```
**Returns:** All medications you need to take today with intake status

#### Use Case 2: This Week's Morning Medications
```json
{
  "fromDate": "14-10-2025",
  "toDate": "20-10-2025",
  "timezone": "Asia/Kolkata",
  "isMorning": true
}
```
**Returns:** Only morning schedules for the week

#### Use Case 3: Track Specific Medication
```json
{
  "fromDate": "10-10-2025",
  "toDate": "20-10-2025",
  "timezone": "Asia/Kolkata",
  "medicationName": "Aspirin"
}
```
**Returns:** All Aspirin schedules with intake tracking

#### Use Case 4: Evening Medications Not Taken
```json
{
  "fromDate": "10-10-2025",
  "toDate": "14-10-2025",
  "timezone": "Asia/Kolkata",
  "isEvening": true
}
```
**Returns:** Evening schedules (filter `isTaken: false` on frontend to see missed doses)

---

## Common Workflows

### Workflow 1: Add a Standard Medication

**Step 1:** Search for the medication
```http
POST /api/medications/list
Content-Type: application/json

{
  "name": "aspirin"
}
```

**Step 2:** Copy the UUID from results
```json
{
  "uuid": "550e8400-e29b-41d4-a716-446655440000",
  ...
}
```

**Step 3:** Add medication with your schedule
```http
POST /api/medications/add
Authorization: Bearer <token>
Content-Type: application/json

{
  "medicationUuid": "550e8400-e29b-41d4-a716-446655440000",
  "startDate": "2025-10-12",
  "endDate": "2025-11-12",
  "morning": true,
  "morningTime": "08:00:00",
  "isAfterMeal": true,
  "doseDescription": "500mg",
  "dosageFrequency": "Once Daily",
  "daysOfWeek": ["Mon", "Tue", "Wed", "Thu", "Fri"]
}
```

### Workflow 2: Add a Custom Medication

**Step 1:** Add medication directly (no search needed)
```http
POST /api/medications/add
Authorization: Bearer <token>
Content-Type: application/json

{
  "medicationUuid": null,
  "medicationName": "My Vitamin",
  "medicationBrand": "Custom Brand",
  "startDate": "2025-10-12",
  "endDate": "2026-10-12",
  "morning": true,
  "morningTime": "09:00:00",
  "isAfterMeal": true,
  "doseDescription": "1 tablet",
  "dosageFrequency": "Once Daily",
  "daysOfWeek": ["Mon", "Wed", "Fri"]
}
```

### Workflow 3: View Current Medications

**Step 1:** Get all your medications
```http
POST /api/medications/my-medications
Authorization: Bearer <token>
Content-Type: application/json

{}
```

### Workflow 4: Find Morning Medications for Next Week

**Step 1:** Filter by morning slot and date range
```http
POST /api/medications/my-medications
Authorization: Bearer <token>
Content-Type: application/json

{
  "isMorning": true,
  "fromDate": "2025-10-20",
  "toDate": "2025-10-27"
}
```

### Workflow 5: Daily Medication Tracking

**Step 1:** Get today's medication schedule
```http
POST /api/medications/schedule-list
Authorization: Bearer <token>
Content-Type: application/json

{
  "fromDate": "14-10-2025",
  "toDate": "14-10-2025",
  "timezone": "Asia/Kolkata"
}
```

**Step 2:** View the response to see what to take and when
```json
{
  "data": {
    "allSchedules": [
      {
        "medicationName": "Aspirin",
        "timeSlot": "MORNING",
        "scheduledTime": "08:00:00",
        "isTaken": false
      },
      {
        "medicationName": "Metformin",
        "timeSlot": "EVENING",
        "scheduledTime": "20:00:00",
        "isTaken": true
      }
    ]
  }
}
```

### Workflow 6: Weekly Schedule Planning

**Step 1:** Get this week's schedule with only specific time slots
```http
POST /api/medications/schedule-list
Authorization: Bearer <token>
Content-Type: application/json

{
  "fromDate": "14-10-2025",
  "toDate": "20-10-2025",
  "timezone": "Asia/Kolkata",
  "isMorning": true,
  "isEvening": true
}
```

**Result:** See all morning and evening medications for the week, grouped by date

---

## 5. Track Medication Intake

Mark a medication dose as taken or not taken for a specific date and time slot. This endpoint is used when users interact with the schedule list to track their medication adherence.

### Endpoint Details

- **URL:** `/api/medications/track-intake`
- **Method:** `POST`
- **Authentication:** Required (Bearer token)
- **Content-Type:** `application/json`

### Purpose

Record medication intake for adherence tracking. This endpoint:

- **Creates or updates** intake records intelligently
- **Tracks each time slot independently** (morning, afternoon, evening)
- **Records timestamps** when marked as taken
- **Supports toggle** - Can mark/unmark repeatedly
- **Validates ownership** - Ensures user can only update their own medications

**Typical Flow:**
1. User views today's schedule from `/schedule-list`
2. User sees "Aspirin - Morning - Not Taken"
3. User clicks "Mark as Taken"
4. Frontend calls this endpoint
5. Backend creates/updates intake record
6. Next schedule list call shows "Taken" status

### Request Body

|| Field | Type | Required | Description | Example |
||-------|------|----------|-------------|---------|
|| scheduleUuid | String | Yes | Schedule UUID from schedule-list response | "550e8400-..." |
|| date | String | Yes | Date of intake (dd-MM-yyyy or yyyy-MM-dd) | "14-10-2025" |
|| timeSlot | String | Yes | Time slot (MORNING, AFTERNOON, EVENING) | "MORNING" |
|| isTaken | Boolean | Yes | Mark as taken (true) or not taken (false) | true |
|| timezone | String | Yes | User timezone | "Asia/Kolkata" |

### Response

|| Field | Type | Description |
||-------|------|-------------|
|| success | Boolean | Always true for successful requests |
|| message | String | User-friendly message describing the action |
|| data | Object | Intake tracking response object |

### Response Data Object

|| Field | Type | Description |
||-------|------|-------------|
|| intakeUuid | String | UUID of the intake record |
|| intakeDate | Date | Date of intake |
|| scheduleUuid | String | Schedule UUID |
|| medicationName | String | Medication name |
|| medicationBrand | String | Brand name |
|| timeSlot | String | Time slot that was updated |
|| isTaken | Boolean | Current status after update |
|| takenAtTime | Timestamp | When it was marked as taken (UTC) |
|| isMorning | Boolean | Morning slot status |
|| isAfternoon | Boolean | Afternoon slot status |
|| isEvening | Boolean | Evening slot status |
|| morningTime | Timestamp | Morning taken timestamp |
|| afternoonTime | Timestamp | Afternoon taken timestamp |
|| eveningTime | Timestamp | Evening taken timestamp |
|| operation | String | "CREATED" or "UPDATED" |
|| message | String | Success message |

### Logic Explained

#### Create New Record
**When:** No intake record exists for this schedule + date

**Action:**
1. Creates new `patient_medication_intake` record
2. Sets `intakeDate` to the specified date
3. Initializes all time slot flags to `false`
4. Updates the specified time slot to true/false
5. Records timestamp if marked as taken

#### Update Existing Record
**When:** Intake record already exists for this schedule + date

**Action:**
1. Fetches existing intake record
2. Updates ONLY the specified time slot flag
3. Updates/clears the time slot timestamp
4. Other time slots remain unchanged

### Example Request 1: Mark Morning Dose as Taken (First Time)

```http
POST /api/medications/track-intake
Authorization: Bearer eyJhbGciOiJIUzUxMiJ9...
Content-Type: application/json

{
  "scheduleUuid": "550e8400-e29b-41d4-a716-446655440000",
  "date": "14-10-2025",
  "timeSlot": "MORNING",
  "isTaken": true,
  "timezone": "Asia/Kolkata"
}
```

### Example Response 1

```json
{
  "success": true,
  "message": "Created morning medication for Aspirin on 2025-10-14 - marked as taken",
  "data": {
    "intakeUuid": "intake-uuid-123",
    "intakeDate": "2025-10-14",
    "scheduleUuid": "550e8400-e29b-41d4-a716-446655440000",
    "medicationName": "Aspirin",
    "medicationBrand": "Bayer",
    "timeSlot": "MORNING",
    "isTaken": true,
    "takenAtTime": "2025-10-14T03:30:00Z",
    "isMorning": true,
    "isAfternoon": false,
    "isEvening": false,
    "morningTime": "2025-10-14T03:30:00Z",
    "afternoonTime": null,
    "eveningTime": null,
    "operation": "CREATED"
  }
}
```

### Example Request 2: Mark Evening Dose as Taken (Same Day)

```http
POST /api/medications/track-intake
Authorization: Bearer eyJhbGciOiJIUzUxMiJ9...
Content-Type: application/json

{
  "scheduleUuid": "550e8400-e29b-41d4-a716-446655440000",
  "date": "14-10-2025",
  "timeSlot": "EVENING",
  "isTaken": true,
  "timezone": "Asia/Kolkata"
}
```

### Example Response 2

```json
{
  "success": true,
  "message": "Updated evening medication for Aspirin on 2025-10-14 - marked as taken",
  "data": {
    "intakeUuid": "intake-uuid-123",
    "intakeDate": "2025-10-14",
    "scheduleUuid": "550e8400-e29b-41d4-a716-446655440000",
    "medicationName": "Aspirin",
    "medicationBrand": "Bayer",
    "timeSlot": "EVENING",
    "isTaken": true,
    "takenAtTime": "2025-10-14T14:30:00Z",
    "isMorning": true,
    "isAfternoon": false,
    "isEvening": true,
    "morningTime": "2025-10-14T03:30:00Z",
    "afternoonTime": null,
    "eveningTime": "2025-10-14T14:30:00Z",
    "operation": "UPDATED"
  }
}
```

**Note:** Same intake record updated - morning status preserved, evening added

### Example Request 3: Unmark (Mark as Not Taken)

```http
POST /api/medications/track-intake
Authorization: Bearer eyJhbGciOiJIUzUxMiJ9...
Content-Type: application/json

{
  "scheduleUuid": "550e8400-e29b-41d4-a716-446655440000",
  "date": "14-10-2025",
  "timeSlot": "MORNING",
  "isTaken": false,
  "timezone": "Asia/Kolkata"
}
```

### Example Response 3

```json
{
  "success": true,
  "message": "Updated morning medication for Aspirin on 2025-10-14 - marked as not taken",
  "data": {
    "intakeUuid": "intake-uuid-123",
    "intakeDate": "2025-10-14",
    "timeSlot": "MORNING",
    "isTaken": false,
    "takenAtTime": null,
    "isMorning": false,
    "isAfternoon": false,
    "isEvening": true,
    "morningTime": null,
    "afternoonTime": null,
    "eveningTime": "2025-10-14T14:30:00Z",
    "operation": "UPDATED"
  }
}
```

**Note:** Morning unmarked (cleared), evening status preserved

### Error Responses

#### Invalid Schedule UUID
```json
{
  "status": 400,
  "message": "Medication schedule not found with UUID: 550e8400-..."
}
```

#### Invalid Time Slot
```json
{
  "status": 400,
  "message": "Invalid time slot: MIDDAY. Must be MORNING, AFTERNOON, or EVENING"
}
```

#### Permission Denied
```json
{
  "status": 400,
  "message": "You don't have permission to update this medication intake"
}
```

#### Invalid Date Format
```json
{
  "status": 400,
  "message": "Invalid date format: 2025/10/10. Expected dd-MM-yyyy or yyyy-MM-dd"
}
```

### Independent Time Slot Tracking

Each time slot (morning, afternoon, evening) is tracked **independently**:

**Scenario:**
- Day 1 Morning: Mark as taken → `isMorning: true`
- Day 1 Evening: Mark as taken → `isEvening: true` (morning unchanged)
- Day 1 Morning: Unmark → `isMorning: false` (evening unchanged)

**Database Record:**
```
intakeDate: 2025-10-14
isMorning: false
isAfternoon: false
isEvening: true
morningTime: null
afternoonTime: null
eveningTime: 2025-10-14T14:30:00Z
```

### Timezone Handling

- **Date:** Parsed in user's timezone, stored as LocalDate
- **Timestamps:** Captured in UTC when marking as taken
- **Example:** User in IST marks taken at 8:30 AM IST → Stored as 03:00:00 UTC

### Notes

- **Idempotent:** Can call multiple times with same parameters
- **Toggle support:** Can mark/unmark repeatedly
- **Security:** Verifies schedule belongs to current user
- **Performance:** 2-3 database queries per request
- **Audit trail:** Tracks who created/modified and when

---

## 6. Medication Intake Statistics

Get comprehensive adherence statistics for a time period. Shows how well the user is following their medication schedule with counts and percentages. **Caller controls what to calculate** via filters.

### Endpoint Details

- **URL:** `/api/medications/intake-stats`
- **Method:** `POST`
- **Authentication:** Required (Bearer token)
- **Content-Type:** `application/json`

### Purpose

Calculate medication adherence statistics for any time period. This endpoint:

- **Counts scheduled doses** - Based on medication schedules and patterns
- **Counts taken doses** - From intake tracking records
- **Calculates adherence percentages** - (Taken / Scheduled) × 100
- **Supports time slot filtering** - Calculate for specific time slots only
- **Optional medication breakdown** - Per-medication statistics if requested
- **Flexible and efficient** - Frontend decides what to calculate

**Use Cases:**
- Weekly/monthly adherence reports
- Dashboard widgets showing overall compliance
- Identify problem time slots (e.g., "User forgets afternoon doses")
- Identify problem medications (e.g., "Low adherence for Medication X")
- Compliance reports for healthcare providers

### Request Body

|| Field | Type | Required | Description | Example |
||-------|------|----------|-------------|---------|
|| fromDate | String | Yes | Start date (dd-MM-yyyy or yyyy-MM-dd) | "01-10-2025" |
|| toDate | String | Yes | End date (dd-MM-yyyy or yyyy-MM-dd) | "07-10-2025" |
|| timezone | String | Yes | User timezone | "Asia/Kolkata" |
|| isMorning | Boolean | No | Include only morning slots in stats | true |
|| isAfternoon | Boolean | No | Include only afternoon slots in stats | true |
|| isEvening | Boolean | No | Include only evening slots in stats | true |
|| isMedicationBreakdownRequired | Boolean | No | Include per-medication breakdown | true |

### Filter Logic

|| Time Slot Filters | Behavior |
|||------------------|----------|
|| All null | Calculate for **ALL** time slots combined |
|| isMorning=true | Calculate for **MORNING** only |
|| isMorning=true, isEvening=true | Calculate for **MORNING + EVENING** only |
|| All true | Calculate for all slots (same as no filter) |

|| Medication Breakdown | Behavior |
|||---------------------|----------|
|| null or false | medicationBreakdown is **null** (not calculated) |
|| true | medicationBreakdown **populated** with per-med stats |

### Response

|| Field | Type | Description |
||-------|------|-------------|
|| success | Boolean | Always true for successful requests |
|| message | String | Summary with key metrics |
|| data | Object | Statistics response object |

### Response Data Object

|| Field | Type | Description |
||-------|------|-------------|
|| fromDate | Date | Start date (YYYY-MM-DD) |
|| toDate | Date | End date (YYYY-MM-DD) |
|| timezone | String | User timezone |
|| totalDays | Integer | Number of days in period |
|| includedTimeSlots | Array | Time slots included in stats |
|| totalMedications | Integer | Number of unique medications |
|| totalScheduled | Integer | Total doses scheduled |
|| totalTaken | Integer | Doses marked as taken |
|| totalNotTaken | Integer | Doses not taken (includes future + missed) |
|| totalMissed | Integer | Doses missed (time passed but not taken) |
|| overallAdherencePercentage | Decimal | Adherence % (0-100) |
|| medicationBreakdown | Array | Per-medication stats (null if not requested) |

### Medication Breakdown Object

|| Field | Type | Description |
||-------|------|-------------|
|| patientMedicationUuid | String | Patient medication UUID |
|| medicationName | String | Medication name |
|| medicationBrand | String | Brand name |
|| totalScheduled | Integer | Scheduled doses for this med |
|| totalTaken | Integer | Taken doses |
|| totalNotTaken | Integer | Not taken doses |
|| totalMissed | Integer | Missed doses for this med |
|| adherencePercentage | Decimal | Adherence % for this med |

### Statistics Calculation

```
Total Scheduled = All doses that should be taken (based on schedules)
Total Taken = Doses marked as taken in intake records
Total Not Taken = Scheduled - Taken (includes both future and missed)
Total Missed = Doses where scheduled time passed but not taken
Adherence % = (Taken / Scheduled) × 100
```

**Missed Dose Logic:**
- A dose is "missed" if:
  1. The scheduled date + time slot has already passed (compared to current time in user's timezone)
  2. AND the dose is not marked as taken in the intake table
- Default times: Morning=8:00, Afternoon=14:00, Evening=20:00

**Rounding:** Percentages rounded to 2 decimal places (e.g., 85.71%)

### Example Request 1: Overall Statistics (All Slots, No Breakdown)

```http
POST /api/medications/intake-stats
Authorization: Bearer eyJhbGciOiJIUzUxMiJ9...
Content-Type: application/json

{
  "fromDate": "01-10-2025",
  "toDate": "07-10-2025",
  "timezone": "Asia/Kolkata"
}
```

**Use Case:** Dashboard widget showing overall adherence

### Example Response 1

```json
{
  "success": true,
  "message": "Statistics for 7 day(s): 42 scheduled, 36 taken, 4 missed, 85.71% adherence",
  "data": {
    "fromDate": "2025-10-01",
    "toDate": "2025-10-07",
    "timezone": "Asia/Kolkata",
    "totalDays": 7,
    "includedTimeSlots": ["MORNING", "AFTERNOON", "EVENING"],
    "totalMedications": 3,
    "totalScheduled": 42,
    "totalTaken": 36,
    "totalNotTaken": 6,
    "totalMissed": 4,
    "overallAdherencePercentage": 85.71,
    "medicationBreakdown": null
  }
}
```

### Example Request 2: Morning Doses Only

```http
POST /api/medications/intake-stats
Authorization: Bearer eyJhbGciOiJIUzUxMiJ9...
Content-Type: application/json

{
  "fromDate": "01-10-2025",
  "toDate": "07-10-2025",
  "timezone": "Asia/Kolkata",
  "isMorning": true
}
```

**Use Case:** Check if user is good at taking morning medications

### Example Response 2

```json
{
  "success": true,
  "message": "Statistics for 7 day(s): 21 scheduled, 19 taken, 2 missed, 90.48% adherence",
  "data": {
    "fromDate": "2025-10-01",
    "toDate": "2025-10-07",
    "timezone": "Asia/Kolkata",
    "totalDays": 7,
    "includedTimeSlots": ["MORNING"],
    "totalMedications": 3,
    "totalScheduled": 21,
    "totalTaken": 19,
    "totalNotTaken": 2,
    "totalMissed": 2,
    "overallAdherencePercentage": 90.48,
    "medicationBreakdown": null
  }
}
```

### Example Request 3: With Medication Breakdown

```http
POST /api/medications/intake-stats
Authorization: Bearer eyJhbGciOiJIUzUxMiJ9...
Content-Type: application/json

{
  "fromDate": "01-10-2025",
  "toDate": "07-10-2025",
  "timezone": "Asia/Kolkata",
  "isMedicationBreakdownRequired": true
}
```

**Use Case:** Detailed report showing which medications need attention

### Example Response 3

```json
{
  "success": true,
  "message": "Statistics for 7 day(s): 42 scheduled, 36 taken, 4 missed, 85.71% adherence",
  "data": {
    "fromDate": "2025-10-01",
    "toDate": "2025-10-07",
    "timezone": "Asia/Kolkata",
    "totalDays": 7,
    "includedTimeSlots": ["MORNING", "AFTERNOON", "EVENING"],
    "totalMedications": 3,
    "totalScheduled": 42,
    "totalTaken": 36,
    "totalNotTaken": 6,
    "totalMissed": 4,
    "overallAdherencePercentage": 85.71,
    "medicationBreakdown": [
      {
        "patientMedicationUuid": "abc-123-uuid",
        "medicationName": "Aspirin",
        "medicationBrand": "Bayer",
        "totalScheduled": 14,
        "totalTaken": 13,
        "totalNotTaken": 1,
        "totalMissed": 1,
        "adherencePercentage": 92.86
      },
      {
        "patientMedicationUuid": "def-456-uuid",
        "medicationName": "Metformin",
        "medicationBrand": "Glucophage",
        "totalScheduled": 21,
        "totalTaken": 18,
        "totalNotTaken": 3,
        "totalMissed": 2,
        "adherencePercentage": 85.71
      },
      {
        "patientMedicationUuid": "ghi-789-uuid",
        "medicationName": "Vitamin D3",
        "medicationBrand": "Nature Made",
        "totalScheduled": 7,
        "totalTaken": 5,
        "totalNotTaken": 2,
        "totalMissed": 1,
        "adherencePercentage": 71.43
      }
    ]
  }
}
```

**Insight:** Vitamin D3 has lowest adherence (71.43%) - needs attention!

### Error Responses

#### Missing Required Fields
```json
{
  "status": 400,
  "message": "Validation failed",
  "errors": [
    {
      "field": "timezone",
      "message": "timezone is required"
    }
  ]
}
```

#### Invalid Date Range
```json
{
  "status": 400,
  "message": "fromDate cannot be after toDate"
}
```

#### Unauthorized
```json
{
  "status": 401,
  "message": "Unauthorized"
}
```

### How Statistics Are Calculated

#### Step-by-Step Process

1. **Get Medications:** Fetch all patient medications for user
2. **Get Schedules:** Get latest schedule for each medication
3. **Generate Schedule Entries:** Like `/schedule-list`, generate all scheduled doses
4. **Apply Filters:** If time slot filters provided, keep only those slots
5. **Match with Intake:** Check intake records for each scheduled dose
6. **Count:** Count taken vs not taken
7. **Calculate Percentages:** (Taken / Scheduled) × 100
8. **Optional Breakdown:** If requested, group by medication and calculate

#### Example Calculation

**Setup:**
- Aspirin: Morning + Evening, Daily, Oct 1-7 (7 days)
- Scheduled: 7 days × 2 slots = **14 doses**
- Taken: 13 doses
- Not Taken: 1 dose
- **Adherence: (13 / 14) × 100 = 92.86%**

### Performance

- **Efficient:** Reuses schedule generation logic
- **Optimized:** Single database query for all intake records
- **Fast:** In-memory aggregation and counting
- **Scalable:** Handles large date ranges efficiently

### Notes

- **Based on schedules:** Respects DAILY vs CUSTOM patterns
- **Date overlap:** Respects schedule start/end dates
- **Active only:** Only counts active schedules and intake records
- **Flexible filtering:** Frontend controls what to calculate
- **No unnecessary data:** Doesn't calculate breakdown unless requested
- **New in v2.0:** Added `totalMissed` field to track doses where time has passed but not taken

---

## 7. Upcoming Medication Schedules

Get future medication schedules (after current time in user's timezone).

### Endpoint Details

- **URL:** `/api/medications/upcoming-schedules`
- **Method:** `POST`
- **Authentication:** Required
- **Content-Type:** `application/json`

### Purpose

Returns only upcoming (future) medication schedules based on current time in user's timezone. Perfect for "What's Next?" dashboard widgets, reminder notifications, and weekly medication planners. Allows control over whether to show only the next immediate dose or all upcoming doses in a date range.

### Request Body

| Field | Type | Required | Description | Example |
|-------|------|----------|-------------|---------|
| fromDate | String | Yes | Start date (dd-MM-yyyy or yyyy-MM-dd) | "15-10-2025" |
| toDate | String | Yes | End date (dd-MM-yyyy or yyyy-MM-dd) | "22-10-2025" |
| timezone | String | Yes | User timezone (IANA format) | "Asia/Kolkata" |
| isAllSchedulesReq | Boolean | Yes | true=all upcoming, false=next slot only | false |
| medicationName | String | No | Filter by medication name (partial match) | "Aspirin" |
| isMorning | Boolean | No | Include only morning slots | true |
| isAfternoon | Boolean | No | Include only afternoon slots | true |
| isEvening | Boolean | No | Include only evening slots | true |

### Control Flag: isAllSchedulesReq

| Value | Behavior | Use Case |
|-------|----------|----------|
| **false** | Returns only NEXT immediate time slot | Dashboard "What's Next?" widget |
| **true** | Returns ALL upcoming schedules in range | Weekly planner, all upcoming view |

### Response Fields

| Field | Type | Description |
|-------|------|-------------|
| fromDate | LocalDate | Query start date |
| toDate | LocalDate | Query end date |
| timezone | String | User timezone |
| currentDateTime | LocalDateTime | Current date-time in user's timezone (reference) |
| totalUpcomingSchedules | Integer | Number of upcoming schedules |
| totalMedications | Integer | Number of unique medications |
| includedTimeSlots | Array[String] | Time slots included in results |
| isAllSchedulesReq | Boolean | Echo of request flag |
| nextTimeSlot | String | If isAllSchedulesReq=false, the next slot name |
| nextSlotDateTime | LocalDateTime | If isAllSchedulesReq=false, next slot date-time |
| schedulesByDate | Map | Schedules grouped by date (null if not grouped) |
| allSchedules | Array | Flat list of all upcoming schedules |

### Upcoming Logic

A schedule is "upcoming" if:
1. Scheduled date + time slot is **AFTER** current time in user's timezone
2. Default times used if scheduledTime is null:
   - Morning: 8:00 AM
   - Afternoon: 2:00 PM
   - Evening: 8:00 PM

### Example Request 1: Next Dose Only

```http
POST /api/medications/upcoming-schedules
Authorization: Bearer eyJhbGciOiJIUzUxMiJ9...
Content-Type: application/json

{
  "fromDate": "15-10-2025",
  "toDate": "22-10-2025",
  "timezone": "Asia/Kolkata",
  "isAllSchedulesReq": false
}
```

**Use Case:** Dashboard widget showing "What's Next?"

### Example Response 1

```json
{
  "status": "success",
  "message": "Next time slot: EVENING on 2025-10-15 - 3 schedule(s)",
  "data": {
    "fromDate": "2025-10-15",
    "toDate": "2025-10-22",
    "timezone": "Asia/Kolkata",
    "currentDateTime": "2025-10-15T14:30:00",
    "totalUpcomingSchedules": 3,
    "totalMedications": 3,
    "includedTimeSlots": ["MORNING", "AFTERNOON", "EVENING"],
    "isAllSchedulesReq": false,
    "nextTimeSlot": "EVENING",
    "nextSlotDateTime": "2025-10-15T20:00:00",
    "schedulesByDate": null,
    "allSchedules": [
      {
        "date": "2025-10-15",
        "dayOfWeek": "WEDNESDAY",
        "patientMedicationUuid": "abc-123-uuid",
        "medicationName": "Aspirin",
        "medicationBrand": "Bayer",
        "medicationForm": "Tablet",
        "medicationStrength": "500mg",
        "scheduleUuid": "schedule-uuid-1",
        "doseDescription": "1 tablet",
        "quantity": "1 tablet",
        "isAfterMeal": true,
        "intakePattern": "DAILY",
        "daysOfWeek": "mon,tue,wed,thu,fri,sat,sun",
        "timeSlot": "EVENING",
        "scheduledTime": "20:00:00",
        "isTaken": false,
        "intakeUuid": null
      },
      {
        "date": "2025-10-15",
        "timeSlot": "EVENING",
        "medicationName": "Metformin",
        "scheduledTime": "20:00:00",
        "isTaken": false
      },
      {
        "date": "2025-10-15",
        "timeSlot": "EVENING",
        "medicationName": "Vitamin D3",
        "scheduledTime": "20:00:00",
        "isTaken": false
      }
    ]
  }
}
```

### Example Request 2: All Upcoming This Week

```http
POST /api/medications/upcoming-schedules
Authorization: Bearer eyJhbGciOiJIUzUxMiJ9...
Content-Type: application/json

{
  "fromDate": "15-10-2025",
  "toDate": "22-10-2025",
  "timezone": "Asia/Kolkata",
  "isAllSchedulesReq": true,
  "isMorning": true
}
```

**Use Case:** Weekly planner showing all upcoming morning doses

### Example Response 2

```json
{
  "status": "success",
  "message": "Found 21 upcoming schedule(s) for 3 medication(s)",
  "data": {
    "fromDate": "2025-10-15",
    "toDate": "2025-10-22",
    "timezone": "Asia/Kolkata",
    "currentDateTime": "2025-10-15T14:30:00",
    "totalUpcomingSchedules": 21,
    "totalMedications": 3,
    "includedTimeSlots": ["MORNING"],
    "isAllSchedulesReq": true,
    "nextTimeSlot": null,
    "nextSlotDateTime": null,
    "schedulesByDate": {
      "2025-10-16": [
        {
          "date": "2025-10-16",
          "timeSlot": "MORNING",
          "medicationName": "Aspirin",
          "scheduledTime": "08:00:00",
          "isTaken": false
        }
      ],
      "2025-10-17": [ ... ]
    },
    "allSchedules": [ ... ]
  }
}
```

### Error Responses

#### Missing Required Fields
```json
{
  "status": 400,
  "message": "Validation failed",
  "errors": [
    {
      "field": "isAllSchedulesReq",
      "message": "isAllSchedulesReq is required"
    }
  ]
}
```

#### Invalid Date Range
```json
{
  "status": 400,
  "message": "fromDate cannot be after toDate"
}
```

### Use Cases

1. **Dashboard Widget:** Show next dose with countdown timer
2. **Reminder System:** Alert user about upcoming medications
3. **Weekly Planner:** View all upcoming doses for the week
4. **Morning Briefing:** "Here's what you need to take today"
5. **Medication Preparation:** Know what to prepare in advance

### Notes

- **Timezone-aware:** Uses user's timezone to determine "future"
- **Excludes past:** Even if in date range, past doses excluded
- **Sorted:** Results sorted by date, then time slot
- **Flexible:** Works with all existing filters (time slots, medication name)
- **Optimized:** Only returns what you need (next vs all)

---

## 8. Missed Medication Schedules

Get detailed list of missed medication schedules (time passed but not taken).

### Endpoint Details

- **URL:** `/api/medications/missed-schedules`
- **Method:** `POST`
- **Authentication:** Required
- **Content-Type:** `application/json`

### Purpose

Returns detailed list of medications that were missed (scheduled time has passed but not marked as taken). Unlike `/intake-stats` which returns counts, this endpoint returns the actual schedule entries with full details. Perfect for "Missed Medications" reports, adherence alerts, and identifying problem medications.

### Request Body

| Field | Type | Required | Description | Example |
|-------|------|----------|-------------|---------|
| fromDate | String | Yes | Start date (dd-MM-yyyy or yyyy-MM-dd) | "01-10-2025" |
| toDate | String | Yes | End date (dd-MM-yyyy or yyyy-MM-dd) | "14-10-2025" |
| timezone | String | Yes | User timezone (IANA format) | "Asia/Kolkata" |
| medicationName | String | No | Filter by medication name (partial match) | "Aspirin" |
| isMorning | Boolean | No | Include only morning slots | true |
| isAfternoon | Boolean | No | Include only afternoon slots | true |
| isEvening | Boolean | No | Include only evening slots | true |

### Response Fields

| Field | Type | Description |
|-------|------|-------------|
| fromDate | LocalDate | Query start date |
| toDate | LocalDate | Query end date |
| timezone | String | User timezone |
| currentDateTime | LocalDateTime | Current date-time in user's timezone |
| totalMissedSchedules | Integer | Total number of missed schedules |
| totalMedications | Integer | Number of unique medications with missed doses |
| includedTimeSlots | Array[String] | Time slots included in results |
| mostMissedMedication | String | Name of most frequently missed medication |
| mostMissedCount | Integer | Number of times most missed med was missed |
| missedSchedulesByDate | Map | Missed schedules grouped by date |
| allMissedSchedules | Array | Flat list of all missed schedules |

### Missed Dose Logic

A dose is "missed" if:
1. Scheduled date + time slot has **ALREADY PASSED** (compared to current time)
2. AND dose is **NOT marked as taken** in intake table

Default times if scheduledTime is null:
- Morning: 8:00 AM
- Afternoon: 2:00 PM
- Evening: 8:00 PM

### Example Request 1: All Missed Doses

```http
POST /api/medications/missed-schedules
Authorization: Bearer eyJhbGciOiJIUzUxMiJ9...
Content-Type: application/json

{
  "fromDate": "01-10-2025",
  "toDate": "14-10-2025",
  "timezone": "Asia/Kolkata"
}
```

**Use Case:** "Missed Medications" report page

### Example Response 1

```json
{
  "status": "success",
  "message": "Found 8 missed schedule(s) for 3 medication(s) - Most missed: Aspirin (4 times)",
  "data": {
    "fromDate": "2025-10-01",
    "toDate": "2025-10-14",
    "timezone": "Asia/Kolkata",
    "currentDateTime": "2025-10-15T14:30:00",
    "totalMissedSchedules": 8,
    "totalMedications": 3,
    "includedTimeSlots": ["MORNING", "AFTERNOON", "EVENING"],
    "mostMissedMedication": "Aspirin",
    "mostMissedCount": 4,
    "missedSchedulesByDate": {
      "2025-10-10": [
        {
          "date": "2025-10-10",
          "dayOfWeek": "FRIDAY",
          "patientMedicationUuid": "abc-123-uuid",
          "medicationName": "Aspirin",
          "medicationBrand": "Bayer",
          "medicationForm": "Tablet",
          "medicationStrength": "500mg",
          "scheduleUuid": "schedule-uuid-1",
          "doseDescription": "1 tablet",
          "timeSlot": "MORNING",
          "scheduledTime": "08:00:00",
          "isTaken": false,
          "intakeUuid": null
        },
        {
          "date": "2025-10-10",
          "timeSlot": "EVENING",
          "medicationName": "Aspirin",
          "scheduledTime": "20:00:00",
          "isTaken": false
        }
      ],
      "2025-10-11": [
        {
          "date": "2025-10-11",
          "timeSlot": "MORNING",
          "medicationName": "Metformin",
          "scheduledTime": "08:00:00",
          "isTaken": false
        }
      ]
    },
    "allMissedSchedules": [
      { "date": "2025-10-10", "timeSlot": "MORNING", "medicationName": "Aspirin", ... },
      { "date": "2025-10-10", "timeSlot": "EVENING", "medicationName": "Aspirin", ... },
      { "date": "2025-10-11", "timeSlot": "MORNING", "medicationName": "Metformin", ... },
      { "date": "2025-10-12", "timeSlot": "MORNING", "medicationName": "Vitamin D3", ... },
      { "date": "2025-10-13", "timeSlot": "MORNING", "medicationName": "Aspirin", ... },
      { "date": "2025-10-13", "timeSlot": "EVENING", "medicationName": "Aspirin", ... },
      { "date": "2025-10-14", "timeSlot": "AFTERNOON", "medicationName": "Metformin", ... },
      { "date": "2025-10-14", "timeSlot": "EVENING", "medicationName": "Vitamin D3", ... }
    ]
  }
}
```

### Example Request 2: Morning Missed Only

```http
POST /api/medications/missed-schedules
Authorization: Bearer eyJhbGciOiJIUzUxMiJ9...
Content-Type: application/json

{
  "fromDate": "01-10-2025",
  "toDate": "14-10-2025",
  "timezone": "Asia/Kolkata",
  "isMorning": true
}
```

**Use Case:** Identify morning medication adherence issues

### Example Response 2

```json
{
  "status": "success",
  "message": "Found 3 missed schedule(s) for 2 medication(s) - Most missed: Aspirin (2 times)",
  "data": {
    "fromDate": "2025-10-01",
    "toDate": "2025-10-14",
    "timezone": "Asia/Kolkata",
    "currentDateTime": "2025-10-15T14:30:00",
    "totalMissedSchedules": 3,
    "totalMedications": 2,
    "includedTimeSlots": ["MORNING"],
    "mostMissedMedication": "Aspirin",
    "mostMissedCount": 2,
    "allMissedSchedules": [
      {
        "date": "2025-10-10",
        "timeSlot": "MORNING",
        "medicationName": "Aspirin",
        "scheduledTime": "08:00:00",
        "isTaken": false
      },
      {
        "date": "2025-10-11",
        "timeSlot": "MORNING",
        "medicationName": "Metformin",
        "scheduledTime": "08:00:00",
        "isTaken": false
      },
      {
        "date": "2025-10-13",
        "timeSlot": "MORNING",
        "medicationName": "Aspirin",
        "scheduledTime": "08:00:00",
        "isTaken": false
      }
    ]
  }
}
```

### Error Responses

#### Missing Required Fields
```json
{
  "status": 400,
  "message": "Validation failed",
  "errors": [
    {
      "field": "timezone",
      "message": "timezone is required"
    }
  ]
}
```

#### Invalid Timezone
```json
{
  "status": 400,
  "message": "Invalid timezone: Invalid/Timezone"
}
```

### Use Cases

1. **Missed Medications Report:** Show user what they missed
2. **Adherence Alerts:** Notify about recent missed doses
3. **Pattern Analysis:** Identify which medications are frequently missed
4. **Intervention Planning:** Focus on problematic medications
5. **Historical Review:** Analyze past adherence issues

### Comparison: Missed Schedules vs Intake Stats

| Feature | /missed-schedules | /intake-stats |
|---------|------------------|---------------|
| **Returns** | Detailed schedule entries | Statistics/counts |
| **Use Case** | See WHAT was missed | See HOW MANY missed |
| **Details** | Full schedule info | Aggregated numbers |
| **Summary** | Most missed medication | Overall adherence % |
| **Best For** | Reports, alerts | Dashboard widgets |

### Notes

- **Reuses logic:** Uses same `isMissedDose()` method as `/intake-stats`
- **Timezone-aware:** Accurately determines if time has passed
- **Sorted:** Results sorted by date, then time slot
- **Flexible filtering:** Supports all time slot and medication filters
- **Summary included:** Automatically identifies most problematic medication

---

## 9. Get Schedule for Editing

Retrieve existing schedule details to pre-fill the edit form. Returns all current schedule values along with medication information for display.

### Endpoint Details

- **URL:** `/api/medications/schedule/{scheduleUuid}`
- **Method:** `GET`
- **Authentication:** Required (Bearer token)
- **Content-Type:** `application/json`

### Purpose

This endpoint retrieves the complete details of an existing medication schedule, used to populate the edit form in the frontend. It returns:
- **Read-only medication info** (name, brand, form, strength) for display
- **All editable schedule fields** with current values for form pre-filling
- **Secure access** - Only the schedule owner can retrieve it

**Typical Flow:**
1. User views their schedule list
2. User clicks "Edit" button on a specific schedule
3. Frontend calls this endpoint with the scheduleUuid
4. Response data pre-fills the edit form
5. User modifies desired fields
6. Frontend calls PUT `/edit-schedule` to save changes

### Path Parameters

|| Parameter | Type | Required | Description | Example |
||-----------|------|----------|-------------|---------|
|| scheduleUuid | String | Yes | UUID of the schedule to retrieve | "550e8400-e29b-41d4-a716-446655440000" |

### Response

Returns schedule details wrapped in a success response.

|| Field | Type | Description |
||-------|------|-------------|
|| success | Boolean | Always true for successful requests |
|| message | String | Success message |
|| data | Object | Schedule details object |

### Response Data Object

|| Field | Type | Editable | Description |
||-------|------|----------|-------------|
|| scheduleUuid | String | No | Schedule UUID (reference) |
|| medicationName | String | No | Medication name (display only) |
|| medicationBrand | String | No | Medication brand (display only) |
|| medicationForm | String | No | Medication form (display only) |
|| medicationStrength | String | No | Medication strength (display only) |
|| startDate | Date | Yes | Current schedule start date |
|| endDate | Date | Yes | Current schedule end date |
|| morning | Boolean | Yes | Morning dose enabled |
|| afternoon | Boolean | Yes | Afternoon dose enabled |
|| evening | Boolean | Yes | Evening dose enabled |
|| morningTime | Time | Yes | Time for morning dose |
|| afternoonTime | Time | Yes | Time for afternoon dose |
|| eveningTime | Time | Yes | Time for evening dose |
|| isAfterMeal | Boolean | Yes | Take after meal |
|| doseDescription | String | Yes | Dosage amount |
|| dosageFrequency | String | Yes | How often to take |
|| daysOfWeek | Array | Yes | Days to take medication |
|| active | Boolean | No | Schedule is active |

### Example Request

```http
GET /api/medications/schedule/550e8400-e29b-41d4-a716-446655440000
Authorization: Bearer eyJhbGciOiJIUzUxMiJ9...
```

### Example Response

```json
{
  "success": true,
  "message": "Schedule retrieved successfully",
  "data": {
    "scheduleUuid": "550e8400-e29b-41d4-a716-446655440000",
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

### Error Responses

#### Schedule Not Found
```json
{
  "status": 400,
  "message": "Medication schedule not found with UUID: 550e8400-..."
}
```

#### Permission Denied
```json
{
  "status": 400,
  "message": "You don't have permission to access this medication schedule"
}
```

#### Unauthorized
```json
{
  "status": 401,
  "message": "Unauthorized"
}
```

### Response Features

#### Days of Week Conversion
- **Database format:** "mon,wed,fri" (lowercase, comma-separated)
- **API response format:** ["Mon", "Wed", "Fri"] (capitalized array)
- **Benefit:** Ready to use in frontend multi-select controls

#### Null Handling
- Fields that aren't set return `null` (e.g., `afternoonTime: null` if afternoon slot disabled)
- Frontend can use null checks to conditionally render fields

#### Security
- Verifies schedule belongs to current authenticated user
- Returns 400 error if user tries to access someone else's schedule
- Only active schedules can be retrieved

### Notes

- **Read-only medication fields:** Name, brand, form, and strength cannot be edited (they're from the medication table)
- **Smart conversion:** daysOfWeek automatically converted from DB format to user-friendly list
- **Form-ready:** Response structure matches the expected edit form structure
- **UUID-based:** Uses schedule UUID, no internal IDs exposed
- **Efficient:** Single database query with joins

---

## 10. Update Medication Schedule

Update an existing medication schedule. Allows users to modify schedule details like time period, frequency, time slots, etc., without changing the medication itself.

### Endpoint Details

- **URL:** `/api/medications/edit-schedule`
- **Method:** `PUT`
- **Authentication:** Required (Bearer token)
- **Content-Type:** `application/json`

### Purpose

This endpoint allows users to edit their existing medication schedules. Users can update:
- **Time period** (extend or reduce medication duration)
- **Days of week** (change from daily to specific days or vice versa)
- **Time slots** (add/remove morning, afternoon, evening doses)
- **Meal timing** (change before/after meal preference)
- **Dosage details** (update frequency or description)

**What CANNOT be updated:**
- Medication name (this is in the medication table)
- Medication brand (this is in the medication table)
- To change the medication, user should create a new schedule

**Typical Flow:**
1. User retrieves schedule details via GET `/schedule/{scheduleUuid}`
2. Form is pre-filled with current values
3. User modifies desired fields
4. Frontend calls this endpoint with updated values
5. Backend updates the schedule and returns confirmation

### Request Body

|| Field | Type | Required | Description | Example |
||-------|------|----------|-------------|---------|
|| scheduleUuid | String | Yes | UUID of schedule to update | "550e8400-..." |
|| startDate | Date | Yes | Updated start date | "2025-10-15" |
|| endDate | Date | Yes | Updated end date | "2025-12-31" |
|| morning | Boolean | No | Enable morning dose | true |
|| afternoon | Boolean | No | Enable afternoon dose | false |
|| evening | Boolean | No | Enable evening dose | true |
|| morningTime | Time | No | Morning dose time | "08:00:00" |
|| afternoonTime | Time | No | Afternoon dose time | "14:00:00" |
|| eveningTime | Time | No | Evening dose time | "20:00:00" |
|| isAfterMeal | Boolean | No | Take after meal | true |
|| doseDescription | String | No | Dosage amount | "500 mg" |
|| dosageFrequency | String | No | How often to take | "Twice Daily" |
|| daysOfWeek | Array | No | Days to take | ["Mon", "Wed", "Fri"] |

### Response

Returns updated schedule details wrapped in a success response.

|| Field | Type | Description |
||-------|------|-------------|
|| success | Boolean | Always true for successful requests |
|| message | String | Success message |
|| data | Object | Updated schedule response object |

### Response Data Object

|| Field | Type | Description |
||-------|------|-------------|
|| scheduleUuid | String | Schedule UUID |
|| medicationName | String | Medication name (unchanged) |
|| medicationBrand | String | Brand name (unchanged) |
|| startDate | Date | Updated start date |
|| endDate | Date | Updated end date |
|| doseDescription | String | Updated dose description |
|| dosageFrequency | String | Updated dosage frequency |
|| daysOfWeek | String | Updated days (comma-separated) |
|| isAfterMeal | Boolean | Updated meal timing |
|| isMorning | Boolean | Updated morning flag |
|| isAfterNoon | Boolean | Updated afternoon flag |
|| isEvening | Boolean | Updated evening flag |
|| morningTime | Time | Updated morning time |
|| afternoonTime | Time | Updated afternoon time |
|| eveningTime | Time | Updated evening time |
|| lastModifiedBy | String | Who updated the schedule |
|| lastModifiedDate | Timestamp | When it was updated (UTC) |
|| message | String | Success message |

### Example Request 1: Extend Medication Period

```http
PUT /api/medications/edit-schedule
Authorization: Bearer eyJhbGciOiJIUzUxMiJ9...
Content-Type: application/json

{
  "scheduleUuid": "550e8400-e29b-41d4-a716-446655440000",
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

### Example Response 1

```json
{
  "success": true,
  "message": "Medication schedule updated successfully",
  "data": {
    "scheduleUuid": "550e8400-e29b-41d4-a716-446655440000",
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

### Example Request 2: Change to Daily and Add Afternoon Dose

```http
PUT /api/medications/edit-schedule
Authorization: Bearer eyJhbGciOiJIUzUxMiJ9...
Content-Type: application/json

{
  "scheduleUuid": "550e8400-e29b-41d4-a716-446655440000",
  "startDate": "2025-10-15",
  "endDate": "2025-12-31",
  "morning": true,
  "afternoon": true,
  "evening": true,
  "morningTime": "08:00:00",
  "afternoonTime": "14:00:00",
  "eveningTime": "20:00:00",
  "isAfterMeal": true,
  "doseDescription": "500 mg",
  "dosageFrequency": "Three Times Daily",
  "daysOfWeek": ["Mon", "Tue", "Wed", "Thu", "Fri", "Sat", "Sun"]
}
```

### Example Response 2

```json
{
  "success": true,
  "message": "Medication schedule updated successfully",
  "data": {
    "scheduleUuid": "550e8400-e29b-41d4-a716-446655440000",
    "medicationName": "Aspirin",
    "medicationBrand": "Bayer",
    "startDate": "2025-10-15",
    "endDate": "2025-12-31",
    "doseDescription": "500 mg",
    "dosageFrequency": "Three Times Daily",
    "daysOfWeek": "mon,tue,wed,thu,fri,sat,sun",
    "isAfterMeal": true,
    "isMorning": true,
    "isAfterNoon": true,
    "isEvening": true,
    "morningTime": "08:00:00",
    "afternoonTime": "14:00:00",
    "eveningTime": "20:00:00",
    "lastModifiedBy": "user@example.com",
    "lastModifiedDate": "2025-10-15T10:35:00Z",
    "message": "Successfully updated medication schedule for Aspirin"
  }
}
```

### Error Responses

#### Schedule Not Found
```json
{
  "status": 400,
  "message": "Medication schedule not found with UUID: 550e8400-..."
}
```

#### Permission Denied
```json
{
  "status": 400,
  "message": "You don't have permission to update this medication schedule"
}
```

#### Invalid Date Range
```json
{
  "status": 400,
  "message": "Start date cannot be after end date"
}
```

#### Validation Error
```json
{
  "status": 400,
  "message": "Validation failed",
  "errors": [
    {
      "field": "scheduleUuid",
      "message": "Schedule UUID is required"
    }
  ]
}
```

#### Unauthorized
```json
{
  "status": 401,
  "message": "Unauthorized"
}
```

### Update Logic

#### What Gets Updated
All fields in the `patient_medication_schedule` table:
- Date range (startDate, endDate)
- Time slots and times (isMorning, morningTime, etc.)
- Intake details (doseDescription, intakePattern, daysOfWeek, afterMeal)
- Audit fields (lastModifiedBy, lastModifiedDate)

#### What Stays the Same
- Schedule UUID (never changes)
- Patient medication relationship (stays linked to same medication)
- Created audit fields (createdBy, createdDate)
- Medication details (name, brand in medication table)

#### Validation Rules
1. **Schedule ownership:** User must own the schedule
2. **Date range:** startDate must be ≤ endDate
3. **Schedule exists:** Must be an active schedule
4. **Required fields:** scheduleUuid, startDate, endDate are mandatory

### Days of Week Conversion

**Request format (from frontend):**
```json
"daysOfWeek": ["Mon", "Wed", "Fri"]
```

**Stored format (in database):**
```
"mon,wed,fri"
```

**Response format (to frontend):**
```json
"daysOfWeek": "mon,wed,fri"
```

### Use Cases

#### Use Case 1: Extend Medication Period
**Scenario:** Doctor extends prescription from 3 months to 6 months
```json
{
  "scheduleUuid": "...",
  "endDate": "2026-06-30"  // Changed from 2025-12-31
  // ... other fields unchanged
}
```

#### Use Case 2: Change Days of Week
**Scenario:** Change from daily to every other day
```json
{
  "scheduleUuid": "...",
  "daysOfWeek": ["Mon", "Wed", "Fri", "Sun"]  // Changed from all days
  // ... other fields unchanged
}
```

#### Use Case 3: Add Afternoon Dose
**Scenario:** Doctor increases frequency from twice to three times daily
```json
{
  "scheduleUuid": "...",
  "afternoon": true,  // Changed from false
  "afternoonTime": "14:00:00",  // Added
  "dosageFrequency": "Three Times Daily"  // Changed from "Twice Daily"
  // ... other fields unchanged
}
```

#### Use Case 4: Change Meal Timing
**Scenario:** Change from after meal to before meal
```json
{
  "scheduleUuid": "...",
  "isAfterMeal": false  // Changed from true
  // ... other fields unchanged
}
```

### Security Features

1. **User verification:** Checks that schedule belongs to current authenticated user
2. **Permission validation:** Returns error if user tries to update someone else's schedule
3. **Active schedule check:** Only active schedules can be updated
4. **Audit trail:** Records who updated and when

### Notes

- **Immutable medication:** Cannot change medication name/brand through this endpoint
- **Complete update:** All editable fields must be provided (partial updates not supported)
- **Audit preserved:** Original creation info (createdBy, createdDate) remains unchanged
- **Timezone-agnostic dates:** Dates stored as LocalDate (no timezone conversion)
- **Efficient:** Updates in single database transaction
- **UUID-based:** No internal IDs exposed in request or response

---

## 11. Delete Medication Schedule

Delete (soft delete) an existing medication schedule. Sets the schedule to inactive so it no longer appears in schedule lists or tracking views.

### Endpoint Details

- **URL:** `/api/medications/schedule/{scheduleUuid}`
- **Method:** `DELETE`
- **Authentication:** Required (Bearer token)

### Purpose

This endpoint allows users to remove medication schedules from their active list. It performs a **soft delete** (sets `active = false`) rather than permanently removing the data from the database.

**Important Notes:**

- Deletion is **always allowed** regardless of intake history
- Performs **soft delete only** (data is preserved in database)
- Schedule becomes inactive and no longer appears in active lists
- Intake records (if any) are **preserved** and not deleted
- Data remains available for audit and historical reporting

**Common Use Cases:**
- Remove schedules added by mistake
- Stop medications that are no longer needed
- Clean up old or unused schedules
- End medication regimen early

### Path Parameters

| Parameter | Type | Required | Description | Example |
|-----------|------|----------|-------------|---------|
| scheduleUuid | String | Yes | UUID of the schedule to delete | "550e8400-e29b-41d4-a716-446655440000" |

### Response

Returns deletion confirmation wrapped in a success response.

| Field | Type | Description |
|-------|------|-------------|
| success | Boolean | Always true for successful requests |
| message | String | Success message |
| data | Object | Deletion response object |

### Response Data Object

| Field | Type | Description |
|-------|------|-------------|
| scheduleUuid | String | UUID of deleted schedule |
| medicationName | String | Name of the medication |
| medicationBrand | String | Brand name |
| deleted | Boolean | Deletion status (true) |
| deletedBy | String | User who deleted the schedule |
| deletedAt | Timestamp | When it was deleted (UTC) |
| message | String | Detailed deletion message |

### Example Request 1: Successful Deletion

```http
DELETE /api/medications/schedule/550e8400-e29b-41d4-a716-446655440000
Authorization: Bearer eyJhbGciOiJIUzUxMiJ9...
```

### Example Response 1: Success

```json
{
  "success": true,
  "message": "Medication schedule deleted successfully",
  "data": {
    "scheduleUuid": "550e8400-e29b-41d4-a716-446655440000",
    "medicationName": "Aspirin",
    "medicationBrand": "Bayer",
    "deleted": true,
    "deletedBy": "user@example.com",
    "deletedAt": "2025-10-16T10:30:00Z",
    "message": "Medication schedule for 'Aspirin' has been successfully deleted"
  }
}
```

### Error Responses

#### Schedule Not Found

```json
{
  "status": 400,
  "message": "Medication schedule not found with UUID: 550e8400-..."
}
```

#### Permission Denied

```json
{
  "status": 400,
  "message": "You don't have permission to delete this medication schedule"
}
```

**Explanation:** The schedule belongs to another user. Only the owner can delete their schedules.

#### Unauthorized

```json
{
  "status": 401,
  "message": "Unauthorized"
}
```

### Deletion Logic

#### Soft Delete vs Hard Delete

**Soft Delete (What This Endpoint Does):**
- Sets `active = false` in the database
- Schedule no longer appears in `/schedule-list` or `/my-medications`
- Data remains in database for audit purposes
- Can potentially be restored by administrator if needed

**Hard Delete (NOT Supported):**
- Permanently removes data from database
- Not implemented to preserve data integrity
- Would break referential integrity if intake records exist

#### Permission Check

1. **Verify schedule exists:** Check by scheduleUuid
2. **Verify ownership:** Schedule must belong to current user
3. **Proceed with soft delete:** Sets `active = false`

#### Audit Trail

When deletion succeeds:
- `active` field set to `false`
- `lastModifiedBy` set to current user
- `lastModifiedDate` set to current timestamp (UTC)
- Response includes `deletedBy` and `deletedAt` for tracking

### Use Cases

#### Use Case 1: Remove Schedule Added by Mistake

**Scenario:** User accidentally added "Aspirin" instead of "Ibuprofen"

```http
DELETE /api/medications/schedule/abc-123-def-456
```

**Result:** ✅ Deleted successfully

**Action:** User can now add the correct medication

---

#### Use Case 2: Stop Active Medication

**Scenario:** User has been taking "Metformin" daily but doctor discontinued it

```http
DELETE /api/medications/schedule/def-456-ghi-789
```

**Result:** ✅ Deleted successfully

**What happens:**
- Schedule becomes inactive (active = false)
- No longer appears in `/my-medications` or `/schedule-list`
- Past intake records are **preserved** for medical history
- Future doses will not be scheduled

---

#### Use Case 3: Clean Up Unused Future Schedule

**Scenario:** User scheduled a medication to start next month but doctor changed the plan

```http
DELETE /api/medications/schedule/xyz-789-ghi-012
```

**Result:** ✅ Deleted successfully

**Action:** Schedule removed from upcoming list

---

#### Use Case 4: End Medication Early

**Scenario:** User's prescription was for 6 months but needs to stop after 2 months

```http
DELETE /api/medications/schedule/ghi-789-jkl-012
```

**Result:** ✅ Deleted successfully

**Benefit:** Immediate removal from active schedules, history preserved

### Security Features

1. **User verification:** Checks that schedule belongs to current authenticated user
2. **Permission validation:** Returns error if user tries to delete someone else's schedule
3. **Active schedule check:** Only active schedules can be deleted
4. **Audit trail:** Records who deleted and when for accountability

### Notes

- **Soft delete only:** Data is never permanently removed
- **Deletion always allowed:** No restrictions based on intake history
- **No cascade delete:** Intake records are NOT deleted (intentionally preserved)
- **Irreversible via API:** Once deleted, schedule cannot be restored via API (admin may restore)
- **Security:** Only schedule owner can delete
- **Efficient:** Single database transaction
- **UUID-based:** No internal IDs exposed

### Frequently Asked Questions

**Q: What happens to my intake records when I delete a schedule?**

A: Intake records are **preserved**. The schedule is soft-deleted (active=false), but all intake history remains in the database for medical record purposes.

**Q: Can I restore a deleted schedule?**

A: Not through the API. The schedule is soft-deleted (active=false). A system administrator may be able to restore it from the database if absolutely necessary.

**Q: Should I use DELETE or UPDATE endDate to stop a medication?**

A: Both work! DELETE is quicker for immediate removal. UPDATE with endDate gives you control over when the schedule ends. Choose based on your preference:
- **DELETE:** Immediate removal, simpler
- **UPDATE endDate:** More control, can set specific end date

**Q: Will deletion affect my adherence reports?**

A: No. Past intake records are preserved, so historical adherence data remains intact. Only future schedules are prevented.

---

**API Version:** 2.4
**Last Updated:** October 21, 2025  
**Status:** Production Ready

**Changelog:**
- v2.4 (Oct 21, 2025): Removed My Medication Menu List feature (deprecated); `isAddToMyMedication` field reserved for future use
- v2.2 (Oct 16, 2025): Added `/schedule/{scheduleUuid}` (DELETE) endpoint for medication schedule deletion; Implements soft delete to preserve medication history
- v2.1 (Oct 15, 2025): Added `/schedule/{scheduleUuid}` (GET) and `/edit-schedule` (PUT) endpoints for schedule editing workflow; Enables users to retrieve and update medication schedules
- v2.0 (Oct 15, 2025): Added `/upcoming-schedules` and `/missed-schedules` endpoints; Enhanced `/intake-stats` with `totalMissed` field; Refactored time slot filtering using interface-based design
- v1.2 (Oct 14, 2025): Added `/track-intake` and `/intake-stats` endpoints for comprehensive medication adherence tracking
- v1.1 (Oct 14, 2025): Added `/schedule-list` endpoint with date range expansion and intake tracking
- v1.0 (Oct 12, 2025): Initial release with search, add, and list endpoints

## Support

For issues or questions:
- Check the error message in the response
- Verify authentication token is valid and included
- Ensure required fields are provided
- Verify date and time formats are correct
