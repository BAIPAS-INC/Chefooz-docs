# Cache Module - Feature Overview

**Module**: `cache`  
**Type**: Infrastructure & Performance  
**Status**: ✅ Production Ready  
**Dependencies**: Redis/Valkey (optional), ioredis  
**Related Modules**: All modules (global dependency)

---

## 🎯 Business Purpose

The Cache module provides enterprise-grade caching, rate limiting, and distributed locking capabilities across the Chefooz platform. It serves as the foundation for:

- **Performance Optimization**: Reducing database queries and API response times
- **Rate Limiting**: Protecting APIs from abuse and ensuring fair usage
- **Distributed Locking**: Preventing race conditions in concurrent operations
- **Pub/Sub Messaging**: Enabling real-time cache invalidation across services

### Value Proposition

1. **Improved User Experience**: Faster API responses through intelligent caching
2. **Cost Reduction**: Reduced database load and AWS API costs
3. **System Reliability**: Graceful fallback to in-memory mode when Redis is unavailable
4. **Security**: Built-in rate limiting and abuse prevention
5. **Scalability**: Distributed locking enables horizontal scaling

---

## 👥 User Personas

### 1. Backend Developers
**Need**: Fast, reliable caching for API responses  
**Benefit**: Simple API for caching with automatic fallback

### 2. DevOps Engineers
**Need**: Production-ready Redis/Valkey integration  
**Benefit**: Zero-config fallback, comprehensive logging, health checks

### 3. Security Engineers
**Need**: Rate limiting and abuse prevention  
**Benefit**: Decorator-based rate limiting with tier-aware policies

### 4. System Architects
**Need**: Distributed coordination primitives  
**Benefit**: Distributed locks, pub/sub, and set operations

---

## ✨ Feature Capabilities

### Core Caching Operations

#### 1. Simple Key-Value Caching
- **Get/Set**: Store and retrieve any JSON-serializable data
- **TTL Support**: Automatic expiration after specified seconds
- **Get-or-Set**: Atomic "compute if missing" pattern
- **Pattern Deletion**: Bulk delete keys matching patterns (e.g., `feed:*`)

**Example Use Cases**:
- Feed page caching (`feed:user:123:page:1`)
- Search results caching (`search:keyword:pizza:page:1`)
- User session data (`session:abc123`)

#### 2. Distributed Locking
- **Acquire/Release**: Prevent concurrent execution of critical sections
- **Automatic Expiry**: Locks expire after TTL to prevent deadlocks
- **Token-Based**: Only the lock holder can release the lock
- **Guard Decorator**: Apply locks via `@WithLock('resource')` decorator

**Example Use Cases**:
- Order creation (prevent duplicate orders)
- Payment processing (prevent double charges)
- Media processing (prevent concurrent uploads)
- Withdrawal requests (prevent race conditions)

#### 3. Rate Limiting
- **Sliding Window**: Accurate rate limit enforcement
- **Tier-Aware**: Different limits for Bronze/Silver/Gold/Diamond users
- **IP-Based Fallback**: Rate limit by IP when user is not authenticated
- **Guard Decorator**: Apply limits via `@RateLimit(10, 60)` decorator

**Example Use Cases**:
- OTP generation (3 per hour per phone)
- Media uploads (tier-based weekly/daily limits)
- Order creation (20 per hour)
- Search queries (100 per hour)

#### 4. Pub/Sub Messaging
- **Subscribe**: Listen for events on named channels
- **Publish**: Broadcast events to all subscribers
- **JSON Support**: Automatic serialization/deserialization

**Example Use Cases**:
- Feed cache invalidation (when new reel is uploaded)
- Real-time notifications
- Cross-service coordination

#### 5. Set Operations (Redis Sets)
- **Add/Remove**: Manage membership in sets
- **Check Membership**: Fast O(1) existence checks
- **Get All Members**: Retrieve entire set
- **Cardinality**: Count set members

**Example Use Cases**:
- User followers/following (`followers:user:123`)
- Reel likes (`likes:reel:456`)
- Blocked users (`blocked:user:789`)

#### 6. Hash Operations (Redis Hashes)
- **Set Field**: Store structured data
- **Get Field**: Retrieve specific field
- **Get All**: Retrieve entire hash
- **Delete Field**: Remove specific fields

**Example Use Cases**:
- User preferences (`prefs:user:123`)
- Feature flags per user (`flags:user:456`)
- Session attributes

#### 7. Counter Operations
- **Increment**: Atomic increment operations
- **Decrement**: Atomic decrement operations
- **Increment By**: Add specific amount

**Example Use Cases**:
- View counts (`views:reel:123`)
- Like counts (`likes:reel:456`)
- API request counters

---

## 🏗️ Architecture

### Dual-Mode Operation

```
┌─────────────────────────────────────────┐
│         Cache Service (Global)          │
├─────────────────────────────────────────┤
│  Mode Detection:                        │
│  - VALKEY_ENABLED=true  → Redis/Valkey  │
│  - VALKEY_ENABLED=false → In-Memory     │
└─────────────────────────────────────────┘
              ↓          ↓
    ┌─────────┴──┐   ┌──┴──────────┐
    │   Redis/   │   │  In-Memory  │
    │   Valkey   │   │  Fallback   │
    │ (Production)   │ (Development)│
    └────────────┘   └─────────────┘
```

### Connection Strategy

1. **Startup**: Non-blocking connection with 8-second timeout
2. **Failure Handling**: Automatic fallback to in-memory mode
3. **Runtime**: Graceful degradation (log errors, continue serving)
4. **Shutdown**: Clean disconnection of both client and subscriber

### Supported Deployments

1. **AWS ElastiCache Serverless (Valkey)** - Production
   - Standalone mode (`VALKEY_CLUSTER=false`)
   - ACL authentication (`VALKEY_USERNAME` + `VALKEY_PASSWORD`)
   - TLS enabled (`VALKEY_TLS=true`)

2. **Local Redis** - Development
   - No authentication required
   - Standalone mode
   - No TLS

3. **In-Memory Fallback** - Always Available
   - No external dependencies
   - Automatic cleanup of expired entries
   - Pub/sub via in-process event emitters

---

## 📊 Business Rules

### Rate Limiting Policies

#### OTP Verification
- **Per Phone**: 3 attempts per hour
- **Per IP**: 10 attempts per hour
- **Cooldown**: 30 seconds between consecutive attempts

#### Content Uploads
- **Bronze Tier**: 10 uploads per week
- **Silver Tier**: 5 uploads per day
- **Gold Tier**: 10 uploads per day
- **Diamond/Legend**: Unlimited

#### Order Creation
- **All Users**: 20 orders per hour
- **Prevents**: Spam orders and system abuse

#### Search Queries
- **Authenticated**: 100 searches per hour
- **Guest**: 20 searches per hour

#### Profile Updates
- **Username Change**: Once per 30 days
- **Other Fields**: 10 updates per hour

### Caching Policies

#### Feed Pages
- **TTL**: 5 minutes (configurable via `FEED_CACHE_TTL_SECONDS`)
- **Invalidation**: On new reel upload (pub/sub)
- **Key Pattern**: `feed:user:{userId}:page:{pageNum}`

#### Search Results
- **TTL**: 15 minutes
- **Key Pattern**: `search:{keyword}:page:{pageNum}`

#### Explore Rankings
- **TTL**: 10 minutes
- **Invalidation**: On ranking algorithm changes
- **Key Pattern**: `explore:ranked:page:{pageNum}`

### Lock Policies

#### Order Creation
- **Lock Key**: `lock:order:create:{userId}`
- **TTL**: 10 seconds
- **Purpose**: Prevent duplicate orders

#### Payment Processing
- **Lock Key**: `lock:payment:process:{orderId}`
- **TTL**: 30 seconds
- **Purpose**: Prevent double charges

#### Media Upload
- **Lock Key**: `lock:upload:user:{userId}`
- **TTL**: 60 seconds (long processing time)
- **Purpose**: Prevent concurrent uploads

---

## 🔒 Security Considerations

### Authentication
- **AWS ElastiCache**: ACL-based (username + password)
- **Local Redis**: No auth (development only)
- **Environment Variables**: Never log passwords

### Network Security
- **TLS**: Required for production (AWS ElastiCache)
- **Private Subnets**: Cache should not be internet-accessible
- **Security Groups**: Restrict to application servers only

### Data Privacy
- **No PII in Keys**: Use user IDs, not names/emails
- **Encryption at Rest**: Handled by AWS ElastiCache
- **TTL Enforcement**: Automatic expiration of sensitive data

### Rate Limit Bypass Prevention
- **Multiple Identifiers**: Check both user ID and IP
- **Violation Tracking**: Log and escalate repeated violations
- **Emergency Controls**: Admin endpoints to clear locks/limits

---

## 🚫 Limitations

### Current Constraints

1. **Pattern Deletion**: Uses `KEYS` command (not suitable for large datasets)
   - **Impact**: Slow on millions of keys
   - **Workaround**: Use `SCAN` iteration (TODO)

2. **In-Memory Mode**: Not distributed
   - **Impact**: Rate limits/locks not shared across instances
   - **Workaround**: Use Redis in production

3. **No Persistence Configuration**: Uses default Redis persistence
   - **Impact**: Data loss on restart
   - **Mitigation**: Acceptable for cache data

4. **No Circuit Breaker**: Retries on every request
   - **Impact**: Slight delay on Redis failure
   - **Mitigation**: Fast timeout (5 seconds)

### Known Issues

1. **Lock Cleanup**: Expired locks cleaned every 30 seconds in memory mode
   - **Risk**: Temporary memory buildup
   - **Fix**: Automatic cleanup job

2. **Subscriber Connection**: Single subscriber client
   - **Risk**: Message loss on disconnection
   - **Fix**: Automatic reconnection (ioredis)

---

## 🎨 User Workflows

### Workflow 1: Developer Adds Caching to API

```typescript
// 1. Inject CacheService
constructor(private cacheService: CacheService) {}

// 2. Use get-or-set pattern
const feed = await this.cacheService.getOrSet(
  `feed:user:${userId}:page:${page}`,
  300, // 5 minutes TTL
  async () => {
    // Expensive DB query
    return await this.reelRepository.findFeedForUser(userId, page);
  }
);
```

### Workflow 2: Developer Adds Rate Limiting

```typescript
// Apply decorator to controller method
@Post('send-otp')
@RateLimit(3, 3600) // 3 requests per hour
async sendOtp(@Body() dto: SendOtpDto) {
  // Method automatically rate-limited
}
```

### Workflow 3: Developer Adds Distributed Lock

```typescript
@Post('create-order')
@WithLock('order:create', 10000) // 10-second lock
async createOrder(@Body() dto: CreateOrderDto) {
  // Only one instance can execute at a time per user
}
```

### Workflow 4: Developer Implements Cache Invalidation

```typescript
// Publisher (after creating reel)
await this.cacheService.publish('invalidate:feed', {
  reelId: newReel.id,
  chefId: newReel.chefId
});

// Subscriber (in FeedService constructor)
this.cacheService.subscribe('invalidate:feed', async (message) => {
  await this.cacheService.delPattern('feed:*');
  this.logger.log('Feed cache invalidated');
});
```

---

## 🧮 Performance Characteristics

### Cache Hit Rates (Production Targets)

- **Feed Pages**: 70-80% (high repeat views)
- **Search Results**: 50-60% (varied queries)
- **Explore Rankings**: 80-90% (same for all users)
- **User Profiles**: 60-70% (public profile views)

### Latency Improvements

| Operation | Without Cache | With Cache | Improvement |
|-----------|---------------|------------|-------------|
| Feed Page | 150-300ms | 5-15ms | **20-60x** |
| Search | 200-400ms | 10-20ms | **20-40x** |
| Explore | 100-200ms | 5-10ms | **20-40x** |
| Profile | 80-150ms | 5-10ms | **16-30x** |

### Redis Operation Performance

| Operation | Latency | Notes |
|-----------|---------|-------|
| GET/SET | <1ms | Single key |
| INCR | <1ms | Atomic counter |
| SADD/SISMEMBER | <1ms | Set operations |
| KEYS (pattern) | 5-50ms | Avoid in production |
| Pub/Sub | <1ms | Message delivery |

---

## 📦 Integration Points

### Dependencies

1. **ioredis**: Redis client library
2. **@nestjs/config**: Environment configuration
3. **@chefooz-app/domain**: Rate limit policies

### Used By (Global Module)

- **Feed Module**: Page caching, invalidation
- **Search Module**: Result caching
- **Auth Module**: OTP rate limiting
- **Order Module**: Creation locks, rate limiting
- **Media Module**: Upload rate limiting, locks
- **Payment Module**: Transaction locks
- **Withdrawal Module**: Request locks
- **All Controllers**: Optional rate limiting decorators

---

## 🎯 Success Metrics

### Operational Metrics

1. **Cache Hit Rate**: >60% across all endpoints
2. **Redis Availability**: >99.9% uptime
3. **Fallback Events**: <10 per day
4. **Lock Contention**: <1% of requests blocked

### Business Impact

1. **API Response Time**: <50ms for cached endpoints
2. **Database Load**: 40-60% reduction in queries
3. **OTP Abuse**: Zero fraud incidents
4. **Double Orders**: Zero occurrences

### Cost Savings

1. **Database Costs**: 30-50% reduction
2. **API Costs**: 20-40% reduction (cached responses)
3. **AWS ElastiCache**: $50-150/month (worth the performance)

---

## 🔮 Future Enhancements

### Planned Features

1. **SCAN-Based Pattern Deletion**: Replace KEYS with SCAN for large datasets
2. **Cache Warming**: Pre-populate cache on app startup
3. **Multi-Region Support**: Redis Cluster for global deployments
4. **Analytics Dashboard**: Cache hit rates, lock contention metrics
5. **Advanced Rate Limiting**: Token bucket algorithm, burst allowance
6. **Circuit Breaker**: Skip Redis entirely during outages

### Under Consideration

1. **Cache Compression**: gzip large payloads before storing
2. **Cache Versioning**: Invalidate by version instead of pattern
3. **Distributed Tracing**: OpenTelemetry integration
4. **Cache Aside Proxy**: Transparent caching layer

---

## 📚 Related Documentation

- [Cache Technical Guide](./TECHNICAL_GUIDE.md)
- [Cache QA Test Cases](./QA_TEST_CASES.md)
- [Feed Module](../feed/FEATURE_OVERVIEW.md)
- [Rate Limiting Policies](@chefooz-app/domain)
- [AWS ElastiCache Setup](../../../infra/README.md)

---

**Document Version**: 1.0  
**Last Updated**: 2026-02-23  
**Status**: ✅ Complete and Production Ready
