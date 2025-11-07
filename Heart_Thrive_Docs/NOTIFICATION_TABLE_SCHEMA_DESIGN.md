# 📘 Notification System — Database Design Overview

This schema powers a **multi-channel, template-driven notification system**, supporting **Email, SMS, Push, In-App**, and future channels like **WhatsApp**.

It ensures:

- ✅ Full configurability (per feature, per user, per channel)  
- ✅ Template-based message management  
- ✅ Complete auditability from trigger to delivery  
- ✅ High performance and reusability  

---

## 🧭 Table of Contents

| # | Table Name | Purpose |
|---|-------------|----------|
| 1️⃣ | **notification_channel** | Defines available communication methods (Email, SMS, Push, etc.) |
| 2️⃣ | **notification_feature** | Defines system features or events that trigger notifications (Food Reminder, etc.) |
| 3️⃣ | **notification_config** | Stores notification preferences and timing for users/features |
| 4️⃣ | **notification_template** | Stores per-channel message templates for each feature |
| 5️⃣ | **notification_trigger** | Logs when and why notifications were triggered |
| 6️⃣ | **notification** | Represents individual user notifications generated for a trigger |
| 7️⃣ | **notification_delivery** | Tracks delivery results for each channel per notification |
| 8️⃣ | **notification_provider** | Manages external providers like Twilio, Firebase, SES, etc. |
| 9️⃣ | **notification_app_status** | Tracks if in-app notifications were seen or not by users |

---

## 1️⃣ notification_channel

### 🎯 Purpose
Defines all possible communication channels supported by the system — such as Email, SMS, Push, or WhatsApp.

### ⭐ Key Columns
| Column | Description |
|---------|-------------|
| `code` | Unique channel identifier (e.g., EMAIL, SMS, PUSH) |
| `name` | Channel display name |
| `description` | Channel details (e.g., used provider or purpose) |
| `is_active` | Soft-enable or disable a channel globally |

---

## 2️⃣ notification_feature

### 🎯 Purpose
Represents different application features or use cases that can trigger notifications.  
Example: *Food Reminder, Medicine Reminder, Weight Alert, High-Risk Alert.*

### ⭐ Key Columns
| Column | Description |
|---------|-------------|
| `code` | Unique identifier (e.g., FOOD_REMINDER) |
| `default_enabled` | Whether the feature sends notifications by default |
| `is_active` | Enables/disables the feature across the system |

---

## 3️⃣ notification_config

### 🎯 Purpose
Defines **what notifications** should be sent, **through which channels**, and **when**.  
It supports a hierarchical configuration — **Global → Feature → User → User+Feature**.

### ⭐ Key Columns
| Column | Description |
|---------|-------------|
| `user_id` | `NULL` = applies to all users; otherwise, user-specific |
| `feature_id` | `NULL` = applies to all features; otherwise, feature-specific |
| `notification_channels` | JSON array of enabled channels (e.g., `["EMAIL","APP_PUSH"]`) |
| `food_time_schedule` | JSON object defining meal reminder times |
| `send_before_minutes` | How many minutes before an event to trigger the reminder |
| `is_enabled` | Enables or disables the entire config |
| `is_active` | Soft control for temporary disablement |

### ⚙️ Hierarchy Logic
| Level | Applies To |
|--------|-------------|
| 1️⃣ Global | All users, all features |
| 2️⃣ Feature | All users for that feature |
| 3️⃣ User | All features for that user |
| 4️⃣ User + Feature | Most specific configuration |

---

## 4️⃣ notification_template 🆕

### 🎯 Purpose
Stores **message templates per feature and channel**, allowing you to define different messages for **Email, SMS, Push**, etc.

### ⭐ Key Columns
| Column | Description |
|---------|-------------|
| `feature_id` | Which feature this template belongs to |
| `channel_id` | For which channel this template is used |
| `title_template` | Title template (used for Email or Push) |
| `message_template` | Message body template (placeholders like `{userName}`, `{mealTime}`) |
| `is_default` | Marks as default template for that (feature × channel) |
| `is_active` | Enables/disables this template |
| `created_by`, `last_modified_by` | Tracks who created or updated it |

### 🧠 Special Note
This allows full flexibility, for example:
- **Food Reminder** → Different text for Email vs Push  
- **Medicine Reminder** → Personalized message per user or language  

---

## 5️⃣ notification_trigger 🆕

### 🎯 Purpose
Logs each time the system **triggers a batch of notifications** — e.g., a morning reminder, a password reset event, or a promotional job.

### ⭐ Key Columns
| Column | Description |
|---------|-------------|
| `feature_id` | Which feature was triggered |
| `trigger_type` | How this notification was triggered — `SCHEDULED`, `USER_ACTION`, `SYSTEM_EVENT`, `MANUAL` |
| `trigger_reference` | External reference ID (e.g., job ID, API ID) |
| `triggered_at` | Timestamp of when the trigger occurred |
| `is_active` | Enables or disables trigger record tracking |

### 🧠 Special Note
This table helps trace the origin of any notification — crucial for **debugging or analytics**  
(e.g., “Which scheduler job generated this?”).

---

## 6️⃣ notification

### 🎯 Purpose
Represents **individual user notifications** generated by a specific trigger and feature.

### ⭐ Key Columns
| Column | Description |
|---------|-------------|
| `feature_id` | Which feature notification belongs to |
| `user_id` | Which user receives it |
| `trigger_id` | FK → `notification_trigger` (which event caused it) |
| `status` | Overall notification state (`PENDING`, `IN_PROGRESS`, `COMPLETED`, `FAILED`) |
| `retry_count` | Retry attempts if delivery fails |
| `triggered_at` | When the notification was generated |
| `is_active` | Soft deletion flag |

### 🧠 Special Note
`notification` is the main record per user per trigger, connecting **configuration to delivery attempts**.

---

## 7️⃣ notification_delivery

### 🎯 Purpose
Tracks **per-channel delivery results** for each notification.  
Each row = one notification sent via one channel.

### ⭐ Key Columns
| Column | Description |
|---------|-------------|
| `notification_id` | FK → `notification` (which user notification this belongs to) |
| `channel_id` | FK → `notification_channel` |
| `template_id` | FK → `notification_template` (which template used) |
| `status` | `PENDING`, `SENT`, `FAILED`, `RETRYING` |
| `failure_reason` | Error message if failed |
| `response_metadata` | JSON response from provider |
| `is_active` | Marks active/inactive delivery record |

### 🧠 Special Note
Allows **multi-channel tracking** per notification:
> e.g., Notification #45 → Email ✅ SENT, SMS ❌ FAILED, Push ✅ SENT

---

## 8️⃣ notification_provider

### 🎯 Purpose
Manages **external service providers** for each channel — e.g., **Twilio (SMS)**, **AWS SES (Email)**, **Firebase (Push)**.

### ⭐ Key Columns
| Column | Description |
|---------|-------------|
| `channel_id` | FK → `notification_channel` |
| `name` | Provider name (Twilio, SES, Firebase) |
| `config` | JSON config with API credentials or region info |
| `is_active` | Soft enable/disable provider |

### 🧠 Special Note
Supports **multi-provider strategy**, e.g. fallback from **AWS SES → SendGrid**.

---

## 9️⃣ notification_app_status

### 🎯 Purpose
Tracks whether **in-app notifications (push)** were seen or not by users.

### ⭐ Key Columns
| Column | Description |
|---------|-------------|
| `notification_id` | FK → `notification` |
| `user_id` | FK → `users` |
| `is_seen` | Whether user opened the notification |
| `seen_at` | Timestamp of when seen |
| `is_active` | Marks valid/archived records |

### 🧠 Special Note
Helps drive **“Unread Notifications Count”** and track **user engagement**.

---

## 🧩 Overall Flow Summary

┌────────────────────────────────────────────────────────────┐
│ Configuration Layer │
│ notification_feature → notification_config → notification_template │
└────────────────────────────────────────────────────────────┘
↓
┌────────────────────────────────────────────────────────────┐
│ Trigger Layer │
│ notification_trigger (event logs) │
└────────────────────────────────────────────────────────────┘
↓
┌────────────────────────────────────────────────────────────┐
│ Runtime Layer │
│ notification → notification_delivery │
└────────────────────────────────────────────────────────────┘
↓
┌────────────────────────────────────────────────────────────┐
│ Tracking Layer │
│ notification_app_status │
└────────────────────────────────────────────────────────────┘
↓
┌────────────────────────────────────────────────────────────┐
│ Reference Layer │
│ notification_channel → notification_provider │
└────────────────────────────────────────────────────────────┘