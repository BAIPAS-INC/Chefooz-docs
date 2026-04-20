# Notification Module — Technical Guide

**Last Updated:** March 2026  
**Module:** notification  
**Path:** `apps/chefooz-apis/src/modules/notification/`

---

## Overview

The notification module manages the user inbox, push token registration, unread counts, and notification preferences. It is driven by events from the `notification-orchestrator.service.ts` via the dispatcher.

---

## Architecture

```
Event (e.g. REEL_LIKED)
  → notification-orchestrator.service.ts
    → notification.dispatcher.ts       ← stores Notification entity
    → FCM push via firebase-admin
    → activity.service.ts              ← stores activity feed record
```

---

## Data Model

### `notifications` table (PostgreSQL / TypeORM)

| Column | Type | Notes |
|---|---|---|
| `id` | `int` (PK) | Auto-increment |
| `userId` | `uuid` | Recipient user ID |
| `title` | `varchar(255)` | Push/display title |
| `body` | `text` | Notification body text |
| `type` | `text` | One of: `order`, `engagement`, `system`, `message`, `chef` |
| `metadata` | `jsonb` | Event-specific payload — see below |
| `isRead` | `boolean` | Default false |
| `createdAt` | `timestamp` | Auto |

### `metadata` field shape (by type)

| Type | Fields |
|---|---|
| `engagement` (reel like) | `{ reelId, username, reelThumbnail }` |
| `engagement` (follow) | `{ username, actorUsername }` |
| `order` | `{ orderId, status }` |
| `system` | `{ deepLink? }` |

---

## Key Constraint: `metadata` vs `payload`

> ⚠️ **Critical — do not rename this field.**
>
> The TypeORM entity column is named **`metadata`**. The `NotificationResponseDto` and `NotificationMapper` must both use `metadata`.
>
> A historical bug existed where the mapper read `notification.payload` instead of `notification.metadata` — since `payload` does not exist on the entity, all metadata was returned as `undefined` from the inbox API. This was fixed in March 2026.
>
> The shared type `NotificationItem` has a deprecated `payload` field for backward compatibility — ignore it. Always read from `metadata`.

---

## API Endpoints

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/v1/notifications` | Get notification inbox (cursor paginated) |
| `GET` | `/api/v1/notifications/unread-count` | Unread count |
| `POST` | `/api/v1/notifications/mark-read/:id` | Mark single notification read |
| `POST` | `/api/v1/notifications/mark-all-read` | Mark all read |
| `GET` | `/api/v1/notifications/preferences` | Get preferences |
| `PUT` | `/api/v1/notifications/preferences` | Update preferences |
| `POST` | `/api/v1/notifications/register-device` | Register FCM push token |

---

## Frontend Tap Routing (`notifications/index.tsx`)

The frontend groups notifications by type and routes on tap:

| Type | Has `metadata.reelId` | Destination |
|---|---|---|
| `engagement` | Yes | `/(tabs)/reels/[reelId]` |
| `engagement` | No | `/profile/${metadata.username}` |
| `order` | — | `/orders/${metadata.orderId}` |
| `message` | — | `/messages` |

The `extractUsername()` helper reads `metadata.username || metadata.actorUsername` — both must be populated in the event dispatch (see `feed.service.ts` `REEL_LIKED` handler).

---

## Edge Cases

- If `metadata` is empty `{}`, all deep-link routing falls back to safe defaults (no crash, but incorrect navigation).
- Push notifications sent via FCM carry their own `data` payload (separate from DB `metadata`). Deep-links via push are handled by `deeplink.generator.ts`.
- The `type` in the DB entity uses lowercase (`engagement`, `order`) while an old DTO `@ApiProperty` description listed uppercase values — the actual runtime values are lowercase.
