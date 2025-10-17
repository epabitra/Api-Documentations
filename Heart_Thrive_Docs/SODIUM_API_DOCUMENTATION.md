# 📚 HeartThrive API Endpoints Documentation

## 🔍 Overview
This document provides comprehensive information about all the HeartThrive API endpoints, including required parameters, request/response formats, and usage examples.

---

## 📋 Table of Contents

1. [Sodium Hero Section](#1-sodium-hero-section)
2. [More Info about Sodium Hero Section](#2-more-info-about-sodium-hero-section)
3. [Sodium Dashboard Endpoint](#3-sodium-dashboard-endpoint)
4. [Daily Meal Intake](#4-daily-meal-intake)
5. [Daily Nutrient Summary (Day-by-Day Breakdown)](#5-daily-nutrient-summary-day-by-day-breakdown)
6. [Filter through Meal Types](#6-filter-through-meal-types)
7. [Browse Food Search Functionality](#7-browse-food-search-functionality)
8. [Get All Meal Types for Dropdown](#8-get-all-meal-types-for-dropdown)
9. [Add Meal (Pass the foodItemId)](#9-add-meal-pass-the-fooditemid)
10. [Get All Food Taken for Meal Type](#10-get-all-food-taken-for-meal-type)
11. [Meal Menu Picker List](#11-meal-menu-picker-list)
12. [Meal Menu Picker Edit Functionality](#12-meal-menu-picker-edit-functionality)
13. [Meal Menu Picker Delete Functionality](#13-meal-menu-picker-delete-functionality)
14. [Custom Meal Adding Process](#14-custom-meal-adding-process)
15. [Edit and Delete Related Endpoints](#15-edit-and-delete-related-endpoints)

---

## 1. Sodium Hero Section

### **Endpoint:** `GET /api/patient-meal-logs/nutrient-summary`

**Description:** Get nutrient summary for the current user within the specified time period. Shows aggregated nutrient data across all meal types.

### **Parameters:**
| Parameter | Type | Required | Description | Example |
|-----------|------|----------|-------------|---------|
| `fromDate` | String | Yes | Start date in format `dd-MM-yyyy` or `dd-MM-yyyy HH:mm` | `"11-09-2025 00:00"` |
| `toDate` | String | Yes | End date in format `dd-MM-yyyy` or `dd-MM-yyyy HH:mm` | `"11-09-2025 23:59"` |
| `mealTypeId` | Long | No | Filter by specific meal type ID | `13` |

### **Request Example:**
```http
GET /api/patient-meal-logs/nutrient-summary?fromDate=11-09-2025 00:00&toDate=11-09-2025 23:59
```

### **Response Structure:**
```json
{
  "nutrients": [
    {
      "name": "Sodium",
      "amount": 1500.50,
      "minValue": 500.0,
      "maxValue": 2300.0,
      "unitName": "mg"
    },
    {
      "name": "Calories",
      "amount": 2500.75,
      "minValue": 1800.0,
      "maxValue": 3000.0,
      "unitName": "kcal"
    }
  ]
}
```

---

## 2. More Info about Sodium Hero Section

### **Endpoint:** `GET /api/patient-meal-logs/nutrient-summary-by-meal-type`

**Description:** Get nutrient summary for the current user grouped by meal type. Shows nutrient data broken down by each meal type (Breakfast, Lunch, Dinner, etc.).

### **Parameters:**
| Parameter | Type | Required | Description | Example |
|-----------|------|----------|-------------|---------|
| `fromDate` | String | Yes | Start date in format `dd-MM-yyyy` or `dd-MM-yyyy HH:mm` | `"11-09-2025 00:00"` |
| `toDate` | String | Yes | End date in format `dd-MM-yyyy` or `dd-MM-yyyy HH:mm` | `"11-09-2025 23:59"` |

### **Request Example:**
```http
GET /api/patient-meal-logs/nutrient-summary-by-meal-type?fromDate=11-09-2025 00:00&toDate=11-09-2025 23:59
```

### **Response Structure:**
```json
{
  "mealTypeNutrientSummaries": [
    {
      "mealType": {
        "id": 1,
        "name": "Breakfast",
        "description": "Morning meal",
        "active": true
      },
      "nutrients": [
        {
          "name": "Sodium",
          "amount": 500.25,
          "minValue": 500.0,
          "maxValue": 2300.0,
          "unitName": "mg"
        }
      ]
    },
    {
      "mealType": {
        "id": 2,
        "name": "Lunch",
        "description": "Midday meal",
        "active": true
      },
      "nutrients": [
        {
          "name": "Sodium",
          "amount": 800.75,
          "minValue": 500.0,
          "maxValue": 2300.0,
          "unitName": "mg"
        }
      ]
    }
  ]
}
```

---

## 3. Sodium Dashboard Endpoint

### **Endpoints:**
- **Collect all day info:** `GET /api/patient-meal-logs/nutrient-summary?fromDate=11-09-2025 00:00&toDate=11-09-2025 23:59`
- **Collect Meal Type wise info:** `GET /api/patient-meal-logs/nutrient-summary-by-meal-type?fromDate=11-09-2025 00:00&toDate=11-09-2025 23:59`

**Description:** These are the same endpoints as #1 and #2, used for dashboard data collection.

---

## 4. Daily Meal Intake

### **Endpoint:** `GET /api/patient-meal-logs/nutrient-summary-by-meal-type`

**Description:** Same as endpoint #2. Used to display daily meal intake information grouped by meal type.

---

## 5. Daily Nutrient Summary (Day-by-Day Breakdown)

### **Endpoint:** `GET /api/patient-meal-logs/daily-nutrient-summary`

**Description:** Get nutrient summary for the current user with daily breakdown. Returns nutrient data for each individual day within the specified date range, allowing users to track their daily nutrient intake trends over time.

### **Parameters:**
|| Parameter | Type | Required | Description | Example |
||-----------|------|----------|-------------|---------|
|| `fromDate` | String | Yes | Start date in format `dd-MM-yyyy` or `dd-MM-yyyy HH:mm` | `"01-10-2025 00:00"` |
|| `toDate` | String | Yes | End date in format `dd-MM-yyyy` or `dd-MM-yyyy HH:mm` | `"07-10-2025 23:59"` |
|| `timezone` | String | Yes | User timezone (IANA format) for date interpretation | `"Asia/Kolkata"` |
|| `mealTypeId` | Long | No | Filter by specific meal type ID | `13` |

### **Request Example:**
```http
GET /api/patient-meal-logs/daily-nutrient-summary?fromDate=01-10-2025 00:00&toDate=07-10-2025 23:59&timezone=Asia/Kolkata
```

### **Response Structure:**
```json
{
  "dailySummaries": [
    {
      "date": "01-10-2025",
      "nutrients": [
        {
          "name": "Sodium",
          "amount": 1500.50,
          "minValue": 500.0,
          "maxValue": 2300.0,
          "unitName": "mg"
        },
        {
          "name": "Calories",
          "amount": 2200.00,
          "minValue": 1800.0,
          "maxValue": 3000.0,
          "unitName": "kcal"
        },
        {
          "name": "Protein",
          "amount": 65.5,
          "minValue": 50.0,
          "maxValue": 150.0,
          "unitName": "g"
        },
        {
          "name": "Carbs",
          "amount": 250.0,
          "minValue": 200.0,
          "maxValue": 400.0,
          "unitName": "g"
        },
        {
          "name": "Fats",
          "amount": 70.0,
          "minValue": 40.0,
          "maxValue": 100.0,
          "unitName": "g"
        }
      ]
    },
    {
      "date": "02-10-2025",
      "nutrients": [
        {
          "name": "Sodium",
          "amount": 1800.75,
          "minValue": 500.0,
          "maxValue": 2300.0,
          "unitName": "mg"
        },
        {
          "name": "Calories",
          "amount": 2400.00,
          "minValue": 1800.0,
          "maxValue": 3000.0,
          "unitName": "kcal"
        }
      ]
    }
  ],
  "fromDate": "01-10-2025",
  "toDate": "07-10-2025",
  "timezone": "Asia/Kolkata"
}
```

### **Key Features:**

#### **Daily Breakdown**
- Each day in the date range gets its own summary
- Nutrients are calculated separately for each day
- Shows trends over time (increasing/decreasing intake)

#### **Timezone Handling**
- Timezone parameter is **required** (unlike other endpoints)
- Ensures accurate date interpretation across different timezones
- Frontend dates are properly converted to backend dates using specified timezone

#### **Complete Nutrient Set**
- Returns all tracked nutrients (Sodium, Calories, Protein, Carbs, Fats)
- Shows min/max recommended values for each nutrient
- Includes unit names for proper display

#### **Optional Filtering**
- Can filter by `mealTypeId` to see nutrients from specific meal types only
- Example: Get daily sodium from breakfast only

### **Use Cases:**

#### **Use Case 1: Weekly Sodium Trend**
Track sodium intake over a week to identify high-sodium days.
```http
GET /api/patient-meal-logs/daily-nutrient-summary?fromDate=01-10-2025&toDate=07-10-2025&timezone=Asia/Kolkata
```

#### **Use Case 2: Monthly Calorie Tracking**
Monitor daily calorie intake for a month to maintain healthy diet.
```http
GET /api/patient-meal-logs/daily-nutrient-summary?fromDate=01-10-2025&toDate=31-10-2025&timezone=Asia/Kolkata
```

#### **Use Case 3: Breakfast Sodium Analysis**
Analyze daily sodium intake from breakfast meals only.
```http
GET /api/patient-meal-logs/daily-nutrient-summary?fromDate=01-10-2025&toDate=07-10-2025&timezone=Asia/Kolkata&mealTypeId=1
```

### **Validation Rules:**

1. **fromDate is required** - Returns error if missing
2. **toDate is required** - Returns error if missing
3. **timezone is required** - Returns error if missing
4. **fromDate ≤ toDate** - Returns error if fromDate is after toDate
5. **Valid timezone** - Must be valid IANA timezone (e.g., "Asia/Kolkata", "America/New_York")

### **Error Responses:**

#### Missing Required Field (fromDate)
```json
{
  "status": 400,
  "title": "Bad Request",
  "detail": "fromDate is required for daily nutrient summary",
  "entityName": "patientMealLog",
  "errorKey": "fromDateRequired"
}
```

#### Missing Required Field (toDate)
```json
{
  "status": 400,
  "title": "Bad Request",
  "detail": "toDate is required for daily nutrient summary",
  "entityName": "patientMealLog",
  "errorKey": "toDateRequired"
}
```

#### Missing Required Field (timezone)
```json
{
  "status": 400,
  "title": "Bad Request",
  "detail": "timezone is required for daily nutrient summary",
  "entityName": "patientMealLog",
  "errorKey": "timezoneRequired"
}
```

#### Invalid Timezone Format
```json
{
  "status": 400,
  "title": "Bad Request",
  "detail": "Invalid timezone format. Please provide a valid IANA timezone (e.g., Asia/Kolkata)",
  "entityName": "patientMealLog",
  "errorKey": "invalidTimezone"
}
```

#### Invalid Date Range
```json
{
  "status": 400,
  "title": "Bad Request",
  "detail": "fromDate cannot be greater than toDate",
  "entityName": "patientMealLog",
  "errorKey": "fromDateAfterToDate"
}
```

### **Response Features:**

#### **Zero Values for Missing Nutrients**
- If no intake recorded for a nutrient on a specific day, amount shows `0`
- Min/Max values still displayed for reference
- Helps identify days with insufficient nutrient intake

#### **Sorted by Date**
- Days are returned in chronological order
- Nutrients within each day are alphabetically sorted
- Easy to track trends over time

#### **Efficient Data Retrieval**
- Single database query for entire date range
- In-memory aggregation by day
- Optimized for performance

### **Comparison: Daily Summary vs Regular Summary**

|| Feature | Daily Summary | Regular Summary |
||---------|---------------|-----------------|
|| **Endpoint** | `/daily-nutrient-summary` | `/nutrient-summary` |
|| **Breakdown** | Per day | Total for period |
|| **Use Case** | Track trends over time | Overall intake for period |
|| **Response Size** | Larger (one object per day) | Smaller (single total) |
|| **Timezone Param** | Required | Not required |
|| **Best For** | Charts, trends, analysis | Dashboard widgets, totals |

### **Example Frontend Usage:**

#### **Line Chart - Sodium Trend**
```javascript
// Call daily-nutrient-summary for last 7 days
const response = await fetch('/api/patient-meal-logs/daily-nutrient-summary?fromDate=01-10-2025&toDate=07-10-2025&timezone=Asia/Kolkata');
const data = await response.json();

// Extract sodium values for chart
const chartData = data.dailySummaries.map(day => ({
  date: day.date,
  sodium: day.nutrients.find(n => n.name === 'Sodium')?.amount || 0
}));

// Render line chart showing sodium trend over 7 days
```

#### **Table - Daily Nutrient Breakdown**
```javascript
// Display in table format:
// Date       | Sodium | Calories | Protein | Carbs | Fats
// 01-10-2025 | 1500mg | 2200kcal | 65g     | 250g  | 70g
// 02-10-2025 | 1800mg | 2400kcal | 70g     | 280g  | 75g
```

### **Notes:**

- **Timezone critical:** Unlike other endpoints, this requires timezone for accurate daily grouping
- **All nutrients included:** Returns data for all tracked nutrients (Sodium, Calories, Protein, Carbs, Fats)
- **Day-level granularity:** Each day is a separate object in the response
- **Performance optimized:** Single database call for entire range, then in-memory grouping
- **Meal type filtering:** Optional mealTypeId filter to analyze specific meal types
- **Min/Max values:** Helps users identify if they're within recommended ranges
- **Zero handling:** Days with no intake show 0 for all nutrients

---

## 6. Filter through Meal Types

### **Endpoints:**
- **Get meal types:** `GET /api/app-param-values/get-meal-types`
- **Filter by meal type:** `GET /api/patient-meal-logs/nutrient-summary?fromDate=11-09-2025 00:00&                      toDate=11-09-2025 23:59&mealTypeId=13`

### **Get Meal Types Request:**
```http
GET /api/app-param-values/get-meal-types
```

### **Get Meal Types Response:**
```json
[
  {
    "id": 1,
    "name": "Breakfast",
    "description": "Morning meal",
    "active": true,
    "group": {
      "id": 1,
      "name": "Meal Type",
      "description": "Types of meals"
    }
  },
  {
    "id": 2,
    "name": "Lunch",
    "description": "Midday meal",
    "active": true,
    "group": {
      "id": 1,
      "name": "Meal Type",
      "description": "Types of meals"
    }
  }
]
```

### **Filter by Meal Type Request:**
```http
GET /api/patient-meal-logs/nutrient-summary?fromDate=11-09-2025 00:00&toDate=11-09-2025 23:59&mealTypeId=13
```

---

## 6. Browse Food Search Functionality

### **Endpoint:** `GET /api/food-items/with-nutrients`

**Description:** Search and browse food items with their nutrient information. Currently searches from the database only.

### **Parameters:**
| Parameter | Type | Required | Description | Example |
|-----------|------|----------|-------------|---------|
| `page` | Integer | No | Page number (0-based) | `0` |
| `size` | Integer | No | Number of items per page | `20` |
| `mealTypeId` | Long | No | Filter by food type (meal type) | `13` |
| `foodItemName` | String | No | Search by food item name (partial match) | `"chicken"` |

### **Request Example:**
```http
GET /api/food-items/with-nutrients?page=0&size=20&mealTypeId=13&foodItemName=chicken
```

### **Response Structure:**
```json
{
  "content": [
    {
      "id": 1,
      "uuid": "abc-123-def",
      "fdcId": 12345,
      "description": "Grilled Chicken Breast",
      "brandName": "Generic",
      "nutrients": [
        {
          "name": "Protein",
          "amount": 25.5,
          "minValue": 10.0,
          "maxValue": 50.0,
          "unitName": "g"
        },
        {
          "name": "Sodium",
          "amount": 150.0,
          "minValue": 0.0,
          "maxValue": 500.0,
          "unitName": "mg"
        }
      ]
    }
  ],
  "pageable": {
    "pageNumber": 0,
    "pageSize": 20
  },
  "totalElements": 100,
  "totalPages": 5
}
```

---

## 7. Get All Meal Types for Dropdown

### **Endpoint:** `GET /api/app-param-values/get-meal-types`

**Description:** Get all active meal types for dropdown selection.

### **Request Example:**
```http
GET /api/app-param-values/get-meal-types
```

### **Response:** Same as endpoint #5 Get Meal Types Response

---

## 8. Add Meal (Pass the foodItemId)

### **Endpoint:** `POST /api/patient-meal-logs/create-meal`

**Description:** Create a new meal log with an existing food item.

### **Request Body:**
```json
{
  "foodItemId": 123,
  "mealTypeId": 1,
  "quantity": "1 cup",
  "logDate": "2025-09-11",
  "addToMealMenu": true
}
```

### **Request Parameters:**
| Parameter | Type | Required | Description | Example |
|-----------|------|----------|-------------|---------|
| `foodItemId` | Long | Yes | ID of existing food item | `123` |
| `mealTypeId` | Long | Yes | ID of meal type | `1` |
| `quantity` | String | Yes | Quantity consumed | `"1 cup"` |
| `logDate` | String | Yes | Date of meal (yyyy-MM-dd) | `"2025-09-11"` |
| `addToMealMenu` | Boolean | No | Save to meal menu for future use | `true` |

### **Response Structure:**
```json
{
  "success": true,
  "message": "Meal log created successfully",
  "data": "Meal log created successfully with food item: Grilled Chicken Breast"
}
```

---

## 9. Get All Food Taken for Meal Type

### **Endpoint:** `GET /api/patient-meal-logs/user-meal-logs`

**Description:** Get all meal logs for the current user with their associated nutrients, filtered by date range and optionally by meal type.

### **Parameters:**
| Parameter | Type | Required | Description | Example |
|-----------|------|----------|-------------|---------|
| `fromDate` | String | Yes | Start date in format `dd-MM-yyyy` or `dd-MM-yyyy HH:mm` | `"11-09-2025 00:00"` |
| `toDate` | String | Yes | End date in format `dd-MM-yyyy` or `dd-MM-yyyy HH:mm` | `"11-09-2025 23:59"` |
| `page` | Integer | No | Page number (0-based) | `0` |
| `size` | Integer | No | Number of items per page | `20` |
| `mealTypeId` | Long | No | Filter by meal type ID | `1` |

### **Request Example:**
```http
GET /api/patient-meal-logs/user-meal-logs?fromDate=11-09-2025 00:00&toDate=11-09-2025 23:59&page=0&size=20
```

### **Response Structure:**
```json
{
  "content": [
    {
      "id": 1,
      "uuid": "meal-log-123",
      "logDate": "2025-09-11",
      "quantity": "1 cup",
      "mealType": {
        "id": 1,
        "name": "Breakfast",
        "description": "Morning meal"
      },
      "foodItemWithNutrients": {
        "id": 123,
        "description": "Grilled Chicken Breast",
        "brandName": "Generic",
        "nutrients": [
          {
            "name": "Protein",
            "amount": 25.5,
            "unitName": "g"
          }
        ]
      }
    }
  ],
  "pageable": {
    "pageNumber": 0,
    "pageSize": 20
  },
  "totalElements": 50,
  "totalPages": 3
}
```

---

## 10. Meal Menu Picker List

### **Endpoint:** `GET /api/patient-meal-menus/with-nutrients`

**Description:** Get all patient meal menus for the current user with their latest menu items and nutrients.

### **Parameters:**
| Parameter | Type | Required | Description | Example |
|-----------|------|----------|-------------|---------|
| `fromDate` | String | No | Start date in format `dd-MM-yyyy` or `dd-MM-yyyy HH:mm` | `"11-09-2025 00:00"` |
| `toDate` | String | No | End date in format `dd-MM-yyyy` or `dd-MM-yyyy HH:mm` | `"11-09-2025 23:59"` |
| `search` | String | No | Search term for meal menu name or food item name | `"chicken"` |
| `mealTypeId` | Long | No | Filter by meal type ID | `13` |
| `page` | Integer | No | Page number (0-based) | `0` |
| `size` | Integer | No | Number of items per page | `20` |

### **Request Example:**
```http
GET /api/patient-meal-menus/with-nutrients?fromDate=11-09-2025 00:00&toDate=11-09-2025 23:59&page=0&size=20
```

### **Response Structure:**
```json
{
  "content": [
    {
      "id": 1,
      "uuid": "menu-123",
      "name": "My Breakfast Menu",
      "active": true,
      "createdDate": "2025-09-11T08:00:00Z",
      "latestQuantity": "1 cup",
      "latestFoodItemWithNutrients": {
        "id": 123,
        "description": "Grilled Chicken Breast",
        "brandName": "Generic",
        "nutrients": [
          {
            "name": "Protein",
            "amount": 25.5,
            "unitName": "g"
          }
        ]
      }
    }
  ],
  "pageable": {
    "pageNumber": 0,
    "pageSize": 20
  },
  "totalElements": 10,
  "totalPages": 1
}
```

---

## 11. Meal Menu Picker Edit Functionality

**Status:** ✅ **IMPLEMENTED** - See [Section 14.3](#143-edit-patient-meal-menu)

**Description:** Edit existing meal menus with proper security validation and ownership verification.

**Endpoint:** `PUT /api/patient-meal-menus/edit`

---

## 12. Meal Menu Picker Delete Functionality

**Status:** ✅ **IMPLEMENTED** - See [Section 14.4](#144-delete-patient-meal-menu)

**Description:** Delete meal menus permanently with proper cleanup and security validation.

**Endpoint:** `DELETE /api/patient-meal-menus/{id}`

---

## 13. Custom Meal Adding Process

### **Endpoint:** `POST /api/patient-meal-logs/create-meal`

**Description:** Create a custom meal log without passing foodItemId. This creates a new food item and meal log.

### **Request Body:**
```json
{
  "mealTypeId": 1,
  "quantity": "1 cup",
  "logDate": "2025-09-11",
  "foodItemName": "Custom Grilled Chicken",
  "brandName": "Homemade",
  "foodCategoryId": 1,
  "foodTypeId": 13,
  "sodiumAmount": 150.0,
  "caloriesAmount": 200.0,
  "carbsAmount": 5.0,
  "proteinAmount": 25.0,
  "fatsAmount": 8.0,
  "addToMealMenu": true
}
```

### **Request Parameters:**
| Parameter | Type | Required | Description | Example |
|-----------|------|----------|-------------|---------|
| `mealTypeId` | Long | Yes | ID of meal type | `1` |
| `quantity` | String | Yes | Quantity consumed | `"1 cup"` |
| `logDate` | String | Yes | Date of meal (yyyy-MM-dd) | `"2025-09-11"` |
| `foodItemName` | String | Yes | Name of the custom food item | `"Custom Grilled Chicken"` |
| `brandName` | String | No | Brand name | `"Homemade"` |
| `foodCategoryId` | Long | No | Food category ID | `1` |
| `foodTypeId` | Long | No | Food type ID | `13` |
| `sodiumAmount` | BigDecimal | No | Sodium amount in mg | `150.0` |
| `caloriesAmount` | BigDecimal | No | Calories amount in kcal | `200.0` |
| `carbsAmount` | BigDecimal | No | Carbohydrates amount in g | `5.0` |
| `proteinAmount` | BigDecimal | No | Protein amount in g | `25.0` |
| `fatsAmount` | BigDecimal | No | Fats amount in g | `8.0` |
| `addToMealMenu` | Boolean | No | Save to meal menu for future use | `true` |

### **Response Structure:**
```json
{
  "success": true,
  "message": "Meal log created successfully",
  "data": "Meal log created successfully with food item: Custom Grilled Chicken"
}
```

---

## 14. Edit and Delete Related Endpoints

This section covers the edit and delete functionality for meal logs and meal menus, providing users with the ability to modify and remove their existing data.

---

### 14.1. Edit Meal Log

### **Endpoint:** `PUT /api/patient-meal-logs/edit-meal/{id}`

**Description:** Edit an existing meal log. Only allows editing of `mealTypeId` and `quantity` fields. If quantity is changed, the system automatically recalculates and updates associated nutrient intake records.

### **Path Parameters:**
| Parameter | Type | Required | Description | Example |
|-----------|------|----------|-------------|---------|
| `id` | Long | Yes | ID of the meal log to edit | `123` |

### **Request Body:**
```json
{
  "mealTypeId": 2,
  "quantity": "1.5"
}
```

### **Request Parameters:**
| Parameter | Type | Required | Description | Example |
|-----------|------|----------|-------------|---------|
| `mealTypeId` | Long | Yes | New meal type ID (must reference existing meal type) | `2` |
| `quantity` | String | Yes | New quantity (max 255 characters) | `"1.5"` |

### **Request Example:**
```http
PUT /api/patient-meal-logs/edit-meal/123
Content-Type: application/json

{
  "mealTypeId": 2,
  "quantity": "1.5"
}
```

### **Response Structure:**
```json
{
  "success": true,
  "message": "Meal log updated successfully",
  "data": "Meal log with ID 123 has been successfully updated"
}
```

### **Features:**
- ✅ User ownership validation (users can only edit their own meal logs)
- ✅ Automatic nutrient recalculation when quantity changes
- ✅ Audit trail maintenance (updates lastModifiedBy and lastModifiedDate)
- ✅ Comprehensive error handling with meaningful messages
- ✅ Validation of meal type existence and meal log status

---

### 14.2. Soft Delete Meal Log

### **Endpoint:** `DELETE /api/patient-meal-logs/{id}`

**Description:** Soft delete a meal log and its associated nutrient intake records. Sets the `active` flag to `false` instead of performing a hard deletion, preserving data integrity and audit trails.

### **Path Parameters:**
| Parameter | Type | Required | Description | Example |
|-----------|------|----------|-------------|---------|
| `id` | Long | Yes | ID of the meal log to delete | `123` |

### **Request Example:**
```http
DELETE /api/patient-meal-logs/123
```

### **Response Structure:**
```json
{
  "success": true,
  "message": "Meal log deleted successfully",
  "data": "Meal log with ID 123 has been successfully soft deleted"
}
```

### **Features:**
- ✅ Soft deletion (preserves data for audit purposes)
- ✅ User ownership validation
- ✅ Associated nutrient intake records cleanup
- ✅ Audit trail maintenance
- ✅ Comprehensive error handling

---

### 14.3. Edit Patient Meal Menu

### **Endpoint:** `PUT /api/patient-meal-menus/edit`

**Description:** Edit an existing patient meal menu. Allows updating of quantity and meal menu preferences with proper security validation and ownership verification.

### **Request Body:**
```json
{
  "mealMenuId": 456,
  "quantity": "2",
  "addToMealMenu": true
}
```

### **Request Parameters:**
| Parameter | Type | Required | Description | Example |
|-----------|------|----------|-------------|---------|
| `mealMenuId` | Long | Yes | ID of the meal menu to edit | `456` |
| `quantity` | String | No | New quantity for the meal menu | `"2"` |
| `addToMealMenu` | Boolean | No | Whether to add/keep in meal menu | `true` |

### **Request Example:**
```http
PUT /api/patient-meal-menus/edit
Content-Type: application/json

{
  "mealMenuId": 456,
  "quantity": "2",
  "addToMealMenu": true
}
```

### **Response Structure:**
```json
{
  "id": 456,
  "uuid": "menu-uuid-123",
  "name": "My Breakfast Menu",
  "active": true,
  "createdDate": "2025-09-11T08:00:00Z",
  "lastModifiedDate": "2025-09-11T10:30:00Z",
  "patient": {
    "id": 1,
    "login": "patient@example.com"
  },
  "mealType": {
    "id": 1,
    "name": "Breakfast",
    "description": "Morning meal"
  }
}
```

### **Features:**
- ✅ Security validation and ownership verification
- ✅ Active menu status validation
- ✅ Comprehensive input validation
- ✅ Audit trail maintenance
- ✅ Error handling with meaningful messages

---

### 14.4. Delete Patient Meal Menu

### **Endpoint:** `DELETE /api/patient-meal-menus/{id}`

**Description:** Delete a patient meal menu permanently. This performs a hard deletion and removes the meal menu from the system.

### **Path Parameters:**
| Parameter | Type | Required | Description | Example |
|-----------|------|----------|-------------|---------|
| `id` | Long | Yes | ID of the meal menu to delete | `456` |

### **Request Example:**
```http
DELETE /api/patient-meal-menus/456
```

### **Response Structure:**
```json
"Meal menu with ID 456 has been successfully deleted"
```

### **Features:**
- ✅ Hard deletion (permanent removal)
- ✅ User ownership validation
- ✅ Proper cleanup of associated data
- ✅ Comprehensive error handling
- ✅ Security validation

---

### 14.5. Edit and Delete Endpoints - Common Features

#### **Security & Validation:**
- All endpoints require authentication
- User ownership validation ensures users can only modify their own data
- Input validation with meaningful error messages
- Active status validation for data integrity

#### **Error Handling:**
All endpoints return appropriate HTTP status codes:
- `200 OK` - Success
- `400 Bad Request` - Invalid request parameters
- `401 Unauthorized` - Authentication required
- `404 Not Found` - Resource not found
- `500 Internal Server Error` - Server error

#### **Error Response Format:**
```json
{
  "success": false,
  "message": "Error description",
  "data": null
}
```

#### **Common Error Scenarios:**
- **Meal log not found:** When trying to edit/delete a non-existent meal log
- **Access denied:** When trying to modify another user's data
- **Invalid meal type:** When providing a non-existent meal type ID
- **Inactive resource:** When trying to modify already inactive data
- **Validation errors:** When required fields are missing or invalid

---

## 📝 Common Response Structures

### **NutrientSummaryDTO:**
```json
{
  "name": "Sodium",
  "amount": 1500.50,
  "minValue": 500.0,
  "maxValue": 2300.0,
  "unitName": "mg"
}
```

### **AppParamValueDTO (Meal Type):**
```json
{
  "id": 1,
  "uuid": "meal-type-123",
  "name": "Breakfast",
  "description": "Morning meal",
  "active": true,
  "createdBy": "system",
  "createdDate": "2025-01-01T00:00:00Z",
  "lastModifiedBy": "system",
  "lastModifiedDate": "2025-01-01T00:00:00Z",
  "group": {
    "id": 1,
    "name": "Meal Type",
    "description": "Types of meals"
  }
}
```

### **ObjectResponse (Success/Error):**
```json
{
  "success": true,
  "message": "Operation completed successfully",
  "data": "Result data here"
}
```

### **EditMealLogRequestDTO:**
```json
{
  "mealTypeId": 2,
  "quantity": "1.5 cups"
}
```

### **EditPatientMealMenuRequestDTO:**
```json
{
  "mealMenuId": 456,
  "quantity": "2 servings",
  "addToMealMenu": true
}
```

---

## 🔧 Error Handling

All endpoints return appropriate HTTP status codes:
- `200 OK` - Success
- `400 Bad Request` - Invalid request parameters
- `401 Unauthorized` - Authentication required
- `404 Not Found` - Resource not found
- `500 Internal Server Error` - Server error

Error responses follow the ObjectResponse structure:
```json
{
  "success": false,
  "message": "Error description",
  "data": null
}
```

---

## 📅 Date Format Notes

- **Input Format:** `dd-MM-yyyy` or `dd-MM-yyyy HH:mm` (e.g., `"11-09-2025 00:00"`)
- **Output Format:** ISO 8601 format (e.g., `"2025-09-11T00:00:00Z"`)
- **Log Date Format:** `yyyy-MM-dd` (e.g., `"2025-09-11"`)

---

## 🔐 Authentication

All endpoints require authentication. Include the appropriate authentication headers in your requests (JWT token, API key, etc.).

---

## 📊 Quick Reference

### **Most Used Endpoints:**
1. **Get Nutrient Summary:** `GET /api/patient-meal-logs/nutrient-summary`
2. **Get Meal Type Summary:** `GET /api/patient-meal-logs/nutrient-summary-by-meal-type`
3. **Get Daily Nutrient Summary:** `GET /api/patient-meal-logs/daily-nutrient-summary`
4. **Get Meal Types:** `GET /api/app-param-values/get-meal-types`
5. **Search Food Items:** `GET /api/food-items/with-nutrients`
6. **Create Meal:** `POST /api/patient-meal-logs/create-meal`
7. **Get User Meal Logs:** `GET /api/patient-meal-logs/user-meal-logs`
8. **Get Meal Menus:** `GET /api/patient-meal-menus/with-nutrients`
9. **Edit Meal Log:** `PUT /api/patient-meal-logs/edit-meal/{id}`
10. **Delete Meal Log:** `DELETE /api/patient-meal-logs/{id}`
11. **Edit Meal Menu:** `PUT /api/patient-meal-menus/edit`
12. **Delete Meal Menu:** `DELETE /api/patient-meal-menus/{id}`

### **Common Parameters:**
- `fromDate` / `toDate` - Date range filtering
- `timezone` - User timezone (required for daily-nutrient-summary)
- `mealTypeId` - Filter by meal type
- `page` / `size` - Pagination
- `search` - Text search

---

## 🎯 Endpoint Summary by Category

### **Nutrient Analysis Endpoints:**
- `GET /nutrient-summary` - Total nutrients for a period
- `GET /nutrient-summary-by-meal-type` - Nutrients grouped by meal type
- `GET /daily-nutrient-summary` - Nutrients broken down by day (requires timezone)

### **Meal Management Endpoints:**
- `POST /create-meal` - Add new meal (existing or custom food item)
- `GET /user-meal-logs` - List all user's meal logs
- `PUT /edit-meal/{id}` - Edit existing meal log
- `DELETE /{id}` - Delete meal log (soft delete)

### **Meal Menu Endpoints:**
- `GET /patient-meal-menus/with-nutrients` - List user's saved meal menus
- `PUT /patient-meal-menus/edit` - Edit meal menu
- `DELETE /patient-meal-menus/{id}` - Delete meal menu

### **Lookup/Reference Endpoints:**
- `GET /app-param-values/get-meal-types` - Get all meal types for dropdown
- `GET /food-items/with-nutrients` - Search and browse food database

---

**Last Updated:** October 15, 2025  
**API Version:** 2.1  
**Status:** Production Ready
