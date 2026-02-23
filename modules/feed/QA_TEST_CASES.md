# Feed Module - QA Test Cases

**Version:** 1.0  
**Last Updated:** February 14, 2026  
**Module:** `apps/chefooz-apis/src/modules/feed/`  
**Test Environment:** Staging (staging.chefooz.app)  
**Total Test Cases:** 89

---

## 📋 Table of Contents

1. [Test Overview](#test-overview)
2. [Feed Retrieval Tests](#feed-retrieval-tests)
3. [Advanced Ranking Tests](#advanced-ranking-tests)
4. [Engagement Tracking Tests](#engagement-tracking-tests)
5. [Following Filter Tests](#following-filter-tests)
6. [Order Context Tests](#order-context-tests)
7. [Cache Tests](#cache-tests)
8. [Abuse Prevention Tests](#abuse-prevention-tests)
9. [Security Tests](#security-tests)
10. [Performance Tests](#performance-tests)
11. [Integration Tests](#integration-tests)
12. [Regression Tests](#regression-tests)

---

## Test Overview

### Testing Scope

| Category | Test Cases | Priority | Status |
|----------|-----------|----------|--------|
| **Feed Retrieval** | 12 | HIGH | ✅ Ready |
| **Advanced Ranking** | 10 | HIGH | ✅ Ready |
| **Engagement Tracking** | 15 | HIGH | ✅ Ready |
| **Following Filter** | 8 | MEDIUM | ✅ Ready |
| **Order Context** | 10 | HIGH | ✅ Ready |
| **Cache** | 8 | MEDIUM | ✅ Ready |
| **Abuse Prevention** | 10 | HIGH | ✅ Ready |
| **Security** | 6 | HIGH | ✅ Ready |
| **Performance** | 4 | MEDIUM | ✅ Ready |
| **Integration** | 2 | MEDIUM | ✅ Ready |

### Test Data Requirements

#### Prerequisites
1. **Users**: 50+ test users with varying reputation tiers (Bronze, Silver, Gold, Diamond, Legend)
2. **Reels**: 100+ test reels with varying stats (views, likes, saves)
3. **Orders**: 20+ test orders with menu items
4. **Follows**: 30+ follow relationships (accepted status)
5. **Reputation Data**: CRS reputation records for all test users

#### Test User Accounts

| Username | Password | Tier | feedBoostWeight | creatorBoostState | Purpose |
|----------|----------|------|----------------|-------------------|---------|
| `test_guest` | N/A | N/A | N/A | N/A | Guest (no auth) |
| `test_bronze` | `Test123!` | Bronze | 0.6 | ACTIVE | Low reputation |
| `test_silver` | `Test123!` | Silver | 1.0 | ACTIVE | Medium reputation |
| `test_gold` | `Test123!` | Gold | 1.5 | ACTIVE | High reputation |
| `test_diamond` | `Test123!` | Diamond | 1.8 | ACTIVE | Elite reputation |
| `test_legend` | `Test123!` | Legend | 2.0 | ACTIVE | Top reputation |
| `test_flagged` | `Test123!` | Silver | 1.0 | FLAGGED_BOOSTING | Gaming detected |
| `test_suspended` | `Test123!` | Bronze | 0.5 | SUSPENDED | Severe violation |

### API Base URLs

| Environment | Base URL | Notes |
|-------------|----------|-------|
| **Development** | `https://api-staging.chefooz.com` | Local testing |
| **Staging** | `https://staging-api.chefooz.app` | QA environment |
| **Production** | `https://api.chefooz.app` | Live environment (CAUTION) |

### Test Data Cleanup

**After Each Test Suite**:
```bash
# Clean test data
POST /api/v1/feed/admin/reseed
Authorization: Bearer <admin_token>

# Clear Redis cache
redis-cli FLUSHDB

# Reset engagement records
DELETE FROM engagements WHERE user_id LIKE 'test_%'
```

---

## Feed Retrieval Tests

### TC-FEED-001: Guest User Fetches First Page (Default Ranking)

**Priority**: HIGH  
**Type**: Functional  
**Prerequisites**: 50+ reels exist in database

**Test Steps**:
1. Send request without auth token:
   ```bash
   GET /api/v1/feed
   ```

**Expected Results**:
- ✅ Status: 200 OK
- ✅ Response structure:
  ```json
  {
    "success": true,
    "message": "Feed retrieved successfully",
    "data": {
      "items": [...], // Array of 10 reels (default limit)
      "nextCursor": "65f3b1c4..." // MongoDB ObjectId
    }
  }
  ```
- ✅ Items sorted by ranking score (highest first)
- ✅ `isLiked` and `isSaved` are false (guest user)
- ✅ Response time < 200ms (cache miss) or < 50ms (cache hit)

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-FEED-002: Authenticated User Fetches First Page (Personalized)

**Priority**: HIGH  
**Type**: Functional  
**Prerequisites**: User account `test_bronze` exists, has liked 2 reels

**Test Steps**:
1. Login as `test_bronze`, obtain JWT token
2. Send request with auth token:
   ```bash
   GET /api/v1/feed
   Authorization: Bearer <jwt_token>
   ```

**Expected Results**:
- ✅ Status: 200 OK
- ✅ Response contains 10 reels
- ✅ `isLiked` is true for 2 reels (previously liked)
- ✅ `isSaved` is false for all reels (not saved yet)
- ✅ `author` object populated for each reel
- ✅ Response time < 200ms

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-FEED-003: Fetch Second Page (Cursor Pagination)

**Priority**: HIGH  
**Type**: Functional  
**Prerequisites**: TC-FEED-001 passed, nextCursor obtained

**Test Steps**:
1. From TC-FEED-001 response, extract `nextCursor`
2. Send request with cursor:
   ```bash
   GET /api/v1/feed?cursor=65f3b1c4...&limit=10
   ```

**Expected Results**:
- ✅ Status: 200 OK
- ✅ Response contains 10 different reels (not from page 1)
- ✅ First reel `_id` < cursor value (cursor pagination working)
- ✅ `nextCursor` is populated (more pages exist) OR null (last page)
- ✅ No cache used (cursor bypasses cache)
- ✅ Response time < 300ms

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-FEED-004: Fetch Feed with Custom Limit (20 items)

**Priority**: MEDIUM  
**Type**: Functional  
**Prerequisites**: None

**Test Steps**:
1. Send request with limit=20:
   ```bash
   GET /api/v1/feed?limit=20
   ```

**Expected Results**:
- ✅ Status: 200 OK
- ✅ Response contains exactly 20 reels
- ✅ Items sorted by ranking score
- ✅ Response time < 250ms

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-FEED-005: Fetch Feed with Maximum Limit (30 items)

**Priority**: MEDIUM  
**Type**: Boundary  
**Prerequisites**: 50+ reels exist

**Test Steps**:
1. Send request with limit=30:
   ```bash
   GET /api/v1/feed?limit=30
   ```

**Expected Results**:
- ✅ Status: 200 OK
- ✅ Response contains exactly 30 reels
- ✅ Response time < 300ms

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-FEED-006: Fetch Feed with Limit > 30 (Clamped)

**Priority**: MEDIUM  
**Type**: Boundary  
**Prerequisites**: None

**Test Steps**:
1. Send request with limit=50:
   ```bash
   GET /api/v1/feed?limit=50
   ```

**Expected Results**:
- ✅ Status: 200 OK
- ✅ Response contains exactly 30 reels (clamped to max)
- ✅ No validation error (automatic clamping)

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-FEED-007: Fetch Feed with Limit < 1 (Invalid)

**Priority**: LOW  
**Type**: Negative  
**Prerequisites**: None

**Test Steps**:
1. Send request with limit=0:
   ```bash
   GET /api/v1/feed?limit=0
   ```

**Expected Results**:
- ✅ Status: 400 Bad Request
- ✅ Error message: "Validation failed"
- ✅ Error details: "limit must not be less than 1"

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-FEED-008: Fetch Trending Feed

**Priority**: HIGH  
**Type**: Functional  
**Prerequisites**: Reels with varying like counts exist

**Test Steps**:
1. Send request with sort=TRENDING:
   ```bash
   GET /api/v1/feed?sort=TRENDING&limit=10
   ```

**Expected Results**:
- ✅ Status: 200 OK
- ✅ Items sorted by `stats.likes DESC`, then `createdAt DESC`
- ✅ First reel has highest like count
- ✅ Response time < 150ms

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-FEED-009: Fetch Recent Feed

**Priority**: HIGH  
**Type**: Functional  
**Prerequisites**: Reels with varying creation times exist

**Test Steps**:
1. Send request with sort=RECENT:
   ```bash
   GET /api/v1/feed?sort=RECENT&limit=10
   ```

**Expected Results**:
- ✅ Status: 200 OK
- ✅ Items sorted by `createdAt DESC` (chronological)
- ✅ First reel is most recent
- ✅ Response time < 150ms

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-FEED-010: Fetch Feed with Invalid Cursor

**Priority**: MEDIUM  
**Type**: Negative  
**Prerequisites**: None

**Test Steps**:
1. Send request with invalid cursor:
   ```bash
   GET /api/v1/feed?cursor=invalid_cursor_123
   ```

**Expected Results**:
- ✅ Status: 400 Bad Request (TODO: Add @IsMongoId validation)
- ✅ Error message: "Invalid cursor format"

**Current Behavior** (without validation):
- ⚠️ Status: 200 OK (returns empty feed, no error)

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-FEED-011: Fetch Feed with Location (Placeholder)

**Priority**: LOW  
**Type**: Functional  
**Prerequisites**: None

**Test Steps**:
1. Send request with lat/lng:
   ```bash
   GET /api/v1/feed?lat=12.9716&lng=77.5946&limit=10
   ```

**Expected Results**:
- ✅ Status: 200 OK
- ✅ Response contains 10 reels
- ⚠️ Location not used in ranking (placeholder feature)
- ✅ Cache key includes lat/lng

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-FEED-012: Fetch Feed with No Reels (Empty State)

**Priority**: MEDIUM  
**Type**: Edge Case  
**Prerequisites**: Empty database (run reseed script, then delete all reels)

**Test Steps**:
1. Send request:
   ```bash
   GET /api/v1/feed
   ```

**Expected Results**:
- ✅ Status: 200 OK
- ✅ Response:
  ```json
  {
    "success": true,
    "message": "Feed retrieved successfully",
    "data": {
      "items": [],
      "nextCursor": null
    }
  }
  ```

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

## Advanced Ranking Tests

### TC-RANK-001: Viral Reel Ranks Higher (High Engagement)

**Priority**: HIGH  
**Type**: Functional  
**Prerequisites**: 2 reels exist:
- Reel A: 1000 likes, 10000 views, 200 saves, 6 hours old
- Reel B: 10 likes, 100 views, 5 saves, 1 hour old (more recent)

**Test Steps**:
1. Fetch feed:
   ```bash
   GET /api/v1/feed?sort=DEFAULT&limit=10
   ```
2. Find positions of Reel A and Reel B in response

**Expected Results**:
- ✅ Reel A appears before Reel B (higher engagement wins despite age)
- ✅ Reel A score > Reel B score

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-RANK-002: Legend Creator Gets Reputation Boost

**Priority**: HIGH  
**Type**: Functional  
**Prerequisites**: 2 reels with identical stats:
- Reel A: From `test_legend` (feedBoostWeight=2.0)
- Reel B: From `test_bronze` (feedBoostWeight=0.6)
- Both: 50 likes, 500 views, 20 saves, created at same time

**Test Steps**:
1. Fetch feed:
   ```bash
   GET /api/v1/feed?sort=DEFAULT&limit=10
   ```
2. Find positions of Reel A and Reel B

**Expected Results**:
- ✅ Reel A (Legend) appears before Reel B (Bronze)
- ✅ Reel A gets +200 reputation boost (2.0 × 100)
- ✅ Reel B gets +60 reputation boost (0.6 × 100)

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-RANK-003: Promoted Reel Gets Boost

**Priority**: HIGH  
**Type**: Functional  
**Prerequisites**: 2 reels with identical stats:
- Reel A: promoted=true
- Reel B: promoted=false
- Both from same creator, created at same time

**Test Steps**:
1. Fetch feed:
   ```bash
   GET /api/v1/feed?sort=DEFAULT&limit=10
   ```
2. Find positions of Reel A and Reel B

**Expected Results**:
- ✅ Reel A (promoted) appears before Reel B
- ✅ Reel A gets +50 promoted boost (from env config)

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-RANK-004: Flagged Creator Gets Visibility Penalty

**Priority**: HIGH  
**Type**: Functional  
**Prerequisites**: 2 reels with identical stats:
- Reel A: From `test_flagged` (creatorBoostState=FLAGGED_BOOSTING)
- Reel B: From `test_silver` (creatorBoostState=ACTIVE)
- Both: Same engagement, same creation time

**Test Steps**:
1. Fetch feed:
   ```bash
   GET /api/v1/feed?sort=DEFAULT&limit=10
   ```
2. Find positions of Reel A and Reel B

**Expected Results**:
- ✅ Reel B (ACTIVE) appears before Reel A (FLAGGED)
- ✅ Reel A gets 0.3 visibility multiplier (70% penalty)
- ✅ Server logs show boost penalty event

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-RANK-005: Suspended Creator Hidden from Feed

**Priority**: HIGH  
**Type**: Functional  
**Prerequisites**: Reel exists from `test_suspended` (creatorBoostState=SUSPENDED)

**Test Steps**:
1. Fetch feed:
   ```bash
   GET /api/v1/feed?sort=DEFAULT&limit=50
   ```
2. Search for reel from `test_suspended`

**Expected Results**:
- ✅ Reel from `test_suspended` NOT in response (hidden)
- ✅ Visibility multiplier = 0.0
- ✅ Server logs show suppression event

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-RANK-006: Recency Decay Applied (48h Half-Life)

**Priority**: MEDIUM  
**Type**: Functional  
**Prerequisites**: 3 reels with identical engagement:
- Reel A: 12 hours old
- Reel B: 48 hours old
- Reel C: 96 hours old

**Test Steps**:
1. Fetch feed:
   ```bash
   GET /api/v1/feed?sort=DEFAULT&limit=10
   ```
2. Find positions of Reel A, B, C

**Expected Results**:
- ✅ Order: A → B → C (newer first)
- ✅ Reel A decay factor: ~0.84 (16% decay)
- ✅ Reel B decay factor: 0.5 (50% decay, half-life)
- ✅ Reel C decay factor: 0.25 (75% decay)

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-RANK-007: Diamond Creator Gets Enhanced Reputation Boost

**Priority**: HIGH  
**Type**: Functional  
**Prerequisites**: 2 reels with identical stats:
- Reel A: From `test_diamond` (feedBoostWeight=1.8)
- Reel B: From `test_silver` (feedBoostWeight=1.0)
- Both: 50 likes, 500 views, 20 saves, created at same time

**Test Steps**:
1. Fetch feed:
   ```bash
   GET /api/v1/feed?sort=DEFAULT&limit=10
   ```
2. Find positions of Reel A and Reel B

**Expected Results**:
- ✅ Reel A (Diamond) appears before Reel B (Silver)
- ✅ Reel A gets +180 reputation boost (1.8 × 100)
- ✅ Reel A gets +7% visibility bonus
- ✅ Reel B gets +100 reputation boost (1.0 × 100)
- ✅ Reel B gets +3% visibility bonus

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-RANK-008: Legend Creator Gets Maximum Boost

**Priority**: HIGH  
**Type**: Functional  
**Prerequisites**: 2 reels with identical stats:
- Reel A: From `test_legend` (feedBoostWeight=2.0)
- Reel B: From `test_gold` (feedBoostWeight=1.5)
- Both: 50 likes, 500 views, 20 saves, created at same time

**Test Steps**:
1. Fetch feed:
   ```bash
   GET /api/v1/feed?sort=DEFAULT&limit=10
   ```
2. Find positions of Reel A and Reel B

**Expected Results**:
- ✅ Reel A (Legend) appears before Reel B (Gold)
- ✅ Reel A gets +200 reputation boost (2.0 × 100, maximum)
- ✅ Reel A gets +10% visibility bonus (maximum)
- ✅ Reel B gets +150 reputation boost (1.5 × 100)
- ✅ Reel B gets +5% visibility bonus

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-RANK-009: Saves Weighted Higher Than Likes

**Priority**: MEDIUM  
**Type**: Functional  
**Prerequisites**: 2 reels:
- Reel A: 100 likes, 1000 views, 10 saves
- Reel B: 50 likes, 1000 views, 30 saves (more saves)
- Both created at same time

**Test Steps**:
1. Calculate expected scores:
   - Reel A: (100×10 + 1000×1 + 10×15) = 2150
   - Reel B: (50×10 + 1000×1 + 30×15) = 1950
2. Fetch feed and verify order

**Expected Results**:
- ✅ Reel A appears before Reel B (likes weight wins)
- ✅ If Reel B had 35 saves: 2000 points, still loses
- ✅ If Reel B had 40 saves: 2100 points, wins (saves × 15 = powerful)

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-RANK-008: Score Clamped to 1000

**Priority**: LOW  
**Type**: Boundary  
**Prerequisites**: Viral reel with huge engagement:
- 10,000 likes, 100,000 views, 5,000 saves, 1 hour old
- Expected raw score > 1000

**Test Steps**:
1. Calculate expected score:
   - Base: (10000×10 + 100000×1 + 5000×15) × 0.986 = 247,150
   - With boosts: > 1000
2. Fetch feed and verify reel appears

**Expected Results**:
- ✅ Reel appears in feed (score clamped to 1000, not hidden)
- ✅ No overflow errors
- ✅ Logs show score clamped to 1000

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-RANK-009: Candidate Pool Size (5× Multiplier)

**Priority**: MEDIUM  
**Type**: Performance  
**Prerequisites**: 200+ reels exist

**Test Steps**:
1. Fetch feed with limit=10:
   ```bash
   GET /api/v1/feed?limit=10
   ```
2. Monitor MongoDB query logs (candidateLimit)

**Expected Results**:
- ✅ MongoDB fetches 50 reels (5 × 10)
- ✅ Service calculates scores for 50 reels
- ✅ Service returns top 10 by score
- ✅ Response time < 200ms

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-RANK-010: Batch Reputation Fetch (N+1 Prevention)

**Priority**: MEDIUM  
**Type**: Performance  
**Prerequisites**: 50 reels from 20 different creators

**Test Steps**:
1. Fetch feed:
   ```bash
   GET /api/v1/feed?limit=10
   ```
2. Monitor PostgreSQL query logs

**Expected Results**:
- ✅ Only 1 SQL query to fetch reputation data (batch IN query)
- ✅ No N+1 query pattern (50 individual queries)
- ✅ Query uses index on `user_id`

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

## Engagement Tracking Tests

### TC-ENGAGE-001: Guest User Views Reel (No Auth)

**Priority**: HIGH  
**Type**: Functional  
**Prerequisites**: Reel with id `65f3b1c...` exists

**Test Steps**:
1. Send engagement request without auth:
   ```bash
   POST /api/v1/feed/engagement
   Content-Type: application/json
   {
     "reelId": "65f3b1c4f8d9e2a1b3c4d5e6",
     "action": "VIEW"
   }
   ```

**Expected Results**:
- ✅ Status: 200 OK
- ✅ Response:
  ```json
  {
    "success": true,
    "message": "Engagement recorded successfully",
    "data": {
      "reelId": "65f3b1c4...",
      "stats": {
        "views": <incremented>,
        "likes": <unchanged>,
        "comments": <unchanged>,
        "saves": <unchanged>,
        "shareCount": <unchanged>
      }
    }
  }
  ```
- ✅ Reel `stats.views` incremented by 1 in database
- ✅ No engagement record created in MongoDB

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-ENGAGE-002: View Throttle Enforced (3 Seconds)

**Priority**: HIGH  
**Type**: Functional  
**Prerequisites**: TC-ENGAGE-001 passed

**Test Steps**:
1. Send first VIEW request (succeeds)
2. Immediately send second VIEW request (within 3 seconds)

**Expected Results**:
- ✅ First request: 200 OK
- ✅ Second request: 429 Too Many Requests
- ✅ Error response:
  ```json
  {
    "success": false,
    "message": "View rate limited. Please wait.",
    "errorCode": "FEED_RATE_LIMITED"
  }
  ```
- ✅ Reel `stats.views` incremented only once

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-ENGAGE-003: View Allowed After 3 Seconds

**Priority**: MEDIUM  
**Type**: Functional  
**Prerequisites**: TC-ENGAGE-002 passed

**Test Steps**:
1. Wait 3 seconds after throttle violation
2. Send VIEW request again

**Expected Results**:
- ✅ Status: 200 OK
- ✅ Reel `stats.views` incremented again
- ✅ Throttle reset

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-ENGAGE-004: Authenticated User Likes Reel

**Priority**: HIGH  
**Type**: Functional  
**Prerequisites**: User `test_bronze` logged in, reel `65f3b1c...` not liked yet

**Test Steps**:
1. Login as `test_bronze`, obtain JWT token
2. Send LIKE request:
   ```bash
   POST /api/v1/feed/engagement
   Authorization: Bearer <jwt_token>
   Content-Type: application/json
   {
     "reelId": "65f3b1c4f8d9e2a1b3c4d5e6",
     "action": "LIKE"
   }
   ```

**Expected Results**:
- ✅ Status: 200 OK
- ✅ Response shows incremented `stats.likes`
- ✅ Engagement record created in MongoDB:
  - `userId`: `test_bronze` UUID
  - `reelId`: `65f3b1c...`
  - `type`: `like`
  - `active`: true
- ✅ Redis set updated: `reel:65f3b1c...:likes` contains userId
- ✅ Notification sent to reel owner (if not self-like)
- ✅ Feed cache invalidated (pub/sub event)

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-ENGAGE-005: Like Idempotency (Duplicate Like)

**Priority**: HIGH  
**Type**: Functional  
**Prerequisites**: TC-ENGAGE-004 passed (reel already liked)

**Test Steps**:
1. Send LIKE request again (duplicate):
   ```bash
   POST /api/v1/feed/engagement
   Authorization: Bearer <jwt_token>
   Content-Type: application/json
   {
     "reelId": "65f3b1c4f8d9e2a1b3c4d5e6",
     "action": "LIKE"
   }
   ```

**Expected Results**:
- ✅ Status: 200 OK (idempotent, not 409 Conflict)
- ✅ Response shows same `stats.likes` (not incremented again)
- ✅ No new engagement record created
- ✅ No duplicate notification sent

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-ENGAGE-006: Like Without Auth (401 Error)

**Priority**: HIGH  
**Type**: Security  
**Prerequisites**: None

**Test Steps**:
1. Send LIKE request without auth token:
   ```bash
   POST /api/v1/feed/engagement
   Content-Type: application/json
   {
     "reelId": "65f3b1c4f8d9e2a1b3c4d5e6",
     "action": "LIKE"
   }
   ```

**Expected Results**:
- ✅ Status: 401 Unauthorized
- ✅ Error response:
  ```json
  {
    "success": false,
    "message": "Authentication required",
    "errorCode": "FEED_AUTH_REQUIRED"
  }
  ```
- ✅ No engagement record created
- ✅ Reel `stats.likes` unchanged

**Current Behavior** (dev mode with fallback):
- ⚠️ Status: 200 OK (fallback userId used)
- ⚠️ TODO: Remove fallback for production

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-ENGAGE-007: User Saves Reel

**Priority**: HIGH  
**Type**: Functional  
**Prerequisites**: User `test_bronze` logged in, reel `65f3b1c...` not saved yet

**Test Steps**:
1. Send SAVE request:
   ```bash
   POST /api/v1/feed/engagement
   Authorization: Bearer <jwt_token>
   Content-Type: application/json
   {
     "reelId": "65f3b1c4f8d9e2a1b3c4d5e6",
     "action": "SAVE"
   }
   ```

**Expected Results**:
- ✅ Status: 200 OK
- ✅ Response shows incremented `stats.saves`
- ✅ Engagement record created (type='save')
- ✅ Redis set updated: `reel:65f3b1c...:saves` contains userId
- ✅ Feed cache invalidated
- ✅ No notification sent (save is private action)

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-ENGAGE-008: Save Idempotency (Duplicate Save)

**Priority**: MEDIUM  
**Type**: Functional  
**Prerequisites**: TC-ENGAGE-007 passed (reel already saved)

**Test Steps**:
1. Send SAVE request again (duplicate)

**Expected Results**:
- ✅ Status: 200 OK (idempotent)
- ✅ Response shows same `stats.saves` (not incremented)
- ✅ No new engagement record created

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-ENGAGE-009: Engagement on Non-Existent Reel (404 Error)

**Priority**: MEDIUM  
**Type**: Negative  
**Prerequisites**: None

**Test Steps**:
1. Send VIEW request with invalid reel ID:
   ```bash
   POST /api/v1/feed/engagement
   Content-Type: application/json
   {
     "reelId": "000000000000000000000000",
     "action": "VIEW"
   }
   ```

**Expected Results**:
- ✅ Status: 404 Not Found
- ✅ Error response:
  ```json
  {
    "success": false,
    "message": "Reel not found",
    "errorCode": "FEED_REEL_NOT_FOUND"
  }
  ```

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-ENGAGE-010: Invalid Engagement Action (400 Error)

**Priority**: LOW  
**Type**: Negative  
**Prerequisites**: None

**Test Steps**:
1. Send request with invalid action:
   ```bash
   POST /api/v1/feed/engagement
   Content-Type: application/json
   {
     "reelId": "65f3b1c4f8d9e2a1b3c4d5e6",
     "action": "INVALID_ACTION"
   }
   ```

**Expected Results**:
- ✅ Status: 400 Bad Request
- ✅ Error message: "action must be a valid enum value"

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-ENGAGE-011: Like Notification Sent to Reel Owner

**Priority**: MEDIUM  
**Type**: Integration  
**Prerequisites**: User A likes User B's reel

**Test Steps**:
1. User A sends LIKE request on User B's reel
2. Check User B's notification inbox

**Expected Results**:
- ✅ Notification created with type `REEL_LIKED`
- ✅ Notification metadata includes:
  - `reelId`: Liked reel ID
  - `username`: User A's username
  - `reelThumbnail`: Reel thumbnail URL
- ✅ Push notification sent to User B's device
- ✅ Activity feed entry created

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-ENGAGE-012: Self-Like No Notification

**Priority**: LOW  
**Type**: Functional  
**Prerequisites**: User likes their own reel

**Test Steps**:
1. User A sends LIKE request on their own reel

**Expected Results**:
- ✅ Status: 200 OK
- ✅ Like recorded successfully
- ✅ No notification sent (self-like)

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-ENGAGE-013: Feed Reflects Like State (isLiked Flag)

**Priority**: HIGH  
**Type**: Integration  
**Prerequisites**: TC-ENGAGE-004 passed (user liked reel)

**Test Steps**:
1. Fetch feed as `test_bronze`:
   ```bash
   GET /api/v1/feed
   Authorization: Bearer <jwt_token>
   ```
2. Find previously liked reel in response

**Expected Results**:
- ✅ Reel has `stats.isLiked = true`
- ✅ Other reels have `stats.isLiked = false`

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-ENGAGE-014: Feed Reflects Save State (isSaved Flag)

**Priority**: HIGH  
**Type**: Integration  
**Prerequisites**: TC-ENGAGE-007 passed (user saved reel)

**Test Steps**:
1. Fetch feed as `test_bronze`
2. Find previously saved reel in response

**Expected Results**:
- ✅ Reel has `stats.isSaved = true`
- ✅ Other reels have `stats.isSaved = false`

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-ENGAGE-015: Feed Cache Invalidated After Like

**Priority**: MEDIUM  
**Type**: Cache  
**Prerequisites**: First page cached

**Test Steps**:
1. Fetch feed (first page, cache miss)
2. Like a reel
3. Fetch feed again immediately

**Expected Results**:
- ✅ First fetch: Cache miss, cached with TTL
- ✅ Like action: Pub/sub event `invalidate:feed` published
- ✅ Second fetch: Cache miss (invalidated), fresh data
- ✅ Liked reel shows updated like count

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

## Following Filter Tests

### TC-FOLLOW-001: User with Following Sees Curated Feed

**Priority**: HIGH  
**Type**: Functional  
**Prerequisites**: User A follows 5 creators (status='accepted'), those creators have 30+ reels

**Test Steps**:
1. Login as User A
2. Fetch following-only feed:
   ```bash
   GET /api/v1/feed?followingOnly=true&userId=<userA_id>&limit=10
   Authorization: Bearer <jwt_token>
   ```

**Expected Results**:
- ✅ Status: 200 OK
- ✅ Response contains 10 reels
- ✅ All reels are from followed creators (5 specific userIds)
- ✅ No reels from non-followed creators

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-FOLLOW-002: User with No Following Sees Empty Feed

**Priority**: HIGH  
**Type**: Edge Case  
**Prerequisites**: User B follows 0 creators

**Test Steps**:
1. Login as User B
2. Fetch following-only feed:
   ```bash
   GET /api/v1/feed?followingOnly=true&userId=<userB_id>
   Authorization: Bearer <jwt_token>
   ```

**Expected Results**:
- ✅ Status: 200 OK
- ✅ Response:
  ```json
  {
    "success": true,
    "message": "Feed retrieved successfully",
    "data": {
      "items": [],
      "nextCursor": null
    }
  }
  ```

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-FOLLOW-003: Following Filter Respects Pagination

**Priority**: MEDIUM  
**Type**: Functional  
**Prerequisites**: User A follows 5 creators, those creators have 50+ reels

**Test Steps**:
1. Fetch first page (limit=10, followingOnly=true)
2. Extract nextCursor
3. Fetch second page with cursor

**Expected Results**:
- ✅ Page 1: 10 reels from followed creators
- ✅ Page 2: 10 different reels from followed creators
- ✅ No duplicates across pages
- ✅ Cursor pagination works with following filter

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-FOLLOW-004: Following Filter Without userId (Error)

**Priority**: LOW  
**Type**: Negative  
**Prerequisites**: None

**Test Steps**:
1. Send request with followingOnly=true but no userId:
   ```bash
   GET /api/v1/feed?followingOnly=true
   ```

**Expected Results**:
- ✅ Status: 400 Bad Request (TODO: Add validation)
- ✅ Error message: "userId required when followingOnly=true"

**Current Behavior** (without validation):
- ⚠️ Returns all reels (followingOnly ignored)

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-FOLLOW-005: Following Filter Excludes Pending Follows

**Priority**: MEDIUM  
**Type**: Functional  
**Prerequisites**: User A follows User B (status='pending'), User B has 10 reels

**Test Steps**:
1. Fetch following-only feed for User A

**Expected Results**:
- ✅ User B's reels NOT in response (pending not accepted)
- ✅ Only accepted follows included

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-FOLLOW-006: Following Filter Excludes Rejected Follows

**Priority**: LOW  
**Type**: Functional  
**Prerequisites**: User A followed User B, User B rejected (status='rejected')

**Test Steps**:
1. Fetch following-only feed for User A

**Expected Results**:
- ✅ User B's reels NOT in response (rejected follow)

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-FOLLOW-007: Following Feed Performance (Large Following List)

**Priority**: MEDIUM  
**Type**: Performance  
**Prerequisites**: User A follows 500 creators

**Test Steps**:
1. Fetch following-only feed:
   ```bash
   GET /api/v1/feed?followingOnly=true&userId=<userA_id>&limit=10
   ```
2. Measure response time

**Expected Results**:
- ✅ Response time < 350ms (acceptable delay for large IN query)
- ✅ MongoDB query uses index on `userId`
- ✅ PostgreSQL query uses index on `followerId`, `status`

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-FOLLOW-008: Following Filter with DEFAULT Sorting

**Priority**: LOW  
**Type**: Functional  
**Prerequisites**: User A follows 5 creators

**Test Steps**:
1. Fetch following-only feed with sort=DEFAULT:
   ```bash
   GET /api/v1/feed?followingOnly=true&userId=<userA_id>&sort=DEFAULT
   ```

**Expected Results**:
- ✅ Reels sorted by ranking score (not just createdAt)
- ✅ Advanced ranking applied to following feed

**Current Behavior**:
- ⚠️ Following feed uses RECENT sort (createdAt desc)
- ⚠️ TODO: Apply advanced ranking to following feed

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

## Order Context Tests

### TC-ORDER-001: Reel with LinkedOrder Shows Banner

**Priority**: HIGH  
**Type**: Functional  
**Prerequisites**: Reel with `linkedOrderId`, order has 3 available menu items

**Test Steps**:
1. Fetch feed:
   ```bash
   GET /api/v1/feed?limit=10
   ```
2. Find reel with linkedOrder

**Expected Results**:
- ✅ Reel has `linkedOrder` object:
  ```json
  {
    "orderId": "order-123",
    "itemCount": 3,
    "totalPaise": 59900,
    "chefId": "chef-456",
    "previewImageUrl": "s3://...",
    "itemPreviews": ["Butter Chicken", "Naan", "Raita"]
  }
  ```
- ✅ Banner displayed in mobile app

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-ORDER-002: Partial Order Shows Banner (At Least One Item Available)

**Priority**: HIGH  
**Type**: Functional  
**Prerequisites**: Reel with `linkedOrderId`, order has 3 items:
- Item 1: available
- Item 2: soldOut=true
- Item 3: isActive=false

**Test Steps**:
1. Fetch feed
2. Find reel with partial order

**Expected Results**:
- ✅ Reel has `linkedOrder` object (banner shown)
- ✅ Only 1 item available, but banner displayed
- ✅ `itemCount`: 3 (total items, not available items)
- ✅ Log: "LinkedOrder populated: 1/3 items available"

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-ORDER-003: Order with No Available Items Hides Banner

**Priority**: MEDIUM  
**Type**: Functional  
**Prerequisites**: Reel with `linkedOrderId`, all order items unavailable (soldOut or isActive=false)

**Test Steps**:
1. Fetch feed
2. Find reel with unavailable order

**Expected Results**:
- ✅ Reel has `linkedOrder = undefined` (banner hidden)
- ✅ Log: "LinkedOrder has no available items, skipping banner"

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-ORDER-004: Order Context Shows Chef Profile

**Priority**: HIGH  
**Type**: Functional  
**Prerequisites**: Reel with `linkedOrderId`, order from chef `test_gold`

**Test Steps**:
1. Fetch feed
2. Find reel with orderContext

**Expected Results**:
- ✅ Reel has `orderContext` object:
  ```json
  {
    "chefId": "chef-456",
    "chefName": "Chef Ramesh",
    "chefAvatarUrl": "s3://...",
    "distanceKm": 2.3,
    "etaMinutes": 45,
    "isOpen": true,
    "openingHours": "9:00 AM - 10:00 PM",
    "orderPreview": { ... }
  }
  ```

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-ORDER-005: Order Context Calculates Distance (With User Location)

**Priority**: MEDIUM  
**Type**: Functional  
**Prerequisites**: User location provided (lat=12.9716, lng=77.5946), chef location exists

**Test Steps**:
1. Fetch feed with location:
   ```bash
   GET /api/v1/feed?lat=12.9716&lng=77.5946&limit=10
   ```
2. Find reel with orderContext

**Expected Results**:
- ✅ `orderContext.distanceKm` is calculated (e.g., 2.3)
- ✅ Distance is accurate (within 10% of actual)

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-ORDER-006: Order Context Without User Location (No Distance)

**Priority**: LOW  
**Type**: Functional  
**Prerequisites**: User location not provided

**Test Steps**:
1. Fetch feed without lat/lng:
   ```bash
   GET /api/v1/feed?limit=10
   ```
2. Find reel with orderContext

**Expected Results**:
- ✅ `orderContext.distanceKm = undefined`
- ✅ No error (graceful fallback)

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-ORDER-007: Order Context Shows Chef Open Status

**Priority**: MEDIUM  
**Type**: Functional  
**Prerequisites**: Chef kitchen status = 'ACTIVE', current time within operating hours

**Test Steps**:
1. Fetch feed during chef's operating hours

**Expected Results**:
- ✅ `orderContext.isOpen = true`
- ✅ `orderContext.openingHours` = "9:00 AM - 10:00 PM"

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-ORDER-008: Order Context Shows Chef Closed Status

**Priority**: MEDIUM  
**Type**: Functional  
**Prerequisites**: Chef kitchen status = 'ACTIVE', current time outside operating hours

**Test Steps**:
1. Fetch feed outside chef's operating hours (e.g., 2 AM)

**Expected Results**:
- ✅ `orderContext.isOpen = false`
- ✅ Mobile app shows "Closed now" badge

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-ORDER-009: Order Context Cached (N+1 Prevention)

**Priority**: MEDIUM  
**Type**: Performance  
**Prerequisites**: 10 reels in feed, 3 from same chef (same linkedOrderId chef)

**Test Steps**:
1. Fetch feed
2. Monitor PostgreSQL query logs

**Expected Results**:
- ✅ Only 1 query to fetch chef profile (cached after first reel)
- ✅ Only 1 query to fetch kitchen (cached)
- ✅ Only 1 query to fetch schedule (cached)
- ✅ Total: 3 queries for 3 reels from same chef (not 9)

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-ORDER-010: Reel with LinkedMenu Shows Menu Item

**Priority**: MEDIUM  
**Type**: Functional  
**Prerequisites**: Reel with `linkedMenu` string ID, menu item available

**Test Steps**:
1. Fetch feed
2. Find reel with linkedMenu

**Expected Results**:
- ✅ Reel has `linkedMenu` object:
  ```json
  {
    "chefId": "chef-456",
    "menuItemIds": ["menu-item-789"],
    "estimatedPaise": 25000,
    "previewImage": "s3://..."
  }
  ```
- ✅ Price converted to paise (decimal × 100)

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

## Cache Tests

### TC-CACHE-001: First Page Cached (Guest Feed)

**Priority**: HIGH  
**Type**: Cache  
**Prerequisites**: Redis cache empty

**Test Steps**:
1. Fetch first page (no cursor):
   ```bash
   GET /api/v1/feed?limit=10
   ```
2. Check Redis for cache key: `feed:DEFAULT:limit:10:lat:null:lng:null`

**Expected Results**:
- ✅ First request: Cache miss, response time ~150ms
- ✅ Redis cache set with TTL (300 seconds)
- ✅ Cache value: JSON string of { items, nextCursor }

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-CACHE-002: Second Request Served from Cache

**Priority**: HIGH  
**Type**: Cache  
**Prerequisites**: TC-CACHE-001 passed (first page cached)

**Test Steps**:
1. Immediately fetch first page again:
   ```bash
   GET /api/v1/feed?limit=10
   ```

**Expected Results**:
- ✅ Second request: Cache hit, response time < 50ms
- ✅ Response identical to first request (same items, same order)
- ✅ No MongoDB query executed

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-CACHE-003: Cursor Bypasses Cache (Page 2+)

**Priority**: MEDIUM  
**Type**: Cache  
**Prerequisites**: First page cached

**Test Steps**:
1. Fetch second page with cursor:
   ```bash
   GET /api/v1/feed?cursor=65f3b1c...&limit=10
   ```

**Expected Results**:
- ✅ Cache not used (cursor bypasses cache)
- ✅ MongoDB query executed
- ✅ Response time ~150-300ms (no cache benefit)

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-CACHE-004: Different Sort Creates Separate Cache

**Priority**: LOW  
**Type**: Cache  
**Prerequisites**: DEFAULT feed cached

**Test Steps**:
1. Fetch TRENDING feed:
   ```bash
   GET /api/v1/feed?sort=TRENDING&limit=10
   ```
2. Check Redis for cache key: `feed:TRENDING:limit:10:lat:null:lng:null`

**Expected Results**:
- ✅ Cache miss (different cache key)
- ✅ New cache entry created for TRENDING

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-CACHE-005: Different Limit Creates Separate Cache

**Priority**: LOW  
**Type**: Cache  
**Prerequisites**: Limit=10 feed cached

**Test Steps**:
1. Fetch with limit=20:
   ```bash
   GET /api/v1/feed?limit=20
   ```
2. Check Redis for cache key: `feed:DEFAULT:limit:20:lat:null:lng:null`

**Expected Results**:
- ✅ Cache miss (different cache key)
- ✅ New cache entry created for limit=20

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-CACHE-006: Cache Expires After TTL

**Priority**: MEDIUM  
**Type**: Cache  
**Prerequisites**: First page cached with TTL=300s

**Test Steps**:
1. Fetch first page (cached)
2. Wait 301 seconds
3. Fetch first page again

**Expected Results**:
- ✅ First request: Cache hit
- ✅ After TTL: Cache expired, cache miss
- ✅ New cache entry created with fresh TTL

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-CACHE-007: Cache Invalidated on Like

**Priority**: HIGH  
**Type**: Cache  
**Prerequisites**: First page cached

**Test Steps**:
1. Fetch first page (cached)
2. Like a reel
3. Fetch first page again immediately

**Expected Results**:
- ✅ Like action: Pub/sub event `invalidate:feed` published
- ✅ Cache cleared (all `feed:*` keys deleted)
- ✅ Third fetch: Cache miss, fresh data with updated like count

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-CACHE-008: Cache Invalidated on Save

**Priority**: MEDIUM  
**Type**: Cache  
**Prerequisites**: First page cached

**Test Steps**:
1. Fetch first page (cached)
2. Save a reel
3. Fetch first page again immediately

**Expected Results**:
- ✅ Save action: Pub/sub event published
- ✅ Cache cleared
- ✅ Third fetch: Cache miss, fresh data with updated save count

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

## Abuse Prevention Tests

### TC-ABUSE-001: Upload Rate Limit (Hourly)

**Priority**: HIGH  
**Type**: Abuse Prevention  
**Prerequisites**: Bronze user (hourly limit = 10)

**Test Steps**:
1. Create 10 reels within 1 hour (succeeds)
2. Attempt to create 11th reel

**Expected Results**:
- ✅ First 10 reels: Success
- ✅ 11th reel: 429 Too Many Requests
- ✅ Error code: `UPLOAD_RATE_EXCEEDED`
- ✅ Error message: "Upload rate exceeded: 10/10 per hour"

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-ABUSE-002: Upload Rate Limit (Daily)

**Priority**: HIGH  
**Type**: Abuse Prevention  
**Prerequisites**: Bronze user (daily limit = 50)

**Test Steps**:
1. Create 50 reels within 24 hours (succeeds)
2. Attempt to create 51st reel

**Expected Results**:
- ✅ First 50 reels: Success
- ✅ 51st reel: 429 Too Many Requests
- ✅ Error code: `UPLOAD_RATE_EXCEEDED`
- ✅ Error message: "Daily upload limit exceeded: 50/50"

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-ABUSE-003: Tier-Adjusted Upload Limits (Legend)

**Priority**: MEDIUM  
**Type**: Abuse Prevention  
**Prerequisites**: Legend user (hourly limit = 50, daily limit = 250)

**Test Steps**:
1. Create 50 reels within 1 hour (succeeds)
2. Attempt to create 51st reel

**Expected Results**:
- ✅ First 50 reels: Success (5× Bronze limit)
- ✅ 51st reel: 429 Too Many Requests

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-ABUSE-004: Tier-Adjusted Upload Limits (Diamond)

**Priority**: MEDIUM  
**Type**: Abuse Prevention  
**Prerequisites**: Diamond user (hourly limit = 30, daily limit = 150)

**Test Steps**:
1. Create 30 reels within 1 hour (succeeds)
2. Attempt to create 31st reel

**Expected Results**:
- ✅ First 30 reels: Success (3× Bronze limit)
- ✅ 31st reel: 429 Too Many Requests
- ✅ Error code: `UPLOAD_RATE_EXCEEDED`

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-ABUSE-005: Duplicate Content Detection

**Priority**: MEDIUM  
**Type**: Abuse Prevention  
**Prerequisites**: Content hash computed for reels

**Test Steps**:
1. Upload reel with content hash `abc123`
2. Immediately upload same reel (hash `abc123`) again

**Expected Results**:
- ✅ First upload: Success
- ✅ Second upload: 409 Conflict
- ✅ Error code: `DUPLICATE_CONTENT`
- ✅ Error message: "Duplicate content detected within 60 minutes"

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-ABUSE-006: Duplicate Allowed After Window

**Priority**: LOW  
**Type**: Abuse Prevention  
**Prerequisites**: TC-ABUSE-005 passed

**Test Steps**:
1. Wait 61 minutes
2. Upload same reel (hash `abc123`) again

**Expected Results**:
- ✅ Upload succeeds (duplicate window expired)

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-ABUSE-007: Engagement Anomaly Detection

**Priority**: MEDIUM  
**Type**: Abuse Prevention  
**Prerequisites**: User has avg engagement rate of 5%, new reel has 60% rate within 10 minutes

**Test Steps**:
1. Check anomaly detection for suspicious reel

**Expected Results**:
- ✅ Anomaly detected (60% vs 5% = 1100% spike > 200% threshold)
- ✅ Error code: `ENGAGEMENT_ANOMALY`
- ✅ User state: WARNED or THROTTLED
- ✅ Visibility multiplier reduced

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-ABUSE-008: Feed State Transition (NORMAL → WARNED)

**Priority**: MEDIUM  
**Type**: Abuse Prevention  
**Prerequisites**: User in NORMAL state (0 violations)

**Test Steps**:
1. Trigger 2 violations (e.g., upload rate, duplicate content)
2. Check user feed state

**Expected Results**:
- ✅ Feed state: WARNED
- ✅ Violation count: 2
- ✅ Visibility multiplier: 0.6 (40% reduction)

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-ABUSE-009: Feed State Transition (WARNED → THROTTLED)

**Priority**: MEDIUM  
**Type**: Abuse Prevention  
**Prerequisites**: User in WARNED state (2 violations)

**Test Steps**:
1. Trigger 2 more violations (total 4)
2. Check user feed state

**Expected Results**:
- ✅ Feed state: THROTTLED
- ✅ Violation count: 4
- ✅ Visibility multiplier: 0.3 (70% reduction)

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-ABUSE-009: Feed State Transition (THROTTLED → SUPPRESSED)

**Priority**: HIGH  
**Type**: Abuse Prevention  
**Prerequisites**: User in THROTTLED state (4 violations)

**Test Steps**:
1. Trigger 3 more violations (total 7)
2. Check user feed state

**Expected Results**:
- ✅ Feed state: SUPPRESSED
- ✅ Violation count: 7
- ✅ Visibility multiplier: 0.0 (100% hidden)
- ✅ User's reels hidden from all feeds

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-ABUSE-010: Recovery from SUPPRESSED (Cooldown + Clean Behavior)

**Priority**: HIGH  
**Type**: Abuse Prevention  
**Prerequisites**: User in SUPPRESSED state (7 violations), no new violations for 30 days

**Test Steps**:
1. Wait 24 hours (cooldown period)
2. Wait 30 days (violation decay)
3. Check user feed state

**Expected Results**:
- ✅ After 24h: Cooldown passed, but violations still high (state remains SUPPRESSED)
- ✅ After 30 days: Violations decayed to 0
- ✅ Feed state: NORMAL (graduated recovery)
- ✅ Visibility multiplier: 1.0 (full recovery)

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-ABUSE-011: Diamond Tier Abuse Exemption

**Priority**: HIGH  
**Type**: Abuse Prevention  
**Prerequisites**: Diamond user (abuse exempt)

**Test Steps**:
1. Diamond user triggers duplicate content detection
2. Diamond user exceeds upload rate (within tier limit)
3. Check if abuse controls applied

**Expected Results**:
- ✅ Duplicate content: Warning only, no state change
- ✅ Upload rate: No 429 error (exempt from controls)
- ✅ Feed state: Remains NORMAL
- ✅ User can continue uploading

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-ABUSE-012: Legend Tier Abuse Exemption

**Priority**: HIGH  
**Type**: Abuse Prevention  
**Prerequisites**: Legend user (abuse exempt)

**Test Steps**:
1. Legend user uploads 100 reels in 1 hour (far exceeds base limit)
2. Check if rate limit triggered

**Expected Results**:
- ✅ All 100 uploads succeed
- ✅ No rate limit errors
- ✅ Legend tier exempt from abuse controls
- ✅ Logs show exemption reason

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

## Security Tests

### TC-SEC-001: JWT Token Required for LIKE

**Priority**: HIGH  
**Type**: Security  
**Prerequisites**: None

**Test Steps**:
1. Send LIKE request without Authorization header:
   ```bash
   POST /api/v1/feed/engagement
   Content-Type: application/json
   {
     "reelId": "65f3b1c4f8d9e2a1b3c4d5e6",
     "action": "LIKE"
   }
   ```

**Expected Results**:
- ✅ Status: 401 Unauthorized
- ✅ Error code: `FEED_AUTH_REQUIRED`

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-SEC-002: JWT Token Required for SAVE

**Priority**: HIGH  
**Type**: Security  
**Prerequisites**: None

**Test Steps**:
1. Send SAVE request without Authorization header

**Expected Results**:
- ✅ Status: 401 Unauthorized
- ✅ Error code: `FEED_AUTH_REQUIRED`

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-SEC-003: Invalid JWT Token Rejected

**Priority**: HIGH  
**Type**: Security  
**Prerequisites**: None

**Test Steps**:
1. Send LIKE request with invalid JWT:
   ```bash
   POST /api/v1/feed/engagement
   Authorization: Bearer invalid_token_123
   Content-Type: application/json
   {
     "reelId": "65f3b1c4f8d9e2a1b3c4d5e6",
     "action": "LIKE"
   }
   ```

**Expected Results**:
- ✅ Status: 401 Unauthorized
- ✅ Error message: "Invalid token"

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-SEC-004: Expired JWT Token Rejected

**Priority**: MEDIUM  
**Type**: Security  
**Prerequisites**: Expired JWT token (issued > 7 days ago)

**Test Steps**:
1. Send LIKE request with expired JWT

**Expected Results**:
- ✅ Status: 401 Unauthorized
- ✅ Error message: "Token expired"

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-SEC-005: Admin Reseed Endpoint Unsecured (Critical)

**Priority**: CRITICAL  
**Type**: Security  
**Prerequisites**: None

**Test Steps**:
1. Send reseed request without admin credentials:
   ```bash
   POST /api/v1/feed/admin/reseed
   ```

**Expected Results** (after security fix):
- ✅ Status: 403 Forbidden
- ✅ Error message: "Admin role required"

**Current Behavior** (before fix):
- ⚠️ Status: 200 OK (SECURITY RISK)
- ⚠️ All reels deleted and reseeded without auth

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-SEC-006: SQL Injection Prevention (Reputation Lookup)

**Priority**: HIGH  
**Type**: Security  
**Prerequisites**: None

**Test Steps**:
1. Attempt SQL injection via userId in followingOnly filter:
   ```bash
   GET /api/v1/feed?followingOnly=true&userId=abc' OR '1'='1
   ```

**Expected Results**:
- ✅ No SQL injection (TypeORM parameterized queries)
- ✅ Returns empty feed or 400 Bad Request (invalid UUID format)

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

## Performance Tests

### TC-PERF-001: Feed Retrieval < 200ms (Cache Miss)

**Priority**: HIGH  
**Type**: Performance  
**Prerequisites**: Redis cache empty, 100+ reels exist

**Test Steps**:
1. Fetch first page:
   ```bash
   GET /api/v1/feed?limit=10
   ```
2. Measure response time

**Expected Results**:
- ✅ Response time < 200ms (p95)
- ✅ Cache miss scenario

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-PERF-002: Feed Retrieval < 50ms (Cache Hit)

**Priority**: HIGH  
**Type**: Performance  
**Prerequisites**: First page cached

**Test Steps**:
1. Fetch first page (second request, cache hit)
2. Measure response time

**Expected Results**:
- ✅ Response time < 50ms (p95)
- ✅ Cache hit scenario

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-PERF-003: Engagement Recording < 100ms

**Priority**: MEDIUM  
**Type**: Performance  
**Prerequisites**: None

**Test Steps**:
1. Send LIKE request
2. Measure response time

**Expected Results**:
- ✅ Response time < 100ms (p95)
- ✅ Includes MongoDB write, Redis cache, notification trigger

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-PERF-004: Feed Under Load (100 concurrent requests)

**Priority**: MEDIUM  
**Type**: Load Testing  
**Prerequisites**: Load testing tool (Apache JMeter, k6)

**Test Steps**:
1. Send 100 concurrent GET /feed requests
2. Measure average response time, error rate

**Expected Results**:
- ✅ Average response time < 300ms
- ✅ Error rate < 1%
- ✅ No timeouts
- ✅ Cache hit rate > 80%

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

## Integration Tests

### TC-INT-001: End-to-End Feed Flow (Guest to Authenticated)

**Priority**: HIGH  
**Type**: Integration  
**Prerequisites**: None

**Test Steps**:
1. Guest user fetches feed (cached)
2. Guest user views reel (throttled)
3. Guest user signs up
4. User fetches feed (personalized, isLiked/isSaved flags)
5. User likes reel
6. User saves reel
7. User fetches feed again (isLiked=true, isSaved=true)

**Expected Results**:
- ✅ All steps succeed
- ✅ Cache invalidated after like/save
- ✅ Feed reflects user engagement state

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-INT-002: Feed to Order Conversion Flow

**Priority**: HIGH  
**Type**: Integration  
**Prerequisites**: Reel with linkedOrder, order available

**Test Steps**:
1. User fetches feed
2. User sees reel with order banner
3. User taps order banner (mobile app)
4. App navigates to order details screen
5. User adds items to cart
6. User proceeds to checkout

**Expected Results**:
- ✅ Order banner displayed correctly
- ✅ Navigation works
- ✅ Order context shows chef profile, distance, ETA
- ✅ Conversion tracked in analytics

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

## Regression Tests

### TC-REG-001: Menu Reel Feature Flag (ON/OFF)

**Priority**: HIGH  
**Type**: Regression  
**Prerequisites**: Feature flag `canViewMenuReel()` exists

**Test Steps**:
1. Set flag to OFF
2. Fetch feed, verify MENU_SHOWCASE reels excluded
3. Set flag to ON
4. Fetch feed, verify MENU_SHOWCASE reels included

**Expected Results**:
- ✅ Flag OFF: No MENU_SHOWCASE reels
- ✅ Flag ON: MENU_SHOWCASE reels visible

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

### TC-REG-002: Legacy getFeedOld() Method (Deprecated)

**Priority**: LOW  
**Type**: Regression  
**Prerequisites**: Legacy method exists for reference

**Test Steps**:
1. Verify `getFeedOld()` method exists but not called
2. Verify `getFeed()` uses advanced ranking (not legacy)

**Expected Results**:
- ✅ Legacy method exists (for reference)
- ✅ Not used in production code path

**Actual Results**: _[To be filled during testing]_

**Status**: ⬜ Not Started | 🟡 In Progress | ✅ Passed | ❌ Failed

---

## Test Execution Summary

### Overall Statistics

| Metric | Value |
|--------|-------|
| **Total Test Cases** | 85 |
| **Executed** | 0 |
| **Passed** | 0 |
| **Failed** | 0 |
| **Blocked** | 0 |
| **Pass Rate** | 0% |

### Priority Breakdown

| Priority | Total | Passed | Failed | Pass Rate |
|----------|-------|--------|--------|-----------|
| **CRITICAL** | 1 | 0 | 0 | 0% |
| **HIGH** | 42 | 0 | 0 | 0% |
| **MEDIUM** | 32 | 0 | 0 | 0% |
| **LOW** | 10 | 0 | 0 | 0% |

### Category Breakdown

| Category | Total | Passed | Failed | Pass Rate |
|----------|-------|--------|--------|-----------|
| **Feed Retrieval** | 12 | 0 | 0 | 0% |
| **Advanced Ranking** | 10 | 0 | 0 | 0% |
| **Engagement Tracking** | 15 | 0 | 0 | 0% |
| **Following Filter** | 8 | 0 | 0 | 0% |
| **Order Context** | 10 | 0 | 0 | 0% |
| **Cache** | 8 | 0 | 0 | 0% |
| **Abuse Prevention** | 10 | 0 | 0 | 0% |
| **Security** | 6 | 0 | 0 | 0% |
| **Performance** | 4 | 0 | 0 | 0% |
| **Integration** | 2 | 0 | 0 | 0% |

---

## Defects Log

| Defect ID | Test Case | Severity | Description | Status |
|-----------|-----------|----------|-------------|--------|
| DEF-001 | TC-FEED-010 | LOW | Invalid cursor returns empty feed (no validation) | Open |
| DEF-002 | TC-FOLLOW-004 | LOW | followingOnly without userId returns all reels | Open |
| DEF-003 | TC-ENGAGE-006 | HIGH | LIKE without auth succeeds (fallback userId in dev mode) | Open |
| DEF-004 | TC-SEC-005 | CRITICAL | Admin reseed endpoint unsecured | Open |
| DEF-005 | TC-FOLLOW-008 | LOW | Following feed uses RECENT sort (not DEFAULT ranking) | Open |

---

## Test Environment Setup

### Docker Compose (Local Testing)

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: chefooz_test
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test123
    ports:
      - "5432:5432"

  mongodb:
    image: mongo:7
    environment:
      MONGO_INITDB_DATABASE: chefooz_test
    ports:
      - "27017:27017"

  redis:
    image: redis:7
    ports:
      - "6379:6379"

  backend:
    build: .
    environment:
      DATABASE_URL: postgresql://test:test123@postgres:5432/chefooz_test
      MONGODB_URI: mongodb://mongodb:27017/chefooz_test
      REDIS_HOST: redis
      REDIS_PORT: 6379
    ports:
      - "3000:3000"
    depends_on:
      - postgres
      - mongodb
      - redis
```

### Seed Test Data

```bash
# Run seed script
npm run seed:test

# Verify data
psql -h localhost -U test -d chefooz_test -c "SELECT COUNT(*) FROM users;"
mongo chefooz_test --eval "db.reels.count()"
```

---

**[SLICE_COMPLETE ✅]**  
**Feed Module - QA Test Cases**  
**Generated:** February 14, 2026  
**Lines:** ~1,850
