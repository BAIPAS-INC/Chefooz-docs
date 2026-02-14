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
```powershell
# 1. Backend running
cd apps/chefooz-apis
npm run start:dev

# 2. PostgreSQL with activity table
# 3. MongoDB with engagements, comments, reels collections
# 4. Test users created
```

### **Test Data Setup**
```powershell
# Create test users
$user1Token = "eyJhbGciOiJIUzI1NiIsInR..." # Chef A
$user2Token = "eyJhbGciOiJIUzI1NiIsInR..." # Chef B

# Create test reel (Chef A)
$reelResponse = Invoke-WebRequest -Uri "http://localhost:3000/api/v1/reels" `
  -Method POST `
  -Headers @{ "Authorization" = "Bearer $user1Token" } `
  -ContentType "application/json" `
  -Body (@{
    caption = "Test Reel for Activity"
    mediaUrls = @(
      @{ url = "https://example.com/video.mp4"; type = "video" }
    )
  } | ConvertTo-Json)

$reelId = ($reelResponse.Content | ConvertFrom-Json).data.id
```

---

## ✅ **Category 1: Activity Creation**

### **Test 1.1: Create Activity on Like**

**Scenario**: User B likes User A's reel → Activity recorded for User A

**Steps**:
```powershell
# Step 1: User B likes User A's reel
$likeResponse = Invoke-WebRequest -Uri "http://localhost:3000/api/v1/engagements/like" `
  -Method POST `
  -Headers @{ "Authorization" = "Bearer $user2Token" } `
  -ContentType "application/json" `
  -Body (@{ reelId = $reelId } | ConvertTo-Json)

# Step 2: User A checks notification feed
$feedResponse = Invoke-WebRequest -Uri "http://localhost:3000/api/v1/activity?limit=1" `
  -Method GET `
  -Headers @{ "Authorization" = "Bearer $user1Token" }

$feed = ($feedResponse.Content | ConvertFrom-Json).data
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
```powershell
$commentResponse = Invoke-WebRequest -Uri "http://localhost:3000/api/v1/comments" `
  -Method POST `
  -Headers @{ "Authorization" = "Bearer $user2Token" } `
  -ContentType "application/json" `
  -Body (@{
    mediaId = $reelId
    text = "Great recipe! 🍕"
  } | ConvertTo-Json)

# Check User A's feed
$feedResponse = Invoke-WebRequest -Uri "http://localhost:3000/api/v1/activity?limit=1" `
  -Method GET `
  -Headers @{ "Authorization" = "Bearer $user1Token" }

$feed = ($feedResponse.Content | ConvertFrom-Json).data
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
```powershell
$followResponse = Invoke-WebRequest -Uri "http://localhost:3000/api/v1/follows" `
  -Method POST `
  -Headers @{ "Authorization" = "Bearer $user2Token" } `
  -ContentType "application/json" `
  -Body (@{ targetUserId = $user1Id } | ConvertTo-Json)

# Check User A's feed
$feedResponse = Invoke-WebRequest -Uri "http://localhost:3000/api/v1/activity?limit=1" `
  -Method GET `
  -Headers @{ "Authorization" = "Bearer $user1Token" }
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
```powershell
# User B likes reel (should succeed even if activity fails)
$likeResponse = Invoke-WebRequest -Uri "http://localhost:3000/api/v1/engagements/like" `
  -Method POST `
  -Headers @{ "Authorization" = "Bearer $user2Token" } `
  -ContentType "application/json" `
  -Body (@{ reelId = $reelId } | ConvertTo-Json)
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
```powershell
# User A checks their notification feed
$feedResponse = Invoke-WebRequest -Uri "http://localhost:3000/api/v1/activity?limit=20" `
  -Method GET `
  -Headers @{ "Authorization" = "Bearer $user1Token" }

$feed = ($feedResponse.Content | ConvertFrom-Json).data
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
```powershell
# Page 1
$page1 = Invoke-WebRequest -Uri "http://localhost:3000/api/v1/activity?limit=5" `
  -Method GET `
  -Headers @{ "Authorization" = "Bearer $user1Token" }

$nextCursor = ($page1.Content | ConvertFrom-Json).data.nextCursor

# Page 2
$page2 = Invoke-WebRequest -Uri "http://localhost:3000/api/v1/activity?limit=5&cursor=$nextCursor" `
  -Method GET `
  -Headers @{ "Authorization" = "Bearer $user1Token" }

$items1 = ($page1.Content | ConvertFrom-Json).data.items
$items2 = ($page2.Content | ConvertFrom-Json).data.items
```

**Validation**:
- ✅ No overlapping items between pages
- ✅ Page 2 items older than Page 1 items
- ✅ Each page has ≤ 5 items

---

### **Test 2.3: Empty Feed**

**Scenario**: New user with no activities

**Steps**:
```powershell
# Create new user with no social interactions
$newUserToken = "..." # New JWT

$feedResponse = Invoke-WebRequest -Uri "http://localhost:3000/api/v1/activity?limit=20" `
  -Method GET `
  -Headers @{ "Authorization" = "Bearer $newUserToken" }
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
```powershell
# 1. User B likes User A's reel (activity created)
# 2. Delete User B's account
# 3. User A checks feed
$feedResponse = Invoke-WebRequest -Uri "http://localhost:3000/api/v1/activity?limit=20" `
  -Method GET `
  -Headers @{ "Authorization" = "Bearer $user1Token" }
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
```powershell
# User A checks their own activity history
$historyResponse = Invoke-WebRequest -Uri "http://localhost:3000/api/v1/activity/me?limit=20" `
  -Method GET `
  -Headers @{ "Authorization" = "Bearer $user1Token" }

$history = ($historyResponse.Content | ConvertFrom-Json).data
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
```powershell
$historyResponse = Invoke-WebRequest -Uri "http://localhost:3000/api/v1/activity/me?type=LIKE&limit=20" `
  -Method GET `
  -Headers @{ "Authorization" = "Bearer $user1Token" }

$history = ($historyResponse.Content | ConvertFrom-Json).data
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
```powershell
$historyResponse = Invoke-WebRequest -Uri "http://localhost:3000/api/v1/activity/me?type=COMMENT&limit=20" `
  -Method GET `
  -Headers @{ "Authorization" = "Bearer $user1Token" }

$history = ($historyResponse.Content | ConvertFrom-Json).data
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
```powershell
$historyResponse = Invoke-WebRequest -Uri "http://localhost:3000/api/v1/activity/me?type=FOLLOW&limit=20" `
  -Method GET `
  -Headers @{ "Authorization" = "Bearer $user1Token" }

$history = ($historyResponse.Content | ConvertFrom-Json).data
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
```powershell
$historyResponse = Invoke-WebRequest -Uri "http://localhost:3000/api/v1/activity/me?type=SAVE&limit=20" `
  -Method GET `
  -Headers @{ "Authorization" = "Bearer $user1Token" }

$history = ($historyResponse.Content | ConvertFrom-Json).data
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
```powershell
# Page 1
$page1 = Invoke-WebRequest -Uri "http://localhost:3000/api/v1/activity/me?limit=10" `
  -Method GET `
  -Headers @{ "Authorization" = "Bearer $user1Token" }

$nextCursor = ($page1.Content | ConvertFrom-Json).data.nextCursor

# Page 2
$page2 = Invoke-WebRequest -Uri "http://localhost:3000/api/v1/activity/me?cursor=$nextCursor&limit=10" `
  -Method GET `
  -Headers @{ "Authorization" = "Bearer $user1Token" }

$items1 = ($page1.Content | ConvertFrom-Json).data.items
$items2 = ($page2.Content | ConvertFrom-Json).data.items
```

**Validation**:
- ✅ No duplicate items
- ✅ `nextCursor` is ISO timestamp string
- ✅ Page 2 items older than Page 1 last item

---

### **Test 3.7: History with Deleted Reels**

**Scenario**: User liked a reel that was later deleted

**Steps**:
```powershell
# 1. User A likes reel (recorded in engagements)
# 2. Reel owner deletes reel (soft-delete)
# 3. User A checks history
$historyResponse = Invoke-WebRequest -Uri "http://localhost:3000/api/v1/activity/me?type=LIKE&limit=20" `
  -Method GET `
  -Headers @{ "Authorization" = "Bearer $user1Token" }
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
```powershell
$countResponse = Invoke-WebRequest -Uri "http://localhost:3000/api/v1/activity/unread-count" `
  -Method GET `
  -Headers @{ "Authorization" = "Bearer $user1Token" }

$count = ($countResponse.Content | ConvertFrom-Json).data.count
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
```powershell
# Get activity ID from feed
$feedResponse = Invoke-WebRequest -Uri "http://localhost:3000/api/v1/activity?limit=1" `
  -Method GET `
  -Headers @{ "Authorization" = "Bearer $user1Token" }

$activityId = ($feedResponse.Content | ConvertFrom-Json).data.items[0].id

# Mark as read
$markResponse = Invoke-WebRequest -Uri "http://localhost:3000/api/v1/activity/read/$activityId" `
  -Method POST `
  -Headers @{ "Authorization" = "Bearer $user1Token" }

# Verify updated
$verifyResponse = Invoke-WebRequest -Uri "http://localhost:3000/api/v1/activity?limit=1" `
  -Method GET `
  -Headers @{ "Authorization" = "Bearer $user1Token" }

$isRead = ($verifyResponse.Content | ConvertFrom-Json).data.items[0].isRead
```

**Expected**:
- ✅ Mark response: `{ success: true }`
- ✅ Verify response: `isRead: true`

---

### **Test 4.3: Mark All as Read**

**Scenario**: Bulk update all unread notifications

**Steps**:
```powershell
# Check initial unread count
$countBefore = (Invoke-WebRequest -Uri "http://localhost:3000/api/v1/activity/unread-count" `
  -Headers @{ "Authorization" = "Bearer $user1Token" }).Content | ConvertFrom-Json

# Mark all as read
$markResponse = Invoke-WebRequest -Uri "http://localhost:3000/api/v1/activity/read-all" `
  -Method POST `
  -Headers @{ "Authorization" = "Bearer $user1Token" }

$markedCount = ($markResponse.Content | ConvertFrom-Json).data.count

# Verify unread count is now 0
$countAfter = (Invoke-WebRequest -Uri "http://localhost:3000/api/v1/activity/unread-count" `
  -Headers @{ "Authorization" = "Bearer $user1Token" }).Content | ConvertFrom-Json
```

**Expected**:
- ✅ `markedCount` = initial unread count
- ✅ `countAfter.data.count` = 0

---

### **Test 4.4: Mark Read (Unauthorized)**

**Scenario**: User tries to mark another user's activity

**Steps**:
```powershell
# Get User A's activity ID
$feedA = Invoke-WebRequest -Uri "http://localhost:3000/api/v1/activity?limit=1" `
  -Method GET `
  -Headers @{ "Authorization" = "Bearer $user1Token" }

$activityIdA = ($feedA.Content | ConvertFrom-Json).data.items[0].id

# User B tries to mark User A's activity
$markResponse = Invoke-WebRequest -Uri "http://localhost:3000/api/v1/activity/read/$activityIdA" `
  -Method POST `
  -Headers @{ "Authorization" = "Bearer $user2Token" } `
  -SkipHttpErrorCheck
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
```powershell
# Mark activity as read
$activityId = "..."
Invoke-WebRequest -Uri "http://localhost:3000/api/v1/activity/read/$activityId" `
  -Method POST `
  -Headers @{ "Authorization" = "Bearer $user1Token" }

# Wait 5 seconds
Start-Sleep -Seconds 5

# Fetch feed again (new request)
$feedResponse = Invoke-WebRequest -Uri "http://localhost:3000/api/v1/activity?limit=20" `
  -Method GET `
  -Headers @{ "Authorization" = "Bearer $user1Token" }

$activity = ($feedResponse.Content | ConvertFrom-Json).data.items | Where-Object { $_.id -eq $activityId }
```

**Expected**:
- ✅ `activity.isRead` is still `true`

---

## 📄 **Category 5: Pagination**

### **Test 5.1: Notification Feed Pagination (50 Items)**

**Scenario**: Paginate through large feed with max limit

**Steps**:
```powershell
$allItems = @()
$cursor = $null

do {
  $url = "http://localhost:3000/api/v1/activity?limit=50"
  if ($cursor) { $url += "&cursor=$cursor" }
  
  $response = Invoke-WebRequest -Uri $url `
    -Method GET `
    -Headers @{ "Authorization" = "Bearer $user1Token" }
  
  $data = ($response.Content | ConvertFrom-Json).data
  $allItems += $data.items
  $cursor = $data.nextCursor
} while ($cursor)

Write-Host "Total items fetched: $($allItems.Count)"
```

**Validation**:
- ✅ All items unique (no duplicates)
- ✅ Items sorted chronologically
- ✅ Pagination completes without errors

---

### **Test 5.2: User History Pagination (Timestamp Cursor)**

**Scenario**: Paginate through user history using timestamp

**Steps**:
```powershell
$page1 = Invoke-WebRequest -Uri "http://localhost:3000/api/v1/activity/me?limit=5" `
  -Method GET `
  -Headers @{ "Authorization" = "Bearer $user1Token" }

$cursor = ($page1.Content | ConvertFrom-Json).data.nextCursor

$page2 = Invoke-WebRequest -Uri "http://localhost:3000/api/v1/activity/me?cursor=$cursor&limit=5" `
  -Method GET `
  -Headers @{ "Authorization" = "Bearer $user1Token" }

$items1 = ($page1.Content | ConvertFrom-Json).data.items
$items2 = ($page2.Content | ConvertFrom-Json).data.items
```

**Validation**:
- ✅ `cursor` is valid ISO timestamp
- ✅ Page 2 items have `createdAt < cursor`
- ✅ No overlap between pages

---

### **Test 5.3: Limit Enforcement (Max 50)**

**Scenario**: Client requests limit > 50, backend enforces max

**Steps**:
```powershell
$response = Invoke-WebRequest -Uri "http://localhost:3000/api/v1/activity?limit=100" `
  -Method GET `
  -Headers @{ "Authorization" = "Bearer $user1Token" }

$items = ($response.Content | ConvertFrom-Json).data.items
```

**Expected**:
- ✅ Max 50 items returned (not 100)
- ✅ `items.Count` ≤ 50

---

## 🔒 **Category 6: Authorization**

### **Test 6.1: Feed Requires JWT**

**Scenario**: Unauthorized access denied

**Steps**:
```powershell
$response = Invoke-WebRequest -Uri "http://localhost:3000/api/v1/activity" `
  -Method GET `
  -SkipHttpErrorCheck
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
```powershell
$response = Invoke-WebRequest -Uri "http://localhost:3000/api/v1/activity/me" `
  -Method GET `
  -SkipHttpErrorCheck
```

**Expected**:
- ✅ HTTP 401 Unauthorized

---

### **Test 6.3: Feed Isolation (User A vs User B)**

**Scenario**: User A cannot see User B's feed

**Steps**:
```powershell
# User A's feed
$feedA = Invoke-WebRequest -Uri "http://localhost:3000/api/v1/activity?limit=20" `
  -Method GET `
  -Headers @{ "Authorization" = "Bearer $user1Token" }

# User B's feed
$feedB = Invoke-WebRequest -Uri "http://localhost:3000/api/v1/activity?limit=20" `
  -Method GET `
  -Headers @{ "Authorization" = "Bearer $user2Token" }

$itemsA = ($feedA.Content | ConvertFrom-Json).data.items
$itemsB = ($feedB.Content | ConvertFrom-Json).data.items
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
```powershell
# User A creates reel
$reelResponse = Invoke-WebRequest -Uri "http://localhost:3000/api/v1/reels" `
  -Method POST `
  -Headers @{ "Authorization" = "Bearer $user1Token" } `
  -Body (@{ caption = "My Reel" } | ConvertTo-Json)

$reelId = ($reelResponse.Content | ConvertFrom-Json).data.id

# User A likes their own reel
Invoke-WebRequest -Uri "http://localhost:3000/api/v1/engagements/like" `
  -Method POST `
  -Headers @{ "Authorization" = "Bearer $user1Token" } `
  -Body (@{ reelId = $reelId } | ConvertTo-Json)

# Check User A's feed
$feedResponse = Invoke-WebRequest -Uri "http://localhost:3000/api/v1/activity?limit=20" `
  -Method GET `
  -Headers @{ "Authorization" = "Bearer $user1Token" }

$selfLikeActivity = ($feedResponse.Content | ConvertFrom-Json).data.items | 
  Where-Object { $_.type -eq "like" -and $_.entityId -eq $reelId }
```

**Expected**:
- ✅ `$selfLikeActivity` is empty (no activity created)

---

### **Test 7.2: Comment on Own Reel (No Activity)**

**Steps**:
```powershell
# User A comments on their own reel
Invoke-WebRequest -Uri "http://localhost:3000/api/v1/comments" `
  -Method POST `
  -Headers @{ "Authorization" = "Bearer $user1Token" } `
  -Body (@{ mediaId = $reelId; text = "Self comment" } | ConvertTo-Json)

# Check feed
$feedResponse = Invoke-WebRequest -Uri "http://localhost:3000/api/v1/activity?limit=20" `
  -Method GET `
  -Headers @{ "Authorization" = "Bearer $user1Token" }

$selfCommentActivity = ($feedResponse.Content | ConvertFrom-Json).data.items | 
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
```powershell
$times = @()

for ($i = 0; $i -lt 100; $i++) {
  $start = Get-Date
  
  Invoke-WebRequest -Uri "http://localhost:3000/api/v1/activity?limit=20" `
    -Method GET `
    -Headers @{ "Authorization" = "Bearer $user1Token" } | Out-Null
  
  $end = Get-Date
  $times += ($end - $start).TotalMilliseconds
}

$p95 = ($times | Sort-Object)[[Math]::Floor($times.Count * 0.95)]
Write-Host "P95 response time: $p95 ms"
```

**Expected**:
- ✅ P95 < 300ms

---

### **Test 8.2: Unread Count Query Time (< 50ms)**

**Steps**:
```powershell
$times = @()

for ($i = 0; $i -lt 50; $i++) {
  $start = Get-Date
  
  Invoke-WebRequest -Uri "http://localhost:3000/api/v1/activity/unread-count" `
    -Method GET `
    -Headers @{ "Authorization" = "Bearer $user1Token" } | Out-Null
  
  $end = Get-Date
  $times += ($end - $start).TotalMilliseconds
}

$avg = ($times | Measure-Object -Average).Average
Write-Host "Average response time: $avg ms"
```

**Expected**:
- ✅ Average < 50ms

---

### **Test 8.3: History Aggregation Time (< 500ms)**

**Scenario**: Multi-source fetch performance

**Steps**:
```powershell
$start = Get-Date

Invoke-WebRequest -Uri "http://localhost:3000/api/v1/activity/me?limit=50" `
  -Method GET `
  -Headers @{ "Authorization" = "Bearer $user1Token" } | Out-Null

$end = Get-Date
$duration = ($end - $start).TotalMilliseconds

Write-Host "User history fetch time: $duration ms"
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
```powershell
# Manually create activity with null metadata (via SQL)
# Then fetch feed
$feedResponse = Invoke-WebRequest -Uri "http://localhost:3000/api/v1/activity?limit=20" `
  -Method GET `
  -Headers @{ "Authorization" = "Bearer $user1Token" }
```

**Expected**:
- ✅ Activity returned with `metadata: {}`
- ✅ No errors thrown

---

### **Test 9.2: Cursor with Invalid Timestamp**

**Scenario**: Malformed cursor in user history

**Steps**:
```powershell
$response = Invoke-WebRequest -Uri "http://localhost:3000/api/v1/activity/me?cursor=invalid-date&limit=20" `
  -Method GET `
  -Headers @{ "Authorization" = "Bearer $user1Token" } `
  -SkipHttpErrorCheck
```

**Expected**:
- ✅ HTTP 400 Bad Request
- ✅ Error message: "Invalid cursor format"

---

### **Test 9.3: Limit = 0**

**Steps**:
```powershell
$response = Invoke-WebRequest -Uri "http://localhost:3000/api/v1/activity?limit=0" `
  -Method GET `
  -Headers @{ "Authorization" = "Bearer $user1Token" } `
  -SkipHttpErrorCheck
```

**Expected**:
- ✅ HTTP 400 Bad Request
- ✅ Validation error: "limit must be at least 1"

---

### **Test 9.4: Invalid Activity Type Filter**

**Steps**:
```powershell
$response = Invoke-WebRequest -Uri "http://localhost:3000/api/v1/activity/me?type=INVALID&limit=20" `
  -Method GET `
  -Headers @{ "Authorization" = "Bearer $user1Token" } `
  -SkipHttpErrorCheck
```

**Expected**:
- ✅ HTTP 400 Bad Request
- ✅ Error: "type must be one of: LIKE, COMMENT, FOLLOW, SAVE"

---

### **Test 9.5: Concurrent Read Operations**

**Scenario**: Two devices mark same activity as read simultaneously

**Steps**:
```powershell
$activityId = "..."

# Simulate concurrent requests
$job1 = Start-Job -ScriptBlock {
  Invoke-WebRequest -Uri "http://localhost:3000/api/v1/activity/read/$using:activityId" `
    -Method POST `
    -Headers @{ "Authorization" = "Bearer $using:user1Token" }
}

$job2 = Start-Job -ScriptBlock {
  Invoke-WebRequest -Uri "http://localhost:3000/api/v1/activity/read/$using:activityId" `
    -Method POST `
    -Headers @{ "Authorization" = "Bearer $using:user1Token" }
}

Wait-Job $job1, $job2
```

**Expected**:
- ✅ Both requests succeed (idempotent)
- ✅ Activity `isRead: true` in database

---

## 🛠️ **PowerShell Test Helpers**

### **Helper 1: Create Test Activity**

```powershell
function Create-TestActivity {
  param(
    [string]$actorToken,
    [string]$targetReelId,
    [string]$action = "like"  # like, comment, follow
  )
  
  switch ($action) {
    "like" {
      Invoke-WebRequest -Uri "http://localhost:3000/api/v1/engagements/like" `
        -Method POST `
        -Headers @{ "Authorization" = "Bearer $actorToken" } `
        -Body (@{ reelId = $targetReelId } | ConvertTo-Json)
    }
    "comment" {
      Invoke-WebRequest -Uri "http://localhost:3000/api/v1/comments" `
        -Method POST `
        -Headers @{ "Authorization" = "Bearer $actorToken" } `
        -Body (@{ mediaId = $targetReelId; text = "Test comment" } | ConvertTo-Json)
    }
  }
}

# Usage
Create-TestActivity -actorToken $user2Token -targetReelId $reelId -action "like"
```

---

### **Helper 2: Get Unread Count**

```powershell
function Get-UnreadCount {
  param([string]$token)
  
  $response = Invoke-WebRequest -Uri "http://localhost:3000/api/v1/activity/unread-count" `
    -Method GET `
    -Headers @{ "Authorization" = "Bearer $token" }
  
  return ($response.Content | ConvertFrom-Json).data.count
}

# Usage
$count = Get-UnreadCount -token $user1Token
Write-Host "Unread notifications: $count"
```

---

### **Helper 3: Mark All Read**

```powershell
function Clear-Notifications {
  param([string]$token)
  
  $response = Invoke-WebRequest -Uri "http://localhost:3000/api/v1/activity/read-all" `
    -Method POST `
    -Headers @{ "Authorization" = "Bearer $token" }
  
  $count = ($response.Content | ConvertFrom-Json).data.count
  Write-Host "Marked $count activities as read"
}

# Usage
Clear-Notifications -token $user1Token
```

---

### **Helper 4: Fetch Full History**

```powershell
function Get-FullHistory {
  param(
    [string]$token,
    [string]$type = $null
  )
  
  $allItems = @()
  $cursor = $null
  
  do {
    $url = "http://localhost:3000/api/v1/activity/me?limit=50"
    if ($type) { $url += "&type=$type" }
    if ($cursor) { $url += "&cursor=$cursor" }
    
    $response = Invoke-WebRequest -Uri $url `
      -Method GET `
      -Headers @{ "Authorization" = "Bearer $token" }
    
    $data = ($response.Content | ConvertFrom-Json).data
    $allItems += $data.items
    $cursor = $data.nextCursor
  } while ($cursor)
  
  return $allItems
}

# Usage
$allLikes = Get-FullHistory -token $user1Token -type "LIKE"
Write-Host "Total likes: $($allLikes.Count)"
```

---

### **Helper 5: Performance Test**

```powershell
function Test-EndpointPerformance {
  param(
    [string]$url,
    [string]$token,
    [int]$iterations = 50
  )
  
  $times = @()
  
  for ($i = 0; $i -lt $iterations; $i++) {
    $start = Get-Date
    Invoke-WebRequest -Uri $url -Headers @{ "Authorization" = "Bearer $token" } | Out-Null
    $end = Get-Date
    $times += ($end - $start).TotalMilliseconds
  }
  
  $sorted = $times | Sort-Object
  $p50 = $sorted[[Math]::Floor($times.Count * 0.50)]
  $p95 = $sorted[[Math]::Floor($times.Count * 0.95)]
  $p99 = $sorted[[Math]::Floor($times.Count * 0.99)]
  
  Write-Host "P50: $p50 ms | P95: $p95 ms | P99: $p99 ms"
}

# Usage
Test-EndpointPerformance -url "http://localhost:3000/api/v1/activity" -token $user1Token
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
