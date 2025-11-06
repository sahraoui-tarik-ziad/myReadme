# Database Schema Architecture Explanation

## Overview

The notification service uses **2 separate PostgreSQL schemas**: `notification_schema` and `ticket_schema`. This document explains why this separation exists and why it's a good design pattern.

## Schema Structure

### notification_schema (5 tables)
- `notifications` - Outbound messages to send (email/push/in-app)
- `email_logs` - Email delivery tracking (bounces, opens, clicks)
- `notification_deduplication` - Prevent duplicate notifications
- `notification_preferences` - User preferences (enable/disable channels, quiet hours)
- `notification_templates` - Message templates with variables

### ticket_schema (3 tables)
- `support_tickets` - Customer support issues/requests
- `ticket_comments` - Conversation thread on tickets
- `ticket_attachments` - Files attached to tickets

## Is There Duplication?

**No direct table duplication.** While both schemas deal with user communication, they serve fundamentally different purposes:

| Feature | `notifications` | `support_tickets` |
|---------|----------------|-------------------|
| **Direction** | System → User (one-way) | User ↔ Support (two-way) |
| **Purpose** | Inform/remind users | Solve user problems |
| **Lifecycle** | Send once & done | Conversation over days/weeks |
| **Conversation** | No replies | Has `ticket_comments` |
| **Attachments** | None | Has `ticket_attachments` |
| **Assignment** | No assignee | Has `assigned_to` field |
| **SLA tracking** | No | Has SLA fields |

### Minor Duplication

The only duplication is the **Priority enum**:
- `NotificationPriority`: urgent, high, normal, low
- `TicketPriority`: urgent, high, normal, low

However, these represent different concepts:
- **Notification priority** = Delivery urgency (how fast to send)
- **Ticket priority** = Business urgency (how fast to resolve)

This separation allows each domain to evolve independently.

## Why Keep Separate Schemas?

### 1. Different Business Domains

**Real-world analogy:**
```
Tickets = Your issue tracking system (like Zendesk/Jira)
Notifications = Your email delivery system (like SendGrid/AWS SES)

You wouldn't combine Zendesk and SendGrid into one system, right?
```

### 2. Different Lifecycles

```
Notification: Create → Send → Deliver → Done (minutes/hours)
Ticket: Create → Assign → Work → Resolve → Close (days/weeks)
```

### 3. Different Query Patterns

**Notifications:**
- "Get pending notifications to send"
- "Find failed deliveries"
- "Check if notification was already sent (deduplication)"

**Tickets:**
- "Get unassigned tickets"
- "Find SLA breaches"
- "Get ticket conversation history"

### 4. Different Scaling Needs

- **Notifications**: High volume (thousands/day), time-sensitive, mostly archived after delivery
- **Tickets**: Lower volume, long retention, complex queries with JOINs

### 5. Microservice Architecture

You're in `notification-service`, which suggests each schema might be owned by different services:
- Combining schemas would tightly couple two services
- Separate schemas allow independent deployment and scaling

### 6. Access Control

Schema-level permissions allow:
- Notification workers only access `notification_schema`
- Support agents only access `ticket_schema`

### 7. Future Flexibility

- Easier to split services later if needed
- Can migrate schemas to separate databases independently
- Database migrations affect domains independently

## How They Work Together

Tickets and notifications should interact via **events**, not direct foreign keys:

```
User creates ticket
  → Ticket stored in ticket_schema.support_tickets
  → Event: "ticket.created"
  → Notification created in notification_schema.notifications
  → Email sent to user: "We received your ticket #12345"

Support agent replies
  → Comment stored in ticket_schema.ticket_comments
  → Event: "ticket.replied"
  → Notification sent: "You have a new response on ticket #12345"
```

### Architecture Diagram

```
┌─────────────────────┐      ┌──────────────────────┐
│  ticket_schema      │      │ notification_schema  │
│                     │      │                      │
│  ┌───────────────┐  │      │  ┌────────────────┐ │
│  │ support_      │  │      │  │ notifications  │ │
│  │ tickets       │  │      │  │                │ │
│  └───────────────┘  │      │  └────────────────┘ │
│         │           │      │         ▲           │
│         │           │      │         │           │
│  ┌──────▼────────┐  │      │  ┌──────┴─────────┐ │
│  │ ticket_       │  │      │  │ email_logs     │ │
│  │ comments      │  │      │  │ preferences    │ │
│  │               │  │      │  │ templates      │ │
│  └───────────────┘  │      │  └────────────────┘ │
│                     │      │                      │
│  ┌───────────────┐  │      │                      │
│  │ ticket_       │  │      │                      │
│  │ attachments   │  │      │                      │
│  └───────────────┘  │      │                      │
└─────────────────────┘      └──────────────────────┘
         │                              ▲
         │         Events               │
         └──────────────────────────────┘
     "ticket.created", "ticket.replied"
```

## Code Organization

```
/src
  /modules
    /notifications  (handles notification_schema)
    /tickets        (handles ticket_schema)
  /shared
    /types          (shared TypeScript enums/types)
```

```typescript
// Good architecture:
ticketService.createTicket(data)
  → stores in ticket_schema
  → emits event

notificationService.onTicketCreated(event)
  → creates notification in notification_schema
  → sends via email/push
```

## Recommended Enhancement

To better track which notifications were sent for which tickets, consider adding optional source tracking:

```prisma
// In notification_schema
model Notification {
  // ... existing fields

  // Add optional reference to track notification source
  sourceType    String?  @db.VarChar(50)  // "ticket", "booking", "payment"
  sourceId      String?  @db.VarChar(255) // ticket_number, booking_id, etc.
}
```

This allows tracking without creating hard foreign keys across schemas.

## Why NOT to Merge Schemas

If you merged into one schema:
- ❌ Harder to split services later if needed
- ❌ Tight coupling between notification and ticket logic
- ❌ Permission management becomes harder (can't grant schema-level access)
- ❌ One large schema file harder to maintain
- ❌ Database migrations affect both domains simultaneously
- ❌ Violates Single Responsibility Principle

## Conclusion

**There is NO duplication.** These are **complementary systems**:

- **Tickets** = "What problems need solving?"
- **Notifications** = "How do we tell users about things?"

The current design follows **domain-driven design principles** where each schema represents a bounded context. This is a **good design pattern** that provides:

✅ Clear domain boundaries
✅ Independent scaling
✅ Better microservice architecture
✅ Easier team ownership
✅ Future flexibility

The minor duplication (Priority enum) is a worthwhile tradeoff for maintaining clean separation of concerns.
