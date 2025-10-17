# HeartThrive API Documentation

## Daily Nutrient Summary Endpoint

### **GET** `/api/patient-meal-logs/daily-nutrient-summary`

**Description:** Retrieves daily nutrient summary for the current user within a specified date range. Returns nutrient data broken down by individual days with proper timezone handling.

---

### **Request Parameters**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `fromDate` | String | ✅ Yes | Start date in format `dd-MM-yyyy` (e.g., "27-09-2024") |
| `toDate` | String | ✅ Yes | End date in format `dd-MM-yyyy` (e.g., "27-09-2024") |
| `timezone` | String | ✅ Yes | User timezone (e.g., "Asia/Kolkata", "America/New_York") |

### **Request Example**
```http
GET /api/patient-meal-logs/daily-nutrient-summary?fromDate=27-09-2024&toDate=27-09-2024&timezone=Asia/Kolkata
```

### **Request Body**
```json
{
  "fromDate": "27-09-2024",
  "toDate": "27-09-2024", 
  "timezone": "Asia/Kolkata"
}
```

---

### **Response Structure**

#### **Success Response (200 OK)**
```json
{
  "dailySummaries": [
    {
      "date": "27-09-2024",
      "nutrients": [
        {
          "name": "Sodium",
          "amount": 2.5,
          "minValue": 1.2,
          "maxValue": 6.0,
          "unitName": "g"
        },
        {
          "name": "Protein",
          "amount": 85.3,
          "minValue": 50.0,
          "maxValue": 150.0,
          "unitName": "g"
        }
      ]
    }
  ],
  "fromDate": "27-09-2024",
  "toDate": "27-09-2024",
  "timezone": "Asia/Kolkata"
}
```

#### **Error Responses**

| Status Code | Error | Description |
|-------------|-------|-------------|
| `400` | `BadRequestAlertException` | Missing required parameters (fromDate, toDate, timezone) |
| `400` | `BadRequestAlertException` | fromDate cannot be after toDate |
| `400` | `BadRequestAlertException` | Invalid date format |
| `400` | `BadRequestAlertException` | Invalid timezone format |
| `401` | `Unauthorized` | User not authenticated |
| `404` | `UserNotFoundException` | Current user not found |
| `500` | `Internal Server Error` | Server error during processing |

---

### **Response Fields**

| Field | Type | Description |
|-------|------|-------------|
| `dailySummaries` | Array | List of daily nutrient summaries |
| `dailySummaries[].date` | String | Date in `dd-MM-yyyy` format |
| `dailySummaries[].nutrients` | Array | List of nutrients for this day |
| `dailySummaries[].nutrients[].name` | String | Nutrient name (e.g., "Sodium", "Protein") |
| `dailySummaries[].nutrients[].amount` | BigDecimal | Total amount consumed |
| `dailySummaries[].nutrients[].minValue` | BigDecimal | Minimum recommended value |
| `dailySummaries[].nutrients[].maxValue` | BigDecimal | Maximum recommended value |
| `dailySummaries[].nutrients[].unitName` | String | Unit of measurement (e.g., "g", "mg") |
| `fromDate` | String | Original start date from request |
| `toDate` | String | Original end date from request |
| `timezone` | String | User timezone used for calculations |

---

### **Key Features**

✅ **Timezone Support:** Properly handles timezone conversion for accurate daily grouping  
✅ **Date Range Validation:** Ensures fromDate ≤ toDate  
✅ **Aggregation:** Groups all nutrient intakes by user timezone date  
✅ **Complete Nutrient Data:** Includes min/max recommended values  
✅ **Single Day Support:** Works for single day queries (fromDate = toDate)  

---

### **Timezone Handling Example**

**Scenario:** User in Asia/Kolkata timezone queries for 27-09-2024

1. **Database Query Range (UTC):**
   - Start: `27-09-2024 00:00:00 Asia/Kolkata` → `26-09-2024 18:30:00 UTC`
   - End: `27-09-2024 23:59:59 Asia/Kolkata` → `27-09-2024 18:29:59 UTC`

2. **Grouping Logic:**
   - Records are fetched from UTC range
   - Converted back to user timezone for grouping
   - All records grouped under "27-09-2024" (user's perceived date)

3. **Result:** Accurate daily nutrient summary respecting user's timezone

---

---

## Heart Risk Metrics Summary Endpoint

### **POST** `/api/patient-heart-risk-metric-histories/heart-risk-summary`

**Description:** Retrieves heart risk metrics summary for the current user within a date range. Returns risk assessment and average metrics with timezone support and latest record per day filtering.

---

### **Request Parameters**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `fromDate` | String | ❌ Optional | Start date in format `dd-MM-yyyy` (e.g., "27-09-2024") |
| `toDate` | String | ❌ Optional | End date in format `dd-MM-yyyy` (e.g., "27-09-2024") |
| `range` | String | ❌ Optional | Predefined range (`today`, `week`, `month`, `year`, `lifetime`) |
| `timezone` | String | ❌ Optional | User timezone (defaults to system default if not provided) |

**Note:** Either provide `fromDate`/`toDate` OR `range`. If neither is provided, retrieves ALL data for the user.

### **Request Example**
```http
POST /api/patient-heart-risk-metric-histories/heart-risk-summary
Content-Type: application/json
```

### **Request Body Examples**

#### **With Date Range:**
```json
{
  "fromDate": "27-09-2024",
  "toDate": "27-09-2024",
  "timezone": "Asia/Kolkata"
}
```

#### **With Range:**
```json
{
  "range": "week",
  "timezone": "Asia/Kolkata"
}
```

#### **All Data (No Filters):**
```json
{
  "timezone": "Asia/Kolkata"
}
```

---

### **Response Structure**

#### **Success Response (200 OK)**
```json
{
  "riskSymptom": {
    "riskSymptom": {
      "score": 75,
      "category": "Moderate Risk",
      "message": "Your heart risk is moderate. Consider lifestyle improvements.",
      "scale": [
        {
          "thresholdMin": 0,
          "thresholdMax": 30,
          "levelName": "Low Risk",
          "color": "green"
        },
        {
          "thresholdMin": 31,
          "thresholdMax": 70,
          "levelName": "Moderate Risk", 
          "color": "yellow"
        },
        {
          "thresholdMin": 71,
          "thresholdMax": 100,
          "levelName": "High Risk",
          "color": "red"
        }
      ]
    }
  },
  "avgMissedMedCount": 2.50,
  "avgSodiumIntake": 1.80,
  "avgBmiValue": 24.75,
  "avgScheduleMedCount": 4.20
}
```

#### **Error Responses**

| Status Code | Error | Description |
|-------------|-------|-------------|
| `400` | `FieldValidationException` | Validation errors (invalid dates, timezone, range) |
| `400` | `FieldValidationException` | fromDate cannot be after toDate |
| `400` | `FieldValidationException` | If only one of fromDate/toDate is provided |
| `401` | `Unauthorized` | User not authenticated |
| `404` | `UserNotFoundException` | Current user not found |
| `500` | `Internal Server Error` | Server error during processing |

---

### **Response Fields**

| Field | Type | Description |
|-------|------|-------------|
| `riskSymptom` | Object | Risk assessment based on average risk score |
| `riskSymptom.riskSymptom.score` | Integer | Average risk score (0-100) |
| `riskSymptom.riskSymptom.category` | String | Risk category (e.g., "Low Risk", "Moderate Risk") |
| `riskSymptom.riskSymptom.message` | String | Risk assessment message |
| `riskSymptom.riskSymptom.scale` | Array | Risk level scale with thresholds and colors |
| `riskSymptom.riskSymptom.scale[].thresholdMin` | Integer | Minimum threshold for this risk level |
| `riskSymptom.riskSymptom.scale[].thresholdMax` | Integer | Maximum threshold for this risk level |
| `riskSymptom.riskSymptom.scale[].levelName` | String | Risk level name |
| `riskSymptom.riskSymptom.scale[].color` | String | Color indicator for UI |
| `avgMissedMedCount` | BigDecimal | Average missed medication count |
| `avgSodiumIntake` | BigDecimal | Average sodium intake |
| `avgBmiValue` | BigDecimal | Average BMI value |
| `avgScheduleMedCount` | BigDecimal | Average scheduled medication count |

---

### **Key Features**

✅ **Flexible Date Filtering:** Supports explicit dates, predefined ranges, or all data  
✅ **Timezone Support:** Properly handles timezone conversion for accurate daily grouping  
✅ **Latest Record Per Day:** Groups by user timezone date and takes latest record per day  
✅ **Risk Assessment:** Calculates risk category based on average risk score  
✅ **Comprehensive Metrics:** Provides averages for all key health metrics  
✅ **Validation:** Comprehensive input validation with detailed error messages  

---

### **Timezone Handling Example**

**Scenario:** User in Asia/Kolkata timezone queries for 27-09-2024

1. **Database Query Range (UTC):**
   - Start: `27-09-2024 00:00:00 Asia/Kolkata` → `26-09-2024 18:30:00 UTC`
   - End: `27-09-2024 23:59:59 Asia/Kolkata` → `27-09-2024 18:29:59 UTC`

2. **Grouping Logic:**
   - Records fetched from UTC range
   - Converted to user timezone for grouping by date
   - Latest record per user timezone day is selected

3. **Average Calculation:**
   - Averages calculated from latest records per day
   - Risk assessment based on average risk score

---

### **Range Options**

| Range Value | Description |
|-------------|-------------|
| `today` | Current day only |
| `week` | Last 7 days |
| `month` | Last 30 days |
| `year` | Last 365 days |
| `lifetime` | All available data |

---

### **Validation Rules**

1. **Date Range:** If `fromDate` and `toDate` provided:
   - Both must be valid dates in `dd-MM-yyyy` format
   - `fromDate` cannot be after `toDate`
   - Both required if either is provided

2. **Timezone:** 
   - Must be valid timezone identifier (e.g., "Asia/Kolkata")
   - Optional - defaults to system default if not provided

3. **Range:**
   - Must be one of: `today`, `week`, `month`, `year`, `lifetime`
   - Cannot be combined with explicit date range

---

## Common Features

### **Authentication**
Both endpoints require user authentication. The current user is automatically determined from the security context.

### **Timezone Handling**
Both endpoints properly handle timezone conversion:
- User-provided dates are interpreted in the user's timezone
- Database queries use UTC timestamps
- Results are grouped by user timezone dates

### **Error Handling**
Comprehensive error handling with:
- Input validation
- Detailed error messages
- Proper HTTP status codes
- Global exception handling

### **Logging**
Extensive logging for:
- Request parameters
- Processing steps
- Calculated averages
- Error conditions

---

## Usage Examples

### **Daily Nutrient Summary - Single Day**
```bash
curl -X GET "http://localhost:8080/api/patient-meal-logs/daily-nutrient-summary?fromDate=27-09-2024&toDate=27-09-2024&timezone=Asia/Kolkata" \
  -H "Authorization: Bearer YOUR_TOKEN"
```

### **Heart Risk Summary - Date Range**
```bash
curl -X POST "http://localhost:8080/api/patient-heart-risk-metric-histories/heart-risk-summary" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "fromDate": "27-09-2024",
    "toDate": "27-09-2024",
    "timezone": "Asia/Kolkata"
  }'
```

### **Heart Risk Summary - Week Range**
```bash
curl -X POST "http://localhost:8080/api/patient-heart-risk-metric-histories/heart-risk-summary" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{
    "range": "week",
    "timezone": "Asia/Kolkata"
  }'
```
