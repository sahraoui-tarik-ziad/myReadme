# 🔔 Notifications Module - Complete Guide

## 📖 Table of Contents
1. [What Is This Module?](#what-is-this-module)
2. [The Big Picture - How It Works](#the-big-picture)
3. [Real-World Scenarios](#real-world-scenarios)
4. [Why Search by userId and status?](#why-search-by-userid-and-status)
5. [DTO Fields Explained](#dto-fields-explained)
6. [Redis Queue Listening](#redis-queue-listening)
7. [API Endpoints Reference](#api-endpoints-reference)
8. [What You Need to Do](#what-you-need-to-do)

---

## What Is This Module?

The **Notifications Module** is the **core brain** of the notification service. Think of it as a **notification database and manager**.

### What It Does:
- ✅ **Stores** all notification records (email, push, in-app)
- ✅ **Tracks** notification status (pending, sent, delivered, failed)
- ✅ **Manages** retry attempts for failed notifications
- ✅ **Schedules** notifications for later delivery
- ✅ **Queries** notifications by user, status, date, etc.

### What It Does NOT Do:
- ❌ Does NOT actually send emails (that's the email service's job - coming later)
- ❌ Does NOT send push notifications (that's Firebase's job - coming later)
- ❌ Does NOT listen to Redis queue (that's the EventsModule processor's job - already built!)

---

## The Big Picture - How It Works

```
┌─────────────────────────────────────────────────────────────────┐
│                    BOOKING SERVICE                               │
│  User books parking → Event published to Redis                  │
└─────────────────────┬───────────────────────────────────────────┘
                      │
                      ▼
               ┌─────────────┐
               │    Redis    │
               │    Queue    │
               └──────┬──────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────────┐
│              EVENTS MODULE (Already Built!)                      │
│  NotificationQueueProcessor listens 24/7                        │
│  - Receives: booking.confirmed event                            │
│  - Checks: User preferences (emailEnabled?)                     │
│  - Calls: NotificationsService.create()  ←──────────────┐      │
└─────────────────────────────────────────────────────────────────┘
                      │                                    │
                      ▼                                    │
┌─────────────────────────────────────────────────────────────────┐
│         NOTIFICATIONS MODULE (THIS MODULE!)              │      │
│                                                          │      │
│  ┌─────────────────────────────────────────────┐        │      │
│  │  NotificationsService.create()              │ ◄──────┘      │
│  │                                             │                │
│  │  1. Saves notification to database          │                │
│  │     - userId: "user_001"                    │                │
│  │     - type: "email"                         │                │
│  │     - template: "booking_confirmation"      │                │
│  │     - status: "pending"                     │                │
│  │     - priority: "high"                      │                │
│  │     - content: { bookingId, parkingName }   │                │
│  │                                             │                │
│  │  2. Returns notification record             │                │
│  └─────────────────────────────────────────────┘                │
│                      │                                           │
│                      │ Notification saved in DB                 │
│                      ▼                                           │
│           ┌──────────────────────┐                              │
│           │  PostgreSQL Database │                              │
│           │  notifications table │                              │
│           └──────────────────────┘                              │
└─────────────────────────────────────────────────────────────────┘
                      │
                      │ Later (Future Step)
                      ▼
┌─────────────────────────────────────────────────────────────────┐
│              EMAIL SERVICE (To Be Built)                         │
│  - Queries: notifications with status="pending"                 │
│  - Sends: actual emails via AWS SES                             │
│  - Updates: status to "sent" → "delivered"                      │
└─────────────────────────────────────────────────────────────────┘
```

---

## Real-World Scenarios

### Scenario 1: User Books a Parking Spot

**What Happens:**

1. **Booking Service** (10:00 AM):
   ```json
   User "Jean Dupont" books parking for tomorrow
   → Publishes event to Redis queue
   ```

2. **EventsModule Processor** (10:00:01 AM):
   ```typescript
   // Receives event from queue
   {
     eventType: "booking.confirmed",
     data: {
       userId: "user_001",
       userEmail: "jean@example.com",
       bookingId: "BKG-123",
       parkingName: "Parking Charles de Gaulle"
     }
   }

   // Checks user preferences
   preferences = await getPreferences("user_001")
   if (preferences.emailEnabled === false) {
     return; // User disabled emails, skip
   }

   // Calls NotificationsModule
   await notificationsService.create({
     userId: "user_001",
     type: "email",
     template: "booking_confirmation",
     subject: "Confirmation de votre réservation",
     content: {
       userName: "Jean Dupont",
       parkingName: "Parking Charles de Gaulle",
       bookingId: "BKG-123"
     },
     channel: "email",
     priority: "high"
   })
   ```

3. **NotificationsModule** (10:00:02 AM):
   ```sql
   -- Saves to database
   INSERT INTO notifications (
     id, user_id, type, template, status,
     priority, content, created_at
   ) VALUES (
     'uuid-xxx',
     'user_001',
     'email',
     'booking_confirmation',
     'pending',  -- ← Not sent yet!
     'high',
     '{"userName": "Jean Dupont", ...}',
     NOW()
   )
   ```

4. **Email Service** (10:00:05 AM - Future):
   ```typescript
   // Cron job runs every minute
   const pendingNotifications = await notificationsService.findAll(
     undefined,      // userId (all users)
     'pending'       // status (only pending)
   )

   for (const notification of pendingNotifications) {
     // Actually send email via AWS SES
     await sendEmail(notification)

     // Update status
     await notificationsService.update(notification.id, {
       status: 'sent',
       sentAt: new Date()
     })
   }
   ```

5. **Result:**
   - User receives email ✅
   - Notification status updated to "sent" → "delivered"
   - If email bounces → status becomes "failed"

---

### Scenario 2: Admin Wants to See All Failed Notifications

**Why?** To retry failed emails or investigate delivery issues.

**How:**

```bash
# Admin queries API
GET /notifications?status=failed

# Returns:
[
  {
    id: "uuid-1",
    userId: "user_002",
    type: "email",
    template: "booking_confirmation",
    status: "failed",
    errorMessage: "SMTP connection timeout",
    retryCount: 3,
    failedAt: "2025-11-02T10:30:00Z"
  },
  {
    id: "uuid-2",
    userId: "user_003",
    type: "email",
    template: "payment_confirmation",
    status: "failed",
    errorMessage: "Invalid email address",
    retryCount: 2,
    failedAt: "2025-11-02T11:15:00Z"
  }
]
```

**What Admin Can Do:**
- See which emails failed
- Check error messages
- Decide to retry manually
- Fix user email addresses
- Investigate SMTP issues

---

### Scenario 3: User Checks Their Notification History

**Why?** User wants to see all notifications they received.

**How:**

```bash
# Frontend calls API
GET /notifications?userId=user_001

# Returns user's notification history:
[
  {
    id: "uuid-1",
    userId: "user_001",
    type: "email",
    template: "booking_confirmation",
    status: "delivered",
    sentAt: "2025-11-02T10:00:00Z",
    deliveredAt: "2025-11-02T10:01:00Z",
    openedAt: "2025-11-02T11:30:00Z"  // User opened email!
  },
  {
    id: "uuid-2",
    userId: "user_001",
    type: "push",
    template: "booking_reminder",
    status: "sent",
    sentAt: "2025-11-14T06:00:00Z"
  }
]
```

---

### Scenario 4: Scheduled Notification (Booking Reminder)

**Use Case:** Send reminder 24 hours before booking starts.

**How:**

1. **Create Scheduled Notification:**
   ```typescript
   await notificationsService.create({
     userId: "user_001",
     type: "push",
     template: "booking_reminder",
     content: {
       parkingName: "Parking CDG",
       startTime: "Tomorrow 8:00 AM"
     },
     channel: "push",
     priority: "urgent",
     scheduledAt: "2025-11-14T06:00:00Z"  // ← Send tomorrow at 6am
   })
   ```

2. **Saved with status: "pending"**
   ```json
   {
     id: "uuid-xxx",
     status: "pending",
     scheduledAt: "2025-11-14T06:00:00Z"
   }
   ```

3. **Cron Job Checks Scheduled Notifications:**
   ```typescript
   // Runs every 5 minutes
   const scheduled = await notificationsService.getScheduledNotifications()
   // Returns notifications where:
   //   status = 'pending'
   //   scheduledAt <= NOW()

   for (const notification of scheduled) {
     await sendNotification(notification)
     await notificationsService.update(notification.id, {
       status: 'sent',
       sentAt: new Date()
     })
   }
   ```

---

## Why Search by userId and status?

### 🔍 Search by `userId`

**Real-World Use Cases:**

1. **User Dashboard:**
   ```typescript
   // User logs in, sees their notification history
   GET /notifications?userId=user_001
   ```

2. **User Preferences:**
   ```typescript
   // Before sending notification, check user's history
   const userNotifications = await findAll('user_001')
   // Did they receive 10 marketing emails this week?
   // Maybe slow down...
   ```

3. **Debugging:**
   ```typescript
   // Support agent: "User says they didn't get email"
   GET /notifications?userId=user_001&status=failed
   // Find out what went wrong
   ```

4. **Compliance (GDPR):**
   ```typescript
   // User requests all their data
   GET /notifications?userId=user_001
   // Export all notifications sent to this user
   ```

---

### 📊 Search by `status`

**Real-World Use Cases:**

1. **Email Service - Send Pending Notifications:**
   ```typescript
   // Every minute, send pending emails
   const pending = await findAll(undefined, 'pending')
   for (const notification of pending) {
     await sendEmail(notification)
   }
   ```

2. **Retry Failed Notifications:**
   ```typescript
   // Every hour, retry failed notifications
   const failed = await getFailedNotifications(maxRetries: 3)
   for (const notification of failed) {
     if (notification.retryCount < 3) {
       await retryNotification(notification)
     }
   }
   ```

3. **Analytics Dashboard:**
   ```typescript
   // Admin sees notification stats
   const delivered = await findAll(undefined, 'delivered')
   const failed = await findAll(undefined, 'failed')
   const pending = await findAll(undefined, 'pending')

   // Display: "95% delivery rate"
   ```

4. **Health Monitoring:**
   ```typescript
   // Alert if too many failures
   const recentFailed = await findAll(undefined, 'failed')
   if (recentFailed.length > 100) {
     sendAlert("HIGH FAILURE RATE - Check email service!")
   }
   ```

---

### 🔄 Combine Both (userId + status)

```typescript
// "Show me all failed notifications for user_001"
GET /notifications?userId=user_001&status=failed

// Use case: User complains they didn't get booking confirmation
// Support agent can see exactly what failed and why
```

---

## DTO Fields Explained

### 📝 CreateNotificationDto

```typescript
{
  userId: string;              // Who gets this notification?
  type: NotificationType;      // How to send? (email/push/in_app)
  template: string;            // Which template to use?
  subject?: string;            // Email subject line
  content?: object;            // Template variables
  channel: string;             // Delivery channel
  priority?: NotificationPriority;  // How urgent?
  retryCount?: number;         // How many retries so far?
  scheduledAt?: string;        // When to send? (future date)
  metadata?: object;           // Extra data for tracking
}
```

#### Field-by-Field Explanation:

**1. userId (Required)**
```typescript
userId: "user_001"
```
**Why?** We need to know WHO to send the notification to!
- Used to look up user's email address
- Used to check user preferences
- Used to query user's notification history

---

**2. type (Required)**
```typescript
type: "email" | "push" | "in_app"
```
**Why?** Different types need different handling!
- `"email"` → Send via AWS SES, needs email address
- `"push"` → Send via Firebase, needs device token
- `"in_app"` → Store in database, show in app

**Example:**
```typescript
// Booking confirmation = email (permanent record)
{ type: "email" }

// Urgent reminder = push (immediate attention)
{ type: "push" }

// New message = in-app (check when convenient)
{ type: "in_app" }
```

---

**3. template (Required)**
```typescript
template: "booking_confirmation"
```
**Why?** Templates define the message structure!

Think of templates like Microsoft Word templates:
- `"booking_confirmation"` template has: userName, parkingName, date
- `"payment_confirmation"` template has: amount, paymentMethod
- `"booking_reminder"` template has: hoursUntilStart, parkingAddress

**Example:**
```typescript
// Template: "booking_confirmation"
content: {
  userName: "Jean",
  parkingName: "Parking CDG",
  date: "Nov 15, 2025"
}

// Renders to:
"Hello Jean, your booking at Parking CDG for Nov 15, 2025 is confirmed!"
```

---

**4. subject (Optional)**
```typescript
subject: "Confirmation de votre réservation"
```
**Why?** Email needs a subject line!
- Only used for `type: "email"`
- Can be overridden from template default
- Push/in-app don't need this

---

**5. content (Optional)**
```typescript
content: {
  userName: "Jean Dupont",
  parkingName: "Parking CDG",
  bookingId: "BKG-123",
  confirmationCode: "CONF-ABC123"
}
```
**Why?** Template variables need actual values!

Think of it like mail merge:
```
Template: "Hello {{userName}}, your booking at {{parkingName}}"
Content: { userName: "Jean", parkingName: "CDG" }
Result: "Hello Jean, your booking at CDG"
```

**Stored as JSON** so it can contain anything:
- Strings: `"Jean Dupont"`
- Numbers: `45.50`
- Dates: `"2025-11-15"`
- Arrays: `["Parking 1", "Parking 2"]`
- Objects: `{ parking: { name: "CDG", address: "..." } }`

---

**6. channel (Required)**
```typescript
channel: "email" | "push" | "sms" | "in_app"
```
**Why?** Specifies the delivery method.

**Similar to `type` but more specific:**
- `type` = category (email, push, in_app)
- `channel` = actual delivery method

**Examples:**
```typescript
{ type: "email", channel: "email" }        // Standard email
{ type: "email", channel: "sms" }          // SMS instead
{ type: "push", channel: "push" }          // Push notification
{ type: "push", channel: "websocket" }     // Real-time WebSocket
```

---

**7. priority (Optional)**
```typescript
priority: "urgent" | "high" | "normal" | "low"
```
**Why?** Determines delivery order and importance!

**Use Cases:**
- `urgent` → Payment failed, booking cancelled (send immediately!)
- `high` → Booking confirmed, reminder (send soon)
- `normal` → General updates (send within hour)
- `low` → Marketing emails (send when convenient)

**Example:**
```typescript
// Payment failed - URGENT!
{ priority: "urgent" }

// Weekly newsletter - low priority
{ priority: "low" }
```

**Email service uses this to prioritize:**
```typescript
// Process urgent notifications first
SELECT * FROM notifications
WHERE status = 'pending'
ORDER BY priority DESC  -- urgent first, low last
```

---

**8. retryCount (Optional)**
```typescript
retryCount: 0  // Default
```
**Why?** Track how many times we tried to send this!

**Example Flow:**
```typescript
// First attempt
{ retryCount: 0, status: "pending" }

// Failed, retry
{ retryCount: 1, status: "pending" }

// Failed again, retry
{ retryCount: 2, status: "pending" }

// Failed 3 times, give up
{ retryCount: 3, status: "failed" }
```

**Max retries: 3** (configurable)

---

**9. scheduledAt (Optional)**
```typescript
scheduledAt: "2025-11-14T06:00:00Z"  // Future date
```
**Why?** Send notification later, not now!

**Use Cases:**
- Booking reminder 24 hours before
- Birthday email on user's birthday
- Campaign email at 9am Monday
- Abandoned cart reminder after 3 hours

**Example:**
```typescript
// Create reminder for tomorrow
await notificationsService.create({
  userId: "user_001",
  type: "push",
  template: "booking_reminder",
  scheduledAt: addHours(new Date(), 24),  // 24 hours from now
  status: "pending"
})

// Cron job checks every 5 minutes:
const ready = await getScheduledNotifications()
// Returns notifications where scheduledAt <= NOW()
```

---

**10. metadata (Optional)**
```typescript
metadata: {
  bookingId: "BKG-123",
  campaignId: "summer-2025",
  source: "mobile-app",
  eventId: "evt-456"
}
```
**Why?** Extra tracking data!

**Use Cases:**
- Link notification to booking/payment
- Track which campaign it belongs to
- Analytics (which source converts better?)
- Debugging (which event triggered this?)

**Stored as JSON** so you can put anything:
```typescript
metadata: {
  bookingId: "BKG-123",           // Link to booking
  eventId: "evt-456",             // Link to original event
  retryReason: "SMTP timeout",    // Why we're retrying
  originalSentAt: "...",          // First attempt time
  ipAddress: "192.168.1.1",       // User's IP (for security)
  userAgent: "iPhone 13"          // User's device
}
```

---

### 📝 UpdateNotificationDto

```typescript
{
  status?: NotificationStatus;     // Update delivery status
  priority?: NotificationPriority; // Change priority
  retryCount?: number;             // Update retry count
  scheduledAt?: string;            // Reschedule
  sentAt?: string;                 // Mark when sent
  deliveredAt?: string;            // Mark when delivered
  openedAt?: string;               // Mark when opened
  failedAt?: string;               // Mark when failed
  errorMessage?: string;           // Store error details
  metadata?: object;               // Update tracking data
}
```

**When to Use:**

1. **Email Service Updates Status:**
   ```typescript
   // After sending email
   await update(notificationId, {
     status: 'sent',
     sentAt: new Date()
   })

   // AWS SES confirms delivery
   await update(notificationId, {
     status: 'delivered',
     deliveredAt: new Date()
   })

   // User opened email
   await update(notificationId, {
     openedAt: new Date()
   })
   ```

2. **Handle Failures:**
   ```typescript
   // Email bounced
   await update(notificationId, {
     status: 'failed',
     failedAt: new Date(),
     errorMessage: 'Invalid email address',
     retryCount: notification.retryCount + 1
   })
   ```

3. **Reschedule:**
   ```typescript
   // Move scheduled notification to different time
   await update(notificationId, {
     scheduledAt: newDate
   })
   ```

---

## Redis Queue Listening

### ❓ Does NotificationsModule Listen to Redis?

**NO!** The **EventsModule** listens to Redis, not NotificationsModule.

### 🔄 Here's How It Works:

```
Redis Queue → EventsModule (Processor) → NotificationsModule (Service)
```

**Step-by-Step:**

1. **Booking Service** publishes event to Redis:
   ```typescript
   // In booking service
   await redisQueue.add('booking-event', {
     eventType: 'booking.confirmed',
     data: { userId, bookingId, ... }
   })
   ```

2. **EventsModule Processor** (already built!) listens 24/7:
   ```typescript
   // src/modules/events/processors/notification-queue.processor.ts

   @Processor('notifications')
   export class NotificationQueueProcessor {

     @Process('booking-event')
     async handleBookingEvent(job: Job<BookingEvent>) {
       // This runs automatically when event arrives!

       // 1. Check user preferences
       const prefs = await this.preferencesService.getPreferences(
         job.data.data.userId
       )

       if (!prefs.emailEnabled) return; // User disabled emails

       // 2. Call NotificationsModule to save notification
       await this.notificationsService.create({
         userId: job.data.data.userId,
         type: 'email',
         template: 'booking_confirmation',
         content: job.data.data,
         channel: 'email',
         priority: 'high'
       })
     }
   }
   ```

3. **NotificationsModule** just saves to database:
   ```typescript
   // src/modules/notifications/notifications.service.ts

   async create(createDto: CreateNotificationDto) {
     // Just save to database, that's it!
     return this.prisma.notification.create({
       data: createDto
     })
   }
   ```

### Why This Separation?

**EventsModule (Processor):**
- Listens to Redis queue ✅
- Receives events from booking service ✅
- Checks user preferences ✅
- Handles different event types ✅

**NotificationsModule (Service):**
- Stores notifications in database ✅
- Queries notifications ✅
- Updates notification status ✅
- Provides REST API ✅

**This way:**
- NotificationsModule can be used by ANYONE (EventsModule, cron jobs, REST API)
- EventsModule is the ONLY one listening to Redis
- Clean separation of concerns!

---

## API Endpoints Reference

### Create Notification (Manual)
```bash
POST /notifications
{
  "userId": "user_001",
  "type": "email",
  "template": "booking_confirmation",
  "subject": "Booking Confirmed",
  "content": { "userName": "Jean", "parkingName": "CDG" },
  "channel": "email",
  "priority": "high"
}
```

### Get All Notifications
```bash
GET /notifications
GET /notifications?userId=user_001
GET /notifications?status=pending
GET /notifications?userId=user_001&status=failed
```

### Get Scheduled Notifications
```bash
GET /notifications/scheduled
# Returns notifications ready to be sent (scheduledAt <= NOW)
```

### Get Failed Notifications
```bash
GET /notifications/failed
GET /notifications/failed?maxRetries=3
# Returns failed notifications with retry count < 3
```

### Get Single Notification
```bash
GET /notifications/:id
```

### Update Notification
```bash
PATCH /notifications/:id
{
  "status": "sent",
  "sentAt": "2025-11-02T10:00:00Z"
}
```

### Delete Notification
```bash
DELETE /notifications/:id
```

---

## What You Need to Do

### ✅ Already Done:
1. NotificationsModule created ✅
2. CRUD operations working ✅
3. Status tracking implemented ✅
4. Scheduled notifications support ✅
5. Redis queue processor listening ✅

### ⏳ Next Steps (In Order):

#### 1. Test the Module (Now!)
```bash
# Start service
npm run start:dev

# Test creating notification
curl -X POST http://localhost:3000/notifications \
  -H "Content-Type: application/json" \
  -d '{
    "userId": "user_001",
    "type": "email",
    "template": "booking_confirmation",
    "channel": "email"
  }'

# Test querying
curl http://localhost:3000/notifications
curl http://localhost:3000/notifications?status=pending
```

#### 2. Add Email Sending Service (Next Week)
Create `EmailService` that:
- Queries `status=pending` notifications
- Sends via AWS SES
- Updates status to `sent` → `delivered`

#### 3. Add Push Notification Service (Later)
Create `PushService` that:
- Sends to Firebase
- Updates notification status

#### 4. Add Webhooks (Later)
Handle callbacks from:
- AWS SES (email delivered/bounced)
- Firebase (push delivered)

#### 5. Add Analytics (Later)
Track:
- Delivery rates
- Open rates
- Click rates

---

## 🎯 Summary

**NotificationsModule = Notification Database Manager**

**What it does:**
- ✅ Stores all notifications
- ✅ Tracks status (pending → sent → delivered)
- ✅ Queries by user, status, date
- ✅ Supports scheduling
- ✅ Tracks retries and failures

**What calls it:**
- EventsModule (from Redis queue events) ✅
- Cron jobs (scheduled notifications)
- REST API (manual operations)
- Email service (to get pending notifications)

**What it calls:**
- PrismaService (database operations)
- Nothing else! (it's a pure service layer)

**Your job tomorrow:**
1. Read this guide ✅
2. Test the endpoints ✅
3. Understand the flow ✅
4. Start building email service ✅

Good luck! 🚀
