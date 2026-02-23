# Search Module - QA Test Cases

**Version:** 1.0  
**Last Updated:** February 14, 2026  
**Module:** Search & Elasticsearch  
**Test Environment:** Staging + Production  
**Total Test Cases:** 48

---

## 📋 Table of Contents

1. [Test Environment Setup](#test-environment-setup)
2. [Test Categories](#test-categories)
3. [Elasticsearch Search Tests](#elasticsearch-search-tests)
4. [Fallback Mode Tests](#fallback-mode-tests)
5. [Filter & Refinement Tests](#filter--refinement-tests)
6. [Geospatial Search Tests](#geospatial-search-tests)
7. [Auto-Suggestion Tests](#auto-suggestion-tests)
8. [Performance Tests](#performance-tests)
9. [Error Handling Tests](#error-handling-tests)
10. [Security Tests](#security-tests)

---

## Test Environment Setup

### Prerequisites

| Component | Required State |
|-----------|----------------|
| **Elasticsearch** | Running on staging (v8+) |
| **Test Data** | 500+ indexed reels |
| **User Accounts** | Customer + Chef + Admin |
| **Location Services** | Enabled (for geospatial tests) |

### Test User Credentials

> ⚠️ **Chefooz uses OTP-only authentication (no passwords)**. Login via mobile phone number + OTP sent via WhatsApp (primary) or Twilio SMS (fallback). Use the `/api/v1/auth/v2/send-otp` → `/api/v1/auth/v2/verify-otp` flow to obtain a JWT token before running tests.

```bash
# Customer Account (phone-number based)
Phone: +919876543210   # Obtain JWT via OTP on mobile

# Chef Account
Phone: +919876543212   # Obtain JWT via OTP on mobile

# Admin Account
Phone: +919876543220   # Admin JWT via admin portal login
```

**OTP Auth Flow (to get JWT for tests)**:
```powershell
# Step 1: Send OTP
$otpResponse = Invoke-WebRequest -Uri "https://api-staging.chefooz.com/api/v1/auth/v2/send-otp" `
    -Method POST `
    -ContentType "application/json" `
    -Body '{"phoneNumber": "+919876543210"}'
$requestId = ($otpResponse.Content | ConvertFrom-Json).data.requestId

# Step 2: Verify OTP (use OTP received on phone via WhatsApp/SMS)
$verifyResponse = Invoke-WebRequest -Uri "https://api-staging.chefooz.com/api/v1/auth/v2/verify-otp" `
    -Method POST `
    -ContentType "application/json" `
    -Body "{`"requestId`": `"$requestId`", `"otp`": `"<OTP_FROM_WHATSAPP_OR_SMS>`"}"
$jwtToken = ($verifyResponse.Content | ConvertFrom-Json).data.accessToken
```

### API Base URL

```
Staging: https://api-staging.chefooz.com/api/v1
Production: https://api.chefooz.com/api/v1
```

### Test Data Requirements

```bash
# Minimum test data needed
- 500+ reels with diverse captions
- 100+ reels with hashtags
- 50+ reels marked as promoted
- 200+ menu items from various chefs
- 50+ chef profiles with ratings
- Reels distributed across Mumbai geography
```

---

## Test Categories

| Category | Test Count | Priority |
|----------|-----------|----------|
| **Elasticsearch Search** | 12 | High |
| **Fallback Mode** | 6 | High |
| **Filter & Refinement** | 8 | High |
| **Geospatial Search** | 6 | Medium |
| **Auto-Suggestion** | 4 | Medium |
| **Performance** | 6 | High |
| **Error Handling** | 4 | High |
| **Security** | 2 | High |

---

## Elasticsearch Search Tests

### Test Case 1: Basic Keyword Search

**ID**: `ES-001`  
**Priority**: High  
**Type**: Functional

**Steps**:
1. Login as customer
2. Search for "butter chicken"
3. Verify results returned

**Expected Results**:
- ✅ Status 200
- ✅ `success: true`
- ✅ At least 5 results
- ✅ All results contain "butter" or "chicken" in caption/hashtags
- ✅ Results sorted by relevance (score descending)
- ✅ Highlights present (caption wrapped in `<em>` tags)

**API Call**:
```bash
GET /v1/search/dishes?query=butter%20chicken&page=1&limit=20
Authorization: Bearer <customer_token>
```

---

### Test Case 2: Fuzzy Matching (Typo Tolerance)

**ID**: `ES-002`  
**Priority**: High  
**Type**: Functional

**Steps**:
1. Search for "butr chiken" (intentional typos)
2. Verify results match "butter chicken"

**Expected Results**:
- ✅ Returns results for "butter chicken"
- ✅ Score slightly lower than exact match
- ✅ No "No results found" error

**API Call**:
```bash
GET /v1/search/dishes?query=butr%20chiken
```

**Validation**:
```typescript
expect(response.data.items.length).toBeGreaterThan(0);
expect(response.data.items[0].caption).toContain('butter chicken');
```

---

### Test Case 3: Multi-Field Search (Caption + Hashtags)

**ID**: `ES-003`  
**Priority**: High  
**Type**: Functional

**Steps**:
1. Search for "biryani"
2. Verify results from both caption and hashtags

**Expected Results**:
- ✅ Results where caption contains "biryani"
- ✅ Results where hashtags contain "#biryani"
- ✅ Caption matches boosted 3× (higher score)
- ✅ Hashtag matches boosted 2× (medium score)

**Validation**:
```typescript
const items = response.data.items;
const captionMatches = items.filter(i => i.caption.includes('biryani'));
const hashtagMatches = items.filter(i => i.hashtags.includes('biryani'));

expect(captionMatches.length + hashtagMatches.length).toBe(items.length);
```

---

### Test Case 4: Phrase Search (Exact Order)

**ID**: `ES-004`  
**Priority**: Medium  
**Type**: Functional

**Steps**:
1. Search for "chicken tikka masala"
2. Verify phrase order preserved

**Expected Results**:
- ✅ Top results have exact phrase
- ✅ Results with all words but different order ranked lower
- ✅ Score reflects phrase proximity

---

### Test Case 5: Boolean Search (AND Logic)

**ID**: `ES-005`  
**Priority**: Medium  
**Type**: Functional

**Steps**:
1. Search for "paneer AND butter"
2. Verify both terms present

**Expected Results**:
- ✅ All results contain both "paneer" and "butter"
- ✅ Results with "paneer butter masala" rank highest
- ✅ No results with only one term

---

### Test Case 6: Highlight Matching Text

**ID**: `ES-006`  
**Priority**: Medium  
**Type**: Functional

**Steps**:
1. Search for "tandoori chicken"
2. Verify highlights present

**Expected Results**:
- ✅ Highlights returned in response
- ✅ Matching text wrapped in `<em>` tags
- ✅ Highlights for caption field
- ✅ Highlights for hashtags field (if matched)

**Example Response**:
```json
{
  "highlights": {
    "caption": ["Making delicious <em>tandoori chicken</em> at home"]
  }
}
```

---

### Test Case 7: Promoted Content Boost

**ID**: `ES-007`  
**Priority**: High  
**Type**: Business Logic

**Steps**:
1. Search for "biryani"
2. Compare scores of promoted vs non-promoted

**Expected Results**:
- ✅ Promoted content has 2× boost
- ✅ Promoted content appears higher in results (if similar relevance)
- ✅ Non-promoted content with higher base relevance can still rank higher

**Validation**:
```typescript
const promoted = response.data.items.find(i => i.promoted === true);
const nonPromoted = response.data.items.find(i => i.promoted === false);

// Promoted should have higher score (if similar base relevance)
if (promoted && nonPromoted) {
  expect(promoted.score).toBeGreaterThanOrEqual(nonPromoted.score * 0.8);
}
```

---

### Test Case 8: Recent Content Boost

**ID**: `ES-008`  
**Priority**: Medium  
**Type**: Business Logic

**Steps**:
1. Search for "pasta"
2. Filter by recent (last 7 days)
3. Verify recent content ranked higher

**Expected Results**:
- ✅ Recent content (< 7 days) has 1.2× boost
- ✅ Recent content appears before older content (if similar relevance)
- ✅ createdAt timestamp within 7 days

**Validation**:
```typescript
const recent = response.data.items.filter(i => {
  const daysSince = (Date.now() - new Date(i.createdAt).getTime()) / (1000 * 60 * 60 * 24);
  return daysSince <= 7;
});

expect(recent.length).toBeGreaterThan(0);
```

---

### Test Case 9: Relevance Sorting (Default)

**ID**: `ES-009`  
**Priority**: High  
**Type**: Functional

**Steps**:
1. Search for "chicken curry"
2. Do NOT specify sortBy parameter
3. Verify relevance sorting

**Expected Results**:
- ✅ Results sorted by Elasticsearch score
- ✅ Most relevant result first (highest score)
- ✅ Score values descending
- ✅ Tie-breaker: likes descending

**Validation**:
```typescript
const scores = response.data.items.map(i => i.score);
for (let i = 1; i < scores.length; i++) {
  expect(scores[i]).toBeLessThanOrEqual(scores[i - 1]);
}
```

---

### Test Case 10: Recent Sorting

**ID**: `ES-010`  
**Priority**: Medium  
**Type**: Functional

**Steps**:
1. Search with `sortBy=recent`
2. Verify chronological order

**Expected Results**:
- ✅ Results sorted by createdAt descending
- ✅ Newest reels first
- ✅ Timestamps descending

**API Call**:
```bash
GET /v1/search/dishes?query=biryani&sortBy=recent
```

**Validation**:
```typescript
const timestamps = response.data.items.map(i => new Date(i.createdAt).getTime());
for (let i = 1; i < timestamps.length; i++) {
  expect(timestamps[i]).toBeLessThanOrEqual(timestamps[i - 1]);
}
```

---

### Test Case 11: Popular Sorting

**ID**: `ES-011`  
**Priority**: Medium  
**Type**: Functional

**Steps**:
1. Search with `sortBy=popular`
2. Verify popularity order

**Expected Results**:
- ✅ Sorted by views descending (primary)
- ✅ Sorted by likes descending (secondary)
- ✅ Most viewed/liked content first

**API Call**:
```bash
GET /v1/search/dishes?query=pasta&sortBy=popular
```

---

### Test Case 12: Trending Sorting

**ID**: `ES-012`  
**Priority**: Medium  
**Type**: Functional

**Steps**:
1. Search with `sortBy=trending`
2. Verify trending algorithm

**Expected Results**:
- ✅ Sorted by likes (primary)
- ✅ Sorted by saves (secondary)
- ✅ Sorted by recent (tertiary)
- ✅ High engagement + recent = top results

**API Call**:
```bash
GET /v1/search/dishes?query=ramen&sortBy=trending
```

---

## Fallback Mode Tests

### Test Case 13: Fallback Activation (ES Down)

**ID**: `FB-001`  
**Priority**: High  
**Type**: Resilience

**Steps**:
1. Stop Elasticsearch service (admin)
2. Search for "biryani"
3. Verify MongoDB fallback

**Expected Results**:
- ✅ Status 200 (no error)
- ✅ `success: true`
- ✅ Results returned from MongoDB
- ✅ Warning message: "Limited search features available"
- ✅ No highlights in response
- ✅ No fuzzy matching (exact terms only)

**Validation**:
```typescript
expect(response.data.warning).toBeDefined();
expect(response.data.items[0].highlights).toBeUndefined();
```

---

### Test Case 14: Fallback Regex Search

**ID**: `FB-002`  
**Priority**: High  
**Type**: Functional

**Steps**:
1. With ES disabled, search for "butter chicken"
2. Verify regex-based matching

**Expected Results**:
- ✅ Case-insensitive match
- ✅ Results with "Butter Chicken", "butter chicken", "BUTTER CHICKEN"
- ✅ No typo tolerance (must be exact)

---

### Test Case 15: Fallback No Fuzzy Matching

**ID**: `FB-003`  
**Priority**: Medium  
**Type**: Negative

**Steps**:
1. With ES disabled, search for "butr chiken" (typos)
2. Verify no results

**Expected Results**:
- ✅ Zero results (no fuzzy matching in fallback)
- ✅ Message: "No results found"
- ✅ Suggestion: "Try different keywords"

---

### Test Case 16: Fallback No Relevance Scoring

**ID**: `FB-004`  
**Priority**: Medium  
**Type**: Functional

**Steps**:
1. With ES disabled, search for "pasta"
2. Verify no score field

**Expected Results**:
- ✅ No `score` field in results
- ✅ No `highlights` field
- ✅ Results sorted by creation date (default)

---

### Test Case 17: Fallback Geospatial Support

**ID**: `FB-005`  
**Priority**: Medium  
**Type**: Functional

**Steps**:
1. With ES disabled, search with lat/lng parameters
2. Verify MongoDB $near query

**Expected Results**:
- ✅ Geospatial still works (MongoDB $near)
- ✅ Results within specified radius
- ✅ Distance calculated correctly

---

### Test Case 18: Automatic Recovery (ES Back Online)

**ID**: `FB-006`  
**Priority**: High  
**Type**: Resilience

**Steps**:
1. Start Elasticsearch service
2. Wait 10 seconds (health check)
3. Search for "biryani"
4. Verify ES mode restored

**Expected Results**:
- ✅ Elasticsearch mode active
- ✅ Highlights present
- ✅ Fuzzy matching works
- ✅ No warning message

---

## Filter & Refinement Tests

### Test Case 19: Hashtag Filter (Single)

**ID**: `FR-001`  
**Priority**: High  
**Type**: Functional

**Steps**:
1. Search with `hashtags=northindian`
2. Verify all results have hashtag

**Expected Results**:
- ✅ All results contain #northindian
- ✅ Results with query + hashtag ranked higher
- ✅ Results with only hashtag (no query match) ranked lower

**API Call**:
```bash
GET /v1/search/dishes?query=chicken&hashtags=northindian
```

**Validation**:
```typescript
response.data.items.forEach(item => {
  expect(item.hashtags).toContain('northindian');
});
```

---

### Test Case 20: Hashtag Filter (Multiple)

**ID**: `FR-002`  
**Priority**: Medium  
**Type**: Functional

**Steps**:
1. Search with `hashtags=northindian,vegetarian`
2. Verify OR logic (either hashtag)

**Expected Results**:
- ✅ Results with #northindian OR #vegetarian
- ✅ Results with both hashtags ranked higher

**API Call**:
```bash
GET /v1/search/dishes?query=curry&hashtags=northindian,vegetarian
```

---

### Test Case 21: Chef ID Filter (Single)

**ID**: `FR-003`  
**Priority**: High  
**Type**: Functional

**Steps**:
1. Get a chef's userId
2. Search with `chefIds=<userId>`
3. Verify only that chef's reels

**Expected Results**:
- ✅ All results from specified chef
- ✅ author.userId matches filter

**API Call**:
```bash
GET /v1/search/dishes?query=biryani&chefIds=chef-uuid-123
```

---

### Test Case 22: Chef ID Filter (Multiple)

**ID**: `FR-004`  
**Priority**: Medium  
**Type**: Functional

**Steps**:
1. Search with `chefIds=chef1,chef2`
2. Verify results from either chef

**Expected Results**:
- ✅ Results from chef1 OR chef2
- ✅ No results from other chefs

---

### Test Case 23: Promoted Only Filter

**ID**: `FR-005`  
**Priority**: Medium  
**Type**: Functional

**Steps**:
1. Search with `promotedOnly=true`
2. Verify only promoted content

**Expected Results**:
- ✅ All results have `promoted: true`
- ✅ No organic content in results

**API Call**:
```bash
GET /v1/search/dishes?query=pasta&promotedOnly=true
```

---

### Test Case 24: Reel Purpose Filter

**ID**: `FR-006`  
**Priority**: Medium  
**Type**: Functional

**Steps**:
1. Search with `reelPurpose=MENU_SHOWCASE`
2. Verify only menu showcase reels

**Expected Results**:
- ✅ All results have `reelPurpose: "MENU_SHOWCASE"`
- ✅ No story or other content types

**API Call**:
```bash
GET /v1/search/dishes?query=special&reelPurpose=MENU_SHOWCASE
```

---

### Test Case 25: Menu Item Price Filter

**ID**: `FR-007`  
**Priority**: Medium  
**Type**: Functional

**Steps**:
1. Search menu items with `minPrice=200&maxPrice=500`
2. Verify price range

**Expected Results**:
- ✅ All items priced between ₹200-₹500
- ✅ No items outside range

**API Call**:
```bash
GET /v1/search/menu-items?query=biryani&minPrice=200&maxPrice=500
```

**Validation**:
```typescript
response.data.items.forEach(item => {
  expect(item.price).toBeGreaterThanOrEqual(200);
  expect(item.price).toBeLessThanOrEqual(500);
});
```

---

### Test Case 26: Vegetarian Only Filter

**ID**: `FR-008`  
**Priority**: Medium  
**Type**: Functional

**Steps**:
1. Search menu items with `vegOnly=true`
2. Verify all vegetarian

**Expected Results**:
- ✅ All items have 'veg' tag
- ✅ No non-vegetarian items

**API Call**:
```bash
GET /v1/search/menu-items?query=curry&vegOnly=true
```

---

## Geospatial Search Tests

### Test Case 27: Location-Based Search (5km Radius)

**ID**: `GS-001`  
**Priority**: High  
**Type**: Functional

**Steps**:
1. Search with `lat=19.0760&lng=72.8777&radiusKm=5` (Mumbai)
2. Verify all results within 5km

**Expected Results**:
- ✅ All results have distance ≤ 5km
- ✅ Results sorted by distance (if sortBy=distance)
- ✅ Distance field present in response

**API Call**:
```bash
GET /v1/search/dishes?query=biryani&lat=19.0760&lng=72.8777&radiusKm=5
```

**Validation**:
```typescript
response.data.items.forEach(item => {
  expect(item.distance).toBeDefined();
  expect(item.distance).toBeLessThanOrEqual(5);
});
```

---

### Test Case 28: Location-Based Search (No Results Outside Radius)

**ID**: `GS-002`  
**Priority**: High  
**Type**: Negative

**Steps**:
1. Search with very small radius (0.5km)
2. Verify strict filtering

**Expected Results**:
- ✅ Zero or few results
- ✅ No results beyond 0.5km
- ✅ Message if no results: "No chefs nearby"

---

### Test Case 29: Distance Sorting

**ID**: `GS-003`  
**Priority**: Medium  
**Type**: Functional

**Steps**:
1. Search with location + `sortBy=distance`
2. Verify distance ascending order

**Expected Results**:
- ✅ Closest result first
- ✅ Distances in ascending order
- ✅ Distance values accurate

**Validation**:
```typescript
const distances = response.data.items.map(i => i.distance);
for (let i = 1; i < distances.length; i++) {
  expect(distances[i]).toBeGreaterThanOrEqual(distances[i - 1]);
}
```

---

### Test Case 30: Location Without Radius (Default 10km)

**ID**: `GS-004`  
**Priority**: Low  
**Type**: Functional

**Steps**:
1. Search with lat/lng but NO radiusKm
2. Verify default 10km radius

**Expected Results**:
- ✅ All results within 10km
- ✅ Default radius applied

---

### Test Case 31: Invalid Coordinates

**ID**: `GS-005`  
**Priority**: High  
**Type**: Negative

**Steps**:
1. Search with invalid lat/lng (e.g., lat=200)
2. Verify error handling

**Expected Results**:
- ✅ Status 400
- ✅ Error: "Invalid coordinates"
- ✅ No results returned

---

### Test Case 32: Missing Coordinates (One Provided)

**ID**: `GS-006`  
**Priority**: Medium  
**Type**: Negative

**Steps**:
1. Search with only `lat=19.0760` (no lng)
2. Verify validation error

**Expected Results**:
- ✅ Status 400
- ✅ Error: "Both lat and lng required"

---

## Auto-Suggestion Tests

### Test Case 33: Basic Suggestions

**ID**: `SG-001`  
**Priority**: High  
**Type**: Functional

**Steps**:
1. Call `/search/suggest?query=but`
2. Verify suggestions returned

**Expected Results**:
- ✅ Status 200
- ✅ 5-10 suggestions
- ✅ All suggestions start with "but"
- ✅ Suggestions sorted by frequency (most popular first)

**API Call**:
```bash
GET /v1/search/suggest?query=but
```

**Expected Response**:
```json
{
  "suggestions": [
    { "text": "butter chicken", "type": "dish", "frequency": 1500 },
    { "text": "butter naan", "type": "dish", "frequency": 800 },
    { "text": "butter masala", "type": "dish", "frequency": 450 }
  ]
}
```

---

### Test Case 34: Minimum Query Length (3 chars)

**ID**: `SG-002`  
**Priority**: High  
**Type**: Validation

**Steps**:
1. Call `/search/suggest?query=bu` (2 chars)
2. Verify validation error

**Expected Results**:
- ✅ Status 400
- ✅ Error: "Query must be at least 3 characters"

---

### Test Case 35: No Suggestions Available

**ID**: `SG-003`  
**Priority**: Medium  
**Type**: Functional

**Steps**:
1. Call `/search/suggest?query=xyzabc`
2. Verify empty response

**Expected Results**:
- ✅ Status 200
- ✅ Empty suggestions array
- ✅ Message: "No suggestions found"

---

### Test Case 36: Suggestion Types (Dish vs Chef)

**ID**: `SG-004`  
**Priority**: Low  
**Type**: Functional

**Steps**:
1. Call `/search/suggest?query=che`
2. Verify mixed types

**Expected Results**:
- ✅ Suggestions with `type: "dish"`
- ✅ Suggestions with `type: "chef"`
- ✅ Types clearly labeled

---

## Performance Tests

### Test Case 37: Response Time (P50)

**ID**: `PF-001`  
**Priority**: High  
**Type**: Performance

**Steps**:
1. Execute 100 searches (various queries)
2. Measure response times
3. Calculate P50 (median)

**Expected Results**:
- ✅ P50 ≤ 200ms (Elasticsearch mode)
- ✅ P50 ≤ 500ms (Fallback mode)

**Validation**:
```bash
for i in {1..100}; do
  curl -w "%{time_total}\n" -o /dev/null -s \
    "https://api-staging.chefooz.com/api/v1/search/dishes?q=biryani"
done | sort -n | awk 'NR==50'
```

---

### Test Case 38: Response Time (P95)

**ID**: `PF-002`  
**Priority**: High  
**Type**: Performance

**Steps**:
1. Execute 100 searches
2. Calculate P95 (95th percentile)

**Expected Results**:
- ✅ P95 ≤ 500ms (Elasticsearch mode)
- ✅ P95 ≤ 1000ms (Fallback mode)

---

### Test Case 39: Response Time (P99)

**ID**: `PF-003`  
**Priority**: Medium  
**Type**: Performance

**Steps**:
1. Execute 100 searches
2. Calculate P99 (99th percentile)

**Expected Results**:
- ✅ P99 ≤ 1000ms (Elasticsearch mode)
- ✅ P99 ≤ 2000ms (Fallback mode)

---

### Test Case 40: Large Result Set (100 results)

**ID**: `PF-004`  
**Priority**: Medium  
**Type**: Performance

**Steps**:
1. Search with `limit=100`
2. Measure response time and payload size

**Expected Results**:
- ✅ Response time ≤ 500ms
- ✅ Payload size < 1MB
- ✅ All 100 results valid

---

### Test Case 41: Deep Pagination (Page 50)

**ID**: `PF-005`  
**Priority**: Low  
**Type**: Performance

**Steps**:
1. Search with `page=50&limit=20` (offset 980)
2. Measure performance degradation

**Expected Results**:
- ✅ Response time ≤ 1000ms
- ⚠️ Warning: "Deep pagination not recommended"
- ✅ Suggestion: Use search_after for better performance

---

### Test Case 42: Rate Limit Enforcement

**ID**: `PF-006`  
**Priority**: High  
**Type**: Security

**Steps**:
1. Send 15 requests in 1 second
2. Verify rate limiting

**Expected Results**:
- ✅ First 10 requests: 200 OK
- ✅ Requests 11-15: 429 Too Many Requests
- ✅ Header: `Retry-After: 1`

**Validation**:
```bash
for i in {1..15}; do
  curl -w "%{http_code}\n" -o /dev/null -s \
    "https://api-staging.chefooz.com/api/v1/search/dishes?q=test" &
done
wait
```

---

## Error Handling Tests

### Test Case 43: Missing Query Parameter

**ID**: `EH-001`  
**Priority**: High  
**Type**: Validation

**Steps**:
1. Call `/search/dishes` without `query` parameter
2. Verify validation error

**Expected Results**:
- ✅ Status 400
- ✅ Error: "Query parameter is required"
- ✅ errorCode: "INVALID_QUERY"

---

### Test Case 44: Empty Query String

**ID**: `EH-002`  
**Priority**: Medium  
**Type**: Validation

**Steps**:
1. Call `/search/dishes?query=` (empty)
2. Verify validation error

**Expected Results**:
- ✅ Status 400
- ✅ Error: "Query cannot be empty"

---

### Test Case 45: Elasticsearch Connection Failure

**ID**: `EH-003`  
**Priority**: High  
**Type**: Resilience

**Steps**:
1. Simulate ES connection timeout
2. Search for "biryani"
3. Verify graceful fallback

**Expected Results**:
- ✅ Status 200 (no 500 error)
- ✅ Fallback mode activated
- ✅ Results from MongoDB
- ✅ Warning message present

---

### Test Case 46: Invalid Sort Parameter

**ID**: `EH-004`  
**Priority**: Medium  
**Type**: Validation

**Steps**:
1. Search with `sortBy=invalid_value`
2. Verify validation error

**Expected Results**:
- ✅ Status 400
- ✅ Error: "Invalid sortBy value"
- ✅ Valid values: "relevance, recent, popular, trending"

---

## Security Tests

### Test Case 47: Unauthorized Access

**ID**: `SEC-001`  
**Priority**: High  
**Type**: Security

**Steps**:
1. Call `/search/dishes` without Authorization header
2. Verify 401 error

**Expected Results**:
- ✅ Status 401
- ✅ Error: "Unauthorized"
- ✅ No data leaked

---

### Test Case 48: Admin Reindex Authorization

**ID**: `SEC-002`  
**Priority**: High  
**Type**: Security

**Steps**:
1. Login as customer (non-admin)
2. Call `POST /search/reindex`
3. Verify 403 error

**Expected Results**:
- ✅ Status 403
- ✅ Error: "Forbidden - Admin role required"
- ✅ Reindex NOT triggered

---

## Test Execution Summary

### Manual Testing Checklist

```markdown
- [ ] ES-001 to ES-012: Elasticsearch Search (12 tests)
- [ ] FB-001 to FB-006: Fallback Mode (6 tests)
- [ ] FR-001 to FR-008: Filters & Refinement (8 tests)
- [ ] GS-001 to GS-006: Geospatial Search (6 tests)
- [ ] SG-001 to SG-004: Auto-Suggestions (4 tests)
- [ ] PF-001 to PF-006: Performance (6 tests)
- [ ] EH-001 to EH-004: Error Handling (4 tests)
- [ ] SEC-001 to SEC-002: Security (2 tests)
```

### Automated Test Script

```typescript
// tests/search.e2e.spec.ts
describe('Search Module E2E', () => {
  let customerToken: string;
  let adminToken: string;
  
  beforeAll(async () => {
    customerToken = await getCustomerToken();
    adminToken = await getAdminToken();
  });
  
  describe('Elasticsearch Search', () => {
    it('ES-001: Basic keyword search', async () => {
      const response = await request(app)
        .get('/v1/search/dishes?query=butter%20chicken')
        .set('Authorization', `Bearer ${customerToken}`)
        .expect(200);
      
      expect(response.body.success).toBe(true);
      expect(response.body.data.items.length).toBeGreaterThan(0);
    });
    
    // ... more tests
  });
});
```

---

## Related Documentation

- **Feature Overview**: `FEATURE_OVERVIEW.md` (Business context)
- **Technical Guide**: `TECHNICAL_GUIDE.md` (API specs, implementation)
- **Explore Module**: `../explore/QA_TEST_CASES.md` (Related discovery tests)

---

**[SLICE_COMPLETE ✅]**  
**Module**: Search  
**Documentation**: QA Test Cases  
**Total Test Cases**: 48  
**Date**: February 14, 2026
