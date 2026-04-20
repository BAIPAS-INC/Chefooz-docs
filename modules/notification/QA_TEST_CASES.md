# Notification Module — QA Test Cases

**Last Updated:** March 2026  
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
