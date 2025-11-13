# Timezone-based Notification Scheduling  
### Problem Statement & Proposed Approaches

---

## 1. Background

Our application is designed to send notifications to users for two main purposes:

### **1. Food Reminders**
Notifications for:
- Breakfast  
- Lunch  
- Snacks  
- Dinner  

### **2. Medication Reminders**  
Notifications based on medicines scheduled by the user.

Each reminder must respect the **user’s timezone**, not the server timezone.  
For example, a user in India and a user in the USA should both receive a “Take Breakfast” notification at **8:00 AM local time**.

---

## 2. Current Requirements

### **2.1 Food Reminder**

Notifications must be sent at fixed **local times**:

| Meal      | Time (Local) |
|-----------|--------------|
| Breakfast | 8:00 AM      |
| Lunch     | 1:00 PM      |
| Snacks    | 5:00 PM      |
| Dinner    | 8:00 PM      |

Each notification uses the **user’s timezone**.

---

### **2.2 Medication Reminder**

- Each user can schedule **multiple medicines per day**.
- Each medicine supports **up to 3 time slots**:
  - Morning  
  - Afternoon  
  - Evening  
- Each slot can have **custom times** provided by the user.

#### **Examples**
- Pentocid L – 8:00 AM  
- Paracetamol – 9:00 AM  
- Vitamin D – 8:30 PM  

Each schedule requires multiple business validations (~5 checks) to determine whether the notification should be fired.

---

## 3. Current Implementation Challenge

A scheduler runs **every 5 minutes** to identify notifications due for the current window.

### Per user calculation:

- 5 medicines × 3 slots/day → ~15 checks  
- Each medicine → ~5 validation operations  
- **15 × 5 = 75 operations per user**

### Load estimation:

| User Count | Operations Every 5 Min |
|------------|-------------------------|
| 1,000 users | 75,000 operations |
| 10,000 users | 750,000+ operations |

### System impact:
- High computation cost  
- Scaling issues  
- High DB reads/writes  
- Complexity increases as user base grows  

---

## 4. Key Technical Challenges

| Area | Problem |
|------|---------|
| **Timezone Handling** | Correctly converting and managing user timezones during scheduling & triggering |
| **High Computation Load** | Recomputing schedules for all users every 5 minutes, even if no notification is due |
| **Scalability** | Load grows exponentially with user count |
| **Precision** | Must trigger notifications exactly on minute-level precision, difficult with batch schedulers |

---

## 5. Potential Approaches (For Discussion)

### **Option 1 – Pre-compute and Cache Next Notification Time**

- Pre-calculate and store each user’s next notification time (in UTC).  
- Scheduler checks:  
  `WHERE next_notification_time <= current_utc_time`

**Pros**
- Dramatically reduces repeated calculations  
- Only due notifications are processed  

**Cons**
- More complex updates when the user changes timezone or schedule  

---

### **Option 2 – Distributed Scheduled Jobs**  
(e.g., Quartz, AWS EventBridge, Celery Beat)

- Register each user’s notification as an individual cron-like scheduled job.

**Pros**
- Very precise delivery  
- Supports horizontal scaling  

**Cons**
- Infrastructure complexity  
- Hard limits on number of jobs in some systems  

---

### **Option 3 – Message Queue + Worker Pattern**  
(e.g., RabbitMQ, Kafka, AWS SQS)

- Store upcoming notification tasks in a queue.  
- Worker nodes consume tasks and send notifications when due.

**Pros**
- Asynchronous  
- Highly scalable  
- Better tolerance for traffic spikes  

**Cons**
- Requires strong queue management  
- More moving components  

---

### **Option 4 – Hybrid Approach**

- Use pre-computed **next event times**.  
- A lightweight scheduler runs every 5 minutes and pushes jobs into a queue.  
- Workers execute the notifications.

**Pros**
- Balanced performance and maintainability  
- Scales to millions of events  

---

## 6. Discussion Points for Senior Team

We need feedback on:

- Best design pattern for **large-scale timezone-based scheduling**
- Whether to move to **event-driven architecture**
- Strategies to minimize **database reads and recalculations**
- Best practices for **timezone-safe scheduling**  
  (store in UTC, convert at trigger-time)
- Recommended **tools/frameworks** for distributed cron systems

---

## 7. Summary

| Aspect | Current | Desired |
|--------|---------|---------|
| **Scheduler Load** | High – checks all users every 5 minutes | Optimized – only due users are processed |
| **Timezone Management** | Manual conversions | Automated & timezone-safe |
| **Scalability** | Limited to a few thousand users | Should support 100K+ users |
| **Accuracy** | 5-min window | Near real-time |
| **Architecture** | Monolithic / Cron-based | Event-driven or hybrid |

---

