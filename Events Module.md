# Events Module - Complete Guide

## What Problem Are We Solving?

In a microservices architecture, **different services need to communicate**. When a user books a parking spot in the **Booking Service**, we need to send them notifications (emails, push notifications, etc.) from the **Notification Service**.

But there's a challenge:
- What if the notification service is down when a booking happens?
- What if sending 1000 emails crashes the system?
- How do we handle failures and retries?

**Solution:** We use an **Event-Driven Architecture** with **Redis Queues (BullMQ)**

---

## The Big Picture

```
┌─────────────────┐         ┌─────────────┐         ┌──────────────────────┐
│ Booking Service │ ──────> │ Redis Queue │ ──────> │ Notification Service │
│ (Other service) │  Event  │   (BullMQ)  │   Job   │   (This service)     │
└─────────────────┘         └─────────────┘         └──────────────────────┘
                                                              │
                                                              ▼
                                                     ┌─────────────────┐
                                                     │ Send Email/Push │
                                                     └─────────────────┘
```

### How It Works:

1. **Booking Service** creates a booking → Pushes an event to Redis queue
2. **Redis Queue** stores the event safely (even if notification service is down)
3. **Notification Service** pulls the event from queue → Processes it → Sends notification
4. If processing fails, **Redis automatically retries** (3-5 times with delays)

---

## File Structure & Purpose

```
src/modules/events/
├── config/
│   └── queue-routing.config.ts     # Maps events to queues
├── controllers/
│   └── mock-events.controller.ts   # Test endpoints (for development)
├── processors/
│   └── notification-queue.processor.ts  # Processes jobs from queue
├── services/
│   └── events-producer.service.ts  # Puts events into queue
├── types/
│   └── booking-events.types.ts     # Event data structures
└── events.module.ts                # Module configuration
```

---

## File-by-File Explanation

### 1. `types/booking-events.types.ts`
**What it does:** Defines the structure of events

**Think of it as:** A contract between services - "Here's what data you'll receive"

**Contains:**
- 7 event types: `booking.created`, `booking.confirmed`, `booking.cancelled`, etc.
- TypeScript interfaces for each event with required fields

**Example:**
```typescript
interface BookingCreatedEvent {
  eventType: 'booking.created';
  eventId: string;
  timestamp: string;
  data: {
    bookingId: string;
    userId: string;
    userEmail: string;
    // ... more fields
  };
}
```

**Why it's important:** Ensures type safety - you can't send wrong data

---

### 2. `config/queue-routing.config.ts`
**What it does:** Maps which events go to which queues

**Think of it as:** A traffic controller directing events to the right lane

**Contains:**
- 3 Queue names: `emails`, `notifications`, `tickets`
- Static mapping:
  - `booking.created` → `emails` queue
  - `booking.reminder` → `notifications` queue
  - `payment.failed` → `tickets` queue

**Why we need this:**
- Different queues have different priorities and retry strategies
- Emails need more retries (5 attempts) because email servers can be flaky
- Notifications need fast processing (3 attempts, shorter timeout)

**Example:**
```typescript
QUEUE_ROUTING_MAP = {
  'booking.created': 'emails',      // Send via email queue
  'booking.reminder': 'notifications', // Send via push notifications
  'payment.failed': 'tickets',      // Create support ticket
}
```

---

### 3. `controllers/mock-events.controller.ts`
**What it does:** Creates HTTP endpoints to manually trigger events

**Think of it as:** A testing tool - like pressing a "simulate booking" button

**Endpoints:**
- `POST /mock-events/booking-created` → Simulates a booking creation
- `POST /mock-events/booking-confirmed` → Simulates confirmation
- `POST /mock-events/payment-failed` → Simulates payment failure
- ... 6 endpoints total

**Why we built this:**
- You can test the notification service WITHOUT the booking service running
- You can use Postman to trigger events manually
- Perfect for development and debugging

**Production Note:** Remove this controller in production - real events will come from the actual Booking Service

**Example Request:**
```bash
POST http://localhost:3000/mock-events/booking-created
Body: {}
```

This creates a fake booking event with default data and pushes it to the queue.

---

### 4. `services/events-producer.service.ts`
**What it does:** Puts events into the Redis queue

**Think of it as:** The "sender" - takes an event and pushes it to Redis

**Key Method:**
```typescript
async produceEvent(event: BookingEvent) {
  await this.notificationQueue.add('booking-event', event, {
    attempts: 3,        // Retry 3 times if fails
    backoff: {
      type: 'exponential',
      delay: 2000,      // Wait 2s, then 4s, then 8s
    },
  });
}
```

**What happens:**
1. Receives an event (e.g., `booking.created`)
2. Pushes it to the `notifications` queue in Redis
3. Redis stores it persistently
4. Returns immediately (doesn't wait for processing)

**Why it's important:** Decouples event creation from processing - even if notification service crashes, events are safe in Redis

---

### 5. `processors/notification-queue.processor.ts`
**What it does:** Pulls events from the queue and processes them

**Think of it as:** The "worker" - continuously checks Redis for new jobs and executes them

**How it works:**
1. BullMQ automatically calls `process()` when a new job arrives
2. Checks the event type (`booking.created`, `payment.failed`, etc.)
3. Checks user preferences (Does user want email notifications?)
4. Calls the appropriate handler (`handleBookingCreated`, etc.)
5. Each handler creates a notification via `NotificationsService`

**Key Features:**
- **Concurrency:** Processes 5 jobs at the same time
- **Automatic Retries:** If processing fails, BullMQ retries (3-5 times)
- **User Preferences:** Respects user's notification settings
- **Logging:** Logs every step for debugging

**Example Flow:**
```
Event arrives → Check preferences → Route to handler → Create notification → Mark as complete
```

**If error occurs:**
```
Error → Log error → Throw exception → BullMQ retries automatically
```

---

### 6. `events.module.ts`
**What it does:** Configures and wires everything together

**Think of it as:** The "glue" that connects all pieces

**Contains:**
- Imports BullMQ module for queue management
- Registers the 3 queues: `emails`, `notifications`, `tickets`
- Connects processor to listen to queues
- Exports services for other modules to use

**Why it's important:** NestJS needs this to know what services/controllers to load

---

## The Complete Flow

Let's trace what happens when a user creates a booking:

### Step 1: Event is Created
```
User creates booking in Booking Service
  ↓
Booking Service calls:
POST http://notification-service/mock-events/booking-created
```

### Step 2: Event Goes to Queue
```
MockEventsController receives the request
  ↓
Creates BookingCreatedEvent object
  ↓
EventsProducerService.produceEvent(event)
  ↓
Event is pushed to Redis "notifications" queue
  ↓
Controller returns immediately: {eventId: "123", message: "Event queued"}
```

### Step 3: Event is Processed
```
Redis has the job stored
  ↓
NotificationQueueProcessor automatically picks it up
  ↓
process() method is called
  ↓
handleBookingEvent() checks user preferences
  ↓
handleBookingCreated() creates notification
  ↓
NotificationsService sends the email/push
  ↓
Job marked as complete ✓
```

### If Something Fails:
```
Email service is down ❌
  ↓
Error is thrown
  ↓
BullMQ catches the error
  ↓
Waits 2 seconds
  ↓
Retries (Attempt 2 of 3)
  ↓
Still fails ❌
  ↓
Waits 4 seconds (exponential backoff)
  ↓
Retries (Attempt 3 of 3)
  ↓
Success ✓ → Job complete
```

---

## Why This Architecture?

### Benefits:

1. **Reliability**
   - Events are never lost (stored in Redis)
   - Automatic retries on failures
   - Persistent storage

2. **Scalability**
   - Can process 5 jobs concurrently
   - Can add more workers to process faster
   - Services are independent

3. **Fault Tolerance**
   - If notification service crashes, events wait in queue
   - When service restarts, processing resumes
   - No data loss

4. **Flexibility**
   - Easy to add new event types
   - Can route events to different queues
   - Different retry strategies per queue

5. **Testing**
   - Mock controller allows independent testing
   - No dependency on booking service for development

---

## Smart Event Routing System

### How Routing Works

When an event arrives, the system automatically routes it to the correct queue based on the event type:

```typescript
// Event arrives
booking.created → Check routing map → Route to "emails" queue
booking.reminder → Check routing map → Route to "notifications" queue
payment.failed → Check routing map → Route to "tickets" queue
```

**Implementation:** `services/events-producer.service.ts`

```typescript
// 1. Get the target queue name from routing config
const targetQueueName = getQueueForEvent(event.eventType);
// Result: 'emails', 'notifications', or 'tickets'

// 2. Get the actual queue instance
const targetQueue = this.queues.get(targetQueueName);

// 3. Get queue-specific retry configuration
const queueConfig = QUEUE_CONFIG[targetQueueName];

// 4. Add job to the correct queue with its config
await targetQueue.add('booking-event', event, {
  attempts: queueConfig.attempts,
  backoff: queueConfig.backoff,
});
```

**Benefits:**
- ✅ Events automatically go to the right queue
- ✅ Each queue has its own retry strategy
- ✅ Easy to add new event types (just update the routing map)
- ✅ Type-safe routing with TypeScript

---

## Retry & Backoff: Why You Need It

### The Problem: Things Fail in Production

Real-world failures that happen:
- **Amazon SES is down** → Email can't be sent
- **Firebase timeout** → Push notification fails
- **Database connection pool full** → Ticket creation fails
- **Network hiccup** → API call drops

**Without retry:** Event is lost forever → User never gets notification → Angry customer calls support

**With retry:** System automatically tries again → Notification eventually sent → Happy customer

---

### How Retry & Backoff Work

#### Exponential Backoff (for Emails & Notifications)

**Strategy:** Wait longer after each failure

```
Attempt 1: Send → ❌ FAIL (SES timeout)
Wait 2 seconds...

Attempt 2: Send → ❌ FAIL (SES still down)
Wait 4 seconds... (2 × 2)

Attempt 3: Send → ❌ FAIL
Wait 8 seconds... (2 × 2 × 2)

Attempt 4: Send → ❌ FAIL
Wait 16 seconds... (2 × 2 × 2 × 2)

Attempt 5: Send → ✅ SUCCESS! (SES back online)
```

**Why exponential?**
- Gives external services time to recover
- Prevents hammering a failing service
- Gradually backs off to avoid making problems worse

---

#### Fixed Backoff (for Tickets)

**Strategy:** Wait the same time after each failure

```
Attempt 1: Create ticket → ❌ FAIL (database busy)
Wait 5 seconds...

Attempt 2: Create ticket → ❌ FAIL (still busy)
Wait 5 seconds...

Attempt 3: Create ticket → ✅ SUCCESS!
```

**Why fixed?**
- Database issues usually resolve quickly
- Consistent timing is predictable
- Simple for operations that don't benefit from long waits

---

## The 3 Queues Explained

### Queue #1: `emails`
**Purpose:** Send email notifications (critical communications)

**Events Routed Here:**
- `booking.created` → Booking confirmation email
- `booking.confirmed` → Payment confirmation email
- `booking.cancelled` → Cancellation confirmation email

**Retry Configuration:**
```typescript
{
  attempts: 5,                    // Try 5 times total
  backoff: {
    type: 'exponential',          // Wait longer each time
    delay: 2000,                  // Start with 2 seconds
  }
}
```

**Why 5 attempts?**
- Emails are CRITICAL (booking confirmations, payment receipts)
- Email services (SES) can be flaky
- Users will call support if they don't get confirmation

**Why exponential backoff?**
- SES might hit rate limits → need time to reset quota
- Don't spam SES when it's having issues
- Give network/service time to recover

**Real Retry Timeline:**
```
Total time if all attempts fail: 2s + 4s + 8s + 16s + 32s = 62 seconds
```

---

### Queue #2: `notifications`
**Purpose:** Send push/in-app notifications (real-time updates)

**Events Routed Here:**
- `booking.reminder` → "Your parking starts in 24 hours"
- `payment.received` → "Payment received"

**Retry Configuration:**
```typescript
{
  attempts: 3,                    // Try 3 times total
  backoff: {
    type: 'exponential',          // Wait longer each time
    delay: 1000,                  // Start with 1 second
  }
}
```

**Why 3 attempts?**
- Push notifications are important but not critical
- Users can still see them later in-app
- Faster failure = quicker feedback loop

**Why exponential backoff?**
- Firebase FCM can timeout under load
- Network issues resolve quickly
- Shorter delays keep notifications timely

**Real Retry Timeline:**
```
Total time if all attempts fail: 1s + 2s + 4s = 7 seconds
```

---

### Queue #3: `tickets`
**Purpose:** Create support tickets (automatic issue tracking)

**Events Routed Here:**
- `payment.failed` → Creates a support ticket for investigation

**Retry Configuration:**
```typescript
{
  attempts: 3,                    // Try 3 times total
  backoff: {
    type: 'fixed',                // Always wait the same time
    delay: 5000,                  // Always wait 5 seconds
  }
}
```

**Why 3 attempts?**
- Ticket creation should work or fail quickly
- Database is usually reliable
- Support team needs to know about issues fast

**Why fixed backoff?**
- Database connection issues resolve in seconds
- No benefit to exponential growth
- Predictable timing for debugging

**Real Retry Timeline:**
```
Total time if all attempts fail: 5s + 5s + 5s = 15 seconds
```

---

## Visual Comparison: With vs Without Retry

### ❌ WITHOUT Retry Config (BAD)

```typescript
// Old code (don't do this)
await queue.add('booking-event', event);
```

**What happens:**
```
10:00:00 - Event arrives: booking.created
10:00:00 - Try to send email via SES
10:00:01 - ❌ SES timeout error
10:00:01 - Event LOST FOREVER
Result: User never gets confirmation email
        User calls support: "I didn't get my confirmation!"
        Support team wastes 15 minutes resending manually
```

---

### ✅ WITH Retry Config (GOOD)

```typescript
// Current code (production-ready)
await queue.add('booking-event', event, {
  attempts: 5,
  backoff: { type: 'exponential', delay: 2000 }
});
```

**What happens:**
```
10:00:00 - Event arrives: booking.created
10:00:00 - Try to send email via SES
10:00:01 - ❌ SES timeout error
10:00:03 - Retry #2 → ❌ Still failing
10:00:07 - Retry #3 → ❌ Still failing
10:00:15 - Retry #4 → ❌ Still failing
10:00:31 - Retry #5 → ✅ SUCCESS! Email sent
Result: User gets confirmation email (31 seconds later)
        User is happy
        No support tickets
        System self-healed
```

---

## Configuration Reference

**Location:** `config/queue-routing.config.ts`

```typescript
// Complete routing map
export const QUEUE_ROUTING_MAP = {
  'booking.created': 'emails',      // Critical confirmation
  'booking.confirmed': 'emails',    // Critical confirmation
  'booking.cancelled': 'emails',    // Critical confirmation
  'booking.reminder': 'notifications', // Timely reminder
  'payment.received': 'notifications', // Quick feedback
  'payment.failed': 'tickets',      // Auto-create support ticket
};

// Queue-specific retry configs
export const QUEUE_CONFIG = {
  emails: {
    attempts: 5,
    backoff: { type: 'exponential', delay: 2000 }
  },
  notifications: {
    attempts: 3,
    backoff: { type: 'exponential', delay: 1000 }
  },
  tickets: {
    attempts: 3,
    backoff: { type: 'fixed', delay: 5000 }
  },
};
```

---

## When to Add More Event Types

### Current Events (MVP - 6 events) ✅
These cover the critical user journey:
1. User books → `booking.created`
2. Payment succeeds → `booking.confirmed`
3. Payment fails → `payment.failed` → Creates ticket
4. 24hr before → `booking.reminder`
5. User cancels → `booking.cancelled`
6. Payment received → `payment.received`

### Future Events (Add When Needed) 🔮
**Don't add these yet!** Wait until:
- Real Booking/Payment services are running
- You have user feedback about missing notifications
- You have actual business requirements

Potential future events:
- `booking.modified` - User changes reservation
- `booking.checked_in` - User arrives at parking
- `booking.extended` - User extends stay
- `user.registered` - Welcome email
- `parking.availability_low` - Admin alert

**How to add:**
```typescript
// Step 1: Add to enum
export enum EventType {
  // ... existing events
  BOOKING_MODIFIED = 'booking.modified',
}

// Step 2: Add to routing map
export const QUEUE_ROUTING_MAP = {
  // ... existing mappings
  [EventType.BOOKING_MODIFIED]: QueueName.EMAILS,
};

// Step 3: Done! Routing automatically works
```

---

## How to Test

### 1. Start Redis
```bash
docker run -d -p 6379:6379 --name redis-notification redis:alpine
```

### 2. Start Notification Service
```bash
npm run start:dev
```

### 3. Open Queue Dashboard
```
http://localhost:3000/queues
```
You'll see all 3 queues and their jobs

### 4. Trigger an Event (Postman)
```
POST http://localhost:3000/mock-events/booking-created
Headers: Content-Type: application/json
Body: {}
```

### 5. Watch the Magic
- Check the dashboard → You'll see the job appear in the queue
- Check logs → You'll see processing messages
- Check database → Notification is created

---

## In Production

### What Changes:

1. **Remove Mock Controller**
   - Delete `mock-events.controller.ts`
   - Real events come from Booking Service

2. **Booking Service Integration**
   - Booking Service pushes events directly to Redis
   - No HTTP calls needed
   - Events go straight to queue

3. **Queue Configuration**
   - Adjust retry attempts based on actual failure rates
   - Monitor queue performance
   - Scale workers if needed

### Production Flow:
```
Booking Service → Redis Queue → Notification Service → Email/Push
        ↑                                    ↓
    (Direct)                          (No controller)
```

---

## Key Concepts to Remember

### 1. Event-Driven Architecture
Events describe **what happened**, not **what to do**

Example:
- Event: "booking.created" (what happened)
- Not: "send-email" (what to do)

This allows multiple services to react to the same event differently.

---

### 2. Producer-Consumer Pattern

**Producer** (EventsProducerService):
- Creates jobs
- Adds them to queue
- Returns immediately

**Consumer** (NotificationQueueProcessor):
- Picks up jobs from queue
- Processes them
- Marks as complete or failed

They run independently and don't know about each other.

---

### 3. Job vs Event

**Event:** Raw data about what happened
```javascript
{
  eventType: 'booking.created',
  data: { bookingId: '123', userId: 'user1' }
}
```

**Job:** Event wrapped with metadata for queue
```javascript
{
  id: 'job-456',
  name: 'booking-event',
  data: { eventType: 'booking.created', ... },
  attempts: 0,
  timestamp: 1699999999
}
```

---

## Common Questions

**Q: Why use Redis instead of direct HTTP calls?**
A: HTTP calls fail if the service is down. Redis stores events safely until they can be processed.

**Q: Why separate queues for emails/notifications/tickets?**
A: Different priorities and retry strategies. Emails need more retries, notifications need speed.

**Q: What happens if Redis crashes?**
A: If configured with persistence, events are saved to disk. When Redis restarts, events are restored.

**Q: Can we process events in order?**
A: Yes! BullMQ supports FIFO (First In First Out) processing. Currently we process concurrently (5 at a time).

**Q: How do we monitor failed jobs?**
A: Use the Bull Board dashboard at `/queues` to see failed jobs and retry them manually.

---

## Summary

This Events Module creates a **robust, scalable, fault-tolerant system** for handling booking events and sending notifications.

**Key Takeaways:**
- Events are stored safely in Redis queues
- Processing happens asynchronously
- Failures are handled with automatic retries
- Services are decoupled and independent
- Easy to test with mock endpoints
- Scalable to handle thousands of events per second

You've built a production-ready event processing system! 🎉
