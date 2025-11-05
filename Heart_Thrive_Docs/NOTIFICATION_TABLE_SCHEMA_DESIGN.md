# 🧩 Notification System Database Design (Enterprise-Grade)

This document describes the **final database schema** for a **multi-channel notification system**, designed for scalability, configurability, and future extensibility.

---

## ⚙️ Overview

This schema supports:
- Multiple notification channels (Email, SMS, Push, WhatsApp, etc.)
- Global and per-user configuration
- Feature-based notification rules
- Full delivery audit trail
- App seen/unseen tracking
- Soft activation/deactivation (`is_active` on all tables)
- Extensible provider support (Twilio, Firebase, AWS SES, etc.)

---

## 🧱 Design Goals

| Goal | Achieved By |
|------|--------------|
| Extensible to new channels | `notification_channel` master |
| Per-user & global configuration | `notification_config` |
| Feature-based notifications | `notification_feature` |
| Full delivery audit tracking | `notification_delivery` |
| Track seen/unseen push notifications | `notification_app_status` |
| Multiple providers (Twilio, Firebase) | `notification_provider` |
| Food reminder times | `food_time_schedule` JSON |
| Early reminders | `send_before_minutes` |
| Soft disable without delete | `is_active` on all tables |

---

## 🗂️ Tables Summary

Below is a table-by-table explanation of purpose, usage, and special columns.

---

### 1️⃣ `notification_channel`

#### 🎯 Purpose
Defines **all available communication methods** used by the system.

#### 🔍 Usage
Used by `notification_delivery` and `notification_provider`.  
Add new channels (like WhatsApp or Slack) without altering schema.

#### ⭐ Key Columns
| Column | Description |
|---------|--------------|
| `code` | Unique code (e.g., `EMAIL`, `PUSH`, `SMS`, `WHATSAPP`) |
| `is_active` | Soft enable/disable for channel |
| `description` | Description or integration notes |

---

### 2️⃣ `notification_feature`

#### 🎯 Purpose
Defines **which app features** trigger notifications (e.g., Food Reminder, Medicine Reminder).

#### 🔍 Usage
Categorizes notifications for clarity, reports, and configuration.

#### ⭐ Key Columns
| Column | Description |
|---------|--------------|
| `code` | Unique system identifier (`FOOD_REMINDER`, `MEDICINE_REMINDER`, etc.) |
| `default_enabled` | Indicates if the feature is enabled by default |
| `is_active` | Soft disable for that feature globally |

---

### 3️⃣ `notification_config`

#### 🎯 Purpose
Central table for **notification settings** — defines what, how, and when to send notifications.  
Supports **hierarchical configurations** (Global → Feature → User → User+Feature).

#### 🔍 Usage
Before sending any notification, the system checks this table to determine:
- If the feature is enabled for that user
- Which channels to use
- Food timing schedules
- How long before the event to send

#### ⭐ Key Columns
| Column | Description |
|---------|--------------|
| `user_id` | NULL = applies to all users; otherwise user-specific |
| `feature_id` | NULL = applies to all features; otherwise feature-specific |
| `is_enabled` | Whether notifications should be sent |
| `notification_channels` (JSON) | Example: `["EMAIL", "PUSH"]` |
| `food_time_schedule` (JSON) | Example: `{"breakfast":"09:00","lunch":"14:00"}` |
| `send_before_minutes` | Send notification X minutes before schedule |
| `is_active` | Temporarily disable the configuration |

#### ⚙️ Configuration Hierarchy

| Level | user_id | feature_id | Applies To |
|--------|----------|-------------|-------------|
| 1️⃣ | NULL | NULL | Global default (all users, all features) |
| 2️⃣ | NULL | X | All users for a specific feature |
| 3️⃣ | X | NULL | Specific user, all features |
| 4️⃣ | X | X | Specific user + specific feature (highest priority) |

---

### 4️⃣ `notification`

#### 🎯 Purpose
Represents a **triggered notification event** (e.g., "Food Reminder for User 101 at 9 AM").

#### 🔍 Usage
Parent table for delivery records and main log for notification lifecycle.

#### ⭐ Key Columns
| Column | Description |
|---------|--------------|
| `feature_id` | Which feature triggered the notification |
| `user_id` | Who received it |
| `title`, `message` | The content sent |
| `triggered_at` | When generated |
| `status` | Lifecycle status (`PENDING`, `COMPLETED`, `FAILED`) |
| `retry_count` | How many retries attempted |
| `is_active` | Enables/hides notification in logs without deletion |

---

### 5️⃣ `notification_delivery`

#### 🎯 Purpose
Tracks **per-channel delivery status** for every notification (Email, Push, SMS).

#### 🔍 Usage
Provides full visibility into success/failure per channel.  
Useful for debugging and delivery analytics.

#### ⭐ Key Columns
| Column | Description |
|---------|--------------|
| `notification_id` | Parent notification |
| `channel_id` | Which channel used |
| `status` | `PENDING`, `SENT`, `FAILED`, `RETRYING` |
| `failure_reason` | Error details if failed |
| `response_metadata` (JSON) | Provider response or message ID |
| `is_active` | Allows disabling invalid/old records |

---

### 6️⃣ `notification_provider`

#### 🎯 Purpose
Manages **external notification providers** for each channel (e.g., Twilio, Firebase).

#### 🔍 Usage
Switch providers or maintain multiple integrations easily.

#### ⭐ Key Columns
| Column | Description |
|---------|--------------|
| `channel_id` | The channel this provider serves |
| `name` | Provider name (`Twilio`, `Firebase`, `AWS_SES`) |
| `config` (JSON) | Provider credentials, API endpoints |
| `is_active` | Enables or disables specific provider |

---

### 7️⃣ `notification_app_status`

#### 🎯 Purpose
Tracks **seen/unseen status** for in-app (push) notifications.

#### 🔍 Usage
Used in mobile apps to display unread notification counts.

#### ⭐ Key Columns
| Column | Description |
|---------|--------------|
| `notification_id` | Related notification |
| `user_id` | Who received it |
| `is_seen` | TRUE if user opened the notification |
| `seen_at` | When the user saw it |
| `is_active` | Used to deactivate entries (e.g., user cleared history) |

---

## 🧠 Data Flow Summary

