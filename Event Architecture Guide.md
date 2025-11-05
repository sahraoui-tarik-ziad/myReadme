# 🎯 Event Architecture Guide - Complete Reference

## Technology Stack for Queues

### What We're Using:
- **Bull** (v4.x) - Redis-based queue for Node.js
- **@nestjs/bull** (v11.x) - NestJS integration for Bull
- **Redis** (v5.x) - In-memory data store (queue backend)

### Why Bull (not BullMQ)?
- **Bull** - Stable, mature, widely used (we're using this)
- **BullMQ** - Newer version, rewrite of Bull (migration path for future)

```json
// package.json
{
  "dependencies": {
    "@nestjs/bull": "^11.0.4",
    "bull": "^4.16.5",
    "redis": "^5.8.3"
  }
}
```

### Queue Configuration:
```typescript
// app.module.ts
BullModule.forRoot({
  redis: {
    host: process.env.REDIS_HOST || 'localhost',
    port: parseInt(process.env.REDIS_PORT || '6379', 10),
    password: process.env.REDIS_PASSWORD,
  },
})
```

---

## 📬 Complete Event Architecture

### Overview: 3 Sources of Notifications

```
┌─────────────────────────────────────────────────────────────┐
│                    1. BOOKING SERVICE                        │
│               (External - Via Redis Queue)                   │
│  ├─ booking.created                                          │
│  ├─ booking.confirmed                                        │
│  ├─ booking.cancelled                                        │
│  ├─ payment.received                                         │
│  ├─ payment.failed                                           │
│  └─ payment.refunded                                         │
└─────────────┬────────────────────────────────────────────────┘
              │
              ▼ Publishes to Redis
       ┌─────────────┐
       │    Redis    │
       │    Queue    │
       └──────┬──────┘
              │
              ▼ Consumes
┌─────────────────────────────────────────────────────────────┐
│              NOTIFICATION SERVICE (YOU)                     │
│                                                             │
│  🎧 Queue Processor (NotificationQueueProcessor)           │
│     └─ Listens 24/7 for events                             │
│                                                             │
│  ⏰ Scheduled Jobs (Cron)                                  │
│     └─ Marketing, Partner Digests, Reminders               │
│                                                             │
│  🎫 Internal Triggers                                      │
│     └─ Support ticket notifications                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
              │
              ▼
      [Send Emails/Push/SMS]


┌─────────────────────────────────────────────────────────────┐
│           2. SCHEDULED JOBS (Internal - Cron)                │
│                (YOU generate these)                          │
│  ├─ Marketing campaigns                                      │
│  ├─ Partner weekly digests                                   │
│  ├─ Booking reminders (24h, 2h before)                      │
│  └─ Re-engagement campaigns                                  │
└─────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────┐
│          3. TICKET EVENTS (Internal - Auto-triggered)        │
│               (Inside notification service)                  │
│  ├─ Ticket created → Notify agent                           │
│  ├─ Ticket assigned → Notify agent                          │
│  ├─ Comment added → Notify user/agent                       │
│  ├─ Ticket escalated → Notify manager                       │
│  └─ Ticket resolved → Notify user                           │
└─────────────────────────────────────────────────────────────┘
```

---

## 1️⃣ Booking Service Events (External)

### Event Structure - Complete Format

```typescript
{
  eventType: "booking.confirmed",      // Event name
  eventId: "550e8400-e29b-41d4...",   // Unique ID (for deduplication)
  timestamp: "2025-11-02T10:00:00Z",  // When event occurred
  data: {
    // Business data varies by event type
    bookingId: "BKG-001",
    userId: "user_001",
    userEmail: "test@example.com",
    userName: "Jean Dupont",
    parkingName: "Parking Charles de Gaulle",
    // ... more fields specific to event type
  }
}
```

### All Event Types from Booking Service

#### 1. booking.created
```typescript
{
  eventType: "booking.created",
  eventId: "uuid",
  timestamp: "2025-11-02T10:00:00Z",
  data: {
    bookingId: "BKG-001",
    userId: "user_001",
    userEmail: "test@example.com",
    userName: "Jean Dupont",
    parkingName: "Parking Charles de Gaulle",
    parkingAddress: "1 Rue de la Paix, Paris",
    startDate: "2025-11-15T08:00:00Z",
    endDate: "2025-11-15T18:00:00Z",
    totalAmount: 45.50,
    currency: "EUR",
    status: "pending"
  }
}
```

**Notification to send:** "Booking created, awaiting payment"

---

#### 2. booking.confirmed
```typescript
{
  eventType: "booking.confirmed",
  eventId: "uuid",
  timestamp: "2025-11-02T10:05:00Z",
  data: {
    bookingId: "BKG-001",
    userId: "user_001",
    userEmail: "test@example.com",
    userName: "Jean Dupont",
    parkingName: "Parking Charles de Gaulle",
    startDate: "2025-11-15T08:00:00Z",
    endDate: "2025-11-15T18:00:00Z",
    confirmationCode: "CONF-ABC123",
    qrCode: "https://storage.../qr-code.png"
  }
}
```

**Notification to send:** "Booking confirmed! Here's your QR code"

---

#### 3. booking.cancelled
```typescript
{
  eventType: "booking.cancelled",
  eventId: "uuid",
  timestamp: "2025-11-02T14:00:00Z",
  data: {
    bookingId: "BKG-001",
    userId: "user_001",
    userEmail: "test@example.com",
    userName: "Jean Dupont",
    parkingName: "Parking Charles de Gaulle",
    cancellationReason: "Plans changed",
    refundAmount: 45.50,
    refundDate: "2025-11-05T00:00:00Z"
  }
}
```

**Notification to send:** "Booking cancelled, refund processed"

---

#### 4. payment.received
```typescript
{
  eventType: "payment.received",
  eventId: "uuid",
  timestamp: "2025-11-02T10:05:00Z",
  data: {
    paymentId: "PAY-001",
    bookingId: "BKG-001",
    userId: "user_001",
    userEmail: "test@example.com",
    amount: 45.50,
    currency: "EUR",
    paymentMethod: "credit_card",
    transactionId: "TXN-123456",
    last4Digits: "4242"
  }
}
```

**Notification to send:** "Payment confirmed - €45.50"

---

#### 5. payment.failed
```typescript
{
  eventType: "payment.failed",
  eventId: "uuid",
  timestamp: "2025-11-02T10:05:30Z",
  data: {
    paymentId: "PAY-002",
    bookingId: "BKG-002",
    userId: "user_002",
    userEmail: "user2@example.com",
    amount: 30.00,
    failureReason: "Insufficient funds",
    failureCode: "card_declined"
  }
}
```

**Notification to send:** "Payment failed, please update payment method"

---

#### 6. payment.refunded
```typescript
{
  eventType: "payment.refunded",
  eventId: "uuid",
  timestamp: "2025-11-05T09:00:00Z",
  data: {
    refundId: "REF-001",
    paymentId: "PAY-001",
    bookingId: "BKG-001",
    userId: "user_001",
    userEmail: "test@example.com",
    refundAmount: 45.50,
    refundReason: "Booking cancelled by user",
    processingDays: "3-5 business days"
  }
}
```

**Notification to send:** "Refund processed - €45.50"

---

### How Your Service Handles These

**File:** `src/modules/events/processors/notification-queue.processor.ts`

```typescript
@Processor('notifications')
export class NotificationQueueProcessor {

  @Process('booking-event')
  async handleBookingEvent(job: Job<BookingEvent>) {
    // 1. Check user preferences
    const preferences = await this.preferencesService.getPreferences(
      job.data.data.userId
    );

    if (!preferences.emailEnabled) {
      return; // User disabled notifications
    }

    // 2. Route to handler based on event type
    switch (job.data.eventType) {
      case BookingEventType.BOOKING_CONFIRMED:
        await this.handleBookingConfirmed(job.data);
        break;
      case BookingEventType.PAYMENT_RECEIVED:
        await this.handlePaymentReceived(job.data);
        break;
      // ... etc
    }
  }

  private async handleBookingConfirmed(event: any) {
    // Create notification in database
    await this.notificationsService.create({
      userId: event.data.userId,
      type: 'email',
      template: 'booking_confirmation',
      subject: 'Confirmation de votre réservation',
      content: {
        userName: event.data.userName,
        parkingName: event.data.parkingName,
        confirmationCode: event.data.confirmationCode,
      },
      channel: 'email',
      priority: 'high',
      metadata: {
        bookingId: event.data.bookingId,
        eventId: event.eventId,
      },
    });

    // Future: Actually send email here
    // await this.emailService.send(...);
  }
}
```

---

## 2️⃣ Scheduled Jobs (Internal - YOU Generate)

### What Are These?

These are **automated notifications** that run on a schedule (cron jobs). The booking service does NOT send these - YOU create them.

### Marketing Campaigns

#### Weekly Newsletter
```typescript
// src/modules/schedulers/marketing-scheduler.service.ts

@Injectable()
export class MarketingSchedulerService {

  @Cron('0 9 * * 1') // Every Monday at 9:00 AM
  async sendWeeklyNewsletter() {
    // 1. Get users who want marketing emails
    const users = await this.prisma.notificationPreference.findMany({
      where: {
        marketingEnabled: true,
        emailEnabled: true
      }
    });

    // 2. Get featured parkings (call booking service API)
    const featuredParkings = await this.bookingServiceApi.getFeaturedParkings();

    // 3. Create notification for each user
    for (const user of users) {
      await this.notificationsService.create({
        userId: user.userId,
        type: 'email',
        template: 'weekly_newsletter',
        subject: 'Nos parkings de la semaine',
        content: {
          userName: user.userName,
          featuredParkings: featuredParkings,
          specialOffers: [...],
        },
        channel: 'email',
        priority: 'low',
        metadata: {
          campaign: 'weekly_newsletter',
          week: getCurrentWeek(),
        }
      });
    }
  }

  @Cron('0 10 * * 3') // Every Wednesday at 10:00 AM
  async sendSpecialOffers() {
    // Similar logic for special offers
  }
}
```

#### Inactive User Re-engagement
```typescript
@Cron('0 14 * * *') // Daily at 2:00 PM
async sendInactiveUserReminder() {
  // 1. Find users who haven't booked in 30 days
  const inactiveUsers = await this.findInactiveUsers(30);

  // 2. Send re-engagement email
  for (const user of inactiveUsers) {
    await this.notificationsService.create({
      userId: user.id,
      type: 'email',
      template: 'reengagement_campaign',
      subject: 'Nous vous avons manqué!',
      content: {
        userName: user.name,
        specialDiscount: '20%',
        nearbyParkings: [...],
      },
      channel: 'email',
      priority: 'low',
    });
  }
}
```

---

### Partner Weekly Digest

```typescript
// src/modules/schedulers/partner-scheduler.service.ts

@Injectable()
export class PartnerSchedulerService {

  @Cron('0 8 * * 0') // Every Sunday at 8:00 AM
  async sendPartnerWeeklyDigest() {
    // 1. Get all partners who want digest
    const partners = await this.prisma.notificationPreference.findMany({
      where: { partnerDigestEnabled: true }
    });

    for (const partner of partners) {
      // 2. Calculate stats by calling booking service API
      const stats = await this.calculatePartnerStats(partner.userId);

      // 3. Create notification
      await this.notificationsService.create({
        userId: partner.userId,
        type: 'email',
        template: 'partner_weekly_digest',
        subject: 'Votre rapport hebdomadaire',
        content: {
          period: 'Semaine 44',
          totalBookings: stats.bookings,
          totalRevenue: stats.revenue,
          occupancyRate: stats.occupancyRate,
          topParking: stats.topParking,
          revenueChange: stats.revenueChange, // "+12%"
        },
        channel: 'email',
        priority: 'normal',
        metadata: {
          partnerId: partner.userId,
          weekStart: stats.weekStart,
          weekEnd: stats.weekEnd,
        }
      });
    }
  }

  private async calculatePartnerStats(partnerId: string) {
    const lastWeekStart = getLastWeekStart();
    const lastWeekEnd = getLastWeekEnd();

    // Call booking service API to get stats
    const response = await this.httpService.get(
      `${BOOKING_SERVICE_URL}/api/partners/${partnerId}/stats`,
      {
        params: {
          startDate: lastWeekStart,
          endDate: lastWeekEnd,
        }
      }
    );

    return response.data;
  }
}
```

---

### Booking Reminders

**Note:** These can be generated TWO ways:

#### Option A: Scheduled Job (YOU check database)
```typescript
@Cron('0 */1 * * *') // Every hour
async sendBookingReminders() {
  const now = new Date();
  const in24Hours = addHours(now, 24);
  const in2Hours = addHours(now, 2);

  // Find bookings starting in 24 hours
  const bookings24h = await this.findBookingsStartingBetween(
    in24Hours,
    addMinutes(in24Hours, 30)
  );

  for (const booking of bookings24h) {
    await this.notificationsService.create({
      userId: booking.userId,
      type: 'push',
      template: 'booking_reminder_24h',
      subject: 'Rappel: Réservation demain',
      content: {
        parkingName: booking.parkingName,
        startTime: booking.startTime,
      },
      channel: 'push',
      priority: 'urgent',
    });
  }

  // Same for 2-hour reminders
}
```

#### Option B: Booking Service Sends Event
```typescript
// Booking service sends:
{
  eventType: "booking.reminder",
  data: {
    bookingId: "BKG-001",
    userId: "user_001",
    hoursUntilStart: 24,
    // ...
  }
}

// Your processor handles it like other events
```

**Recommendation:** Option A (scheduled job) gives you more control.

---

## 3️⃣ Support Ticket Events (Internal)

These are triggered when actions happen in the tickets module.

### Implementation

```typescript
// src/modules/tickets/tickets.service.ts

@Injectable()
export class TicketsService {

  async create(createDto: CreateTicketDto) {
    // 1. Create ticket
    const ticket = await this.prisma.supportTicket.create({
      data: createDto
    });

    // 2. Trigger notification to assigned agent
    if (ticket.assignedTo) {
      await this.notificationsService.create({
        userId: ticket.assignedTo,
        type: 'in_app',
        template: 'ticket_assigned',
        subject: 'Nouveau ticket assigné',
        content: {
          ticketNumber: ticket.ticketNumber,
          subject: ticket.subject,
          priority: ticket.priority,
        },
        channel: 'in_app',
        priority: ticket.priority === 'urgent' ? 'urgent' : 'normal',
      });
    }

    // 3. Notification to user (ticket created)
    await this.notificationsService.create({
      userId: ticket.userId,
      type: 'email',
      template: 'ticket_created',
      subject: 'Votre demande a été reçue',
      content: {
        ticketNumber: ticket.ticketNumber,
        estimatedResponse: '4 hours',
      },
      channel: 'email',
      priority: 'normal',
    });

    return ticket;
  }

  async update(id: string, updateDto: UpdateTicketDto) {
    const ticket = await this.findOne(id);

    // If status changed to resolved
    if (updateDto.status === 'resolved' && ticket.status !== 'resolved') {
      await this.notificationsService.create({
        userId: ticket.userId,
        type: 'email',
        template: 'ticket_resolved',
        subject: 'Votre demande a été résolue',
        content: {
          ticketNumber: ticket.ticketNumber,
          resolution: updateDto.resolution,
        },
        channel: 'email',
        priority: 'normal',
      });
    }

    // If escalated
    if (updateDto.escalated && !ticket.escalated) {
      await this.notificationsService.create({
        userId: 'manager-id', // or get from config
        type: 'in_app',
        template: 'ticket_escalated',
        subject: 'Ticket escaladé',
        content: {
          ticketNumber: ticket.ticketNumber,
          priority: ticket.priority,
        },
        channel: 'in_app',
        priority: 'urgent',
      });
    }

    return this.prisma.supportTicket.update({
      where: { id },
      data: updateDto,
    });
  }
}
```

---

## 📋 Complete Summary

### What You Listen To (From Booking Service):
```
✅ booking.created
✅ booking.confirmed
✅ booking.cancelled
✅ payment.received
✅ payment.failed
✅ payment.refunded
```
**Already implemented!** Queue processor handles these.

---

### What YOU Generate (Scheduled Jobs):
```
⏰ marketing.weekly_newsletter (Every Monday 9am)
⏰ marketing.special_offers (When needed)
⏰ marketing.abandoned_cart (Daily)
⏰ marketing.reengagement (Daily)

⏰ partner.weekly_digest (Every Sunday 8am)
⏰ partner.monthly_report (1st of month)

⏰ booking.reminder_24h (Hourly check)
⏰ booking.reminder_2h (Hourly check)
```
**Need to implement:** Create scheduler modules with @Cron decorators.

---

### What YOU Trigger (Internal Events):
```
🎫 ticket.created → Notify agent + user
🎫 ticket.assigned → Notify agent
🎫 ticket.commented → Notify user/agent
🎫 ticket.escalated → Notify manager
🎫 ticket.resolved → Notify user
🎫 ticket.closed → Notify user
```
**Need to implement:** Add notification calls inside tickets service methods.

---

## 🚀 Next Steps

### Phase 1: Current (✅ Done)
- Queue processor for booking events
- Mock events for testing
- All CRUD modules

### Phase 2: Scheduled Jobs
1. Create `MarketingSchedulerService`
2. Create `PartnerSchedulerService`
3. Create `BookingReminderService`

### Phase 3: Ticket Notifications
1. Add notification triggers in `TicketsService`
2. Add notification triggers in `TicketCommentsService`

### Phase 4: Actual Sending
1. Email provider (AWS SES / SendGrid)
2. Push notifications (Firebase)
3. SMS (Twilio)

---

## 🔧 Technology Stack Summary

```json
{
  "Queue System": "Bull v4 + Redis v5",
  "NestJS Integration": "@nestjs/bull v11",
  "Scheduled Jobs": "@nestjs/schedule",
  "Database": "PostgreSQL + Prisma",
  "Email": "AWS SES / SendGrid (future)",
  "Push": "Firebase FCM (future)",
  "SMS": "Twilio (future)"
}
```

---

## 📞 Quick Reference

### Test Mock Event
```bash
curl -X POST http://localhost:3000/mock-events/booking-confirmed \
  -H "Content-Type: application/json" \
  -d '{
    "userId": "user_001",
    "bookingId": "TEST-001",
    "parkingName": "Test Parking"
  }'
```

### Check Queue Status
```bash
redis-cli
> LLEN bull:notifications:waiting
> LLEN bull:notifications:active
> LLEN bull:notifications:completed
> LLEN bull:notifications:failed
```

### View Logs
```bash
# Watch console for:
[NotificationQueueProcessor] Processing booking event: booking.confirmed
[NotificationQueueProcessor] Successfully processed event
```

---

**Save this file for reference!** 📖
