# Notification Orchestration — Complete Integration Guide

**Version:** 1.0  
**Last Updated:** February 2026  
**Scope:** Full notification pipeline — Event → Orchestrator → [Activity Feed + Push + Email]  
**Business Rule Summary:** Every notification event flows through a single `NotificationOrchestrator.handleEvent()` entry point. No module should send push or email notifications directly. Self-actions are always suppressed. Email is only sent for critical business events (orders, payouts, refunds). Activity feed entries are only created for social events between two distinct users.  
**Rate Limits:** Push: Expo batch limit 100/req. Email: Resend 100 req/s. Preference cache TTL: 5 minutes.  
**Key Constraints:** Circuit breaker trips at 10 consecutive failures. Duplicate emails are suppressed by idempotency guard (eventType + referenceId + recipient).

---

## 📋 Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Complete Event Catalog](#complete-event-catalog)
3. [Orchestrator Entry Point](#orchestrator-entry-point)
4. [Activity Feed Channel](#activity-feed-channel)
5. [Push Notification Channel](#push-notification-channel)
6. [Email Channel](#email-channel)
7. [Notification Templates](#notification-templates)
8. [User Preference System](#user-preference-system)
9. [Deep Link Generation](#deep-link-generation)
10. [Circuit Breaker](#circuit-breaker)
11. [Notification Entities](#notification-entities)
12. [Integration Patterns](#integration-patterns)
13. [Testing Checklist](#testing-checklist)

---

## Architecture Overview

### Pipeline Diagram

```
Any Module (order, social, payment, moderation)
    │
    └── notificationOrchestrator.handleEvent(event)
                │
    ┌───────────┼───────────────────────────────────┐
    │           │                                   │
    ▼           ▼                                   ▼
ActivityService  NotificationDispatcher         EmailService
    │                   │                           │
    ▼                   ▼                           ▼
activity_feed       notification               email_notification
(MongoDB)           (PostgreSQL)               _log (PostgreSQL)
                        │
                        ▼
                  Expo Push API
              https://exp.host/--/api/v2/push/send
```

### Module Structure

```
apps/chefooz-apis/src/modules/notification/
├── dto/
├── email/
│   ├── email-event.dispatcher.ts     # Dispatches email events
│   ├── email.constants.ts            # EMAIL_CONSTANTS, EMAIL_SUBJECTS
│   ├── email.logger.ts               # Structured email logging
│   ├── email.module.ts               # Email sub-module
│   ├── email.provider.ts             # Resend API wrapper
│   ├── email.service.ts              # Core email service (idempotency)
│   ├── email.templates.ts            # HTML email templates
│   └── email.types.ts                # EmailEventType, email DTOs
├── entities/
│   ├── notification.entity.ts        # Notification record
│   ├── notification-preferences.entity.ts
│   └── push-token.entity.ts
├── utils/
│   ├── notification-preference.guard.ts  # Preference check + Redis cache
│   └── deeplink.generator.ts             # Deep link URL generation
├── notification-orchestrator.service.ts  # ← MAIN ENTRY POINT
├── notification.controller.ts
├── notification.dispatcher.ts            # Push sending + preference guard
├── notification.mapper.ts
├── notification.module.ts
├── notification.service.ts
└── notification.templates.ts            # NOTIFICATION_TEMPLATES map
```

---

## Complete Event Catalog

### EVENT_CONFIG_MAP

Every notification event type maps to a configuration object:

```typescript
interface EventConfig {
  createActivity: boolean;    // Create activity feed entry?
  sendPush: boolean;          // Send Expo push notification?
  sendEmail: boolean;         // Send transactional email?
  notificationCategory: 'social' | 'orders' | 'creator' | 'system';
  templateKey: string;        // Key into NOTIFICATION_TEMPLATES
  emailType?: string;         // Key into EmailEventType
  activityType?: string;      // Activity feed action type
  activityEntityType?: string;
}
```

### Social Events

| Event Type | Activity | Push | Email | Category | Template Key |
|-----------|----------|------|-------|----------|-------------|
| `REEL_LIKED` | ✅ | ✅ | ❌ | social | `engagement.like` |
| `REEL_COMMENTED` | ✅ | ✅ | ❌ | social | `engagement.comment` |
| `COMMENT_REPLIED` | ✅ | ✅ | ❌ | social | `engagement.reply` |
| `REEL_SHARED` | ✅ | ✅ | ❌ | social | `engagement.share` |
| `FOLLOWED` | ✅ | ✅ | ❌ | social | `follow.new_follower` |
| `FOLLOW_REQUEST_ACCEPTED` | ✅ | ✅ | ❌ | social | `follow.accepted` |

### Order Events (Customer)

| Event Type | Activity | Push | Email | Category | Template Key |
|-----------|----------|------|-------|----------|-------------|
| `ORDER_PLACED` | ❌ | ✅ | ✅ | orders | `order.placed` |
| `ORDER_ACCEPTED` | ❌ | ✅ | ❌ | orders | `order.accepted` |
| `ORDER_PREP_TIME_UPDATED` | ❌ | ✅ | ❌ | orders | `order.prep_time_updated` |
| `ORDER_READY` | ❌ | ✅ | ❌ | orders | `order.ready` |
| `ORDER_DELIVERED` | ❌ | ✅ | ✅ | orders | `order.delivered` |
| `ORDER_CANCELLED` | ❌ | ✅ | ✅ | orders | `order.cancelled` |
| `ORDER_REFUNDED` | ❌ | ✅ | ✅ | orders | `order.refunded` |

### Order Events (Chef)

| Event Type | Activity | Push | Email | Category | Template Key |
|-----------|----------|------|-------|----------|-------------|
| `NEW_ORDER_RECEIVED` | ❌ | ✅ | ✅ | orders | `chef.new_order` |
| `ORDER_PAYMENT_CONFIRMED` | ❌ | ✅ | ❌ | orders | `payment.success` |

### Delivery Events (Rider)

| Event Type | Activity | Push | Email | Category | Template Key |
|-----------|----------|------|-------|----------|-------------|
| `DELIVERY_REQUEST_RECEIVED` | ❌ | ✅ | ❌ | orders | `delivery.request` |
| `DELIVERY_ASSIGNED` | ❌ | ✅ | ❌ | orders | `delivery.assigned` |
| `DELIVERY_CANCELLED` | ❌ | ✅ | ❌ | orders | `delivery.cancelled` |
| `TIP_RECEIVED` | ❌ | ✅ | ❌ | creator | `delivery.tip` |

### Financial Events (Chef / Creator)

| Event Type | Activity | Push | Email | Category | Template Key |
|-----------|----------|------|-------|----------|-------------|
| `PAYOUT_INITIATED` | ❌ | ✅ | ✅ | creator | `payout.initiated` |
| `PAYOUT_COMPLETED` | ❌ | ✅ | ✅ | creator | `payout.completed` |
| `PAYOUT_FAILED` | ❌ | ✅ | ✅ | creator | `payout.failed` |
| `COMMISSION_EARNED` | ❌ | ✅ | ❌ | creator | `commission.earned` |

### Moderation Events

| Event Type | Activity | Push | Email | Category | Template Key |
|-----------|----------|------|-------|----------|-------------|
| `CONTENT_FLAGGED` | ❌ | ✅ | ❌ | system | `moderation.flagged` |
| `CONTENT_REMOVED` | ❌ | ✅ | ✅ | system | `moderation.removed` |
| `ACCOUNT_WARNING` | ❌ | ✅ | ✅ | system | `moderation.warning` |
| `APPEAL_RESOLVED` | ❌ | ✅ | ✅ | system | `appeal.resolved` |

### System Events

| Event Type | Activity | Push | Email | Category | Template Key |
|-----------|----------|------|-------|----------|-------------|
| `PROFILE_VERIFICATION` | ❌ | ✅ | ✅ | system | `system.verification` |
| `MESSAGE_RECEIVED` | ❌ | ✅ | ❌ | system | `message.received` |
| `ANNOUNCEMENT` | ❌ | ✅ | ❌ | system | `system.announcement` |

**Total: 29 event types across 6 categories**

---

## Orchestrator Entry Point

All notification events enter the system via a single method:

```typescript
// ANY module that needs to trigger a notification:
await this.notificationOrchestrator.handleEvent({
  type: NotificationEventType.ORDER_PLACED,
  recipientUserId: 'user-uuid',
  actorUserId: null,           // null for system events
  entityId: 'order-uuid',      // The entity this notification is about
  metadata: {                  // Template variable data
    orderId: 'ORD-1234',
    chefName: 'Chef Rakesh',
  }
});
```

### NotificationEvent Interface

```typescript
interface NotificationEvent {
  type: NotificationEventType;
  recipientUserId: string;
  actorUserId?: string | null;    // Who triggered (null = system)
  entityId?: string;              // Related entity ID
  metadata?: Record<string, any>; // Template variables
}
```

### Processing Pipeline

```typescript
async handleEvent(event: NotificationEvent): Promise<NotificationOrchestrationResult> {
  // 1. Validate: type + recipientUserId required
  if (!this.validateEvent(event)) return { error: 'Invalid event' };

  // 2. Self-action suppression
  if (event.actorUserId && event.actorUserId === event.recipientUserId) {
    return { activityCreated: false, pushSent: false, emailSent: false };
  }

  // 3. Circuit breaker check
  if (this.circuitOpen) return { error: 'Circuit breaker open' };

  // 4. Lookup EVENT_CONFIG_MAP
  const config = EVENT_CONFIG_MAP[event.type];

  // 5. Generate deep link
  const deepLink = generateDeepLink(event.type, event.metadata);

  // 6. Parallel processing
  await Promise.allSettled([
    config.createActivity ? this.createActivityEntry(event, config) : Promise.resolve(false),
    config.sendPush ? this.sendPushNotification(event, config, deepLink) : Promise.resolve(false),
    config.sendEmail ? this.sendEmailNotification(event, config) : Promise.resolve(false),
  ]);

  // 7. Reset circuit breaker on success
  this.resetCircuitBreaker();
}
```

### Result Object

```typescript
interface NotificationOrchestrationResult {
  activityCreated: boolean;
  pushSent: boolean;
  emailSent: boolean;
  deepLink: string | null;
  error?: string;
}
```

---

## Activity Feed Channel

Activity feed entries capture social interactions for the "Activity" tab in the app.

### When Activities Are Created

- ✅ Social events between two different users
- ❌ Order events (not shown in activity feed)
- ❌ Financial events (not shown in activity feed)
- ❌ System events (not shown in activity feed)
- ❌ Self-actions (actor === recipient)

### Activity Entry Structure (MongoDB)

```typescript
// Stored in MongoDB: chefooz_activity collection
interface ActivityEntry {
  _id: ObjectId;
  recipientUserId: string;   // Who sees this activity
  actorUserId: string;       // Who performed the action
  actorUsername: string;     // Denormalized for display
  actorAvatarUrl: string;    // Denormalized
  activityType: string;      // 'LIKE', 'COMMENT', 'FOLLOW', etc.
  entityId: string;          // Related entity (reelId, orderId)
  entityType: string;        // 'REEL', 'ORDER', 'USER'
  metadata: Record<string, any>;
  isRead: boolean;
  createdAt: Date;
}
```

### Creation Logic

```typescript
private async createActivityEntry(event: NotificationEvent, config: EventConfig): Promise<boolean> {
  if (!config.activityType || !config.activityEntityType) return false;
  if (!event.actorUserId || !event.entityId) return false;

  // Fetch actor profile (username, avatar)
  const actor = await this.userRepo.findOne({ where: { id: event.actorUserId } });
  if (!actor) return false;

  await this.activityService.create({
    recipientUserId: event.recipientUserId,
    actorUserId: event.actorUserId,
    actorUsername: actor.username,
    actorAvatarUrl: actor.avatarUrl,
    activityType: config.activityType,
    entityId: event.entityId,
    entityType: config.activityEntityType,
    metadata: event.metadata || {},
  });

  return true;
}
```

---

## Push Notification Channel

### Push Provider

- **Provider**: Expo Push API (supports both iOS APNs and Android FCM)
- **Endpoint**: `https://exp.host/--/api/v2/push/send`
- **Format**: Expo Push Token (`ExponentPushToken[xxxxxxxxxxxxxxxxxxxx]`)

### Dispatcher Flow

```typescript
// notification.dispatcher.ts

async sendPush(
  event: NotificationEvent,
  config: EventConfig,
  deepLink: string | null
): Promise<boolean> {
  // 1. Get notification category
  const category = config.notificationCategory; // 'social' | 'orders' | 'creator' | 'system'

  // 2. Check user preferences (Redis-cached)
  const allowed = await this.preferenceGuard.isAllowed(event.recipientUserId, category);
  if (!allowed) return false;

  // 3. Get active push tokens for user
  const tokens = await this.getActivePushTokens(event.recipientUserId);
  if (tokens.length === 0) return false;

  // 4. Build notification from template
  const template = NOTIFICATION_TEMPLATES[config.templateKey];
  const title = this.interpolate(template.title, event.metadata);
  const body = this.interpolate(template.body, event.metadata);

  // 5. Save notification record to DB
  await this.saveNotificationRecord({ recipientUserId, title, body, type: template.type, deepLink });

  // 6. Send to all active devices
  const messages = tokens.map(token => ({
    to: token.expoPushToken,
    title,
    body,
    data: { deepLink, type: config.templateKey, entityId: event.entityId },
    sound: 'default',
    badge: 1,
  }));

  await axios.post('https://exp.host/--/api/v2/push/send', messages, {
    headers: { 'Content-Type': 'application/json' }
  });

  return true;
}
```

### Template Variable Interpolation

```typescript
// {{variable}} syntax
const interpolate = (template: string, metadata: Record<string, any> = {}): string => {
  return template.replace(/\{\{(\w+)\}\}/g, (_, key) => String(metadata[key] ?? ''));
};
```

**Example**:
```
Template: "{{username}} liked your reel"
Metadata: { username: "foodie_priya" }
Result:   "foodie_priya liked your reel"
```

### Push Token Management

```typescript
// Push tokens stored in: push_tokens table (PostgreSQL)
interface PushToken {
  id: string;
  userId: string;
  expoPushToken: string;     // "ExponentPushToken[...]"
  platform: 'ios' | 'android';
  deviceId: string;
  isActive: boolean;
  lastUsedAt: Date;
  createdAt: Date;
}
```

**Lifecycle**:
- Registered: `POST /api/v1/notification/push-token` (on app launch)
- Invalidated: When Expo returns `DeviceNotRegistered` error
- Multiple tokens per user: ✅ (user may have multiple devices)

---

## Email Channel

### Email Provider

- **Provider**: Resend (`https://api.resend.com`)
- **From**: `no-reply@chefooz.com`
- **Why Resend**: Developer-first, simple API, great deliverability, GDPR-compliant
- **Fallback**: Can swap to SES / SendGrid without breaking contracts (EmailProvider abstraction)

### Email Event Types

Only **critical transactional events** trigger emails:

```typescript
enum EmailEventType {
  ORDER_CANCELLED = 'order_cancelled',
  REFUND_INITIATED = 'refund_initiated',
  REFUND_COMPLETED = 'refund_completed',
  PAYOUT_INITIATED = 'payout_initiated',
  PAYOUT_COMPLETED = 'payout_completed',
}
```

**Note**: Marketing emails are explicitly forbidden. Only transactional notifications.

### Idempotency Guard

```typescript
// EmailService prevents duplicate emails:
const isAlreadySent = async (
  eventType: EmailEventType,
  referenceId: string,
  to: string
): Promise<boolean> => {
  const log = await this.emailLogRepo.findOne({
    where: { eventType, referenceId, recipient: to }
  });
  return !!log;
};
```

If the same `(eventType + referenceId + recipient)` combination was already sent, the email is skipped. This prevents duplicate emails from retries or duplicate webhook events.

### Email Template Data Interfaces

```typescript
// ORDER_CANCELLED
interface OrderCancelledEmailData {
  orderId: string;
  chefName: string;
  customerName: string;
  cancellationReason: string;
  refundExpected: boolean;
  recipientRole: 'user' | 'chef';
}

// REFUND_INITIATED
interface RefundInitiatedEmailData {
  orderId: string;
  refundAmount: number;
  paymentMethod: string;
  expectedDays: number;  // 3 for UPI, 7 for card
}

// PAYOUT_INITIATED
interface PayoutInitiatedEmailData {
  payoutId: string;
  amount: number;
  periodStart: string;  // "Jan 1, 2026"
  periodEnd: string;    // "Jan 31, 2026"
  expectedDate: string; // "Feb 2, 2026"
}

// PAYOUT_COMPLETED
interface PayoutCompletedEmailData {
  payoutId: string;
  amount: number;
  referenceId: string;  // Bank reference number
}
```

### Email Timeline Constants

```typescript
// email.constants.ts
const EMAIL_CONSTANTS = {
  REFUND_EXPECTED_DAYS_UPI:  3,
  REFUND_EXPECTED_DAYS_CARD: 7,
  PAYOUT_EXPECTED_DAYS:      2,
  SUPPORT_EMAIL: 'support@chefooz.com',
};
```

---

## Notification Templates

All push notification text is defined in `notification.templates.ts`:

### Order Templates

| Template Key | Title | Body |
|-------------|-------|------|
| `order.placed` | `Order Placed Successfully! 🎉` | `Your order #{{orderId}} has been placed. Waiting for chef to accept.` |
| `order.accepted` | `Order Accepted! 🎉` | `Chef {{chefName}} accepted your order #{{orderId}}. Estimated time: {{estimatedTime}} mins.` |
| `order.prep_time_updated` | `Prep Time Updated ⏱️` | `Your order #{{orderId}} prep time has been updated to {{newPrepTime}} minutes.` |
| `order.cooking` | `Order Preparing 👨‍🍳` | `{{chefName}} is now preparing your order #{{orderId}}.` |
| `order.ready` | `Order Ready! 🍽️` | `Your order #{{orderId}} is ready for pickup or delivery.` |
| `order.dispatched` | `Order Out for Delivery 🚴` | `Your order #{{orderId}} is on its way! Track your delivery in real-time.` |
| `order.delivered` | `Order Delivered ✅` | `Your order #{{orderId}} has been delivered. Enjoy your meal!` |
| `order.cancelled` | `Order Cancelled` | `Your order #{{orderId}} has been cancelled. {{reason}}` |
| `order.refunded` | `Refund Processed 💰` | `Your refund for order #{{orderId}} has been processed. Amount: ₹{{amount}}` |

### Chef Templates

| Template Key | Title | Body |
|-------------|-------|------|
| `chef.new_order` | `New Order Received! 🍽️` | `You have a new order from {{customerName}}. Order #{{orderId}}` |
| `chef.new_rating` | `New Rating Received! ⭐` | `{{customerName}} rated you {{rating}}/5 for order #{{orderId}}` |

### Social Templates

| Template Key | Title | Body |
|-------------|-------|------|
| `engagement.like` | `New Like ❤️` | `{{username}} liked your reel` |
| `engagement.comment` | `New Comment 💬` | `{{username}} commented: "{{comment}}"` |
| `engagement.reply` | `New Reply 💬` | `{{username}} replied to your comment: "{{replyText}}"` |
| `engagement.share` | `Reel Shared 📤` | `{{username}} shared your reel` |
| `engagement.follow` | `New Follower 🎉` | `{{username}} started following you!` |
| `follow.accepted` | `Follow Request Accepted ✅` | `{{username}} accepted your follow request.` |
| `follow.requested` | `New Follow Request` | `{{username}} wants to follow you.` |

### Payout Templates

| Template Key | Title | Body |
|-------------|-------|------|
| `payout.approved` | `Withdrawal Approved ✅` | `Your withdrawal request for ₹{{amount}} has been approved.` |
| `payout.paid` | `Payment Sent 💰` | `₹{{amount}} has been sent to your account.` |
| `payout.rejected` | `Withdrawal Rejected` | `Your withdrawal request for ₹{{amount}} was rejected. Reason: {{reason}}` |
| `payout.threshold_reached` | `Withdrawal Available! 💵` | `You've reached ₹{{amount}} in earnings. You can now request a withdrawal.` |

### Moderation Templates

| Template Key | Title | Body |
|-------------|-------|------|
| `moderation.approved` | `Content Approved ✅` | `Your reel is now live and visible to everyone!` |
| `moderation.rejected` | `Content Rejected ❌` | `Your reel violates community guidelines: {{reason}}` |
| `moderation.under_review` | `Under Review 👀` | `Your reel is being reviewed by our moderation team.` |
| `account.suspended` | `Account Suspended 🚫` | `Your account has been suspended due to: {{reason}}` |
| `appeal.approved` | `Appeal Approved ✅` | `Your appeal has been approved. The original decision has been reversed.` |
| `appeal.rejected` | `Appeal Denied` | `Your appeal has been reviewed. The original decision stands.` |

---

## User Preference System

### Preference Categories

Each user can independently toggle notifications per category:

```typescript
type NotificationCategory = 'social' | 'orders' | 'creator' | 'system';

interface NotificationPreferences {
  userId: string;
  social: boolean;     // Likes, comments, follows
  orders: boolean;     // Order lifecycle updates
  creator: boolean;    // Payouts, commissions
  system: boolean;     // Moderation, announcements (⚠️ always on for critical)
}
```

### Preference API Endpoints

```
GET  /api/v1/notification/preferences         → Get current preferences
PUT  /api/v1/notification/preferences         → Update preferences
```

**Update Request**:
```json
{
  "social": true,
  "orders": true,
  "creator": false,
  "system": true
}
```

### Redis Caching

```typescript
// notification-preference.guard.ts

class NotificationPreferenceGuard {
  private readonly CACHE_TTL = 300; // 5 minutes in seconds

  async isAllowed(userId: string, category: NotificationCategory): Promise<boolean> {
    const cacheKey = `notification:prefs:${userId}`;

    // 1. Try Redis cache
    const cached = await this.redis.get(cacheKey);
    if (cached) {
      const prefs = JSON.parse(cached);
      return prefs[category] !== false; // Default: true if not set
    }

    // 2. DB fallback
    const prefs = await this.preferencesRepo.findOne({ where: { userId } });
    if (!prefs) return true; // Default: all notifications allowed

    // 3. Cache for 5 minutes
    await this.redis.setex(cacheKey, this.CACHE_TTL, JSON.stringify(prefs));

    return prefs[category] !== false;
  }
}
```

**Invalidation**: Cache is invalidated whenever preferences are updated via `PUT /notification/preferences`.

### Category Mapping

The dispatcher maps notification types to preference categories:

```typescript
// notification.dispatcher.ts
const mapTypeToCategory = (type: NotificationType): NotificationCategory => {
  switch (type) {
    case 'order':         return 'orders';
    case 'engagement':
    case 'chef':          return 'social';
    case 'system':
    case 'message':       return 'system';
    default:              return 'system';
  }
};
```

---

## Deep Link Generation

Deep links route the user to the correct screen when tapping a notification.

```typescript
// utils/deeplink.generator.ts

const generateDeepLink = (
  eventType: NotificationEventType,
  metadata: Record<string, any>
): string | null => {
  switch (eventType) {
    case NotificationEventType.ORDER_PLACED:
    case NotificationEventType.ORDER_ACCEPTED:
    case NotificationEventType.ORDER_DELIVERED:
      return `chefooz://orders/${metadata.orderId}`;

    case NotificationEventType.NEW_ORDER_RECEIVED:
      return `chefooz://chef/orders/${metadata.orderId}`;

    case NotificationEventType.REEL_LIKED:
    case NotificationEventType.REEL_COMMENTED:
    case NotificationEventType.REEL_SHARED:
      return `chefooz://reels/${metadata.reelId}`;

    case NotificationEventType.FOLLOWED:
    case NotificationEventType.FOLLOW_REQUEST_ACCEPTED:
      return `chefooz://profile/${metadata.actorUserId}`;

    case NotificationEventType.PAYOUT_COMPLETED:
    case NotificationEventType.PAYOUT_INITIATED:
      return `chefooz://wallet/payouts/${metadata.payoutId}`;

    case NotificationEventType.CONTENT_FLAGGED:
    case NotificationEventType.CONTENT_REMOVED:
      return `chefooz://moderation/appeal`;

    case NotificationEventType.MESSAGE_RECEIVED:
      return `chefooz://chat/${metadata.senderId}`;

    default:
      return null;
  }
};
```

**Deep Link Handling** (React Native):
```typescript
// apps/chefooz-app — handled via Expo Router linking config
const linking = {
  prefixes: ['chefooz://'],
  config: {
    screens: {
      '(tabs)': {
        screens: {
          orders: 'orders/:orderId',
          profile: 'profile/:userId',
          reels: 'reels/:reelId',
        }
      }
    }
  }
};
```

---

## Circuit Breaker

The orchestrator implements a simple circuit breaker to prevent cascade failures:

```typescript
// Circuit breaker state
private failureCount = 0;
private readonly CIRCUIT_BREAKER_THRESHOLD = 10;
private circuitOpen = false;
private circuitResetTimeout: NodeJS.Timeout | null = null;

// Opens circuit after 10 consecutive failures
private handleError(event: NotificationEvent, error: Error): void {
  this.failureCount++;

  if (this.failureCount >= this.CIRCUIT_BREAKER_THRESHOLD) {
    this.circuitOpen = true;
    this.logger.error('🔴 Circuit breaker OPEN — notifications disabled');

    // Auto-reset after 60 seconds
    this.circuitResetTimeout = setTimeout(() => {
      this.circuitOpen = false;
      this.failureCount = 0;
      this.logger.log('🟢 Circuit breaker RESET');
    }, 60_000);
  }
}

// Resets on successful processing
private resetCircuitBreaker(): void {
  if (this.failureCount > 0) {
    this.failureCount = Math.max(0, this.failureCount - 1);
  }
}
```

**States**:
- 🟢 **Closed** (normal): All notifications processed
- 🔴 **Open** (tripped): All `handleEvent()` calls return immediately with `error: 'Circuit breaker open'`
- ⏱️ **Auto-reset**: After 60 seconds, circuit closes again

---

## Notification Entities

### `notifications` Table (PostgreSQL)

```sql
CREATE TABLE notifications (
  id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id         UUID NOT NULL REFERENCES users(id),
  title           VARCHAR(255) NOT NULL,
  body            TEXT NOT NULL,
  type            notification_type_enum NOT NULL,
  -- Enum: 'order', 'engagement', 'chef', 'message', 'system'
  is_read         BOOLEAN DEFAULT false,
  deep_link       VARCHAR(512),
  metadata        JSONB,
  created_at      TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_notifications_user_id ON notifications(user_id);
CREATE INDEX idx_notifications_user_unread ON notifications(user_id) WHERE is_read = false;
```

### `push_tokens` Table (PostgreSQL)

```sql
CREATE TABLE push_tokens (
  id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id         UUID NOT NULL REFERENCES users(id),
  expo_push_token VARCHAR(256) NOT NULL,
  platform        VARCHAR(10) NOT NULL, -- 'ios' | 'android'
  device_id       VARCHAR(256),
  is_active       BOOLEAN DEFAULT true,
  last_used_at    TIMESTAMP,
  created_at      TIMESTAMP DEFAULT NOW(),
  UNIQUE(user_id, expo_push_token)
);
```

### `notification_preferences` Table (PostgreSQL)

```sql
CREATE TABLE notification_preferences (
  id        UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id   UUID NOT NULL UNIQUE REFERENCES users(id),
  social    BOOLEAN DEFAULT true,
  orders    BOOLEAN DEFAULT true,
  creator   BOOLEAN DEFAULT true,
  system    BOOLEAN DEFAULT true,
  updated_at TIMESTAMP DEFAULT NOW()
);
```

### `email_notification_logs` Table (PostgreSQL)

```sql
CREATE TABLE email_notification_logs (
  id           UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  event_type   VARCHAR(64) NOT NULL,
  reference_id VARCHAR(256) NOT NULL, -- orderId, payoutId, etc.
  recipient    VARCHAR(256) NOT NULL,
  message_id   VARCHAR(256),          -- Resend message ID
  status       VARCHAR(32) NOT NULL,  -- 'sent', 'failed'
  error        TEXT,
  sent_at      TIMESTAMP DEFAULT NOW(),
  UNIQUE(event_type, reference_id, recipient)
);
```

---

## Integration Patterns

### Pattern 1 — Order Module Integration

```typescript
// apps/chefooz-apis/src/modules/chef-orders/chef-orders.service.ts

@Injectable()
export class ChefOrdersService {
  constructor(
    private readonly notificationOrchestrator: NotificationOrchestrator,
  ) {}

  async acceptOrder(orderId: string, chefId: string): Promise<void> {
    // ... update order status ...

    await this.notificationOrchestrator.handleEvent({
      type: NotificationEventType.ORDER_ACCEPTED,
      recipientUserId: order.userId,          // Notify customer
      actorUserId: chefId,                    // Chef triggered it
      entityId: orderId,
      metadata: {
        orderId: order.shortId,
        chefName: chef.displayName,
        estimatedTime: order.estimatedMinutes,
      },
    });
  }
}
```

### Pattern 2 — Social Module Integration

```typescript
// When a reel is liked
await this.notificationOrchestrator.handleEvent({
  type: NotificationEventType.REEL_LIKED,
  recipientUserId: reel.userId,      // Reel author gets notified
  actorUserId: likerUserId,          // Who liked it
  entityId: reelId,
  metadata: {
    username: liker.username,
    reelId,
  },
});

// Self-like: actorUserId === recipientUserId
// Orchestrator suppresses automatically — no code needed in caller
```

### Pattern 3 — Financial Module Integration

```typescript
// When payout is initiated (withdrawal module)
await this.notificationOrchestrator.handleEvent({
  type: NotificationEventType.PAYOUT_INITIATED,
  recipientUserId: chefId,
  actorUserId: null,            // System-initiated
  entityId: payoutId,
  metadata: {
    amount: payout.amount,
    payoutId,
    periodStart: formatDate(payout.periodStart),
    periodEnd: formatDate(payout.periodEnd),
    expectedDate: formatDate(addDays(new Date(), 2)),
  },
});
// Result: Push notification + Email to chef
```

### Pattern 4 — Moderation Integration

```typescript
// Content removed by admin
await this.notificationOrchestrator.handleEvent({
  type: NotificationEventType.CONTENT_REMOVED,
  recipientUserId: content.userId,
  actorUserId: null,            // System/admin action
  entityId: content.id,
  metadata: {
    reason: 'Violates community guidelines: explicit content',
    contentType: 'REEL',
  },
});
// Result: Push notification + Email to user + Strike count incremented
```

---

## Testing Checklist

### Unit Tests

- [ ] `handleEvent()` — self-action suppressed when actorUserId === recipientUserId
- [ ] `handleEvent()` — circuit breaker skips when circuitOpen = true
- [ ] `handleEvent()` — invalid event (no type) returns error
- [ ] `preferenceGuard.isAllowed()` — respects user social=false
- [ ] `preferenceGuard.isAllowed()` — Redis cache hit avoids DB call
- [ ] `interpolate()` — template variables replaced correctly
- [ ] Email idempotency — duplicate (eventType + referenceId + recipient) skipped
- [ ] `generateDeepLink()` — correct URL per event type

### Integration Tests

```powershell
# Register push token
Invoke-WebRequest -Method POST `
  -Uri "http://localhost:3000/api/v1/notification/push-token" `
  -Headers @{ Authorization = "Bearer $jwt" } `
  -ContentType "application/json" `
  -Body '{ "expoPushToken": "ExponentPushToken[test]", "platform": "ios" }'

# Get notification preferences
Invoke-WebRequest -Method GET `
  -Uri "http://localhost:3000/api/v1/notification/preferences" `
  -Headers @{ Authorization = "Bearer $jwt" }

# Update preferences
Invoke-WebRequest -Method PUT `
  -Uri "http://localhost:3000/api/v1/notification/preferences" `
  -Headers @{ Authorization = "Bearer $jwt" } `
  -ContentType "application/json" `
  -Body '{ "social": false, "orders": true, "creator": true, "system": true }'
```

### Scenario Tests

- [ ] `ORDER_PLACED` → Push ✅ + Email ✅ + Activity ❌
- [ ] `REEL_LIKED` → Push ✅ + Email ❌ + Activity ✅
- [ ] `REEL_LIKED` self-like → No notification at all
- [ ] `PAYOUT_COMPLETED` → Push ✅ + Email ✅ + Activity ❌
- [ ] User with `orders: false` → ORDER_ACCEPTED push skipped
- [ ] Duplicate email blocked by idempotency guard
- [ ] Circuit opens after 10 failures, auto-resets after 60s
- [ ] Invalid push token → `DeviceNotRegistered` → token deactivated

---

## Analytics Events

| Event | Trigger | Properties |
|-------|---------|-----------|
| `notification_sent` | Successful push dispatch | `type`, `category`, `userId` |
| `notification_opened` | User taps notification | `notificationId`, `deepLink` |
| `notification_dismissed` | User swipes away | `notificationId`, `type` |
| `email_sent` | Resend API success | `eventType`, `referenceId` |
| `email_failed` | Resend API failure | `eventType`, `error` |
| `preference_updated` | User changes settings | `category`, `oldValue`, `newValue` |
| `circuit_breaker_opened` | 10 consecutive failures | `failureCount` |

---

[SLICE_COMPLETE ✅]
