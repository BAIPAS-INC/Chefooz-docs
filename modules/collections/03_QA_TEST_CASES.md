# Collections Module - QA Test Cases

**Version:** 1.0  
**Last Updated:** February 14, 2026  
**Module:** Collections  
**Test Coverage**: API, Business Logic, Security, Performance

---

## 📋 Table of Contents

1. [Test Environment Setup](#test-environment-setup)
2. [Save/Unsave Tests](#saveunsave-tests)
3. [Saved Reels Tests](#saved-reels-tests)
4. [Collection Management Tests](#collection-management-tests)
5. [Collection Items Tests](#collection-items-tests)
6. [Authorization Tests](#authorization-tests)
7. [Pagination Tests](#pagination-tests)
8. [Performance Tests](#performance-tests)
9. [Edge Cases](#edge-cases)

---

## Test Environment Setup

### Prerequisites

```bash
# 1. Start test databases
docker-compose -f docker-compose.test.yml up -d

# 2. Run migrations
npm run migration:run

# 3. Seed test data
npm run seed:test

# 4. Start test server
npm run test:e2e
```

### Test Data

```typescript
// Test users
const users = {
  user1: {
    id: '550e8400-e29b-41d4-a716-446655440001',
    username: 'john_doe',
    token: '<JWT_TOKEN_USER1>',
  },
  user2: {
    id: '550e8400-e29b-41d4-a716-446655440002',
    username: 'jane_smith',
    token: '<JWT_TOKEN_USER2>',
  },
};

// Test reels (MongoDB)
const testReels = {
  reel1: {
    mediaId: '691ca197b8be29951a3caf07',
    userId: users.user2.id,
    stats: { savedCount: 0 },
  },
  reel2: {
    mediaId: '691ca197b8be29951a3caf08',
    userId: users.user2.id,
    stats: { savedCount: 0 },
  },
};
```

### PowerShell Test Helper

```bash

# test-collections.ps1
$baseUrl = "https://api-staging.chefooz.com/api/v1"
$token = "<JWT_TOKEN>"

function Test-ToggleSave {
    param($MediaId)
    
curl -s \
  -X POST \
  "$baseUrl/collections/save" \
  -H "Content-Type: application/json"
}

function Test-CreateCollection {
    param($Name, $Emoji = $null)
    
    $body = @{ name = $Name }
    if ($Emoji) { $body.emoji = $Emoji }
    
curl -s \
  -X POST \
  "$baseUrl/collections/create" \
  -H "Content-Type: application/json"
}
```

---

## Save/Unsave Tests

### TC-SAV-001: Save Reel (First Time)

**Objective**: Verify save creates record and increments counter

**Steps**:
```bash

RESPONSE=$(curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/collections/save" \
  -H "Content-Type: application/json")
```

**Expected Result**:
- Status: 200 OK
- Response:
  ```json
  {
    "success": true,
    "message": "Reel saved",
    "data": {
      "saved": true,
      "totalSavedCount": 1
    }
  }
  ```
- Database: `saved_reels` table has new record
- MongoDB: `reel.stats.savedCount = 1`

**Priority**: P0 (Critical)

---

### TC-SAV-002: Unsave Reel (Toggle)

**Objective**: Verify unsave removes record and decrements counter

**Steps**:
```bash

# 1. Save reel first
Test-ToggleSave -MediaId "691ca197b8be29951a3caf07"

# 2. Unsave (toggle again)
RESPONSE=$(curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/collections/save" \
  -H "Content-Type: application/json")
```

**Expected Result**:
- Status: 200 OK
- Response:
  ```json
  {
    "success": true,
    "message": "Reel unsaved",
    "data": {
      "saved": false,
      "totalSavedCount": 0
    }
  }
  ```
- Database: `saved_reels` record deleted
- MongoDB: `reel.stats.savedCount = 0`

**Priority**: P0

---

### TC-SAV-003: Save Non-Existent Reel

**Objective**: Handle invalid reel ID

**Steps**:
```bash

RESPONSE=$(curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/collections/save" \
  -H "Content-Type: application/json")
```

**Expected Result**:
- Status: 404 Not Found
- Response:
  ```json
  {
    "success": false,
    "message": "Reel not found",
    "errorCode": "REEL_NOT_FOUND"
  }
  ```

**Priority**: P1

---

### TC-SAV-004: Multiple Users Save Same Reel

**Objective**: Verify independent save tracking per user

**Steps**:
```bash

# 1. User1 saves
curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/collections/save" \
  -H "Content-Type: application/json"

# 2. User2 saves same reel
RESPONSE=$(curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/collections/save" \
  -H "Content-Type: application/json")
```

**Expected Result**:
- User1 save: `totalSavedCount = 1`
- User2 save: `totalSavedCount = 2`
- Database: 2 records in `saved_reels` (different user_id)
- MongoDB: `reel.stats.savedCount = 2`

**Priority**: P0

---

### TC-SAV-005: Save Without Authentication

**Objective**: Require JWT token

**Steps**:
```bash

RESPONSE=$(curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/collections/save" \
  -H "Content-Type: application/json")
```

**Expected Result**:
- Status: 401 Unauthorized
- Response:
  ```json
  {
    "success": false,
    "message": "Unauthorized",
    "errorCode": "AUTH_REQUIRED"
  }
  ```

**Priority**: P0

---

## Saved Reels Tests

### TC-SAR-001: Get Saved Reels (Pagination)

**Objective**: List user's saved reels with pagination

**Steps**:
```bash

# 1. Save 3 reels
Test-ToggleSave -MediaId "691ca197b8be29951a3caf07"
Start-Sleep -Seconds 1
Test-ToggleSave -MediaId "691ca197b8be29951a3caf08"
Start-Sleep -Seconds 1
Test-ToggleSave -MediaId "691ca197b8be29951a3caf09"

# 2. Get saved reels
RESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/collections/saved?limit=20")
```

**Expected Result**:
- Status: 200 OK
- Response:
  ```json
  {
    "success": true,
    "message": "Saved reels retrieved",
    "data": {
      "items": [
        {
          "mediaId": "691ca197b8be29951a3caf09",
          "savedAt": "2026-02-14T10:30:02Z",
          "collectionsCount": 0
        },
        {
          "mediaId": "691ca197b8be29951a3caf08",
          "savedAt": "2026-02-14T10:30:01Z",
          "collectionsCount": 0
        },
        {
          "mediaId": "691ca197b8be29951a3caf07",
          "savedAt": "2026-02-14T10:30:00Z",
          "collectionsCount": 0
        }
      ],
      "nextCursor": null
    }
  }
  ```
- Order: Newest first (DESC)

**Priority**: P0

---

### TC-SAR-002: Get Saved Count for Reel

**Objective**: Count how many users saved a reel

**Steps**:
```bash

# 1. Multiple users save
# User1 saves
curl -s \
  -X POST \
  "$baseUrl/collections/save" \
  -H "Content-Type: application/json"

# User2 saves
curl -s \
  -X POST \
  "$baseUrl/collections/save" \
  -H "Content-Type: application/json"

# 2. Get saved count
RESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/collections/saved/count/691ca197b8be29951a3caf07")
```

**Expected Result**:
- Status: 200 OK
- Response:
  ```json
  {
    "success": true,
    "message": "Saved count retrieved",
    "data": {
      "count": 2
    }
  }
  ```

**Priority**: P1

---

### TC-SAR-003: Check Save Status

**Objective**: Check if user saved specific reel

**Steps**:
```bash

# 1. Save reel
Test-ToggleSave -MediaId "691ca197b8be29951a3caf07"

# 2. Check status
RESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/collections/saved/status/691ca197b8be29951a3caf07")
```

**Expected Result**:
- Status: 200 OK
- Response:
  ```json
  {
    "success": true,
    "message": "Save status retrieved",
    "data": {
      "saved": true
    }
  }
  ```

**Priority**: P1

---

## Collection Management Tests

### TC-COL-001: Create Collection

**Objective**: Create new collection with name and emoji

**Steps**:
```bash

RESPONSE=$(curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/collections/create")
        name = "Italian Cuisine"
        emoji = "🍝"
    } | ConvertTo-Json)  -ContentType "application/json"
```

**Expected Result**:
- Status: 201 Created
- Response:
  ```json
  {
    "success": true,
    "message": "Collection created",
    "data": {
      "collection": {
        "id": "<UUID>",
        "userId": "550e8400-e29b-41d4-a716-446655440001",
        "name": "Italian Cuisine",
        "emoji": "🍝",
        "itemCount": 0,
        "createdAt": "2026-02-14T10:35:00.000Z",
        "updatedAt": "2026-02-14T10:35:00.000Z"
      }
    }
  }
  ```

**Priority**: P0

---

### TC-COL-002: Create Collection Without Emoji

**Objective**: Emoji is optional

**Steps**:
```bash

RESPONSE=$(curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/collections/create" \
  -H "Content-Type: application/json")
```

**Expected Result**:
- Status: 201 Created
- Response includes `"emoji": null`

**Priority**: P1

---

### TC-COL-003: Enforce Collection Limit (200 Max)

**Objective**: Prevent creating more than 200 collections

**Steps**:
```bash

# 1. Create 200 collections (loop)
1..200 | ForEach-Object {
    Test-CreateCollection -Name "Collection $_"
}

# 2. Try to create 201st collection
RESPONSE=$(curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/collections/create" \
  -H "Content-Type: application/json")
```

**Expected Result**:
- Status: 400 Bad Request
- Response:
  ```json
  {
    "success": false,
    "message": "Maximum 200 collections allowed",
    "errorCode": "COLLECTION_LIMIT_REACHED"
  }
  ```

**Priority**: P1

---

### TC-COL-004: Collection Name Length Validation

**Objective**: Enforce 1-60 character limit

**Steps**:
```bash

# Test empty name
RESPONSE1=$(curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/collections/create" \
  -H "Content-Type: application/json")

# Test long name (61 chars)
$longName = "a" * 61
RESPONSE2=$(curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/collections/create" \
  -H "Content-Type: application/json")
```

**Expected Result**:
- Empty name: 400 Bad Request (validation error)
- Long name: 400 Bad Request (MaxLength violation)

**Priority**: P1

---

### TC-COL-005: Get My Collections

**Objective**: List all user's collections

**Steps**:
```bash

# 1. Create 3 collections
Test-CreateCollection -Name "Favorites" -Emoji "⭐"
Test-CreateCollection -Name "Quick Meals" -Emoji "⚡"
Test-CreateCollection -Name "Desserts" -Emoji "🍰"

# 2. Get collections
RESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/collections/my")
```

**Expected Result**:
- Status: 200 OK
- Response includes 3 collections
- Ordered by `createdAt DESC` (newest first)

**Priority**: P0

---

### TC-COL-006: Rename Collection

**Objective**: Update collection name and emoji

**Steps**:
```bash

# 1. Create collection
$create = Test-CreateCollection -Name "Old Name" -Emoji "🔖"
$collectionId = ($create.Content | jq .).data.collection.id

# 2. Rename
RESPONSE=$(curl -s \
  -X PATCH \
  "https://api-staging.chefooz.com/api/v1/collections/$collectionId/rename")
        name = "New Name"
        emoji = "📌"
    } | ConvertTo-Json)  -ContentType "application/json"
```

**Expected Result**:
- Status: 200 OK
- Response:
  ```json
  {
    "success": true,
    "message": "Collection renamed",
    "data": {
      "collection": {
        "id": "<UUID>",
        "name": "New Name",
        "emoji": "📌",
        "itemCount": 0,
        "updatedAt": "2026-02-14T10:40:00.000Z"
      }
    }
  }
  ```

**Priority**: P0

---

### TC-COL-007: Delete Collection

**Objective**: Remove collection and cascade delete items

**Steps**:
```bash

# 1. Create collection
$create = Test-CreateCollection -Name "Temporary"
$collectionId = ($create.Content | jq .).data.collection.id

# 2. Add 2 reels to collection
curl -s \
  -X POST \
  "$baseUrl/collections/$collectionId/add" \
  -H "Content-Type: application/json"
curl -s \
  -X POST \
  "$baseUrl/collections/$collectionId/add" \
  -H "Content-Type: application/json"

# 3. Delete collection
RESPONSE=$(curl -s \
  -X DELETE \
  "https://api-staging.chefooz.com/api/v1/collections/$collectionId")
```

**Expected Result**:
- Status: 200 OK
- Response:
  ```json
  {
    "success": true,
    "message": "Collection deleted",
    "data": null
  }
  ```
- Database: `collection_items` records deleted (cascade)
- Saved status of reels unchanged

**Priority**: P0

---

## Collection Items Tests

### TC-ITM-001: Add Reel to Collection

**Objective**: Add reel to collection

**Steps**:
```bash

# 1. Create collection
$create = Test-CreateCollection -Name "Favorites"
$collectionId = ($create.Content | jq .).data.collection.id

# 2. Save reel
Test-ToggleSave -MediaId "691ca197b8be29951a3caf07"

# 3. Add to collection
RESPONSE=$(curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/collections/$collectionId/add" \
  -H "Content-Type: application/json")
```

**Expected Result**:
- Status: 200 OK
- Response:
  ```json
  {
    "success": true,
    "message": "Added to collection",
    "data": null
  }
  ```
- Database: `collection_items` has new record

**Priority**: P0

---

### TC-ITM-002: Add Reel to Multiple Collections

**Objective**: Reel can exist in multiple collections

**Steps**:
```bash

# 1. Create 2 collections
$coll1 = Test-CreateCollection -Name "Favorites"
$coll2 = Test-CreateCollection -Name "Italian"
$collectionId1 = ($coll1.Content | jq .).data.collection.id
$collectionId2 = ($coll2.Content | jq .).data.collection.id

# 2. Save reel
Test-ToggleSave -MediaId "691ca197b8be29951a3caf07"

# 3. Add to both collections
curl -s \
  -X POST \
  "$baseUrl/collections/$collectionId1/add" \
  -H "Content-Type: application/json"

curl -s \
  -X POST \
  "$baseUrl/collections/$collectionId2/add" \
  -H "Content-Type: application/json"

# 4. Get collections for reel
RESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/collections/reel/691ca197b8be29951a3caf07/collections")
```

**Expected Result**:
- Both adds succeed
- Response shows 2 collections

**Priority**: P0

---

### TC-ITM-003: Idempotent Add (Duplicate Prevention)

**Objective**: Adding same reel twice is idempotent

**Steps**:
```bash

# 1. Create collection
$create = Test-CreateCollection -Name "Favorites"
$collectionId = ($create.Content | jq .).data.collection.id

# 2. Add reel twice
curl -s \
  -X POST \
  "$baseUrl/collections/$collectionId/add" \
  -H "Content-Type: application/json"

RESPONSE=$(curl -s \
  -X POST \
  "$baseUrl/collections/$collectionId/add" \
  -H "Content-Type: application/json")
```

**Expected Result**:
- Both requests: 200 OK
- Database: Only 1 record in `collection_items`
- No error thrown (idempotent)

**Priority**: P0

---

### TC-ITM-004: Remove Reel from Collection

**Objective**: Remove reel from specific collection

**Steps**:
```bash

# 1. Add reel to collection
# (setup code)

# 2. Remove from collection
RESPONSE=$(curl -s \
  -X DELETE \
  "https://api-staging.chefooz.com/api/v1/collections/$collectionId/remove/691ca197b8be29951a3caf07")
```

**Expected Result**:
- Status: 200 OK
- Database: `collection_items` record deleted
- Reel still saved (not unsaved)

**Priority**: P0

---

### TC-ITM-005: Get Collection Items

**Objective**: List reels in collection

**Steps**:
```bash

# 1. Create collection
$create = Test-CreateCollection -Name "Favorites"
$collectionId = ($create.Content | jq .).data.collection.id

# 2. Add 3 reels
curl -s \
  -X POST \
  "$baseUrl/collections/$collectionId/add" \
  -H "Content-Type: application/json"
Start-Sleep -Seconds 1
curl -s \
  -X POST \
  "$baseUrl/collections/$collectionId/add" \
  -H "Content-Type: application/json"
Start-Sleep -Seconds 1
curl -s \
  -X POST \
  "$baseUrl/collections/$collectionId/add" \
  -H "Content-Type: application/json"

# 3. Get items
RESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/collections/$collectionId/items?limit=20")
```

**Expected Result**:
- Status: 200 OK
- Response:
  ```json
  {
    "success": true,
    "message": "Collection items retrieved",
    "data": {
      "mediaIds": [
        "691ca197b8be29951a3caf09",
        "691ca197b8be29951a3caf08",
        "691ca197b8be29951a3caf07"
      ],
      "nextCursor": null
    }
  }
  ```
- Order: Newest added first (DESC)

**Priority**: P0

---

### TC-ITM-006: Get Collections Containing Reel

**Objective**: Show which collections contain specific reel

**Steps**:
```bash

# 1. Create 2 collections
$coll1 = Test-CreateCollection -Name "Favorites"
$coll2 = Test-CreateCollection -Name "Italian"
$collectionId1 = ($coll1.Content | jq .).data.collection.id
$collectionId2 = ($coll2.Content | jq .).data.collection.id

# 2. Add reel to both
curl -s \
  -X POST \
  "$baseUrl/collections/$collectionId1/add" \
  -H "Content-Type: application/json"
curl -s \
  -X POST \
  "$baseUrl/collections/$collectionId2/add" \
  -H "Content-Type: application/json"

# 3. Get collections for reel
RESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/collections/reel/691ca197b8be29951a3caf07/collections")
```

**Expected Result**:
- Status: 200 OK
- Response shows both collections

**Priority**: P1

---

## Authorization Tests

### TC-AUTH-001: Cannot Rename Another User's Collection

**Objective**: Ownership check on rename

**Steps**:
```bash

# 1. User1 creates collection
CREATE=$(curl -s \
  -X POST \
  "$baseUrl/collections/create" \
  -H "Content-Type: application/json")
$collectionId = ($create.Content | jq .).data.collection.id

# 2. User2 tries to rename
RESPONSE=$(curl -s \
  -X PATCH \
  "https://api-staging.chefooz.com/api/v1/collections/$collectionId/rename" \
  -H "Content-Type: application/json")
```

**Expected Result**:
- Status: 403 Forbidden
- Response:
  ```json
  {
    "success": false,
    "message": "Cannot modify another user's collection",
    "errorCode": "FORBIDDEN"
  }
  ```

**Priority**: P0

---

### TC-AUTH-002: Cannot Delete Another User's Collection

**Objective**: Ownership check on delete

**Steps**:
```bash

# 1. User1 creates collection
CREATE=$(curl -s \
  -X POST \
  "$baseUrl/collections/create" \
  -H "Content-Type: application/json")
$collectionId = ($create.Content | jq .).data.collection.id

# 2. User2 tries to delete
RESPONSE=$(curl -s \
  -X DELETE \
  "https://api-staging.chefooz.com/api/v1/collections/$collectionId")
```

**Expected Result**:
- Status: 403 Forbidden

**Priority**: P0

---

### TC-AUTH-003: Cannot Add to Another User's Collection

**Objective**: Ownership check on add item

**Steps**:
```bash

# 1. User1 creates collection
CREATE=$(curl -s \
  -X POST \
  "$baseUrl/collections/create" \
  -H "Content-Type: application/json")
$collectionId = ($create.Content | jq .).data.collection.id

# 2. User2 tries to add reel
RESPONSE=$(curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/collections/$collectionId/add" \
  -H "Content-Type: application/json")
```

**Expected Result**:
- Status: 403 Forbidden

**Priority**: P0

---

## Pagination Tests

### TC-PAG-001: Saved Reels Pagination

**Objective**: Cursor-based pagination works correctly

**Steps**:
```bash

# 1. Save 25 reels
1..25 | ForEach-Object {
    $mediaId = "691ca197b8be29951a3caf" + ($_ -as [string]).PadLeft(2, '0')
    Test-ToggleSave -MediaId $mediaId
    Start-Sleep -Milliseconds 100
}

# 2. Fetch page 1
PAGE1=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/collections/saved?limit=20")

# 3. Fetch page 2
$nextCursor = ($page1.Content | jq .).data.nextCursor
PAGE2=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/collections/saved?limit=20&cursor=$nextCursor")
```

**Expected Result**:
- Page 1: 20 items + `nextCursor`
- Page 2: 5 items, no `nextCursor`
- No duplicate items

**Priority**: P0

---

### TC-PAG-002: Collection Items Pagination

**Objective**: Paginate items within collection

**Steps**:
```bash

# 1. Create collection
$create = Test-CreateCollection -Name "Big Collection"
$collectionId = ($create.Content | jq .).data.collection.id

# 2. Add 25 reels
1..25 | ForEach-Object {
    $mediaId = "691ca197b8be29951a3caf" + ($_ -as [string]).PadLeft(2, '0')
curl -s \
  -X POST \
  "$baseUrl/collections/$collectionId/add" \
  -H "Content-Type: application/json"
    Start-Sleep -Milliseconds 100
}

# 3. Fetch page 1
PAGE1=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/collections/$collectionId/items?limit=20")

# 4. Fetch page 2
$nextCursor = ($page1.Content | jq .).data.nextCursor
PAGE2=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/collections/$collectionId/items?limit=20&cursor=$nextCursor")
```

**Expected Result**:
- Page 1: 20 mediaIds
- Page 2: 5 mediaIds
- No duplicates

**Priority**: P1

---

## Performance Tests

### TC-PERF-001: Query Performance with Large Dataset

**Objective**: Verify acceptable response times

**Steps**:
```bash

# 1. Create 100 collections for user
1..100 | ForEach-Object {
    Test-CreateCollection -Name "Collection $_"
}

# 2. Save 500 reels
1..500 | ForEach-Object {
    $mediaId = "691ca197b8be29951a3c" + ($_ -as [string]).PadLeft(4, '0')
    Test-ToggleSave -MediaId $mediaId
}

# 3. Measure query time
$start = Get-Date
RESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/collections/my")
$duration = (Get-Date) - $start

echo "Query time: $($duration.TotalMilliseconds)ms"
```

**Expected Result**:
- Response time: < 200ms
- Uses index: `idx_collections_user_id`

**Priority**: P1

---

### TC-PERF-002: Concurrent Save Operations

**Objective**: Handle concurrent saves gracefully

**Steps**:
```bash

# 10 users save same reel simultaneously
$jobs = 1..10 | ForEach-Object {
    Start-Job -ScriptBlock {
        param($token, $mediaId)
curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/collections/save" \
  -H "Content-Type: application/json"
    } -ArgumentList $USER_TOKENS[$_], "691ca197b8be29951a3caf07"
}

$jobs | Wait-Job | Receive-Job
```

**Expected Result**:
- All saves succeed
- MongoDB `savedCount = 10` (accurate)
- No race conditions

**Priority**: P1

---

## Edge Cases

### TC-EDGE-001: Delete Collection Preserves Saved Status

**Objective**: Verify cascade delete doesn't unsave reels

**Steps**:
```bash

# 1. Save reel
Test-ToggleSave -MediaId "691ca197b8be29951a3caf07"

# 2. Create collection and add reel
$create = Test-CreateCollection -Name "Temporary"
$collectionId = ($create.Content | jq .).data.collection.id
curl -s \
  -X POST \
  "$baseUrl/collections/$collectionId/add" \
  -H "Content-Type: application/json"

# 3. Delete collection
curl -s \
  -X DELETE \
  "$baseUrl/collections/$collectionId"

# 4. Check save status
RESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/collections/saved/status/691ca197b8be29951a3caf07")
```

**Expected Result**:
- Save status: `saved: true`
- Collection deleted, but reel still saved

**Priority**: P0

---

### TC-EDGE-002: Emoji with Modifiers (Multi-Byte)

**Objective**: Support complex emoji

**Steps**:
```bash

RESPONSE=$(curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/collections/create")
        name = "Chef Collections"
        emoji = "👨‍🍳"  # Chef with skin tone modifier
    } | ConvertTo-Json)  -ContentType "application/json"
```

**Expected Result**:
- Status: 201 Created
- Emoji stored correctly (no truncation)

**Priority**: P2

---

## Test Summary

### Test Coverage

| Category | Total Tests | P0 (Critical) | P1 (High) | P2 (Low) |
|----------|-------------|---------------|-----------|----------|
| Save/Unsave | 5 | 4 | 1 | 0 |
| Saved Reels | 3 | 1 | 2 | 0 |
| Collection Management | 7 | 3 | 4 | 0 |
| Collection Items | 6 | 5 | 1 | 0 |
| Authorization | 3 | 3 | 0 | 0 |
| Pagination | 2 | 1 | 1 | 0 |
| Performance | 2 | 0 | 2 | 0 |
| Edge Cases | 2 | 1 | 0 | 1 |
| **TOTAL** | **30** | **18** | **11** | **1** |

### Test Execution Plan

**Phase 1: Critical (P0)** - 18 tests
- Must pass before QA sign-off
- Covers core save/unsave, authorization, collection CRUD

**Phase 2: High Priority (P1)** - 11 tests
- Must pass before release
- Covers pagination, performance, edge cases

**Phase 3: Low Priority (P2)** - 1 test
- Should pass for production
- Covers emoji edge cases

---

## Related Documentation

- **Feature Overview**: `01_FEATURE_OVERVIEW.md` (Business context)
- **Technical Guide**: `02_TECHNICAL_GUIDE.md` (Implementation details)
- **Backend API**: `apps/chefooz-apis/src/modules/collections/`

---

**[QA_COMPLETE ✅]**  
**Module**: Collections  
**Test Cases**: 30 scenarios across 9 categories  
**Date**: February 14, 2026
