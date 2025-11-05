# 🏗️ Notification Service Architecture

## Overview

Your notification service has **TWO ways** to receive requests:

### 1. 🌐 REST API (Direct Access)
For manual operations, admin tasks, and direct queries.

### 2. 📬 Event Queue (Async from Booking Service)
For automated notifications triggered by booking service events.

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                      Booking Service                             │
│  (Creates bookings, payments, cancellations)                    │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     │ Publishes Events
                     ▼
          ┌──────────────────────┐
          │    Redis Queue       │
          │  (Bull/BullMQ)       │
          └──────────┬───────────┘
                     │
                     │ Consumes Events
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│              Notification Service (This Service)                 │
│                                                                  │
│  ┌─────────────────┐         ┌────────────────────────┐        │
│  │  REST API       │         │  Event Queue Processor │        │
│  │  (port 3000)    │         │  (Background Worker)   │        │
│  │                 │         │                        │        │
│  │  - GET/POST     │         │  - booking.created     │        │
│  │  - CRUD ops     │         │  - booking.confirmed   │        │
│  │  - Queries      │         │  - payment.received    │        │
│  └────────┬────────┘         └──────────┬─────────────┘        │
│           │                             │                       │
│           └──────────┬──────────────────┘                       │
│                      │                                           │
│           ┌──────────▼───────────┐                              │
│           │  Business Logic      │                              │
│           │  (Services)          │                              │
│           │                      │                              │
│           │  - NotificationSvc   │                              │
│           │  - EmailLogsSvc      │                              │
│           │  - TicketsSvc        │                              │
│           └──────────┬───────────┘                              │
│                      │                                           │
│           ┌──────────▼───────────┐                              │
│           │   Prisma (ORM)       │                              │
│           └──────────┬───────────┘                              │
└──────────────────────┼───────────────────────────────────────────┘
                       │
                       ▼
            ┌──────────────────────┐
            │   PostgreSQL         │
            │   (Database)         │
            └──────────────────────┘
```

---

## How It Works

### Scenario 1: Booking Service Creates a Booking

1. **Booking Service** creates a booking
2. **Booking Service** publishes event to Redis queue:
   ```json
   {
     "eventType": "booking.confirmed",
     "eventId": "evt-123",
     "timestamp": "2025-11-02T10:00:00Z",
     "data": {
       "bookingId": "BKG-456",
       "userId": "user_001",
       "userEmail": "john@example.com",
       "parkingName": "Paris CDG"
     }
   }
   ```
3. **Notification Service** queue processor picks up the event
4. **Processor** checks user preferences
5. **Processor** creates notification in database
6. **Processor** sends email (will implement later)

### Scenario 2: Admin Manually Creates Notification

1. **Admin** calls REST API directly:
   ```bash
   POST /notifications
   ```
2. **Controller** receives request
3. **Service** validates and creates notification
4. **Response** returned immediately

---

## Testing Without Booking Service

### Problem
You don't want to run the entire booking service just to test notifications!

### Solution
**Mock Events Controller** simulates booking service events!

#### Test Event Flow:

```bash
# 1. Trigger a mock booking confirmation
POST http://localhost:3000/mock-events/booking-confirmed
{
  "userId": "user_001",
  "userEmail": "test@example.com",
  "bookingId": "BKG-TEST-001",
  "parkingName": "Test Parking"
}

# 2. The mock event goes to Redis queue
# 3. Queue processor picks it up
# 4. Notification is created
# 5. Check the result:
GET http://localhost:3000/notifications
```

---

## Event Types Supported

From Booking Service:

1. **booking.created** - New booking created
2. **booking.confirmed** - Booking confirmed (payment successful)
3. **booking.cancelled** - Booking cancelled
4. **booking.reminder** - Reminder before booking starts
5. **payment.received** - Payment successful
6. **payment.failed** - Payment failed
7. **payment.refunded** - Refund processed

---

## File Structure

```
src/
├── modules/
│   ├── events/                     ← NEW! Event-driven architecture
│   │   ├── types/
│   │   │   └── booking-events.types.ts    # Event type definitions
│   │   ├── processors/
│   │   │   └── notification-queue.processor.ts  # Consumes events
│   │   ├── services/
│   │   │   └── events-producer.service.ts      # Produces mock events
│   │   ├── controllers/
│   │   │   └── mock-events.controller.ts       # Testing endpoints
│   │   └── events.module.ts
│   │
│   ├── notifications/              # REST API + Business Logic
│   ├── email-logs/
│   ├── templates/
│   ├── tickets/
│   └── preferences/
└── app.module.ts
```

---

## Development vs Production

### Development (Current Setup)
- ✅ REST API active
- ✅ Queue processor active
- ✅ Mock events controller active (for testing)
- ✅ Redis required (localhost:6379)

### Production (Future)
- ✅ REST API active
- ✅ Queue processor active
- ❌ Mock events controller **DISABLED** (remove or protect)
- ✅ Redis required (production instance)
- ✅ Real events from booking service

---

## Dependencies

### Already Installed
- `@nestjs/bull` - Queue management
- `bull` - Redis-based queue
- `redis` - Redis client

### Required Services
- **PostgreSQL** (port 5432) - Database
- **Redis** (port 6379) - Queue

---

## Quick Start Commands

```bash
# 1. Start Redis (if not running)
# Windows: Run Redis from Windows Subsystem for Linux (WSL)
# Or use Docker:
docker run -d -p 6379:6379 redis:alpine

# 2. Start notification service
npm run start:dev

# 3. Test with mock event
curl -X POST http://localhost:3000/mock-events/booking-confirmed \
  -H "Content-Type: application/json" \
  -d '{"userId": "user_001", "bookingId": "TEST-001"}'

# 4. Check notification was created
curl http://localhost:3000/notifications
```

---

## When Booking Service Is Ready

When your booking service is implemented, it will:

1. **Publish events** to the same Redis queue
2. **Use the same event types** (booking.created, etc.)
3. **Notification service automatically picks them up**
4. **No changes needed** to notification service!

### In Booking Service (Future):
```typescript
// In booking service
await queue.add('booking-event', {
  eventType: 'booking.confirmed',
  eventId: uuidv4(),
  timestamp: new Date().toISOString(),
  data: {
    bookingId: booking.id,
    userId: booking.userId,
    // ... etc
  }
});
```

---

## Monitoring Queue

### Bull Board (Optional)
View queue status in browser:

```bash
npm install @bull-board/express @bull-board/api

# Access at: http://localhost:3000/admin/queues
```

### Redis CLI
```bash
redis-cli
> LLEN bull:notifications:waiting
> LLEN bull:notifications:active
> LLEN bull:notifications:completed
> LLEN bull:notifications:failed
```

---

## Summary

✅ **Current Setup:**
- REST API for direct access ✓
- Queue processor for async events ✓
- Mock events for testing WITHOUT booking service ✓

✅ **You Can:**
- Test notifications independently
- Don't need booking service running
- Use mock events to simulate real scenarios

✅ **When Booking Service Ready:**
- It publishes real events to Redis
- Notification service automatically consumes them
- Just disable mock events controller in production!

---

## Next Steps

1. ✅ Start Redis
2. ✅ Test mock events
3. ⏳ Add email sending (AWS SES, SendGrid)
4. ⏳ Add push notifications (Firebase)
5. ⏳ Connect to real booking service when ready
