# 🧪 Activity Module - QA Test Cases

## 📋 **Table of Contents**
- [Test Environment Setup](#test-environment-setup)
- [Category 1: Activity Creation](#category-1-activity-creation)
- [Category 2: Notification Feed](#category-2-notification-feed)
- [Category 3: User Activity History](#category-3-user-activity-history)
- [Category 4: Read/Unread Tracking](#category-4-readunread-tracking)
- [Category 5: Pagination](#category-5-pagination)
- [Category 6: Authorization](#category-6-authorization)
- [Category 7: Self-Action Filtering](#category-7-self-action-filtering)
- [Category 8: Performance](#category-8-performance)
- [Category 9: Edge Cases](#category-9-edge-cases)
- [PowerShell Test Helpers](#powershell-test-helpers)

---

## 🛠️ **Test Environment Setup**

### **Prerequisites**
```bash

# 1. Backend running
cd apps/chefooz-apis
npm run start:dev

# 2. PostgreSQL with activity table
# 3. MongoDB with engagements, comments, reels collections
# 4. Test users created
```

### **Test Data Setup**
```bash

# Create test users
$user1Token = "eyJhbGciOiJIUzI1NiIsInR..." # Chef A
$user2Token = "eyJhbGciOiJIUzI1NiIsInR..." # Chef B

# Create test reel (Chef A)
REELRESPONSE=$(curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/reels" \
  -H "Authorization: Bearer $user1Token" \
  -H "Content-Type: application/json")
    caption = "Test Reel for Activity"
    mediaUrls = @(
      @{ url = "https://example.com/video.mp4"; type = "video" }
    )
  } | ConvertTo-Json)

$reelId = ($reelResponse.Content | jq .).data.id
```

---

## ✅ **Category 1: Activity Creation**

### **Test 1.1: Create Activity on Like**

**Scenario**: User B likes User A's reel → Activity recorded for User A

**Steps**:
```bash

# Step 1: User B likes User A's reel
LIKERESPONSE=$(curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/engagements/like" \
  -H "Authorization: Bearer $user2Token" \
  -H "Content-Type: application/json")

# Step 2: User A checks notification feed
FEEDRESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/activity?limit=1" \
  -H "Authorization: Bearer $user1Token")

$feed = ($feedResponse.Content | jq .).data
```

**Expected**:
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "id": "<uuid>",
        "type": "like",
        "actor": {
          "id": "<user2-id>",
          "username": "chef_b",
          "fullName": "Chef B"
        },
        "entityType": "reel",
        "entityId": "<reel-id>",
        "isRead": false
      }
    ]
  }
}
```

**Validation**:
- ✅ Activity created with `type: 'like'`
- ✅ `userId` = User A (reel owner)
- ✅ `actorId` = User B (liker)
- ✅ `isRead` = false

---

### **Test 1.2: Create Activity on Comment**

**Scenario**: User B comments on User A's reel → Activity recorded

**Steps**:
```bash

COMMENTRESPONSE=$(curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/comments" \
  -H "Authorization: Bearer $user2Token" \
  -H "Content-Type: application/json")
    mediaId = $reelId
    text = "Great recipe! 🍕"
  } | ConvertTo-Json)

# Check User A's feed
FEEDRESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/activity?limit=1" \
  -H "Authorization: Bearer $user1Token")

$feed = ($feedResponse.Content | jq .).data
```

**Expected**:
```json
{
  "items": [
    {
      "type": "comment",
      "actor": { "username": "chef_b" },
      "metadata": {
        "commentText": "Great recipe! 🍕"
      },
      "isRead": false
    }
  ]
}
```

**Validation**:
- ✅ `type: 'comment'`
- ✅ `metadata.commentText` preserved
- ✅ Activity created immediately after comment

---

### **Test 1.3: Create Activity on Follow**

**Scenario**: User B follows User A → Activity recorded

**Steps**:
```bash

FOLLOWRESPONSE=$(curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/follows" \
  -H "Authorization: Bearer $user2Token" \
  -H "Content-Type: application/json")

# Check User A's feed
FEEDRESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/activity?limit=1" \
  -H "Authorization: Bearer $user1Token")
```

**Expected**:
```json
{
  "items": [
    {
      "type": "follow",
      "actor": { "username": "chef_b" },
      "entityType": "user",
      "entityId": "<user1-id>"
    }
  ]
}
```

**Validation**:
- ✅ `type: 'follow'`
- ✅ `entityType: 'user'`
- ✅ `entityId` = User A's ID

---

### **Test 1.4: Activity Creation Failure (Non-Critical)**

**Scenario**: Activity service fails, but parent operation succeeds

**Setup**:
```typescript
// Mock ActivityService to throw error
jest.spyOn(activityService, 'createActivity').mockRejectedValue(
  new Error('Database connection lost')
);
```

**Steps**:
```bash

# User B likes reel (should succeed even if activity fails)
LIKERESPONSE=$(curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/engagements/like" \
  -H "Authorization: Bearer $user2Token" \
  -H "Content-Type: application/json")
```

**Expected**:
- ✅ Like operation succeeds (HTTP 200)
- ✅ Activity creation error logged
- ✅ No HTTP error thrown to client

---

## 📬 **Category 2: Notification Feed**

### **Test 2.1: Get Notification Feed (Basic)**

**Scenario**: Fetch notification timeline (what others did to you)

**Steps**:
```bash

# User A checks their notification feed
FEEDRESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/activity?limit=20" \
  -H "Authorization: Bearer $user1Token")

$feed = ($feedResponse.Content | jq .).data
```

**Expected**:
```json
{
  "success": true,
  "data": {
    "items": [
      { "type": "like", "actor": {...}, "isRead": false },
      { "type": "comment", "actor": {...}, "isRead": false },
      { "type": "follow", "actor": {...}, "isRead": false }
    ],
    "nextCursor": "<uuid>"
  }
}
```

**Validation**:
- ✅ All activities have `userId: user1Id`
- ✅ All actors are OTHER users (not User A)
- ✅ Sorted by `createdAt DESC`

---

### **Test 2.2: Feed Pagination (Cursor-Based)**

**Scenario**: Paginate through notification feed using activity ID cursor

**Steps**:
```bash

# Page 1
PAGE1=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/activity?limit=5" \
  -H "Authorization: Bearer $user1Token")

$nextCursor = ($page1.Content | jq .).data.nextCursor

# Page 2
PAGE2=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/activity?limit=5&cursor=$nextCursor" \
  -H "Authorization: Bearer $user1Token")

$items1 = ($page1.Content | jq .).data.items
$items2 = ($page2.Content | jq .).data.items
```

**Validation**:
- ✅ No overlapping items between pages
- ✅ Page 2 items older than Page 1 items
- ✅ Each page has ≤ 5 items

---

### **Test 2.3: Empty Feed**

**Scenario**: New user with no activities

**Steps**:
```bash

# Create new user with no social interactions
$newUserToken = "..." # New JWT

FEEDRESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/activity?limit=20" \
  -H "Authorization: Bearer $newUserToken")
```

**Expected**:
```json
{
  "success": true,
  "data": {
    "items": [],
    "nextCursor": null
  }
}
```

**Validation**:
- ✅ Empty array returned
- ✅ `nextCursor` is null
- ✅ No errors thrown

---

### **Test 2.4: Feed with Deleted Actors**

**Scenario**: Activity exists but actor account deleted

**Steps**:
```bash

# 1. User B likes User A's reel (activity created)
# 2. Delete User B's account
# 3. User A checks feed
FEEDRESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/activity?limit=20" \
  -H "Authorization: Bearer $user1Token")
```

**Expected**:
```json
{
  "items": [
    {
      "type": "like",
      "actor": {
        "username": "unknown",
        "fullName": "Unknown User"
      }
    }
  ]
}
```

**Validation**:
- ✅ Activity still returned (not filtered out)
- ✅ Actor info shows "Unknown User"
- ✅ No database errors

---

## 📜 **Category 3: User Activity History**

### **Test 3.1: Get User History (All Types)**

**Scenario**: Fetch aggregated history of user's own actions

**Steps**:
```bash

# User A checks their own activity history
HISTORYRESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/activity/me?limit=20" \
  -H "Authorization: Bearer $user1Token")

$history = ($historyResponse.Content | jq .).data
```

**Expected**:
```json
{
  "success": true,
  "data": {
    "items": [
      { "id": "1", "type": "LIKE", "createdAt": "2025-01-10T15:00:00Z", "reel": {...} },
      { "id": "2", "type": "COMMENT", "createdAt": "2025-01-10T14:30:00Z", "reel": {...} },
      { "id": "3", "type": "FOLLOW", "createdAt": "2025-01-10T12:00:00Z", "target": {...} },
      { "id": "4", "type": "SAVE", "createdAt": "2025-01-10T10:00:00Z", "reel": {...} }
    ],
    "nextCursor": "2025-01-09T08:00:00Z",
    "totalCount": 150
  }
}
```

**Validation**:
- ✅ All activity types included (LIKE, COMMENT, FOLLOW, SAVE)
- ✅ Sorted by `createdAt DESC`
- ✅ Each item has correct `type` enum value
- ✅ `totalCount` matches actual count

---

### **Test 3.2: Filter History by Type (LIKE)**

**Scenario**: View only reels you liked

**Steps**:
```bash

HISTORYRESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/activity/me?type=LIKE&limit=20" \
  -H "Authorization: Bearer $user1Token")

$history = ($historyResponse.Content | jq .).data
```

**Expected**:
```json
{
  "items": [
    { "type": "LIKE", "reel": { "reelId": "...", "reelThumbnail": "..." } },
    { "type": "LIKE", "reel": {...} }
  ]
}
```

**Validation**:
- ✅ ALL items have `type: "LIKE"`
- ✅ No other types included
- ✅ Each item has `reel` field
- ✅ No `target` field (only for FOLLOW)

---

### **Test 3.3: Filter History by Type (COMMENT)**

**Scenario**: View comments you posted

**Steps**:
```bash

HISTORYRESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/activity/me?type=COMMENT&limit=20" \
  -H "Authorization: Bearer $user1Token")

$history = ($historyResponse.Content | jq .).data
```

**Expected**:
```json
{
  "items": [
    {
      "type": "COMMENT",
      "commentText": "Looks delicious! 🍕",
      "reel": { "reelId": "...", "reelCaption": "Pizza Recipe" }
    }
  ]
}
```

**Validation**:
- ✅ ALL items have `type: "COMMENT"`
- ✅ `commentText` field populated
- ✅ `reel` field included

---

### **Test 3.4: Filter History by Type (FOLLOW)**

**Scenario**: View users you followed

**Steps**:
```bash

HISTORYRESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/activity/me?type=FOLLOW&limit=20" \
  -H "Authorization: Bearer $user1Token")

$history = ($historyResponse.Content | jq .).data
```

**Expected**:
```json
{
  "items": [
    {
      "type": "FOLLOW",
      "target": {
        "userId": "...",
        "username": "chef_rahul",
        "fullName": "Rahul Kumar"
      }
    }
  ]
}
```

**Validation**:
- ✅ ALL items have `type: "FOLLOW"`
- ✅ `target` field with user info
- ✅ No `reel` field

---

### **Test 3.5: Filter History by Type (SAVE)**

**Scenario**: View reels you saved to collections

**Steps**:
```bash

HISTORYRESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/activity/me?type=SAVE&limit=20" \
  -H "Authorization: Bearer $user1Token")

$history = ($historyResponse.Content | jq .).data
```

**Expected**:
```json
{
  "items": [
    { "type": "SAVE", "reel": { "reelId": "...", "reelThumbnail": "..." } }
  ]
}
```

**Validation**:
- ✅ ALL items have `type: "SAVE"`
- ✅ Each item has `reel` field

---

### **Test 3.6: History Pagination (Timestamp Cursor)**

**Scenario**: Paginate through user history using timestamp cursor

**Steps**:
```bash

# Page 1
PAGE1=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/activity/me?limit=10" \
  -H "Authorization: Bearer $user1Token")

$nextCursor = ($page1.Content | jq .).data.nextCursor

# Page 2
PAGE2=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/activity/me?cursor=$nextCursor&limit=10" \
  -H "Authorization: Bearer $user1Token")

$items1 = ($page1.Content | jq .).data.items
$items2 = ($page2.Content | jq .).data.items
```

**Validation**:
- ✅ No duplicate items
- ✅ `nextCursor` is ISO timestamp string
- ✅ Page 2 items older than Page 1 last item

---

### **Test 3.7: History with Deleted Reels**

**Scenario**: User liked a reel that was later deleted

**Steps**:
```bash

# 1. User A likes reel (recorded in engagements)
# 2. Reel owner deletes reel (soft-delete)
# 3. User A checks history
HISTORYRESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/activity/me?type=LIKE&limit=20" \
  -H "Authorization: Bearer $user1Token")
```

**Expected**:
```json
{
  "items": [
    {
      "type": "LIKE",
      "reel": undefined  // ✅ Gracefully handled
    }
  ]
}
```

**Validation**:
- ✅ Activity item still returned
- ✅ `reel` field is undefined (not included in JSON)
- ✅ No errors thrown

---

## 🔵 **Category 4: Read/Unread Tracking**

### **Test 4.1: Get Unread Count**

**Scenario**: Check badge count for unread notifications

**Steps**:
```bash

COUNTRESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/activity/unread-count" \
  -H "Authorization: Bearer $user1Token")

$count = ($countResponse.Content | jq .).data.count
```

**Expected**:
```json
{
  "success": true,
  "data": {
    "count": 5
  }
}
```

**Validation**:
- ✅ Count matches activities with `isRead: false`
- ✅ Response time < 100ms (indexed query)

---

### **Test 4.2: Mark Single Activity as Read**

**Scenario**: Mark specific notification as read

**Steps**:
```bash

# Get activity ID from feed
FEEDRESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/activity?limit=1" \
  -H "Authorization: Bearer $user1Token")

$activityId = ($feedResponse.Content | jq .).data.items[0].id

# Mark as read
MARKRESPONSE=$(curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/activity/read/$activityId" \
  -H "Authorization: Bearer $user1Token")

# Verify updated
VERIFYRESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/activity?limit=1" \
  -H "Authorization: Bearer $user1Token")

$isRead = ($verifyResponse.Content | jq .).data.items[0].isRead
```

**Expected**:
- ✅ Mark response: `{ success: true }`
- ✅ Verify response: `isRead: true`

---

### **Test 4.3: Mark All as Read**

**Scenario**: Bulk update all unread notifications

**Steps**:
```bash

# Check initial unread count
COUNTBEFORE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/activity/unread-count" \
  -H "Authorization: Bearer $user1Token")

# Mark all as read
MARKRESPONSE=$(curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/activity/read-all" \
  -H "Authorization: Bearer $user1Token")

$markedCount = ($markResponse.Content | jq .).data.count

# Verify unread count is now 0
COUNTAFTER=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/activity/unread-count" \
  -H "Authorization: Bearer $user1Token")
```

**Expected**:
- ✅ `markedCount` = initial unread count
- ✅ `countAfter.data.count` = 0

---

### **Test 4.4: Mark Read (Unauthorized)**

**Scenario**: User tries to mark another user's activity

**Steps**:
```bash

# Get User A's activity ID
FEEDA=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/activity?limit=1" \
  -H "Authorization: Bearer $user1Token")

$activityIdA = ($feedA.Content | jq .).data.items[0].id

# User B tries to mark User A's activity
MARKRESPONSE=$(curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/activity/read/$activityIdA" \
  -H "Authorization: Bearer $user2Token")
```

**Expected**:
```json
{
  "success": false,
  "message": "Activity not found or not authorized",
  "errorCode": "ACTIVITY_NOT_FOUND"
}
```

**Validation**:
- ✅ HTTP 404 status
- ✅ `errorCode: "ACTIVITY_NOT_FOUND"`
- ✅ Database not updated

---

### **Test 4.5: Read State Persistence**

**Scenario**: Marked activities stay read across sessions

**Steps**:
```bash

# Mark activity as read
$activityId = "..."
curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/activity/read/$activityId" \
  -H "Authorization: Bearer $user1Token"

# Wait 5 seconds
Start-Sleep -Seconds 5

# Fetch feed again (new request)
FEEDRESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/activity?limit=20" \
  -H "Authorization: Bearer $user1Token")

$activity = ($feedResponse.Content | jq .).data.items | Where-Object { $_.id -eq $activityId }
```

**Expected**:
- ✅ `activity.isRead` is still `true`

---

## 📄 **Category 5: Pagination**

### **Test 5.1: Notification Feed Pagination (50 Items)**

**Scenario**: Paginate through large feed with max limit

**Steps**:
```bash

$allItems = @()
$cursor = $null

do {
  $url = "https://api-staging.chefooz.com/api/v1/activity?limit=50"
  if ($cursor) { $url += "&cursor=$cursor" }
  
RESPONSE=$(curl -s \
  -X GET \
  "$url" \
  -H "Authorization: Bearer $user1Token")
  
  $data = ($response.Content | jq .).data
  $allItems += $data.items
  $cursor = $data.nextCursor
} while ($cursor)

echo "Total items fetched: $($allItems.Count)"
```

**Validation**:
- ✅ All items unique (no duplicates)
- ✅ Items sorted chronologically
- ✅ Pagination completes without errors

---

### **Test 5.2: User History Pagination (Timestamp Cursor)**

**Scenario**: Paginate through user history using timestamp

**Steps**:
```bash

PAGE1=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/activity/me?limit=5" \
  -H "Authorization: Bearer $user1Token")

$cursor = ($page1.Content | jq .).data.nextCursor

PAGE2=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/activity/me?cursor=$cursor&limit=5" \
  -H "Authorization: Bearer $user1Token")

$items1 = ($page1.Content | jq .).data.items
$items2 = ($page2.Content | jq .).data.items
```

**Validation**:
- ✅ `cursor` is valid ISO timestamp
- ✅ Page 2 items have `createdAt < cursor`
- ✅ No overlap between pages

---

### **Test 5.3: Limit Enforcement (Max 50)**

**Scenario**: Client requests limit > 50, backend enforces max

**Steps**:
```bash

RESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/activity?limit=100" \
  -H "Authorization: Bearer $user1Token")

$items = ($response.Content | jq .).data.items
```

**Expected**:
- ✅ Max 50 items returned (not 100)
- ✅ `items.Count` ≤ 50

---

## 🔒 **Category 6: Authorization**

### **Test 6.1: Feed Requires JWT**

**Scenario**: Unauthorized access denied

**Steps**:
```bash

RESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/activity")
```

**Expected**:
```json
{
  "statusCode": 401,
  "message": "Unauthorized"
}
```

---

### **Test 6.2: User History Requires JWT**

**Steps**:
```bash

RESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/activity/me")
```

**Expected**:
- ✅ HTTP 401 Unauthorized

---

### **Test 6.3: Feed Isolation (User A vs User B)**

**Scenario**: User A cannot see User B's feed

**Steps**:
```bash

# User A's feed
FEEDA=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/activity?limit=20" \
  -H "Authorization: Bearer $user1Token")

# User B's feed
FEEDB=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/activity?limit=20" \
  -H "Authorization: Bearer $user2Token")

$itemsA = ($feedA.Content | jq .).data.items
$itemsB = ($feedB.Content | jq .).data.items
```

**Validation**:
- ✅ NO overlapping activity IDs
- ✅ All itemsA have `userId: user1Id`
- ✅ All itemsB have `userId: user2Id`

---

## 🚫 **Category 7: Self-Action Filtering**

### **Test 7.1: Like Own Reel (No Activity)**

**Scenario**: User likes their own reel → No activity recorded

**Steps**:
```bash

# User A creates reel
REELRESPONSE=$(curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/reels" \
  -H "Authorization: Bearer $user1Token")

$reelId = ($reelResponse.Content | jq .).data.id

# User A likes their own reel
curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/engagements/like" \
  -H "Authorization: Bearer $user1Token"

# Check User A's feed
FEEDRESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/activity?limit=20" \
  -H "Authorization: Bearer $user1Token")

$selfLikeActivity = ($feedResponse.Content | jq .).data.items | 
  Where-Object { $_.type -eq "like" -and $_.entityId -eq $reelId }
```

**Expected**:
- ✅ `$selfLikeActivity` is empty (no activity created)

---

### **Test 7.2: Comment on Own Reel (No Activity)**

**Steps**:
```bash

# User A comments on their own reel
curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/comments" \
  -H "Authorization: Bearer $user1Token"

# Check feed
FEEDRESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/activity?limit=20" \
  -H "Authorization: Bearer $user1Token")

$selfCommentActivity = ($feedResponse.Content | jq .).data.items | 
  Where-Object { $_.type -eq "comment" -and $_.entityId -eq $reelId }
```

**Expected**:
- ✅ No self-comment activity recorded

---

### **Test 7.3: Self-Action Logged (Debugging)**

**Scenario**: Verify self-actions are logged but not saved

**Steps**:
1. Enable debug logging in ActivityService
2. User A likes own reel
3. Check application logs

**Expected Log**:
```
[ActivityService] Ignoring self-action: like by user-abc-123
```

---

## ⚡ **Category 8: Performance**

### **Test 8.1: Feed Load Time (p95 < 300ms)**

**Scenario**: Measure notification feed response time

**Steps**:
```bash

$times = @()

for ($i = 0; $i -lt 100; $i++) {
  $start = Get-Date
  
curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/activity?limit=20" \
  -H "Authorization: Bearer $user1Token"
  
  $end = Get-Date
  $times += ($end - $start).TotalMilliseconds
}

$p95 = ($times | Sort-Object)[[Math]::Floor($times.Count * 0.95)]
echo "P95 response time: $p95 ms"
```

**Expected**:
- ✅ P95 < 300ms

---

### **Test 8.2: Unread Count Query Time (< 50ms)**

**Steps**:
```bash

$times = @()

for ($i = 0; $i -lt 50; $i++) {
  $start = Get-Date
  
curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/activity/unread-count" \
  -H "Authorization: Bearer $user1Token"
  
  $end = Get-Date
  $times += ($end - $start).TotalMilliseconds
}

$avg = ($times | Measure-Object -Average).Average
echo "Average response time: $avg ms"
```

**Expected**:
- ✅ Average < 50ms

---

### **Test 8.3: History Aggregation Time (< 500ms)**

**Scenario**: Multi-source fetch performance

**Steps**:
```bash

$start = Get-Date

curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/activity/me?limit=50" \
  -H "Authorization: Bearer $user1Token"

$end = Get-Date
$duration = ($end - $start).TotalMilliseconds

echo "User history fetch time: $duration ms"
```

**Expected**:
- ✅ Duration < 500ms (parallel fetch optimization)

---

### **Test 8.4: Batch User Fetching (N+1 Prevention)**

**Scenario**: Verify single query for multiple actors

**Steps**:
1. Enable database query logging
2. Fetch feed with 20 activities from 10 different actors
3. Count User table queries

**Expected**:
- ✅ Only 1 User query (batch fetch)
- ✅ NOT 20 separate queries

---

## 🧩 **Category 9: Edge Cases**

### **Test 9.1: Activity with Missing Metadata**

**Scenario**: Metadata field is null or empty

**Steps**:
```bash

# Manually create activity with null metadata (via SQL)
# Then fetch feed
FEEDRESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/activity?limit=20" \
  -H "Authorization: Bearer $user1Token")
```

**Expected**:
- ✅ Activity returned with `metadata: {}`
- ✅ No errors thrown

---

### **Test 9.2: Cursor with Invalid Timestamp**

**Scenario**: Malformed cursor in user history

**Steps**:
```bash

RESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/activity/me?cursor=invalid-date&limit=20" \
  -H "Authorization: Bearer $user1Token")
```

**Expected**:
- ✅ HTTP 400 Bad Request
- ✅ Error message: "Invalid cursor format"

---

### **Test 9.3: Limit = 0**

**Steps**:
```bash

RESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/activity?limit=0" \
  -H "Authorization: Bearer $user1Token")
```

**Expected**:
- ✅ HTTP 400 Bad Request
- ✅ Validation error: "limit must be at least 1"

---

### **Test 9.4: Invalid Activity Type Filter**

**Steps**:
```bash

RESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/activity/me?type=INVALID&limit=20" \
  -H "Authorization: Bearer $user1Token")
```

**Expected**:
- ✅ HTTP 400 Bad Request
- ✅ Error: "type must be one of: LIKE, COMMENT, FOLLOW, SAVE"

---

### **Test 9.5: Concurrent Read Operations**

**Scenario**: Two devices mark same activity as read simultaneously

**Steps**:
```bash

$activityId = "..."

# Simulate concurrent requests
$job1 = Start-Job -ScriptBlock {
curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/activity/read/$using:activityId"
}

$job2 = Start-Job -ScriptBlock {
curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/activity/read/$using:activityId"
}

Wait-Job $job1, $job2
```

**Expected**:
- ✅ Both requests succeed (idempotent)
- ✅ Activity `isRead: true` in database

---

## 🛠️ **PowerShell Test Helpers**

### **Helper 1: Create Test Activity**

```bash

function Create-TestActivity {
  param(
    [string]$actorToken,
    [string]$targetReelId,
    [string]$action = "like"  # like, comment, follow
  )
  
  switch ($action) {
    "like" {
curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/engagements/like" \
  -H "Authorization: Bearer $actorToken"
    }
    "comment" {
curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/comments" \
  -H "Authorization: Bearer $actorToken"
    }
  }
}

# Usage
Create-TestActivity -actorToken $user2Token -targetReelId $reelId -action "like"
```

---

### **Helper 2: Get Unread Count**

```bash

function Get-UnreadCount {
  param([string]$token)
  
RESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/activity/unread-count" \
  -H "Authorization: Bearer $token")
  
  return ($response.Content | jq .).data.count
}

# Usage
$count = Get-UnreadCount -token $user1Token
echo "Unread notifications: $count"
```

---

### **Helper 3: Mark All Read**

```bash

function Clear-Notifications {
  param([string]$token)
  
RESPONSE=$(curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/activity/read-all" \
  -H "Authorization: Bearer $token")
  
  $count = ($response.Content | jq .).data.count
  echo "Marked $count activities as read"
}

# Usage
Clear-Notifications -token $user1Token
```

---

### **Helper 4: Fetch Full History**

```bash

function Get-FullHistory {
  param(
    [string]$token,
    [string]$type = $null
  )
  
  $allItems = @()
  $cursor = $null
  
  do {
    $url = "https://api-staging.chefooz.com/api/v1/activity/me?limit=50"
    if ($type) { $url += "&type=$type" }
    if ($cursor) { $url += "&cursor=$cursor" }
    
RESPONSE=$(curl -s \
  -X GET \
  "$url" \
  -H "Authorization: Bearer $token")
    
    $data = ($response.Content | jq .).data
    $allItems += $data.items
    $cursor = $data.nextCursor
  } while ($cursor)
  
  return $allItems
}

# Usage
$allLikes = Get-FullHistory -token $user1Token -type "LIKE"
echo "Total likes: $($allLikes.Count)"
```

---

### **Helper 5: Performance Test**

```bash

function Test-EndpointPerformance {
  param(
    [string]$url,
    [string]$token,
    [int]$iterations = 50
  )
  
  $times = @()
  
  for ($i = 0; $i -lt $iterations; $i++) {
    $start = Get-Date
curl -s \
  -X GET \
  "$url" \
  -H "Authorization: Bearer $token"
    $end = Get-Date
    $times += ($end - $start).TotalMilliseconds
  }
  
  $sorted = $times | Sort-Object
  $p50 = $sorted[[Math]::Floor($times.Count * 0.50)]
  $p95 = $sorted[[Math]::Floor($times.Count * 0.95)]
  $p99 = $sorted[[Math]::Floor($times.Count * 0.99)]
  
  echo "P50: $p50 ms | P95: $p95 ms | P99: $p99 ms"
}

# Usage
Test-EndpointPerformance -url "https://api-staging.chefooz.com/api/v1/activity" -token $user1Token
```

---

## 📊 **Test Summary**

| Category | Test Count | Critical |
|----------|------------|----------|
| Activity Creation | 4 | ✅ |
| Notification Feed | 4 | ✅ |
| User Activity History | 7 | ✅ |
| Read/Unread Tracking | 5 | ✅ |
| Pagination | 3 | ✅ |
| Authorization | 3 | ✅ |
| Self-Action Filtering | 3 | ✅ |
| Performance | 4 | ⚠️ |
| Edge Cases | 5 | ⚠️ |
| **TOTAL** | **38** | **31 Critical** |

---

## ✅ **Test Completion Checklist**

### **Functional Tests** (38/38)
- [x] Activity creation for like, comment, follow
- [x] Notification feed retrieval and pagination
- [x] User activity history aggregation
- [x] Type filters (LIKE, COMMENT, FOLLOW, SAVE)
- [x] Read/unread tracking (single + bulk)
- [x] Cursor-based pagination (ID + timestamp)
- [x] Authorization and feed isolation
- [x] Self-action filtering (no self-notifications)

### **Performance Tests** (4/4)
- [x] Feed load time < 300ms (p95)
- [x] Unread count < 50ms
- [x] History aggregation < 500ms
- [x] N+1 query prevention

### **Edge Cases** (5/5)
- [x] Deleted actors/reels
- [x] Invalid cursors and limits
- [x] Concurrent operations
- [x] Empty feeds
- [x] Malformed requests

---

**[TEST_CASES_COMPLETE ✅]**

*This document provides comprehensive QA test cases for the Activity Module. For feature context, see `01_FEATURE_OVERVIEW.md`. For implementation details, see `02_TECHNICAL_GUIDE.md`.*

---

**Document Version**: 1.0  
**Last Updated**: January 2025  
**Test Coverage**: 38 test cases across 9 categories  
**Automation**: PowerShell scripts provided for all scenarios  
**Next Review**: After mobile UI implementation (Q2 2025)
