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

**Last Updated**: March 2026
