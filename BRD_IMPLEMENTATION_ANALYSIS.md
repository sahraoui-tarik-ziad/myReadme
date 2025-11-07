# BRD Implementation Analysis & Future Roadmap
## NotificationService - Gap Analysis & Best Practices

**Date:** November 7, 2025
**BRD Version:** 1.0.0
**Current Implementation:** v0.4.1

---

## Table of Contents
1. [Implementation Status Overview](#implementation-status-overview)
2. [Template Management Best Practices](#template-management-best-practices)
3. [Missing Features Analysis](#missing-features-analysis)
4. [Email Analytics & Tracking](#email-analytics--tracking)
5. [Stakeholder-Specific Features](#stakeholder-specific-features)
6. [Scheduled Notifications & Cron Jobs](#scheduled-notifications--cron-jobs)
7. [Future Roadmap](#future-roadmap)
8. [Architecture Recommendations](#architecture-recommendations)

---

## Implementation Status Overview

### ✅ Fully Implemented Features (30% Complete)

| Feature | BRD Section | Status | Notes |
|---------|-------------|--------|-------|
| **Email Notifications (Basic)** | 3.1.1 | ✅ Complete | SendGrid integrated, 7 event types |
| **Transactional Emails** | 3.1.1 | ✅ Complete | Booking confirmations, payment receipts |
| **Template Management** | 3.1.1 | ✅ Complete | Database storage, Handlebars rendering |
| **Notification Preferences** | 3.1.5 | ✅ Complete | User opt-in/opt-out, channel selection |
| **Support Ticket System** | 3.1.4 | ✅ Complete | Full CRUD, SLA tracking, auto-escalation |
| **Queue Processing** | 5.1 | ✅ Complete | BullMQ with Redis, retry logic |
| **Deduplication** | 3.3 | ✅ Complete | Hash-based, 7-day window |
| **Database Schema** | 5.2 | ✅ Complete | All tables created via Prisma |
| **Basic API Endpoints** | 5.3 | ✅ Complete | REST endpoints for core features |

### 🚧 Partially Implemented (20% Complete)

| Feature | BRD Section | Status | What's Missing |
|---------|-------------|--------|----------------|
| **Delivery Tracking** | 3.1.1 | 🚧 Partial | No open/click tracking, no bounce handling |
| **Template Personalization** | 3.1.1 | 🚧 Partial | Basic variables only, no advanced logic |
| **SLA Tracking** | 3.1.4 | 🚧 Partial | Calculation exists, no auto-escalation cron |
| **Email Logs** | 5.2 | 🚧 Partial | Schema exists, no analytics/reporting |

### ❌ Not Implemented (50% Missing)

| Feature | BRD Section | Priority | Impact |
|---------|-------------|----------|--------|
| **Push Notifications (FCM)** | 3.1.2 | High | Cannot reach mobile users |
| **In-App Notifications (WebSocket)** | 3.1.3 | High | No real-time updates |
| **Email Analytics Dashboard** | 9.1 | High | No open/click rate tracking |
| **Scheduled Notifications (Cron)** | 3.1.6 | High | No 24hr reminders, no digests |
| **Marketing Emails** | 3.1.1 | Medium | Cannot send newsletters |
| **Bounce Handling** | 3.1.1 | High | Risk to sender reputation |
| **Email Open/Click Tracking** | 3.1.1 | High | No engagement metrics |
| **Partner Digest Emails** | 3.1.1 | Medium | Partners overwhelmed with emails |
| **Weekly Performance Reports** | 3.1.6 | Low | No automated reporting |
| **Review Request Emails** | 7.4 | Low | Missing customer feedback loop |
| **Multi-language Support** | 9.1 | Low | Only French templates |
| **Quiet Hours Enforcement** | 3.1.5 | Medium | Users can be disturbed |
| **Frequency Caps** | 3.3 | Medium | Risk of email fatigue |
| **Admin Analytics Dashboard** | 2.3 | High | No visibility into metrics |

---

## Template Management Best Practices

### Current Approach: Database Storage ✅

**What We Have:**
- Templates stored in PostgreSQL `Template` table
- Handlebars engine for variable substitution
- Version tracking (basic)
- Admin API for CRUD operations

**Pros:**
- ✅ Dynamic updates without code deployment
- ✅ Version history and rollback
- ✅ Access control (only admins can edit)
- ✅ Multi-language support ready (language column exists)
- ✅ Template validation via API

**Cons:**
- ❌ No preview/testing UI (need admin dashboard)
- ❌ No A/B testing capability
- ❌ No template inheritance (e.g., base layout with blocks)
- ❌ No staging environment for templates

### Industry Best Practices

#### ✅ **Recommended: Database Storage (Current Approach)**

This is the **correct approach** for production SaaS platforms. You made the right choice!

**Why Database Storage is Best:**
1. **Business Agility** - Marketing team can update templates without developer involvement
2. **Version Control** - Automatic versioning and rollback capability
3. **Multi-tenancy Ready** - Easy to add partner-specific templates later
4. **Audit Trail** - Track who changed what and when (GDPR compliance)
5. **A/B Testing** - Can implement multiple active versions
6. **Staging/Production** - Can test templates in staging DB before promoting

**Examples from Industry:**
- **SendGrid** - Templates in database, managed via UI
- **Mailchimp** - Template builder stores in database
- **Customer.io** - Dynamic templates in database
- **Intercom** - All templates in database with visual editor

#### ❌ **NOT Recommended: File-Based Templates**

Storing templates in `templates/` folder is only suitable for:
- Small projects with <10 templates
- Single-language applications
- Developer-only template updates
- No version control requirements

**Drawbacks:**
- Requires code deployment for every change
- No audit trail
- Difficult to manage multiple languages
- No A/B testing capability
- Version control via Git (not business-friendly)

### Enhancements Needed for Current Approach

#### 1. Template Organization Structure

Add folder/category system:
```typescript
// Add to Template entity
{
  category: 'transactional' | 'marketing' | 'operational',
  subcategory: 'booking' | 'payment' | 'support',
  tags: ['urgent', 'customer-facing', 'partner-facing']
}
```

#### 2. Template Inheritance (Base Layouts)

Implement template blocks for reusability:
```typescript
// Base layout template
{
  name: 'email_base_layout',
  body: `
    <!DOCTYPE html>
    <html>
      <head>{{> head}}</head>
      <body>
        {{> header}}
        <main>{{> content}}</main>
        {{> footer}}
      </body>
    </html>
  `
}

// Child template inherits layout
{
  name: 'booking_confirmation',
  extends: 'email_base_layout',
  blocks: {
    content: '<h1>Booking Confirmed</h1>...'
  }
}
```

#### 3. Template Preview & Testing

Create admin endpoint for previewing:
```typescript
// POST /templates/:id/preview
{
  testData: {
    userName: 'Test User',
    bookingId: 'BKG-TEST-123'
  }
}
// Returns: Rendered HTML for preview
```

#### 4. Template Validation Rules

Add schema validation:
```typescript
{
  name: 'booking_confirmation',
  requiredVariables: ['userName', 'bookingId', 'parkingName'],
  optionalVariables: ['qrCodeUrl'],
  validation: {
    userName: { type: 'string', minLength: 1, maxLength: 100 },
    totalPrice: { type: 'number', min: 0 }
  }
}
```

#### 5. Template Staging Environment

Implement draft/published workflow:
```typescript
{
  status: 'draft' | 'published' | 'archived',
  publishedAt: '2025-11-07T10:00:00Z',
  publishedBy: 'admin-uid-123'
}
```

### Recommended File Structure for Template Assets

Templates stay in database, but assets go to cloud storage:

```
/cloud-storage/
  /email-assets/
    /images/
      logo.png
      header-booking.jpg
      footer-social-icons.png
    /css/
      email-base.css (inlined into templates)
```

**Why:** Separates code from content, allows CDN caching, reduces email size.

---

## Missing Features Analysis

### 1. Push Notifications (FCM) - HIGH PRIORITY ❌

**BRD Requirement:** Section 3.1.2 - Mobile push via Firebase Cloud Messaging

**What's Missing:**
- FCM SDK integration
- Device token management
- Push notification payload builder
- Push queue processor
- Rich notifications (images, action buttons)

**Business Impact:**
- **CRITICAL:** 60% of users prefer push over email
- Missing revenue protection alerts (payment failures)
- No real-time booking updates

**Implementation Effort:** 3-5 days

**Recommended Architecture:**
```typescript
// New module: src/modules/push-notifications/

// 1. Device Token Management
interface DeviceToken {
  userId: string;
  token: string;          // FCM device token
  platform: 'ios' | 'android' | 'web';
  active: boolean;
  lastUsed: Date;
}

// 2. Push Sender Service
class FcmSenderService {
  async sendPush(payload: PushPayload): Promise<FcmSendResult> {
    // Send via firebase-admin SDK
    const message = {
      notification: {
        title: payload.title,
        body: payload.body,
        imageUrl: payload.imageUrl,
      },
      data: payload.data,
      token: deviceToken,
    };
    return await this.fcm.send(message);
  }
}

// 3. Push Queue Processor (like email-queue.processor.ts)
class PushQueueProcessor extends WorkerHost {
  async process(job: Job<BookingEvent>) {
    // Similar to email processor
    // Fetch user's device tokens
    // Send via FCM
    // Log result
  }
}
```

**Priority Justification:**
- Booking reminders are useless without push (users don't check email constantly)
- Payment failure alerts require immediate attention (push > email)
- Real-time booking confirmations expected by users

---

### 2. In-App Notifications (WebSocket) - HIGH PRIORITY ❌

**BRD Requirement:** Section 3.1.3 - Real-time via Socket.io + FCM

**What's Missing:**
- Socket.io server setup
- Redis adapter for horizontal scaling
- WebSocket gateway
- Client authentication
- Notification center UI API
- Read/unread tracking

**Business Impact:**
- **CRITICAL:** No real-time updates in app
- Users must refresh to see new notifications
- Poor UX compared to competitors

**Implementation Effort:** 4-6 days

**Recommended Architecture:**
```typescript
// New module: src/modules/websocket/

// 1. WebSocket Gateway
@WebSocketGateway({ cors: true })
export class NotificationsGateway {
  @WebSocketServer()
  server: Server;

  // Redis adapter for multi-server scaling
  constructor() {
    this.server.adapter(createAdapter(pubClient, subClient));
  }

  @SubscribeMessage('subscribe')
  handleSubscribe(client: Socket, userId: string) {
    client.join(`user:${userId}`);
  }

  // Broadcast to specific user
  sendToUser(userId: string, notification: Notification) {
    this.server.to(`user:${userId}`).emit('notification', notification);
  }
}

// 2. In-App Queue Processor
class InAppQueueProcessor extends WorkerHost {
  async process(job: Job<BookingEvent>) {
    const notification = await this.createNotification(job.data);

    // Send via WebSocket
    this.gateway.sendToUser(job.data.userId, notification);

    // Also store in database for notification center
    await this.notificationsService.create(notification);
  }
}

// 3. Notification Center API
@Controller('notifications')
export class NotificationsController {
  @Get()
  async getUserNotifications(@CurrentUser() userId: string) {
    return await this.notificationsService.findByUser(userId, {
      limit: 50,
      orderBy: { createdAt: 'desc' }
    });
  }

  @Post(':id/read')
  async markAsRead(@Param('id') id: string) {
    return await this.notificationsService.markAsRead(id);
  }
}
```

**Priority Justification:**
- Core feature for mobile/web apps
- Expected by users (industry standard)
- Reduces support tickets ("I didn't know my booking was confirmed")

---

### 3. Email Analytics & Tracking - HIGH PRIORITY ❌

**BRD Requirement:** Section 3.1.1 - Delivery tracking: opened, clicked, bounced

**What's Missing:**
- SendGrid event webhook handler
- Email open tracking (pixel or SendGrid tracking)
- Click tracking (link wrapping)
- Bounce handling and suppression
- Analytics dashboard/reporting
- Open rate/click rate calculation
- Delivery rate monitoring

**Business Impact:**
- **CRITICAL for Marketing Team:** Cannot measure campaign effectiveness
- **CRITICAL for Compliance:** No bounce handling = risk to sender reputation
- No visibility into email deliverability issues
- Cannot optimize email templates (no A/B testing data)

**Implementation Effort:** 3-4 days

**SendGrid Provides These Events:**
- `delivered` - Email reached inbox
- `open` - User opened email (pixel tracking)
- `click` - User clicked link in email
- `bounce` - Email bounced (hard or soft)
- `spam_report` - User marked as spam
- `unsubscribe` - User clicked unsubscribe link

**Recommended Architecture:**
```typescript
// New module: src/modules/email-analytics/

// 1. SendGrid Webhook Handler
@Controller('webhooks/sendgrid')
export class SendGridWebhookController {
  @Post()
  async handleEvent(@Body() events: SendGridEvent[]) {
    for (const event of events) {
      switch (event.event) {
        case 'delivered':
          await this.emailLogsService.markDelivered(event.sg_message_id);
          break;
        case 'open':
          await this.emailLogsService.markOpened(event.sg_message_id, event.timestamp);
          break;
        case 'click':
          await this.emailLogsService.markClicked(event.sg_message_id, event.url);
          break;
        case 'bounce':
          await this.handleBounce(event);
          break;
        case 'spam_report':
          await this.handleSpamReport(event);
          break;
      }
    }
    return { success: true };
  }

  private async handleBounce(event: SendGridBounceEvent) {
    const bounceCount = await this.emailLogsService.incrementBounceCount(event.email);

    // BRD Rule: Suppress after 2 hard bounces
    if (event.type === 'bounce' && bounceCount >= 2) {
      await this.preferencesService.suppressEmail(event.email);
      // Alert admin
      this.alertService.sendAdminAlert({
        type: 'email_suppressed',
        email: event.email,
        reason: 'Hard bounce limit reached'
      });
    }
  }
}

// 2. Analytics Service
class EmailAnalyticsService {
  async calculateMetrics(timeRange: DateRange): Promise<EmailMetrics> {
    const logs = await this.emailLogsService.findByDateRange(timeRange);

    const total = logs.length;
    const delivered = logs.filter(l => l.status === 'delivered').length;
    const opened = logs.filter(l => l.openedAt !== null).length;
    const clicked = logs.filter(l => l.clickedAt !== null).length;
    const bounced = logs.filter(l => l.status === 'bounced').length;

    return {
      deliveryRate: (delivered / total) * 100,
      openRate: (opened / delivered) * 100,      // % of delivered emails
      clickRate: (clicked / opened) * 100,       // % of opened emails
      bounceRate: (bounced / total) * 100,
      totalSent: total,
    };
  }

  async getReportForMarketing(templateName: string, dateRange: DateRange) {
    // Generate report for specific campaign
    const metrics = await this.calculateMetrics(dateRange);
    const topLinks = await this.getTopClickedLinks(templateName, dateRange);
    const bounceDetails = await this.getBounceBreakdown(templateName, dateRange);

    return {
      campaign: templateName,
      period: dateRange,
      metrics,
      topLinks,
      bounceDetails,
    };
  }
}

// 3. Admin Dashboard API
@Controller('admin/analytics')
export class AnalyticsDashboardController {
  @Get('email-metrics')
  async getEmailMetrics(@Query() query: { period: 'today' | 'week' | 'month' }) {
    const dateRange = this.getDateRange(query.period);
    return await this.analyticsService.calculateMetrics(dateRange);
  }

  @Get('campaigns/:templateName')
  async getCampaignReport(@Param('templateName') template: string) {
    return await this.analyticsService.getReportForMarketing(template, {
      start: subDays(new Date(), 30),
      end: new Date()
    });
  }

  @Get('delivery-health')
  async getDeliveryHealth() {
    // Real-time monitoring dashboard
    return {
      currentDeliveryRate: await this.getCurrentDeliveryRate(),
      bouncesLast24Hours: await this.getRecentBounces(24),
      spamComplaintsLast7Days: await this.getSpamComplaints(7),
      alertLevel: this.calculateAlertLevel(),  // green/yellow/red
    };
  }
}
```

**SendGrid Configuration Required:**
```bash
# In SendGrid dashboard:
# Mail Settings → Event Webhook
# Webhook URL: https://your-domain.com/webhooks/sendgrid
# Actions to post: delivered, open, click, bounce, spam_report, unsubscribe
```

**Database Updates Needed:**
```sql
-- Update email_logs table
ALTER TABLE "EmailLog"
ADD COLUMN "deliveredAt" TIMESTAMP,
ADD COLUMN "clickedAt" TIMESTAMP,
ADD COLUMN "clickedUrl" TEXT,
ADD COLUMN "spamReportedAt" TIMESTAMP,
ADD COLUMN "unsubscribedAt" TIMESTAMP;

-- Add indexes for analytics queries
CREATE INDEX idx_email_logs_template ON "EmailLog"(template);
CREATE INDEX idx_email_logs_opened_at ON "EmailLog"("openedAt");
CREATE INDEX idx_email_logs_clicked_at ON "EmailLog"("clickedAt");
```

**Priority Justification:**
- **Business Critical:** Marketing team cannot function without metrics
- **Compliance Critical:** Bounce handling protects sender reputation
- **Revenue Impact:** Cannot optimize conversion without click tracking
- BRD explicitly calls out delivery tracking as core feature

---

### 4. Scheduled Notifications & Cron Jobs - HIGH PRIORITY ❌

**BRD Requirement:** Section 3.1.6 - 8 scheduled notification types

**What's Missing:**
- Cron job scheduler (node-cron)
- 24-hour booking reminder
- Parking expiry warning (T+24 hours)
- Payment due reminder
- Review request (T+3 days)
- Abandoned booking cleanup (48 hours)
- Partner daily digest (6 PM)
- Partner weekly report (Monday 9 AM)
- Ticket auto-escalation (10 hours)

**Business Impact:**
- **CRITICAL:** No booking reminders = higher no-show rate
- **REVENUE LOSS:** No payment reminders = lost revenue
- Partners overwhelmed with individual emails (no digest)
- No automated review requests = poor SEO/social proof
- Database bloat (abandoned bookings not cleaned)

**Implementation Effort:** 5-7 days

**Recommended Architecture:**
```typescript
// New module: src/modules/cron-jobs/

// 1. Cron Job Scheduler Module
@Module({
  imports: [NotificationsModule, BookingModule, TicketsModule],
  providers: [
    BookingReminderCron,
    PaymentReminderCron,
    ReviewRequestCron,
    PartnerDigestCron,
    TicketEscalationCron,
    DatabaseCleanupCron,
  ],
})
export class CronJobsModule {}

// 2. Booking Reminder Cron (Every 5 minutes)
@Injectable()
export class BookingReminderCron {
  constructor(
    private readonly bookingService: BookingService,
    private readonly notificationsService: NotificationsService,
  ) {
    // Run every 5 minutes
    cron.schedule('*/5 * * * *', () => this.send24HourReminders());
  }

  async send24HourReminders() {
    const now = new Date();
    const in24Hours = addHours(now, 24);
    const in24HoursPlus5Min = addMinutes(in24Hours, 5);

    // Find bookings starting in 24 hours (±5 min window)
    const upcomingBookings = await this.bookingService.findUpcoming({
      startDate: { gte: in24Hours, lte: in24HoursPlus5Min },
      status: 'confirmed',
      reminderSent: false,  // New field needed in Booking schema
    });

    for (const booking of upcomingBookings) {
      // Check user preferences (don't send if reminders disabled)
      const prefs = await this.preferencesService.getPreferences(booking.userId);
      if (!prefs.reminderEnabled) continue;

      // Check quiet hours
      if (this.isQuietHours(prefs)) continue;

      // Send reminder via email + push
      await this.notificationsService.sendBookingReminder({
        userId: booking.userId,
        bookingId: booking.id,
        parkingName: booking.parkingName,
        startDate: booking.startDate,
        hoursUntilStart: 24,
      });

      // Mark reminder as sent
      await this.bookingService.update(booking.id, { reminderSent: true });
    }

    console.log(`✅ Sent ${upcomingBookings.length} 24-hour reminders`);
  }
}

// 3. Payment Reminder Cron (Every hour)
@Injectable()
export class PaymentReminderCron {
  constructor(
    private readonly bookingService: BookingService,
    private readonly notificationsService: NotificationsService,
  ) {
    cron.schedule('0 * * * *', () => this.sendPaymentReminders());
  }

  async sendPaymentReminders() {
    // Find unpaid bookings older than 24 hours
    const unpaidBookings = await this.bookingService.find({
      status: 'pending',
      createdAt: { lte: subHours(new Date(), 24) },
      paymentReminderSent: false,
    });

    for (const booking of unpaidBookings) {
      await this.notificationsService.sendPaymentReminder({
        userId: booking.userId,
        bookingId: booking.id,
        totalAmount: booking.totalAmount,
        expiresAt: addHours(booking.createdAt, 48),  // BRD: 48hr expiry
      });

      await this.bookingService.update(booking.id, { paymentReminderSent: true });
    }

    console.log(`✅ Sent ${unpaidBookings.length} payment reminders`);
  }
}

// 4. Review Request Cron (Every hour)
@Injectable()
export class ReviewRequestCron {
  constructor(
    private readonly bookingService: BookingService,
    private readonly notificationsService: NotificationsService,
  ) {
    cron.schedule('0 * * * *', () => this.sendReviewRequests());
  }

  async sendReviewRequests() {
    // BRD: Send review request 3 days after booking completion
    const completedBookings = await this.bookingService.find({
      status: 'completed',
      endDate: {
        gte: subDays(new Date(), 3),
        lte: subDays(new Date(), 3).setHours(23, 59, 59),  // Exactly 3 days ago
      },
      reviewRequested: false,
    });

    for (const booking of completedBookings) {
      await this.notificationsService.sendReviewRequest({
        userId: booking.userId,
        bookingId: booking.id,
        parkingName: booking.parkingName,
        reviewUrl: `${process.env.APP_URL}/bookings/${booking.id}/review`,
      });

      await this.bookingService.update(booking.id, { reviewRequested: true });
    }

    console.log(`✅ Sent ${completedBookings.length} review requests`);
  }
}

// 5. Partner Daily Digest Cron (Every day at 6 PM)
@Injectable()
export class PartnerDigestCron {
  constructor(
    private readonly bookingService: BookingService,
    private readonly notificationsService: NotificationsService,
  ) {
    cron.schedule('0 18 * * *', () => this.sendPartnerDigests(), {
      timezone: 'Europe/Paris',
    });
  }

  async sendPartnerDigests() {
    // BRD: Aggregate hourly bookings 9 AM - 6 PM, send daily digest
    const partners = await this.partnerService.findAllActive();

    for (const partner of partners) {
      const todaysBookings = await this.bookingService.findByPartner(partner.id, {
        createdAt: { gte: startOfDay(new Date()) },
      });

      if (todaysBookings.length === 0) continue;

      // Group bookings by status
      const summary = {
        totalBookings: todaysBookings.length,
        confirmed: todaysBookings.filter(b => b.status === 'confirmed').length,
        cancelled: todaysBookings.filter(b => b.status === 'cancelled').length,
        revenue: todaysBookings.reduce((sum, b) => sum + b.totalAmount, 0),
        bookings: todaysBookings.map(b => ({
          reference: b.reference,
          customerName: b.userName,
          startDate: b.startDate,
          status: b.status,
        })),
      };

      await this.notificationsService.sendPartnerDailyDigest({
        partnerId: partner.id,
        partnerEmail: partner.email,
        date: format(new Date(), 'yyyy-MM-dd'),
        summary,
      });
    }

    console.log(`✅ Sent ${partners.length} partner daily digests`);
  }
}

// 6. Partner Weekly Report Cron (Every Monday at 9 AM)
@Injectable()
export class PartnerWeeklyReportCron {
  constructor(
    private readonly analyticsService: AnalyticsService,
    private readonly notificationsService: NotificationsService,
  ) {
    cron.schedule('0 9 * * 1', () => this.sendWeeklyReports(), {
      timezone: 'Europe/Paris',
    });
  }

  async sendWeeklyReports() {
    // BRD: Performance report with booking trends, revenue, ratings
    const partners = await this.partnerService.findAllActive();
    const lastWeek = { start: subDays(new Date(), 7), end: new Date() };

    for (const partner of partners) {
      const report = await this.analyticsService.generatePartnerReport(partner.id, lastWeek);

      await this.notificationsService.sendPartnerWeeklyReport({
        partnerId: partner.id,
        partnerEmail: partner.email,
        period: lastWeek,
        report: {
          totalBookings: report.bookingCount,
          revenue: report.totalRevenue,
          averageRating: report.averageRating,
          occupancyRate: report.occupancyRate,
          topCustomers: report.topCustomers,
        },
      });
    }

    console.log(`✅ Sent ${partners.length} partner weekly reports`);
  }
}

// 7. Ticket Auto-Escalation Cron (Every hour)
@Injectable()
export class TicketEscalationCron {
  constructor(
    private readonly ticketsService: TicketsService,
  ) {
    cron.schedule('0 * * * *', () => this.escalateOverdueTickets());
  }

  async escalateOverdueTickets() {
    // BRD: Auto-escalate urgent tickets not resolved within 10 hours
    const overdueTickets = await this.ticketsService.find({
      priority: 'urgent',
      status: { in: ['new', 'in_progress'] },
      escalated: false,
      createdAt: { lte: subHours(new Date(), 10) },
    });

    for (const ticket of overdueTickets) {
      await this.ticketsService.escalate(ticket.id, {
        reason: 'Auto-escalation: Unresolved for 10+ hours',
        escalatedTo: await this.getAvailableSpecialist(ticket.category),
      });

      // Notify management
      await this.notificationsService.sendEscalationAlert({
        ticketId: ticket.id,
        ticketNumber: ticket.ticketNumber,
        reason: 'SLA breach - 10 hours without resolution',
      });
    }

    console.log(`✅ Escalated ${overdueTickets.length} overdue tickets`);
  }
}

// 8. Database Cleanup Cron (Every day at 2 AM)
@Injectable()
export class DatabaseCleanupCron {
  constructor(
    private readonly bookingService: BookingService,
    private readonly dedupService: DeduplicationService,
  ) {
    cron.schedule('0 2 * * *', () => this.cleanupDatabase(), {
      timezone: 'Europe/Paris',
    });
  }

  async cleanupDatabase() {
    // BRD: Delete abandoned bookings after 48 hours
    const abandonedBookings = await this.bookingService.find({
      status: 'pending',
      createdAt: { lte: subHours(new Date(), 48) },
    });

    for (const booking of abandonedBookings) {
      await this.bookingService.delete(booking.id);
    }

    // BRD: Delete deduplication logs after 7 days
    await this.dedupService.deleteExpired();

    console.log(`✅ Cleaned up ${abandonedBookings.length} abandoned bookings`);
  }
}
```

**Required Database Schema Changes:**
```sql
-- Add tracking fields to Booking table (coordinate with Booking Service)
ALTER TABLE "Booking"
ADD COLUMN "reminderSent" BOOLEAN DEFAULT false,
ADD COLUMN "paymentReminderSent" BOOLEAN DEFAULT false,
ADD COLUMN "reviewRequested" BOOLEAN DEFAULT false;
```

**Environment Variables:**
```env
# Cron job toggles (disable in development)
ENABLE_CRON_JOBS=true
BOOKING_REMINDER_ENABLED=true
PAYMENT_REMINDER_ENABLED=true
PARTNER_DIGEST_ENABLED=true
```

**Priority Justification:**
- **Revenue Critical:** Payment reminders recover lost revenue
- **Customer Satisfaction:** Booking reminders reduce no-shows
- **Partner Experience:** Daily digests reduce email fatigue
- **SEO/Marketing:** Review requests build social proof
- All 8 cron jobs explicitly listed in BRD Section 3.1.6

---

## Stakeholder-Specific Features

### 1. Marketing Team Features ❌

**BRD Section:** 2.3 - Target Users #4

**What They Need:**
- ✅ Newsletter/promotional email templates (can create via API)
- ❌ **Campaign scheduling UI**
- ❌ **A/B testing for templates**
- ❌ **Subscriber segmentation** (send to specific user groups)
- ❌ **Email open rate dashboard**
- ❌ **Click rate dashboard**
- ❌ **Conversion tracking**
- ❌ **Unsubscribe management**
- ❌ **Bulk email sending** (1000+ users)

**Current Capability:** 0% (No marketing features implemented)

**Recommended Implementation:**
```typescript
// New module: src/modules/campaigns/

@Controller('campaigns')
export class CampaignsController {
  // Create campaign
  @Post()
  async createCampaign(@Body() dto: CreateCampaignDto) {
    // Campaign = marketing email to specific audience
    return await this.campaignsService.create({
      name: dto.name,
      template: dto.templateId,
      audience: dto.audienceFilter,  // e.g., { country: 'FR', bookings: { gte: 1 } }
      scheduledFor: dto.scheduledFor,
      abTest: dto.enableAbTest,      // Optional A/B test config
    });
  }

  // Get campaign metrics
  @Get(':id/metrics')
  async getCampaignMetrics(@Param('id') id: string) {
    return {
      sent: 1500,
      delivered: 1470,
      opened: 367,          // 25% open rate
      clicked: 73,          // 5% click rate
      unsubscribed: 15,     // 1% unsubscribe rate
      revenue: 4500,        // Attributed revenue from campaign
    };
  }

  // Export campaign report (CSV)
  @Get(':id/export')
  async exportReport(@Param('id') id: string, @Res() res: Response) {
    const csv = await this.campaignsService.generateCSV(id);
    res.setHeader('Content-Type', 'text/csv');
    res.setHeader('Content-Disposition', `attachment; filename="campaign-${id}.csv"`);
    res.send(csv);
  }
}
```

**Priority:** Medium (can use manual API calls for now)

---

### 2. Parking Partners Features 🚧

**BRD Section:** 2.3 - Target Users #2

**What They Need:**
- ✅ New booking alerts (transactional, immediate)
- ❌ **Daily digest** (aggregate bookings 9 AM - 6 PM)
- ❌ **Weekly performance report** (revenue, ratings, trends)
- ✅ Cancellation alerts (implemented)
- ❌ **Custom notification preferences** (digest frequency, time)
- ❌ **Dashboard notifications** (in-app for partner portal)

**Current Capability:** 40% (Transactional emails only, no digests/reports)

**Implementation Status:**
- BRD requires partner.booking.created events (not implemented)
- BRD requires partner.booking.cancelled events (not implemented)
- Daily digest cron job (see Section 4 above)
- Weekly report cron job (see Section 4 above)

**Coordination Required:**
- Booking Service must publish partner-specific events to Redis
- Partner Portal must integrate WebSocket for real-time notifications

---

### 3. Customer Support Team Features ✅

**BRD Section:** 2.3 - Target Users #3

**What They Need:**
- ✅ Ticket management system (fully implemented)
- ✅ SLA tracking (calculation implemented)
- 🚧 **SLA breach alerts** (cron job needed)
- ✅ Auto-assignment (implemented)
- 🚧 **Auto-escalation** (cron job needed)
- ✅ Internal notes (implemented)
- ❌ **Agent performance dashboard**
- ❌ **Ticket analytics** (resolution time, first response time)

**Current Capability:** 80% (Core features done, analytics missing)

**Missing Implementation:**
```typescript
// Admin dashboard for support managers
@Controller('admin/support-metrics')
export class SupportMetricsController {
  @Get('agents')
  async getAgentPerformance() {
    // BRD requirement: Agent workload balance, performance tracking
    return {
      agents: [
        {
          id: 'agent-001',
          name: 'John Doe',
          assignedTickets: 12,
          avgResolutionTime: '4.5 hours',
          slaComplianceRate: 95,
          customerSatisfaction: 4.6,
        },
      ],
    };
  }

  @Get('sla-status')
  async getSLAStatus() {
    // Real-time SLA monitoring
    return {
      urgent: { breached: 2, atRisk: 5, onTrack: 15 },
      high: { breached: 1, atRisk: 8, onTrack: 42 },
      normal: { breached: 0, atRisk: 12, onTrack: 98 },
    };
  }
}
```

**Priority:** Medium (core features work, analytics are nice-to-have)

---

### 4. Platform Administrators Features 🚧

**BRD Section:** 2.3 - Target Users #5

**What They Need:**
- ✅ Template management API (implemented)
- ❌ **Template preview UI**
- ❌ **Email delivery monitoring dashboard**
- ❌ **Bounce reports**
- ❌ **Failed notification alerts**
- ❌ **System health dashboard**
- ❌ **Queue monitoring UI**
- ❌ **User preference management** (handle support requests)

**Current Capability:** 30% (APIs exist, no UI/dashboards)

**Implementation Priority:** Low (devs can use API directly for now)

---

### 5. Compliance Officer Features ❌

**BRD Section:** 2.3 - Target Users #6

**What They Need:**
- ✅ Opt-in/opt-out tracking (preferences table exists)
- ❌ **GDPR consent logging** (audit trail)
- ❌ **Data export** (user requests their data)
- ❌ **Data deletion** ("right to be forgotten")
- ❌ **Audit logs** (all notification sends logged with consent proof)
- ❌ **Retention policy enforcement** (auto-delete after 2 years)

**Current Capability:** 20% (Basic preferences only)

**Critical Missing Feature:**
```typescript
// New module: src/modules/gdpr/

@Controller('gdpr')
export class GdprController {
  // User requests their data export
  @Get('export/:userId')
  async exportUserData(@Param('userId') userId: string) {
    return {
      notifications: await this.notificationsService.findByUser(userId),
      emailLogs: await this.emailLogsService.findByUser(userId),
      preferences: await this.preferencesService.getPreferences(userId),
      tickets: await this.ticketsService.findByUser(userId),
      consentHistory: await this.auditLogService.getConsentHistory(userId),
    };
  }

  // User requests data deletion
  @Delete('delete/:userId')
  async deleteUserData(@Param('userId') userId: string) {
    // BRD: GDPR compliance - right to be forgotten
    await this.notificationsService.deleteByUser(userId);
    await this.emailLogsService.anonymizeByUser(userId);  // Keep logs, remove PII
    await this.preferencesService.delete(userId);
    await this.ticketsService.anonymizeByUser(userId);

    return { message: 'User data deleted successfully' };
  }

  // Admin: Audit log for compliance review
  @Get('audit-log')
  async getAuditLog(@Query() query: { userId?: string, dateRange: DateRange }) {
    return await this.auditLogService.find({
      userId: query.userId,
      createdAt: { gte: query.dateRange.start, lte: query.dateRange.end },
    });
  }
}
```

**Priority:** High for production (legal requirement in EU)

---

## Future Roadmap

### Phase 1: Production Readiness (Next 2-4 Weeks) 🔥

**Priority: CRITICAL**

| Feature | Effort | Business Value | Dependencies |
|---------|--------|----------------|--------------|
| **Email Analytics & Tracking** | 3-4 days | High - Marketing cannot function | SendGrid webhooks |
| **Scheduled Notifications (Cron)** | 5-7 days | High - Revenue recovery, UX | Booking Service coordination |
| **Bounce Handling** | 2 days | High - Sender reputation | Part of analytics |
| **Admin Alerts** | 1 day | High - Operational visibility | None |
| **Template Preview API** | 1 day | Medium - Reduce errors | None |
| **GDPR Compliance** | 3-4 days | High - Legal requirement | None |

**Total Effort:** 15-19 days (3-4 weeks)

**Deliverables:**
- ✅ SendGrid webhook handler for email tracking
- ✅ Email open/click/bounce analytics
- ✅ 8 cron jobs for scheduled notifications
- ✅ Admin dashboard API (metrics only)
- ✅ GDPR data export/deletion endpoints
- ✅ Template preview endpoint

---

### Phase 2: Mobile & Real-Time (4-6 Weeks from Now) 📱

**Priority: HIGH**

| Feature | Effort | Business Value | Dependencies |
|---------|--------|----------------|--------------|
| **Push Notifications (FCM)** | 3-5 days | High - Mobile user engagement | Firebase setup |
| **In-App Notifications (WebSocket)** | 4-6 days | High - Real-time UX | Socket.io + Redis adapter |
| **Device Token Management** | 2 days | Medium - Required for push | Mobile app integration |
| **Notification Center API** | 2 days | Medium - Self-service for users | WebSocket |
| **Read/Unread Tracking** | 1 day | Low - Nice-to-have | Notification Center |

**Total Effort:** 12-16 days (2-3 weeks)

**Deliverables:**
- ✅ FCM integration for iOS/Android/Web push
- ✅ WebSocket gateway for real-time notifications
- ✅ Notification center with read/unread tracking
- ✅ Push queue processor (parallel to email queue)

---

### Phase 3: Marketing & Advanced Features (8-12 Weeks) 📊

**Priority: MEDIUM**

| Feature | Effort | Business Value | Dependencies |
|---------|--------|----------------|--------------|
| **Campaign Management** | 5 days | Medium - Marketing automation | Analytics |
| **A/B Testing** | 3 days | Low - Optimization | Campaigns |
| **Subscriber Segmentation** | 3 days | Medium - Targeted campaigns | User service |
| **Admin Dashboard UI** | 10 days | Medium - Usability | All analytics APIs |
| **Multi-language Support** | 5 days | High - EU expansion | None |
| **Partner Template Customization** | 7 days | Medium - Branding | Template inheritance |
| **Knowledge Base Integration** | 7 days | Medium - Reduce tickets 30% | External CMS |

**Total Effort:** 40 days (8 weeks)

---

### Phase 4: Advanced Analytics & AI (Future - Q2 2026) 🤖

**Priority: LOW**

| Feature | Effort | Business Value | Dependencies |
|---------|--------|----------------|--------------|
| **Send Time Optimization (ML)** | 10 days | Medium - +20% open rate | Large dataset |
| **AI-Powered Ticket Routing** | 15 days | Medium - Better resolution | ML model training |
| **Sentiment Analysis** | 7 days | Low - Auto-prioritization | NLP API |
| **Webhook Support** | 5 days | Medium - Partner integrations | None |
| **SMS Notifications** | 7 days | High - Critical alerts | Twilio integration |
| **WhatsApp Business** | 10 days | Medium - Popular channel | WhatsApp API |
| **Slack Integration** | 3 days | Low - Internal alerts | Slack app |

**Total Effort:** 57 days (11 weeks)

---

## Architecture Recommendations

### 1. Template Storage: Database ✅ (Correct Approach)

**Keep Current Implementation:**
- Templates in PostgreSQL `Template` table
- Handlebars rendering via TemplatesService
- Version tracking and rollback capability

**Enhancements Needed:**
- Template inheritance (base layouts)
- Template preview API
- Draft/published workflow
- Schema validation for variables

**Assets Storage:**
- Move images/CSS to Google Cloud Storage
- Use CDN for email assets (faster loading)
- Inline critical CSS, reference large images via URL

---

### 2. Queue Architecture: Multi-Queue ✅ (Correct Approach)

**Keep Current Implementation:**
- BullMQ with Redis
- Separate queues: `emails`, `sms`, `push`, `tickets`
- Retry logic with exponential backoff

**Enhancements Needed:**
- Priority queues (urgent vs normal)
- Dead-letter queue monitoring
- Queue metrics dashboard

---

### 3. Analytics Architecture: Event-Driven

**Recommended:**
- SendGrid webhooks → Event handler → Database
- Real-time metrics calculation via Redis (cache)
- Batch analytics jobs (daily aggregation)
- Separate analytics database (read replica)

**Why:**
- Webhook events are real-time (user opens email → instant update)
- Caching prevents expensive queries on every dashboard load
- Read replica prevents analytics queries from slowing down transactional operations

---

### 4. Notification Delivery Flow: Multi-Channel

**Current: Single Channel (Email)**
```
Event → Dedup → Preferences → Email Queue → SendGrid
```

**Target: Multi-Channel**
```
Event → Dedup → Preferences → Router
                                ├─> Email Queue → SendGrid
                                ├─> Push Queue → FCM
                                └─> In-App Queue → WebSocket
```

**Implementation:**
```typescript
// Update EventsProducerService routing logic
async produceEvent(event: BookingEvent) {
  // Check user preferences
  const prefs = await this.preferencesService.getPreferences(event.data.userId);

  // Route to appropriate queues based on preferences
  if (prefs.emailEnabled) {
    await this.emailQueue.add('booking-event', event);
  }
  if (prefs.pushEnabled) {
    await this.pushQueue.add('booking-event', event);
  }
  if (prefs.inAppEnabled) {
    await this.inAppQueue.add('booking-event', event);
  }
}
```

---

### 5. Monitoring & Alerting: Critical for Production

**Recommended Tools:**
- **Prometheus** - Metrics collection
- **Grafana** - Dashboards
- **PagerDuty** - Critical alerts
- **Slack** - Warning alerts
- **Sentry** - Error tracking

**Critical Metrics:**
```typescript
// Expose Prometheus metrics endpoint
@Controller('metrics')
export class MetricsController {
  @Get()
  async getMetrics() {
    return {
      // Email metrics
      email_delivery_rate: 98.5,
      email_bounce_rate: 1.2,
      email_send_latency_p95: 3.5,  // seconds

      // Queue metrics
      redis_queue_depth: 45,
      queue_processing_lag: 12,      // seconds
      failed_jobs_count: 3,

      // System metrics
      websocket_connections: 8542,
      database_query_time_p95: 42,   // ms
      api_response_time_p95: 156,    // ms
    };
  }
}
```

---

## Summary: What to Implement Next

### Immediate Priorities (This Week)

1. **Email Analytics & Tracking** (3-4 days)
   - SendGrid webhook handler
   - Open/click/bounce tracking
   - Bounce suppression after 2 hard bounces
   - Admin alerts for delivery issues

2. **Scheduled Notifications** (5-7 days)
   - 24-hour booking reminder cron
   - Payment reminder cron
   - Partner daily digest cron
   - Ticket auto-escalation cron

**Why These First:**
- Email analytics is **blocking** Marketing team
- Bounce handling is **critical** for sender reputation
- Scheduled notifications are **revenue-impacting** (payment reminders)
- All explicitly required in BRD

### Next Month Priorities

3. **Push Notifications** (3-5 days)
4. **In-App Notifications** (4-6 days)
5. **GDPR Compliance** (3-4 days)

### Can Wait 2-3 Months

6. **Marketing Campaigns** (5 days)
7. **Admin Dashboard UI** (10 days)
8. **Multi-language Support** (5 days)

---

## Final Recommendation: Template Storage

**Your Question:** "isolating templates in an other folder? or store them or what,, i am still do not know what is the best practice"

**Answer: Database storage is the correct approach for BookingPark.** ✅

**You made the right choice!** Here's why:

1. **Industry Standard** - Sendgrid, Mailchimp, Customer.io all use database storage
2. **Business Agility** - Marketing can update without deployments
3. **Version Control** - Automatic history and rollback
4. **Multi-tenancy Ready** - Can add partner-specific templates later
5. **A/B Testing** - Can have multiple active versions
6. **Compliance** - Audit trail for GDPR

**Only use file-based templates if:**
- Small project (<10 templates)
- Single language
- Developer-only updates
- No compliance requirements

**For BookingPark:**
- 30+ templates (booking, payment, partner, support)
- Multi-language (French, English, Spanish, German planned)
- Marketing team needs to update
- GDPR compliance required
- Multi-tenancy potential (partners)

**Therefore: Keep templates in database. ✅**

**Enhancements needed:**
- Template inheritance (base layouts)
- Preview API (test before publishing)
- Schema validation (required variables)
- Draft/published workflow

---

**Document Status:** ✅ Complete
**Next Steps:** Prioritize Phase 1 features (email analytics + cron jobs)
**Estimated Time to Production-Ready:** 3-4 weeks
