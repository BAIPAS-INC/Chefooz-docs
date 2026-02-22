# Cache Module - QA Test Cases

**Module**: `cache`  
**Test Coverage**: Backend Infrastructure  
**Last Updated**: 2026-02-23  
**Status**: ✅ Production Ready

---

## 🎯 Test Scope

### In Scope
- ✅ Basic cache operations (get/set/delete)
- ✅ TTL expiration behavior
- ✅ Distributed locking
- ✅ Rate limiting enforcement
- ✅ Pub/sub messaging
- ✅ Set/hash/counter operations
- ✅ Redis connection handling
- ✅ In-memory fallback behavior
- ✅ Decorator functionality
- ✅ Admin endpoints

### Out of Scope
- ❌ Redis cluster setup (infrastructure)
- ❌ Network latency testing
- ❌ Large-scale load testing (separate performance tests)

---

## 📋 Test Categories

### 1. Functional Tests (20 test cases)
### 2. Edge Case Tests (12 test cases)
### 3. Error Handling Tests (10 test cases)
### 4. Security Tests (8 test cases)
### 5. Performance Tests (5 test cases)
### 6. Integration Tests (8 test cases)
### 7. Regression Tests (5 test cases)

**Total**: 68 test cases

---

## 1️⃣ Functional Tests (20 cases)

### TC-CACHE-F001: Set and Get Simple Value
**Priority**: P0  
**Type**: Positive  

**Preconditions**:
- Backend server running
- Cache service initialized

**Test Steps**:
1. Call `cacheService.set('test:key', 'test-value')`
2. Call `cacheService.get('test:key')`
3. Verify returned value

**Expected Result**:
```typescript
value === 'test-value'
```

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-F002: Set Value with TTL
**Priority**: P0  
**Type**: Positive  

**Test Steps**:
1. Set key with 2-second TTL: `cacheService.set('test:ttl', 'value', 2)`
2. Immediately get value: `cacheService.get('test:ttl')`
3. Wait 3 seconds
4. Get value again

**Expected Result**:
- Step 2: Returns `'value'`
- Step 4: Returns `undefined` (expired)

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-F003: Get Non-Existent Key
**Priority**: P0  
**Type**: Negative  

**Test Steps**:
1. Get non-existent key: `cacheService.get('nonexistent')`

**Expected Result**:
```typescript
value === undefined
```

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-F004: Delete Existing Key
**Priority**: P0  
**Type**: Positive  

**Test Steps**:
1. Set key: `cacheService.set('test:delete', 'value')`
2. Delete key: `cacheService.del('test:delete')`
3. Get key: `cacheService.get('test:delete')`

**Expected Result**:
- Step 3 returns `undefined`

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-F005: Pattern Deletion
**Priority**: P1  
**Type**: Positive  

**Test Steps**:
1. Set multiple keys:
   ```typescript
   await cacheService.set('feed:user:123:page:1', 'data1');
   await cacheService.set('feed:user:123:page:2', 'data2');
   await cacheService.set('feed:user:456:page:1', 'data3');
   ```
2. Delete pattern: `const count = await cacheService.delPattern('feed:user:123:*')`
3. Verify keys deleted

**Expected Result**:
- Returns `count === 2`
- `feed:user:123:page:1` is undefined
- `feed:user:123:page:2` is undefined
- `feed:user:456:page:1` still exists

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-F006: Get-or-Set Pattern (Cache Miss)
**Priority**: P0  
**Type**: Positive  

**Test Steps**:
1. Call `getOrSet` with non-existent key:
   ```typescript
   const value = await cacheService.getOrSet(
     'test:compute',
     60,
     async () => {
       return { computed: true, timestamp: Date.now() };
     }
   );
   ```
2. Get same key again

**Expected Result**:
- Step 1: Function executes, returns computed value
- Step 2: Returns same value (cached)

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-F007: Get-or-Set Pattern (Cache Hit)
**Priority**: P0  
**Type**: Positive  

**Test Steps**:
1. Set key manually: `cacheService.set('test:cached', { cached: true })`
2. Call `getOrSet`:
   ```typescript
   const value = await cacheService.getOrSet(
     'test:cached',
     60,
     async () => {
       throw new Error('Should not execute');
     }
   );
   ```

**Expected Result**:
- Returns `{ cached: true }`
- Function never executes (no error thrown)

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-F008: Acquire Lock Successfully
**Priority**: P0  
**Type**: Positive  

**Test Steps**:
1. Acquire lock: `const token = await cacheService.acquireLock('test:lock', 5000)`
2. Verify token is returned

**Expected Result**:
- `token` is a non-empty string
- Second acquire attempt returns `null`

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-F009: Release Lock Successfully
**Priority**: P0  
**Type**: Positive  

**Test Steps**:
1. Acquire lock: `const token = await cacheService.acquireLock('test:lock', 5000)`
2. Release lock: `const released = await cacheService.releaseLock('test:lock', token)`
3. Acquire lock again

**Expected Result**:
- Step 2: `released === true`
- Step 3: New token returned (lock available)

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-F010: Lock Auto-Expiry
**Priority**: P1  
**Type**: Positive  

**Test Steps**:
1. Acquire lock with 2-second TTL: `const token = await cacheService.acquireLock('test:expiry', 2000)`
2. Wait 3 seconds
3. Acquire same lock again

**Expected Result**:
- Step 3: New token returned (old lock expired)

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-F011: Rate Limit Check (Within Limit)
**Priority**: P0  
**Type**: Positive  

**Test Steps**:
1. Check rate limit: `const allowed = await cacheService.checkRateLimit('test:rate', 5, 60)`
2. Repeat 4 more times (total 5 requests)

**Expected Result**:
- All 5 requests return `allowed === true`

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-F012: Rate Limit Check (Exceeded)
**Priority**: P0  
**Type**: Negative  

**Test Steps**:
1. Make 5 requests to rate-limited endpoint (limit: 5 per minute)
2. Make 6th request

**Expected Result**:
- First 5 requests: `allowed === true`
- 6th request: `allowed === false`

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-F013: Pub/Sub Message Delivery
**Priority**: P1  
**Type**: Positive  

**Test Steps**:
1. Subscribe to channel:
   ```typescript
   let received = null;
   await cacheService.subscribe('test:channel', (msg) => {
     received = msg;
   });
   ```
2. Publish message: `await cacheService.publish('test:channel', { data: 'hello' })`
3. Wait 100ms
4. Check `received` variable

**Expected Result**:
- `received === { data: 'hello' }`

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-F014: Set Add Members
**Priority**: P1  
**Type**: Positive  

**Test Steps**:
1. Add members: `await cacheService.sadd('test:set', 'member1', 'member2', 'member3')`
2. Get cardinality: `const count = await cacheService.scard('test:set')`
3. Check membership: `const exists = await cacheService.sismember('test:set', 'member2')`

**Expected Result**:
- `count === 3`
- `exists === true`

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-F015: Set Remove Members
**Priority**: P1  
**Type**: Positive  

**Test Steps**:
1. Add members: `await cacheService.sadd('test:set', 'a', 'b', 'c')`
2. Remove member: `await cacheService.srem('test:set', 'b')`
3. Get all members: `const members = await cacheService.smembers('test:set')`

**Expected Result**:
- `members` contains `['a', 'c']` (no 'b')

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-F016: Hash Set and Get
**Priority**: P1  
**Type**: Positive  

**Test Steps**:
1. Set hash fields:
   ```typescript
   await cacheService.hset('test:hash', 'field1', 'value1');
   await cacheService.hset('test:hash', 'field2', 'value2');
   ```
2. Get single field: `const val = await cacheService.hget('test:hash', 'field1')`
3. Get all fields: `const all = await cacheService.hgetall('test:hash')`

**Expected Result**:
- `val === 'value1'`
- `all === { field1: 'value1', field2: 'value2' }`

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-F017: Counter Increment
**Priority**: P1  
**Type**: Positive  

**Test Steps**:
1. Increment: `const count1 = await cacheService.incr('test:counter')`
2. Increment by 5: `const count2 = await cacheService.incrby('test:counter', 5)`
3. Get value: `const final = await cacheService.get('test:counter')`

**Expected Result**:
- `count1 === 1`
- `count2 === 6`
- `final === 6`

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-F018: TTL Check
**Priority**: P1  
**Type**: Positive  

**Test Steps**:
1. Set key with 60-second TTL: `await cacheService.set('test:ttl', 'value', 60)`
2. Check TTL: `const ttl = await cacheService.ttl('test:ttl')`
3. Check non-existent key: `const ttl2 = await cacheService.ttl('nonexistent')`

**Expected Result**:
- `ttl` is between 55-60 seconds
- `ttl2 === -2` (key doesn't exist)

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-F019: Expire Existing Key
**Priority**: P1  
**Type**: Positive  

**Test Steps**:
1. Set key without TTL: `await cacheService.set('test:expire', 'value')`
2. Set expiry: `await cacheService.expire('test:expire', 2)`
3. Wait 3 seconds
4. Get key

**Expected Result**:
- Step 4: Returns `undefined` (expired)

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-F020: Reset Entire Cache
**Priority**: P2  
**Type**: Positive  

**Test Steps**:
1. Set multiple keys:
   ```typescript
   await cacheService.set('key1', 'value1');
   await cacheService.set('key2', 'value2');
   ```
2. Reset cache: `await cacheService.reset()`
3. Get keys

**Expected Result**:
- Both keys return `undefined` (cache cleared)

**Actual Result**: _[To be filled during testing]_

---

## 2️⃣ Edge Case Tests (12 cases)

### TC-CACHE-E001: Store Large Object (1MB)
**Priority**: P1  
**Type**: Edge Case  

**Test Steps**:
1. Create 1MB object:
   ```typescript
   const largeObj = { data: 'x'.repeat(1024 * 1024) };
   ```
2. Store in cache: `await cacheService.set('test:large', largeObj)`
3. Retrieve: `const result = await cacheService.get('test:large')`

**Expected Result**:
- Object stored and retrieved successfully
- `result.data.length === 1024 * 1024`

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-E002: Store Empty String
**Priority**: P2  
**Type**: Edge Case  

**Test Steps**:
1. Store empty string: `await cacheService.set('test:empty', '')`
2. Retrieve: `const result = await cacheService.get('test:empty')`

**Expected Result**:
- Returns `''` (not `undefined`)

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-E003: Store Null Value
**Priority**: P2  
**Type**: Edge Case  

**Test Steps**:
1. Store null: `await cacheService.set('test:null', null)`
2. Retrieve: `const result = await cacheService.get('test:null')`

**Expected Result**:
- Returns `null` (not `undefined`)

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-E004: TTL of 0 Seconds
**Priority**: P2  
**Type**: Edge Case  

**Test Steps**:
1. Set key with TTL=0: `await cacheService.set('test:zero', 'value', 0)`
2. Immediately get key

**Expected Result**:
- Returns `undefined` (expired immediately) or error

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-E005: Negative TTL
**Priority**: P2  
**Type**: Edge Case  

**Test Steps**:
1. Set key with TTL=-1: `await cacheService.set('test:negative', 'value', -1)`

**Expected Result**:
- Key stored with no expiry (persistent) or error

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-E006: Lock with Zero TTL
**Priority**: P2  
**Type**: Edge Case  

**Test Steps**:
1. Acquire lock with TTL=0: `const token = await cacheService.acquireLock('test:zero-ttl', 0)`

**Expected Result**:
- Lock acquired but expires immediately or error

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-E007: Release Lock with Wrong Token
**Priority**: P1  
**Type**: Edge Case  

**Test Steps**:
1. Acquire lock: `const token = await cacheService.acquireLock('test:lock', 5000)`
2. Try to release with wrong token: `const released = await cacheService.releaseLock('test:lock', 'wrong-token')`

**Expected Result**:
- `released === false`
- Lock still held

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-E008: Rate Limit with Zero Limit
**Priority**: P2  
**Type**: Edge Case  

**Test Steps**:
1. Check rate limit with limit=0: `const allowed = await cacheService.checkRateLimit('test:zero', 0, 60)`

**Expected Result**:
- `allowed === false` (no requests allowed)

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-E009: Subscribe to Same Channel Twice
**Priority**: P2  
**Type**: Edge Case  

**Test Steps**:
1. Subscribe to channel twice with different callbacks
2. Publish message
3. Verify both callbacks receive message

**Expected Result**:
- Both callbacks invoked

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-E010: Add Duplicate Members to Set
**Priority**: P2  
**Type**: Edge Case  

**Test Steps**:
1. Add members: `await cacheService.sadd('test:set', 'member1', 'member2')`
2. Add duplicates: `const added = await cacheService.sadd('test:set', 'member1', 'member3')`
3. Get cardinality

**Expected Result**:
- `added === 1` (only member3 added)
- Cardinality = 3 (member1, member2, member3)

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-E011: Pattern Delete with No Matches
**Priority**: P2  
**Type**: Edge Case  

**Test Steps**:
1. Delete pattern with no matches: `const count = await cacheService.delPattern('nonexistent:*')`

**Expected Result**:
- `count === 0`
- No error thrown

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-E012: Concurrent Lock Acquisition
**Priority**: P1  
**Type**: Edge Case  

**Test Steps**:
1. Acquire same lock from 10 concurrent requests:
   ```typescript
   const promises = Array.from({ length: 10 }, () => 
     cacheService.acquireLock('test:concurrent', 5000)
   );
   const tokens = await Promise.all(promises);
   ```

**Expected Result**:
- Only 1 token is non-null
- Other 9 are null

**Actual Result**: _[To be filled during testing]_

---

## 3️⃣ Error Handling Tests (10 cases)

### TC-CACHE-ERR001: Redis Connection Failure
**Priority**: P0  
**Type**: Error Handling  

**Test Steps**:
1. Stop Redis server
2. Restart backend
3. Perform cache operation: `await cacheService.set('test:key', 'value')`
4. Verify fallback to in-memory mode

**Expected Result**:
- Warning logged: "Using in-memory fallback"
- Operation succeeds (in-memory)
- No error thrown

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-ERR002: Redis Connection Timeout
**Priority**: P1  
**Type**: Error Handling  

**Test Steps**:
1. Configure invalid Redis host (non-responsive)
2. Start backend
3. Verify 8-second timeout
4. Check fallback to in-memory

**Expected Result**:
- Startup not blocked
- Error logged after 8 seconds
- In-memory mode activated

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-ERR003: Invalid Serialization
**Priority**: P2  
**Type**: Error Handling  

**Test Steps**:
1. Try to cache circular object:
   ```typescript
   const obj: any = { name: 'test' };
   obj.self = obj; // Circular reference
   await cacheService.set('test:circular', obj);
   ```

**Expected Result**:
- Error thrown or logged
- Operation fails gracefully

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-ERR004: Rate Limit Guard with Missing Config
**Priority**: P1  
**Type**: Error Handling  

**Test Steps**:
1. Apply `@RateLimit()` decorator without parameters
2. Make request

**Expected Result**:
- Uses default values (20 requests per 60 seconds)
- No error thrown

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-ERR005: Lock Guard with Missing Config
**Priority**: P1  
**Type**: Error Handling  

**Test Steps**:
1. Apply `@WithLock()` decorator without parameters
2. Make request

**Expected Result**:
- Error logged or default behavior
- Request not blocked

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-ERR006: Publish to Non-Existent Channel
**Priority**: P2  
**Type**: Error Handling  

**Test Steps**:
1. Publish message without any subscribers:
   ```typescript
   await cacheService.publish('nonexistent:channel', { data: 'test' });
   ```

**Expected Result**:
- No error thrown
- Operation succeeds silently

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-ERR007: Get After Reset
**Priority**: P2  
**Type**: Error Handling  

**Test Steps**:
1. Set key: `await cacheService.set('test:key', 'value')`
2. Reset cache: `await cacheService.reset()`
3. Get key: `await cacheService.get('test:key')`

**Expected Result**:
- Returns `undefined`

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-ERR008: Redis AUTH Failure
**Priority**: P1  
**Type**: Error Handling  

**Test Steps**:
1. Configure wrong password in `VALKEY_PASSWORD`
2. Start backend
3. Check logs

**Expected Result**:
- Error logged: "WRONGPASS Authentication failed"
- Fallback to in-memory mode
- Application still starts

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-ERR009: Cluster Mode on Standalone Endpoint
**Priority**: P1  
**Type**: Error Handling  

**Test Steps**:
1. Set `VALKEY_CLUSTER=true` for standalone Redis
2. Start backend
3. Check logs

**Expected Result**:
- Error logged: "Failed to refresh slots cache"
- Fallback to in-memory mode

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-ERR010: Memory Overflow (In-Memory Mode)
**Priority**: P2  
**Type**: Error Handling  

**Test Steps**:
1. Use in-memory mode
2. Store 100,000 keys
3. Monitor memory usage

**Expected Result**:
- No crash
- Automatic cleanup of expired entries
- Memory usage stabilizes

**Actual Result**: _[To be filled during testing]_

---

## 4️⃣ Security Tests (8 cases)

### TC-CACHE-SEC001: Rate Limit Prevents Brute Force
**Priority**: P0  
**Type**: Security  

**Test Steps**:
1. Apply rate limit to OTP endpoint: `@RateLimit(3, 3600)`
2. Make 10 rapid requests
3. Verify 429 status after 3 requests

**Expected Result**:
- First 3 requests: Success
- Remaining 7 requests: 429 Too Many Requests

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-SEC002: Lock Prevents Duplicate Orders
**Priority**: P0  
**Type**: Security  

**Test Steps**:
1. Apply lock to order creation: `@WithLock('order:create', 10000)`
2. Send 5 concurrent order creation requests
3. Verify only 1 succeeds

**Expected Result**:
- 1 request: Order created
- 4 requests: 409 Conflict

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-SEC003: Lock Token Cannot Be Guessed
**Priority**: P1  
**Type**: Security  

**Test Steps**:
1. Acquire 100 locks
2. Analyze token format
3. Attempt to release with guessed token

**Expected Result**:
- Tokens are random (no predictable pattern)
- Guessed token fails to release lock

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-SEC004: Rate Limit by IP (Unauthenticated)
**Priority**: P1  
**Type**: Security  

**Test Steps**:
1. Make requests without authentication
2. Verify rate limit applied by IP address

**Expected Result**:
- Rate limit key includes IP address
- Different IPs get separate limits

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-SEC005: Rate Limit by User (Authenticated)
**Priority**: P1  
**Type**: Security  

**Test Steps**:
1. Make requests with authentication
2. Verify rate limit applied by user ID

**Expected Result**:
- Rate limit key includes user ID
- Same user from different IPs shares limit

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-SEC006: No PII in Cache Keys
**Priority**: P1  
**Type**: Security  

**Test Steps**:
1. List all cache keys: `GET /cache/keys` (admin endpoint)
2. Verify no sensitive data in keys

**Expected Result**:
- No emails, phone numbers, names in keys
- Only user/chef IDs used

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-SEC007: TLS Connection in Production
**Priority**: P0  
**Type**: Security  

**Test Steps**:
1. Check `VALKEY_TLS` environment variable
2. Verify TLS handshake in logs

**Expected Result**:
- `VALKEY_TLS=true` in production
- Log: "TLS enabled for Valkey/Redis connection"

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-SEC008: Admin Endpoints Protected
**Priority**: P0  
**Type**: Security  

**Test Steps**:
1. Try to access `/cache/reset` without authentication
2. Verify 401 Unauthorized

**Expected Result**:
- Admin endpoints require authentication
- Only admin role can access

**Actual Result**: _[To be filled during testing]_

---

## 5️⃣ Performance Tests (5 cases)

### TC-CACHE-PERF001: Cache Hit Latency
**Priority**: P1  
**Type**: Performance  

**Test Steps**:
1. Store 1000 keys
2. Measure 100 GET operations (cache hits)
3. Calculate p50, p95, p99 latencies

**Expected Result**:
- p50: <5ms
- p95: <10ms
- p99: <20ms

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-PERF002: Cache Miss Latency
**Priority**: P1  
**Type**: Performance  

**Test Steps**:
1. Measure 100 GET operations for non-existent keys
2. Calculate p50, p95, p99 latencies

**Expected Result**:
- p50: <10ms
- p95: <20ms
- p99: <50ms

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-PERF003: Lock Contention Overhead
**Priority**: P2  
**Type**: Performance  

**Test Steps**:
1. Measure lock-free operation latency (baseline)
2. Add `@WithLock()` decorator
3. Measure latency increase

**Expected Result**:
- Overhead: <5ms when lock available
- Overhead: <10ms when waiting for lock

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-PERF004: Pattern Deletion Performance
**Priority**: P2  
**Type**: Performance  

**Test Steps**:
1. Store 10,000 keys matching pattern `test:*`
2. Measure time to delete: `await cacheService.delPattern('test:*')`

**Expected Result**:
- Deletion completes in <100ms

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-PERF005: Pub/Sub Message Latency
**Priority**: P2  
**Type**: Performance  

**Test Steps**:
1. Subscribe to channel
2. Publish 100 messages
3. Measure time from publish to callback

**Expected Result**:
- p50: <1ms
- p99: <10ms

**Actual Result**: _[To be filled during testing]_

---

## 6️⃣ Integration Tests (8 cases)

### TC-CACHE-INT001: Feed Cache Integration
**Priority**: P0  
**Type**: Integration  

**Test Steps**:
1. Get feed (cache miss): `GET /feed?page=1`
2. Get feed again (cache hit): `GET /feed?page=1`
3. Upload new reel
4. Get feed (cache invalidated): `GET /feed?page=1`

**Expected Result**:
- Step 1: Slow (DB query)
- Step 2: Fast (<50ms, cached)
- Step 4: Contains new reel

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-INT002: Order Creation Lock
**Priority**: P0  
**Type**: Integration  

**Test Steps**:
1. Create order: `POST /orders` (user A)
2. Immediately create another order (user A)
3. Wait 10 seconds
4. Create order again (user A)

**Expected Result**:
- Step 1: Success
- Step 2: 409 Conflict (lock held)
- Step 4: Success (lock expired)

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-INT003: OTP Rate Limit Integration
**Priority**: P0  
**Type**: Integration  

**Test Steps**:
1. Send OTP: `POST /auth/send-otp` (phone: +911234567890)
2. Repeat 2 more times
3. Try 4th time
4. Wait 1 hour
5. Try again

**Expected Result**:
- Steps 1-3: Success
- Step 4: 429 Too Many Requests
- Step 5: Success (window reset)

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-INT004: Upload Rate Limit (Tier-Based)
**Priority**: P1  
**Type**: Integration  

**Test Steps**:
1. Upload 10 reels as Bronze user (within 1 week)
2. Try 11th upload
3. Upgrade to Silver tier
4. Try upload again

**Expected Result**:
- Step 2: 429 Too Many Requests
- Step 4: Success (Silver tier allows daily uploads)

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-INT005: Search Result Caching
**Priority**: P1  
**Type**: Integration  

**Test Steps**:
1. Search: `GET /search?keyword=pizza&page=1`
2. Measure response time
3. Search same keyword again
4. Measure response time

**Expected Result**:
- Step 2: 200-400ms (Elasticsearch query)
- Step 4: <50ms (cached)

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-INT006: Payment Lock Integration
**Priority**: P0  
**Type**: Integration  

**Test Steps**:
1. Submit payment: `POST /payments/process`
2. Immediately submit again (double-click scenario)
3. Verify only 1 payment processed

**Expected Result**:
- Step 1: Payment processed
- Step 2: 409 Conflict
- Only 1 charge on Razorpay

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-INT007: Redis Reconnection
**Priority**: P1  
**Type**: Integration  

**Test Steps**:
1. Backend running with Redis connected
2. Stop Redis server
3. Make cache request (should fallback)
4. Start Redis server
5. Make cache request (should reconnect)

**Expected Result**:
- Step 3: Uses in-memory fallback
- Step 5: Reconnects to Redis automatically

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-INT008: Multi-Instance Cache Sharing
**Priority**: P1  
**Type**: Integration  

**Test Steps**:
1. Start 2 backend instances (both connected to same Redis)
2. Acquire lock on instance 1
3. Try to acquire same lock on instance 2
4. Release lock on instance 1
5. Acquire lock on instance 2

**Expected Result**:
- Step 3: Lock denied (held by instance 1)
- Step 5: Lock acquired (distributed lock works)

**Actual Result**: _[To be filled during testing]_

---

## 7️⃣ Regression Tests (5 cases)

### TC-CACHE-REG001: Valkey Authentication Fix
**Priority**: P0  
**Type**: Regression  

**Reference**: VALKEY_FIX_SUMMARY.md

**Test Steps**:
1. Configure ElastiCache Serverless credentials:
   ```bash
   VALKEY_USERNAME=default
   VALKEY_PASSWORD=<password>
   VALKEY_TLS=true
   VALKEY_CLUSTER=false
   ```
2. Start backend
3. Verify connection

**Expected Result**:
- Log: "✅ Connected to Valkey/Redis"
- Log: "✅ Redis client is ready and authenticated"
- No NOAUTH errors

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-REG002: Cluster Mode Fix
**Priority**: P0  
**Type**: Regression  

**Reference**: VALKEY_FIX_SUMMARY.md

**Test Steps**:
1. Set `VALKEY_CLUSTER=false` for ElastiCache Serverless
2. Start backend
3. Verify no cluster errors

**Expected Result**:
- No "Failed to refresh slots cache" errors
- Standalone mode used

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-REG003: Lock Cleanup Job
**Priority**: P1  
**Type**: Regression  

**Test Steps**:
1. Use in-memory mode
2. Acquire 100 locks with 1-second TTL
3. Wait 2 minutes
4. Check cache size

**Expected Result**:
- Expired locks cleaned up automatically
- Cache size returns to normal

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-REG004: Feed Invalidation on Reel Upload
**Priority**: P0  
**Type**: Regression  

**Test Steps**:
1. Get feed page (cache it)
2. Upload new reel
3. Get feed page again
4. Verify new reel appears

**Expected Result**:
- Cache invalidated via pub/sub
- New reel visible immediately

**Actual Result**: _[To be filled during testing]_

---

### TC-CACHE-REG005: Admin Endpoint Security
**Priority**: P0  
**Type**: Regression  

**Test Steps**:
1. Try to access admin endpoints without auth:
   - `POST /cache/clear-locks`
   - `POST /cache/reset`
   - `GET /cache/keys`
   - `GET /cache/stats`

**Expected Result**:
- All return 401 Unauthorized

**Actual Result**: _[To be filled during testing]_

---

## 🧪 Manual Testing Checklists

### Checklist 1: Basic Cache Operations
- [ ] Set and get simple value
- [ ] Set and get with TTL (verify expiration)
- [ ] Delete key
- [ ] Pattern deletion
- [ ] Reset entire cache

### Checklist 2: Distributed Locking
- [ ] Acquire lock successfully
- [ ] Second acquire fails (lock held)
- [ ] Release lock successfully
- [ ] Lock auto-expires after TTL
- [ ] Wrong token cannot release lock

### Checklist 3: Rate Limiting
- [ ] Within limit: Requests allowed
- [ ] Exceeded limit: 429 status
- [ ] Different users have separate limits
- [ ] Window reset after TTL

### Checklist 4: Redis Connection
- [ ] Connect to Redis successfully
- [ ] Fallback to in-memory on failure
- [ ] TLS connection in production
- [ ] ACL authentication works
- [ ] Logs show connection status

### Checklist 5: Decorators
- [ ] `@RateLimit()` enforces limits
- [ ] `@WithLock()` acquires locks
- [ ] Decorators can be combined
- [ ] Guards handle errors gracefully

---

## 🔧 Test Environment Setup

### Prerequisites
```bash
# Install Redis locally (for testing)
brew install redis  # macOS
sudo apt install redis-server  # Ubuntu

# Start Redis
redis-server

# Configure test environment
cp .env.test.example .env.test
```

### Test Environment Variables
```bash
# .env.test
NODE_ENV=test
VALKEY_ENABLED=true
VALKEY_HOST=localhost
VALKEY_PORT=6379
VALKEY_TLS=false
VALKEY_CLUSTER=false
```

### Running Tests
```bash
# Unit tests
npm run test:cache

# Integration tests
npm run test:integration

# E2E tests
npm run test:e2e

# Coverage
npm run test:coverage
```

---

## 📊 Test Metrics

### Success Criteria
- ✅ 100% of P0 test cases pass
- ✅ 95% of P1 test cases pass
- ✅ 90% of P2 test cases pass
- ✅ Code coverage >80%

### Performance Benchmarks
| Operation | Target | Actual |
|-----------|--------|--------|
| Cache GET (hit) | <5ms | _TBD_ |
| Cache SET | <5ms | _TBD_ |
| Lock acquire | <10ms | _TBD_ |
| Rate limit check | <5ms | _TBD_ |
| Pattern delete (1000 keys) | <100ms | _TBD_ |

---

## 🐛 Known Issues & Workarounds

### Issue 1: Pattern Deletion Slowness
**Description**: `delPattern()` uses KEYS command (slow on large datasets)  
**Workaround**: Use specific key deletion instead of patterns  
**Status**: Tracked in TODO

### Issue 2: In-Memory Mode Not Distributed
**Description**: Locks/limits not shared across instances in fallback mode  
**Workaround**: Always use Redis in production  
**Status**: By design

---

## 📚 Related Test Documentation

- [Feed Module QA Test Cases](../feed/QA_TEST_CASES.md)
- [Auth Module QA Test Cases](../auth/QA_TEST_CASES.md)
- [Order Module QA Test Cases](../order/QA_TEST_CASES.md)

---

**Document Version**: 1.0  
**Last Updated**: 2026-02-23  
**Test Coverage**: 68 test cases  
**Estimated Testing Time**: 8-12 hours
