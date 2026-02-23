# Social Module - QA Test Cases

**Module**: `social`  
**Type**: Core Feature  
**Category**: Social & Discovery  
**Status**: Production  
**Last Updated**: February 14, 2026

---

## 📋 Table of Contents

1. [Test Environment Setup](#test-environment-setup)
2. [Follow System Tests](#follow-system-tests)
3. [Follow Request Tests](#follow-request-tests)
4. [Blocking Tests](#blocking-tests)
5. [Privacy Settings Tests](#privacy-settings-tests)
6. [Social Lists Tests](#social-lists-tests)
7. [Search Tests](#search-tests)
8. [Relationship State Tests](#relationship-state-tests)
9. [Reel Sharing Tests](#reel-sharing-tests)
10. [Cache Tests](#cache-tests)
11. [Notification Tests](#notification-tests)
12. [Performance Tests](#performance-tests)
13. [Security Tests](#security-tests)
14. [Integration Tests](#integration-tests)
15. [Regression Tests](#regression-tests)

---

## Test Environment Setup

### Prerequisites

**Database Setup**:
```sql
-- PostgreSQL (Social relationships)
CREATE DATABASE chefooz_test;

-- MongoDB (Reels)
use chefooz_test;
```

**Test Data**:
```typescript
const testUsers = {
  publicUser: {
    id: 'public-user-uuid',
    username: 'public_user',
    isPrivate: false,
  },
  privateUser: {
    id: 'private-user-uuid',
    username: 'private_user',
    isPrivate: true,
  },
  chefUser: {
    id: 'chef-user-uuid',
    username: 'chef_maria',
    isPrivate: false,
    chefProfile: {
      businessName: 'Maria\'s Kitchen',
    },
  },
  blockedUser: {
    id: 'blocked-user-uuid',
    username: 'blocked_user',
    isPrivate: false,
  },
};

const testReels = {
  reel1: {
    _id: '691ca198b8be29951a3caf6b',
    userId: 'chef-user-uuid',
    stats: {
      shareCount: 0,
      copyLinkCount: 0,
      externalShareCount: 0,
      directShareCount: 0,
      storyShareCount: 0,
    },
  },
};
```

**Test Configuration**:
```typescript
const testConfig = {
  baseUrl: 'https://api-staging.chefooz.com/api/v1',
  jwtTokens: {
    publicUser: 'eyJhbGciOiJIUzI1NiIs...',
    privateUser: 'eyJhbGciOiJIUzI1NiIs...',
    chefUser: 'eyJhbGciOiJIUzI1NiIs...',
  },
};
```

---

## Follow System Tests

### Test 1.1: Follow Public Account (Auto-Accept)

**Priority**: High  
**Type**: Functional

**Preconditions**:
- User A authenticated
- User B exists (isPrivate = false)
- User A not following User B

**Test Steps**:
1. Call `POST /social/follow/:userBId` with User A's JWT
2. Verify response:
   - status: 200
   - data.status: 'accepted'
   - data.isPrivate: false
3. Verify database:
   - UserFollow record created (followerId=A, targetId=B, status='accepted')
   - UserPrivacy.followersCount (User B) incremented by 1
   - UserPrivacy.followingCount (User A) incremented by 1
4. Verify cache invalidated:
   - social:profile:A
   - social:profile:B
   - social:privacy:A
   - social:privacy:B
5. Verify notification sent:
   - Type: FOLLOWED
   - Recipient: User B
   - Context: isPending = false

**Expected Result**: User A follows User B immediately (auto-accept), counters incremented, notification sent.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 1.2: Follow Private Account (Pending)

**Priority**: High  
**Type**: Functional

**Preconditions**:
- User A authenticated
- Chef C exists (isPrivate = true)
- User A not following Chef C

**Test Steps**:
1. Call `POST /social/follow/:chefCId` with User A's JWT
2. Verify response:
   - status: 200
   - data.status: 'pending'
   - data.isPrivate: true
3. Verify database:
   - UserFollow record created (followerId=A, targetId=C, status='pending')
   - Counters NOT incremented (pending state)
4. Verify notification sent:
   - Type: FOLLOWED
   - Recipient: Chef C
   - Context: isPending = true

**Expected Result**: Follow request pending, no counter changes, notification sent with isPending flag.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 1.3: Prevent Duplicate Follow

**Priority**: High  
**Type**: Validation

**Preconditions**:
- User A authenticated
- User A already following User B (status='accepted')

**Test Steps**:
1. Call `POST /social/follow/:userBId` with User A's JWT
2. Verify response:
   - status: 400
   - errorCode: 'SOCIAL_ALREADY_FOLLOWING'
   - message: 'Already following or request pending'

**Expected Result**: 400 error, no database changes.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 1.4: Prevent Self-Follow

**Priority**: High  
**Type**: Validation

**Preconditions**:
- User A authenticated

**Test Steps**:
1. Call `POST /social/follow/:userAId` with User A's JWT (follow self)
2. Verify response:
   - status: 400
   - errorCode: 'SOCIAL_SELF_FOLLOW_NOT_ALLOWED'
   - message: 'Cannot follow yourself'

**Expected Result**: 400 error, no database changes.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 1.5: Follow with Blocking (Bidirectional Check)

**Priority**: High  
**Type**: Security

**Preconditions**:
- User A authenticated
- User B blocked User A (UserBlock: userId=B, blockedUserId=A)

**Test Steps**:
1. Call `POST /social/follow/:userBId` with User A's JWT
2. Verify response:
   - status: 403
   - errorCode: 'SOCIAL_USER_BLOCKED'
   - message: 'Cannot follow this user'

**Expected Result**: 403 error, follow prevented due to block.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 1.6: Unfollow User

**Priority**: High  
**Type**: Functional

**Preconditions**:
- User A authenticated
- User A following User B (status='accepted')

**Test Steps**:
1. Call `POST /social/unfollow/:userBId` with User A's JWT
2. Verify response:
   - status: 200
   - message: 'Successfully unfollowed user'
3. Verify database:
   - UserFollow record deleted
   - UserPrivacy.followersCount (User B) decremented by 1
   - UserPrivacy.followingCount (User A) decremented by 1
4. Verify cache invalidated
5. Verify no notification sent

**Expected Result**: Unfollow successful, counters decremented, no notification.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 1.7: Unfollow Without Following

**Priority**: Medium  
**Type**: Validation

**Preconditions**:
- User A authenticated
- User A NOT following User B

**Test Steps**:
1. Call `POST /social/unfollow/:userBId` with User A's JWT
2. Verify response:
   - status: 400
   - errorCode: 'SOCIAL_NOT_FOLLOWING'
   - message: 'Not following this user'

**Expected Result**: 400 error, no database changes.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 1.8: Follow Rate Limiting

**Priority**: High  
**Type**: Performance

**Preconditions**:
- User A authenticated
- 30 different target users (B1, B2, ..., B30)

**Test Steps**:
1. Call `POST /social/follow/:B1`, ..., `POST /social/follow/:B30` rapidly (within 60 seconds)
2. Verify first 30 requests succeed (200 status)
3. Call `POST /social/follow/:B31` (31st request within 60 seconds)
4. Verify response:
   - status: 429
   - errorCode: 'FOLLOW_RATE_LIMIT_EXCEEDED'
   - message: 'Too many follow requests. Please wait.'

**Expected Result**: First 30 succeed, 31st fails with 429 error.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 1.9: Follow with Username Resolution

**Priority**: Medium  
**Type**: Functional

**Preconditions**:
- User A authenticated
- User B username: 'chef_maria'

**Test Steps**:
1. Call `POST /social/follow/chef_maria` with User A's JWT (username instead of UUID)
2. Verify response:
   - status: 200
   - data.status: 'accepted'
   - data.targetUserId: User B's UUID

**Expected Result**: Username resolved to UUID, follow successful.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 1.10: Remove Follower (Instagram-style)

**Priority**: Medium  
**Type**: Functional

**Preconditions**:
- Chef C authenticated
- User F following Chef C (status='accepted')

**Test Steps**:
1. Call `POST /social/remove-follower/:userFId` with Chef C's JWT
2. Verify response:
   - status: 200
   - message: 'Follower removed'
3. Verify database:
   - UserFollow record deleted (followerId=F, targetId=C)
   - Counters decremented
4. Verify no notification sent (soft removal)

**Expected Result**: Follower removed, counters updated, no notification.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

## Follow Request Tests

### Test 2.1: Accept Follow Request

**Priority**: High  
**Type**: Functional

**Preconditions**:
- Chef C authenticated (private account)
- User A sent follow request (status='pending')

**Test Steps**:
1. Call `GET /social/requests/pending` with Chef C's JWT
2. Verify response contains User A's request
3. Extract requestId from response
4. Call `POST /social/requests/:requestId/accept` with Chef C's JWT
5. Verify response:
   - status: 200
   - message: 'Follow request accepted'
6. Verify database:
   - UserFollow.status updated to 'accepted'
   - Counters incremented
7. Verify notification sent:
   - Type: FOLLOW_REQUEST_ACCEPTED
   - Recipient: User A

**Expected Result**: Request accepted, status updated, counters incremented, notification sent.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 2.2: Reject Follow Request

**Priority**: High  
**Type**: Functional

**Preconditions**:
- Chef C authenticated
- User A sent follow request (status='pending')

**Test Steps**:
1. Call `GET /social/requests/pending` with Chef C's JWT
2. Extract requestId
3. Call `POST /social/requests/:requestId/reject` with Chef C's JWT
4. Verify response:
   - status: 200
   - message: 'Follow request rejected'
5. Verify database:
   - UserFollow record deleted
   - Counters NOT changed
6. Verify no notification sent (soft rejection)

**Expected Result**: Request deleted, no counter changes, no notification.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 2.3: Accept Non-Existent Request

**Priority**: Medium  
**Type**: Validation

**Preconditions**:
- Chef C authenticated
- requestId: 'invalid-uuid'

**Test Steps**:
1. Call `POST /social/requests/invalid-uuid/accept` with Chef C's JWT
2. Verify response:
   - status: 404
   - errorCode: 'SOCIAL_REQUEST_NOT_FOUND'
   - message: 'Follow request not found'

**Expected Result**: 404 error, no database changes.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 2.4: Get Pending Requests with Pagination

**Priority**: Medium  
**Type**: Functional

**Preconditions**:
- Chef C authenticated
- 25 pending follow requests

**Test Steps**:
1. Call `GET /social/requests/pending?limit=20` with Chef C's JWT
2. Verify response:
   - status: 200
   - data.items.length: 20
   - data.hasMore: true
   - data.nextCursor: ISO date string
3. Call `GET /social/requests/pending?cursor={nextCursor}&limit=20`
4. Verify response:
   - data.items.length: 5
   - data.hasMore: false
   - data.nextCursor: null

**Expected Result**: Pagination works correctly, 20 items first page, 5 items second page.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 2.5: Pending Requests Sorted by Date (Newest First)

**Priority**: Low  
**Type**: Functional

**Preconditions**:
- Chef C authenticated
- 10 pending requests with different timestamps

**Test Steps**:
1. Call `GET /social/requests/pending` with Chef C's JWT
2. Verify response items sorted by createdAt DESC (newest first)
3. Check: items[0].createdAt > items[1].createdAt > ... > items[9].createdAt

**Expected Result**: Requests sorted newest first.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

## Blocking Tests

### Test 3.1: Block User

**Priority**: High  
**Type**: Functional

**Preconditions**:
- User A authenticated
- User D exists
- User A following User D (status='accepted')
- User D following User A (status='accepted')

**Test Steps**:
1. Call `POST /social/block/:userDId` with User A's JWT
2. Verify response:
   - status: 200
   - message: 'User blocked successfully'
3. Verify database:
   - UserBlock record created (userId=A, blockedUserId=D)
   - UserFollow(followerId=A, targetId=D) deleted
   - UserFollow(followerId=D, targetId=A) deleted
   - Counters decremented for both users (2 follows removed)
4. Verify cache invalidated
5. Verify no notification sent (soft block)

**Expected Result**: Block created, all follows removed, counters updated, no notification.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 3.2: Unblock User

**Priority**: High  
**Type**: Functional

**Preconditions**:
- User A authenticated
- User A blocked User D (UserBlock: userId=A, blockedUserId=D)

**Test Steps**:
1. Call `POST /social/unblock/:userDId` with User A's JWT
2. Verify response:
   - status: 200
   - message: 'User unblocked successfully'
3. Verify database:
   - UserBlock record deleted
4. Verify cache invalidated
5. Verify User A can now follow User D (no longer blocked)

**Expected Result**: Block removed, users can interact again.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 3.3: Prevent Self-Block

**Priority**: High  
**Type**: Validation

**Preconditions**:
- User A authenticated

**Test Steps**:
1. Call `POST /social/block/:userAId` with User A's JWT (block self)
2. Verify response:
   - status: 400
   - errorCode: 'SOCIAL_SELF_BLOCK_NOT_ALLOWED'
   - message: 'Cannot block yourself'

**Expected Result**: 400 error, no database changes.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 3.4: Block Rate Limiting

**Priority**: High  
**Type**: Performance

**Preconditions**:
- User A authenticated
- 20 different users (D1, D2, ..., D20)

**Test Steps**:
1. Call `POST /social/block/:D1`, ..., `POST /social/block/:D20` rapidly (within 60 seconds)
2. Verify first 20 requests succeed (200 status)
3. Call `POST /social/block/:D21` (21st request within 60 seconds)
4. Verify response:
   - status: 429
   - errorCode: 'BLOCK_RATE_LIMIT_EXCEEDED'
   - message: 'Too many block requests. Please wait.'

**Expected Result**: First 20 succeed, 21st fails with 429 error.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 3.5: Check Block Status (Bidirectional)

**Priority**: Medium  
**Type**: Functional

**Preconditions**:
- User A authenticated
- User A blocked User D (UserBlock: userId=A, blockedUserId=D)

**Test Steps**:
1. Call `GET /social/blocked/check/:userDId` with User A's JWT
2. Verify response:
   - status: 200
   - data.isBlocked: true
   - data.hasBlocked: true (User A blocked User D)
   - data.isBlockedBy: false (User D did NOT block User A)

**Expected Result**: Correct block status flags.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 3.6: Check Block Status (Reverse)

**Priority**: Medium  
**Type**: Functional

**Preconditions**:
- User D authenticated
- User A blocked User D (UserBlock: userId=A, blockedUserId=D)

**Test Steps**:
1. Call `GET /social/blocked/check/:userAId` with User D's JWT
2. Verify response:
   - status: 200
   - data.isBlocked: true
   - data.hasBlocked: false (User D did NOT block User A)
   - data.isBlockedBy: true (User A blocked User D)

**Expected Result**: Correct block status flags (reverse perspective).

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 3.7: Get Blocked Users List

**Priority**: Medium  
**Type**: Functional

**Preconditions**:
- User A authenticated
- User A blocked 3 users (D1, D2, D3)

**Test Steps**:
1. Call `GET /social/blocked` with User A's JWT
2. Verify response:
   - status: 200
   - data.items.length: 3
   - data.items[0].username: 'user_d1' (or similar)
   - data.items[0].blockedAt: ISO date string

**Expected Result**: List contains 3 blocked users with user data.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 3.8: Block Prevents Follow

**Priority**: High  
**Type**: Integration

**Preconditions**:
- User A authenticated
- User A blocked User D

**Test Steps**:
1. Call `POST /social/follow/:userDId` with User A's JWT
2. Verify response:
   - status: 403
   - errorCode: 'SOCIAL_USER_BLOCKED'

**Expected Result**: Follow prevented due to block.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

## Privacy Settings Tests

### Test 4.1: Get Privacy Settings

**Priority**: Medium  
**Type**: Functional

**Preconditions**:
- User A authenticated
- User A has privacy record (isPrivate=false, followersCount=50, followingCount=30)

**Test Steps**:
1. Call `GET /social/profile/privacy` with User A's JWT
2. Verify response:
   - status: 200
   - data.isPrivate: false
   - data.followersCount: 50
   - data.followingCount: 30

**Expected Result**: Privacy settings returned correctly.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 4.2: Update Privacy to Private

**Priority**: High  
**Type**: Functional

**Preconditions**:
- User A authenticated
- User A is public (isPrivate=false)

**Test Steps**:
1. Call `POST /social/profile/privacy` with body `{ "isPrivate": true }` and User A's JWT
2. Verify response:
   - status: 200
   - data.isPrivate: true
3. Verify database:
   - UserPrivacy.isPrivate updated to true
4. Verify cache invalidated: social:privacy:A
5. Verify existing followers unaffected
6. Follow User A with new user → status should be 'pending'

**Expected Result**: Privacy toggled, existing followers remain, new follows require approval.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 4.3: Update Privacy to Public

**Priority**: Medium  
**Type**: Functional

**Preconditions**:
- Chef C authenticated
- Chef C is private (isPrivate=true)
- Chef C has 3 pending follow requests

**Test Steps**:
1. Call `POST /social/profile/privacy` with body `{ "isPrivate": false }` and Chef C's JWT
2. Verify response:
   - status: 200
   - data.isPrivate: false
3. Verify pending requests remain pending (manual accept still required)
4. Follow Chef C with new user → status should be 'accepted' (auto-accept now)

**Expected Result**: Privacy toggled, pending requests unchanged, new follows auto-accept.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 4.4: Auto-Create Privacy Record

**Priority**: Low  
**Type**: Functional

**Preconditions**:
- User A authenticated
- User A has NO UserPrivacy record

**Test Steps**:
1. Call `GET /social/profile/privacy` with User A's JWT
2. Verify response:
   - status: 200
   - data.isPrivate: false (default)
   - data.followersCount: 0 (default)
   - data.followingCount: 0 (default)
3. Verify database:
   - UserPrivacy record created for User A

**Expected Result**: Privacy record auto-created with defaults.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

## Social Lists Tests

### Test 5.1: Get Own Followers

**Priority**: High  
**Type**: Functional

**Preconditions**:
- User A authenticated
- User A has 5 accepted followers (B1, B2, B3, B4, B5)

**Test Steps**:
1. Call `GET /social/followers` with User A's JWT
2. Verify response:
   - status: 200
   - data.items.length: 5
   - data.items[0].username: 'user_b1' (or similar)
   - data.items[0].isFollowing: boolean (User A's relationship with follower)
   - data.items[0].isFollowedBy: true (all are followers)

**Expected Result**: List contains 5 followers with relationship status.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 5.2: Get User's Followers by UUID

**Priority**: High  
**Type**: Functional

**Preconditions**:
- User A authenticated
- Chef C has 10 accepted followers

**Test Steps**:
1. Call `GET /social/followers/:chefCId` with User A's JWT
2. Verify response:
   - status: 200
   - data.items.length: 10
   - data.items include relationship status for User A (viewer)

**Expected Result**: List shows Chef C's followers with User A's relationship to each.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 5.3: Get User's Followers by Username

**Priority**: Medium  
**Type**: Functional

**Preconditions**:
- User A authenticated
- Chef C username: 'chef_maria'
- Chef C has 10 accepted followers

**Test Steps**:
1. Call `GET /social/followers/chef_maria` with User A's JWT (username instead of UUID)
2. Verify response:
   - status: 200
   - data.items.length: 10

**Expected Result**: Username resolved to UUID, followers list returned.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 5.4: Get Own Following

**Priority**: High  
**Type**: Functional

**Preconditions**:
- User A authenticated
- User A following 8 users (C1, C2, ..., C8)

**Test Steps**:
1. Call `GET /social/following` with User A's JWT
2. Verify response:
   - status: 200
   - data.items.length: 8
   - data.items[0].isFollowing: true (User A follows all)
   - data.items[0].isFollowedBy: boolean (mutual follow status)

**Expected Result**: List contains 8 following with relationship status.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 5.5: Followers List with Pagination

**Priority**: Medium  
**Type**: Functional

**Preconditions**:
- User A authenticated
- User A has 25 accepted followers

**Test Steps**:
1. Call `GET /social/followers?limit=20` with User A's JWT
2. Verify response:
   - data.items.length: 20
   - data.hasMore: true
   - data.nextCursor: ISO date string
3. Call `GET /social/followers?cursor={nextCursor}&limit=20`
4. Verify response:
   - data.items.length: 5
   - data.hasMore: false

**Expected Result**: Pagination works, 20 items first page, 5 items second page.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 5.6: Followers List Excludes Pending

**Priority**: Medium  
**Type**: Functional

**Preconditions**:
- Chef C authenticated (private account)
- Chef C has 5 accepted followers
- Chef C has 3 pending follow requests

**Test Steps**:
1. Call `GET /social/followers` with Chef C's JWT
2. Verify response:
   - data.items.length: 5 (only accepted, excludes pending)

**Expected Result**: Only accepted followers shown, pending excluded.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

## Search Tests

### Test 6.1: Search by Full Name

**Priority**: High  
**Type**: Functional

**Preconditions**:
- User A authenticated
- Database has users with fullName containing "Maria" (Maria Chef, Maria Smith)

**Test Steps**:
1. Call `GET /social/search?query=Maria` with User A's JWT
2. Verify response:
   - status: 200
   - data.items includes users with fullName ILIKE '%Maria%'
   - data.items[0].fullName: 'Maria Chef' (or similar)

**Expected Result**: Search returns users matching full name.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 6.2: Search by Username

**Priority**: High  
**Type**: Functional

**Preconditions**:
- User A authenticated
- Database has users with username containing "chef" (chef_maria, chef_carlos)

**Test Steps**:
1. Call `GET /social/search?query=chef` with User A's JWT
2. Verify response:
   - data.items includes users with username ILIKE '%chef%'

**Expected Result**: Search returns users matching username.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 6.3: Search by Chef Business Name

**Priority**: High  
**Type**: Functional

**Preconditions**:
- User A authenticated
- Chef with businessName: "Maria's Kitchen"

**Test Steps**:
1. Call `GET /social/search?query=Kitchen` with User A's JWT
2. Verify response:
   - data.items includes chef with businessName ILIKE '%Kitchen%'
   - data.items[0].isChef: true
   - data.items[0].chefBusinessName: "Maria's Kitchen"

**Expected Result**: Search returns chefs matching business name.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 6.4: Search Excludes Blocked Users

**Priority**: High  
**Type**: Security

**Preconditions**:
- User A authenticated
- User A blocked User D (username: 'blocked_user')
- Database has user with fullName: "Blocked User"

**Test Steps**:
1. Call `GET /social/search?query=Blocked` with User A's JWT
2. Verify response:
   - data.items does NOT include User D
   - Other users with "Blocked" in name included

**Expected Result**: Blocked users excluded from search results.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 6.5: Search Excludes Users Who Blocked Me

**Priority**: High  
**Type**: Security

**Preconditions**:
- User A authenticated
- User E blocked User A (UserBlock: userId=E, blockedUserId=A)
- User E's fullName: "Enemy User"

**Test Steps**:
1. Call `GET /social/search?query=Enemy` with User A's JWT
2. Verify response:
   - data.items does NOT include User E

**Expected Result**: Users who blocked me excluded from search results.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 6.6: Search with Pagination

**Priority**: Medium  
**Type**: Functional

**Preconditions**:
- User A authenticated
- Database has 25 users with "chef" in username

**Test Steps**:
1. Call `GET /social/search?query=chef&limit=20` with User A's JWT
2. Verify response:
   - data.items.length: 20
   - data.hasMore: true
   - data.nextCursor: ISO date string
3. Call `GET /social/search?query=chef&cursor={nextCursor}&limit=20`
4. Verify response:
   - data.items.length: 5
   - data.hasMore: false

**Expected Result**: Pagination works correctly.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 6.7: Case-Insensitive Search

**Priority**: Medium  
**Type**: Functional

**Preconditions**:
- User A authenticated
- User with fullName: "Maria Chef"

**Test Steps**:
1. Call `GET /social/search?query=maria` with User A's JWT (lowercase)
2. Verify response includes "Maria Chef"
3. Call `GET /social/search?query=MARIA` (uppercase)
4. Verify response includes "Maria Chef"

**Expected Result**: Search is case-insensitive (ILIKE).

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

## Relationship State Tests

### Test 7.1: Get Relationship with Public User (Following)

**Priority**: High  
**Type**: Functional

**Preconditions**:
- User A authenticated
- User A following User B (status='accepted')
- User B NOT following User A
- User B is public (isPrivate=false)
- No blocks

**Test Steps**:
1. Call `GET /social/relationship/:userBId` with User A's JWT
2. Verify response:
   - data.isFollowing: true
   - data.isFollowedBy: false
   - data.isRequested: false
   - data.isPrivate: false
   - data.isBlockedByMe: false
   - data.hasBlockedMe: false

**Expected Result**: Correct relationship flags.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 7.2: Get Relationship with Private User (Pending Request)

**Priority**: High  
**Type**: Functional

**Preconditions**:
- User A authenticated
- User A sent follow request to Chef C (status='pending')
- Chef C is private (isPrivate=true)

**Test Steps**:
1. Call `GET /social/relationship/:chefCId` with User A's JWT
2. Verify response:
   - data.isFollowing: false (not accepted yet)
   - data.isRequested: true (pending request)
   - data.isPrivate: true

**Expected Result**: isRequested=true for pending follow request.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 7.3: Get Relationship with Mutual Follow

**Priority**: Medium  
**Type**: Functional

**Preconditions**:
- User A authenticated
- User A following User B (status='accepted')
- User B following User A (status='accepted')

**Test Steps**:
1. Call `GET /social/relationship/:userBId` with User A's JWT
2. Verify response:
   - data.isFollowing: true
   - data.isFollowedBy: true

**Expected Result**: Both flags true for mutual follow.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 7.4: Get Relationship with Blocked User

**Priority**: High  
**Type**: Security

**Preconditions**:
- User A authenticated
- User A blocked User D

**Test Steps**:
1. Call `GET /social/relationship/:userDId` with User A's JWT
2. Verify response:
   - data.isFollowing: false (follows removed on block)
   - data.isBlockedByMe: true
   - data.hasBlockedMe: false

**Expected Result**: Block status reflected in relationship.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

## Reel Sharing Tests

### Test 8.1: Share Reel (Copy Link)

**Priority**: High  
**Type**: Functional

**Preconditions**:
- User A authenticated
- Reel R exists (shareCount=0, copyLinkCount=0)

**Test Steps**:
1. Call `POST /reels/:reelRId/share` with body `{ "type": "copy-link" }` and User A's JWT
2. Verify response:
   - status: 200
   - data.type: 'copy-link'
   - data.updatedShareCount: 1
   - data.deepLink: { short, universal, app }
3. Verify database:
   - Reel.stats.shareCount: 1
   - Reel.stats.copyLinkCount: 1

**Expected Result**: Share recorded, counters incremented, deep link returned.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 8.2: Share Reel (External)

**Priority**: High  
**Type**: Functional

**Preconditions**:
- User A authenticated
- Reel R exists (shareCount=1, externalShareCount=0)

**Test Steps**:
1. Call `POST /reels/:reelRId/share` with body `{ "type": "external" }` and User A's JWT
2. Verify response:
   - data.updatedShareCount: 2
3. Verify database:
   - Reel.stats.shareCount: 2
   - Reel.stats.externalShareCount: 1

**Expected Result**: External share counter incremented.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 8.3: Share Reel Directly to User

**Priority**: High  
**Type**: Functional

**Preconditions**:
- User A authenticated
- Reel R exists
- User E exists (not blocked)

**Test Steps**:
1. Call `POST /reels/:reelRId/share` with body `{ "type": "direct", "targetUserId": "userEId" }` and User A's JWT
2. Verify response:
   - status: 200
   - data.type: 'direct'
3. Verify database:
   - Reel.stats.directShareCount: 1
4. Verify notification sent (TODO):
   - Type: REEL_SHARED_DIRECT
   - Recipient: User E

**Expected Result**: Direct share recorded, notification sent.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 8.4: Direct Share Missing Target User

**Priority**: High  
**Type**: Validation

**Preconditions**:
- User A authenticated
- Reel R exists

**Test Steps**:
1. Call `POST /reels/:reelRId/share` with body `{ "type": "direct" }` (missing targetUserId) and User A's JWT
2. Verify response:
   - status: 400
   - errorCode: 'SHARE_TARGET_USER_REQUIRED'

**Expected Result**: 400 error, targetUserId required for direct share.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 8.5: Direct Share to Blocked User

**Priority**: High  
**Type**: Security

**Preconditions**:
- User A authenticated
- Reel R exists
- User A blocked User D

**Test Steps**:
1. Call `POST /reels/:reelRId/share` with body `{ "type": "direct", "targetUserId": "userDId" }` and User A's JWT
2. Verify response:
   - status: 403
   - errorCode: 'SHARE_USER_BLOCKED'

**Expected Result**: Share prevented due to block.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 8.6: Prevent Self-Share

**Priority**: Medium  
**Type**: Validation

**Preconditions**:
- User A authenticated
- Reel R exists

**Test Steps**:
1. Call `POST /reels/:reelRId/share` with body `{ "type": "direct", "targetUserId": "userAId" }` (share to self) and User A's JWT
2. Verify response:
   - status: 400
   - errorCode: 'SHARE_SELF_NOT_ALLOWED'

**Expected Result**: 400 error, cannot share to self.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 8.7: Share Rate Limiting

**Priority**: High  
**Type**: Performance

**Preconditions**:
- User A authenticated
- Reel R exists

**Test Steps**:
1. Call `POST /reels/:reelRId/share` 20 times rapidly (within 60 seconds)
2. Verify first 20 succeed (200 status)
3. Call 21st share within 60 seconds
4. Verify response:
   - status: 429
   - errorCode: 'SHARE_RATE_LIMIT_EXCEEDED'

**Expected Result**: First 20 succeed, 21st fails with 429.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 8.8: Get Share Statistics

**Priority**: Medium  
**Type**: Functional

**Preconditions**:
- Reel R exists
- shareCount=10, copyLinkCount=3, externalShareCount=5, directShareCount=2, storyShareCount=0

**Test Steps**:
1. Call `GET /reels/:reelRId/share/stats` with User A's JWT
2. Verify response:
   - data.shareCount: 10
   - data.copyLinkCount: 3
   - data.externalShareCount: 5
   - data.directShareCount: 2
   - data.storyShareCount: 0

**Expected Result**: Correct share stats breakdown.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 8.9: Get Targetable Users

**Priority**: Medium  
**Type**: Functional

**Preconditions**:
- User A authenticated
- User A following 5 users
- User A has 3 followers (not mutual)

**Test Steps**:
1. Call `GET /reels/:reelRId/share/targetable-users` with User A's JWT
2. Verify response:
   - data.items.length: 8 (5 following + 3 followers)
   - data.items include relationship status (isFollowing, isFollowedBy)
   - Excludes blocked users

**Expected Result**: List contains followers + following (excluding blocks).

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

## Cache Tests

### Test 9.1: Cache Invalidation on Follow

**Priority**: Medium  
**Type**: Performance

**Preconditions**:
- User A authenticated
- User B exists
- Cache populated: social:profile:A, social:privacy:A

**Test Steps**:
1. Call `POST /social/follow/:userBId` with User A's JWT
2. Verify cache keys deleted:
   - social:profile:A
   - social:profile:B
   - social:privacy:A
   - social:privacy:B
3. Call `GET /social/profile/:userAId` → Verify cache miss, DB query, cache repopulated

**Expected Result**: Cache invalidated on follow, repopulated on next read.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 9.2: Cache Invalidation on Block

**Priority**: Medium  
**Type**: Performance

**Preconditions**:
- User A authenticated
- User D exists
- Cache populated for both users

**Test Steps**:
1. Call `POST /social/block/:userDId` with User A's JWT
2. Verify cache keys deleted for both users
3. Verify next profile fetch repopulates cache

**Expected Result**: Cache invalidated on block.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 9.3: Cache Invalidation on Privacy Toggle

**Priority**: Low  
**Type**: Performance

**Preconditions**:
- User A authenticated
- Cache populated: social:privacy:A

**Test Steps**:
1. Call `POST /social/profile/privacy` with body `{ "isPrivate": true }` and User A's JWT
2. Verify cache key deleted: social:privacy:A
3. Call `GET /social/profile/privacy` → Verify cache miss, DB query, cache repopulated

**Expected Result**: Cache invalidated on privacy toggle.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

## Notification Tests

### Test 10.1: FOLLOWED Notification (Public Account)

**Priority**: High  
**Type**: Integration

**Preconditions**:
- User A authenticated
- User B exists (isPrivate=false)

**Test Steps**:
1. Call `POST /social/follow/:userBId` with User A's JWT
2. Verify notification created:
   - Type: 'FOLLOWED'
   - Recipient: User B
   - ActorId: User A
   - Context: isPending=false

**Expected Result**: FOLLOWED notification sent to User B.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 10.2: FOLLOWED Notification (Private Account, Pending)

**Priority**: High  
**Type**: Integration

**Preconditions**:
- User A authenticated
- Chef C exists (isPrivate=true)

**Test Steps**:
1. Call `POST /social/follow/:chefCId` with User A's JWT
2. Verify notification created:
   - Type: 'FOLLOWED'
   - Recipient: Chef C
   - Context: isPending=true

**Expected Result**: FOLLOWED notification with isPending flag sent to Chef C.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 10.3: FOLLOW_REQUEST_ACCEPTED Notification

**Priority**: High  
**Type**: Integration

**Preconditions**:
- User A sent follow request to Chef C (status='pending')
- Chef C authenticated

**Test Steps**:
1. Call `POST /social/requests/:requestId/accept` with Chef C's JWT
2. Verify notification created:
   - Type: 'FOLLOW_REQUEST_ACCEPTED'
   - Recipient: User A
   - ActorId: Chef C

**Expected Result**: FOLLOW_REQUEST_ACCEPTED notification sent to User A.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 10.4: No Notification on Unfollow

**Priority**: Medium  
**Type**: Integration

**Preconditions**:
- User A authenticated
- User A following User B

**Test Steps**:
1. Call `POST /social/unfollow/:userBId` with User A's JWT
2. Verify NO notification sent to User B

**Expected Result**: No notification on unfollow.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 10.5: No Notification on Block

**Priority**: Medium  
**Type**: Integration

**Preconditions**:
- User A authenticated
- User D exists

**Test Steps**:
1. Call `POST /social/block/:userDId` with User A's JWT
2. Verify NO notification sent to User D (soft block for privacy)

**Expected Result**: No notification on block.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

## Performance Tests

### Test 11.1: Follow 1000 Users (Counter Accuracy)

**Priority**: Medium  
**Type**: Performance

**Preconditions**:
- User A authenticated
- 1000 public users (B1, B2, ..., B1000)

**Test Steps**:
1. Follow all 1000 users sequentially
2. Verify UserPrivacy.followingCount (User A): 1000
3. Verify each user's followersCount incremented by 1

**Expected Result**: Counters accurate after 1000 follows.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 11.2: Follower List Performance (10K Followers)

**Priority**: Medium  
**Type**: Performance

**Preconditions**:
- Chef C has 10,000 accepted followers

**Test Steps**:
1. Call `GET /social/followers/:chefCId?limit=20` with User A's JWT
2. Measure response time
3. Verify response time < 500ms (database index used)

**Expected Result**: Query fast due to index on targetId.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 11.3: Search Performance (1M Users)

**Priority**: Medium  
**Type**: Performance

**Preconditions**:
- Database has 1 million users
- User A authenticated

**Test Steps**:
1. Call `GET /social/search?query=chef&limit=20` with User A's JWT
2. Measure response time
3. Verify response time < 1000ms

**Expected Result**: Search reasonably fast (ILIKE with indexes).

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 11.4: Concurrent Follow Requests (Race Condition)

**Priority**: High  
**Type**: Concurrency

**Preconditions**:
- User A authenticated
- User B exists (isPrivate=false)

**Test Steps**:
1. Send 10 concurrent `POST /social/follow/:userBId` requests with User A's JWT
2. Verify only 1 follow record created (UNIQUE constraint prevents duplicates)
3. Verify 9 requests fail with 'SOCIAL_ALREADY_FOLLOWING'
4. Verify counters incremented by exactly 1

**Expected Result**: UNIQUE constraint prevents duplicate follows, counters accurate.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

## Security Tests

### Test 12.1: Unauthorized Access (No JWT)

**Priority**: Critical  
**Type**: Security

**Preconditions**:
- No authentication

**Test Steps**:
1. Call `POST /social/follow/:userBId` without JWT token
2. Verify response:
   - status: 401
   - error: 'Unauthorized'

**Expected Result**: 401 error, all endpoints require authentication.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 12.2: Invalid JWT Token

**Priority**: Critical  
**Type**: Security

**Preconditions**:
- Invalid JWT: 'invalid-token-12345'

**Test Steps**:
1. Call `POST /social/follow/:userBId` with invalid JWT
2. Verify response:
   - status: 401
   - error: 'Unauthorized'

**Expected Result**: 401 error, invalid JWT rejected.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 12.3: SQL Injection in Search Query

**Priority**: Critical  
**Type**: Security

**Preconditions**:
- User A authenticated

**Test Steps**:
1. Call `GET /social/search?query=' OR 1=1--` with User A's JWT
2. Verify no SQL injection (parameterized query)
3. Verify response: data.items empty or safe results

**Expected Result**: SQL injection prevented (TypeORM parameterized queries).

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 12.4: XSS in Username/Full Name

**Priority**: High  
**Type**: Security

**Preconditions**:
- User with fullName: '<script>alert("XSS")</script>'
- User A authenticated

**Test Steps**:
1. Call `GET /social/search?query=script` with User A's JWT
2. Verify response: fullName returned as-is (backend doesn't render HTML)
3. Verify frontend properly escapes HTML (client-side responsibility)

**Expected Result**: Backend returns raw string, no XSS execution in API layer.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

## Integration Tests

### Test 13.1: Full Follow Flow (Public Account)

**Priority**: High  
**Type**: Integration

**Preconditions**:
- User A, User B authenticated
- No existing relationship

**Test Steps**:
1. User A follows User B (POST /follow/:userBId)
2. Verify relationship (GET /relationship/:userBId): isFollowing=true
3. Verify User B's followers list includes User A
4. Verify User A's following list includes User B
5. User A unfollows User B (POST /unfollow/:userBId)
6. Verify relationship: isFollowing=false
7. Verify lists updated

**Expected Result**: Full follow/unfollow cycle works end-to-end.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 13.2: Full Follow Request Flow (Private Account)

**Priority**: High  
**Type**: Integration

**Preconditions**:
- User A, Chef C authenticated
- Chef C is private

**Test Steps**:
1. User A follows Chef C (POST /follow/:chefCId) → status='pending'
2. Chef C views pending requests (GET /requests/pending) → includes User A
3. Chef C accepts request (POST /requests/:requestId/accept)
4. Verify relationship (GET /relationship/:chefCId): isFollowing=true, isRequested=false
5. Verify counters incremented
6. Verify User A received FOLLOW_REQUEST_ACCEPTED notification

**Expected Result**: Full request flow works end-to-end.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 13.3: Block → Remove Follows → Unblock → Re-Follow

**Priority**: High  
**Type**: Integration

**Preconditions**:
- User A, User D authenticated
- User A following User D (mutual)

**Test Steps**:
1. User A blocks User D (POST /block/:userDId)
2. Verify all follows removed
3. Verify counters decremented
4. User A unblocks User D (POST /unblock/:userDId)
5. User A follows User D again (POST /follow/:userDId)
6. Verify new follow created

**Expected Result**: Block clears follows, unblock allows re-follow.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 13.4: Privacy Toggle Impact on New Follows

**Priority**: Medium  
**Type**: Integration

**Preconditions**:
- Chef C authenticated (public)
- User A authenticated

**Test Steps**:
1. User A follows Chef C → status='accepted' (public)
2. Chef C toggles privacy to private (POST /profile/privacy)
3. User B follows Chef C → status='pending' (now private)
4. Verify User A still following (existing follows unaffected)
5. Chef C toggles back to public
6. User C follows Chef C → status='accepted' (public again)

**Expected Result**: Privacy toggle affects new follows, not existing.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 13.5: Share Reel → Check Stats → Search Target User

**Priority**: Medium  
**Type**: Integration

**Preconditions**:
- User A, User E authenticated
- Reel R exists

**Test Steps**:
1. User A shares reel directly to User E (POST /reels/:reelRId/share)
2. Verify share stats updated (GET /reels/:reelRId/share/stats)
3. User E searches for User A (GET /social/search?query=userA)
4. Verify User A appears in search results

**Expected Result**: Share, stats, and search work together.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

## Regression Tests

### Test 14.1: Follow/Unfollow Counter Drift Detection

**Priority**: High  
**Type**: Regression

**Preconditions**:
- User A authenticated
- User B exists (followersCount=100, followingCount=50)

**Test Steps**:
1. User A follows User B → followersCount=101
2. User A unfollows User B → followersCount=100
3. Repeat 10 times
4. Verify final followersCount=100 (no drift)

**Expected Result**: Counters accurate after multiple follow/unfollow cycles.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 14.2: Block Cleanup Regression (Ensure All Follows Removed)

**Priority**: High  
**Type**: Regression

**Preconditions**:
- User A, User D authenticated
- User A following User D
- User D following User A

**Test Steps**:
1. User A blocks User D (POST /block/:userDId)
2. Query database: Verify NO UserFollow records (followerId=A, targetId=D) OR (followerId=D, targetId=A)
3. Verify counters decremented for both users

**Expected Result**: All follow records cleaned up on block.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 14.3: Search Block Filter Regression (Both Directions)

**Priority**: High  
**Type**: Regression

**Preconditions**:
- User A authenticated
- User A blocked User D
- User E blocked User A

**Test Steps**:
1. Call `GET /social/search?query=user` with User A's JWT
2. Verify User D NOT in results (User A blocked User D)
3. Verify User E NOT in results (User E blocked User A)

**Expected Result**: Bidirectional block filtering works in search.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 14.4: Pagination Consistency (No Phantom Reads)

**Priority**: Medium  
**Type**: Regression

**Preconditions**:
- Chef C has 30 followers
- User A authenticated

**Test Steps**:
1. Call `GET /social/followers/:chefCId?limit=20` with User A's JWT → Save page 1 results
2. Between requests, Chef C gains 5 new followers (total 35)
3. Call `GET /social/followers/:chefCId?cursor={nextCursor}&limit=20` → Save page 2 results
4. Verify no duplicate users between page 1 and page 2 (cursor-based pagination prevents phantom reads)

**Expected Result**: No duplicate results across pages (cursor-based pagination advantage).

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

### Test 14.5: Share Rate Limit Reset After 60 Seconds

**Priority**: Medium  
**Type**: Regression

**Preconditions**:
- User A authenticated
- Reel R exists

**Test Steps**:
1. Share reel 20 times (reach rate limit)
2. Attempt 21st share → 429 error
3. Wait 61 seconds
4. Attempt share again → 200 success (rate limit window reset)

**Expected Result**: Rate limit resets after 60 seconds.

**Actual Result**: [To be filled during testing]

**Status**: ☐ Pass ☐ Fail

---

## Test Summary

### Coverage Metrics

| Category | Total Tests | Critical | High | Medium | Low |
|----------|-------------|----------|------|--------|-----|
| Follow System | 10 | 4 | 6 | 0 | 0 |
| Follow Requests | 5 | 2 | 2 | 1 | 0 |
| Blocking | 8 | 4 | 4 | 0 | 0 |
| Privacy | 4 | 1 | 1 | 1 | 1 |
| Social Lists | 6 | 2 | 2 | 2 | 0 |
| Search | 7 | 3 | 3 | 1 | 0 |
| Relationship State | 4 | 2 | 2 | 0 | 0 |
| Reel Sharing | 9 | 4 | 4 | 1 | 0 |
| Cache | 3 | 0 | 0 | 2 | 1 |
| Notifications | 5 | 0 | 3 | 2 | 0 |
| Performance | 4 | 1 | 1 | 2 | 0 |
| Security | 4 | 3 | 1 | 0 | 0 |
| Integration | 5 | 0 | 3 | 2 | 0 |
| Regression | 5 | 0 | 3 | 2 | 0 |
| **Total** | **79** | **26** | **39** | **13** | **1** |

### Test Execution Checklist

**Pre-Execution**:
- ☐ Test environment setup complete
- ☐ Test data seeded
- ☐ JWT tokens generated
- ☐ Database clean state verified

**Execution**:
- ☐ Functional tests (Pass rate: __%)
- ☐ Validation tests (Pass rate: __%)
- ☐ Security tests (Pass rate: __%)
- ☐ Performance tests (Pass rate: __%)
- ☐ Integration tests (Pass rate: __%)
- ☐ Regression tests (Pass rate: __%)

**Post-Execution**:
- ☐ Test results documented
- ☐ Bugs logged (if any)
- ☐ Performance metrics recorded
- ☐ Regression suite updated

---

**[SLICE_COMPLETE ✅]**
