# Notification Preferences Module - Usage Examples

This document provides real-world examples of how to use the Preferences module in your notification workflows.

## Table of Contents

1. [Service Layer Usage](#service-layer-usage)
2. [Queue Processor Integration](#queue-processor-integration)
3. [Controller Examples](#controller-examples)
4. [Utility Functions](#utility-functions)
5. [Common Patterns](#common-patterns)

---

## Service Layer Usage

### Example 1: Check Preferences Before Sending Email

```typescript
import { Injectable, Logger } from '@nestjs/common';
import { PreferencesService } from '../preferences/preferences.service';
import { PreferenceGuards } from '../preferences/utils';

@Injectable()
export class EmailService {
  private readonly logger = new Logger(EmailService.name);

  constructor(private readonly preferencesService: PreferencesService) {}

  async sendBookingConfirmation(userId: string, bookingData: any) {
    // Get user preferences (auto-creates if doesn't exist)
    const preferences = await this.preferencesService.getPreferences(userId);

    // Check if user allows emails
    if (!PreferenceGuards.canSendEmail(preferences)) {
      this.logger.log(`User ${userId} has email notifications disabled`);
      return { sent: false, reason: 'email_disabled' };
    }

    // Check quiet hours (optional - transactional emails often bypass this)
    if (PreferenceGuards.isWithinQuietHours(preferences)) {
      this.logger.log(`User ${userId} is in quiet hours, sending anyway (transactional)`);
      // For transactional emails, we send anyway
    }

    // Send email
    await this.sendEmail(userId, bookingData);
    return { sent: true };
  }
}
```

### Example 2: Marketing Email with GDPR Compliance

```typescript
import { Injectable, Logger } from '@nestjs/common';
import { PreferencesService } from '../preferences/preferences.service';
import { PreferenceGuards } from '../preferences/utils';

@Injectable()
export class MarketingService {
  private readonly logger = new Logger(MarketingService.name);

  constructor(private readonly preferencesService: PreferencesService) {}

  async sendMarketingCampaign(userId: string, campaign: any) {
    const preferences = await this.preferencesService.getPreferences(userId);

    // GDPR: Check marketing consent (required!)
    if (!PreferenceGuards.hasMarketingConsent(preferences)) {
      this.logger.log(`User ${userId} has not opted into marketing emails (GDPR)`);
      return { sent: false, reason: 'no_marketing_consent' };
    }

    // Check email channel enabled
    if (!PreferenceGuards.canSendEmail(preferences)) {
      this.logger.log(`User ${userId} has disabled email notifications`);
      return { sent: false, reason: 'email_disabled' };
    }

    // Respect quiet hours for marketing emails
    if (PreferenceGuards.shouldDelayForQuietHours(preferences, false)) {
      this.logger.log(`User ${userId} is in quiet hours, scheduling for later`);
      await this.scheduleForLater(userId, campaign, preferences.quietHoursEnd);
      return { sent: false, reason: 'quiet_hours', scheduledFor: preferences.quietHoursEnd };
    }

    // Send marketing email
    await this.sendEmail(userId, campaign);
    return { sent: true };
  }
}
```

---

## Queue Processor Integration

### Example 3: Email Queue Processor with Preference Checks

```typescript
import { Processor, WorkerHost } from '@nestjs/bullmq';
import { Logger } from '@nestjs/common';
import { Job } from 'bullmq';
import { PreferencesService } from '../../preferences/preferences.service';
import { PreferenceGuards } from '../../preferences/utils';

interface BookingEvent {
  userId: string;
  bookingId: string;
  type: 'created' | 'updated' | 'cancelled';
  data: any;
}

@Processor('emails', { concurrency: 5 })
export class EmailQueueProcessor extends WorkerHost {
  private readonly logger = new Logger(EmailQueueProcessor.name);

  constructor(
    private readonly preferencesService: PreferencesService,
  ) {
    super();
  }

  async process(job: Job<BookingEvent>): Promise<any> {
    this.logger.log(`Processing email job: ${job.id}`);

    // STEP 1: Get user preferences
    const preferences = await this.preferencesService.getPreferences(
      job.data.userId,
    );

    // STEP 2: Check channel enabled
    if (!PreferenceGuards.canSendEmail(preferences)) {
      this.logger.log(
        `User ${job.data.userId} has email notifications disabled - skipping`,
      );
      return { skipped: true, reason: 'email_disabled' };
    }

    // STEP 3: Check quiet hours (optional for non-urgent)
    const isUrgent = job.data.type === 'cancelled'; // Cancellations are urgent

    if (PreferenceGuards.shouldDelayForQuietHours(preferences, isUrgent)) {
      this.logger.log(
        `User ${job.data.userId} is in quiet hours - rescheduling`,
      );
      // Re-queue for after quiet hours
      await this.rescheduleJob(job, preferences.quietHoursEnd);
      return { delayed: true, reason: 'quiet_hours' };
    }

    // STEP 4: Send email
    await this.sendBookingEmail(job.data);

    return { sent: true };
  }

  private async sendBookingEmail(event: BookingEvent) {
    // Implementation...
  }

  private async rescheduleJob(job: Job, delayUntil: Date) {
    // Implementation...
  }
}
```

### Example 4: Push Notification Processor

```typescript
import { Processor, WorkerHost } from '@nestjs/bullmq';
import { Logger } from '@nestjs/common';
import { Job } from 'bullmq';
import { PreferencesService } from '../../preferences/preferences.service';
import { PreferenceGuards } from '../../preferences/utils';

@Processor('notifications', { concurrency: 10 })
export class PushNotificationProcessor extends WorkerHost {
  private readonly logger = new Logger(PushNotificationProcessor.name);

  constructor(
    private readonly preferencesService: PreferencesService,
  ) {
    super();
  }

  async process(job: Job): Promise<any> {
    const userId = job.data.userId;

    // Get preferences
    const preferences = await this.preferencesService.getPreferences(userId);

    // Check push enabled
    if (!PreferenceGuards.canSendPush(preferences)) {
      this.logger.log(`User ${userId} has push notifications disabled`);
      return { skipped: true };
    }

    // Check if it's a reminder and user wants reminders
    if (job.data.category === 'reminder' && !PreferenceGuards.wantsReminders(preferences)) {
      this.logger.log(`User ${userId} has reminders disabled`);
      return { skipped: true };
    }

    // Send push notification
    await this.sendPushNotification(job.data);
    return { sent: true };
  }

  private async sendPushNotification(data: any) {
    // Implementation...
  }
}
```

---

## Controller Examples

### Example 5: User Updates Their Preferences

```typescript
import { Controller, Put, Body, Param } from '@nestjs/common';
import { PreferencesService } from './preferences.service';
import { UpdatePreferenceDto } from './dto/update-preference.dto';

@Controller('api/v1/user')
export class UserSettingsController {
  constructor(private readonly preferencesService: PreferencesService) {}

  @Put(':userId/settings/notifications')
  async updateNotificationSettings(
    @Param('userId') userId: string,
    @Body() updateDto: UpdatePreferenceDto,
  ) {
    // Update preferences (upsert behavior)
    const preferences = await this.preferencesService.updatePreferences(
      userId,
      updateDto,
    );

    return {
      message: 'Notification preferences updated successfully',
      preferences,
    };
  }
}
```

---

## Utility Functions

### Example 6: Using PreferenceGuards for Complex Logic

```typescript
import { Injectable, Logger } from '@nestjs/common';
import { PreferencesService } from '../preferences/preferences.service';
import { PreferenceGuards, NotificationChannel } from '../preferences/utils';

@Injectable()
export class NotificationRouter {
  private readonly logger = new Logger(NotificationRouter.name);

  constructor(private readonly preferencesService: PreferencesService) {}

  /**
   * Determine which channels to use for a notification
   */
  async getEnabledChannels(userId: string): Promise<NotificationChannel[]> {
    const preferences = await this.preferencesService.getPreferences(userId);
    const enabledChannels: NotificationChannel[] = [];

    if (PreferenceGuards.canSendEmail(preferences)) {
      enabledChannels.push('email');
    }

    if (PreferenceGuards.canSendPush(preferences)) {
      enabledChannels.push('push');
    }

    if (PreferenceGuards.canSendInApp(preferences)) {
      enabledChannels.push('in_app');
    }

    return enabledChannels;
  }

  /**
   * Send notification on all enabled channels
   */
  async sendMultiChannel(
    userId: string,
    notification: any,
    isUrgent: boolean = false,
  ) {
    const preferences = await this.preferencesService.getPreferences(userId);
    const enabledChannels = await this.getEnabledChannels(userId);

    if (enabledChannels.length === 0) {
      this.logger.warn(`User ${userId} has all channels disabled`);
      return { sent: false, reason: 'all_channels_disabled' };
    }

    // Check quiet hours
    if (PreferenceGuards.shouldDelayForQuietHours(preferences, isUrgent)) {
      this.logger.log(`User ${userId} is in quiet hours - delaying`);
      return { sent: false, reason: 'quiet_hours', delayed: true };
    }

    // Send on all enabled channels
    const results = await Promise.all(
      enabledChannels.map((channel) =>
        this.sendOnChannel(userId, notification, channel),
      ),
    );

    return { sent: true, channels: enabledChannels, results };
  }

  private async sendOnChannel(
    userId: string,
    notification: any,
    channel: NotificationChannel,
  ) {
    // Implementation...
  }
}
```

### Example 7: Validation and Error Handling

```typescript
import { Injectable, BadRequestException } from '@nestjs/common';
import { validateQuietHours } from '../preferences/validators';
import { isSupportedTimezone } from '../preferences/validators';

@Injectable()
export class PreferenceValidationService {
  validatePreferenceUpdate(updateDto: any) {
    // Validate timezone if provided
    if (updateDto.timezone && !isSupportedTimezone(updateDto.timezone)) {
      throw new BadRequestException(
        `Unsupported timezone: ${updateDto.timezone}. ` +
        `Please use a valid IANA timezone (e.g., Europe/Paris, America/New_York)`,
      );
    }

    // Validate quiet hours if both are provided
    if (updateDto.quietHoursStart && updateDto.quietHoursEnd) {
      const validation = validateQuietHours(
        updateDto.quietHoursStart,
        updateDto.quietHoursEnd,
      );

      if (!validation.valid) {
        throw new BadRequestException(validation.error);
      }
    }

    // Validate quiet hours not set without both values
    if (
      (updateDto.quietHoursStart && !updateDto.quietHoursEnd) ||
      (!updateDto.quietHoursStart && updateDto.quietHoursEnd)
    ) {
      throw new BadRequestException(
        'Both quietHoursStart and quietHoursEnd must be set together',
      );
    }

    return true;
  }
}
```

---

## Common Patterns

### Pattern 1: Auto-create Preferences on User Registration

```typescript
import { Injectable } from '@nestjs/common';
import { PreferencesService } from '../preferences/preferences.service';

@Injectable()
export class UserRegistrationService {
  constructor(private readonly preferencesService: PreferencesService) {}

  async registerUser(userData: any) {
    // Create user account
    const user = await this.createUserAccount(userData);

    // Option 1: Let preferences auto-create on first notification (recommended)
    // No action needed - preferences will be created automatically

    // Option 2: Explicitly create preferences with custom defaults
    await this.preferencesService.create({
      userId: user.id,
      emailEnabled: true,
      pushEnabled: userData.enablePushNotifications ?? true,
      marketingEnabled: userData.optInToMarketing ?? false, // GDPR
      timezone: userData.timezone ?? 'Europe/Paris',
    });

    return user;
  }

  private async createUserAccount(userData: any) {
    // Implementation...
    return { id: 'user_123' };
  }
}
```

### Pattern 2: Partner Digest Check

```typescript
import { Injectable, Logger } from '@nestjs/common';
import { Cron } from '@nestjs/schedule';
import { PreferencesService } from '../preferences/preferences.service';
import { PreferenceGuards } from '../preferences/utils';

@Injectable()
export class PartnerDigestService {
  private readonly logger = new Logger(PartnerDigestService.name);

  constructor(private readonly preferencesService: PreferencesService) {}

  @Cron('0 8 * * *') // Every day at 8 AM
  async sendDailyDigest() {
    const partners = await this.getAllPartners();

    for (const partner of partners) {
      const preferences = await this.preferencesService.getPreferences(
        partner.userId,
      );

      // Check if partner wants digest
      if (!PreferenceGuards.wantsPartnerDigest(preferences)) {
        this.logger.log(`Partner ${partner.userId} has digest disabled - skipping`);
        continue;
      }

      // Check email enabled
      if (!PreferenceGuards.canSendEmail(preferences)) {
        this.logger.log(`Partner ${partner.userId} has email disabled - skipping`);
        continue;
      }

      // Send digest
      await this.sendDigestEmail(partner);
    }
  }

  private async getAllPartners() {
    return []; // Implementation...
  }

  private async sendDigestEmail(partner: any) {
    // Implementation...
  }
}
```

### Pattern 3: GDPR Data Export

```typescript
import { Injectable } from '@nestjs/common';
import { PreferencesService } from '../preferences/preferences.service';

@Injectable()
export class GdprService {
  constructor(private readonly preferencesService: PreferencesService) {}

  /**
   * Export all user data for GDPR compliance
   */
  async exportUserData(userId: string) {
    const preferences = await this.preferencesService.getPreferences(userId);

    return {
      userId,
      notificationPreferences: {
        channels: {
          email: preferences.emailEnabled,
          push: preferences.pushEnabled,
          inApp: preferences.inAppEnabled,
        },
        categories: {
          marketing: preferences.marketingEnabled,
          reminders: preferences.reminderEnabled,
          partnerDigest: preferences.partnerDigestEnabled,
        },
        quietHours: {
          start: preferences.quietHoursStart,
          end: preferences.quietHoursEnd,
          timezone: preferences.timezone,
        },
        metadata: {
          createdAt: preferences.createdAt,
          updatedAt: preferences.updatedAt,
        },
      },
      // ... other user data
    };
  }

  /**
   * Delete all user data (Right to be Forgotten)
   */
  async deleteUserData(userId: string) {
    // Delete preferences
    await this.preferencesService.deletePreferences(userId);

    // Delete other user data...
  }
}
```

---

## Testing Examples

### Example 8: Unit Test for Preference Guards

```typescript
import { PreferenceGuards } from '../utils/preference-guards.util';

describe('PreferenceGuards', () => {
  const mockPreferences = {
    id: 'pref_123',
    userId: 'user_001',
    emailEnabled: true,
    pushEnabled: false,
    inAppEnabled: true,
    marketingEnabled: false,
    reminderEnabled: true,
    partnerDigestEnabled: false,
    quietHoursStart: new Date('1970-01-01T22:00:00Z'),
    quietHoursEnd: new Date('1970-01-01T08:00:00Z'),
    timezone: 'Europe/Paris',
    createdAt: new Date(),
    updatedAt: new Date(),
  };

  describe('canSendEmail', () => {
    it('should return true when email is enabled', () => {
      expect(PreferenceGuards.canSendEmail(mockPreferences)).toBe(true);
    });

    it('should return false when email is disabled', () => {
      const prefs = { ...mockPreferences, emailEnabled: false };
      expect(PreferenceGuards.canSendEmail(prefs)).toBe(false);
    });
  });

  describe('hasMarketingConsent', () => {
    it('should return false by default (GDPR)', () => {
      expect(PreferenceGuards.hasMarketingConsent(mockPreferences)).toBe(false);
    });

    it('should return true when user opts in', () => {
      const prefs = { ...mockPreferences, marketingEnabled: true };
      expect(PreferenceGuards.hasMarketingConsent(prefs)).toBe(true);
    });
  });

  describe('isWithinQuietHours', () => {
    it('should return true during quiet hours', () => {
      const nightTime = new Date('2025-01-01T23:00:00Z'); // 11 PM
      expect(
        PreferenceGuards.isWithinQuietHours(mockPreferences, nightTime),
      ).toBe(true);
    });

    it('should return false outside quiet hours', () => {
      const dayTime = new Date('2025-01-01T14:00:00Z'); // 2 PM
      expect(
        PreferenceGuards.isWithinQuietHours(mockPreferences, dayTime),
      ).toBe(false);
    });
  });
});
```

---

## Best Practices

1. **Always get preferences** before sending any notification
2. **Respect quiet hours** for non-urgent notifications
3. **Check GDPR consent** for marketing emails
4. **Use PreferenceGuards** for consistent checking logic
5. **Log skipped notifications** for debugging and analytics
6. **Handle auto-creation** gracefully (no errors on first access)
7. **Validate timezone** before updating preferences
8. **Limit marketing emails** to 1 per day (business rule)

---

## API Response Examples

### Success Response

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "userId": "user_001",
  "emailEnabled": true,
  "pushEnabled": true,
  "inAppEnabled": true,
  "marketingEnabled": false,
  "reminderEnabled": true,
  "partnerDigestEnabled": true,
  "quietHoursStart": "22:00:00",
  "quietHoursEnd": "08:00:00",
  "timezone": "Europe/Paris",
  "createdAt": "2025-11-05T10:00:00.000Z",
  "updatedAt": "2025-11-05T10:00:00.000Z"
}
```

### Validation Error Response

```json
{
  "statusCode": 400,
  "message": [
    "quietHoursStart must be in HH:MM:SS format (e.g., 22:00:00)",
    "timezone must be a valid IANA timezone (e.g., Europe/Paris, America/New_York)"
  ],
  "error": "Bad Request"
}
```

---

For more information, see the main [README.md](./README.md) and [BRD](../../BRD_NOTIFICATION_SERVICE.md).
