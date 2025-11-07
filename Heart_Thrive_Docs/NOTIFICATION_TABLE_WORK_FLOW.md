# 🔧 Notification System — Core Principles & Flow Design

A production-grade, **template-driven, multi-channel notification system** with async dispatch, caching, idempotency, and full auditability.

---

## ⚙️ Core Principles

### 🧩 Separation of Concerns

| Responsibility | Layer / Table |
|----------------|----------------|
| **What / When** | `notification_config` & `notification_trigger` |
| **Who** | `users` |
| **What Kind** | `notification_feature` |
| **How (Content)** | `notification_template` |
| **How (Transport)** | `notification_channel` + `notification_provider` |
| **What Happened** | `notification` + `notification_delivery` + `notification_app_status` |

---

### 🧭 Hierarchical Configuration Resolution
`Global → Feature → User → User + Feature`  
Always pick the **most specific** configuration.

---

### 🧱 System Design Rules

- **Idempotent & Deduplicated** → No double sends per trigger/user/sub-event.  
- **Queue-first + Outbox-safe** → Asynchronous dispatch; retries with backoff.  
- **Cached Configuration** → Cache configs, templates, channels, and providers for speed.  
- **Extensible** → Easily add new channels or event types without schema redesign.

---

## 🧱 Core Tables (Usage Summary)

| Table | Description |
|--------|-------------|
| `notification_feature` | List of features (`FOOD_REMINDER`, `MEDICINE_REMINDER`, `WEIGHT_ALERT`, `HIGH_RISK_ALERT`) |
| `notification_channel` | Defines channels — Email, SMS, APP_PUSH, WhatsApp (with active flags) |
| `notification_provider` | Providers per channel — SES, Twilio, Firebase |
| `notification_config` | Rules per `(user_id, feature_id)` with JSON channel list, food times, send_before_minutes, enabled, active |
| `notification_template` | Message templates per `(feature_id × channel_id)` with default and active flags |
| `notification_trigger` | Each scheduler/manual/system event; the batch anchor |
| `notification` | One row per user per trigger (optionally per sub-event) |
| `notification_delivery` | One row per channel per notification (status, metadata) |
| `notification_app_status` | Tracks whether push notifications were seen by user |

🧠 **Optional Extension:**  
Add `sub_event_type` to `notification` (e.g., `BREAKFAST`, `LUNCH`, `SNACK`, `DINNER`, `MEDICINE`, `WEIGHT`, `RISK`)  
and a **unique key** `(user_id, trigger_id, sub_event_type)` to prevent duplicates when one trigger produces multiple sub-events per user.

---

## 🧭 End-to-End Flow (All Scenarios)

### 🕓 0) Precompute & Cache (Boot + Refresh on Change)
- Cache active **channels** and **providers** per channel.  
- Cache **templates** by `(feature_id, channel_id, locale?)`.  
- Cache **config resolution** per `user + feature` (with TTL; invalidate on update).  
- Cache **feature metadata** for quick lookup.

---

### 🚀 1) Create a Trigger (`notification_trigger`)
**When:**  
- Scheduled (e.g., cron for Food/Medicine in `Asia/Kolkata`)  
- System event (e.g., weight anomaly detected)  
- User action (e.g., password reset)  
- Manual (e.g., operations campaign)

**Insert:**  
One row into `notification_trigger` with:
- `feature_id`
- `trigger_type`
- `trigger_reference`
- `triggered_at`

---

### 👥 2) Build Candidate Audience
- For the `feature_id`, find target users:
  - **Food/Medicine:** Users opted-in via `notification_config` and have upcoming schedules.
  - **Weight/High Risk:** Users matched by rule engine.
- Resolve configuration hierarchy (prefer most specific).
- If not enabled or active → **skip**.

---

### 🧾 3) Create `notification` Rows (Idempotent)
For each **(user, sub-event)** candidate:

- Compute **idempotency key:** `(user_id, trigger_id, sub_event_type?)`
- **Upsert** notification:
  - If exists → reuse (no duplicate)
  - Else insert with:
    - `status = 'PENDING'`
    - `retry_count = 0`
    - `triggered_at = now()`
    - `feature_id`, `user_id`, `trigger_id`, `sub_event_type`

---

### 💬 4) Decide Channels & Templates
From resolved config:

- Use intersection of `notification_config.notification_channels` with **active channels**.
- For each channel:
  - Pick template by `(feature_id, channel_id)` — prefer `is_default=1` and `is_active=1`.
  - If none found → log and skip.

---

### 📬 5) Enqueue Deliveries (`notification_delivery`)
For each `(notification, channel, template)`:
- Insert `notification_delivery` with:
  - `status='PENDING'`
  - `template_id`, `channel_id`
- Push to queue:
  ```json
  { "delivery_id", "notification_id", "user_id", "feature_id", "channel_id", "template_id" }
  ```
  
---

### ⚡ 6) Dispatch Worker (Per Channel)

For each queued delivery:

1. **Fetch Provider**
   - Get active provider from `notification_provider` for the channel.
   - If none found → mark delivery as `FAILED`.

2. **Render Template Variables**
   - Common: `{userName}`, `{featureName}`, `{sendTime}`
   - Food: `{mealName}`, `{mealTime}`
   - Medicine: `{medicineName}`, `{dose}`, `{scheduleTime}`
   - Weight: `{diffKg}`, `{currentWeight}`

3. **Call Provider API**
   - Handle rate limits, retries, and timeouts gracefully.
   - Log provider responses for auditability.

4. **Update Delivery Status**
   - `SENT` → add `sent_at`, `response_metadata` (e.g., message IDs)
   - `FAILED` → add `failure_reason`
   - Schedule retry with **exponential backoff** for failed deliveries.

---

### 🔄 7) Aggregate Notification Status

A lightweight background process updates parent `notification.status` based on its delivery rows:

| Condition | Status |
|------------|---------|
| Any delivery `RETRYING` | `IN_PROGRESS` |
| All selected channels `SENT` | `COMPLETED` |
| All terminal and none `SENT` | `FAILED` |

This ensures that notification state reflects real delivery outcomes across all channels.

---

### 👁️ 8) App Seen Tracking (Push Notifications)

When a user opens an in-app or push notification:

- Call an API endpoint → upsert record in `notification_app_status`
- Update:
  - `is_seen = 1`
  - `seen_at = now()`

Used for:
- **Unread badge count**
- **Engagement analytics**
- **Retention metrics**

---

### 🔁 9) Retries & Dead-Letter Handling

- Retry failed deliveries with **exponential backoff** (e.g., `1m → 5m → 15m → 1h → stop`).
- Implement **maximum retry count**; after that, move to **Dead Letter Queue (DLQ)**.
- Support **auto-provider failover**, e.g.:
  - Email: `AWS SES → SendGrid`
  - SMS: `Twilio A → Twilio B`

Retries should be idempotent and logged for transparency.

---

### 📊 10) Monitoring & Reporting

Build observability around notification operations:

- 📈 **Delivery metrics** — success/failure rates per channel, feature, and time window.  
- 👤 **User-level history** — all notifications, deliveries, and statuses per user.  
- 🕹️ **Trigger dashboards** — from `notification_trigger`: volume, success %, failure %, and retries.  
- 🔔 **Push engagement** — unseen vs seen counts from `notification_app_status`.  
- 🚨 **Alerting** — detect and notify on sudden provider failure spikes or retry floods.

---

## 🧪 Scenario Playbooks

### 🍽️ A) Food Reminder

- Scheduler (IST) creates a `SCHEDULED` trigger for `FOOD_REMINDER` at times derived from `food_time_schedule - send_before_minutes`.
- Resolve configs → for each user + meal, create a `notification` (with `sub_event_type = BREAKFAST|LUNCH|SNACK|DINNER`).
- Determine channels (from config intersection) → create `notification_delivery` entries.
- Dispatch → update statuses.
- Push "seen" tracking in `notification_app_status`.

---

### 💊 B) Medicine Reminder

- Similar to **Food Reminder**, but event times come from **medication plan**.
- `send_before_minutes` commonly set to **15 minutes** before schedule.
- Typical channels: **SMS + Push**.
- Template variables include `{medicineName}`, `{dose}`, `{scheduleTime}`.

---

### ⚖️ C) Weight Gain Alert

- Triggered by **system event** when data service detects threshold breach.
- Creates a `SYSTEM_EVENT` trigger for `WEIGHT_ALERT`.
- Generates per-user notifications → Channel = Push.
- Template uses variables `{diffKg}`, `{currentWeight}` for personalized content.

---

### 🚨 D) High Risk Alert

- Trigger type: `MANUAL` or `SYSTEM_EVENT`.
- Forces **critical channels** (e.g., SMS + Push) even if user opted out.
- High-priority retries and alert logging.
- Useful for urgent or emergency notifications.

---

## 🧩 Idempotency & Concurrency Control

| Table | Unique Key | Purpose |
|--------|-------------|----------|
| `notification` | `(user_id, trigger_id, sub_event_type)` | Prevent duplicate creation per user per event |
| `notification_delivery` | `(notification_id, channel_id)` | Prevent multiple sends through the same channel |

**Concurrency Safety:**
- Use **database-level locks** (`FOR UPDATE NOWAIT`) or distributed locks (e.g., Redis `SETNX`) when batch jobs run in parallel.
- Batch workers should be **stateless**, and idempotency should always be enforced at the persistence layer.

---

> 💡 **Summary:**  
> This architecture ensures clean separation between **configuration**, **content**, and **transport** — providing high scalability, auditability, and reliability.  
> With **caching, async processing, idempotent logic**, and **provider failover**, the system guarantees consistency and high throughput even under load.
