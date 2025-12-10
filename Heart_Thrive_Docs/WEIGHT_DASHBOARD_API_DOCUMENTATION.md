# Weight Dashboard API Documentation

## Overview

The Weight Dashboard API provides comprehensive weight tracking data for users, including daily weight records, delta weight calculations, and BMI information. This API handles timezone conversions and calculates weight differences between days accurately.

---

## API Endpoint

**Endpoint:** `GET /api/patient-weight-height-logs/dashborad`

**Method:** `GET`

**Authentication:** Required (User must be logged in)

---

## Request Parameters

| Parameter | Type | Required | Description | Example |
|-----------|------|----------|-------------|---------|
| `startDate` | String | Yes | Start date of the range (inclusive) | `"01-12-2025"` or `"2025-12-01"` |
| `endDate` | String | Yes | End date of the range (inclusive) | `"07-12-2025"` or `"2025-12-07"` |
| `timezone` | String | Yes | User's timezone identifier | `"Asia/Kolkata"`, `"America/New_York"`, `"UTC"` |
| `groupBy` | String | No | Grouping type (default: "day") | `"day"` or `"week"` |

### Date Format Support
- `dd-MM-yyyy` (e.g., `01-12-2025`)
- `yyyy-MM-dd` (e.g., `2025-12-01`)

### Example Request
```
GET /api/patient-weight-height-logs/dashborad?startDate=01-12-2025&endDate=07-12-2025&timezone=Asia/Kolkata&groupBy=day
```

---

## Response Structure

```json
{
  "success": true,
  "message": "Found 7 record(s)",
  "weight unit": "kgs",
  "currentWeight": "88",
  "safeWeight": "68.8",
  "data": [
    {
      "day": 1,
      "currentWeight": "50",
      "deltaWeight": {
        "currentWeight": "50",
        "previousWeight": "0",
        "changeType": "0",
        "deltaValue": "0"
      },
      "deltaDay": {
        "currentWeight": "50",
        "previousWeight": "50",
        "changeType": "no change",
        "deltaValue": "0"
      },
      "bmiValue": 27.46,
      "bmiStatus": "Normal",
      "recordedAt": "2025-12-01"
    }
  ]
}
```

### Response Fields

#### Root Level
- `success` (Boolean): Request success status
- `message` (String): Descriptive message about the result
- `weight unit` (String): Unit used for weight values (`"kgs"` or `"lbs"`)
- `currentWeight` (String): User's latest weight (global)
- `safeWeight` (String): Calculated safe/ideal weight

#### Data Array - Each Day Object
- `day` (Integer): Day number in the range (1, 2, 3, ...)
- `currentWeight` (String): Weight for this specific day (or latest before this day if no record)
- `deltaWeight` (Object): Delta Weight - difference from previous day
- `deltaDay` (Object): Delta Day - difference from start date
- `bmiValue` (Double): BMI value for this day (0.0 if no record)
- `bmiStatus` (String): BMI status like "Normal", "Overweight" (null if no record)
- `recordedAt` (String): Date in `yyyy-MM-dd` format

#### Delta Weight Object
- `currentWeight` (String): Current day's weight
- `previousWeight` (String): Previous day's weight (or latest before previous day)
- `changeType` (String): `"increased"`, `"decreased"`, `"no change"`, or `"0"` (if no data)
- `deltaValue` (String): Absolute difference value

#### Delta Day Object
- `currentWeight` (String): Current day's weight
- `previousWeight` (String): Start date's weight (or latest before start date)
- `changeType` (String): `"increased"`, `"decreased"`, `"no change"`, or `"0"` (if no data)
- `deltaValue` (String): Absolute difference from start date

---

## Weight Calculation Logic

### Understanding Current Weight for Each Day

For each day in the date range, the system determines the weight as follows:

1. **If a record exists for that day:**
   - Use the **latest record** for that day (if multiple records exist on the same day)

2. **If no record exists for that day:**
   - Use the **latest weight record** that was recorded **before** that day
   - This ensures continuity - if you didn't update weight on Dec 5, it shows your last known weight

3. **If no weight records exist at all:**
   - Show `"0"` for all weight values

### Delta Weight (DW) Calculation

**Formula:** `Current Day Weight - Previous Day Weight`

**Logic:**
- **Day 1:** `Day 1 Weight - Latest Weight Before Start Date`
- **Day 2:** `Day 2 Weight - Day 1 Weight` (or latest before Day 1 if Day 1 has no record)
- **Day 3:** `Day 3 Weight - Day 2 Weight` (or latest before Day 2 if Day 2 has no record)
- And so on...

**Example:**
- Dec 1: 50 kg (no previous day) → DW = 0
- Dec 2: 70 kg (previous: 50 kg) → DW = 70 - 50 = 20 kg
- Dec 3: 71 kg (previous: 70 kg) → DW = 71 - 70 = 1 kg

### Delta Day (DW1, DW2, DW3...) Calculation

**Formula:** `Current Day Weight - Start Date Weight`

**Logic:**
- **Start Date Weight:** Latest record for start date, or latest before start date if no record
- **Day 1:** `Day 1 Weight - Start Date Weight`
- **Day 2:** `Day 2 Weight - Start Date Weight`
- **Day 3:** `Day 3 Weight - Start Date Weight`
- And so on...

**Note:** Delta Day compares each day against the **start date**, not the previous day.

**Example:**
- Start Date (Dec 1): 50 kg
- Dec 1: 50 kg → DW1 = 50 - 50 = 0
- Dec 2: 70 kg → DW2 = 70 - 50 = 20
- Dec 3: 71 kg → DW3 = 71 - 50 = 21
- Dec 7: 69 kg → DW7 = 69 - 50 = 19

---

## Scenarios with Examples

### Scenario 1: Daily Weight Updates

**Date Range:** Dec 1 - Dec 7  
**Weight Records:**
- Dec 1: 50 kg
- Dec 2: 70 kg
- Dec 3: 71 kg
- Dec 4: 71 kg
- Dec 5: 71 kg
- Dec 6: 72 kg
- Dec 7: 69 kg

**Result:**

| Day | Date | Current Weight | Delta Weight | Delta Day |
|-----|------|----------------|--------------|-----------|
| 1 | Dec 1 | 50 | 0 (no previous) | 0 (50 - 50) |
| 2 | Dec 2 | 70 | 20 (70 - 50) | 20 (70 - 50) |
| 3 | Dec 3 | 71 | 1 (71 - 70) | 21 (71 - 50) |
| 4 | Dec 4 | 71 | 0 (71 - 71) | 21 (71 - 50) |
| 5 | Dec 5 | 71 | 0 (71 - 71) | 21 (71 - 50) |
| 6 | Dec 6 | 72 | 1 (72 - 71) | 22 (72 - 50) |
| 7 | Dec 7 | 69 | -3 (69 - 72) | 19 (69 - 50) |

---

### Scenario 2: No Updates During Week

**Date Range:** Dec 1 - Dec 7  
**Last Weight Record:** Nov 25 = 60 kg  
**No records during Dec 1-7**

**Result:**

| Day | Date | Current Weight | Delta Weight | Delta Day |
|-----|------|----------------|--------------|-----------|
| 1 | Dec 1 | 60 | 0 | 0 (60 - 60) |
| 2 | Dec 2 | 60 | 0 (60 - 60) | 0 (60 - 60) |
| 3 | Dec 3 | 60 | 0 (60 - 60) | 0 (60 - 60) |
| 4 | Dec 4 | 60 | 0 (60 - 60) | 0 (60 - 60) |
| 5 | Dec 5 | 60 | 0 (60 - 60) | 0 (60 - 60) |
| 6 | Dec 6 | 60 | 0 (60 - 60) | 0 (60 - 60) |
| 7 | Dec 7 | 60 | 0 (60 - 60) | 0 (60 - 60) |

**Explanation:** Since no records exist during the week, all days show the last known weight (60 kg from Nov 25). All delta values are 0 because weights are the same.

---

### Scenario 3: Single Update During Week

**Date Range:** Dec 1 - Dec 7  
**Last Weight Before Week:** Nov 25 = 60 kg  
**Update:** Dec 4 (Thursday) = 64 kg  
**No other updates**

**Result:**

| Day | Date | Current Weight | Delta Weight | Delta Day |
|-----|------|----------------|--------------|-----------|
| 1 | Dec 1 | 60 | 0 | 0 (60 - 60) |
| 2 | Dec 2 | 60 | 0 (60 - 60) | 0 (60 - 60) |
| 3 | Dec 3 | 60 | 0 (60 - 60) | 0 (60 - 60) |
| 4 | Dec 4 | 64 | 4 (64 - 60) | 4 (64 - 60) |
| 5 | Dec 5 | 64 | 0 (64 - 64) | 4 (64 - 60) |
| 6 | Dec 6 | 64 | 0 (64 - 64) | 4 (64 - 60) |
| 7 | Dec 7 | 64 | 0 (64 - 64) | 4 (64 - 60) |

**Explanation:**
- Days 1-3: Show last known weight (60 kg) before the week
- Day 4: New weight recorded (64 kg), so DW = 4 (64 - 60)
- Days 5-7: Continue showing 64 kg (last known weight), DW = 0 (no change from previous day), but DW4-DW7 = 4 (compared to start date)

---

### Scenario 4: Multiple Records on Same Day

**Date Range:** Dec 1 - Dec 2  
**Dec 1 Records:**
- 8:00 AM: 50 kg
- 6:00 PM: 52 kg
- 10:00 PM: 51 kg

**Dec 2:** 70 kg

**Result:**

| Day | Date | Current Weight | Delta Weight | Delta Day |
|-----|------|----------------|--------------|-----------|
| 1 | Dec 1 | 51 | 0 | 0 (51 - 51) |
| 2 | Dec 2 | 70 | 19 (70 - 51) | 19 (70 - 51) |

**Explanation:** For Dec 1, the system uses the **latest record** (10:00 PM = 51 kg), not the first or average.

---

## Key Concepts

### 1. Current Weight Per Day
- **With Record:** Latest record for that day
- **Without Record:** Latest weight before that day
- **No Records:** Shows "0"

### 2. Delta Weight (DW)
- Compares current day with **previous day**
- Shows day-to-day weight change
- Helps track daily fluctuations

### 3. Delta Day (DW1, DW2, DW3...)
- Compares current day with **start date**
- Shows progress from the beginning of the period
- Helps track overall progress

### 4. Timezone Handling
- All dates are interpreted in the user's timezone
- Database stores UTC timestamps
- API converts between user timezone and UTC automatically

### 5. Unit Conversion
- Supports both `kgs` and `lbs`
- Automatically converts weights to the user's preferred unit
- Unit is specified at root level: `"weight unit": "kgs"` or `"lbs"`

---

## Error Handling

### Invalid Date Format
```json
{
  "error": "Invalid date format: 01/12/2025. Expected dd-MM-yyyy or yyyy-MM-dd"
}
```

### Invalid Timezone
```json
{
  "error": "Invalid timezone: InvalidTZ"
}
```

### Date Range Validation
```json
{
  "error": "fromDate cannot be after toDate"
}
```

---

## Notes

1. **Weight Values:** All weights are returned as strings without unit suffix (unit is specified at root level)
2. **Missing Data:** When no data exists, values show `"0"` instead of `"NR"` (Not Recorded)
3. **BMI Status:** Returns `null` if no record exists for that day (not `"0"`)
4. **Multiple Records:** If multiple records exist on the same day, only the latest (by timestamp) is used
5. **Timezone:** Always provide the user's actual timezone for accurate date calculations

---

## Example Complete Response

```json
{
  "success": true,
  "message": "Found 7 record(s)",
  "weight unit": "kgs",
  "currentWeight": "69",
  "safeWeight": "68.8",
  "data": [
    {
      "day": 1,
      "currentWeight": "50",
      "deltaWeight": {
        "currentWeight": "50",
        "previousWeight": "0",
        "changeType": "0",
        "deltaValue": "0"
      },
      "deltaDay": {
        "currentWeight": "50",
        "previousWeight": "50",
        "changeType": "no change",
        "deltaValue": "0"
      },
      "bmiValue": 27.46,
      "bmiStatus": "Normal",
      "recordedAt": "2025-12-01"
    },
    {
      "day": 2,
      "currentWeight": "70",
      "deltaWeight": {
        "currentWeight": "70",
        "previousWeight": "50",
        "changeType": "increased",
        "deltaValue": "20"
      },
      "deltaDay": {
        "currentWeight": "70",
        "previousWeight": "50",
        "changeType": "increased",
        "deltaValue": "20"
      },
      "bmiValue": 27.77,
      "bmiStatus": "Overweight",
      "recordedAt": "2025-12-02"
    },
    {
      "day": 3,
      "currentWeight": "71",
      "deltaWeight": {
        "currentWeight": "71",
        "previousWeight": "70",
        "changeType": "increased",
        "deltaValue": "1"
      },
      "deltaDay": {
        "currentWeight": "71",
        "previousWeight": "50",
        "changeType": "increased",
        "deltaValue": "21"
      },
      "bmiValue": 27.88,
      "bmiStatus": "Overweight",
      "recordedAt": "2025-12-03"
    }
  ]
}
```

---

## Summary

This API provides a comprehensive view of weight tracking with:
- **Daily weight records** (or latest known weight if not updated)
- **Delta Weight:** Day-to-day weight changes
- **Delta Day:** Progress from start date
- **BMI information** for each day
- **Timezone-aware** date handling
- **Automatic unit conversion** (kgs ↔ lbs)

The system ensures continuity by using the latest known weight when a day has no record, making it ideal for dashboard displays where users may not update weight daily.

