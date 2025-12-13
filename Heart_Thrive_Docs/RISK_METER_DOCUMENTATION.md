# Heart Risk Meter Functionality - Complete Documentation

## Table of Contents
1. [Overview](#overview)
2. [Risk Meter Calculation](#risk-meter-calculation)
3. [Three Risk Factors](#three-risk-factors)
4. [API Endpoints](#api-endpoints)
5. [Request/Response Examples](#requestresponse-examples)
6. [Implementation Details](#implementation-details)
7. [Timezone Handling](#timezone-handling)
8. [Database Schema](#database-schema)
9. [Constants Reference](#constants-reference)

---

## Overview

The Heart Risk Meter is a comprehensive health monitoring system that calculates a patient's heart failure risk score based on three critical factors:
- **Sodium Intake** (15% weight = 3 points base, max 8.5 points)
- **Medication Compliance** (42.5% weight = 8.5 points max)
- **Weight Monitoring** (42.5% weight = 8.5 points max)

**Total Maximum Score:** 20 points (but displayed as 0-8+ scale)

The system provides two main endpoints:
1. `/riskMetricDashboard` - Calculates and saves current risk metrics
2. `/heart-risk-summary` - Retrieves aggregated summary for date ranges

---

## Risk Meter Calculation

### Risk Score Categories

The risk meter displays scores in the following categories (configurable via `RiskLevel` table):

| Range | Category | Description |
|-------|----------|-------------|
| 0-2 | Very Low | Minimal risk |
| 2-4 | Low | Low risk |
| 4-6 | Moderate | Moderate risk |
| 6-8 | High | High risk |
| 8+ | Critical | Critical risk |

**Note:** Categories are dynamically loaded from the `risk_level` database table, sorted by `threshold_min` ascending. The first level is used as default if no levels match.

### Total Score Calculation

```
Total Risk Score = Sodium Score + Medication Score + Weight Score
```

Each component is capped at its maximum:
- Sodium: Max 8.5 points
- Medication: Max 8.5 points  
- Weight: Max 8.5 points

---

## Three Risk Factors

### 1. Sodium Intake (15% of total points)

**Share:** 15% = 3 points base, maximum 8.5 points

**Calculation Logic:**
- **Threshold:** 2,500 mg per day
- **Base Points:** 3 points (for crossing 2,500 mg)
- **Incremental Points:** 0.5 points per 500 mg above threshold
- **Maximum:** 8.5 points

**Formula:**
```
If sodiumIntake < 2500 mg:
    sodiumScore = 0
    
If sodiumIntake >= 2500 mg:
    sodiumScore = 3 + ceil((sodiumIntake - 2500) / 500) * 0.5
    sodiumScore = min(sodiumScore, 8.5)  // Cap at 8.5
```

**Examples:**

| Sodium Intake | Calculation | Risk Points |
|---------------|-------------|-------------|
| 2,000 mg | Below threshold | 0 points |
| 2,500 mg | At threshold (not shown) | 0 points (threshold not crossed) |
| 2,501 mg | 3 + ceil(1/500) * 0.5 = 3.5 | 3.5 points |
| 3,000 mg | 3 + ceil(500/500) * 0.5 = 3.5 | 3.5 points |
| 3,001 mg | 3 + ceil(501/500) * 0.5 = 4.0 | 4.0 points |
| 3,500 mg | 3 + ceil(1000/500) * 0.5 = 4.0 | 4.0 points |
| 3,501 mg | 3 + ceil(1001/500) * 0.5 = 4.5 | 4.5 points |
| 4,000 mg | 3 + ceil(1500/500) * 0.5 = 4.5 | 4.5 points |
| 4,001 mg | 3 + ceil(1501/500) * 0.5 = 5.0 | 5.0 points |
| 4,500 mg | 3 + ceil(2000/500) * 0.5 = 5.0 | 5.0 points |
| 4,501 mg | 3 + ceil(2001/500) * 0.5 = 5.5 | 5.5 points |
| 5,000 mg | 3 + ceil(2500/500) * 0.5 = 5.5 | 5.5 points |
| 5,001 mg | 3 + ceil(2501/500) * 0.5 = 6.0 | 6.0 points |
| 5,500+ mg | Capped at maximum | 6.0 points (max 8.5) |

**For Multi-Day Ranges:**
- Sodium intake is **summed** across all days
- Threshold is calculated as: `dailyThreshold * actualDays`
- Example: 7 days = 2,500 mg × 7 = 17,500 mg total threshold

---

### 2. Medication Compliance (42.5% of total points)

**Share:** 42.5% = 8.5 points maximum

**Calculation Logic:**
- **Minimum Points per Missed Dose:** 2.125 points
- **Maximum Points:** 8.5 points (even if more doses missed)
- **Per Dose Calculation:** `pointsPerDose = max(8.5 / scheduledCount, 2.125)`

**Formula:**
```
pointsPerMedication = max(MEDICATION_MAX_POINTS / scheduledCount, MEDICATION_MIN_POINTS_PER_DOSE)
medicationScore = missedCount * pointsPerMedication
medicationScore = min(medicationScore, 8.5)  // Cap at 8.5
```

**Examples:**

| Scheduled | Missed | Points per Dose | Total Score |
|-----------|--------|----------------|-------------|
| 4 | 1 | max(8.5/4, 2.125) = 2.125 | 2.125 points |
| 4 | 2 | 2.125 | 4.25 points (2.125 × 2) |
| 4 | 3 | 2.125 | 6.375 points (2.125 × 3) |
| 4 | 4 | 2.125 | 8.5 points (capped) |
| 2 | 1 | max(8.5/2, 2.125) = 4.25 | 4.25 points |
| 2 | 2 | 4.25 | 8.5 points (capped) |
| 10 | 1 | max(8.5/10, 2.125) = 2.125 | 2.125 points |
| 10 | 10 | 2.125 | 8.5 points (capped, not 21.25) |

**For Multi-Day Ranges:**
- Scheduled and missed counts are **summed** across all days
- Average score per day is calculated: `totalScore / actualDays`
- Minimum threshold applied: if `avgScore < 2.125`, set to `2.125`
- Percentage calculation: `(missed / scheduled) * 100`

**Example - 7 Days:**
```
Day 1: 4 scheduled, 1 missed = 2.125 points
Day 2: 4 scheduled, 0 missed = 0 points
Day 3: 4 scheduled, 1 missed = 2.125 points
Day 4: 4 scheduled, 0 missed = 0 points
Day 5: 4 scheduled, 1 missed = 2.125 points
Day 6: 4 scheduled, 0 missed = 0 points
Day 7: 4 scheduled, 0 missed = 0 points

Total: 28 scheduled, 3 missed
Total Score: 6.375 points
Average Score per Day: 6.375 / 7 = 0.91 → 2.125 (minimum applied)
Percentage: (3 / 28) * 100 = 10.71%
```

---

### 3. Weight Monitoring (42.5% of total points)

**Share:** 42.5% = 8.5 points maximum

**Calculation Logic:**
- **Baseline Weight:** Calculated from height using BMI formula (target BMI = 21.7)
- **Time Window:** Last 48 hours for weight increase detection
- **Base Threshold:** 2 lbs increase = 100% = 8.5 points
- **Dynamic Calculation:** Percentage-based scoring

**Baseline Weight Calculation:**
```
heightInMeters = convertHeightToMeters(height, unit)
baselineWeight = 21.7 * (heightInMeters²)
baselineWeight = convertToLbs(baselineWeight)  // Always stored in lbs
```

**Weight Score Formula:**
```
weightIncrease = maxWeightInLast48Hours - baselineWeight

If weightIncrease <= 0:
    weightScore = 0
    
If weightIncrease > 0:
    percentage = (weightIncrease / 2.0) * 100
    percentage = min(percentage, 100.0)  // Cap at 100%
    weightScore = (percentage / 100.0) * 8.5
```

**Examples:**

| Weight Increase | Percentage | Risk Points |
|----------------|------------|-------------|
| 0 lbs | 0% | 0 points |
| 0.5 lbs | (0.5/2) × 100 = 25% | 2.125 points |
| 1.0 lbs | (1.0/2) × 100 = 50% | 4.25 points |
| 1.5 lbs | (1.5/2) × 100 = 75% | 6.375 points |
| 2.0 lbs | (2.0/2) × 100 = 100% | 8.5 points |
| 3.0 lbs | Capped at 100% | 8.5 points (not 12.75) |

**Weight Detection Logic:**
1. Query weight logs from last 7 days (from `fromDate` minus 7 days to `toDate`)
2. Filter to last 48 hours from `toDate` end time
3. Find maximum weight in that 48-hour window
4. If no weight in 48 hours, check previous day's latest weight
5. Calculate increase from baseline

**For Multi-Day Ranges (Weekly Calculation):**
- Uses latest weight from the date range
- Compares against baseline weight
- Weekly limit: 5 lbs increase = 100%
- Formula: `percentage = (increase / 5.0) * 100`, capped at 100%

**Example:**
```
Baseline Weight: 80 kg (176 lbs)
Day 1: 80 kg
Day 2: 80 kg
Day 3: 80 kg
Day 4: 90 kg (198 lbs) - spike
Day 5: 80 kg
Day 6: 80 kg
Day 7: 81 kg (178.5 lbs) - latest

Latest Weight: 81 kg (178.5 lbs)
Increase: 178.5 - 176 = 2.5 lbs
Percentage: (2.5 / 5.0) * 100 = 50%
Score: 4.25 points
```

---

## API Endpoints

### 1. POST `/api/patient-heart-risk-metrics/riskMetricDashboard`

**Purpose:** Calculate current risk metrics for a date range and save/update the record.

**Flow:**
1. Validates `SearchRootObject` (fromDate, toDate, timezone)
2. Retrieves current user from authentication context
3. Resolves date range using `DateRangeResolver`
4. Calls `PatientRiskMeterService.calculateRiskMeter()` to calculate scores
5. Determines risk category from `RiskLevel` table
6. Saves/updates `PatientHeartRiskMetric` record
7. Returns `RiskSymptomDTO` with current score and category

**Key Features:**
- Supports single-day and multi-day date ranges
- Handles timezone conversion properly
- Updates existing record if found (by `metricDate`), otherwise creates new
- Uses `createdDate` (UTC) for querying to avoid timezone boundary issues

**Request Body:**
```json
{
  "fromDate": "12-12-2024",
  "toDate": "12-12-2024",
  "timezone": "America/Phoenix"
}
```

**Response:**
```json
{
  "riskSymptom": {
    "score": 12.25,
    "category": "Critical",
    "message": "",
    "scale": [
      {
        "range": "0-2",
        "label": "Very Low"
      },
      {
        "range": "2-4",
        "label": "Low"
      },
      {
        "range": "4-6",
        "label": "Moderate"
      },
      {
        "range": "6-8",
        "label": "High"
      },
      {
        "range": ">8",
        "label": "Critical"
      }
    ]
  }
}
```

---

### 2. POST `/api/patient-heart-risk-metrics/heart-risk-summary`

**Purpose:** Retrieve aggregated summary of risk metrics for a date range.

**Flow:**
1. Validates `SearchRootObject`
2. Retrieves current user from authentication context
3. Resolves date range using `DateRangeResolver`
4. Queries `PatientHeartRiskMetric` records by `createdDate` (UTC) in range
5. Groups records by user timezone day, keeping latest per day
6. Calculates averages and summaries:
   - Average risk score
   - Total sodium intake and percentage
   - Total medication scheduled/missed and percentage
   - Weight summary (baseline, current, percentage)
7. Returns `PatientHeartRiskMetricsSummaryDTO`

**Key Features:**
- Aggregates data across multiple days
- Handles timezone conversion for grouping
- Calculates dynamic thresholds based on actual days
- Returns detailed breakdowns for each factor

**Request Body:**
```json
{
  "fromDate": "01-12-2024",
  "toDate": "07-12-2024",
  "timezone": "America/Phoenix"
}
```

**Response:**
```json
{
  "riskSymptom": {
    "score": 8.5,
    "category": "High",
    "message": "",
    "scale": [
      {
        "range": "0-2",
        "label": "Very Low"
      },
      {
        "range": "2-4",
        "label": "Low"
      },
      {
        "range": "4-6",
        "label": "Moderate"
      },
      {
        "range": "6-8",
        "label": "High"
      },
      {
        "range": ">8",
        "label": "Critical"
      }
    ]
  },
  "sodium": {
    "value": 17500.0,
    "percentage": 100.0
  },
  "medication": {
    "missedCount": 3.0,
    "scheduleCount": 28.0,
    "percentage": 10.71
  },
  "weight": {
    "baselineWeight": 176.0,
    "currentWeight": 2.5,
    "percentage": 50.0,
    "weightUnitType": "lbs"
  }
}
```

---

## Request/Response Examples

### Example 1: Single Day Risk Calculation

**Request:**
```http
POST /api/patient-heart-risk-metrics/riskMetricDashboard
Content-Type: application/json

{
  "fromDate": "12-12-2024",
  "toDate": "12-12-2024",
  "timezone": "America/Phoenix"
}
```

**Response:**
```json
{
  "riskSymptom": {
    "score": 12.25,
    "category": "Critical",
    "message": "",
    "scale": [
      {
        "range": "0-2",
        "label": "Very Low"
      },
      {
        "range": "2-4",
        "label": "Low"
      },
      {
        "range": "4-6",
        "label": "Moderate"
      },
      {
        "range": "6-8",
        "label": "High"
      },
      {
        "range": ">8",
        "label": "Critical"
      }
    ]
  }
}
```

**Calculation Breakdown:**
- Sodium: 3,000 mg = 3.5 points
- Medication: 2 missed out of 3 scheduled = 4.25 points
- Weight: 1 lb increase in 48hr = 4.5 points
- **Total: 12.25 points → Critical**

---

### Example 2: Weekly Summary

**Request:**
```http
POST /api/patient-heart-risk-metrics/heart-risk-summary
Content-Type: application/json

{
  "fromDate": "01-12-2024",
  "toDate": "07-12-2024",
  "timezone": "America/Phoenix"
}
```

**Response:**
```json
{
  "riskSymptom": {
    "score": 6.5,
    "category": "High",
    "message": "",
    "scale": [...]
  },
  "sodium": {
    "value": 21000.0,
    "percentage": 120.0
  },
  "medication": {
    "missedCount": 5.0,
    "scheduleCount": 28.0,
    "percentage": 17.86
  },
  "weight": {
    "baselineWeight": 176.0,
    "currentWeight": 3.0,
    "percentage": 60.0,
    "weightUnitType": "lbs"
  }
}
```

---

## Implementation Details

### Service Layer Architecture

```
PatientHeartRiskMetricResource (REST Controller)
    ↓
PatientHeartRiskMetricServiceImpl (Service)
    ↓
PatientRiskMeterService (Calculation Service)
    ├── MedicationService.getMedicationIntakeStats()
    ├── PatientMealLogService.getCurrentUserNutrientSummary()
    └── PatientWeightHeightLogRepository (Weight queries)
```

### Key Classes

1. **PatientHeartRiskMetricServiceImpl**
   - `patientHeartRiskMetricsDashboard()` - Main dashboard endpoint logic
   - `getHeartRiskMetricsSummaryByCurrentUserAndDateRange()` - Summary endpoint logic
   - `getLatestRecordsPerUserTimezoneDay()` - Groups records by timezone day
   - `calculateAndCreateSummary()` - Aggregates data for summary
   - `getSodiumPercentageAndScore()` - Sodium summary calculation
   - `getMedicationSummary()` - Medication summary calculation
   - `getWeightSummary()` - Weight summary calculation

2. **PatientRiskMeterService**
   - `calculateRiskMeter()` - Core calculation logic
   - `calculateMedicationScore()` - Medication scoring
   - `calculateBmiAndNormalWeight()` - Baseline weight calculation
   - `getIncreasedWeightInLast48Hours()` - Weight increase detection

3. **DateRangeResolver**
   - Resolves date ranges from `SearchRootObject`
   - Handles timezone conversions
   - Supports range types: today, week, month, year, lifetime

### Data Flow

#### Dashboard Endpoint Flow:
```
1. SearchRootObject → DateRangeResolver → UTC LocalDateTime[]
2. Format dates for PatientRiskMeterService (dd-MM-yyyy)
3. PatientRiskMeterService.calculateRiskMeter():
   a. Get sodium from PatientMealLogService
   b. Get medication stats from MedicationService
   c. Get weight logs from repository
   d. Calculate all three scores
4. Determine risk category from RiskLevel table
5. Query existing record by createdDate (UTC range)
6. Save/update PatientHeartRiskMetric
7. Return RiskSymptomDTO
```

#### Summary Endpoint Flow:
```
1. SearchRootObject → DateRangeResolver → UTC LocalDateTime[]
2. Query PatientHeartRiskMetric by createdDate (UTC range)
3. Group by user timezone day (using metricDate or createdDate)
4. Keep latest record per day
5. Calculate averages:
   - Average risk score
   - Sum sodium intake
   - Sum medication scheduled/missed
   - Latest weight
6. Calculate summaries with dynamic thresholds
7. Return PatientHeartRiskMetricsSummaryDTO
```

---

## Timezone Handling

### Critical Timezone Considerations

1. **Database Storage:**
   - `createdDate`: Stored as `TIMESTAMP` in UTC
   - `metricDate`: Stored as `DATE` (timezone-agnostic, represents business date)

2. **Query Strategy:**
   - Always query by `createdDate` (UTC) for timezone-agnostic retrieval
   - Convert `createdDate` to user timezone for grouping/display
   - Use `metricDate` for business logic (which day the metric represents)

3. **Date Range Resolution:**
   - Frontend sends dates in user timezone (e.g., "12-12-2024")
   - `DateRangeResolver` converts to UTC `LocalDateTime` for database queries
   - User timezone is preserved for business date calculations

4. **Multi-Day Range Handling:**
   - Query entire date range using UTC `Instant` range
   - Group records by user timezone day
   - Keep latest record per day to avoid duplicates

### Example Timezone Conversion:

```
User Timezone: America/Phoenix (UTC-7)
Date: 12-12-2024

Frontend sends: "12-12-2024" (Phoenix timezone)
↓
DateRangeResolver converts:
  fromDateTime: 2024-12-12T00:00:00 (Phoenix) → 2024-12-12T07:00:00 (UTC)
  toDateTime: 2024-12-12T23:59:59 (Phoenix) → 2024-12-13T06:59:59 (UTC)
↓
Database Query:
  WHERE createdDate BETWEEN '2024-12-12T07:00:00Z' AND '2024-12-13T06:59:59Z'
↓
Group by metricDate (business date): 2024-12-12
```

---

## Database Schema

### PatientHeartRiskMetric Table

| Column | Type | Description |
|--------|------|-------------|
| `id` | BIGINT | Primary key |
| `uuid` | VARCHAR(36) | Unique identifier |
| `patient_id` | BIGINT | Foreign key to user table |
| `metric_date` | DATE | Business date (user timezone) |
| `created_date` | TIMESTAMP | UTC timestamp of record creation |
| `risk_score` | DECIMAL | Total risk score (0-20) |
| `risk_symptom_level` | VARCHAR(255) | Category (Very Low, Low, etc.) |
| `sodium_intake` | DECIMAL | Total sodium in mg |
| `sodium_score` | DOUBLE | Sodium component score |
| `missed_med_count` | INTEGER | Total missed medications |
| `schedule_med_count` | INTEGER | Total scheduled medications |
| `medicine_score` | DOUBLE | Medication component score |
| `bmi_value` | DECIMAL | BMI value |
| `base_weight` | DECIMAL | Baseline weight (in lbs) |
| `current_weight` | DECIMAL | Current weight (in lbs) |
| `weight_score` | DOUBLE | Weight component score |
| `weight_unit_type` | VARCHAR(10) | Unit type (lbs/kg) |
| `active` | BOOLEAN | Active status |
| `created_by` | VARCHAR(255) | Creator user ID |
| `last_modified_date` | TIMESTAMP | Last modification timestamp |
| `last_modified_by` | VARCHAR(255) | Last modifier user ID |

### RiskLevel Table

| Column | Type | Description |
|--------|------|-------------|
| `id` | BIGINT | Primary key |
| `label` | VARCHAR(255) | Category label (Very Low, Low, etc.) |
| `threshold_min` | DECIMAL | Minimum score for this level |
| `threshold_max` | DECIMAL | Maximum score for this level (null for open-ended) |
| `active` | BOOLEAN | Active status |

**Example RiskLevel Records:**
```
id | label      | threshold_min | threshold_max | active
1  | Very Low   | 0            | 2             | true
2  | Low        | 2            | 4             | true
3  | Moderate   | 4            | 6             | true
4  | High       | 6            | 8             | true
5  | Critical   | 8            | null          | true
```

---

## Constants Reference

All constants are defined in `Constants.java`:

### Sodium Constants
```java
public static final double DAILY_SODIUM_THRESHOLD_MG = 2500.0;
public static final double SODIUM_SCORING_INCREMENT_MG = 500.0;
public static final double SODIUM_MAX_SCORE = 8.5;
```

### Medication Constants
```java
public static final double MEDICATION_MAX_POINTS = 8.5;
public static final double MEDICATION_MIN_POINTS_PER_DOSE = 2.125;
```

### Weight Constants
```java
public static final double WEIGHT_INCREASE_DAY_LIMIT_LBS = 2.0;
public static final double WEIGHT_INCREASE_WEEKLY_LIMIT_LBS = 5.0;
public static final double WEIGHT_SCORE_MAX_POINTS = 8.5;
public static final double WEIGHT_SCORE_BASE_THRESHOLD_LBS = 2.0;
public static final int WEIGHT_MONITERING_FROM_LAST_HOURS = 48;
```

### BMI Constants
```java
public static final BigDecimal NORMAL_BMI_VALUE = BigDecimal.valueOf(21.7);
```

### Weight Unit Constants
```java
public static final String WEIGHT_UNIT_LBS = "lbs";
public static final String WEIGHT_UNIT_KG = "kg";
```

### Date Format Constants
```java
public static final String DATE_FORMAT_DD_MM_YYYY = "dd-MM-yyyy";
```

### Timezone Constants
```java
public static final String TIME_ZONE_UTC = "UTC";
public static final String DEFAULT_USER_TIMEZONE = "America/New_York";
```

---

## Complete Example Scenario

### Scenario: Monday Risk Calculation

**Input Data:**
- Sodium Intake: 3,000 mg
- Medications: 3 scheduled, 2 missed
- Weight: Increased 1 lb in last 48 hours
- Baseline Weight: 176 lbs

**Calculation:**

1. **Sodium Score:**
   ```
   Intake: 3,000 mg
   Threshold: 2,500 mg
   Excess: 3,000 - 2,500 = 500 mg
   Increments: ceil(500 / 500) = 1
   Score: 3 + (1 × 0.5) = 3.5 points
   ```

2. **Medication Score:**
   ```
   Scheduled: 3
   Missed: 2
   Points per dose: max(8.5 / 3, 2.125) = 2.125
   Score: 2 × 2.125 = 4.25 points
   ```

3. **Weight Score:**
   ```
   Increase: 1 lb
   Percentage: (1 / 2.0) × 100 = 50%
   Score: (50 / 100) × 8.5 = 4.25 points
   ```

4. **Total Score:**
   ```
   Total: 3.5 + 4.25 + 4.25 = 12.0 points
   Category: Critical (8+)
   ```

**API Response:**
```json
{
  "riskSymptom": {
    "score": 12.0,
    "category": "Critical",
    "message": "",
    "scale": [...]
  }
}
```

---

## Important Notes

1. **Score Capping:** All individual scores are capped at their maximum (8.5 points each), but total can exceed 8.5 (up to 20 theoretically, though typically displayed as 8+).

2. **Timezone Consistency:** Always use `createdDate` (UTC) for database queries, and `metricDate` (business date) for grouping and display.

3. **Multi-Day Aggregation:** For summary endpoint, sodium is summed, medication counts are summed, and weight uses latest value with weekly threshold.

4. **Record Deduplication:** The system groups records by user timezone day and keeps only the latest record per day to prevent duplicates.

5. **Baseline Weight:** Always calculated from height using BMI formula (target BMI = 21.7) and stored in lbs for consistency.

6. **Weight Unit Handling:** All internal calculations use lbs. Conversion to kg is done only for display purposes.

7. **Medication Minimum:** Even if many medications are scheduled, each missed dose contributes at least 2.125 points.

8. **Dynamic Thresholds:** For multi-day ranges, sodium threshold is multiplied by actual days, and medication uses average score per day with minimum threshold.

---

## Troubleshooting

### Common Issues:

1. **Duplicate Records:**
   - Ensure querying by `createdDate` (UTC) covers entire date range
   - Use `metricDate` for grouping, not for querying

2. **Timezone Mismatches:**
   - Always use `DateRangeResolver` for date range resolution
   - Convert UTC timestamps to user timezone only for display/grouping

3. **Score Mismatches Between Endpoints:**
   - Dashboard calculates current score
   - Summary calculates average of saved records
   - Ensure records are saved correctly in dashboard endpoint

4. **Weight Not Detected:**
   - Check 48-hour window calculation (from `toDate` end time)
   - Verify weight logs exist in the time window
   - Check unit conversion (all weights converted to lbs internally)

---

## Future Enhancements

1. **BMI Integration:** Currently BMI is calculated but not fully integrated into risk scoring (marked as TODO).

2. **Historical Trends:** Could add trend analysis for risk scores over time.

3. **Notifications:** Integration with notification system for critical risk levels.

4. **Custom Thresholds:** Allow per-patient customization of thresholds.

---

**Document Version:** 1.0  
**Last Updated:** December 2024  
**Maintained By:** Development Team

