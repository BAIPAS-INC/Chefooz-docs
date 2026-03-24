# Explore Module - QA Test Cases

**Version:** 1.0  
**Last Updated:** February 14, 2026  
**Module:** `apps/chefooz-apis/src/modules/explore/`  
**Test Environment:** Staging (staging.chefooz.app)  
**Total Test Cases:** 45

---

## 📋 Test Overview

| Category | Test Cases | Priority | Status |
|----------|------------|----------|--------|
| **Section Retrieval** | 10 | High | ✅ Ready |
| **Ranking Algorithm** | 12 | High | ✅ Ready |
| **Privacy & Filtering** | 8 | Critical | ✅ Ready |
| **Caching & Performance** | 6 | Medium | ✅ Ready |
| **Chef Discovery** | 5 | Medium | ✅ Ready |
| **Category Browsing** | 4 | Low | ✅ Ready |

---

## Test Data Requirements

### Test Users

| Username | Role | Tier | Location | Follows | Orders | Purpose |
|----------|------|------|----------|---------|--------|---------|
| test_customer_1 | Customer | Silver | Mumbai (19.0760, 72.8777) | 10 chefs | 25 orders | Active user with history |
| test_customer_2 | Customer | Bronze | Delhi (28.6139, 77.2090) | 0 chefs | 0 orders | New user, no history |
| test_chef_gold | Chef | Gold | Mumbai (19.0760, 72.8777) | - | 150 orders fulfilled | High-reputation chef |
| test_chef_bronze | Chef | Bronze | Mumbai (19.0820, 72.8850) | - | 5 orders fulfilled | New chef |
| test_blocked_user | Customer | Silver | Mumbai | - | - | For block testing |

### Test Reels

| ID | Chef | Purpose | Age | Stats | Tags |
|----|------|---------|-----|-------|------|
| reel_trending_1 | test_chef_gold | MENU_SHOWCASE | 2 days | 5000 views, 400 likes, 15 orders | #butterchicken #north indian |
| reel_trending_2 | test_chef_gold | MENU_SHOWCASE | 3 days | 3000 views, 250 likes, 10 orders | #biryani #spicy |
| reel_new_1 | test_chef_bronze | MENU_SHOWCASE | 3 hours | 50 views, 10 likes, 0 orders | #paneer #vegetarian |
| reel_exclusive_1 | test_chef_gold | MENU_SHOWCASE | 1 day | 1000 views, 100 likes, 20 orders | #signature #premium, promoted=true |

---

## Section Retrieval Tests

### TC-SEC-001: Get Consolidated Sections

**Priority**: High  
**Endpoint**: `GET /api/v1/explore/sections`

**Prerequisites**:
- User logged in (test_customer_1)
- Minimum 20 reels exist in database
- At least 5 active chefs

**Test Steps**:
1. Call `GET /api/v1/explore/sections?limit=5`
2. Verify response structure

**Expected Results**:
```json
{
  "success": true,
  "data": {
    "trending": { "items": [5 items], "hasMore": true },
    "recommended": { "items": [5 items], "hasMore": true },
    "newDish": { "items": [5 items], "hasMore": true },
    "exclusive": { "items": [≤5 items], "hasMore": false },
    "topChefs": [5 chef objects],
    "trendingHashtags": [10 hashtag objects]
  }
}
```

**Pass Criteria**:
- ✅ Response status = 200
- ✅ All sections present
- ✅ Each section has correct item structure
- ✅ Response time < 500ms

---

### TC-SEC-002: Get Trending Section (Paginated)

**Priority**: High  
**Endpoint**: `GET /api/v1/explore/trending`

**Test Steps**:
1. Call `GET /api/v1/explore/trending?limit=24`
2. Verify first page results
3. Extract `nextCursor` from response
4. Call `GET /api/v1/explore/trending?limit=24&cursor={nextCursor}`
5. Verify second page results

**Expected Results**:
- Page 1: 24 items, `hasMore = true`, cursor present
- Page 2: Up to 24 items, no overlap with page 1
- Items sorted by trending score (highest first)

**Pass Criteria**:
- ✅ No duplicate items across pages
- ✅ Items ranked by engagement score
- ✅ All items visible to user (privacy respected)

---

### TC-SEC-003: Get Recommended Section (Personalized)

**Priority**: High  
**Endpoint**: `GET /api/v1/explore/recommended`

**Prerequisites**:
- test_customer_1 follows test_chef_gold
- test_customer_1 has ordered from test_chef_gold before

**Test Steps**:
1. Call `GET /api/v1/explore/recommended?limit=24`
2. Verify personalization

**Expected Results**:
- Reels from test_chef_gold appear in top 5
- Cuisine types match user's past orders (North Indian, if ordered before)
- Items are deliverable to user's location

**Pass Criteria**:
- ✅ Followed chefs boosted (appear higher)
- ✅ No unavailable chefs shown
- ✅ Cuisine diversity maintained

---

### TC-SEC-004: Get New Dish Section

**Priority**: High  
**Endpoint**: `GET /api/v1/explore/new_dish`

**Prerequisites**:
- reel_new_1 uploaded within last 24 hours

**Test Steps**:
1. Call `GET /api/v1/explore/new_dish?limit=24`
2. Verify recency

**Expected Results**:
- reel_new_1 appears in results
- All reels created within last 24 hours
- Sorted by creation time (newest first)

**Pass Criteria**:
- ✅ Only reels with `reelPurpose = MENU_SHOWCASE`
- ✅ No reels older than 24 hours
- ✅ Chef must be online

---

### TC-SEC-005: Get Exclusive Section

**Priority**: Medium  
**Endpoint**: `GET /api/v1/explore/exclusive`

**Prerequisites**:
- reel_exclusive_1 has `promoted = true`
- test_chef_gold has Gold tier or higher

**Test Steps**:
1. Call `GET /api/v1/explore/exclusive?limit=24`
2. Verify promoted content

**Expected Results**:
- reel_exclusive_1 appears in results
- All reels have `promoted = true`
- Only chefs with Gold+ tier

**Pass Criteria**:
- ✅ Only promoted content shown
- ✅ Max 10 exclusive items
- ✅ Chef reputation factored in ranking

---

### TC-SEC-006: Empty Section Handling

**Priority**: Medium  
**Endpoint**: `GET /api/v1/explore/new_dish`

**Prerequisites**:
- No reels uploaded in last 24 hours

**Test Steps**:
1. Call `GET /api/v1/explore/new_dish?limit=24`

**Expected Results**:
```json
{
  "success": true,
  "data": {
    "items": [],
    "nextCursor": null,
    "hasMore": false
  }
}
```

**Pass Criteria**:
- ✅ Returns empty array (not error)
- ✅ Status = 200

---

### TC-SEC-007: Invalid Section Name

**Priority**: Low  
**Endpoint**: `GET /api/v1/explore/invalid_section`

**Test Steps**:
1. Call with invalid section name

**Expected Results**:
```json
{
  "success": false,
  "message": "Invalid section name",
  "errorCode": "INVALID_SECTION"
}
```

**Pass Criteria**:
- ✅ Status = 400
- ✅ Clear error message

---

### TC-SEC-008: Pagination Limits

**Priority**: Medium  
**Endpoint**: `GET /api/v1/explore/trending`

**Test Steps**:
1. Call with `limit=5` (min valid)
2. Call with `limit=48` (max valid)
3. Call with `limit=100` (exceeds max)

**Expected Results**:
- `limit=5`: Returns 5 items
- `limit=48`: Returns up to 48 items
- `limit=100`: Validation error

**Pass Criteria**:
- ✅ Enforces min/max limits
- ✅ Validation error for invalid values

---

### TC-SEC-009: Unauthenticated Access

**Priority**: High  
**Endpoint**: `GET /api/v1/explore/sections`

**Test Steps**:
1. Call without Authorization header

**Expected Results**:
```json
{
  "success": false,
  "message": "Authentication required",
  "errorCode": "UNAUTHORIZED"
}
```

**Pass Criteria**:
- ✅ Status = 401
- ✅ Requires JWT token

---

### TC-SEC-010: Rate Limiting

**Priority**: Medium  
**Endpoint**: `GET /api/v1/explore/trending`

**Test Steps**:
1. Make 61 requests within 1 minute (exceeds 60/min limit)

**Expected Results**:
- First 60 requests: Success (200)
- 61st request: Rate limit error (429)

**Pass Criteria**:
- ✅ Rate limiter enforces 60 req/min
- ✅ Error includes `retryAfter` seconds

---

## Ranking Algorithm Tests

### TC-RANK-001: Availability Hard Filter

**Priority**: Critical  
**Description**: Only available chefs should appear in explore

**Prerequisites**:
- test_chef_gold: Online, kitchen open, ETA 30 min
- test_chef_offline: Offline

**Test Steps**:
1. Call `GET /api/v1/explore/trending?limit=24`
2. Check if test_chef_offline's reels appear

**Expected Results**:
- test_chef_gold's reels: ✅ Appear
- test_chef_offline's reels: ❌ Excluded

**Pass Criteria**:
- ✅ Offline chefs excluded
- ✅ Overloaded chefs excluded
- ✅ ETA > 90 min excluded

---

### TC-RANK-002: Followed Chef Boost

**Priority**: High  
**Description**: Followed chefs rank higher in recommendations

**Prerequisites**:
- test_customer_1 follows test_chef_gold
- test_customer_1 does NOT follow test_chef_bronze

**Test Steps**:
1. Call `GET /api/v1/explore/recommended?limit=24`
2. Compare ranks

**Expected Results**:
- test_chef_gold's reels appear in top 10
- test_chef_bronze's reels appear lower (if at all)

**Pass Criteria**:
- ✅ Followed chefs get 5.0× boost
- ✅ Appear higher than unfollowed chefs with similar engagement

---

### TC-RANK-003: Order History Boost

**Priority**: High  
**Description**: Chefs user ordered from before rank higher

**Prerequisites**:
- test_customer_1 ordered from test_chef_gold (3 orders)
- test_customer_1 never ordered from test_chef_bronze

**Test Steps**:
1. Call `GET /api/v1/explore/recommended?limit=24`
2. Verify test_chef_gold appears

**Expected Results**:
- test_chef_gold gets personal relevance boost (+30 points)

**Pass Criteria**:
- ✅ Past order history factored
- ✅ Boosts ranking appropriately

---

### TC-RANK-004: Trending Score Calculation

**Priority**: High  
**Description**: Trending score calculated correctly

**Test Data**:
```
Reel A (2 days old):
  Views: 5000, Likes: 400, Saves: 100, Orders: 15
  engagementScore = 5000×1 + 400×2 + 100×4 + 15×10 = 6450
  timeFactor = 0.8^2 = 0.64
  trendingScore = 6450 × 0.64 = 4128

Reel B (5 days old):
  Views: 10000, Likes: 600, Saves: 150, Orders: 10
  engagementScore = 10000 + 1200 + 600 + 100 = 11900
  timeFactor = 0.8^5 = 0.328
  trendingScore = 11900 × 0.328 = 3903
```

**Expected Results**:
- Reel A ranks higher than Reel B (fresher + better engagement/day)

**Pass Criteria**:
- ✅ Correct formula applied
- ✅ Time decay enforced
- ✅ Orders weighted 10× views

---

### TC-RANK-005: Diversity Guard (Max 2 Per Chef)

**Priority**: High  
**Description**: No single chef dominates feed

**Prerequisites**:
- test_chef_gold has 10 trending reels

**Test Steps**:
1. Call `GET /api/v1/explore/trending?limit=24`
2. Count reels from test_chef_gold in results

**Expected Results**:
- Max 2 reels from test_chef_gold in top 24

**Pass Criteria**:
- ✅ Diversity guard enforced
- ✅ Other chefs get visibility

---

### TC-RANK-006: Cuisine Diversity (Max 5 Per Cuisine)

**Priority**: Medium  
**Description**: Cuisine diversity maintained

**Prerequisites**:
- 20 North Indian reels with high scores

**Test Steps**:
1. Call `GET /api/v1/explore/trending?limit=24`
2. Count North Indian reels

**Expected Results**:
- Max 5 North Indian reels in top 24
- Other cuisines represented

**Pass Criteria**:
- ✅ Max 5 items per cuisine
- ✅ Diverse cuisine mix

---

### TC-RANK-007: ETA Penalty

**Priority**: Medium  
**Description**: High ETA reduces availability score

**Test Data**:
```
Chef A: Online, ETA 25 min → 100 availability score
Chef B: Online, ETA 50 min → 80 availability score
Chef C: Online, ETA 70 min → 70 availability score
```

**Expected Results**:
- Chef A ranks higher (all else equal)

**Pass Criteria**:
- ✅ ETA factored into availability score
- ✅ Lower ETA = higher rank

---

### TC-RANK-008: Visual Trust Score

**Priority**: Low  
**Description**: High-quality reels rank higher

**Test Data**:
```
Reel A: 4K video, 85% watch completion
Reel B: 480p video, 30% watch completion
```

**Expected Results**:
- Reel A gets higher visual trust score (87 vs 32)

**Pass Criteria**:
- ✅ Video quality factored
- ✅ Watch completion rate factored

---

### TC-RANK-009: Freshness Boost

**Priority**: Medium  
**Description**: Recent content gets freshness boost

**Test Data**:
```
Reel uploaded 2 hours ago: 100 freshness score
Reel uploaded 5 days ago: 40 freshness score
```

**Expected Results**:
- Fresher reel ranks higher (all else equal)

**Pass Criteria**:
- ✅ Freshness decays with age
- ✅ 24h = 100, 7d = 20

---

### TC-RANK-010: Score Jitter (Exploration)

**Priority**: Low  
**Description**: ±5% jitter prevents ranking staleness

**Test Steps**:
1. Call explore API twice with same user
2. Compare ranking order

**Expected Results**:
- Slight variation in order (not identical)
- Difference < 5% per item

**Pass Criteria**:
- ✅ Jitter applied
- ✅ Encourages exploration

---

### TC-RANK-011: No Duplicate Dishes

**Priority**: High  
**Description**: Same dish from same chef not duplicated

**Prerequisites**:
- test_chef_gold has 2 reels for "Butter Chicken"

**Test Steps**:
1. Call `GET /api/v1/explore/trending?limit=24`
2. Check for duplicate "Butter Chicken" from test_chef_gold

**Expected Results**:
- Only 1 "Butter Chicken" reel appears

**Pass Criteria**:
- ✅ Duplicate prevention enforced

---

### TC-RANK-012: Promoted Content Boost

**Priority**: Medium  
**Description**: Promoted reels rank higher in exclusive section

**Prerequisites**:
- reel_exclusive_1: promoted=true, reputation 75
- reel_regular_1: promoted=false, reputation 80

**Test Steps**:
1. Call `GET /api/v1/explore/exclusive?limit=24`

**Expected Results**:
- reel_exclusive_1 ranks higher despite lower reputation

**Pass Criteria**:
- ✅ Promotion boost applied
- ✅ Paid promotions prioritized

---

## Privacy & Filtering Tests

### TC-PRIV-001: Blocked User Content Hidden

**Priority**: Critical  
**Description**: User A blocks User B → Neither sees other's content

**Prerequisites**:
- test_customer_1 blocks test_chef_blocked

**Test Steps**:
1. test_customer_1 calls `GET /api/v1/explore/trending`
2. Verify test_chef_blocked's reels absent

**Expected Results**:
- test_chef_blocked's reels: ❌ Not visible to test_customer_1

**Pass Criteria**:
- ✅ Bidirectional block enforced
- ✅ Content completely hidden

---

### TC-PRIV-002: Private Account Content Hidden

**Priority**: Critical  
**Description**: Private chef only visible to followers

**Prerequisites**:
- test_chef_private: isPrivate=true
- test_customer_1: Does NOT follow test_chef_private
- test_customer_2: Follows test_chef_private

**Test Steps**:
1. test_customer_1 calls `GET /api/v1/explore/trending`
2. test_customer_2 calls `GET /api/v1/explore/trending`

**Expected Results**:
- test_customer_1: ❌ Cannot see test_chef_private's reels
- test_customer_2: ✅ Can see test_chef_private's reels

**Pass Criteria**:
- ✅ Privacy respected
- ✅ Followers can see content

---

### TC-PRIV-003: Deactivated User Content Hidden

**Priority**: Critical  
**Description**: Deactivated users excluded from explore

**Prerequisites**:
- test_chef_deactivated: isDeactivated=true

**Test Steps**:
1. Call `GET /api/v1/explore/trending`

**Expected Results**:
- test_chef_deactivated's reels: ❌ Not visible

**Pass Criteria**:
- ✅ Deactivated users excluded

---

### TC-PRIV-004: Flagged Content Hidden

**Priority**: High  
**Description**: Moderation-flagged content excluded

**Prerequisites**:
- reel_flagged: moderationStatus=FLAGGED

**Test Steps**:
1. Call `GET /api/v1/explore/trending`

**Expected Results**:
- reel_flagged: ❌ Not visible

**Pass Criteria**:
- ✅ Flagged content hidden from explore

---

### TC-PRIV-005: NSFW Content Filtered

**Priority**: High  
**Description**: NSFW/violent content excluded

**Prerequisites**:
- reel_nsfw: isNSFW=true

**Test Steps**:
1. Call `GET /api/v1/explore/trending`

**Expected Results**:
- reel_nsfw: ❌ Not visible

**Pass Criteria**:
- ✅ NSFW filter enforced

---

### TC-PRIV-006: Location-Based Filtering

**Priority**: Medium  
**Description**: Only show deliverable content

**Prerequisites**:
- test_customer_1: Location = Mumbai
- test_chef_delhi: Location = Delhi (700 km away, not deliverable)

**Test Steps**:
1. Call `GET /api/v1/explore/recommended` from Mumbai

**Expected Results**:
- test_chef_delhi's reels: ❌ Not visible (too far)

**Pass Criteria**:
- ✅ Distance filter enforced
- ✅ Only deliverable items shown

---

### TC-PRIV-007: User's Own Content Excluded

**Priority**: Low  
**Description**: User doesn't see own content in explore

**Prerequisites**:
- test_chef_gold logs in as customer

**Test Steps**:
1. test_chef_gold calls `GET /api/v1/explore/trending`

**Expected Results**:
- test_chef_gold's own reels: ❌ Excluded from results

**Pass Criteria**:
- ✅ Own content filtered out

---

### TC-PRIV-008: Privacy Rules Applied Consistently

**Priority**: High  
**Description**: All sections respect privacy

**Test Steps**:
1. Block test_chef_blocked
2. Call all sections (trending, recommended, new_dish, exclusive)
3. Verify blocked content absent in ALL sections

**Expected Results**:
- test_chef_blocked absent from all sections

**Pass Criteria**:
- ✅ Consistent privacy enforcement

---

## Caching & Performance Tests

### TC-CACHE-001: Trending Section Cached

**Priority**: High  
**Description**: Trending results cached for 1 hour

**Test Steps**:
1. Call `GET /api/v1/explore/trending` (cold cache)
2. Measure response time (T1)
3. Call again immediately (warm cache)
4. Measure response time (T2)

**Expected Results**:
- T1 > 300ms (cold cache, DB query)
- T2 < 100ms (warm cache, Redis hit)
- T2 << T1

**Pass Criteria**:
- ✅ Cache hit significantly faster
- ✅ TTL = 1 hour

---

### TC-CACHE-002: Recommendations Cached Per User

**Priority**: High  
**Description**: User-specific recommendations cached

**Test Steps**:
1. test_customer_1 calls recommended (cold)
2. test_customer_2 calls recommended (cold)
3. test_customer_1 calls again (warm)

**Expected Results**:
- test_customer_1 cached separately from test_customer_2
- test_customer_1's second call hits cache

**Pass Criteria**:
- ✅ Per-user cache keys
- ✅ No cross-user cache leakage

---

### TC-CACHE-003: Cache Invalidation on Action

**Priority**: Medium  
**Description**: User actions invalidate recommendations

**Test Steps**:
1. test_customer_1 calls recommended
2. test_customer_1 follows a new chef
3. test_customer_1 calls recommended again

**Expected Results**:
- Recommendations update (cache invalidated)
- New chef's content appears

**Pass Criteria**:
- ✅ Follow action invalidates cache
- ✅ Fresh results returned

---

### TC-CACHE-004: Trending Aggregator Job

**Priority**: High  
**Description**: Hourly job pre-calculates trending scores

**Test Steps**:
1. Trigger trending aggregator manually
2. Verify Redis key `explore:trending:sorted` exists
3. Verify top 500 reels cached

**Expected Results**:
- Job completes in < 30 seconds
- Redis key has TTL = 1 hour
- Top 500 reels present

**Pass Criteria**:
- ✅ Job runs hourly
- ✅ Caches top 500 reels
- ✅ No errors

---

### TC-CACHE-005: Cache Hit Rate

**Priority**: Medium  
**Description**: Monitor cache effectiveness

**Test Steps**:
1. Make 100 explore API calls
2. Measure cache hits vs misses

**Expected Results**:
- Cache hit rate > 80%

**Pass Criteria**:
- ✅ High cache hit rate
- ✅ Reduced DB load

---

### TC-CACHE-006: Response Time SLA

**Priority**: High  
**Description**: All endpoints meet latency targets

**Test Steps**:
1. Measure P50, P95, P99 latencies
2. Compare to SLA targets

**Expected Results**:
- P50 < 200ms
- P95 < 500ms
- P99 < 1000ms

**Pass Criteria**:
- ✅ Meets latency SLA
- ✅ Consistent performance

---

## Chef Discovery Tests

### TC-CHEF-001: Trending Chefs

**Priority**: Medium  
**Endpoint**: `GET /api/v1/explore/chefs/trending`

**Test Steps**:
1. Call endpoint with `limit=20`

**Expected Results**:
- Returns 20 top-rated chefs
- Sorted by reputation score + activity
- Online chefs ranked higher

**Pass Criteria**:
- ✅ Reputation factored
- ✅ Recent activity factored

---

### TC-CHEF-002: Nearby Chefs

**Priority**: Medium  
**Endpoint**: `GET /api/v1/explore/chefs/nearby`

**Test Steps**:
1. Call with lat=19.0760, lng=72.8777 (Mumbai)
2. Call with maxDistance=5 (5 km radius)

**Expected Results**:
- Only chefs within 5 km returned
- Sorted by distance (nearest first)

**Pass Criteria**:
- ✅ Geospatial filter applied
- ✅ Distance calculated correctly

---

### TC-CHEF-003: No Location Provided

**Priority**: Low  
**Endpoint**: `GET /api/v1/explore/chefs/nearby`

**Test Steps**:
1. Call without lat/lng params

**Expected Results**:
```json
{
  "success": true,
  "message": "Location required for nearby chefs",
  "data": { "chefs": [], "nextCursor": null }
}
```

**Pass Criteria**:
- ✅ Graceful handling
- ✅ Returns empty array (not error)

---

### TC-CHEF-004: Chef Pagination

**Priority**: Low  
**Endpoint**: `GET /api/v1/explore/chefs/trending`

**Test Steps**:
1. Call with `limit=10`
2. Extract `nextCursor`
3. Call with cursor

**Expected Results**:
- Page 1: 10 chefs
- Page 2: Next 10 chefs, no overlap

**Pass Criteria**:
- ✅ Cursor-based pagination works

---

### TC-CHEF-005: Chef Profile Data

**Priority**: Medium  
**Description**: Verify chef card data complete

**Test Steps**:
1. Call trending chefs endpoint
2. Inspect chef object

**Expected Results**:
```json
{
  "userId": "uuid",
  "username": "chef_rakesh",
  "fullName": "Rakesh Kumar",
  "avatarUrl": "https://...",
  "reputation": { "tier": "gold", "score": 72 },
  "stats": {
    "followers": 2500,
    "totalOrders": 1200,
    "avgRating": 4.5
  },
  "isOnline": true,
  "distance": 2.5
}
```

**Pass Criteria**:
- ✅ All fields present
- ✅ Valid data types

---

## Category Browsing Tests

### TC-CAT-001: List Categories

**Priority**: Low  
**Endpoint**: `GET /api/v1/explore/categories`

**Test Steps**:
1. Call endpoint (no auth required)

**Expected Results**:
- Returns list of cuisine categories
- Each category has name, icon, reelCount

**Pass Criteria**:
- ✅ Public endpoint (no auth)
- ✅ Returns all active categories

---

### TC-CAT-002: Category Detail

**Priority**: Low  
**Endpoint**: `GET /api/v1/explore/category/:key/detail`

**Test Steps**:
1. Call with `key=NORTH_INDIAN`

**Expected Results**:
- Returns category info + reels
- Only North Indian reels returned

**Pass Criteria**:
- ✅ Correct filtering by category

---

### TC-CAT-003: Invalid Category Key

**Priority**: Low  
**Endpoint**: `GET /api/v1/explore/category/INVALID_KEY`

**Test Steps**:
1. Call with non-existent category

**Expected Results**:
```json
{
  "success": false,
  "message": "Category not found",
  "errorCode": "CATEGORY_NOT_FOUND"
}
```

**Pass Criteria**:
- ✅ Status = 404
- ✅ Clear error message

---

### TC-CAT-004: Category Reel Counts

**Priority**: Low  
**Description**: Verify reel counts accurate

**Test Steps**:
1. Call `/categories`
2. Check `reelCount` for each category
3. Verify against actual DB count

**Expected Results**:
- Reel counts match database

**Pass Criteria**:
- ✅ Accurate counts
- ✅ Updated regularly

---

## Test Execution Summary

### Pre-Test Checklist

- [ ] Staging environment deployed
- [ ] Test database seeded with test data
- [ ] Test users created and configured
- [ ] Redis/Valkey cache cleared
- [ ] Trending aggregator job run manually
- [ ] Monitoring tools enabled

### Post-Test Report

| Status | Count | Percentage |
|--------|-------|------------|
| ✅ Pass | - | - % |
| ❌ Fail | - | - % |
| ⚠️ Blocked | - | - % |
| ⏭️ Skipped | - | - % |

### Known Issues

| ID | Title | Severity | Status |
|----|-------|----------|--------|
| - | - | - | - |

---

## Related Documentation

- **Feature Overview**: `FEATURE_OVERVIEW.md`
- **Technical Guide**: `TECHNICAL_GUIDE.md`
- **Feed Module Tests**: `../feed/QA_TEST_CASES.md`

---

**[SLICE_COMPLETE ✅]**  
**Module**: Explore  
**Documentation**: QA Test Cases  
**Date**: February 14, 2026

---

## Dark Mode Regression Tests (Added March 2026)

### TC-EXPLORE-DM-001: Category Detail Screen in Dark Mode

**Type:** Bug Regression / Manual  
**Feature area:** Category detail screen (`app/explore/category/[categoryKey].tsx`)  
**Priority:** P1

**Preconditions:**
- Device set to dark appearance

**Steps:**
1. Open app in dark mode
2. Navigate to Explore tab
3. Tap any category (e.g. "Biryani")
4. Observe: category background, sort chips, item cards, popular badges, bottom load-more button

**Expected result:** Background `colors.background`, cards `colors.surface`, sort chips `colors.interactiveSubtle`, popular badge uses warning tint  
**Actual result (before fix):** White backgrounds on all cards and sort section  
**Fix applied:** `[categoryKey].tsx` converted to `makeStyles(colors)` factory; replaced `#fff`, `#f0f0f0`, `#f8f8f8`, `#1a1a1a` etc.  
**Regression test:** `apps/chefooz-app/src/app/explore/category/[categoryKey].tsx`  
**Status:** Fixed ✅

---

## Moderation & Content Safety Tests (Added March 2026)

### TC-EXPLORE-MOD-001: Rejected/Banned Reels Hidden from All Explore Feeds

**Type:** Bug Regression / Security / P0
**Feature area:** All explore section endpoints + View All page + Chef public profile

**Preconditions:**
- DB has at least one reel with `moderationStatus: 'rejected'`
- User is logged in

**Steps:**
1. Mark a reel as `rejected` in the moderation system
2. Open Explore tab — Trending, Recommended, New Dishes sections
3. Open any "View All" reel collection page (`/explore/reels/[sectionKey]`)
4. Open a chef's public profile page
5. Perform a category search (e.g., Main Course)

**Expected result:** The rejected reel does NOT appear in any of the above surfaces
**Actual result (before fix):** Rejected/banned reels appeared in all Explore sections and "View All" pages because `moderationStatus` filter was absent from 7 MongoDB queries
**Root cause:** `explore.service.ts` `getTrendingItems`, `getRecommendedItems`, `getNewDishItems`, `getPromotedItems`, `getVirtualSectionDetail`, `getReelsByCategory`, `getRankedExploreItems` — none had a moderation filter. `chef-public.service.ts` `getChefReels` was also missing it.
**Fix applied:** Added `moderationStatus: { $ne: 'rejected' }` to all 7 queries in `explore.service.ts` and 1 query in `chef-public.service.ts`. Legacy reels with `moderationStatus: undefined` (pre-moderation deployment) remain visible, matching the feed.service.ts behaviour.
**Regression test:** Manual API test — create a reel, set `moderationStatus: 'rejected'`, verify absent from GET `/api/v1/explore/sections`
**Status:** Fixed ✅

---

### TC-EXPLORE-NAV-001: Back from Reel Returns to Explore (Not Home)

**Type:** Bug Regression / Navigation / P1
**Feature area:** `(tabs)/reels/[reelId].tsx`, `explore.tsx`, `explore/reels/[sectionKey].tsx`, `explore/category/[categoryKey].tsx`

**Preconditions:**
- User is on Explore tab (not Home)

**Steps:**
1. Open Explore tab
2. Tap any reel thumbnail (from trending grid, trending themes, near-you strip, etc.)
3. Reel player opens fullscreen
4. Tap the back arrow (top-left)

**Expected result:** Returns to Explore tab
**Actual result (before fix):** Navigated to the Home tab
**Root cause:** `reels/[reelId]` is a `Tabs.Screen` (hidden tab), not a stack push inside the explore tab's stack. `router.back()` from a tab screen falls through the root navigation history to the initial tab (home), bypassing explore entirely.
**Fix applied:** All explore callers now pass a `source` URL param (e.g., `source: '/(tabs)/explore'` or `source: '/explore/reels/trending'`). In `handleBack`, `router.navigate(source)` is called when a source is present, which activates the correct screen without adding a forward history entry.
**Regression test:** Manual: open reel from explore → tap back → should be on explore tab
**Status:** Fixed ✅


### TC-EXPLORE-UI-001: TrendingTagsStrip "See All" Navigation

**Type:** Manual / UI
**Feature area:** TrendingTagsStrip component + explore.tsx

**Preconditions:**
- User is on Explore tab with hashtag data loaded

**Steps:**
1. Observe TrendingTagsStrip at top of Explore screen
2. Tap "See All →" button on the right of the "Trending" label row
3. Observe navigation destination

**Expected result:** Navigates to `/explore/reels/trending` showing the full trending reel grid
**Actual result (before fix):** No "See All" existed; the duplicate `ExploreTrendingThemes` section in the scroll provided it
**Fix applied:** Added `onSeeAll` prop to `TrendingTagsStrip`; removed the old `ExploreTrendingThemes` component from the scroll section. Import also removed from `explore.tsx`.
**Status:** Fixed ✅

---

---

### TC-EXPLORE-FILTER-001: "Food Fun" filter shows only 3 items while "Mixed Vibe" has more

**Type:** Bug Regression / Manual
**Feature area:** `[sectionKey].tsx` — client-side filter tabs
**Priority:** P1

**Preconditions:**
- User is on any Explore section page (e.g. "From the Community")
- Page has fewer than ~5 PROMOTIONAL reels in the first 30 returned

**Steps:**
1. Open any category section screen (not "Recommended for you")
2. Tap "🎬 Food Fun" filter tab
3. Note number of items displayed (e.g. 3)
4. Tap "🎉 Mixed Vibe" filter tab
5. Observe number of items displayed

**Expected result:** Mixed Vibe shows all reels (30+), Food Fun shows only the PROMOTIONAL subset
**Actual result (before fix):** Mixed Vibe showed only ~30 reels from page 2, skipping page 1
**Fix applied:** Page accumulation pattern — `accumulatedReels` state (Map<page, reels[]>) accumulates every fetched page; `allLoadedReels` derives a deduplicated flat list across all pages; `filteredReels` operates on `allLoadedReels` instead of `data?.reels`
**Regression test:** Manual — automated test requires mock React Query
**Status:** Fixed ✅

---

### TC-EXPLORE-FILTER-002: Switching filter tabs after load-more shows wrong dataset

**Type:** Bug Regression / Manual
**Feature area:** `[sectionKey].tsx` — filter tab switching + pagination
**Priority:** P1

**Preconditions:**
- User is on any Explore section page
- Active filter yields fewer items than screen height (< ~6 cards)

**Steps:**
1. Open a section with a sparse filter (e.g. "🎬 Food Fun" shows 3 items)
2. Scroll down slightly to trigger `onEndReached`
3. Wait for next page to load
4. Tap "🎉 Mixed Vibe" tab

**Expected result:** Mixed Vibe shows all reels loaded across both pages
**Actual result (before fix):** Mixed Vibe showed only page-2 reels; page-1 was lost because `data?.reels` was replaced by React Query on page increment
**Fix applied:** Same accumulation fix as TC-EXPLORE-FILTER-001
**Regression test:** Manual
**Status:** Fixed ✅

---

### TC-EXPLORE-FILTER-003: Filter tabs work in "Recommended for you" but not other sections

**Type:** Bug Regression / Manual
**Feature area:** `[sectionKey].tsx` — filter state + section density
**Priority:** P2

**Preconditions:**
- Navigate to both "Recommended for you" and another section (e.g. "New on Chefooz")

**Steps:**
1. Open "Recommended for you" → tap "🎬 Food Fun" → switch back to "🎉 Mixed Vibe" — correct
2. Open "New on Chefooz" → tap "🎬 Food Fun" → switch back to "🎉 Mixed Vibe" — observe

**Expected result:** Both sections should behave identically for filter switching
**Actual result (before fix):** "Recommended" worked because its page-1 density (~30 mixed reels) filled the viewport so `onEndReached` never fired — the bug was masked. Sparse sections triggered it.
**Fix applied:** Root cause resolved via accumulation — behaviour is now consistent regardless of page-1 density
**Regression test:** Manual
**Status:** Fixed ✅

---

### TC-EXPLORE-LOC-001: Dishes Near You shows only chefs within delivery range

**Type:** Bug Regression / Manual
**Feature area:** `ExploreMenuShowcase` + `getNearYouItems` backend
**Priority:** P0

**Preconditions:**
- User has a default delivery address set with valid lat/lng
- At least one chef with kitchen lat/lng set within 25 km exists
- At least one chef with kitchen lat/lng set outside 50 km exists

**Steps:**
1. Open Explore tab with a default address set
2. View "Order Something Delicious" or "Dishes Near You" section
3. Note the chef names shown
4. Compare against chefs known to be far away

**Expected result:** Only chefs within `min(EXPLORE_NEAR_YOU_RADIUS_KM, chef.deliveryRadiusKm)` km appear
**Actual result (before fix):** Global trending+recommended reels shown regardless of distance — subtitle "near you" was a lie
**Fix applied:** Implemented `getNearYouItems` with Haversine filtering on `ChefKitchen.latitude`/`longitude`; frontend prefers `nearYou` section; location passed from `useAddressStore`
**Regression test:** Manual — requires a test chef with known lat/lng set in `chef_kitchen` table
**Status:** Fixed ✅

---

### TC-EXPLORE-LOC-002: Dishes section falls back gracefully when no location available

**Type:** Manual
**Feature area:** `ExploreMenuShowcase` + `menuFoodItems` fallback logic
**Priority:** P1

**Preconditions:**
- User has NO default delivery address set (new user or address cleared)

**Steps:**
1. Open Explore tab
2. View the food showcase section

**Expected result:** Section shows "Popular Dishes / from home kitchens" (no "near you" text); content is populated from global trending+recommended
**Actual result (before fix):** Same global content shown but with "near you" subtitle — misleading
**Fix applied:** Frontend checks `userLat != null` before showing "near you" copy; `nearYou` section is empty so fallback to `trending + recommended` is used
**Regression test:** Manual
**Status:** Fixed ✅

---

## PAN-India Packaged Food Delivery — Test Cases (March 2026)

---

### TC-PACKAGED-001: National item with short shelf life blocked in explore

**Type:** Manual
**Feature area:** `ExploreMenuShowcase` + `enrichWithDeliveryFeasibility`
**Priority:** P0

**Preconditions:**
- User has a default delivery address with pincode set (e.g., Mumbai — 400001)
- A chef kitchen in Delhi (110001) has a menu item: `nationalDeliveryEnabled=true`, `dispatchBufferDays=1`, `shelfLifeDays=3`, `isPackagedFood=true`
- Shiprocket estimated transit from 110001 → 400001 is ~2 days

**Steps:**
1. Open Explore tab
2. Scroll to the national packaged item card

**Expected result:**
- "🇮🇳 Ships India" badge visible
- CTA shows "Not Available" (grey/disabled)
- Red reason text: e.g., "Shelf life too short to deliver to your location"

**Actual result (before feature):** CTA showed "Order Now" regardless — customer could order an item guaranteed to arrive expired

**Fix applied:** `enrichWithDeliveryFeasibility()` calls shelf-life domain rule; frontend disables CTA when `isOrderable=false`
**Regression test:** `libs/domain/src/lib/packaged-food-delivery.spec.ts`
**Status:** Fixed ✅

---

### TC-PACKAGED-002: National item with long shelf life — CTA enabled

**Type:** Manual
**Feature area:** `ExploreMenuShowcase` + `PackagedDeliveryService`
**Priority:** P0

**Preconditions:**
- User in Bangalore (560001), chef kitchen in Delhi (110001)
- Menu item: `nationalDeliveryEnabled=true`, `dispatchBufferDays=1`, `shelfLifeDays=14`

**Steps:**
1. Open Explore tab
2. View the national packaged item card

**Expected result:**
- "🇮🇳 Ships India" badge visible
- "Delivers in N days" label shown (totalDaysFromOrder)
- CTA = "Order Now" (active, enabled)

**Actual result (before feature):** No badge, no days label
**Fix applied:** Full national delivery enrichment pipeline
**Regression test:** Manual
**Status:** Fixed ✅

---

### TC-PACKAGED-003: No user address → CTA optimistically enabled

**Type:** Manual
**Feature area:** `PackagedDeliveryService.checkFeasibility`
**Priority:** P1

**Preconditions:**
- User has NO default delivery address (new account or address cleared)
- A national item with short shelf life exists in explore

**Steps:**
1. Open Explore tab (no address set)
2. View the national packaged item card

**Expected result:**
- "🇮🇳 Ships India" badge visible
- CTA = "Order Now" (enabled — device location unknown, cannot gate)
- No reason text shown

**Rationale:** Blocking the CTA with no location data harms conversion; the final gate exists at cart validation.
**Regression test:** Manual
**Status:** Verified ✅

---

### TC-PACKAGED-004: Shiprocket returns not-serviceable route → CTA blocked

**Type:** Manual (requires Shiprocket staging)
**Feature area:** `ShiprocketClient` + `PackagedDeliveryService`
**Priority:** P1

**Preconditions:**
- Origin pincode and destination pincode are a known non-serviceable pair in Shiprocket staging

**Steps:**
1. User's default address set to a remote non-serviceable pincode
2. Open Explore tab with the national item from that kitchen

**Expected result:** CTA disabled; reason text shows route is not serviceable
**Regression test:** Mock Shiprocket returning empty couriers array in `packaged-delivery.service.spec.ts` (to be written)
**Status:** Verified via manual review ✅

---

### TC-PACKAGED-005: National item badge visibility

**Type:** Manual
**Feature area:** `ExploreMenuShowcase` visual
**Priority:** P2

**Steps:**
1. View any reel card with `deliveryType='national'`
2. Check image overlay

**Expected result:** "🇮🇳 Ships India" badge visible bottom-right of image; does not appear on local items
**Status:** Verified ✅

---

### TC-PACKAGED-006: Cart validates and removes not_deliverable item

**Type:** Manual / Automated
**Feature area:** `cart.service.ts` → `validateCart()`
**Priority:** P0

**Preconditions:**
- Cart contains a `nationalDeliveryEnabled=true` item
- User's address → chef kitchen distance makes `isOrderable=false`

**Steps:**
1. Add item to cart
2. Navigate to cart / call `validateCart`

**Expected result:**
- Item removed from cart automatically
- `CartValidationChange.type = 'not_deliverable'` returned
- Cart shows appropriate removal message to user

**Regression test:** Manual
**Status:** Implemented ✅

---

### TC-PACKAGED-007: Strict shelf-life boundary (buffer + delivery === shelfLife)

**Type:** Automated unit test
**Feature area:** `libs/domain/src/lib/packaged-food-delivery.ts`
**Priority:** P0

**Scenario:** `shelfLifeDays=3`, `dispatchBufferDays=1`, `estimatedDeliveryDays=2` → sum equals shelf life

**Expected result:** `isPackagedFoodOrderable` returns `false` (arrives on expiry day = NOT orderable)
**Regression test:** `libs/domain/src/lib/packaged-food-delivery.spec.ts` → "returns false when buffer + delivery === shelfLife"
**Status:** Passing ✅

---

**Last Updated**: March 2026
