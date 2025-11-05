# Support Ticket System Documentation

**Version:** 1.0.0
**Last Updated:** November 2025
**Module:** `src/modules/tickets/`

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Database Schema](#database-schema)
4. [Ticket Categories & Priorities](#ticket-categories--priorities)
5. [Ticket Lifecycle](#ticket-lifecycle)
6. [SLA Requirements](#sla-requirements)
7. [Implementation Guide](#implementation-guide)
8. [Business Rules](#business-rules)
9. [API Endpoints](#api-endpoints)
10. [Future Enhancements](#future-enhancements)

---

## Overview

### Purpose

The Support Ticket System is a **critical customer service management feature** that:
- **Auto-creates tickets** from critical events (payment failures, access issues)
- **Manages manual ticket submissions** from customers via API
- **Enforces SLA-based prioritization** ensuring urgent issues are resolved fast
- **Enables agent collaboration** via internal notes
- **Auto-escalates tickets** preventing neglected issues

### Key Features

| Feature | Description | Status |
|---------|-------------|--------|
| **Ticket Creation** | Auto-create from events + manual submission | ✅ Implemented |
| **Category Assignment** | 10 categories with subcategories | ✅ Schema ready |
| **Priority Levels** | Urgent, high, normal, low | ✅ Implemented |
| **SLA Tracking** | Monitor response/resolution times | 🚧 Partial |
| **Auto-Escalation** | Escalate unresolved tickets after 10 hours | ❌ Planned |
| **Internal Notes** | Private comments between agents | ✅ Schema ready |
| **Status Tracking** | New, in_progress, waiting, resolved, closed | ✅ Implemented |
| **Attachment Support** | Upload screenshots, documents | ✅ Schema ready |

---

## Architecture

### 3-Layer Architecture

```
┌─────────────────────────────────────────────────────────┐
│           BUSINESS LOGIC LAYER (Processors)             │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  TicketQueueProcessor                                   │
│  - Auto-create tickets from events                      │
│  - Set priority based on event type                     │
│  - Create alert notifications                           │
│  - Business rules enforcement                           │
│                                                         │
└────────────────────┬────────────────────────────────────┘
                     │ calls
                     ▼
┌─────────────────────────────────────────────────────────┐
│        REPOSITORY LAYER (Data Access Services)          │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  TicketsService                                         │
│  - create(dto)           → Create ticket                │
│  - findAll(filters)      → List tickets                 │
│  - findOne(id)           → Get ticket details           │
│  - update(id, dto)       → Update ticket                │
│  - getEscalatedTickets() → Find escalated tickets       │
│  - getOverdueSlaTickets()→ Find SLA breaches            │
│                                                         │
│  NotificationsService                                   │
│  - create(dto)           → Create alert notification    │
│                                                         │
└────────────────────┬────────────────────────────────────┘
                     │ uses
                     ▼
┌─────────────────────────────────────────────────────────┐
│                    PRISMA ORM                           │
│                                                         │
│  prisma.supportTicket.create()                          │
│  prisma.ticketComment.create()                          │
│  prisma.ticketAttachment.create()                       │
│                                                         │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│              GOOGLE CLOUD POSTGRESQL                    │
│                  (Cloud SQL)                            │
│                                                         │
│  ticket_schema.support_tickets                          │
│  ticket_schema.ticket_comments                          │
│  ticket_schema.ticket_attachments                       │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## Database Schema

### 1. `support_tickets` Table

**Location:** `ticket_schema.support_tickets`

```sql
CREATE TABLE ticket_schema.support_tickets (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  ticket_number       VARCHAR(20) UNIQUE NOT NULL,  -- TKT-2025-001234
  user_id             VARCHAR(255) NOT NULL,        -- Customer Firebase UID
  assigned_to         VARCHAR(255),                 -- Agent Firebase UID
  category            VARCHAR(50) NOT NULL,         -- 'booking_issues', 'payment_billing', etc.
  subcategory         VARCHAR(100) NOT NULL,
  priority            VARCHAR(10) NOT NULL,         -- 'urgent', 'high', 'normal', 'low'
  status              VARCHAR(20) NOT NULL,         -- 'new', 'in_progress', 'waiting', 'resolved', 'closed'
  subject             VARCHAR(200) NOT NULL,
  description         TEXT NOT NULL,
  resolution          TEXT,
  sla_response_due    TIMESTAMP,                    -- Calculated based on priority
  sla_resolution_due  TIMESTAMP,
  first_response_at   TIMESTAMP,
  resolved_at         TIMESTAMP,
  closed_at           TIMESTAMP,
  escalated           BOOLEAN DEFAULT false,
  escalated_at        TIMESTAMP,
  created_at          TIMESTAMP DEFAULT NOW(),
  updated_at          TIMESTAMP DEFAULT NOW()
);

-- Indexes for performance
CREATE INDEX idx_tickets_user_id ON support_tickets(user_id);
CREATE INDEX idx_tickets_assigned_to ON support_tickets(assigned_to);
CREATE INDEX idx_tickets_status ON support_tickets(status);
CREATE INDEX idx_tickets_priority ON support_tickets(priority);
CREATE INDEX idx_tickets_sla_response_due ON support_tickets(sla_response_due);
CREATE INDEX idx_tickets_created_at ON support_tickets(created_at);
```

### 2. `ticket_comments` Table

**Location:** `ticket_schema.ticket_comments`

```sql
CREATE TABLE ticket_schema.ticket_comments (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  ticket_id    UUID NOT NULL REFERENCES support_tickets(id) ON DELETE CASCADE,
  user_id      VARCHAR(255) NOT NULL,        -- Firebase UID (customer or agent)
  comment      TEXT NOT NULL,
  is_internal  BOOLEAN DEFAULT false,        -- Internal notes vs customer-visible
  created_at   TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_comments_ticket_id ON ticket_comments(ticket_id);
```

**Key Feature:** `is_internal` flag
- `false` → Customer can see (public comments)
- `true` → Only agents can see (private notes for collaboration)

### 3. `ticket_attachments` Table

**Location:** `ticket_schema.ticket_attachments`

```sql
CREATE TABLE ticket_schema.ticket_attachments (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  ticket_id   UUID NOT NULL REFERENCES support_tickets(id) ON DELETE CASCADE,
  file_name   VARCHAR(255) NOT NULL,
  file_type   VARCHAR(50) NOT NULL,
  file_size   BIGINT NOT NULL,
  file_url    TEXT NOT NULL,                -- Cloud Storage URL
  uploaded_by VARCHAR(255) NOT NULL,        -- Firebase UID
  created_at  TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_attachments_ticket_id ON ticket_attachments(ticket_id);
```

**Validation Rules:**
- Max 5 files per ticket
- Max file size: 10MB (10,485,760 bytes)
- Allowed types: `image/png`, `image/jpeg`, `application/pdf`

---

## Ticket Categories & Priorities

### Complete Taxonomy

#### 1. Booking Issues → **High Priority**
**Subcategories:**
- Cannot create booking
- Need to modify/extend booking
- Booking confirmation not received
- Wrong parking spot assigned

**SLA:** Response: 4hr, Resolution: 8hr

---

#### 2. Payment & Billing → **Urgent Priority**
**Subcategories:**
- Payment failed
- Refund request
- Incorrect charge
- Invoice/receipt needed

**SLA:** Response: 1hr, Resolution: 2hr

---

#### 3. Access & Entry Problems → **Urgent Priority**
**Subcategories:**
- Cannot enter parking facility
- Gate/barrier not working
- QR code not scanning
- Lost parking ticket

**SLA:** Response: 1hr, Resolution: 2hr

---

#### 4. Account & Profile → **High Priority**
**Subcategories:**
- Login/password issues
- Cannot update profile
- Account verification problems
- Delete account request

**SLA:** Response: 4hr, Resolution: 8hr

---

#### 5. Vehicle Registration → **Normal Priority**
**Subcategories:**
- Cannot add vehicle
- Vehicle information incorrect
- License plate issues

**SLA:** Response: 24hr, Resolution: 48hr

---

#### 6. Cancellation & Refunds → **High Priority**
**Subcategories:**
- Cancel booking
- Late cancellation fee dispute
- Refund status inquiry

**SLA:** Response: 4hr, Resolution: 8hr

---

#### 7. Technical Issues → **Normal Priority**
**Subcategories:**
- App crashing
- Website not loading
- Feature not working
- Bug report

**SLA:** Response: 24hr, Resolution: 48hr

---

#### 8. General Inquiry → **Low Priority**
**Subcategories:**
- How-to questions
- Pricing information
- Facility information
- Operating hours

**SLA:** Response: 48hr, Resolution: 96hr

---

#### 9. Complaint → **High Priority**
**Subcategories:**
- Facility cleanliness
- Staff behavior
- Service quality
- Safety concerns

**SLA:** Response: 4hr, Resolution: 8hr

---

#### 10. Feedback & Suggestions → **Low Priority**
**Subcategories:**
- Feature requests
- Improvement suggestions
- Positive feedback

**SLA:** Response: 48hr, Resolution: 96hr

---

## Ticket Lifecycle

### State Diagram

```
┌───────────┐     ┌─────────────┐     ┌────────────┐     ┌──────────┐     ┌────────┐
│    NEW    │────>│ IN_PROGRESS │────>│  WAITING   │────>│ RESOLVED │────>│ CLOSED │
└───────────┘     └─────────────┘     └────────────┘     └──────────┘     └────────┘
      │                  │                   │                  │
      │                  │                   │                  │
      └──────────────────┴───────────────────┴──────────────────┘
                                 │
                                 ▼
                          ┌────────────┐
                          │ ESCALATED  │
                          └────────────┘
```

### State Details

#### **1. NEW (Initial State)**

**Trigger:** Customer submits ticket OR auto-created from event

**Actions:**
1. Generate unique ticket number: `TKT-2025-XXXXXX`
2. Calculate priority based on category
3. Calculate SLA deadlines:
   ```typescript
   const now = new Date();

   switch (priority) {
     case 'urgent':
       slaResponseDue = now + 1 hour;
       slaResolutionDue = now + 2 hours;
       break;
     case 'high':
       slaResponseDue = now + 4 hours;
       slaResolutionDue = now + 8 hours;
       break;
     case 'normal':
       slaResponseDue = now + 24 hours;
       slaResolutionDue = now + 48 hours;
       break;
     case 'low':
       slaResponseDue = now + 48 hours;
       slaResolutionDue = now + 96 hours;
       break;
   }
   ```
4. Auto-assign to available agent (future feature)
5. Send notification to assigned agent
6. Send confirmation email to customer

**Duration:** < 1 minute

**Database Example:**
```json
{
  "id": "uuid-123",
  "ticketNumber": "TKT-2025-001234",
  "userId": "user_001",
  "assignedTo": null,
  "category": "payment",
  "subcategory": "payment_failed",
  "priority": "urgent",
  "status": "new",
  "subject": "Payment Failure - BKG-123",
  "description": "Automatic ticket created for payment failure...",
  "slaResponseDue": "2025-11-05T01:00:00Z",
  "slaResolutionDue": "2025-11-05T02:00:00Z",
  "firstResponseAt": null,
  "resolvedAt": null,
  "closedAt": null,
  "escalated": false,
  "createdAt": "2025-11-05T00:00:00Z"
}
```

---

#### **2. IN_PROGRESS (Agent Working)**

**Trigger:** Agent claims ticket or adds first response

**Actions:**
1. Record `first_response_at` timestamp
2. Check if within SLA response time:
   ```typescript
   if (firstResponseAt > slaResponseDue) {
     // ❌ SLA BREACHED - Flag for management review
     ticket.slaResponseBreached = true;
     await notifyManagement('SLA breach', ticket);
   }
   ```
3. Update ticket status to `in_progress`
4. Notify customer that agent is working on issue

**Duration:** Variable (based on priority)

**Database Updates:**
```json
{
  "status": "in_progress",
  "assignedTo": "agent_001",
  "firstResponseAt": "2025-11-05T00:30:00Z",
  "updatedAt": "2025-11-05T00:30:00Z"
}
```

---

#### **3. WAITING (Waiting for Customer Response)**

**Trigger:** Agent requests additional information

**Actions:**
1. Add internal note indicating what's needed
2. Send email to customer requesting info
3. **Pause SLA timer** (don't count against agent)
4. Set reminder to follow up in 48 hours

**Duration:** Up to 48 hours

**Business Logic:**
```typescript
// When ticket goes to WAITING status
ticket.status = 'waiting';
ticket.slaPausedAt = now();

// SLA doesn't count time while waiting for customer
// If customer responds in 24 hours, agent still has original SLA time remaining
```

---

#### **4. RESOLVED (Solution Provided)**

**Trigger:** Agent marks ticket as resolved

**Actions:**
1. Record `resolved_at` timestamp
2. Check if within SLA resolution time:
   ```typescript
   if (resolvedAt > slaResolutionDue) {
     // ❌ Resolution SLA BREACHED
     ticket.resolutionSlaBreached = true;
   }
   ```
3. Add resolution notes
4. Send resolution email to customer with solution
5. Request feedback/satisfaction rating
6. **Auto-close after 7 days** if no customer response

**Duration:** 7 days (then auto-close)

**Database Updates:**
```json
{
  "status": "resolved",
  "resolvedAt": "2025-11-05T01:45:00Z",
  "resolution": "Payment was declined by bank. User contacted and alternative payment method provided.",
  "updatedAt": "2025-11-05T01:45:00Z"
}
```

---

#### **5. CLOSED (Final State)**

**Trigger:** Customer confirms resolution OR 7 days pass

**Actions:**
1. Record `closed_at` timestamp
2. Archive ticket (keep forever for history)
3. Update agent performance metrics
4. **Ticket cannot be reopened** (customer must create new ticket)

**Duration:** Final state

**Database Updates:**
```json
{
  "status": "closed",
  "closedAt": "2025-11-12T01:45:00Z",
  "updatedAt": "2025-11-12T01:45:00Z"
}
```

---

#### **6. ESCALATED (Complex Issue)**

**Trigger 1:** Auto-escalation after 10 hours without resolution
**Trigger 2:** Agent manually escalates

**Auto-escalation Logic:**
```typescript
// Cron job runs every hour
cron.schedule('0 * * * *', async () => {
  // Find urgent tickets older than 10 hours
  const now = new Date();
  const tenHoursAgo = new Date(now.getTime() - 10 * 60 * 60 * 1000);

  const ticketsToEscalate = await prisma.supportTicket.findMany({
    where: {
      status: { in: ['new', 'in_progress'] },
      priority: 'urgent',
      createdAt: { lte: tenHoursAgo },
      escalated: false,
      resolvedAt: null,
    },
  });

  for (const ticket of ticketsToEscalate) {
    await escalateTicket(ticket.id);
  }
});
```

**Actions:**
1. Set `escalated` flag to true
2. Record `escalated_at` timestamp
3. Reassign to senior agent or specialist
4. Notify management
5. Extend SLA targets by 4 hours
6. Add escalation note with reason

**Database Updates:**
```json
{
  "escalated": true,
  "escalatedAt": "2025-11-05T10:00:00Z",
  "assignedTo": "senior_agent_001",
  "slaResolutionDue": "2025-11-05T06:00:00Z",
  "updatedAt": "2025-11-05T10:00:00Z"
}
```

---

## SLA Requirements

### Response & Resolution Times

| Priority | Response SLA | Resolution SLA | Auto-Escalate After |
|----------|--------------|----------------|---------------------|
| **Urgent** | 1 hour | 2 hours | 10 hours |
| **High** | 4 hours | 8 hours | 24 hours |
| **Normal** | 24 hours | 48 hours | 72 hours |
| **Low** | 48 hours | 96 hours | Not applicable |

### SLA Calculation Example

```typescript
// Ticket created at 2024-10-14 10:00 AM with URGENT priority
const ticket = {
  priority: 'urgent',
  createdAt: '2024-10-14T10:00:00Z',
  slaResponseDue: '2024-10-14T11:00:00Z',      // +1 hour
  slaResolutionDue: '2024-10-14T12:00:00Z',    // +2 hours
};

// Agent responds at 10:45 AM (within 1 hour SLA) ✅
ticket.firstResponseAt = '2024-10-14T10:45:00Z';
// Response time: 45 minutes < 1 hour SLA ✅

// Agent resolves at 11:30 AM (within 2 hour SLA) ✅
ticket.resolvedAt = '2024-10-14T11:30:00Z';
// Resolution time: 1.5 hours < 2 hours SLA ✅
```

### Business Rules

| Rule | Implementation | Rationale |
|------|---------------|-----------|
| **SLA Response Times** | Urgent: 1hr, High: 4hr, Normal: 24hr, Low: 48hr | Customer expectations, quality standards |
| **Auto-Escalation** | Escalate after 10 hours without resolution | Prevent neglected tickets |
| **Ticket Rate Limit** | Max 5 tickets per user per day | Anti-spam, abuse prevention |
| **Category Routing** | Auto-assign based on ticket category | Efficient resolution |
| **Resolution Targets** | 2x response time (Urgent: 2hr, High: 8hr) | Complete issue resolution |

---

## Implementation Guide

### Current Flow: Auto-Created Ticket (Payment Failure)

```
┌──────────────────┐
│  Payment Failed  │ Event from Payment Service
└────────┬─────────┘
         │
         ▼
┌─────────────────────────────────────────────────────┐
│ EventsProducerService routes to 'tickets' queue     │
│ File: events-producer.service.ts                    │
│ Route: PAYMENT_FAILED → 'tickets' queue             │
└────────┬────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────┐
│ TicketQueueProcessor picks up job                   │
│ File: ticket-queue.processor.ts                     │
│ Method: handleBookingEvent()                        │
└────────┬────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────┐
│ handlePaymentFailed() - Business Logic   │
├──────────────────────────────────────────┤
│                                          │
│ 1. Generate ticket number:               │
│    TKT-2025-299565                       │
│                                          │
│ 2. Call TicketsService.create():         │
│    - category: 'payment'                 │
│    - subcategory: 'payment_failed'       │
│    - priority: 'urgent' ⚡               │
│    - status: 'new'                       │
│                                          │
│ 3. Call NotificationsService.create():   │
│    - Alert customer via email            │
│    - Include ticket number in message    │
│                                          │
└──────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────┐
│ Database Tables Updated:                 │
│                                          │
│ ✅ ticket_schema.support_tickets         │
│ ✅ notification_schema.notifications     │
└──────────────────────────────────────────┘
```

### Code Example: Auto-Create Ticket

**File:** `src/modules/events/processors/ticket-queue.processor.ts`

```typescript
private async handlePaymentFailed(event: any) {
  // 1. Generate unique ticket number
  const ticketNumber = `TKT-${new Date().getFullYear()}-${Date.now().toString().slice(-6)}`;

  // 2. Create support ticket (Repository Layer)
  const ticket = await this.ticketsService.create({
    ticketNumber,
    userId: event.data.userId,
    category: 'payment',
    subcategory: 'payment_failed',
    priority: 'urgent',              // Payment failures are URGENT
    status: 'new',
    subject: `Payment Failure - ${event.data.bookingId}`,
    description: `Automatic ticket created for payment failure.

Booking ID: ${event.data.bookingId}
User ID: ${event.data.userId}
Amount: ${event.data.amount}
Failure Reason: ${event.data.failureReason}
Transaction ID: ${event.data.transactionId || 'N/A'}

Please investigate and contact the customer.`,
  });

  this.logger.log(`Created support ticket: ${ticketNumber} for payment failure`);

  // 3. Create notification to alert user
  await this.notificationsService.create({
    userId: event.data.userId,
    type: 'email',
    template: 'payment_failed',
    subject: 'Échec du paiement',
    content: {
      amount: event.data.amount,
      failureReason: event.data.failureReason,
      ticketNumber: ticketNumber,    // Include ticket number in email
    },
    channel: 'email',
    priority: 'urgent',
    metadata: {
      paymentId: event.data.paymentId,
      bookingId: event.data.bookingId,
      eventId: event.eventId,
      ticketId: ticket.id,
    },
  });
}
```

### Code Example: TicketsService (Repository Layer)

**File:** `src/modules/tickets/tickets.service.ts`

```typescript
@Injectable()
export class TicketsService {
  constructor(private readonly prisma: PrismaService) {}

  async create(createDto: CreateTicketDto) {
    // Check for duplicate ticket number
    const existing = await this.prisma.supportTicket.findUnique({
      where: { ticketNumber: createDto.ticketNumber },
    });

    if (existing) {
      throw new ConflictException(
        `Ticket with number ${createDto.ticketNumber} already exists`,
      );
    }

    // Create ticket in database
    return this.prisma.supportTicket.create({
      data: {
        ...createDto,
        slaResponseDue: createDto.slaResponseDue
          ? new Date(createDto.slaResponseDue)
          : undefined,
        slaResolutionDue: createDto.slaResolutionDue
          ? new Date(createDto.slaResolutionDue)
          : undefined,
      },
      include: {
        comments: true,
        attachments: true,
      },
    });
  }

  async getOverdueSlaTickets() {
    // Find tickets that have breached SLA
    return this.prisma.supportTicket.findMany({
      where: {
        OR: [
          {
            slaResponseDue: { lte: new Date() },
            firstResponseAt: null,
          },
          {
            slaResolutionDue: { lte: new Date() },
            resolvedAt: null,
          },
        ],
      },
      include: {
        comments: true,
        attachments: true,
      },
      orderBy: {
        priority: 'desc',
      },
    });
  }

  async getEscalatedTickets() {
    // Find all escalated tickets
    return this.prisma.supportTicket.findMany({
      where: { escalated: true },
      include: {
        comments: true,
        attachments: true,
      },
      orderBy: {
        escalatedAt: 'desc',
      },
    });
  }
}
```

---

## Business Rules

### Validation Rules

```typescript
const TICKET_VALIDATION_RULES = {
  // Rate limiting
  maxPerUserPerDay: 5,              // Anti-spam: max 5 tickets per day

  // Attachments
  maxAttachments: 5,
  maxAttachmentSize: 10485760,      // 10MB in bytes
  allowedFileTypes: [
    'image/png',
    'image/jpeg',
    'application/pdf',
  ],

  // Text fields
  subjectMaxLength: 200,
  descriptionMaxLength: 5000,

  // Escalation
  escalationHours: 10,              // Auto-escalate urgent after 10 hours

  // Auto-close
  autoCloseAfterResolved: 7,        // Close resolved tickets after 7 days
};
```

### Data Retention

| Data Type | Retention Period | Justification |
|-----------|-----------------|--------------|
| **Support Tickets** | Forever | Customer history, legal protection |
| **Ticket Comments** | Forever (linked to tickets) | Complete conversation history |
| **Ticket Attachments** | Forever (linked to tickets) | Evidence, dispute resolution |

---

## API Endpoints

### Planned Endpoints (Not Yet Implemented)

| Endpoint | Method | Purpose | Access |
|----------|--------|---------|--------|
| `/api/v1/tickets` | POST | Create support ticket | Customer, Agent |
| `/api/v1/tickets` | GET | List tickets (filtered) | Customer (own), Agent (all) |
| `/api/v1/tickets/:id` | GET | Get ticket details | Customer (own), Agent (assigned) |
| `/api/v1/tickets/:id` | PATCH | Update ticket status/assignment | Agent only |
| `/api/v1/tickets/:id/comments` | POST | Add comment | Customer, Agent |
| `/api/v1/tickets/:id/comments` | GET | List comments | Customer (public), Agent (all) |
| `/api/v1/tickets/:id/attachments` | POST | Upload attachment | Customer, Agent |
| `/api/v1/tickets/:id/escalate` | POST | Manually escalate ticket | Agent, Manager |
| `/api/v1/tickets/sla-breaches` | GET | List SLA breaches | Manager only |

### Example API Request

```bash
# Create ticket
POST /api/v1/tickets
Authorization: Bearer <firebase-token>
Content-Type: application/json

{
  "category": "booking_issues",
  "subcategory": "cannot_create_booking",
  "subject": "Cannot complete booking payment",
  "description": "I've tried to book parking for CDG airport but the payment keeps failing. I've tried 3 different cards.",
  "priority": "high"
}
```

**Response:**
```json
{
  "id": "uuid-123",
  "ticketNumber": "TKT-2025-001234",
  "userId": "user_001",
  "category": "booking_issues",
  "subcategory": "cannot_create_booking",
  "priority": "high",
  "status": "new",
  "subject": "Cannot complete booking payment",
  "slaResponseDue": "2025-11-05T14:00:00Z",
  "slaResolutionDue": "2025-11-05T18:00:00Z",
  "createdAt": "2025-11-05T10:00:00Z",
  "comments": [],
  "attachments": []
}
```

---

## Future Enhancements

### Phase 1 (Q1 2026) - SLA Automation

- [ ] Auto-calculate SLA deadlines based on priority
- [ ] Auto-escalation cron job (every hour)
- [ ] Auto-close resolved tickets after 7 days
- [ ] SLA breach alerts to management

**Implementation:**
```typescript
// Auto-escalation cron job
cron.schedule('0 * * * *', async () => {
  const ticketsToEscalate = await ticketsService.findTicketsForEscalation();
  for (const ticket of ticketsToEscalate) {
    await ticketsService.escalate(ticket.id);
  }
});

// Auto-close cron job
cron.schedule('0 0 * * *', async () => {
  const ticketsToClose = await ticketsService.findResolvedTicketsOlderThan(7);
  for (const ticket of ticketsToClose) {
    await ticketsService.update(ticket.id, { status: 'closed' });
  }
});
```

### Phase 2 (Q2 2026) - Agent Assignment

- [ ] Auto-assign tickets to agents based on category
- [ ] Round-robin assignment for load balancing
- [ ] Agent workload tracking
- [ ] Agent performance metrics

### Phase 3 (Q3 2026) - Advanced Features

- [ ] AI-powered ticket routing
- [ ] Sentiment analysis for priority adjustment
- [ ] Chatbot for common issues
- [ ] Knowledge base integration

---

## Related Documentation

- [BRD - Notification Service](../../BRD_NOTIFICATION_SERVICE.md) - Complete business requirements
- [Events Module README](../events/README.md) - Event-driven architecture
- [Notifications Module README](../notifications/README.md) - Notification system

---

## Appendix: Example Ticket

### Auto-Created Payment Failure Ticket

```json
{
  "id": "709fa84b-d2a1-47cb-8b54-15f862c56a16",
  "ticketNumber": "TKT-2025-299565",
  "userId": "user_001",
  "assignedTo": null,
  "category": "payment",
  "subcategory": "payment_failed",
  "priority": "urgent",
  "status": "new",
  "subject": "Payment Failure - BKG-1762299565303",
  "description": "Automatic ticket created for payment failure.\n\nBooking ID: BKG-1762299565303\nUser ID: user_001\nAmount: 45.50\nFailure Reason: Insufficient funds\nTransaction ID: N/A\n\nPlease investigate and contact the customer.",
  "resolution": null,
  "slaResponseDue": "2025-11-05T01:00:00Z",
  "slaResolutionDue": "2025-11-05T02:00:00Z",
  "firstResponseAt": null,
  "resolvedAt": null,
  "closedAt": null,
  "escalated": false,
  "escalatedAt": null,
  "createdAt": "2025-11-05T00:00:00Z",
  "updatedAt": "2025-11-05T00:00:00Z",
  "comments": [],
  "attachments": []
}
```

### Related Notification

```json
{
  "id": "a825d0a2-be4a-47af-9eaa-3cff728b7d63",
  "userId": "user_001",
  "type": "email",
  "template": "payment_failed",
  "subject": "Échec du paiement",
  "content": {
    "amount": 45.50,
    "failureReason": "Insufficient funds",
    "ticketNumber": "TKT-2025-299565"
  },
  "status": "pending",
  "channel": "email",
  "priority": "urgent",
  "metadata": {
    "paymentId": "pay_123",
    "bookingId": "BKG-1762299565303",
    "eventId": "aede3a05-1340-4cd3-916b-3e3adcaa2ea2",
    "ticketId": "709fa84b-d2a1-47cb-8b54-15f862c56a16"
  },
  "createdAt": "2025-11-05T00:00:00Z"
}
```

---

**For Questions or Updates:** Contact the development team or refer to the main BRD document.
