# Notification Preferences Module Documentation

**Version:** 1.0.0
**Last Updated:** November 2025
**Module:** `src/modules/preferences/`

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Database Schema](#database-schema)
4. [Preference Flags](#preference-flags)
5. [API Endpoints](#api-endpoints)
6. [Business Logic](#business-logic)
7. [Integration with Other Modules](#integration-with-other-modules)
8. [Code Examples](#code-examples)
9. [GDPR Compliance](#gdpr-compliance)

---

## Overview

### Purpose

The **Notification Preferences Module** manages user notification settings, enabling:
- **User control** over which notifications they receive
- **GDPR compliance** through explicit opt-in/opt-out management
- **Channel selection** (email, push, in-app)
- **Quiet hours** configuration for do-not-disturb periods
- **Privacy-first defaults** (marketing OFF by default)

### Key Features

| Feature | Description | Status |
|---------|-------------|--------|
| **Channel Toggles** | Enable/disable email, push, in-app notifications | ✅ Implemented |
| **Category Control** | Opt-out of marketing, reminders, partner digests | ✅ Implemented |
| **Quiet Hours** | Set do-not-disturb periods | ✅ Implemented |
| **Timezone Support** | User timezone for scheduled notifications | ✅ Implemented |
| **Auto-Creation** | Auto-create defaults on first notification | ✅ Implemented |
| **GDPR Compliant** | Explicit consent tracking, data export/deletion | ✅ Implemented |

---

## Architecture

### 3-Layer Architecture

```
┌──────────────────────────────────────────────────────────┐
│              PRESENTATION LAYER (API)                    │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  PreferencesController                                   │
│  - POST   /api/v1/preferences      → Create preferences  │
│  - GET    /api/v1/preferences/:id  → Get preferences     │
│  - PUT    /api/v1/preferences/:id  → Update preferences  │
│  - DELETE /api/v1/preferences/:id  → Delete preferences  │
│                                                          │
│  Responsibilities:                                       │
│  - HTTP request/response handling                        │
│  - Input validation (via DTOs)                           │
│  - Route protection (Firebase auth)                      │
│                                                          │
└────────────────────┬─────────────────────────────────────┘
                     │ calls
                     ▼
┌──────────────────────────────────────────────────────────┐
│         BUSINESS LOGIC LAYER (Service)                   │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  PreferencesService                                      │
│  - create(dto)                                           │
│  - getPreferences(userId)    ← Most important!           │
│  - updatePreferences(userId, dto)                        │
│  - deletePreferences(userId)                             │
│                                                          │
│  Business Rules:                                         │
│  ✅ Auto-create defaults if user has no preferences      │
│  ✅ Prevent duplicate preferences per user               │
│  ✅ Marketing emails require explicit opt-in             │
│  ✅ Transactional notifications always enabled           │
│  ✅ Quiet hours max 12 hours (anti-abuse)                │
│                                                          │
└────────────────────┬─────────────────────────────────────┘
                     │ uses
                     ▼
┌──────────────────────────────────────────────────────────┐
│            REPOSITORY LAYER (Prisma ORM)                 │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  PrismaService                                           │
│  - prisma.notificationPreference.create()                │
│  - prisma.notificationPreference.findUnique()            │
│  - prisma.notificationPreference.update()                │
│  - prisma.notificationPreference.delete()                │
│                                                          │
└────────────────────┬─────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────┐
│           GOOGLE CLOUD POSTGRESQL                        │
│                  (Cloud SQL)                             │
│                                                          │
│  notification_schema.notification_preferences            │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### Layer Responsibilities

#### **Presentation Layer** (`preferences.controller.ts`)
- HTTP endpoint definitions
- Request validation via DTOs
- Response serialization
- Authentication/authorization checks (future)

#### **Business Logic Layer** (`preferences.service.ts`)
- **Core business rule**: Auto-create defaults if preferences don't exist
- Duplicate prevention
- GDPR compliance logic
- Preference validation

#### **Repository Layer** (`PrismaService`)
- Database CRUD operations
- Query optimization
- Transaction management

---

## Database Schema

### `notification_preferences` Table

**Location:** `notification_schema.notification_preferences`

```sql
CREATE TABLE notification_schema.notification_preferences (
  id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id                 VARCHAR(255) UNIQUE NOT NULL,  -- Firebase UID

  -- Channel toggles (what channels user wants to receive notifications on)
  email_enabled           BOOLEAN DEFAULT true NOT NULL,
  push_enabled            BOOLEAN DEFAULT true NOT NULL,
  in_app_enabled          BOOLEAN DEFAULT true NOT NULL,

  -- Category toggles (types of notifications)
  marketing_enabled       BOOLEAN DEFAULT false NOT NULL,  -- GDPR: Opt-in required
  reminder_enabled        BOOLEAN DEFAULT true NOT NULL,
  partner_digest_enabled  BOOLEAN DEFAULT true NOT NULL,

  -- Quiet hours (do-not-disturb period)
  quiet_hours_start       TIME,                          -- e.g., '22:00:00'
  quiet_hours_end         TIME,                          -- e.g., '08:00:00'
  timezone                VARCHAR(50) DEFAULT 'Europe/Paris' NOT NULL,

  created_at              TIMESTAMP DEFAULT NOW() NOT NULL,
  updated_at              TIMESTAMP DEFAULT NOW() NOT NULL
);

-- Index for fast user lookup
CREATE UNIQUE INDEX idx_preferences_user_id ON notification_preferences(user_id);
```

### Field Descriptions

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `id` | UUID | Yes | Auto | Unique identifier |
| `user_id` | String | Yes | - | Firebase UID (unique per user) |
| **Channel Toggles** |
| `email_enabled` | Boolean | Yes | `true` | Receive email notifications |
| `push_enabled` | Boolean | Yes | `true` | Receive push notifications |
| `in_app_enabled` | Boolean | Yes | `true` | Receive in-app notifications |
| **Category Toggles** |
| `marketing_enabled` | Boolean | Yes | `false` | Marketing emails (GDPR opt-in) |
| `reminder_enabled` | Boolean | Yes | `true` | Booking reminders |
| `partner_digest_enabled` | Boolean | Yes | `true` | Daily digest for partners |
| **Quiet Hours** |
| `quiet_hours_start` | Time | No | `null` | Do-not-disturb start (HH:MM:SS) |
| `quiet_hours_end` | Time | No | `null` | Do-not-disturb end (HH:MM:SS) |
| `timezone` | String | Yes | `'Europe/Paris'` | User timezone (IANA format) |

---

## Preference Flags

### Channel Toggles

#### `emailEnabled` ✉️
**Purpose:** Control email notifications

**Use Cases:**
- User doesn't want any emails
- Temporary disable during vacation
- Inbox management

**Business Rule:**
```typescript
// Transactional emails (booking confirmations, payment receipts)
// ALWAYS sent regardless of this flag (legal requirement)
if (notification.type === 'transactional') {
  send(); // Bypass preference check
} else if (preferences.emailEnabled) {
  send(); // Check preference
}
```

**Impact:**
- ✅ `true`: User receives marketing, reminders, digests
- ❌ `false`: User only receives transactional emails

---

#### `pushEnabled` 📱
**Purpose:** Control push notifications

**Use Cases:**
- User finds push notifications intrusive
- Device doesn't support push
- Battery saving mode

**Impact:**
- ✅ `true`: Mobile/browser push notifications sent
- ❌ `false`: No push notifications

---

#### `inAppEnabled` 🔔
**Purpose:** Control in-app notifications

**Use Cases:**
- User prefers email over in-app alerts
- Reduce notification clutter

**Impact:**
- ✅ `true`: Notification center shows updates
- ❌ `false`: No in-app notifications

---

### Category Toggles

#### `marketingEnabled` 📣
**Purpose:** Marketing emails and promotional campaigns

**GDPR Requirement:** **Explicit opt-in required** (default `false`)

**Use Cases:**
- Newsletters
- Promotional offers
- Feature announcements

**Business Rule:**
```typescript
// Marketing emails MUST check this flag
if (!preferences.marketingEnabled) {
  return; // ⛔ Stop - user didn't opt-in
}

// Also enforce daily limit
if (marketingEmailsSentToday >= 1) {
  return; // ⛔ Max 1 marketing email per day
}

send(); // ✅ User opted in, within limits
```

---

#### `reminderEnabled` ⏰
**Purpose:** Booking reminders and alerts

**Use Cases:**
- 24hr arrival reminder
- Parking expiry warning
- Payment due reminder

**Business Rule:**
```typescript
// Check before sending reminders
if (preferences.reminderEnabled) {
  await send24HourReminder(booking);
}
```

---

#### `partnerDigestEnabled` 📊
**Purpose:** Daily/weekly summaries for parking partners

**Use Cases:**
- Aggregate booking notifications
- Reduce email fatigue for partners
- Daily performance reports

**Business Rule:**
```typescript
// Only partners receive digests
if (user.role === 'partner' && preferences.partnerDigestEnabled) {
  await sendDailyDigest(partnerId);
}
```

---

### Quiet Hours

#### `quietHoursStart` & `quietHoursEnd` 🌙

**Purpose:** Do-not-disturb period

**Format:** `HH:MM:SS` (24-hour format)

**Example:**
```json
{
  "quietHoursStart": "22:00:00",  // 10 PM
  "quietHoursEnd": "08:00:00",    // 8 AM
  "timezone": "Europe/Paris"
}
```

**Business Rule:**
```typescript
// Before sending non-urgent notification
const now = new Date();
const userTime = now.toLocaleString('en-US', {
  timeZone: preferences.timezone
});

const currentHour = parseInt(userTime.split(':')[0]);
const quietStart = parseInt(preferences.quietHoursStart.split(':')[0]);
const quietEnd = parseInt(preferences.quietHoursEnd.split(':')[0]);

if (isWithinQuietHours(currentHour, quietStart, quietEnd)) {
  // ⛔ Delay notification until quiet hours end
  await scheduleForLater(notification, preferences.quietHoursEnd);
  return;
}

send(); // ✅ Outside quiet hours
```

**Validation:**
- Max 12 hours quiet period (anti-abuse)
- Transactional/urgent notifications bypass quiet hours

---

## API Endpoints

### 1. Create Preferences

```http
POST /api/v1/preferences
Authorization: Bearer <firebase-token>
Content-Type: application/json
```

**Request Body:**
```json
{
  "userId": "user_001",
  "emailEnabled": true,
  "pushEnabled": true,
  "inAppEnabled": true,
  "marketingEnabled": false,
  "reminderEnabled": true,
  "partnerDigestEnabled": false,
  "quietHoursStart": "22:00:00",
  "quietHoursEnd": "08:00:00",
  "timezone": "Europe/Paris"
}
```

**Response:** `201 Created`
```json
{
  "id": "uuid-123",
  "userId": "user_001",
  "emailEnabled": true,
  "pushEnabled": true,
  "inAppEnabled": true,
  "marketingEnabled": false,
  "reminderEnabled": true,
  "partnerDigestEnabled": false,
  "quietHoursStart": "22:00:00",
  "quietHoursEnd": "08:00:00",
  "timezone": "Europe/Paris",
  "createdAt": "2025-11-05T00:00:00Z",
  "updatedAt": "2025-11-05T00:00:00Z"
}
```

**Error:** `409 Conflict` (if preferences already exist)
```json
{
  "statusCode": 409,
  "message": "Preferences for user user_001 already exist"
}
```

---

### 2. Get Preferences

```http
GET /api/v1/preferences/:userId
Authorization: Bearer <firebase-token>
```

**Response:** `200 OK`
```json
{
  "id": "uuid-123",
  "userId": "user_001",
  "emailEnabled": true,
  "pushEnabled": true,
  "inAppEnabled": true,
  "marketingEnabled": false,
  "reminderEnabled": true,
  "partnerDigestEnabled": false,
  "quietHoursStart": "22:00:00",
  "quietHoursEnd": "08:00:00",
  "timezone": "Europe/Paris",
  "createdAt": "2025-11-05T00:00:00Z",
  "updatedAt": "2025-11-05T00:00:00Z"
}
```

**Auto-Creation Feature:**
```typescript
// If user has NO preferences, auto-create defaults
if (!preferences) {
  preferences = await prisma.notificationPreference.create({
    data: {
      userId,
      emailEnabled: true,
      pushEnabled: true,
      inAppEnabled: true,
      marketingEnabled: false,  // ❌ Privacy-first default
      reminderEnabled: true,
      partnerDigestEnabled: true,
      timezone: 'Europe/Paris',
    },
  });
}
```

---

### 3. Update Preferences

```http
PUT /api/v1/preferences/:userId
Authorization: Bearer <firebase-token>
Content-Type: application/json
```

**Request Body:** (all fields optional)
```json
{
  "emailEnabled": false,
  "marketingEnabled": true,
  "quietHoursStart": "23:00:00",
  "quietHoursEnd": "07:00:00"
}
```

**Response:** `200 OK`
```json
{
  "id": "uuid-123",
  "userId": "user_001",
  "emailEnabled": false,  // ← Updated
  "pushEnabled": true,
  "inAppEnabled": true,
  "marketingEnabled": true,  // ← Updated
  "reminderEnabled": true,
  "partnerDigestEnabled": false,
  "quietHoursStart": "23:00:00",  // ← Updated
  "quietHoursEnd": "07:00:00",    // ← Updated
  "timezone": "Europe/Paris",
  "createdAt": "2025-11-05T00:00:00Z",
  "updatedAt": "2025-11-05T10:30:00Z"  // ← Updated
}
```

**Smart Behavior:**
```typescript
// If preferences don't exist, create them with provided values
if (!existingPreferences) {
  return prisma.notificationPreference.create({
    data: { userId, ...updateDto },
  });
}

// Otherwise, update existing
return prisma.notificationPreference.update({
  where: { userId },
  data: updateDto,
});
```

---

### 4. Delete Preferences

```http
DELETE /api/v1/preferences/:userId
Authorization: Bearer <firebase-token>
```

**Response:** `204 No Content`

**Error:** `404 Not Found` (if preferences don't exist)
```json
{
  "statusCode": 404,
  "message": "Preferences for user user_001 not found"
}
```

**GDPR Note:** Used for "Right to be Forgotten" requests

---

## Business Logic

### Smart Auto-Creation

**Location:** `preferences.service.ts:27-50`

**The Most Important Feature!**

```typescript
async getPreferences(userId: string) {
  // STEP 1: Try to find existing preferences
  let preferences = await this.prisma.notificationPreference.findUnique({
    where: { userId },
  });

  // STEP 2: If user has NO preferences (first-time notification)
  if (!preferences) {
    // Auto-create sensible defaults
    preferences = await this.prisma.notificationPreference.create({
      data: {
        userId,
        emailEnabled: true,        // ✅ YES to emails
        pushEnabled: true,         // ✅ YES to push
        inAppEnabled: true,        // ✅ YES to in-app
        marketingEnabled: false,   // ❌ NO to marketing (GDPR)
        reminderEnabled: true,     // ✅ YES to reminders
        partnerDigestEnabled: true,// ✅ YES to digest (partners only)
        timezone: 'Europe/Paris',  // Default timezone
      },
    });
  }

  // STEP 3: Return preferences (found or created)
  return preferences;
}
```

**Why This Is Smart:**
- ✅ First-time users automatically get preferences
- ✅ No manual setup required
- ✅ Privacy-first defaults (marketing OFF)
- ✅ Seamless user experience
- ✅ GDPR compliant (no marketing without consent)

---

### Validation Rules

**Location:** DTOs with `class-validator` decorators

#### `CreatePreferenceDto`
```typescript
export class CreatePreferenceDto {
  @IsNotEmpty()
  @IsString()
  userId: string;  // ← Required

  @IsOptional()
  @IsBoolean()
  emailEnabled?: boolean;

  @IsOptional()
  @IsBoolean()
  pushEnabled?: boolean;

  // ... other boolean flags

  @IsOptional()
  @IsString()
  @Matches(/^([0-1][0-9]|2[0-3]):[0-5][0-9]:[0-5][0-9]$/, {
    message: 'quietHoursStart must be in HH:MM:SS format (e.g., 22:00:00)',
  })
  quietHoursStart?: string;  // ← Regex validation

  @IsOptional()
  @IsString()
  @Matches(/^([0-1][0-9]|2[0-3]):[0-5][0-9]:[0-5][0-9]$/, {
    message: 'quietHoursEnd must be in HH:MM:SS format (e.g., 08:00:00)',
  })
  quietHoursEnd?: string;

  @IsOptional()
  @IsString()
  timezone?: string;  // IANA timezone (e.g., 'America/New_York')
}
```

**Validation Errors Example:**
```json
{
  "statusCode": 400,
  "message": [
    "quietHoursStart must be in HH:MM:SS format (e.g., 22:00:00)"
  ],
  "error": "Bad Request"
}
```

---

## Integration with Other Modules

### Used By: EmailQueueProcessor

**File:** `src/modules/events/processors/email-queue.processor.ts`

```typescript
@Processor('emails', { concurrency: 5 })
export class EmailQueueProcessor extends WorkerHost {
  constructor(
    private readonly notificationsService: NotificationsService,
    private readonly preferencesService: PreferencesService,  // ← Injected
  ) {
    super();
  }

  private async handleBookingEvent(job: Job<BookingEvent>) {
    // STEP 1: Get user preferences
    const preferences = await this.preferencesService.getPreferences(
      job.data.data.userId,
    );

    // STEP 2: Check if user allows emails
    if (!preferences.emailEnabled) {
      this.logger.log(`User ${job.data.data.userId} has email notifications disabled`);
      return;  // ⛔ STOP - Don't send email
    }

    // STEP 3: Check quiet hours (future feature)
    if (isWithinQuietHours(preferences)) {
      return;  // ⛔ STOP - Respect quiet hours
    }

    // STEP 4: Send notification
    await this.handleBookingCreated(job.data);  // ✅ Proceed
  }
}
```

**Data Flow:**
```
Event Received
    ↓
EmailQueueProcessor
    ↓
PreferencesService.getPreferences(userId)
    ↓
    ├─ Preferences found? → Return preferences
    └─ No preferences?    → Auto-create defaults → Return preferences
    ↓
Check emailEnabled flag
    ↓
    ├─ true  → ✅ Send notification
    └─ false → ❌ Skip notification
```

---

### Used By: NotificationQueueProcessor

**File:** `src/modules/events/processors/notification-queue.processor.ts`

```typescript
@Processor('notifications', { concurrency: 5 })
export class NotificationQueueProcessor extends WorkerHost {
  constructor(
    private readonly notificationsService: NotificationsService,
    private readonly preferencesService: PreferencesService,  // ← Injected
  ) {
    super();
  }

  private async handleBookingReminder(event: any) {
    // Get preferences
    const preferences = await this.preferencesService.getPreferences(
      event.data.userId,
    );

    // Check if reminders enabled
    if (!preferences.reminderEnabled) {
      return;  // ⛔ User disabled reminders
    }

    // Check push enabled
    if (!preferences.pushEnabled) {
      return;  // ⛔ User disabled push notifications
    }

    // Send reminder
    await this.notificationsService.create({
      userId: event.data.userId,
      type: 'push',
      template: 'booking_reminder',
      subject: 'Rappel de réservation',
      content: { hours: 24, parkingName: event.data.parkingName },
      channel: 'push',
      priority: 'normal',
    });
  }
}
```

---

## Code Examples

### Example 1: Check Preferences Before Sending

```typescript
// Business Logic Layer
async sendMarketingEmail(userId: string, campaign: Campaign) {
  // 1. Get user preferences
  const preferences = await this.preferencesService.getPreferences(userId);

  // 2. Check marketing consent (GDPR)
  if (!preferences.marketingEnabled) {
    this.logger.log(`User ${userId} has not opted into marketing emails`);
    return { sent: false, reason: 'marketing_disabled' };
  }

  // 3. Check email channel enabled
  if (!preferences.emailEnabled) {
    this.logger.log(`User ${userId} has disabled email notifications`);
    return { sent: false, reason: 'email_disabled' };
  }

  // 4. Check quiet hours
  if (this.isWithinQuietHours(preferences)) {
    this.logger.log(`User ${userId} is in quiet hours, scheduling for later`);
    await this.scheduleForLater(userId, campaign, preferences.quietHoursEnd);
    return { sent: false, reason: 'quiet_hours', scheduledFor: preferences.quietHoursEnd };
  }

  // 5. Check daily marketing limit
  const sentToday = await this.getMarketingEmailsSentToday(userId);
  if (sentToday >= 1) {
    this.logger.log(`User ${userId} already received marketing email today`);
    return { sent: false, reason: 'daily_limit_reached' };
  }

  // 6. All checks passed - send email
  await this.notificationsService.create({
    userId,
    type: 'email',
    template: campaign.template,
    subject: campaign.subject,
    content: campaign.data,
    channel: 'email',
    priority: 'normal',
  });

  return { sent: true };
}
```

---

### Example 2: User Updates Preferences

```typescript
// User goes to settings page and disables emails
const result = await fetch('/api/v1/preferences/user_001', {
  method: 'PUT',
  headers: {
    'Authorization': 'Bearer <firebase-token>',
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({
    emailEnabled: false,  // User turns OFF emails
  }),
});

// Response
{
  "id": "uuid-123",
  "userId": "user_001",
  "emailEnabled": false,  // ← Updated
  "pushEnabled": true,
  "inAppEnabled": true,
  "marketingEnabled": false,
  "reminderEnabled": true,
  "partnerDigestEnabled": false,
  "timezone": "Europe/Paris",
  "updatedAt": "2025-11-05T10:30:00Z"
}

// Next booking event for user_001
// EmailQueueProcessor runs:
const preferences = await preferencesService.getPreferences('user_001');
// Returns: { emailEnabled: false, ... }

if (!preferences.emailEnabled) {
  return;  // ⛔ STOP - Email notification NOT sent
}
```

---

### Example 3: Real Database Records

**Current preferences in your database:**

```bash
# From check_preferences.js output earlier

User: user_001
   Email: ✅ (emailEnabled = true)
   Push: ✅ (pushEnabled = true)
   In-App: ✅ (inAppEnabled = true)
   Marketing: ✅ (marketingEnabled = true)
   Reminders: ✅ (reminderEnabled = true)
   Partner Digest: ❌ (partnerDigestEnabled = false)
   Timezone: Europe/Paris

User: user_002
   Email: ✅ (emailEnabled = true)
   Push: ❌ (pushEnabled = false)  # ← This user won't receive push!
   In-App: ✅ (inAppEnabled = true)
   Marketing: ❌ (marketingEnabled = false)
   Reminders: ✅ (reminderEnabled = true)
   Partner Digest: ❌ (partnerDigestEnabled = false)
   Timezone: Europe/Paris

User: partner_001
   Email: ✅ (emailEnabled = true)
   Push: ✅ (pushEnabled = true)
   In-App: ✅ (inAppEnabled = true)
   Marketing: ❌ (marketingEnabled = false)
   Reminders: ✅ (reminderEnabled = true)
   Partner Digest: ✅ (partnerDigestEnabled = true)  # ← Partner receives digest
   Timezone: Europe/Paris
```

---

## GDPR Compliance

### Consent Management

**Marketing Opt-In Requirement:**
```typescript
// Default is FALSE - user MUST explicitly opt-in
const defaults = {
  marketingEnabled: false,  // ❌ GDPR: No marketing without consent
};

// User must actively click "I agree to receive marketing emails"
await preferencesService.updatePreferences('user_001', {
  marketingEnabled: true,  // ✅ Explicit consent
});
```

### Right to Be Forgotten

**Delete User Data:**
```http
DELETE /api/v1/preferences/user_001
Authorization: Bearer <firebase-token>
```

**Implementation:**
```typescript
async deletePreferences(userId: string) {
  const preferences = await this.prisma.notificationPreference.findUnique({
    where: { userId },
  });

  if (!preferences) {
    throw new NotFoundException(`Preferences for user ${userId} not found`);
  }

  // Delete all preference data
  return this.prisma.notificationPreference.delete({
    where: { userId },
  });
}
```

### Data Export

**Export User Preferences:**
```http
GET /api/v1/preferences/user_001
Authorization: Bearer <firebase-token>
```

Returns complete preference data for GDPR data portability.

---

## Module Files

```
src/modules/preferences/
│
├── preferences.module.ts              # Module definition
├── preferences.controller.ts          # API endpoints (Presentation Layer)
├── preferences.service.ts             # Business logic (Service Layer)
│
└── dto/
    ├── create-preference.dto.ts       # Create request validation
    ├── update-preference.dto.ts       # Update request validation
    ├── preference-response.dto.ts     # API response format
    └── README.md                      # DTO documentation
```

### Module Configuration

**File:** `preferences.module.ts`

```typescript
@Module({
  controllers: [PreferencesController],  // REST API endpoints
  providers: [PreferencesService],       // Business logic
  exports: [PreferencesService],         // ← Exported for use by other modules
})
export class PreferencesModule {}
```

**Exported to:**
- `EventsModule` (for queue processors)
- Any module that needs to check user preferences

---

## Summary

### What is PreferencesModule?

A **GDPR-compliant notification settings manager** that:
- ✅ Auto-creates sensible defaults for first-time users
- ✅ Allows users to control which notifications they receive
- ✅ Supports channel selection (email, push, in-app)
- ✅ Enables quiet hours configuration
- ✅ Enforces privacy-first defaults (marketing OFF)
- ✅ Integrates seamlessly with all queue processors

### How It Works (High-Level Flow)

```
1. User receives first notification
   ↓
2. EmailQueueProcessor calls preferencesService.getPreferences(userId)
   ↓
3. PreferencesService checks database
   ↓
   ├─ Found?     → Return existing preferences
   └─ Not found? → Auto-create defaults → Return preferences
   ↓
4. Processor checks emailEnabled flag
   ↓
   ├─ true  → ✅ Send notification
   └─ false → ❌ Skip notification (user's choice respected)
```

### Key Takeaway

**PreferencesService is the gatekeeper** that ensures:
- Users control their notification experience
- GDPR compliance through explicit consent
- Privacy-first approach (marketing disabled by default)
- Seamless auto-creation for new users
- **Respects user choice before EVERY notification**

---

## Related Documentation

- [Events Module README](../events/README.md) - Queue processors that use preferences
- [Notifications Module README](../notifications/README.md) - Notification creation
- [BRD - Notification Service](../../BRD_NOTIFICATION_SERVICE.md) - Complete business requirements

---

**For Questions or Updates:** Contact the development team or refer to the main BRD document.
