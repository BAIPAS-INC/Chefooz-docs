# Reels Module - QA Test Cases

**Module**: `apps/chefooz-apis/src/modules/reels`  
**Test Coverage**: Functional, Security, Performance, Integration  
**Version**: 1.0  
**Last Updated**: 2026-03-XX

---

## 📋 Test Overview

### Test Strategy

| Test Type | Count | Automation | Priority |
|-----------|-------|------------|----------|
| **Functional** | 45 | 90% | High |
| **Security** | 12 | 100% | Critical |
| **Performance** | 8 | 75% | Medium |
| **Integration** | 10 | 80% | High |
| **Regression** | 16 | 94% | High |
| **Total** | 91 | 88% | - |

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

### Category 0: Upload V2 UI Gating (Temporary)

#### TC-UPL-UI-001: Text action hidden in edit screen right rail
**Priority**: High  
**Automation**: ❌ Manual

**Pre-conditions**:
- App launched to mobile upload flow
- Navigate to `/reels/upload-v2/edit`

**Test Steps**:
1. Open edit screen with selected media
2. Inspect right action rail items

**Expected Result**:
- Right action rail does **not** show `Text` action
- No tap target exists to open `TextEditModal` for creating a new overlay

---

#### TC-UPL-UI-002: Filter action hidden in edit screen right rail
**Priority**: High  
**Automation**: ❌ Manual

**Pre-conditions**:
- App launched to mobile upload flow
- Navigate to `/reels/upload-v2/edit` with video media

**Test Steps**:
1. Open edit screen with selected video
2. Inspect right action rail items

**Expected Result**:
- Right action rail does **not** show `Filter` action
- No direct rail tap target opens `FilterPickerSheet`

---

#### TC-UPL-UI-003: Filter chip hidden when filter state exists
**Priority**: Medium  
**Automation**: ❌ Manual

**Pre-conditions**:
- Upload store has existing `filter` value (from saved draft/state)
- Edit screen opened with media

**Test Steps**:
1. Open `/reels/upload-v2/edit`
2. Inspect lower-left chip area and preview overlays

**Expected Result**:
- Filter chip is not displayed
- UI does not expose edit entry point for filter while gate is active

---

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

#### TC-R016: Profile reel viewer uses full viewport when tab bar is hidden
**Bug**: Next reel peeked into view when opening profile reels (tab bar hidden) because reel height subtracted tab bar size.  
**Fixed**: 2026-02-27  
**Priority**: Medium  
**Automation**: ⏸️ Manual

**Test Steps**:
1. Open any profile, tap a reel to launch the profile reel viewer (tab bar hidden).
2. Confirm viewport height matches full device window (no tab bar overlay).
3. Scroll to next reel and release.

**Expected Result**:
- Only one reel is fully visible at a time (no partial peek of the next reel).
- Snap pagination aligns with reel boundaries on both scroll and idle states.

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

## 🐛 Bug Regression Test Cases (March 2026 QA Round)

### TC-REELS-BUG-001: Like Icon Not Filled After Liking a Reel (Heart Stays Outlined)

**Type:** Bug Regression
**Feature area:** Reel - Engagement / Like
**Priority:** P1

**Preconditions:**
- User is logged in
- At least one reel visible in the feed

**Steps:**
1. Open the feed
2. Tap the heart icon on any reel
3. Observe the heart icon immediately after tapping
4. Scroll away from the reel and then scroll back

**Expected result:** Heart icon turns filled (red/solid) and stays filled when scrolling back
**Actual result (before fix):**
- Optimistic update incremented like count but did NOT set `isLiked: true` in the React Query cache
- `React.memo` equality check only compared `item.id` and `isVisible`, blocking re-renders when cache updated
- `useState(item.stats?.isLiked || false)` never re-ran after initial render, so icon reverted to outlined on any re-mount
**Fix applied (3 locations):**
1. `libs/api-client/src/lib/hooks/useEngagement.ts` — optimistic update now toggles `isLiked`/`isSaved` boolean and correctly adjusts count by ±1 (not always +1)
2. `apps/chefooz-app/src/components/ReelCard.tsx` — added `useEffect` to sync `isLiked` state from `item.stats?.isLiked` on prop changes
3. `apps/chefooz-app/src/components/ReelCard.tsx` — `React.memo` now also compares `item.stats?.isLiked` to allow re-renders on engagement cache changes
**Regression test:**
1. Tap like on a reel
2. Scroll to next reel and back
3. Confirm heart is still filled
4. Tap unlike — confirm count decrements correctly
**Status:** Fixed ✅

---

### TC-REELS-BUG-002: Video Plays During Text Overlay Editing (Upload Edit Screen)

**Type:** Bug Regression
**Feature area:** Upload Edit - Text Overlay
**Priority:** P2

**Preconditions:**
- User has selected or recorded a video in the upload flow
- Video is playing in preview on the edit screen

**Steps:**
1. Open the upload edit screen with a video
2. Tap the "Text" button on the right action rail
3. Observe the video in the background while `TextEditModal` is open

**Expected result:** Video pauses when the text edit modal opens; resumes when it closes
**Actual result (before fix):** Video continued playing in the background while the user was editing text, causing distraction and potential audio interference
**Fix applied:** Added `useEffect` in `apps/chefooz-app/src/app/reels/upload-v2/edit.tsx` to call `player.pause()` when `showTextEditModal` becomes `true` and `player.play()` when it becomes `false`
**Regression test:**
1. Select a video on the upload edit screen
2. Tap the "Text" button
3. Confirm video is silent/paused
4. Dismiss the text modal
5. Confirm video resumes playing
**Status:** Fixed ✅

---

**Document Version**: 1.0  
**Last Updated**: 2026-03-03  
**Next Review**: 2026-03-14

---

### TC-REELS-BUG-003: Like Count Shows 0 After Liking — Heart Not Preserved Across Reel Views

**Type:** Bug Regression  
**Feature area:** Feed / Reel engagement — Like action  
**Priority:** P0

**Preconditions:**
- User is authenticated
- Feed contains at least one reel the user has not yet liked

**Steps:**
1. Open the feed, scroll to a reel
2. Tap the heart icon — count should increment and heart should fill red
3. Scroll past the reel, then scroll back
4. Observe the heart icon and count

**Expected result:** Heart is filled red; like count persists  
**Actual result (before fix):** Heart shows outline; count shows 0

**Root causes (multi-layer):**
1. `handleLikeEngagement` used `$setOnInsert + { active: true }` upsert — second tap was idempotent (never toggled), so UI and server diverged
2. Backend returned stats without `isLiked` in the response, so `mergeEngagementStats` could not set correct state
3. `mapReelToFeedItem` received `userId=undefined` (controller never extracted it), so `isLiked` was always `false` on feed refetch
4. `chefooz.tsx` never tracked `isTabFocused`, so the viewability seed never ran — `isVisible` stayed `false` — `ReelCard` useEffect never synced

**Fix applied:**
- `feed.service.ts`: `handleLikeEngagement` now reads existing record and toggles `active` flag; includes `isLiked` in response
- `feed.service.ts`: `getFeed` accepts and passes `userId` to `mapReelToFeedItem`
- `feed.controller.ts`: `@UseGuards(OptionalJwtAuthGuard)` added to GET /feed; `userId` extracted from `req.user`
- `useEngagement.ts`: `mergeEngagementStats` uses server-returned `isLiked`/`isSaved` when present; falls back to toggle
- `ReelCard.tsx`: local `localLikesCount` state for instant feedback; syncs from underlying cache item

**Status:** Fixed ✅

---

## Dark Mode Regression Tests (Added March 2026)

### TC-REELS-DM-001: Share Sheet in Dark Mode

**Type:** Bug Regression / Manual  
**Feature area:** Share Sheet (`components/share/ShareSheet.tsx`)  
**Priority:** P1

**Preconditions:**
- Device set to dark appearance
- User is on the home feed or reel detail screen

**Steps:**
1. Open app in dark mode
2. Tap the share icon on any reel
3. Observe the bottom sheet background, title, option labels, cancel button

**Expected result:** Sheet background `colors.surface` (`#1C1C1E`), handle `colors.textMuted`, text `colors.textPrimary` / `colors.textSecondary`, cancel button `colors.interactiveSubtle`  
**Actual result (before fix):** White sheet on dark background — stark white bottom sheet  
**Fix applied:** `ShareSheet.tsx` converted to `makeStyles(colors)` factory  
**Regression test:** `apps/chefooz-app/src/components/share/ShareSheet.tsx`  
**Status:** Fixed ✅

### TC-REELS-DM-002: Report Sheet in Dark Mode

**Type:** Bug Regression / Manual  
**Feature area:** Report Sheet (`components/report/ReportSheet.tsx`)  
**Priority:** P1

**Steps:**
1. Open app in dark mode
2. Tap the 3-dot menu on a reel and select "Report"
3. Observe the bottom sheet background, reason list, text input, guidelines box

**Expected result:** Sheet background `colors.surface`, reasons list uses dark text `colors.textPrimary`, textarea uses `colors.surfaceElevated` with `colors.border`  
**Actual result (before fix):** White sheet regardless of theme  
**Fix applied:** `ReportSheet.tsx` converted to `makeStyles(colors, isDark)` factory  
**Regression test:** `apps/chefooz-app/src/components/report/ReportSheet.tsx`  
**Status:** Fixed ✅

---

### TC-REELS-OVERLAY-001: Text overlay position mismatch between edit and feed

**Type:** Bug Regression / Manual  
**Feature area:** Upload Edit screen (`reels/upload-v2/edit.tsx`), ReelCard (`components/ReelCard.tsx`)  
**Priority:** P0

**Preconditions:**
- User is logged in
- A photo or video has been selected/recorded

**Steps:**
1. Navigate to Upload → select a photo (portrait or any aspect ratio)
2. Add a text overlay, position it at a distinctive location (e.g., center, top-third)
3. Proceed to share and publish the reel
4. View the published reel in the feed
5. Observe position of the text overlay

**Expected result:** Text overlay appears at the same relative position as placed in the edit screen.  
**Actual result (before fix):** Text overlay appeared shifted downward — most noticeably toward the bottom of the frame. Shift was proportional to `(VIEWPORT_HEIGHT − SCREEN_WIDTH×16/9) / 2` (≈ 17–47px depending on device).  
**Root cause:** The edit screen (AspectRatioPreview with `forceNineSixteen=true`) normalized overlay positions against a `SCREEN_WIDTH × SCREEN_WIDTH*(16/9)` canvas.  ReelCard was rendering overlays against `SCREEN_WIDTH × VIEWPORT_HEIGHT` (a taller canvas), shifting all Y positions.  
**Fix applied:** In `ReelCard.tsx`, the OverlayCanvas now uses `canvasHeight = SCREEN_WIDTH * (16/9)` and is wrapped in a `View` with `top = (VIEWPORT_HEIGHT − NINE_SIXTEEN_CANVAS_HEIGHT) / 2` to align with the vertically-centered video content.  
**Regression test:** Manual — add text at known normalized position, verify in feed  
**Status:** Fixed ✅

---

### TC-REELS-CONTENTTYPE-001: Reel/Post content-type selection not accessible when drafts exist

**Type:** Bug Regression / Manual  
**Feature area:** Upload Select screen (`reels/upload-v2/select.tsx`), Upload Edit screen  
**Priority:** P1

**Preconditions:**
- User has at least one saved draft

**Steps:**
1. Navigate to Upload
2. Observe that the select screen shows draft cards, not the Reel/Post selection UI
3. Tap "Start New Upload" — navigate directly to edit screen with no way to choose content type

**Expected result:** User can always select Reel vs Post content type regardless of draft state.  
**Actual result (before fix):** The Reel/Post selection cards were only shown when no drafts existed. When drafts were present, the content-type section was hidden entirely.  
**Fix applied:** Added an inline Reel/Post toggle pill to the top-center of the edit screen (`edit.tsx`). The toggle shows whenever media is selected and writes to the upload store via `setContentType`. The flash toggle is shown in the same slot when camera is active (no media).  
**Regression test:** Manual — start upload with existing draft, verify toggle renders on edit screen  
**Status:** Fixed ✅

---

### TC-REELS-DRAFT-DELETE-NAV-001: Deleting last draft navigates to deprecated select screen

**Type:** Bug Regression / Manual
**Feature area:** Upload Select screen (`reels/upload-v2/select.tsx`)
**Priority:** P1

**Preconditions:**
- User has one or more saved upload drafts

**Steps:**
1. Open the Upload flow — draft cards are shown
2. Tap the delete (✕) button on a draft card and confirm deletion
3. If this was the last draft, observe where the user is taken

**Expected result:** User is navigated to the Edit screen (`/reels/upload-v2/edit`) so they can start a fresh upload with the camera-first flow.
**Actual result (before fix):** After the last draft was deleted, the screen stayed on `select.tsx` and rendered the deprecated Reel/Post content-type selection cards, which no longer match the actual upload flow.
**Fix applied:** In `select.tsx → handleDeleteDraft` and `handleContinueDraft` (media not found case), after deleting the last draft `router.replace('/reels/upload-v2/edit')` is called.
**Regression test:** Manual — delete last draft, verify user lands on edit (camera) screen
**Status:** Fixed ✅

---

### TC-REELS-SHARE-WATCH-UX-001: Share page still editable after user opts to "Watch progress"

**Type:** Bug Regression / UX
**Feature area:** Share screen (`reels/upload-v2/share.tsx`), `UploadS3SuccessSheet`
**Priority:** P1

**Preconditions:**
- User has uploaded a reel (S3 upload completed)
- The `UploadS3SuccessSheet` is visible with "Watch progress" and "Go to feed" options

**Steps:**
1. Complete reel upload
2. When the success sheet appears, tap "Watch progress"
3. Observe the share screen state

**Expected result:** The share form (caption, metadata, publish button) is replaced by a locked processing status view showing 3 steps: Uploaded → Transcoding → Going live. The user can see live progress. A "Go to Feed" button is available.
**Actual result (before fix):** The success sheet closed but the share form remained fully editable, leaving the user on an interactive form with no processing status. "Watch progress" didn't communicate what the user was watching or provide any status feedback.
**Fix applied:**
- Added `isWatching` state to `share.tsx`
- `handleStayAndWatch` now sets `isWatching = true`
- When `isWatching` is true, the ScrollView is replaced by a `watchingView` with: a cloud-check icon, "Reel Uploaded!" title, subtitle, 3-step progress tracker driven by `progressPhase` from `useUploadProgressStore`, and a hint about the notification banner
- Footer shows just "Go to Feed" / "View Your Reel" button when watching
- `UploadS3SuccessSheet` "Stay & watch" button renamed to "Watch progress" with updated body copy
- Media guard updated to `if (!media && !isWatching)` so the watching view stays visible during the brief window between `resetUpload()` and navigation
**Regression test:** Manual — publish reel, tap "Watch progress", verify form is replaced by status view
**Status:** Fixed ✅

---

### TC-REELS-BUG-PROFILE-GRID-REVIEW: Profile grid shows "Under Review" badge correctly after AI moderation

**Type:** Bug Regression  
**Feature area:** `apps/chefooz-app/src/app/profile/_components/ProfileGrid.tsx`  
**Priority:** P1

**Preconditions:**
- User has uploaded a reel that was flagged by AI moderation (AWS Rekognition)

**Steps:**
1. Upload a reel
2. Wait for processing and AI moderation to complete
3. Navigate to profile → grid view

**Expected result:** Reel thumbnail shows ⏳ "Under Review" overlay  
**Actual result (before fix):** Badge was NOT shown — reel appeared as a normal thumbnail with no status indicator, giving the user no feedback that it was pending manual moderation  
**Root cause:** `isPending` check used the legacy value `'reviewing'` (`moderationStatus === 'pending' || moderationStatus === 'reviewing'`). The backend Reel schema enum uses `'needs_review'` (not `'reviewing'`), so reels in AI-flagged state were invisible to the badge renderer.  
**Fix applied:** Changed `'reviewing'` → `'needs_review'` in `ProfileGrid.tsx` line 41.  
**Regression test:** Manual — upload a reel that triggers AI flag, check profile grid for ⏳ overlay  
**Status:** Fixed ✅

---

### TC-REELS-BUG-UPLOAD-QUOTA-MESSAGE: User-friendly message shown when upload quota exceeded

**Type:** Bug Regression  
**Feature area:** `apps/chefooz-app/src/app/reels/upload-v2/share.tsx` — upload catch block  
**Priority:** P1

**Preconditions:**
- User has used all review reels for their completed orders OR reached weekly promo limit OR uploaded showcase for an item

**Steps:**
1. Try to upload a reel when quota is full
2. Observe the error alert

**Expected result:**
- Review quota: "You've reached your review reel limit for completed orders. Complete more orders to unlock additional review reels."
- Menu showcase quota: "You've already uploaded a showcase reel for this menu item. Only 1 showcase reel is allowed per menu item."
- Weekly promo quota: "You've reached your weekly upload limit. Your quota resets every Monday - come back then!"

**Actual result (before fix):** Generic `Alert.alert('Upload Error', 'Failed to upload reel. Please try again.')` — no guidance on what to do next  
**Root cause:** The `catch` block in `handlePublish` did not inspect `error.response.data.errorCode`. The backend returns `POLICY_UPLOAD_QUOTA_EXCEEDED` with a `data.type` field (`review` | `menu_showcase` | `promotional`) but the frontend ignored it.  
**Fix applied:** Added `POLICY_UPLOAD_QUOTA_EXCEEDED` branch in catch, with type-specific messages keyed on `errData.data.type`.  
**Status:** Fixed ✅

---

### TC-REELS-BUG-DETAIL-OVERLAY-MISSING: Order/menu overlay not shown when opening reel via deep-link or explore tap

**Type:** Bug Regression
**Feature area:** `[reelId].tsx` tab reel viewer, `ReelCard.tsx` overlay logic
**Priority:** P1

**Preconditions:**
- At least one `MENU_SHOWCASE` reel with `linkedMenu` exists
- At least one `USER_REVIEW` reel with `linkedOrder` exists
- User taps a reel from "Cooking Right Now" section or any direct-link path

**Steps:**
1. From the Explore tab, tap a reel card in "Cooking Right Now"
2. The reel opens in the tab `[reelId].tsx` full-screen viewer
3. Wait for reel to load

**Expected result:**
- `MENU_SHOWCASE` reels: "Add to cart" / "View menu" button visible at bottom-left of the reel
- `USER_REVIEW` reels: "Order again ₹X" button visible at bottom-left of the reel

**Actual result (before fix):** No overlay rendered at all; `reelPurpose` and `linkedMenu` were absent in the `GET /v1/reels/detail/:id` response.

**Root cause:** `mapToDetailResponse()` in `reels.service.ts` returned `linkedOrder` correctly but omitted `reelPurpose` and `linkedMenu`. Since `ReelCard` checks `(item as any).reelPurpose === 'MENU_SHOWCASE'` first, an absent `reelPurpose` (undefined) caused the MENU_SHOWCASE overlay condition to be falsy. For USER_REVIEW reels, `reelPurpose` being undefined means the condition `!== 'MENU_SHOWCASE'` is true, but the overlay still depends on `linkedOrder` which requires correct data flow.

**Fix applied:** Added `reelPurpose: reel.reelPurpose` and `linkedMenu: reel.linkedMenu || null` to the return object of `mapToDetailResponse()`. Added corresponding fields to `ReelDetailResponseDto`. Also added `linkedMenu` availability check (inactive/sold-out items suppress overlay), `linkedOrder` availability check (all-items-unavailable suppresses overlay), and `orderContext` (chef isOpen + identity) to `getReelDetail` — bringing it to parity with feed enrichment. Imported `ChefKitchenModule` into `ReelsModule` to enable `chefKitchenService`/`chefScheduleService`.

**Files changed:**
- `apps/chefooz-apis/src/modules/reels/reels.service.ts`
- `apps/chefooz-apis/src/modules/reels/dto/get-reel-detail.dto.ts`

**Regression test:** Verify `getReelDetail` API response body includes `reelPurpose` and `linkedMenu` fields.
**Status:** Fixed ✅

---

### TC-REELS-BUG-005: Text Overlay Vertical Position Shifted for 4:5 and 1:1 REEL Images

**Type:** Bug Regression  
**Feature area:** Upload — Edit screen text overlays / Feed playback  
**Priority:** P1  
**Last Updated:** 2026-03-25

**Preconditions:**
- User selects a 4:5 (portrait, non-9:16) image from gallery for a REEL
- User adds one or more text overlays (e.g. centre, top, bottom of visible image)
- User saves and posts the reel

**Steps:**
1. Open the upload flow, select a 4:5 portrait image from gallery as a REEL
2. On the edit screen, add text overlays — one near the top of the image, one at centre, one near the bottom
3. Publish the reel
4. Open the feed and observe the reel with overlays in playback

**Expected result:** Text overlays appear at the same relative position on the image in playback as they appeared in the edit preview.

**Actual result (before fix):** Overlays appeared significantly lower in playback. For a 4:5 image the centre overlay shifted by ~14% downward; a text placed near the top of the image in edit appeared roughly 18% from the top in playback. After the first partial fix (forced 9:16 dimensions), text still shifted upward by ~16px due to the canvas not being at the same screen-y position as the feed.

**Root cause (final):**
The edit screen had `paddingTop: insets.top` on the outer container, pushing the REEL preview below the status bar. The `AspectRatioPreview` container height was `SCREEN_WIDTH*(16/9)` ≈ 693px. In the feed, `ReelCard` renders the image fullscreen from `y=0` (behind transparent status bar) at `VIEWPORT_HEIGHT` ≈ 756px, with `OverlayCanvas` offset by `(VIEWPORT_HEIGHT - 693) / 2 ≈ 34px`. Combined with the 47px status-bar padding, the canvas start in edit was ~81px below the feed canvas position → direct vertical mismatch in perceived text position.

**Fix applied:**
Full WYSIWYG approach in `apps/chefooz-app/src/app/reels/upload-v2/edit.tsx`:
1. `paddingTop/Bottom` removed from the REEL container (POST keeps safe-area padding)
2. `AspectRatioPreview` receives `videoWidth=windowWidth, videoHeight=REEL_VIEWPORT_HEIGHT` with `forceNineSixteen=false` — container fills exactly `SCREEN_WIDTH × VIEWPORT_HEIGHT` starting at y=0
3. `VideoView` changed to `contentFit="cover"` to match feed playback
4. `VideoFilterPreview.tsx` changed to `contentFit="cover"` for the same reason

`apps/chefooz-app/src/components/ReelCard.tsx`:
5. `NINE_SIXTEEN_CANVAS_HEIGHT` and `overlayCanvasTopOffset` removed
6. `OverlayCanvas` now uses `canvasHeight=VIEWPORT_HEIGHT, top=0` — identical coordinate space to edit

**Regression test:** Manual — place text at top/centre/bottom of a 4:5 image REEL in edit; positions must be pixel-accurate in feed playback.  
**Status:** Fixed ✅

---

## 🐛 Bug Regression Test Cases (March 2026 QA Round — Verified Purchase Badge)

### TC-REELS-BUG-COMMENT-001: "Verified Purchase" Badge Appears in Reel Comments

**Type:** Feature / Regression Guard
**Feature area:** Reels — Comment Sheet (feed & home-feed)
**Priority:** P1

**Preconditions:**
- A reel is linked to at least one menu item via `linkedMenu.menuItemIds`
- User A has a delivered order (`status = 'delivered'`) that contains that menu item
- User B has NOT ordered the same dish

**Steps:**
1. Open the reel in the feed or home-feed
2. Open the comment sheet
3. Scroll to a comment by User A (who has a delivered order for the same dish)
4. Observe the area next to User A's username
5. Also observe a comment by User B (no order)

**Expected result:**
- User A's comment displays a green "Verified Purchase" pill badge (`✓ Verified Purchase`) next to the username
- User B's comment shows no badge

**Actual result (before feature):** No badge was shown for any commenter regardless of purchase history

**Root cause (final):**
For `USER_REVIEW` reels, `reel.userId` is the **reviewer/customer** who posted the review, NOT the chef. The original query used `reel.userId` as the chef ID for all reel types, so it compared commenter orders against the wrong user (the reviewer), yielding zero matches. Chef must be resolved via `reel.linkedOrderId → order.chefId`.

**Fix applied:**
`comments.service.ts` — before the order batch query, check `reel.reelPurpose`:
- `USER_REVIEW`: resolve chef via `reel.linkedOrderId → orders.chefId` (one extra Postgres lookup)
- `MENU_SHOWCASE` / `PROMOTIONAL`: use `reel.userId` directly (the chef who posted)

**Regression test:**
1. Create a test reel with `linkedMenu.menuItemIds = ['item-uuid-1']`
2. Place a delivered order as User A that includes `item-uuid-1`
3. User A posts a comment on that reel
4. User B (no order) posts a comment
5. Fetch comments via `GET /api/v1/comments/reel/:reelId` — confirm User A's comment has `hasPurchasedDish: true`, User B has `false` or `undefined`
6. In the app, open the comment sheet for that reel — confirm green badge appears only for User A
7. For a reel with NO `linkedMenu` — confirm no badge for any commenter and no errors

**Status:** Fixed ✅

---

### TC-REELS-BUG-COMMENT-002: Verified Purchase Badge — Reel with No Linked Menu

**Type:** Edge Case / Regression Guard
**Feature area:** Reels — Comment Sheet
**Priority:** P2

**Preconditions:**
- A reel has `linkedMenu = null` or `linkedMenu.menuItemIds = []`
- Any users have posted comments

**Steps:**
1. Open the reel's comment sheet

**Expected result:** Comments load normally; no "Verified Purchase" badge appears; no errors
**Actual result (before fix):** N/A (this is the defensive path test)
**Fix applied:** `listComments` wraps the purchase check in `try/catch`; if no linked menu items are found or the query fails, `hasPurchasedDish` defaults to `false` for all comments
**Status:** Fixed ✅

---

