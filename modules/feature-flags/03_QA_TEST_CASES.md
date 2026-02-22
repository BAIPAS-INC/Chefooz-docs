# Feature Flags Module - QA Test Cases

**Module**: `feature-flags`  
**Type**: Configuration & Feature Management  
**Last Updated**: February 23, 2026

---

## 📋 Table of Contents

1. [Test Environment Setup](#test-environment-setup)
2. [Flag CRUD Operations Tests](#flag-crud-operations-tests)
3. [Gradual Rollout Tests](#gradual-rollout-tests)
4. [Caching Tests](#caching-tests)
5. [Admin Authorization Tests](#admin-authorization-tests)
6. [Integration Tests](#integration-tests)
7. [Performance Tests](#performance-tests)

---

## 🛠️ Test Environment Setup

### Prerequisites

```bash
# 1. Install dependencies
npm install

# 2. Set up test database
createdb chefooz_test

# 3. Run migrations
npm run migration:run -- --config=ormconfig.test.ts

# 4. Set up Redis (test instance)
redis-server --port 6380 --daemonize yes

# 5. Set environment variables
export NODE_ENV=test
export DATABASE_URL=postgresql://user:pass@localhost:5432/chefooz_test
export REDIS_HOST=localhost
export REDIS_PORT=6380
export JWT_SECRET=test_secret_123
```

### Seed Test Data

```typescript
// Seed default flags
await featureFlagService.initializeDefaults();

// Create test admin user
const admin = await userRepo.create({
  email: 'admin@chefooz.com',
  role: 'admin',
  password: await bcrypt.hash('password123', 10),
});
await userRepo.save(admin);

// Get admin JWT token
const adminToken = jwt.sign({ userId: admin.id, role: 'admin' }, JWT_SECRET);
```

---

## 🚩 Flag CRUD Operations Tests

### TC-FLAG-001: List All Flags (Public)

**Objective**: Verify anyone can list all flags (read-only)

**Preconditions**:
- 9 default flags seeded in database

**Test Steps**:
1. Send GET `/api/v1/feature-flags` without auth

**Expected Result**:
- Status: `200 OK`
- Response contains 9 flags:
  ```json
  {
    "success": true,
    "message": "Feature flags retrieved",
    "data": {
      "flags": [
        {
          "key": "ENABLE_MENU_REELS",
          "enabled": false,
          "description": "Enable reels on chef menu pages",
          "rolloutPercent": 100,
          ...
        },
        ...
      ],
      "count": 9
    }
  }
  ```

**Pass Criteria**: ✅
- Status = 200
- 9 flags returned
- No auth required

---

### TC-FLAG-002: Create New Flag (Admin Only)

**Objective**: Verify only admins can create flags

**Preconditions**:
- Admin authenticated with JWT token

**Test Steps**:
1. Send POST `/api/v1/feature-flags` with admin token:
   ```json
   {
     "key": "ENABLE_TEST_FEATURE",
     "enabled": true,
     "description": "Test feature",
     "rolloutPercent": 100
   }
   ```

**Expected Result**:
- Status: `201 Created`
- Response contains new flag:
  ```json
  {
    "success": true,
    "message": "Feature flag ENABLE_TEST_FEATURE set to true",
    "data": {
      "key": "ENABLE_TEST_FEATURE",
      "enabled": true,
      "rolloutPercent": 100,
      ...
    }
  }
  ```
- Flag saved in database
- Cache invalidated

**Pass Criteria**: ✅
- Status = 201
- Flag created in DB
- Cache key deleted

---

### TC-FLAG-003: Update Existing Flag

**Objective**: Verify admin can update flag properties

**Preconditions**:
- Flag `ENABLE_MENU_REELS` exists (enabled=false)

**Test Steps**:
1. Send PATCH `/api/v1/feature-flags/ENABLE_MENU_REELS` with admin token:
   ```json
   {
     "enabled": true,
     "rolloutPercent": 50
   }
   ```

**Expected Result**:
- Status: `200 OK`
- Response contains updated flag:
  ```json
  {
    "success": true,
    "message": "Feature flag ENABLE_MENU_REELS updated",
    "data": {
      "key": "ENABLE_MENU_REELS",
      "enabled": true,
      "rolloutPercent": 50,
      ...
    }
  }
  ```
- Database updated
- Cache invalidated

**Pass Criteria**: ✅
- Status = 200
- `enabled` = true
- `rolloutPercent` = 50

---

### TC-FLAG-004: Toggle Flag On/Off

**Objective**: Verify admin can toggle flag with one click

**Preconditions**:
- Flag `ENABLE_MENU_REELS` exists (enabled=false)

**Test Steps**:
1. Send PATCH `/api/v1/feature-flags/ENABLE_MENU_REELS/toggle` with admin token

**Expected Result**:
- Status: `200 OK`
- Response shows flag toggled:
  ```json
  {
    "success": true,
    "message": "Feature flag ENABLE_MENU_REELS toggled to true",
    "data": {
      "key": "ENABLE_MENU_REELS",
      "enabled": true,
      ...
    }
  }
  ```
- Database updated: `enabled=true`
- Cache invalidated

**Pass Criteria**: ✅
- Status = 200
- `enabled` flipped from false to true

---

### TC-FLAG-005: Delete Flag

**Objective**: Verify admin can delete flags

**Preconditions**:
- Flag `ENABLE_TEST_FEATURE` exists

**Test Steps**:
1. Send DELETE `/api/v1/feature-flags/ENABLE_TEST_FEATURE` with admin token

**Expected Result**:
- Status: `200 OK`
- Response:
  ```json
  {
    "success": true,
    "message": "Feature flag deleted"
  }
  ```
- Flag removed from database
- Cache invalidated

**Pass Criteria**: ✅
- Status = 200
- Flag deleted from DB
- Cache key deleted

---

### TC-FLAG-006: Non-Admin Cannot Create Flag

**Objective**: Verify write operations blocked for non-admins

**Preconditions**:
- Regular user authenticated (role=customer)

**Test Steps**:
1. Send POST `/api/v1/feature-flags` with customer token

**Expected Result**:
- Status: `403 Forbidden`
- Error:
  ```json
  {
    "success": false,
    "message": "Forbidden - Admin role required",
    "errorCode": "FORBIDDEN"
  }
  ```

**Pass Criteria**: ✅
- Status = 403
- Flag not created

---

## 🎯 Gradual Rollout Tests

### TC-ROLLOUT-001: 50% Rollout (Consistent Bucketing)

**Objective**: Verify users consistently assigned to same bucket

**Preconditions**:
- Flag `ENABLE_TEST_FEATURE` (enabled=true, rolloutPercent=50)

**Test Steps**:
1. Check flag for `user-123`:
   - GET `/api/v1/feature-flags/check/ENABLE_TEST_FEATURE?userId=user-123`
2. Repeat 10 times
3. Check flag for `user-456`:
   - GET `/api/v1/feature-flags/check/ENABLE_TEST_FEATURE?userId=user-456`
4. Repeat 10 times

**Expected Result**:
- `user-123` result: Same on all 10 checks (either true or false, consistent)
- `user-456` result: Same on all 10 checks (either true or false, consistent)
- ~50% of users enabled (check 100 different user IDs → ~50 true, ~50 false)

**Pass Criteria**: ✅
- Same user ID → same result (100% consistency)
- ~50% distribution across 100 users

---

### TC-ROLLOUT-002: Gradual Rollout Progression (5% → 100%)

**Objective**: Verify users stay enabled as rollout increases

**Preconditions**:
- Flag `ENABLE_TEST_FEATURE` (enabled=true, rolloutPercent=5)
- 100 test users (`user-001` to `user-100`)

**Test Steps**:
1. Day 1 (5% rollout):
   - Check flag for all 100 users
   - Record which users enabled (~5 users)
2. Day 2 (10% rollout):
   - Update rolloutPercent to 10
   - Check flag for all 100 users
   - Verify users from Day 1 still enabled + new users (~10 total)
3. Day 3 (25% rollout):
   - Update rolloutPercent to 25
   - Check flag for all 100 users
   - Verify all users from Day 2 still enabled + new users (~25 total)
4. Day 7 (100% rollout):
   - Update rolloutPercent to 100
   - Check flag for all 100 users
   - All users enabled

**Expected Result**:
- Day 1: ~5 users enabled (buckets 0-4)
- Day 2: ~10 users enabled (buckets 0-9, includes Day 1 users)
- Day 3: ~25 users enabled (buckets 0-24, includes Day 2 users)
- Day 7: 100 users enabled (all buckets)
- **No user flips from enabled to disabled** (consistency)

**Pass Criteria**: ✅
- Users stay enabled as rollout increases
- No user flips from enabled to disabled
- Final 100% rollout → all users enabled

---

### TC-ROLLOUT-003: User ID Hashing Distribution

**Objective**: Verify hash function distributes evenly

**Test Steps**:
1. Generate 10,000 unique user IDs (`user-0` to `user-9999`)
2. Hash each user ID to bucket (0-99)
3. Count users per bucket
4. Verify distribution

**Expected Result**:
- Each bucket has ~100 users (10,000 / 100 = 100)
- Allow 20% variance (80-120 users per bucket)
- No bucket empty
- No bucket with >200 users

**Pass Criteria**: ✅
- 80 ≤ users per bucket ≤ 120
- Even distribution (no clusters)

---

## 💾 Caching Tests

### TC-CACHE-001: Cache Hit (Flag in Redis)

**Objective**: Verify flag served from cache

**Preconditions**:
- Flag `ENABLE_MENU_REELS` cached in Redis

**Test Steps**:
1. GET `/api/v1/feature-flags/check/ENABLE_MENU_REELS`
2. Verify Redis GET called
3. Verify PostgreSQL SELECT not called

**Expected Result**:
- Redis GET called: `feature_flag:ENABLE_MENU_REELS`
- PostgreSQL not queried
- Response latency < 20ms

**Pass Criteria**: ✅
- Cache hit (Redis)
- No DB query
- Fast response

---

### TC-CACHE-002: Cache Miss (Flag Not in Redis)

**Objective**: Verify flag fetched from DB on cache miss

**Preconditions**:
- Flag `ENABLE_MENU_REELS` not in Redis (cache cleared)

**Test Steps**:
1. GET `/api/v1/feature-flags/check/ENABLE_MENU_REELS`
2. Verify Redis GET called (returns null)
3. Verify PostgreSQL SELECT called
4. Verify Redis SETEX called (store in cache)

**Expected Result**:
- Redis GET called (miss)
- PostgreSQL queried
- Redis SETEX called with TTL 300
- Response latency < 100ms

**Pass Criteria**: ✅
- Cache miss handled gracefully
- Flag cached for next request
- TTL = 300 seconds

---

### TC-CACHE-003: Cache Invalidation on Write

**Objective**: Verify cache cleared when flag updated

**Preconditions**:
- Flag `ENABLE_MENU_REELS` cached in Redis

**Test Steps**:
1. PATCH `/api/v1/feature-flags/ENABLE_MENU_REELS/toggle` (admin)
2. Verify Redis DEL called: `feature_flag:ENABLE_MENU_REELS`
3. Verify Redis DEL called: `feature_flag:all`
4. Next GET request → cache miss → DB query

**Expected Result**:
- Cache invalidated immediately
- Both keys deleted (specific + all)
- Next request fetches from DB

**Pass Criteria**: ✅
- Redis DEL called twice
- Cache cleared
- Fresh data on next request

---

### TC-CACHE-004: 5-Minute TTL Expiration

**Objective**: Verify cache expires after 5 minutes

**Test Steps**:
1. GET `/api/v1/feature-flags/check/ENABLE_MENU_REELS`
2. Verify Redis SETEX called with TTL 300
3. Wait 5 minutes
4. GET `/api/v1/feature-flags/check/ENABLE_MENU_REELS` again
5. Verify cache miss → DB query

**Expected Result**:
- Initial request: Cache stored with TTL 300
- After 5 minutes: Cache expired
- Second request: Cache miss → DB query

**Pass Criteria**: ✅
- Cache expires after 300 seconds
- Fresh data fetched from DB

---

## 🔐 Admin Authorization Tests

### TC-AUTH-001: Admin Can Create Flags

**Objective**: Verify JWT + admin role allows flag creation

**Preconditions**:
- Admin authenticated with JWT token (role=admin)

**Test Steps**:
1. POST `/api/v1/feature-flags` with admin token

**Expected Result**:
- Status: `201 Created`
- Flag created successfully

**Pass Criteria**: ✅
- Status = 201
- Admin authorized

---

### TC-AUTH-002: Customer Cannot Create Flags

**Objective**: Verify customer role blocked from creating flags

**Preconditions**:
- Customer authenticated with JWT token (role=customer)

**Test Steps**:
1. POST `/api/v1/feature-flags` with customer token

**Expected Result**:
- Status: `403 Forbidden`
- Error: "Admin role required"

**Pass Criteria**: ✅
- Status = 403
- Customer blocked

---

### TC-AUTH-003: Unauthenticated Request Blocked

**Objective**: Verify no JWT token → blocked

**Test Steps**:
1. POST `/api/v1/feature-flags` without auth header

**Expected Result**:
- Status: `401 Unauthorized`
- Error: "No token provided"

**Pass Criteria**: ✅
- Status = 401
- Auth required

---

### TC-AUTH-004: Public Read Endpoints Work

**Objective**: Verify read endpoints public (no auth)

**Test Steps**:
1. GET `/api/v1/feature-flags` without auth
2. GET `/api/v1/feature-flags/check/ENABLE_MENU_REELS` without auth

**Expected Result**:
- Status: `200 OK` for both requests
- Flags returned successfully

**Pass Criteria**: ✅
- Status = 200
- No auth required for reads

---

## 🔗 Integration Tests

### TC-INT-001: Mobile App Fetches Flags on Startup

**Objective**: Test end-to-end flag fetching from mobile app

**Test Steps**:
1. Mobile app starts
2. App calls GET `/api/v1/feature-flags`
3. App caches flags locally (AsyncStorage)
4. App checks flag: `isEnabled('ENABLE_MENU_REELS')`
5. App shows/hides feature based on flag

**Expected Result**:
- Flags fetched successfully
- Flags cached locally
- Feature visibility controlled by flag

**Pass Criteria**: ✅
- API call succeeds
- Flags cached
- Feature visibility correct

---

### TC-INT-002: Admin Toggles Flag → Mobile App Updates

**Objective**: Test flag change propagation to mobile app

**Test Steps**:
1. Mobile app caches flags (ENABLE_MENU_REELS=false)
2. Admin toggles flag to true via admin panel
3. Mobile app background refresh (30 min interval)
4. Mobile app fetches fresh flags
5. Mobile app detects change (false → true)
6. Mobile app triggers re-render

**Expected Result**:
- Flag change detected
- Feature enabled in app (reels appear on menu)
- Change propagates within 30 minutes

**Pass Criteria**: ✅
- Flag change detected
- App UI updates
- Change propagates

---

### TC-INT-003: Backend Service Checks Flag

**Objective**: Test backend service integration

**Test Steps**:
1. Order service calls `featureFlagService.isEnabled('ENABLE_AUTO_RIDER_ASSIGNMENT')`
2. Flag check returns true
3. Order service auto-assigns rider

**Expected Result**:
- Flag check succeeds
- Feature logic executed
- Rider assigned automatically

**Pass Criteria**: ✅
- Flag check works
- Feature enabled
- Logic executes

---

## ⚡ Performance Tests

### TC-PERF-001: 1000 Concurrent Flag Checks

**Objective**: Verify system handles high load

**Test Steps**:
1. Simulate 1000 concurrent requests:
   - GET `/api/v1/feature-flags/check/ENABLE_MENU_REELS`
2. Measure response times

**Expected Result**:
- All 1000 requests succeed
- p95 latency < 50ms
- p99 latency < 100ms
- Cache hit rate > 90%

**Pass Criteria**: ✅
- No failed requests
- Latency within limits
- High cache hit rate

---

### TC-PERF-002: Cache Hit Rate Measurement

**Objective**: Verify 95% cache hit rate achieved

**Test Steps**:
1. Make 1000 requests for same flag
2. Measure cache hits vs DB queries

**Expected Result**:
- 950+ cache hits (95%)
- 50 or fewer DB queries (5%)

**Pass Criteria**: ✅
- Cache hit rate ≥ 95%
- DB load minimal

---

### TC-PERF-003: Flag Toggle Performance

**Objective**: Verify kill switch is fast

**Test Steps**:
1. Admin toggles flag OFF
2. Measure time to propagate
3. Next request fetches updated flag

**Expected Result**:
- Toggle API response < 100ms
- Cache invalidated immediately
- Next request sees new value within 5 seconds

**Pass Criteria**: ✅
- Toggle fast (< 100ms)
- Change propagates quickly

---

## 📊 Test Summary

| Category | Total Tests | Pass | Fail | Coverage |
|----------|-------------|------|------|----------|
| Flag CRUD | 6 | TBD | TBD | 100% |
| Gradual Rollout | 3 | TBD | TBD | 95% |
| Caching | 4 | TBD | TBD | 100% |
| Authorization | 4 | TBD | TBD | 100% |
| Integration | 3 | TBD | TBD | 90% |
| Performance | 3 | TBD | TBD | N/A |
| **TOTAL** | **23** | **TBD** | **TBD** | **97%** |

---

**[QA_TEST_CASES_COMPLETE ✅]**

**Module**: Feature Flags  
**Test Cases**: 23 comprehensive test scenarios  
**Coverage**: Flag CRUD, gradual rollout, caching, authorization, integration, performance
