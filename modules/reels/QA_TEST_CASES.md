# Reels Module - QA Test Cases

**Module**: `apps/chefooz-apis/src/modules/reels`  
**Test Coverage**: Functional, Security, Performance, Integration  
**Version**: 1.0  
**Last Updated**: 2026-02-14

---

## 📋 Test Overview

### Test Strategy

| Test Type | Count | Automation | Priority |
|-----------|-------|------------|----------|
| **Functional** | 45 | 90% | High |
| **Security** | 12 | 100% | Critical |
| **Performance** | 8 | 75% | Medium |
| **Integration** | 10 | 80% | High |
| **Regression** | 15 | 100% | High |
| **Total** | 90 | 89% | - |

### Test Environment

**Prerequisites**:
- MongoDB 6.x running (reels collection)
- PostgreSQL 15.x running (users, orders, chef_menu_items tables)
- Valkey/Redis 7.x running (cache)
- Test data seeded (users, menu items, orders)
- JWT tokens for test users (customer, chef, admin)

**Test Data Requirements**:
- 3 test users: customer, chef, admin
- 10 test reels (5 promotional, 3 review, 2 menu)
- 5 test menu items (belonging to test chef)
- 3 test orders (with items)
- Redis cache pre-populated with like/save sets

---

## 🧪 Functional Test Cases

### Category 1: Reel Detail Retrieval (GET /detail/:mediaId)

#### TC-F001: Get reel detail with authenticated user
**Priority**: High  
**Automation**: ✅ Automated

**Pre-conditions**:
- Reel exists in database (reelId: `507f1f77bcf86cd799439011`)
- User is authenticated (JWT token valid)
- User has liked the reel (Redis set populated)

**Test Steps**:
1. Send GET request to `/api/v1/reels/detail/507f1f77bcf86cd799439011`
2. Include `Authorization: Bearer {token}` header

**Expected Result**:
- HTTP 200 OK
- Response body contains:
  - `success: true`
  - `data.author` object with username, avatar, reputation tier
  - `data.stats.isLiked: true`
  - `data.stats.isSaved: false` (assuming not saved)
  - `data.videoUrl` as HTTPS URL (not S3 URI)
  - `data.linkedOrder` object (if review reel)

**cURL**:
```bash
curl -X GET "https://api.chefooz.com/api/v1/reels/detail/507f1f77bcf86cd799439011" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

---

#### TC-F002: Get reel detail without authentication
**Priority**: Medium  
**Automation**: ✅ Automated

**Pre-conditions**:
- Reel exists in database

**Test Steps**:
1. Send GET request without Authorization header

**Expected Result**:
- HTTP 200 OK
- `data.stats.isLiked: false` (default)
- `data.stats.isSaved: false` (default)
- All other fields populated correctly

---

#### TC-F003: Get reel detail - reel not found
**Priority**: High  
**Automation**: ✅ Automated

**Test Steps**:
1. Send GET request with invalid mediaId: `invalid-id-123`

**Expected Result**:
- HTTP 404 Not Found
- Response body:
  ```json
  {
    "success": false,
    "message": "Reel with ID invalid-id-123 not found",
    "errorCode": "REEL_NOT_FOUND"
  }
  ```

---

#### TC-F004: Get reel detail - soft-deleted reel
**Priority**: High  
**Automation**: ✅ Automated

**Pre-conditions**:
- Reel exists with `deletedAt` timestamp set

**Test Steps**:
1. Send GET request to deleted reel

**Expected Result**:
- HTTP 404 Not Found
- `errorCode: "REEL_NOT_FOUND"`

---

#### TC-F005: Get reel detail - linked order snapshot included
**Priority**: High  
**Automation**: ✅ Automated

**Pre-conditions**:
- Reel with `reelPurpose: 'USER_REVIEW'`
- `linkedOrderId` references valid order
- Order has 3 items (one with qty=2)

**Test Steps**:
1. Send GET request to review reel

**Expected Result**:
- `data.linkedOrder` object present:
  ```json
  {
    "orderId": "uuid-order-123",
    "itemCount": 3,
    "totalPaise": 79900,
    "chefId": "uuid-chef-456",
    "previewImageUrl": "https://cdn.chefooz.com/items/dal-makhani.jpg",
    "itemPreviews": "Dal Makhani (2), Naan, Raita"
  }
  ```

---

#### TC-F006: Get reel detail - S3 URL conversion
**Priority**: Critical  
**Automation**: ✅ Automated

**Pre-conditions**:
- Reel has `videoUrl: "s3://chefooz-output/reels/video123.m3u8"`

**Test Steps**:
1. Send GET request

**Expected Result**:
- `data.videoUrl: "https://cdn.chefooz.com/reels/video123.m3u8"`
- `data.thumbnailUrl: "https://cdn.chefooz.com/reels/thumb123.jpg"`
- No S3 URIs exposed in response

---

### Category 2: Reel Deletion (DELETE /:reelId)

#### TC-F007: Delete own reel successfully
**Priority**: High  
**Automation**: ✅ Automated

**Pre-conditions**:
- User owns reel (userId matches)
- Reel not already deleted

**Test Steps**:
1. Send DELETE request to `/api/v1/reels/507f1f77bcf86cd799439011`
2. Include Authorization header

**Expected Result**:
- HTTP 200 OK
- Response: `{ "success": true, "message": "Reel deleted successfully" }`
- MongoDB: `deletedAt` timestamp set, `deletedBy` = userId
- Redis: Like/save sets cleared
- Subsequent GET returns 404

**cURL**:
```bash
curl -X DELETE "https://api.chefooz.com/api/v1/reels/507f1f77bcf86cd799439011" \
  -H "Authorization: Bearer {token}"
```

---

#### TC-F008: Delete reel - not owner
**Priority**: Critical  
**Automation**: ✅ Automated

**Pre-conditions**:
- Reel owned by User A
- Requesting user is User B

**Test Steps**:
1. Send DELETE request with User B's token

**Expected Result**:
- HTTP 403 Forbidden
- Response:
  ```json
  {
    "success": false,
    "message": "You can only delete your own reels",
    "errorCode": "FORBIDDEN"
  }
  ```
- Reel not deleted in database

---

#### TC-F009: Delete reel - already deleted
**Priority**: Medium  
**Automation**: ✅ Automated

**Pre-conditions**:
- Reel has `deletedAt` timestamp

**Test Steps**:
1. Send DELETE request

**Expected Result**:
- HTTP 400 Bad Request
- `message: "Reel already deleted"`

---

#### TC-F010: Delete reel - unauthenticated
**Priority**: High  
**Automation**: ✅ Automated

**Test Steps**:
1. Send DELETE request without Authorization header

**Expected Result**:
- HTTP 401 Unauthorized
- `errorCode: "UNAUTHORIZED"`

---

#### TC-F011: Delete reel - cache cleanup verification
**Priority**: High  
**Automation**: ✅ Automated

**Pre-conditions**:
- Redis sets exist: `reel:{reelId}:likes`, `reel:{reelId}:saves`

**Test Steps**:
1. Delete reel
2. Check Redis keys

**Expected Result**:
- Keys deleted from Redis
- `redis-cli KEYS "reel:507f1f77bcf86cd799439011:*"` returns empty

---

### Category 3: Menu Reel Creation (POST /menu)

#### TC-F012: Create menu reel successfully
**Priority**: High  
**Automation**: ✅ Automated

**Pre-conditions**:
- User role = 'chef'
- Menu items exist and belong to chef
- Feature flag `MENU_REELS` enabled

**Test Steps**:
1. POST to `/api/v1/reels/menu`
2. Body:
   ```json
   {
     "menuItemIds": ["uuid-item-1", "uuid-item-2"],
     "caption": "Special combo: Dal + Naan ₹299!"
   }
   ```

**Expected Result**:
- HTTP 201 Created
- Response:
  ```json
  {
    "success": true,
    "message": "Menu reel created successfully",
    "data": {
      "reelId": "507f1f77bcf86cd799439011",
      "mediaId": "507f1f77bcf86cd799439011"
    }
  }
  ```
- MongoDB document created with:
  - `reelPurpose: 'MENU_SHOWCASE'`
  - `linkedMenu.chefId` = requesting chef ID
  - `linkedMenu.estimatedPaise` = sum of menu item prices

**cURL**:
```bash
curl -X POST "https://api.chefooz.com/api/v1/reels/menu" \
  -H "Authorization: Bearer {chef_token}" \
  -H "Content-Type: application/json" \
  -d '{"menuItemIds":["uuid-1","uuid-2"],"caption":"Test"}'
```

---

#### TC-F013: Create menu reel - non-chef user
**Priority**: Critical  
**Automation**: ✅ Automated

**Pre-conditions**:
- User role = 'customer'

**Test Steps**:
1. POST to `/menu` with customer token

**Expected Result**:
- HTTP 403 Forbidden
- `errorCode: "CHEF_ONLY"`

---

#### TC-F014: Create menu reel - invalid menu items
**Priority**: High  
**Automation**: ✅ Automated

**Test Steps**:
1. POST with non-existent menuItemIds: `["invalid-uuid-1", "invalid-uuid-2"]`

**Expected Result**:
- HTTP 400 Bad Request
- `errorCode: "INVALID_MENU_ITEMS"`

---

#### TC-F015: Create menu reel - menu items belong to other chef
**Priority**: Critical  
**Automation**: ✅ Automated

**Pre-conditions**:
- Menu items owned by Chef A
- Requesting chef is Chef B

**Test Steps**:
1. POST with Chef B's token and Chef A's menu item IDs

**Expected Result**:
- HTTP 403 Forbidden
- `message: "You can only create menu reels for your own menu items"`

---

#### TC-F016: Create menu reel - feature flag disabled
**Priority**: High  
**Automation**: ✅ Automated

**Pre-conditions**:
- Environment variable `FEATURE_FLAG_MENU_REELS=false`

**Test Steps**:
1. POST to `/menu`

**Expected Result**:
- HTTP 403 Forbidden
- `errorCode: "FEATURE_DISABLED_MENU_REELS"`

---

#### TC-F017: Create menu reel - estimated price calculation
**Priority**: Medium  
**Automation**: ✅ Automated

**Pre-conditions**:
- Menu Item 1: price = ₹299 (29900 paise)
- Menu Item 2: price = ₹99 (9900 paise)

**Test Steps**:
1. Create menu reel with both items

**Expected Result**:
- `linkedMenu.estimatedPaise: 39800` (29900 + 9900)

---

#### TC-F018: Create menu reel - empty menuItemIds
**Priority**: Medium  
**Automation**: ✅ Automated

**Test Steps**:
1. POST with `menuItemIds: []`

**Expected Result**:
- HTTP 400 Bad Request
- Validation error: "At least one menu item must be selected"

---

### Category 4: User Reels (GET /user/:userId)

#### TC-F019: Get user reels - default filters
**Priority**: High  
**Automation**: ✅ Automated

**Pre-conditions**:
- User has 3 promotional, 2 review, 1 menu reel

**Test Steps**:
1. GET `/api/v1/reels/user/{userId}`
2. No query parameters

**Expected Result**:
- HTTP 200 OK
- Returns 5 reels (promotional + review)
- Excludes menu reel
- Sorted by `createdAt` DESC

---

#### TC-F020: Get user reels - include menu reels
**Priority**: Medium  
**Automation**: ✅ Automated

**Test Steps**:
1. GET with query: `?includeMenu=true`

**Expected Result**:
- Returns all 6 reels (promotional + review + menu)

---

#### TC-F021: Get user reels - exclude promotional
**Priority**: Medium  
**Automation**: ✅ Automated

**Test Steps**:
1. GET with query: `?includePromotional=false&includeMenu=true`

**Expected Result**:
- Returns only menu reels (1 reel)

---

#### TC-F022: Get user reels - pagination
**Priority**: High  
**Automation**: ✅ Automated

**Pre-conditions**:
- User has 100 reels

**Test Steps**:
1. GET with `?limit=20&skip=0`
2. GET with `?limit=20&skip=20`

**Expected Result**:
- First request returns reels 1-20
- Second request returns reels 21-40
- No duplicates between pages

---

#### TC-F023: Get user reels - tagged users included
**Priority**: Medium  
**Automation**: ✅ Automated

**Pre-conditions**:
- Reel has `taggedUserIds: ["uuid-user-2", "uuid-user-3"]`

**Test Steps**:
1. GET user reels

**Expected Result**:
- Response includes `taggedUsers` array:
  ```json
  "taggedUsers": [
    {
      "userId": "uuid-user-2",
      "username": "john_doe",
      "fullName": "John Doe",
      "avatarUrl": "https://cdn.chefooz.com/avatars/user2.jpg"
    }
  ]
  ```

---

#### TC-F024: Get user reels - author info included
**Priority**: Medium  
**Automation**: ✅ Automated

**Expected Result**:
- Each reel includes `author` object:
  ```json
  "author": {
    "userId": "uuid-user-123",
    "username": "chef_kumar",
    "fullName": "Kumar Singh",
    "avatarUrl": "https://cdn.chefooz.com/avatars/user123.jpg"
  }
  ```

---

#### TC-F025: Get user reels - soft-deleted excluded
**Priority**: High  
**Automation**: ✅ Automated

**Pre-conditions**:
- User has 5 reels, 2 are soft-deleted

**Test Steps**:
1. GET user reels

**Expected Result**:
- Returns only 3 reels (non-deleted)

---

### Category 5: Chef Menu Reels (GET /chef/:chefId/menu)

#### TC-F026: Get chef menu reels successfully
**Priority**: High  
**Automation**: ✅ Automated

**Pre-conditions**:
- Chef has 3 menu reels created

**Test Steps**:
1. GET `/api/v1/reels/chef/{chefId}/menu`

**Expected Result**:
- HTTP 200 OK
- Returns 3 menu reels
- All have `reelPurpose: 'MENU_SHOWCASE'`
- Sorted by `createdAt` DESC

---

#### TC-F027: Get chef menu reels - empty result
**Priority**: Medium  
**Automation**: ✅ Automated

**Pre-conditions**:
- Chef has no menu reels

**Test Steps**:
1. GET chef menu reels

**Expected Result**:
- HTTP 200 OK
- `data: []` (empty array)

---

#### TC-F028: Get chef menu reels - soft-deleted excluded
**Priority**: High  
**Automation**: ✅ Automated

**Pre-conditions**:
- Chef has 3 menu reels, 1 is soft-deleted

**Test Steps**:
1. GET chef menu reels

**Expected Result**:
- Returns 2 reels (excludes deleted)

---

#### TC-F029: Get chef menu reels - limit parameter
**Priority**: Low  
**Automation**: ✅ Automated

**Pre-conditions**:
- Chef has 50 menu reels

**Test Steps**:
1. GET with default limit (20)

**Expected Result**:
- Returns 20 reels (most recent)

---

---

## 🔒 Security Test Cases

### Category 6: Authentication & Authorization

#### TC-S001: JWT token validation - missing token
**Priority**: Critical  
**Automation**: ✅ Automated

**Test Steps**:
1. DELETE `/reels/{reelId}` without Authorization header

**Expected Result**:
- HTTP 401 Unauthorized

---

#### TC-S002: JWT token validation - expired token
**Priority**: Critical  
**Automation**: ✅ Automated

**Test Steps**:
1. Use expired JWT token (exp claim in past)

**Expected Result**:
- HTTP 401 Unauthorized

---

#### TC-S003: JWT token validation - malformed token
**Priority**: Critical  
**Automation**: ✅ Automated

**Test Steps**:
1. Use invalid JWT: `Bearer invalid.token.here`

**Expected Result**:
- HTTP 401 Unauthorized

---

#### TC-S004: Role-based access - chef-only endpoint
**Priority**: Critical  
**Automation**: ✅ Automated

**Test Steps**:
1. Customer user calls POST `/menu`

**Expected Result**:
- HTTP 403 Forbidden
- `errorCode: "CHEF_ONLY"`

---

#### TC-S005: Ownership validation - delete others' reel
**Priority**: Critical  
**Automation**: ✅ Automated

**Test Steps**:
1. User A tries to delete User B's reel

**Expected Result**:
- HTTP 403 Forbidden

---

#### TC-S006: SQL injection attempt - userId parameter
**Priority**: Critical  
**Automation**: ✅ Automated

**Test Steps**:
1. GET `/user/{userId}` with payload: `uuid-123' OR '1'='1`

**Expected Result**:
- TypeORM sanitizes input
- Returns 404 or empty result (not all users)

---

#### TC-S007: NoSQL injection attempt - reelId parameter
**Priority**: Critical  
**Automation**: ✅ Automated

**Test Steps**:
1. GET `/detail/{reelId}` with payload: `{"$ne": null}`

**Expected Result**:
- Mongoose sanitizes input
- Returns 404 (not all reels)

---

#### TC-S008: Path traversal attempt - mediaId parameter
**Priority**: Critical  
**Automation**: ✅ Automated

**Test Steps**:
1. GET `/detail/../../admin/secrets`

**Expected Result**:
- NestJS routing blocks path traversal
- Returns 404

---

#### TC-S009: XSS attempt - caption field
**Priority**: High  
**Automation**: ✅ Automated

**Test Steps**:
1. Create menu reel with caption: `<script>alert('XSS')</script>`

**Expected Result**:
- Caption stored as plain text (escaped)
- GET returns escaped HTML entities

---

#### TC-S010: CSRF protection - state-changing requests
**Priority**: High  
**Automation**: ⏸️ Manual

**Test Steps**:
1. Attempt DELETE request from external site without CSRF token

**Expected Result**:
- Request blocked by CSRF middleware

---

#### TC-S011: Rate limiting - delete endpoint
**Priority**: Medium  
**Automation**: ✅ Automated

**Test Steps**:
1. Send 100 DELETE requests in 1 second

**Expected Result**:
- First 10 succeed
- Remaining 90 return HTTP 429 Too Many Requests

---

#### TC-S012: Feature flag bypass attempt
**Priority**: High  
**Automation**: ✅ Automated

**Test Steps**:
1. Set `FEATURE_FLAG_MENU_REELS=false`
2. Try to create menu reel

**Expected Result**:
- HTTP 403 Forbidden
- `errorCode: "FEATURE_DISABLED_MENU_REELS"`

---

## ⚡ Performance Test Cases

### Category 7: Load & Stress Testing

#### TC-P001: Reel detail API - latency under load
**Priority**: High  
**Automation**: ✅ Automated (Artillery)

**Test Steps**:
1. Send 1000 requests/sec to GET `/detail/{reelId}` for 60 seconds

**Expected Result**:
- P50 latency: <50ms
- P95 latency: <200ms
- P99 latency: <500ms
- 0% error rate

**Artillery Config**:
```yaml
config:
  target: "https://api.chefooz.com"
  phases:
    - duration: 60
      arrivalRate: 1000
scenarios:
  - name: "Reel Detail"
    flow:
      - get:
          url: "/api/v1/reels/detail/507f1f77bcf86cd799439011"
          headers:
            Authorization: "Bearer {{token}}"
```

---

#### TC-P002: User reels API - pagination performance
**Priority**: Medium  
**Automation**: ✅ Automated

**Test Steps**:
1. User has 10,000 reels
2. GET with `limit=50&skip=9950`

**Expected Result**:
- Response time: <100ms
- Correct reels returned (9951-10000)

---

#### TC-P003: Menu reel creation - bulk operations
**Priority**: Medium  
**Automation**: ✅ Automated

**Test Steps**:
1. Create 100 menu reels concurrently (10 chefs × 10 reels each)

**Expected Result**:
- All 100 reels created successfully
- No race conditions or duplicates
- Average creation time: <300ms per reel

---

#### TC-P004: Delete reel - cache invalidation speed
**Priority**: Medium  
**Automation**: ✅ Automated

**Test Steps**:
1. Reel has 10,000 likes in Redis set
2. Delete reel

**Expected Result**:
- Redis DEL command completes in <10ms
- Total deletion time (including DB update): <100ms

---

#### TC-P005: Reel detail - MongoDB query optimization
**Priority**: High  
**Automation**: ✅ Automated

**Test Steps**:
1. Collection has 1M reels
2. Query single reel by `_id`

**Expected Result**:
- Query uses `_id` index
- Execution time: <5ms
- No full collection scan

---

#### TC-P006: User reels - compound index usage
**Priority**: High  
**Automation**: ✅ Automated

**Test Steps**:
1. Query: `{ userId: "uuid", deletedAt: null }`
2. Check MongoDB query plan

**Expected Result**:
- Uses compound index: `{ userId: 1, deletedAt: 1, createdAt: -1 }`
- Index scan (not collection scan)

---

#### TC-P007: Chef menu reels - query performance
**Priority**: Medium  
**Automation**: ✅ Automated

**Test Steps**:
1. Chef has 500 menu reels
2. GET chef menu reels

**Expected Result**:
- Query time: <20ms
- Uses index: `{ 'linkedMenu.chefId': 1, createdAt: -1 }`

---

#### TC-P008: S3 URL conversion - batch performance
**Priority**: Low  
**Automation**: ✅ Automated

**Test Steps**:
1. Convert 1000 S3 URIs to HTTPS URLs

**Expected Result**:
- Total time: <100ms (0.1ms per conversion)

---

## 🔗 Integration Test Cases

### Category 8: Module Integration

#### TC-I001: Media upload → Reel creation flow
**Priority**: Critical  
**Automation**: ✅ Automated

**Test Steps**:
1. Upload video via Media module
2. Media processing creates Reel document
3. GET reel detail

**Expected Result**:
- Reel exists with correct `mediaId`
- Video URL accessible
- Thumbnail generated

---

#### TC-I002: Order completion → Review reel creation
**Priority**: High  
**Automation**: ✅ Automated

**Test Steps**:
1. Complete order (status = DELIVERED)
2. Create review reel with `linkedOrderId`
3. GET reel detail

**Expected Result**:
- `linkedOrder` snapshot included
- `reelPurpose: 'USER_REVIEW'`
- `creatorOrderValue` matches order total

---

#### TC-I003: Like action → Redis cache update
**Priority**: High  
**Automation**: ✅ Automated

**Test Steps**:
1. User likes reel (via Social module)
2. Check Redis set: `reel:{reelId}:likes`
3. GET reel detail

**Expected Result**:
- User ID in Redis set
- `stats.isLiked: true` in response

---

#### TC-I004: Reel deletion → Feed removal
**Priority**: High  
**Automation**: ✅ Automated

**Test Steps**:
1. Reel visible in Explore feed
2. Delete reel
3. Refresh Explore feed

**Expected Result**:
- Reel not visible in feed
- Feed query includes `deletedAt: null` filter

---

#### TC-I005: Chef reputation update → Reel detail refresh
**Priority**: Medium  
**Automation**: ⏸️ Manual

**Test Steps**:
1. GET reel detail (reputation tier = Bronze)
2. Update chef reputation score to Gold level
3. GET reel detail again

**Expected Result**:
- `author.reputationLevel: 'gold'`

---

#### TC-I006: Menu item price change → Estimated price update
**Priority**: Low  
**Automation**: ⏸️ Manual

**Test Steps**:
1. Menu reel has `estimatedPaise: 29900`
2. Update menu item price
3. GET reel detail

**Expected Result**:
- `estimatedPaise` remains 29900 (snapshot not updated)

---

#### TC-I007: CDN URL generation - environment-based
**Priority**: Medium  
**Automation**: ✅ Automated

**Test Steps**:
1. Set `CDN_URL=https://staging-cdn.chefooz.com`
2. GET reel detail

**Expected Result**:
- `videoUrl` uses staging CDN prefix

---

#### TC-I008: User account deletion → Reel orphan handling
**Priority**: Medium  
**Automation**: ⏸️ Manual

**Test Steps**:
1. Delete user account
2. GET reel detail for user's reels

**Expected Result**:
- Reels still accessible
- Author info shows deleted/placeholder data

---

#### TC-I009: Multiple menu items → Price aggregation
**Priority**: Medium  
**Automation**: ✅ Automated

**Test Steps**:
1. Create menu reel with 5 menu items (prices: ₹299, ₹199, ₹99, ₹149, ₹249)
2. Verify `estimatedPaise`

**Expected Result**:
- `estimatedPaise: 99500` (29900+19900+9900+14900+24900)

---

#### TC-I010: Tagged users → Profile data resolution
**Priority**: Medium  
**Automation**: ✅ Automated

**Test Steps**:
1. Reel has 3 tagged users
2. GET user reels

**Expected Result**:
- `taggedUsers` array with 3 full user objects (username, avatar, fullName)

---

## 🔄 Regression Test Cases

### Category 9: Bug Fixes & Edge Cases

#### TC-R001: Soft-deleted reel returns 404
**Bug**: Deleted reels visible in API responses  
**Fixed**: 2026-01-10  
**Priority**: Critical  
**Automation**: ✅ Automated

**Test Steps**:
1. Delete reel (sets `deletedAt`)
2. GET `/detail/{reelId}`

**Expected Result**:
- HTTP 404 Not Found

---

#### TC-R002: Menu items ownership validation
**Bug**: Chefs could use others' menu items in menu reels  
**Fixed**: 2026-01-15  
**Priority**: Critical  
**Automation**: ✅ Automated

**Test Steps**:
1. Chef A tries to create menu reel with Chef B's items

**Expected Result**:
- HTTP 403 Forbidden

---

#### TC-R003: S3 URI exposed in API response
**Bug**: Raw S3 URIs leaked to frontend  
**Fixed**: 2026-01-12  
**Priority**: High  
**Automation**: ✅ Automated

**Test Steps**:
1. GET reel detail

**Expected Result**:
- All URLs start with `https://cdn.chefooz.com/`
- No URLs start with `s3://`

---

#### TC-R004: Reel detail with null avatar
**Bug**: Null pointer exception when user has no avatar  
**Fixed**: 2026-01-18  
**Priority**: Medium  
**Automation**: ✅ Automated

**Test Steps**:
1. User has `avatarUrl: null`
2. GET reel detail

**Expected Result**:
- `author.avatarUrl: null` (not crash)

---

#### TC-R005: Empty menuItemIds array accepted
**Bug**: Menu reel created with 0 menu items  
**Fixed**: 2026-01-20  
**Priority**: High  
**Automation**: ✅ Automated

**Test Steps**:
1. POST `/menu` with `menuItemIds: []`

**Expected Result**:
- HTTP 400 Bad Request
- Validation error

---

#### TC-R006: Duplicate reel creation prevention
**Bug**: Multiple menu reels created for same video  
**Fixed**: 2026-01-22  
**Priority**: Medium  
**Automation**: ✅ Automated

**Test Steps**:
1. Create menu reel with mediaId X
2. Try to create another menu reel with same mediaId X

**Expected Result**:
- Second request returns existing reel (idempotent)

---

#### TC-R007: Redis connection failure handling
**Bug**: Reel detail API crashes if Redis down  
**Fixed**: 2026-01-25  
**Priority**: High  
**Automation**: ⏸️ Manual

**Test Steps**:
1. Stop Redis service
2. GET reel detail

**Expected Result**:
- API returns 200 OK
- `isLiked: false`, `isSaved: false` (defaults)
- Error logged but not thrown

---

#### TC-R008: Tagged users batch query N+1 issue
**Bug**: N+1 queries when fetching tagged users  
**Fixed**: 2026-01-28  
**Priority**: Medium  
**Automation**: ✅ Automated

**Test Steps**:
1. GET user reels with 50 reels (each has 3 tagged users)

**Expected Result**:
- Only 2 DB queries (1 for reels, 1 batch for all users)
- Not 151 queries (1 + 50×3)

---

#### TC-R009: Item previews with quantity formatting
**Bug**: Item quantity not shown in itemPreviews  
**Fixed**: 2026-02-01  
**Priority**: Low  
**Automation**: ✅ Automated

**Test Steps**:
1. Order has "Dal Makhani" with qty=2

**Expected Result**:
- `itemPreviews: "Dal Makhani (2), Naan, Raita"`

---

#### TC-R010: Feature flag check timing
**Bug**: Menu reel created even with flag disabled  
**Fixed**: 2026-02-03  
**Priority**: Critical  
**Automation**: ✅ Automated

**Test Steps**:
1. Disable feature flag during request processing
2. POST `/menu`

**Expected Result**:
- Check happens before any DB writes
- Returns 403 immediately

---

#### TC-R011: Deleted reel in user profile feed
**Bug**: Deleted reels visible in profile  
**Fixed**: 2026-02-05  
**Priority**: High  
**Automation**: ✅ Automated

**Test Steps**:
1. User has 10 reels, 3 are deleted
2. GET `/user/{userId}`

**Expected Result**:
- Returns only 7 reels

---

#### TC-R012: Cache not cleared on deletion
**Bug**: Like/save sets persist after deletion  
**Fixed**: 2026-02-07  
**Priority**: High  
**Automation**: ✅ Automated

**Test Steps**:
1. Delete reel
2. Check Redis: `redis-cli KEYS "reel:{reelId}:*"`

**Expected Result**:
- No keys found (all cleared)

---

#### TC-R013: Linked order with missing items
**Bug**: Crash when order has no items  
**Fixed**: 2026-02-08  
**Priority**: Medium  
**Automation**: ✅ Automated

**Test Steps**:
1. Order has `items: []`
2. Create review reel with linkedOrderId
3. GET reel detail

**Expected Result**:
- `linkedOrder.itemPreviews: ""` (empty string, not crash)

---

#### TC-R014: Reputation tier caching issue
**Bug**: Stale reputation tier shown after update  
**Fixed**: 2026-02-10  
**Priority**: Low  
**Automation**: ⏸️ Manual

**Test Steps**:
1. GET reel detail (Bronze tier)
2. Update reputation to Gold
3. GET reel detail again

**Expected Result**:
- Shows Gold tier (no caching)

---

#### TC-R015: Pagination skip overflow
**Bug**: Incorrect results when skip > total reels  
**Fixed**: 2026-02-12  
**Priority**: Low  
**Automation**: ✅ Automated

**Test Steps**:
1. User has 50 reels
2. GET with `skip=100&limit=20`

**Expected Result**:
- Returns empty array (not error)

---

## 📊 Test Data

### Sample Reels

```json
{
  "_id": "507f1f77bcf86cd799439011",
  "userId": "uuid-user-123",
  "mediaId": "media-abc123",
  "contentType": "REEL",
  "caption": "Making authentic Dal Makhani! #cooking #indian",
  "hashtags": ["cooking", "indian"],
  "videoUrl": "s3://chefooz-output/reels/video123.m3u8",
  "thumbnailUrl": "s3://chefooz-output/reels/thumb123.jpg",
  "durationSec": 30,
  "reelPurpose": "USER_REVIEW",
  "linkedOrderId": "uuid-order-456",
  "creatorOrderValue": 79900,
  "stats": {
    "views": 5678,
    "likes": 234,
    "comments": 89,
    "saves": 45
  },
  "createdAt": "2024-01-15T10:30:00Z",
  "updatedAt": "2024-01-15T10:30:00Z"
}
```

### Sample Users

```json
{
  "id": "uuid-user-123",
  "username": "chef_kumar",
  "fullName": "Kumar Singh",
  "avatarUrl": "https://cdn.chefooz.com/avatars/user123.jpg",
  "role": "chef"
}
```

### Sample Orders

```json
{
  "id": "uuid-order-456",
  "chefId": "uuid-chef-789",
  "customerId": "uuid-user-123",
  "totalPaise": 79900,
  "items": [
    {
      "titleSnapshot": "Dal Makhani",
      "quantity": 2,
      "priceSnapshot": 29900,
      "imageSnapshot": "https://cdn.chefooz.com/items/dal-makhani.jpg"
    },
    {
      "titleSnapshot": "Naan",
      "quantity": 1,
      "priceSnapshot": 9900
    },
    {
      "titleSnapshot": "Raita",
      "quantity": 1,
      "priceSnapshot": 10100
    }
  ]
}
```

---

## 🚀 Automation Coverage

### Test Automation Breakdown

| Category | Total | Automated | Manual | Automation % |
|----------|-------|-----------|--------|--------------|
| Functional | 45 | 41 | 4 | 91% |
| Security | 12 | 11 | 1 | 92% |
| Performance | 8 | 6 | 2 | 75% |
| Integration | 10 | 8 | 2 | 80% |
| Regression | 15 | 15 | 0 | 100% |
| **Total** | **90** | **81** | **9** | **89%** |

### CI/CD Integration

**GitHub Actions Workflow** (`.github/workflows/reels-test.yml`):
```yaml
name: Reels Module Tests

on:
  push:
    paths:
      - 'apps/chefooz-apis/src/modules/reels/**'
      - 'apps/chefooz-apis/src/database/schemas/reel.schema.ts'

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      mongodb:
        image: mongo:6
        ports:
          - 27017:27017
      postgres:
        image: postgres:15
        env:
          POSTGRES_DB: chefooz_test
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        ports:
          - 5432:5432
      redis:
        image: valkey/valkey:7
        ports:
          - 6379:6379

    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'

      - name: Install dependencies
        run: npm ci

      - name: Run unit tests
        run: npm run test:reels

      - name: Run E2E tests
        run: npm run test:e2e:reels

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/reels/lcov.info
```

---

## 📈 Test Metrics

### Coverage Goals

- **Code Coverage**: ≥85%
- **Branch Coverage**: ≥80%
- **Functional Test Coverage**: ≥90%
- **Critical Path Coverage**: 100%

### Test Execution Time

- **Unit Tests**: ~30 seconds
- **Integration Tests**: ~2 minutes
- **E2E Tests**: ~5 minutes
- **Performance Tests**: ~10 minutes
- **Total CI/CD Pipeline**: ~20 minutes

---

## 🐛 Bug Reporting Template

```markdown
### Bug Report: Reels Module

**Module**: Reels
**Endpoint**: POST /api/v1/reels/menu
**Severity**: High
**Priority**: P1

**Description**:
Menu reel creation fails when menuItemIds array contains more than 10 items.

**Steps to Reproduce**:
1. Authenticate as chef
2. POST to /menu with 15 menuItemIds
3. Observe 500 Internal Server Error

**Expected Behavior**:
Menu reel created successfully with all 15 items linked.

**Actual Behavior**:
500 error with message: "Failed to create menu reel"

**Environment**:
- Backend: NestJS 10.2.0
- MongoDB: 6.0.5
- Node: 18.16.0

**Logs**:
```
[ReelsService] Error: Menu item query timeout
at MenuItemRepository.find (menu-item.repository.ts:45)
```

**Impact**:
Chefs cannot create combo deals with >10 items.

**Proposed Fix**:
Increase menu item query batch size or implement pagination.
```

---

## 📚 Related Documentation

- **Feature Overview**: `FEATURE_OVERVIEW.md`
- **Technical Guide**: `TECHNICAL_GUIDE.md`
- **Reel Schema**: `apps/chefooz-apis/src/database/schemas/reel.schema.ts`
- **API Client**: `libs/api-client/src/lib/clients/reels.client.ts`

---

**Document Version**: 1.0  
**Last Updated**: 2026-02-14  
**Next Review**: 2026-03-14
