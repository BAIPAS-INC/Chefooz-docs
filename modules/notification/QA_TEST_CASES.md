# Notification Module — QA Test Cases

**Last Updated:** 30 April 2026  
**Module:** notification  
**Path:** `apps/chefooz-apis/src/modules/notification/`

---

## Test Case Index

| TC ID | Description | Priority | Status |
|---|---|---|---|
| TC-NOTIF-001 | Tapping a "likes your reel" notification navigates to the reel | P0 | Fixed ✅ |
| TC-NOTIF-002 | Notification inbox returns metadata with reelId and username | P0 | Fixed ✅ |
| TC-NOTIF-003 | Tapping a follow notification navigates to the follower's profile | P1 | Not tested |
| TC-NOTIF-004 | Unread badge count decrements after marking read | P1 | Not tested |
| TC-NOTIF-005 | Push notification tap deep-links to correct reel | P0 | Not tested |
| TC-NOTIF-006 | Like notification shows "liked your reel" not "interacted with your content" | P0 | Fixed ✅ |
| TC-NOTIF-007 | Grouped multi-actor like shows correct copy | P1 | Fixed ✅ |
| TC-NOTIF-008 | Comment notification includes comment text | P0 | Fixed ✅ |
| TC-NOTIF-009 | Mention notification includes comment text in body | P1 | Fixed ✅ |
| TC-NOTIF-010 | Tag notification does not start with @ in body | P2 | Fixed ✅ |
| TC-NOTIF-011 | Direct reel share notification includes sender name in body | P2 | Fixed ✅ |

---

### TC-NOTIF-001: Tapping a "likes your reel" notification navigates to the reel

**Type:** Bug Regression  
**Feature area:** Notification tap handling (`notifications/index.tsx`)  
**Priority:** P0

**Preconditions:**
- User A has posted a reel
- User B has liked User A's reel
- User A is logged in and has at least one unread "likes your reel" notification

**Steps:**
1. Open the app as User A
2. Tap the bell icon to open the notification inbox
3. Tap the notification that says "User B liked your reel"

**Expected result:** The app navigates to the reel viewer (`/(tabs)/reels/[reelId]`) showing the liked reel.

**Actual result (before fix):** App navigated to `/profile/Someone` and displayed a 404 error: `API Error: User @Someone not found`.

**Root cause:** The `NotificationMapper.toNotificationDto()` read `notification.payload` but the TypeORM entity column is named `metadata`. Since `payload` does not exist on the entity, the API always returned `metadata: undefined`. The frontend `extractUsername()` and `targetId` fell back to `'Someone'` and `null` respectively. Without `targetId`, the handler routed to `/profile/${actors[0].username}` which resolved to `/profile/Someone` → 404.

**Fix applied:**  
- `notification.mapper.ts`: Changed `payload: notification.payload` → `metadata: notification.metadata`  
- `notification.dto.ts`: Changed `payload!: Record<string, any>` → `metadata!: Record<string, any>` in `NotificationResponseDto`

**Regression test:** `apps/chefooz-apis/src/modules/notification/notification.mapper.spec.ts`  
**Status:** Fixed ✅

---

### TC-NOTIF-002: Notification inbox returns metadata with reelId and username

**Type:** Bug Regression / Automated  
**Feature area:** Notification mapper (`notification.mapper.ts`)  
**Priority:** P0

**Preconditions:**
- Notification record exists in DB with `metadata = { reelId: "abc", username: "chefuser", reelThumbnail: "..." }`

**Steps:**
1. Call `GET /api/v1/notifications` with a valid auth token
2. Inspect the response body for the notification item

**Expected result:**  
```json
{
  "id": 123,
  "type": "engagement",
  "metadata": { "reelId": "abc", "username": "chefuser", "reelThumbnail": "..." },
  "isRead": false
}
```

**Actual result (before fix):** `metadata` was missing from the response (mapper read non-existent `payload` field).

**Fix applied:** Mapper now correctly maps `metadata: notification.metadata`.  
**Status:** Fixed ✅

---

### TC-NOTIF-003: Tapping a follow notification navigates to the follower's profile

**Type:** Manual  
**Feature area:** Notification tap handling (`notifications/index.tsx`)  
**Priority:** P1

**Steps:**
1. Have User B follow User A
2. As User A, tap the "User B started following you" notification

**Expected result:** Navigates to `/profile/user_b_username`  
**Status:** Not tested

---

### TC-NOTIF-006: Like notification shows specific copy, not generic fallback

**Type:** Bug Regression  
**Feature area:** Notifications inbox — message rendering  
**Priority:** P0

**Preconditions:**
- User B likes User A's reel

**Steps:**
1. User A opens the notifications inbox
2. Find the like notification row

**Expected result:** `"{username} liked your reel"`  
**Actual result (before fix):** `"{username} interacted with your content"` — shown for ALL engagement notification types (likes, comments, follows, saves, mentions, tags)  
**Root cause:** `buildGroupedMessage` in `notifications/index.tsx` had a `switch(type)` where `type` is the coarse DB value `'engagement'`. All `case` branches checked legacy strings like `'REEL_LIKED'` or `'FOLLOWED'` — none ever matched, so every notification fell to the `default` branch returning the generic copy.  
**Fix applied:**
1. Backend `notification.dispatcher.ts`: `templateKey` is now stored in the notification metadata (DB + Expo push payload).
2. Frontend `notifications/index.tsx`: For single-actor groups, `buildGroupedMessage` returns the backend-rendered `body` verbatim. For multi-actor groups, the switch now uses `metadata.templateKey` instead of the coarse `type`.
**Regression test:** Manual — automated unit test recommended for `buildGroupedMessage`  
**Status:** Fixed ✅

---

### TC-NOTIF-007: Grouped multi-actor notification shows correct action word

**Type:** Bug Regression  
**Feature area:** Notifications inbox — grouping  
**Priority:** P1

**Preconditions:**
- Users B, C, D all liked User A's reel within 1 hour (same reelId)

**Steps:**
1. User A opens notifications inbox
2. The three likes should be merged into one row

**Expected result:** `"chefmaria and 2 others liked your reel"`  
**Actual result (before fix):** `"chefmaria and 2 others interacted with your content"`  
**Fix applied:** `buildGroupedMessage` now reads `metadata.templateKey` (e.g. `'engagement.like'`) to pick `NL.multiple.likedReel` template  
**Status:** Fixed ✅

---

### TC-NOTIF-008: Mention push notification includes comment text

**Type:** Bug Regression  
**Feature area:** Push notification — `engagement.mention` template  
**Priority:** P1

**Preconditions:**
- User B comments "@userA great recipe!" on any reel

**Steps:**
1. User A receives push notification

**Expected result:**
- Title: `"{username} mentioned you 💬"`
- Body: `"{username} mentioned you in a comment: "@userA great recipe!"`

**Actual result (before fix):**  
- Title: `"@{username} mentioned you"`  
- Body: `"@{username} mentioned you"` — body was identical to title; comment text was absent  
**Fix applied:** Updated `engagement.mention` template: new body = `{{username}} mentioned you in a comment: "{{comment}}"`. Removed `@` prefix from title (username is already shown without it).  
**Status:** Fixed ✅

---

### TC-NOTIF-009: Tag notification title does not start with @

**Type:** Bug Regression  
**Feature area:** Push notification — `engagement.tag` template  
**Priority:** P2

**Preconditions:**
- User B tags User A in a reel

**Steps:**
1. User A receives push notification  

**Expected result:**
- Title: `"{username} tagged you in a post 🏷️"`
- Body: `"{username} tagged you in a reel."`

**Actual result (before fix):** Title: `"@{username} tagged you"` — the `@` looked like a system artefact rather than rendered copy  
**Fix applied:** Removed `@` prefix; added `🏷️` emoji and "in a post" context to title  
**Status:** Fixed ✅

---

### TC-NOTIF-010: Direct reel share includes sender name in push body

**Type:** Bug Regression  
**Feature area:** Push notification — `reel.shared_direct` template  
**Priority:** P2

**Preconditions:**
- User B shares a reel directly with User A via DM

**Steps:**
1. User A receives push notification

**Expected result:**
- Title: `"{username} shared a reel with you 🍳"`
- Body: `"{username} sent you a reel. Tap to watch!"`

**Actual result (before fix):** Body was just `"Tap to watch!"` — sender context was absent; recipient could not tell who sent it  
**Fix applied:** Updated `reel.shared_direct` body to include `{{username}}`  
**Status:** Fixed ✅
