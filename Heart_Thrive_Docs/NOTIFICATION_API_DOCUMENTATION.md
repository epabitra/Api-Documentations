# Notification API Documentation

## Table of Contents
1. [Get User Notifications](#1-get-user-notifications) - Retrieve paginated list of notifications for the authenticated user
2. [Mark Notifications as Seen](#2-mark-notifications-as-seen) - Mark multiple notifications as seen using UUIDs
3. [Mark Notifications as Unseen](#3-mark-notifications-as-unseen) - Mark multiple notifications as unseen using UUIDs

---

## Overview

This API provides endpoints for managing user notifications in the Heart Thrive application. It allows users to:
- Retrieve their notifications with pagination support
- Filter notifications by seen/unseen status
- Mark multiple notifications as seen or unseen in bulk operations

All endpoints require authentication and automatically filter results based on the authenticated user.

---

## Base URL

```
Development: http://localhost:8080/api
Production: [To be configured based on deployment]
```

All notification endpoints are prefixed with `/notifications`:
```
Base Path: /api/notifications
```

---

## Authentication

All endpoints require **JWT (JSON Web Token) authentication**. The token must be included in the request headers:

```
Authorization: Bearer <your-jwt-token>
```

**Important Notes:**
- All endpoints automatically filter results based on the authenticated user
- Users can only access their own notifications
- Attempting to mark notifications that don't belong to the authenticated user will result in an error

---

## Common Response Formats

### Success Response (200 OK)
```json
{
  "data": [...],
  "headers": {
    "X-Total-Count": "100",
    "Link": "<http://localhost:8080/api/notifications?page=0&size=20>; rel=\"first\", ..."
  }
}
```

### Error Response (400 Bad Request)
```json
{
  "type": "https://www.jhipster.tech/problem/constraint-violation",
  "title": "Bad Request",
  "status": 400,
  "detail": "Notification UUIDs list cannot be null or empty",
  "message": "error.validation"
}
```

### Error Response (401 Unauthorized)
```json
{
  "type": "https://www.jhipster.tech/problem/unauthorized",
  "title": "Unauthorized",
  "status": 401,
  "detail": "Full authentication is required to access this resource"
}
```

### Error Response (500 Internal Server Error)
```json
{
  "type": "https://www.jhipster.tech/problem/server-error",
  "title": "Internal Server Error",
  "status": 500,
  "detail": "An unexpected error occurred"
}
```

---

## Pagination

The GET notifications endpoint supports pagination using standard Spring Data pagination parameters.

### Pagination Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `page` | Integer | No | 0 | Page number (0-indexed) |
| `size` | Integer | No | 20 | Number of items per page |
| `sort` | String | No | `createdAt,desc` | Sort criteria (format: `field,direction`) |

### Sort Format
- Format: `field,direction`
- Direction: `asc` (ascending) or `desc` (descending)
- Multiple sorts: `sort=createdAt,desc&sort=uuid,asc`
- Default: `createdAt,desc` (newest first)

### Pagination Response Headers

The API returns pagination metadata in response headers:

| Header | Description |
|--------|-------------|
| `X-Total-Count` | Total number of items across all pages |
| `Link` | Pagination links (first, prev, next, last) |

### Example Pagination Headers
```
X-Total-Count: 150
Link: <http://localhost:8080/api/notifications?page=0&size=20>; rel="first",
      <http://localhost:8080/api/notifications?page=1&size=20>; rel="prev",
      <http://localhost:8080/api/notifications?page=3&size=20>; rel="next",
      <http://localhost:8080/api/notifications?page=7&size=20>; rel="last"
```

---

## Endpoints

### 1. Get User Notifications

Retrieve paginated list of notifications for the authenticated user.

#### Endpoint
```
GET /api/notifications
```

#### Query Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `includeSeen` | Boolean | No | `true` | Whether to include seen notifications. Set to `false` to get only unseen notifications |
| `page` | Integer | No | 0 | Page number (0-indexed) |
| `size` | Integer | No | 20 | Number of items per page |
| `sort` | String | No | `createdAt,desc` | Sort criteria |

#### Request Example

**Get all notifications (first page, 20 items):**
```http
GET /api/notifications?includeSeen=true&page=0&size=20&sort=createdAt,desc
Authorization: Bearer <your-jwt-token>
```

**Get only unseen notifications:**
```http
GET /api/notifications?includeSeen=false&page=0&size=10
Authorization: Bearer <your-jwt-token>
```

**Get second page with custom page size:**
```http
GET /api/notifications?page=1&size=50
Authorization: Bearer <your-jwt-token>
```

#### Response

**Status Code:** `200 OK`

**Response Headers:**
```
X-Total-Count: 150
Link: <http://localhost:8080/api/notifications?page=0&size=20>; rel="first", ...
Content-Type: application/json
```

**Response Body:**
```json
[
  {
    "uuid": "550e8400-e29b-41d4-a716-446655440000",
    "userId": 123,
    "seen": false,
    "seenAt": null,
    "active": true,
    "createdBy": "system",
    "lastModifiedBy": null,
    "createdAt": "2024-01-15T10:30:00Z",
    "lastModifiedAt": "2024-01-15T10:30:00Z"
  },
  {
    "uuid": "550e8400-e29b-41d4-a716-446655440001",
    "userId": 123,
    "seen": true,
    "seenAt": "2024-01-15T11:00:00Z",
    "active": true,
    "createdBy": "system",
    "lastModifiedBy": "user",
    "createdAt": "2024-01-15T09:15:00Z",
    "lastModifiedAt": "2024-01-15T11:00:00Z"
  }
]
```

#### Response Fields

| Field | Type | Description |
|-------|------|-------------|
| `uuid` | String | Universally unique identifier (36 characters) - **Use this for all operations** |
| `userId` | Long | ID of the user who owns this notification |
| `seen` | Boolean | Whether the notification has been seen (`true`) or not (`false`) |
| `seenAt` | String (ISO 8601) | Timestamp when the notification was marked as seen. `null` if not seen yet |
| `active` | Boolean | Whether the notification is active |
| `createdBy` | String | User/system who created the notification |
| `lastModifiedBy` | String | User/system who last modified the notification. `null` if never modified |
| `createdAt` | String (ISO 8601) | Timestamp when the notification was created |
| `lastModifiedAt` | String (ISO 8601) | Timestamp when the notification was last modified |

**Important:** Only `uuid` is exposed to the frontend. All operations (mark as seen/unseen) must use the `uuid` field.

#### Error Responses

| Status Code | Description |
|-------------|-------------|
| `401 Unauthorized` | Missing or invalid authentication token |
| `500 Internal Server Error` | Server error occurred |

---

### 2. Mark Notifications as Seen

Mark multiple notifications as seen for the authenticated user.

#### Endpoint
```
PUT /api/notifications/mark-seen
```

#### Request Headers
```
Authorization: Bearer <your-jwt-token>
Content-Type: application/json
```

#### Request Body

```json
{
  "notificationUuids": [
    "550e8400-e29b-41d4-a716-446655440000",
    "550e8400-e29b-41d4-a716-446655440001",
    "550e8400-e29b-41d4-a716-446655440002"
  ]
}
```

**Request Body Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `notificationUuids` | Array of String | Yes | List of notification app status UUIDs to mark as seen. Must not be empty |

**Validation Rules:**
- `notificationUuids` must not be `null`
- `notificationUuids` must not be empty (at least one UUID required)
- Each UUID must be a valid UUID format (36 characters)
- Each UUID must not be blank
- All notifications must belong to the authenticated user

#### Request Example

```http
PUT /api/notifications/mark-seen
Authorization: Bearer <your-jwt-token>
Content-Type: application/json

{
  "notificationUuids": [
    "550e8400-e29b-41d4-a716-446655440000",
    "550e8400-e29b-41d4-a716-446655440001",
    "550e8400-e29b-41d4-a716-446655440002"
  ]
}
```

#### Response

**Status Code:** `200 OK`

**Response Body:** Empty (no content)

#### Success Behavior

- All specified notifications are marked as seen
- `seen` field is set to `true`
- `seenAt` field is set to the current timestamp
- `lastModifiedAt` field is updated to the current timestamp
- Only notifications belonging to the authenticated user are updated

#### Error Responses

| Status Code | Description | Response Body Example |
|-------------|-------------|---------------------|
| `400 Bad Request` | Invalid request (null/empty list, invalid format) | `{"type": "...", "title": "Bad Request", "status": 400, "detail": "Notification UUIDs list cannot be null or empty"}` |
| `400 Bad Request` | Some notifications not found or don't belong to user | `{"type": "...", "title": "Bad Request", "status": 400, "detail": "Some notifications were not found or do not belong to user: 123"}` |
| `401 Unauthorized` | Missing or invalid authentication token | `{"type": "...", "title": "Unauthorized", "status": 401, "detail": "Full authentication is required..."}` |
| `500 Internal Server Error` | Server error occurred | `{"type": "...", "title": "Internal Server Error", "status": 500, "detail": "An unexpected error occurred"}` |

**Important Notes:**
- The operation is **atomic** - either all notifications are updated or none
- If any notification UUID in the list doesn't exist or doesn't belong to the user, the entire operation fails
- The error message will indicate that some notifications were not found or don't belong to the user

---

### 3. Mark Notifications as Unseen

Mark multiple notifications as unseen for the authenticated user.

#### Endpoint
```
PUT /api/notifications/mark-unseen
```

#### Request Headers
```
Authorization: Bearer <your-jwt-token>
Content-Type: application/json
```

#### Request Body

```json
{
  "notificationUuids": [
    "550e8400-e29b-41d4-a716-446655440000",
    "550e8400-e29b-41d4-a716-446655440001",
    "550e8400-e29b-41d4-a716-446655440002"
  ]
}
```

**Request Body Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `notificationUuids` | Array of String | Yes | List of notification app status UUIDs to mark as unseen. Must not be empty |

**Validation Rules:**
- `notificationUuids` must not be `null`
- `notificationUuids` must not be empty (at least one UUID required)
- Each UUID must be a valid UUID format (36 characters)
- Each UUID must not be blank
- All notifications must belong to the authenticated user

#### Request Example

```http
PUT /api/notifications/mark-unseen
Authorization: Bearer <your-jwt-token>
Content-Type: application/json

{
  "notificationUuids": [
    "550e8400-e29b-41d4-a716-446655440000",
    "550e8400-e29b-41d4-a716-446655440001",
    "550e8400-e29b-41d4-a716-446655440002"
  ]
}
```

#### Response

**Status Code:** `200 OK`

**Response Body:** Empty (no content)

#### Success Behavior

- All specified notifications are marked as unseen
- `seen` field is set to `false`
- `seenAt` field is set to `null`
- `lastModifiedAt` field is updated to the current timestamp
- Only notifications belonging to the authenticated user are updated

#### Error Responses

| Status Code | Description | Response Body Example |
|-------------|-------------|---------------------|
| `400 Bad Request` | Invalid request (null/empty list, invalid format) | `{"type": "...", "title": "Bad Request", "status": 400, "detail": "Notification UUIDs list cannot be null or empty"}` |
| `400 Bad Request` | Some notifications not found or don't belong to user | `{"type": "...", "title": "Bad Request", "status": 400, "detail": "Some notifications were not found or do not belong to user: 123"}` |
| `401 Unauthorized` | Missing or invalid authentication token | `{"type": "...", "title": "Unauthorized", "status": 401, "detail": "Full authentication is required..."}` |
| `500 Internal Server Error` | Server error occurred | `{"type": "...", "title": "Internal Server Error", "status": 500, "detail": "An unexpected error occurred"}` |

**Important Notes:**
- The operation is **atomic** - either all notifications are updated or none
- If any notification UUID in the list doesn't exist or doesn't belong to the user, the entire operation fails
- The error message will indicate that some notifications were not found or don't belong to the user

---

## Data Models

### NotificationAppStatusDTO

Represents a notification status for a user in the application.

```typescript
interface NotificationAppStatusDTO {
  uuid: string;                   // UUID (36 characters) - **Use this for all operations**
  userId: number;                 // Owner user ID
  seen: boolean;                  // Seen status
  seenAt: string | null;          // ISO 8601 timestamp or null
  active: boolean;                // Active status
  createdBy: string;              // Creator identifier
  lastModifiedBy: string | null;  // Last modifier or null
  createdAt: string;             // ISO 8601 timestamp
  lastModifiedAt: string;        // ISO 8601 timestamp
}
```

**Note:** Only `uuid` is exposed to the frontend. Internal IDs (`id`, `notificationId`, `templateId`) are not included in the response for security reasons.

### MarkNotificationsRequestDTO

Request body for marking notifications as seen/unseen.

```typescript
interface MarkNotificationsRequestDTO {
  notificationUuids: string[];    // Array of notification UUIDs (required, non-empty, each must be valid UUID format)
}
```

---

## Error Handling

### Common Error Scenarios

1. **Missing Authentication Token**
   - **Status:** `401 Unauthorized`
   - **Solution:** Include valid JWT token in `Authorization` header

2. **Invalid Token**
   - **Status:** `401 Unauthorized`
   - **Solution:** Refresh token or re-authenticate

3. **Empty Request Body**
   - **Status:** `400 Bad Request`
   - **Message:** "Notification UUIDs list cannot be null or empty"
   - **Solution:** Ensure request body contains valid `notificationUuids` array with at least one UUID

4. **Invalid Notification UUIDs**
   - **Status:** `400 Bad Request`
   - **Message:** "Some notifications were not found or do not belong to user: {userId}"
   - **Solution:** Verify all notification UUIDs exist and belong to the authenticated user. Ensure UUIDs are in correct format (36 characters)

5. **Server Error**
   - **Status:** `500 Internal Server Error`
   - **Solution:** Retry request or contact support

### Error Response Format

All errors follow the RFC 7807 Problem Details format:

```json
{
  "type": "https://www.jhipster.tech/problem/constraint-violation",
  "title": "Bad Request",
  "status": 400,
  "detail": "Detailed error message",
  "message": "error.validation",
  "params": "additional parameters if applicable"
}
```

---

## Example Usage

### Complete Frontend Integration Example

#### 1. Fetch Notifications with Pagination

```javascript
// Fetch first page of all notifications
async function fetchNotifications(page = 0, size = 20, includeSeen = true) {
  const token = localStorage.getItem('authToken');
  
  const response = await fetch(
    `/api/notifications?includeSeen=${includeSeen}&page=${page}&size=${size}&sort=createdAt,desc`,
    {
      method: 'GET',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      }
    }
  );
  
  if (!response.ok) {
    throw new Error(`HTTP error! status: ${response.status}`);
  }
  
  const notifications = await response.json();
  const totalCount = parseInt(response.headers.get('X-Total-Count') || '0');
  
  return {
    notifications,
    totalCount,
    currentPage: page,
    pageSize: size,
    totalPages: Math.ceil(totalCount / size)
  };
}

// Usage
const result = await fetchNotifications(0, 20, true);
console.log(`Found ${result.totalCount} notifications`);
console.log(`Showing page ${result.currentPage + 1} of ${result.totalPages}`);
```

#### 2. Fetch Only Unseen Notifications

```javascript
async function fetchUnseenNotifications() {
  const token = localStorage.getItem('authToken');
  
  const response = await fetch(
    `/api/notifications?includeSeen=false&page=0&size=50`,
    {
      method: 'GET',
      headers: {
        'Authorization': `Bearer ${token}`,
        'Content-Type': 'application/json'
      }
    }
  );
  
  if (!response.ok) {
    throw new Error(`HTTP error! status: ${response.status}`);
  }
  
  return await response.json();
}

// Usage
const unseenNotifications = await fetchUnseenNotifications();
console.log(`You have ${unseenNotifications.length} unseen notifications`);
```

#### 3. Mark Notifications as Seen

```javascript
async function markAsSeen(notificationUuids) {
  const token = localStorage.getItem('authToken');
  
  if (!notificationUuids || notificationUuids.length === 0) {
    throw new Error('Notification UUIDs array cannot be empty');
  }
  
  const response = await fetch('/api/notifications/mark-seen', {
    method: 'PUT',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      notificationUuids: notificationUuids
    })
  });
  
  if (!response.ok) {
    const error = await response.json();
    throw new Error(error.detail || `HTTP error! status: ${response.status}`);
  }
  
  return true; // Success
}

// Usage - Mark single notification
await markAsSeen(['550e8400-e29b-41d4-a716-446655440000']);

// Usage - Mark multiple notifications
await markAsSeen([
  '550e8400-e29b-41d4-a716-446655440000',
  '550e8400-e29b-41d4-a716-446655440001',
  '550e8400-e29b-41d4-a716-446655440002'
]);

// Usage - Mark all visible notifications
const notifications = await fetchNotifications();
const allUuids = notifications.map(n => n.uuid);
await markAsSeen(allUuids);
```

#### 4. Mark Notifications as Unseen

```javascript
async function markAsUnseen(notificationUuids) {
  const token = localStorage.getItem('authToken');
  
  if (!notificationUuids || notificationUuids.length === 0) {
    throw new Error('Notification UUIDs array cannot be empty');
  }
  
  const response = await fetch('/api/notifications/mark-unseen', {
    method: 'PUT',
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      notificationUuids: notificationUuids
    })
  });
  
  if (!response.ok) {
    const error = await response.json();
    throw new Error(error.detail || `HTTP error! status: ${response.status}`);
  }
  
  return true; // Success
}

// Usage
await markAsUnseen([
  '550e8400-e29b-41d4-a716-446655440000',
  '550e8400-e29b-41d4-a716-446655440001',
  '550e8400-e29b-41d4-a716-446655440002'
]);
```

#### 5. Complete React Hook Example

```javascript
import { useState, useEffect, useCallback } from 'react';

function useNotifications() {
  const [notifications, setNotifications] = useState([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);
  const [pagination, setPagination] = useState({
    page: 0,
    size: 20,
    totalCount: 0,
    totalPages: 0
  });
  
  const token = localStorage.getItem('authToken');
  
  const fetchNotifications = useCallback(async (page = 0, includeSeen = true) => {
    setLoading(true);
    setError(null);
    
    try {
      const response = await fetch(
        `/api/notifications?includeSeen=${includeSeen}&page=${page}&size=${pagination.size}&sort=createdAt,desc`,
        {
          headers: {
            'Authorization': `Bearer ${token}`,
            'Content-Type': 'application/json'
          }
        }
      );
      
      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }
      
      const data = await response.json();
      const totalCount = parseInt(response.headers.get('X-Total-Count') || '0');
      
      setNotifications(data);
      setPagination(prev => ({
        ...prev,
        page,
        totalCount,
        totalPages: Math.ceil(totalCount / prev.size)
      }));
    } catch (err) {
      setError(err.message);
    } finally {
      setLoading(false);
    }
  }, [token, pagination.size]);
  
  const markAsSeen = useCallback(async (uuids) => {
    try {
      const response = await fetch('/api/notifications/mark-seen', {
        method: 'PUT',
        headers: {
          'Authorization': `Bearer ${token}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({ notificationUuids: uuids })
      });
      
      if (!response.ok) {
        const error = await response.json();
        throw new Error(error.detail || 'Failed to mark as seen');
      }
      
      // Refresh notifications after marking as seen
      await fetchNotifications(pagination.page);
    } catch (err) {
      setError(err.message);
    }
  }, [token, pagination.page, fetchNotifications]);
  
  const markAsUnseen = useCallback(async (uuids) => {
    try {
      const response = await fetch('/api/notifications/mark-unseen', {
        method: 'PUT',
        headers: {
          'Authorization': `Bearer ${token}`,
          'Content-Type': 'application/json'
        },
        body: JSON.stringify({ notificationUuids: uuids })
      });
      
      if (!response.ok) {
        const error = await response.json();
        throw new Error(error.detail || 'Failed to mark as unseen');
      }
      
      // Refresh notifications after marking as unseen
      await fetchNotifications(pagination.page);
    } catch (err) {
      setError(err.message);
    }
  }, [token, pagination.page, fetchNotifications]);
  
  useEffect(() => {
    fetchNotifications(0);
  }, [fetchNotifications]);
  
  return {
    notifications,
    loading,
    error,
    pagination,
    fetchNotifications,
    markAsSeen,
    markAsUnseen
  };
}

// Usage in component
function NotificationList() {
  const {
    notifications,
    loading,
    error,
    pagination,
    fetchNotifications,
    markAsSeen
  } = useNotifications();
  
  const handleMarkAsSeen = (uuid) => {
    markAsSeen([uuid]);
  };
  
  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;
  
  return (
    <div>
      <h2>Notifications ({pagination.totalCount})</h2>
      {notifications.map(notification => (
        <div key={notification.uuid}>
          <p>{notification.uuid} - Seen: {notification.seen ? 'Yes' : 'No'}</p>
          {!notification.seen && (
            <button onClick={() => handleMarkAsSeen(notification.uuid)}>
              Mark as Seen
            </button>
          )}
        </div>
      ))}
      <div>
        <button
          onClick={() => fetchNotifications(pagination.page - 1)}
          disabled={pagination.page === 0}
        >
          Previous
        </button>
        <span>Page {pagination.page + 1} of {pagination.totalPages}</span>
        <button
          onClick={() => fetchNotifications(pagination.page + 1)}
          disabled={pagination.page >= pagination.totalPages - 1}
        >
          Next
        </button>
      </div>
    </div>
  );
}
```

---

## Best Practices

### 1. Pagination
- **Default page size:** Use 20-50 items per page for optimal performance
- **Infinite scroll:** For mobile apps, implement infinite scroll using pagination
- **Cache management:** Cache notifications locally and refresh when needed

### 2. Bulk Operations
- **Batch size:** When marking multiple notifications, consider batching in groups of 50-100 for better performance
- **Error handling:** Always handle the case where some notifications might not belong to the user
- **Optimistic updates:** Update UI optimistically, then refresh from server
- **UUID validation:** Validate UUID format (36 characters) on client side before sending request

### 3. Error Handling
- **Retry logic:** Implement exponential backoff for 500 errors
- **User feedback:** Show clear error messages to users
- **Validation:** Validate notification UUIDs on client side before sending request
- **UUID format:** Ensure all UUIDs are in correct format (e.g., `550e8400-e29b-41d4-a716-446655440000`)

### 4. Performance
- **Debounce:** Debounce rapid API calls (e.g., when user scrolls quickly)
- **Caching:** Cache notification list and update incrementally
- **Polling:** For real-time updates, consider WebSocket or polling every 30-60 seconds

### 5. User Experience
- **Loading states:** Show loading indicators during API calls
- **Empty states:** Handle empty notification lists gracefully
- **Notifications count:** Display unseen notification count in app header/badge
- **Auto-refresh:** Refresh notification list after marking as seen/unseen

---

## Support

For questions or issues regarding the Notification API, please contact:
- **Backend Team:** [Contact Information]
- **API Documentation:** [Link to Swagger/OpenAPI docs if available]

---

## Changelog

### Version 1.1.0 (Current)
- **BREAKING CHANGE:** Removed all entity IDs from API responses
- **BREAKING CHANGE:** Changed `notificationIds` to `notificationUuids` in request bodies
- Only UUIDs are exposed to frontend for security
- Updated all endpoints to use UUID-based operations
- Removed `id`, `notificationId`, `templateId` from response DTOs

### Version 1.0.0
- Initial release
- GET notifications with pagination
- Bulk mark as seen/unseen operations
- Filter by seen status

---

**Document Version:** 1.1.0  
**Last Updated:** January 2024  
**API Version:** 0.0.1

