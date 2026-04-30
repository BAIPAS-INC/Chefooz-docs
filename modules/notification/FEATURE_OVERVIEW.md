# Notification Module — Feature Overview

**Last Updated:** 30 April 2026

---

## Purpose

Delivers real-time push notifications and an in-app inbox to all user types (customer, chef, rider). Covers social engagement, order lifecycle, financial events, moderation actions, and system announcements.

---

## Notification Channels

| Channel | Description |
|---------|-------------|
| Push (Expo) | Device push via Expo Push Service → APNs / FCM |
| In-app inbox | Persisted in `notifications` table; fetched with cursor pagination |
| Email | Critical events only (orders placed/delivered, payouts, cancellations) |

---

## Notification Categories

| Category | Events |
|----------|--------|
| `social` | Likes, comments, replies, follows, follow requests, tags, mentions, shares |
| `orders` | Order placed/accepted/cooking/ready/dispatched/delivered/cancelled/refunded |
| `creator` | Payout approved/paid/rejected, commission earned, reel viral |
| `system` | Moderation decisions, account warnings, announcements, maintenance |
| `message` | Direct messages |

---

## Engagement Notification Messages (Instagram-style)

All engagement push notifications use specific, personalised copy:

| Event | Push Title | Push Body |
|-------|-----------|-----------|
| Like | `New Like ❤️` | `{username} liked your reel` |
| Comment | `New Comment 💬` | `{username} commented: "{comment}"` |
| Reply | `New Reply 💬` | `{username} replied to your comment: "{replyText}"` |
| Follow | `New Follower 🎉` | `{username} started following you!` |
| Follow request | `New Follow Request` | `{username} has requested to follow you.` |
| Mention | `{username} mentioned you 💬` | `{username} mentioned you in a comment: "{comment}"` |
| Tag | `{username} tagged you in a post 🏷️` | `{username} tagged you in a reel.` |
| Save | `Reel Saved 🔖` | `{username} saved your reel` |
| Share | `Reel Shared 📤` | `{username} shared your reel` |
| Direct reel share | `{username} shared a reel with you 🍳` | `{username} sent you a reel. Tap to watch!` |

---

## Grouped Notifications (Inbox)

The in-app inbox groups engagement notifications for the same target within 24 hours:

- **Single actor**: shows the backend-rendered body verbatim  
  e.g. `chefmaria liked your reel`
- **Multiple actors**: shows the grouped summary  
  e.g. `chefmaria and 3 others liked your reel`
- **Non-groupable** (orders, payouts, system, messages): always shown individually

Sub-type is identified via `metadata.templateKey` stored by the dispatcher.

---

## User Preference Controls

Users can toggle per-category push and email in **Settings → Notifications**:

| Control | Toggleable |
|---------|-----------|
| All push notifications | Yes |
| All email notifications | Yes |
| Order push/email | Yes |
| Social push/email | Yes |
| Creator (payouts) push/email | Yes |
| System | No — always on |

---

## Key Screens

- **Notifications inbox**: `apps/chefooz-app/src/app/notifications/index.tsx`  
- **Notification settings**: `apps/chefooz-app/src/app/profile/settings/notifications.tsx`
