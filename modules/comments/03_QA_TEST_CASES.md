# Comments Module - QA Test Cases

**Version:** 1.0  
**Last Updated:** February 14, 2026  
**Module:** Comments  
**Test Coverage**: API, Business Logic, Security, Performance

---

## 📋 Table of Contents

1. [Test Environment Setup](#test-environment-setup)
2. [Comment Creation Tests](#comment-creation-tests)
3. [Reply System Tests](#reply-system-tests)
4. [User Tagging Tests](#user-tagging-tests)
5. [Like/Unlike Tests](#likeunlike-tests)
6. [Pin/Unpin Tests](#pinunpin-tests)
7. [Comment Deletion Tests](#comment-deletion-tests)
8. [Pagination Tests](#pagination-tests)
9. [Security Tests](#security-tests)
10. [Performance Tests](#performance-tests)
11. [Notification Tests](#notification-tests)

---

## Test Environment Setup

### Prerequisites

```bash
# 1. Start test database
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
    username: 'chef_rakesh',
    token: '<JWT_TOKEN_USER2>',
  },
  user3: {
    id: '550e8400-e29b-41d4-a716-446655440003',
    username: 'jane_smith',
    token: '<JWT_TOKEN_USER3>',
  },
};

// Test reel
const testReel = {
  id: '507f1f77bcf86cd799439011',
  userId: users.user2.id,  // chef_rakesh's reel
  stats: { comments: 0 },
};
```

### PowerShell Test Helper

```bash

# test-comments.ps1
$baseUrl = "https://api-staging.chefooz.com/api/v1"
$token = "<JWT_TOKEN>"

function Test-CreateComment {
    param($MediaId, $Text, $ParentId = $null)
    
    $body = @{
        mediaId = $MediaId
        text = $Text
    }
    if ($ParentId) { $body.parentId = $ParentId }
    
curl -s \
  -X POST \
  "$baseUrl/comments/create" \
  -H "Content-Type: application/json"
}
```

---

## Comment Creation Tests

### TC-COM-001: Create Valid Comment

**Objective**: Verify comment creation with valid data

**Steps**:
```bash

RESPONSE=$(curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/comments/create")
        mediaId = "507f1f77bcf86cd799439011"
        text = "This recipe looks amazing!"
    } | ConvertTo-Json)  -ContentType "application/json"
```

**Expected Result**:
- Status: 201 Created
- Response:
  ```json
  {
    "success": true,
    "message": "Comment posted",
    "data": {
      "id": "<COMMENT_ID>",
      "text": "This recipe looks amazing!",
      "likesCount": 0,
      "isPinned": false
    }
  }
  ```
- Reel `stats.comments` incremented by 1
- Notification sent to reel owner

**Priority**: P0 (Critical)

---

### TC-COM-002: Create Comment with Mentions

**Objective**: Verify @mention parsing

**Steps**:
```bash

RESPONSE=$(curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/comments/create")
        mediaId = "507f1f77bcf86cd799439011"
        text = "Love this! @chef_rakesh @jane_smith"
    } | ConvertTo-Json)  -ContentType "application/json"
```

**Expected Result**:
- Status: 201 Created
- Response includes:
  ```json
  {
    "mentions": ["chef_rakesh", "jane_smith"]
  }
  ```
- Mention notifications sent to both users

**Priority**: P0

---

### TC-COM-003: Create Comment with User Tags

**Objective**: Verify ID-based tagging

**Steps**:
```bash

RESPONSE=$(curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/comments/create")
        mediaId = "507f1f77bcf86cd799439011"
        text = "Check this out!"
        taggedUserIds = @(
            "550e8400-e29b-41d4-a716-446655440002",
            "550e8400-e29b-41d4-a716-446655440003"
        )
    } | ConvertTo-Json)  -ContentType "application/json"
```

**Expected Result**:
- Status: 201 Created
- Mention notifications sent to tagged users
- Max 10 tags enforced (slice to first 10)

**Priority**: P1

---

### TC-COM-004: Empty Comment Text

**Objective**: Reject empty text

**Steps**:
```bash

RESPONSE=$(curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/comments/create")
        mediaId = "507f1f77bcf86cd799439011"
        text = "   "
    } | ConvertTo-Json)  -ContentType "application/json"
```

**Expected Result**:
- Status: 400 Bad Request
- Response:
  ```json
  {
    "success": false,
    "message": "Comment text cannot be empty",
    "errorCode": "INVALID_INPUT"
  }
  ```

**Priority**: P0

---

### TC-COM-005: Comment Text Length Limit

**Objective**: Enforce 500 character limit

**Steps**:
```bash

$longText = "a" * 501  # 501 characters

RESPONSE=$(curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/comments/create")
        mediaId = "507f1f77bcf86cd799439011"
        text = $longText
    } | ConvertTo-Json)  -ContentType "application/json"
```

**Expected Result**:
- Status: 400 Bad Request
- Response includes validation error for `text` field

**Priority**: P1

---

### TC-COM-006: Comment on Non-Existent Reel

**Objective**: Handle invalid reel ID

**Steps**:
```bash

RESPONSE=$(curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/comments/create")
        mediaId = "507f1f77bcf86cd799439099"
        text = "Test comment"
    } | ConvertTo-Json)  -ContentType "application/json"
```

**Expected Result**:
- Status: 404 Not Found
- Response:
  ```json
  {
    "success": false,
    "message": "Reel with ID 507f1f77bcf86cd799439099 not found",
    "errorCode": "REEL_NOT_FOUND"
  }
  ```

**Priority**: P0

---

### TC-COM-007: Comment Without Authentication

**Objective**: Require JWT token

**Steps**:
```bash

RESPONSE=$(curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/comments/create")
        mediaId = "507f1f77bcf86cd799439011"
        text = "Test comment"
    } | ConvertTo-Json)  -ContentType "application/json"
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

## Reply System Tests

### TC-REP-001: Create Valid Reply

**Objective**: Create reply to top-level comment

**Steps**:
```bash

# 1. Create parent comment
$parent = Test-CreateComment -MediaId "507f1f77bcf86cd799439011" -Text "Great recipe!"

# 2. Create reply
RESPONSE=$(curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/comments/$($parent.data.id)/replies")
        mediaId = "507f1f77bcf86cd799439011"
        text = "Thanks!"
    } | ConvertTo-Json)  -ContentType "application/json"
```

**Expected Result**:
- Status: 201 Created
- Reply has `parentId = parent.data.id`
- Parent `replyCount` incremented by 1
- Notification sent to parent comment owner

**Priority**: P0

---

### TC-REP-002: Prevent Nested Replies

**Objective**: Enforce 1-level threading

**Steps**:
```bash

# 1. Create comment
$comment = Test-CreateComment -MediaId "507f1f77bcf86cd799439011" -Text "Top level"

# 2. Create reply
REPLY=$(curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/comments/$($comment.data.id)/replies" \
  -H "Content-Type: application/json")

# 3. Try to reply to reply
RESPONSE=$(curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/comments/$($reply.data.id)/replies" \
  -H "Content-Type: application/json")
```

**Expected Result**:
- Status: 400 Bad Request
- Response:
  ```json
  {
    "success": false,
    "message": "Cannot reply to a reply. Only 1-level threading supported.",
    "errorCode": "INVALID_NESTING"
  }
  ```

**Priority**: P0 (Critical - core feature)

---

### TC-REP-003: Reply to Non-Existent Comment

**Objective**: Handle invalid parent ID

**Steps**:
```bash

RESPONSE=$(curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/comments/507f1f77bcf86cd799439099/replies")
        mediaId = "507f1f77bcf86cd799439011"
        text = "Reply"
    } | ConvertTo-Json)  -ContentType "application/json"
```

**Expected Result**:
- Status: 404 Not Found
- Response:
  ```json
  {
    "success": false,
    "message": "Parent comment not found",
    "errorCode": "COMMENT_NOT_FOUND"
  }
  ```

**Priority**: P1

---

### TC-REP-004: List Replies Chronologically

**Objective**: Verify reply ordering (oldest first)

**Steps**:
```bash

# 1. Create comment
$comment = Test-CreateComment -MediaId "507f1f77bcf86cd799439011" -Text "Top level"

# 2. Create 3 replies
$reply1 = Test-CreateReply -CommentId $comment.data.id -Text "First reply"
Start-Sleep -Seconds 1
$reply2 = Test-CreateReply -CommentId $comment.data.id -Text "Second reply"
Start-Sleep -Seconds 1
$reply3 = Test-CreateReply -CommentId $comment.data.id -Text "Third reply"

# 3. List replies
RESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/comments/$($comment.data.id)/replies")
```

**Expected Result**:
- Status: 200 OK
- Replies ordered by `createdAt ASC`:
  ```json
  {
    "data": {
      "items": [
        { "text": "First reply", "createdAt": "2026-02-14T10:00:00Z" },
        { "text": "Second reply", "createdAt": "2026-02-14T10:00:01Z" },
        { "text": "Third reply", "createdAt": "2026-02-14T10:00:02Z" }
      ]
    }
  }
  ```

**Priority**: P1

---

## User Tagging Tests

### TC-TAG-001: Tag Up to 10 Users

**Objective**: Allow max 10 tags

**Steps**:
```bash

$taggedUserIds = @(
    "550e8400-e29b-41d4-a716-446655440001",
    "550e8400-e29b-41d4-a716-446655440002",
    "550e8400-e29b-41d4-a716-446655440003",
    "550e8400-e29b-41d4-a716-446655440004",
    "550e8400-e29b-41d4-a716-446655440005",
    "550e8400-e29b-41d4-a716-446655440006",
    "550e8400-e29b-41d4-a716-446655440007",
    "550e8400-e29b-41d4-a716-446655440008",
    "550e8400-e29b-41d4-a716-446655440009",
    "550e8400-e29b-41d4-a716-446655440010"
)

RESPONSE=$(curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/comments/create")
        mediaId = "507f1f77bcf86cd799439011"
        text = "Check this out!"
        taggedUserIds = $taggedUserIds
    } | ConvertTo-Json)  -ContentType "application/json"
```

**Expected Result**:
- Status: 201 Created
- All 10 users receive mention notifications
- `taggedUserIds` array has 10 elements

**Priority**: P1

---

### TC-TAG-002: Tag More Than 10 Users (Limit Enforcement)

**Objective**: Truncate to 10 tags

**Steps**:
```bash

$taggedUserIds = @(1..15 | ForEach-Object { 
    "550e8400-e29b-41d4-a716-44665544000$_" 
})

RESPONSE=$(curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/comments/create")
        mediaId = "507f1f77bcf86cd799439011"
        text = "Tag spam"
        taggedUserIds = $taggedUserIds
    } | ConvertTo-Json)  -ContentType "application/json"
```

**Expected Result**:
- Status: 201 Created
- Only first 10 tags saved (slice(0, 10))
- Only 10 mention notifications sent

**Priority**: P1

---

### TC-TAG-003: Self-Tag Filtering

**Objective**: Prevent self-tagging in notifications

**Steps**:
```bash

RESPONSE=$(curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/comments/create")
        mediaId = "507f1f77bcf86cd799439011"
        text = "Tagging myself"
        taggedUserIds = @(
            "550e8400-e29b-41d4-a716-446655440001",  # Self (user1)
            "550e8400-e29b-41d4-a716-446655440002"
        )
    } | ConvertTo-Json)  -ContentType "application/json"
```

**Expected Result**:
- Status: 201 Created
- Only user2 receives mention notification
- User1 (self) does NOT receive notification

**Priority**: P1

---

### TC-TAG-004: Duplicate Tag Deduplication

**Objective**: Filter duplicate tags

**Steps**:
```bash

RESPONSE=$(curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/comments/create")
        mediaId = "507f1f77bcf86cd799439011"
        text = "Duplicate tags"
        taggedUserIds = @(
            "550e8400-e29b-41d4-a716-446655440002",
            "550e8400-e29b-41d4-a716-446655440002",
            "550e8400-e29b-41d4-a716-446655440002"
        )
    } | ConvertTo-Json)  -ContentType "application/json"
```

**Expected Result**:
- Status: 201 Created
- User2 receives only ONE mention notification
- Duplicates deduplicated before sending

**Priority**: P2

---

## Like/Unlike Tests

### TC-LIKE-001: Like Comment

**Objective**: Add like to comment

**Steps**:
```bash

# 1. Create comment
$comment = Test-CreateComment -MediaId "507f1f77bcf86cd799439011" -Text "Test"

# 2. Like comment
RESPONSE=$(curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/comments/$($comment.data.id)/like")
```

**Expected Result**:
- Status: 200 OK
- Response:
  ```json
  {
    "success": true,
    "message": "Comment liked",
    "data": {
      "commentId": "<COMMENT_ID>",
      "likesCount": 1,
      "isLiked": true
    }
  }
  ```
- User2 added to `likedBy` array
- `likesCount` incremented
- Notification sent to comment owner

**Priority**: P0

---

### TC-LIKE-002: Unlike Comment

**Objective**: Remove like from comment

**Steps**:
```bash

# 1. Create and like comment
$comment = Test-CreateComment -MediaId "507f1f77bcf86cd799439011" -Text "Test"
curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/comments/$($comment.data.id)/like"

# 2. Unlike comment
RESPONSE=$(curl -s \
  -X DELETE \
  "https://api-staging.chefooz.com/api/v1/comments/$($comment.data.id)/like")
```

**Expected Result**:
- Status: 200 OK
- Response:
  ```json
  {
    "success": true,
    "message": "Comment unliked",
    "data": {
      "commentId": "<COMMENT_ID>",
      "likesCount": 0,
      "isLiked": false
    }
  }
  ```
- User2 removed from `likedBy` array
- `likesCount` decremented
- No notification sent

**Priority**: P0

---

### TC-LIKE-003: Like Idempotency

**Objective**: Handle duplicate likes gracefully

**Steps**:
```bash

# 1. Create comment
$comment = Test-CreateComment -MediaId "507f1f77bcf86cd799439011" -Text "Test"

# 2. Like twice
RESPONSE1=$(curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/comments/$($comment.data.id)/like")

RESPONSE2=$(curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/comments/$($comment.data.id)/like")
```

**Expected Result**:
- Both responses: 200 OK
- Both `likesCount = 1` (not incremented twice)
- Only one entry in `likedBy` array
- Only one notification sent (on first like)

**Priority**: P0

---

### TC-LIKE-004: Unlike Without Prior Like (Idempotent)

**Objective**: Handle unlike on non-liked comment

**Steps**:
```bash

# 1. Create comment (don't like)
$comment = Test-CreateComment -MediaId "507f1f77bcf86cd799439011" -Text "Test"

# 2. Unlike
RESPONSE=$(curl -s \
  -X DELETE \
  "https://api-staging.chefooz.com/api/v1/comments/$($comment.data.id)/like")
```

**Expected Result**:
- Status: 200 OK
- Response:
  ```json
  {
    "success": true,
    "message": "Comment unliked",
    "data": {
      "likesCount": 0,
      "isLiked": false
    }
  }
  ```
- No error thrown (idempotent)

**Priority**: P1

---

### TC-LIKE-005: Self-Like No Notification

**Objective**: No notification for self-likes

**Steps**:
```bash

# 1. User1 creates comment
$comment = Test-CreateComment -MediaId "507f1f77bcf86cd799439011" -Text "Test"

# 2. User1 likes own comment
RESPONSE=$(curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/comments/$($comment.data.id)/like")
```

**Expected Result**:
- Status: 200 OK
- `likesCount` incremented
- NO notification sent (self-like filter)

**Priority**: P1

---

## Pin/Unpin Tests

### TC-PIN-001: Pin Comment (Creator Only)

**Objective**: Reel creator can pin comment

**Steps**:
```bash

# 1. User1 comments on User2's reel
$comment = Test-CreateComment -MediaId "507f1f77bcf86cd799439011" -Text "Great!"

# 2. User2 (creator) pins comment
RESPONSE=$(curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/comments/$($comment.data.id)/pin" \
  -H "Content-Type: application/json")
```

**Expected Result**:
- Status: 200 OK
- Response:
  ```json
  {
    "success": true,
    "message": "Comment pinned",
    "data": {
      "commentId": "<COMMENT_ID>",
      "isPinned": true
    }
  }
  ```
- Comment `isPinned = true`

**Priority**: P1

---

### TC-PIN-002: Only One Pinned Comment Per Reel

**Objective**: Auto-unpin previous when pinning new

**Steps**:
```bash

# 1. Create two comments
$comment1 = Test-CreateComment -MediaId "507f1f77bcf86cd799439011" -Text "First"
$comment2 = Test-CreateComment -MediaId "507f1f77bcf86cd799439011" -Text "Second"

# 2. Pin first comment
curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/comments/$($comment1.data.id)/pin" \
  -H "Content-Type: application/json"

# 3. Pin second comment
curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/comments/$($comment2.data.id)/pin" \
  -H "Content-Type: application/json"

# 4. Verify only second is pinned
LIST=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/comments/list?mediaId=507f1f77bcf86cd799439011")
```

**Expected Result**:
- Only `comment2.isPinned = true`
- `comment1.isPinned = false` (auto-unpinned)

**Priority**: P1

---

### TC-PIN-003: Non-Creator Cannot Pin

**Objective**: Reject pin by non-creator

**Steps**:
```bash

# 1. User1 comments on User2's reel
$comment = Test-CreateComment -MediaId "507f1f77bcf86cd799439011" -Text "Test"

# 2. User1 tries to pin (not creator)
RESPONSE=$(curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/comments/$($comment.data.id)/pin" \
  -H "Content-Type: application/json")
```

**Expected Result**:
- Status: 403 Forbidden
- Response:
  ```json
  {
    "success": false,
    "message": "Only reel creator can pin comments",
    "errorCode": "FORBIDDEN"
  }
  ```

**Priority**: P0

---

### TC-PIN-004: Cannot Pin Reply

**Objective**: Only top-level comments can be pinned

**Steps**:
```bash

# 1. Create comment and reply
$comment = Test-CreateComment -MediaId "507f1f77bcf86cd799439011" -Text "Top"
$reply = Test-CreateReply -CommentId $comment.data.id -Text "Reply"

# 2. Try to pin reply
RESPONSE=$(curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/comments/$($reply.data.id)/pin" \
  -H "Content-Type: application/json")
```

**Expected Result**:
- Status: 400 Bad Request
- Response:
  ```json
  {
    "success": false,
    "message": "Cannot pin a reply, only top-level comments",
    "errorCode": "INVALID_ACTION"
  }
  ```

**Priority**: P1

---

### TC-PIN-005: Unpin Comment

**Objective**: Remove pin from reel

**Steps**:
```bash

# 1. Pin comment
$comment = Test-CreateComment -MediaId "507f1f77bcf86cd799439011" -Text "Test"
curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/comments/$($comment.data.id)/pin" \
  -H "Content-Type: application/json"

# 2. Unpin
RESPONSE=$(curl -s \
  -X DELETE \
  "https://api-staging.chefooz.com/api/v1/comments/pin" \
  -H "Content-Type: application/json")
```

**Expected Result**:
- Status: 200 OK
- All comments on reel have `isPinned = false`

**Priority**: P2

---

## Comment Deletion Tests

### TC-DEL-001: Delete Own Comment

**Objective**: User can delete own comment

**Steps**:
```bash

# 1. Create comment
$comment = Test-CreateComment -MediaId "507f1f77bcf86cd799439011" -Text "Test"

# 2. Delete comment
RESPONSE=$(curl -s \
  -X DELETE \
  "https://api-staging.chefooz.com/api/v1/comments/delete/$($comment.data.id)")
```

**Expected Result**:
- Status: 200 OK
- Response:
  ```json
  {
    "success": true,
    "message": "Comment deleted",
    "data": null
  }
  ```
- Comment `isDeleted = true` (soft delete)
- Reel `stats.comments` decremented

**Priority**: P0

---

### TC-DEL-002: Cannot Delete Others' Comments

**Objective**: Authorization check

**Steps**:
```bash

# 1. User1 creates comment
$comment = Test-CreateComment -MediaId "507f1f77bcf86cd799439011" -Text "Test"

# 2. User2 tries to delete
RESPONSE=$(curl -s \
  -X DELETE \
  "https://api-staging.chefooz.com/api/v1/comments/delete/$($comment.data.id)")
```

**Expected Result**:
- Status: 403 Forbidden
- Response:
  ```json
  {
    "success": false,
    "message": "You can only delete your own comments",
    "errorCode": "FORBIDDEN"
  }
  ```

**Priority**: P0

---

### TC-DEL-003: Deleted Comments Not in List

**Objective**: Filter soft-deleted comments

**Steps**:
```bash

# 1. Create and delete comment
$comment = Test-CreateComment -MediaId "507f1f77bcf86cd799439011" -Text "Test"
curl -s \
  -X DELETE \
  "https://api-staging.chefooz.com/api/v1/comments/delete/$($comment.data.id)"

# 2. List comments
RESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/comments/list?mediaId=507f1f77bcf86cd799439011")
```

**Expected Result**:
- Status: 200 OK
- Deleted comment NOT in `items` array
- Filter: `{ isDeleted: false }`

**Priority**: P0

---

### TC-DEL-004: Delete Top-Level Comment Preserves Replies

**Objective**: Soft delete maintains structure

**Steps**:
```bash

# 1. Create comment with replies
$comment = Test-CreateComment -MediaId "507f1f77bcf86cd799439011" -Text "Top"
$reply1 = Test-CreateReply -CommentId $comment.data.id -Text "Reply 1"
$reply2 = Test-CreateReply -CommentId $comment.data.id -Text "Reply 2"

# 2. Delete parent comment
curl -s \
  -X DELETE \
  "https://api-staging.chefooz.com/api/v1/comments/delete/$($comment.data.id)"

# 3. Check replies still exist
REPLIES=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/comments/$($comment.data.id)/replies")
```

**Expected Result**:
- Parent comment `isDeleted = true`
- Replies still accessible (not deleted)
- `replyCount` unchanged (preserves count)

**Priority**: P1

---

## Pagination Tests

### TC-PAG-001: Cursor-Based Pagination

**Objective**: Verify cursor pagination works

**Steps**:
```bash

# 1. Create 25 comments
1..25 | ForEach-Object {
    Test-CreateComment -MediaId "507f1f77bcf86cd799439011" -Text "Comment $_"
    Start-Sleep -Milliseconds 100
}

# 2. Fetch page 1
PAGE1=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/comments/list?mediaId=507f1f77bcf86cd799439011&limit=20")

# 3. Fetch page 2
$nextCursor = ($page1.Content | jq .).data.nextCursor
PAGE2=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/comments/list?mediaId=507f1f77bcf86cd799439011&limit=20&cursor=$nextCursor")
```

**Expected Result**:
- Page 1: 20 items + `nextCursor`
- Page 2: 5 items, no `nextCursor`
- No duplicate comments

**Priority**: P0

---

### TC-PAG-002: Pinned Comments Always First

**Objective**: Pinned comment shown first regardless of date

**Steps**:
```bash

# 1. Create 5 comments
1..5 | ForEach-Object {
    Test-CreateComment -MediaId "507f1f77bcf86cd799439011" -Text "Comment $_"
    Start-Sleep -Milliseconds 100
}

# 2. Pin the 3rd oldest comment
COMMENTS=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/comments/list?mediaId=507f1f77bcf86cd799439011")

$thirdCommentId = $comments.data.items[2].id
curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/comments/$thirdCommentId/pin" \
  -H "Content-Type: application/json"

# 3. List comments again
RESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/comments/list?mediaId=507f1f77bcf86cd799439011")
```

**Expected Result**:
- First item in list is pinned comment
- Remaining sorted by `createdAt DESC`

**Priority**: P1

---

## Security Tests

### TC-SEC-001: SQL Injection in Text

**Objective**: Prevent SQL injection attacks

**Steps**:
```bash

RESPONSE=$(curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/comments/create")
        mediaId = "507f1f77bcf86cd799439011"
        text = "'; DROP TABLE comments; --"
    } | ConvertTo-Json)  -ContentType "application/json"
```

**Expected Result**:
- Status: 201 Created
- Text stored as-is (MongoDB parameterized queries)
- No SQL execution

**Priority**: P0

---

### TC-SEC-002: XSS in Comment Text

**Objective**: Sanitize HTML/JavaScript

**Steps**:
```bash

RESPONSE=$(curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/comments/create")
        mediaId = "507f1f77bcf86cd799439011"
        text = "<script>alert('XSS')</script>"
    } | ConvertTo-Json)  -ContentType "application/json"
```

**Expected Result**:
- Status: 201 Created
- Text sanitized (HTML encoded or stripped)
- No script execution on frontend

**Priority**: P0

---

### TC-SEC-003: Rate Limiting

**Objective**: Prevent spam

**Steps**:
```bash

# Send 100 comments rapidly
1..100 | ForEach-Object {
curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/comments/create"
            mediaId = "507f1f77bcf86cd799439011"
            text = "Spam $_"
        } | ConvertTo-Json)  -ContentType "application/json"
}
```

**Expected Result**:
- First N requests: 201 Created
- After threshold: 429 Too Many Requests
- Rate limit header: `X-RateLimit-Remaining: 0`

**Priority**: P1

---

## Performance Tests

### TC-PERF-001: List Comments with 1000 Comments

**Objective**: Verify pagination performance

**Steps**:
```bash

# 1. Create 1000 comments
1..1000 | ForEach-Object {
    Test-CreateComment -MediaId "507f1f77bcf86cd799439011" -Text "Comment $_"
}

# 2. Measure list query time
$start = Get-Date
RESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/comments/list?mediaId=507f1f77bcf86cd799439011&limit=20")
$duration = (Get-Date) - $start
```

**Expected Result**:
- Response time: < 200ms
- Memory usage: < 50MB
- Query uses index: `{ mediaId: 1, createdAt: -1 }`

**Priority**: P1

---

### TC-PERF-002: Batch User Lookup (No N+1)

**Objective**: Verify efficient user enrichment

**Steps**:
```bash

# 1. 10 users create 100 comments each (1000 total)
# 2. Enable DB query logging
# 3. List comments
RESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/comments/list?mediaId=507f1f77bcf86cd799439011&limit=20")

# 4. Check query count
```

**Expected Result**:
- Only 2 DB queries:
  1. MongoDB: Fetch comments
  2. PostgreSQL: Batch fetch users (single `IN` query)
- No N+1 problem (no loop of user queries)

**Priority**: P0

---

## Notification Tests

### TC-NOTIF-001: Comment Notification

**Objective**: Reel owner receives notification

**Steps**:
```bash

# 1. User1 comments on User2's reel
$comment = Test-CreateComment -MediaId "507f1f77bcf86cd799439011" -Text "Great!"

# 2. Check User2's notifications
NOTIFICATIONS=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/notifications/list")
```

**Expected Result**:
- User2 has notification:
  ```json
  {
    "type": "REEL_COMMENTED",
    "actorUserId": "550e8400-e29b-41d4-a716-446655440001",
    "actorUsername": "john_doe",
    "metadata": {
      "reelId": "507f1f77bcf86cd799439011",
      "commentId": "<COMMENT_ID>",
      "commentText": "Great!"
    }
  }
  ```

**Priority**: P1

---

### TC-NOTIF-002: Reply Notification

**Objective**: Comment owner receives reply notification

**Steps**:
```bash

# 1. User1 comments
$comment = Test-CreateComment -MediaId "507f1f77bcf86cd799439011" -Text "Question?"

# 2. User2 replies
$reply = Test-CreateReply -CommentId $comment.data.id -Text "Answer!"

# 3. Check User1's notifications
NOTIFICATIONS=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/notifications/list")
```

**Expected Result**:
- User1 has notification:
  ```json
  {
    "type": "COMMENT_REPLIED",
    "actorUserId": "550e8400-e29b-41d4-a716-446655440002",
    "actorUsername": "chef_rakesh",
    "metadata": {
      "commentId": "<COMMENT_ID>",
      "replyId": "<REPLY_ID>",
      "replyText": "Answer!"
    }
  }
  ```

**Priority**: P1

---

### TC-NOTIF-003: Mention Notification

**Objective**: Tagged users receive mention notifications

**Steps**:
```bash

# 1. User1 tags User3 in comment
COMMENT=$(curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/comments/create")
        mediaId = "507f1f77bcf86cd799439011"
        text = "Check this out!"
        taggedUserIds = @("550e8400-e29b-41d4-a716-446655440003")
    } | ConvertTo-Json)  -ContentType "application/json"

# 2. Check User3's notifications
NOTIFICATIONS=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/notifications/list")
```

**Expected Result**:
- User3 has notification:
  ```json
  {
    "type": "REEL_COMMENTED",
    "metadata": {
      "type": "mention",
      "commentId": "<COMMENT_ID>",
      "username": "john_doe",
      "mentionType": "comment"
    }
  }
  ```

**Priority**: P1

---

## Test Summary

### Test Coverage

| Category | Total Tests | P0 (Critical) | P1 (High) | P2 (Medium) |
|----------|-------------|---------------|-----------|-------------|
| Comment Creation | 7 | 4 | 3 | 0 |
| Reply System | 4 | 2 | 2 | 0 |
| User Tagging | 4 | 0 | 3 | 1 |
| Like/Unlike | 5 | 3 | 2 | 0 |
| Pin/Unpin | 5 | 1 | 3 | 1 |
| Comment Deletion | 4 | 3 | 1 | 0 |
| Pagination | 2 | 1 | 1 | 0 |
| Security | 3 | 2 | 1 | 0 |
| Performance | 2 | 1 | 1 | 0 |
| Notifications | 3 | 0 | 3 | 0 |
| **TOTAL** | **39** | **17** | **20** | **2** |

### Test Execution Plan

**Phase 1: Critical (P0)** - 17 tests
- Must pass before QA sign-off
- Covers core functionality and security

**Phase 2: High Priority (P1)** - 20 tests
- Must pass before release
- Covers important features and edge cases

**Phase 3: Medium Priority (P2)** - 2 tests
- Should pass for production
- Covers nice-to-have features

---

## Related Documentation

- **Feature Overview**: `01_FEATURE_OVERVIEW.md` (Business context)
- **Technical Guide**: `02_TECHNICAL_GUIDE.md` (Implementation details)
- **Backend API**: `apps/chefooz-apis/src/modules/comments/`

---

**[QA_COMPLETE ✅]**  
**Module**: Comments  
**Test Cases**: 39 scenarios across 11 categories  
**Date**: February 14, 2026
