# Cache Module - Technical Guide

**Module**: `cache`  
**Location**: `apps/chefooz-apis/src/modules/cache/`  
**Type**: Global Infrastructure Module  
**Dependencies**: ioredis, @nestjs/config  

---

## 🏗️ Architecture Overview

### Module Structure

```
apps/chefooz-apis/src/modules/cache/
├── cache.module.ts              # Global module definition
├── cache.service.ts             # Core caching service (911 lines)
├── cache.controller.ts          # Admin/debug endpoints
├── enhanced-rate-limit.service.ts  # Domain-aware rate limiting
├── decorators/
│   ├── rate-limit.decorator.ts  # @RateLimit() decorator
│   └── with-lock.decorator.ts   # @WithLock() decorator
└── guards/
    ├── rate-limit.guard.ts      # Rate limit enforcement
    └── lock.guard.ts            # Distributed lock enforcement
```

### Component Relationships

```
┌─────────────────────────────────────────────────┐
│           CacheModule (Global)                  │
│  Exported to all other modules automatically    │
└─────────────────────────────────────────────────┘
                    │
        ┌───────────┼───────────┐
        ↓           ↓           ↓
┌──────────────┐ ┌─────────────┐ ┌──────────────────┐
│ CacheService │ │  Enhanced   │ │ CacheController  │
│  (Core ops)  │ │ RateLimit   │ │  (Admin only)    │
└──────────────┘ │  Service    │ └──────────────────┘
        ↑         └─────────────┘         ↑
        │                │                │
        │         ┌──────┴──────┐         │
        │         ↓             ↓         │
┌───────────────┐  ┌─────────────────┐   │
│ RateLimit     │  │  Lock Guard     │   │
│ Guard         │  │                 │   │
└───────────────┘  └─────────────────┘   │
        ↑                  ↑              │
        │                  │              │
┌───────────────┐  ┌─────────────────┐   │
│ @RateLimit()  │  │  @WithLock()    │   │
│ Decorator     │  │  Decorator      │   │
└───────────────┘  └─────────────────┘   │
                                          │
                                     [Used by]
                                          │
                        ┌─────────────────┴──────────────┐
                        ↓                                ↓
                 [All Controllers]              [All Services]
```

---

## 📊 Database Schema

The Cache module does not have its own database tables. It manages external Redis/Valkey storage and in-memory data structures.

### Redis Key Patterns

#### Cache Keys
```
feed:user:{userId}:page:{pageNum}          # Feed page cache
search:{keyword}:page:{pageNum}            # Search results
explore:ranked:page:{pageNum}              # Explore rankings
profile:user:{userId}                      # User profile
chef:public:{chefId}                       # Chef public page
```

#### Rate Limit Keys
```
ratelimit:otp:phone:{phone}                # OTP per phone
ratelimit:otp:ip:{ip}                      # OTP per IP
ratelimit:upload:user:{userId}:week        # Weekly upload count
ratelimit:upload:user:{userId}:day         # Daily upload count
ratelimit:user:{userId}:{endpoint}         # Per-user endpoint limit
ratelimit:ip:{ip}:{endpoint}               # Per-IP endpoint limit
```

#### Lock Keys
```
lock:order:create:{userId}                 # Order creation lock
lock:payment:process:{orderId}             # Payment lock
lock:upload:user:{userId}                  # Upload lock
lock:withdrawal:request:{userId}           # Withdrawal lock
```

#### Set Keys (for followers, likes, etc.)
```
followers:user:{userId}                    # User's followers
following:user:{userId}                    # Users being followed
likes:reel:{reelId}                        # Users who liked reel
blocked:user:{userId}                      # Blocked users
```

#### Pub/Sub Channels
```
invalidate:feed                            # Feed cache invalidation
invalidate:search                          # Search cache invalidation
invalidate:explore                         # Explore cache invalidation
```

---

## 🔌 API Reference

### Core Cache Operations

#### `getOrSet<T>(key: string, ttl: number, fn: () => Promise<T>): Promise<T>`

**Purpose**: Get cached value or compute and cache if missing  
**Returns**: Cached or computed value

**Example**:
```typescript
const feed = await this.cacheService.getOrSet(
  'feed:user:123:page:1',
  300, // 5 minutes
  async () => {
    return await this.reelRepository.findFeedForUser(123, 1);
  }
);
```

#### `set(key: string, value: any, ttl?: number): Promise<void>`

**Purpose**: Store value with optional TTL  
**Parameters**:
- `key`: Cache key
- `value`: Any JSON-serializable object
- `ttl`: Time-to-live in seconds (optional)

**Example**:
```typescript
await this.cacheService.set('user:123:prefs', { theme: 'dark' }, 3600);
```

#### `get<T>(key: string): Promise<T | undefined>`

**Purpose**: Retrieve cached value  
**Returns**: Value or undefined if not found/expired

**Example**:
```typescript
const prefs = await this.cacheService.get<UserPrefs>('user:123:prefs');
```

#### `del(key: string): Promise<void>`

**Purpose**: Delete specific key

**Example**:
```typescript
await this.cacheService.del('feed:user:123:page:1');
```

#### `delPattern(pattern: string): Promise<number>`

**Purpose**: Delete all keys matching pattern  
**Returns**: Number of keys deleted  
**Warning**: Uses `KEYS` command (slow on large datasets)

**Example**:
```typescript
const deleted = await this.cacheService.delPattern('feed:user:123:*');
this.logger.log(`Deleted ${deleted} feed cache entries`);
```

#### `reset(): Promise<void>`

**Purpose**: Clear entire cache (nuclear option)  
**Warning**: Use only in emergencies or testing

---

### Distributed Locking

#### `acquireLock(lockKey: string, ttlMs: number): Promise<string | null>`

**Purpose**: Acquire distributed lock  
**Returns**: Token if acquired, null if already locked  
**TTL**: Lock expires after ttlMs milliseconds

**Example**:
```typescript
const token = await this.cacheService.acquireLock('order:create:123', 10000);
if (!token) {
  throw new ConflictException('Another operation in progress');
}

try {
  // Critical section
  await this.createOrder(userId, orderData);
} finally {
  await this.cacheService.releaseLock('order:create:123', token);
}
```

#### `releaseLock(lockKey: string, token: string): Promise<boolean>`

**Purpose**: Release distributed lock  
**Returns**: true if released, false if token doesn't match  
**Safety**: Only the lock holder (with matching token) can release

---

### Rate Limiting

#### `checkRateLimit(key: string, limit: number, ttlSeconds: number): Promise<boolean>`

**Purpose**: Check if rate limit exceeded  
**Returns**: true if allowed, false if limit exceeded  
**Algorithm**: Sliding window counter

**Example**:
```typescript
const allowed = await this.cacheService.checkRateLimit(
  'user:123:POST:/orders',
  20,  // max 20 requests
  3600 // per hour
);

if (!allowed) {
  throw new HttpException('Rate limit exceeded', 429);
}
```

---

### Pub/Sub

#### `subscribe(channel: string, callback: (message: any) => void): Promise<void>`

**Purpose**: Subscribe to pub/sub channel  
**Callback**: Invoked for each message received

**Example**:
```typescript
await this.cacheService.subscribe('invalidate:feed', async (message) => {
  this.logger.log('Feed invalidation event:', message);
  await this.cacheService.delPattern('feed:*');
});
```

#### `publish(channel: string, message: any): Promise<void>`

**Purpose**: Publish message to channel  
**Serialization**: Automatic JSON stringify

**Example**:
```typescript
await this.cacheService.publish('invalidate:feed', {
  reelId: newReel.id,
  chefId: newReel.chefId,
  timestamp: Date.now()
});
```

---

### Set Operations

#### `sadd(key: string, ...members: string[]): Promise<number>`
**Purpose**: Add members to set  
**Returns**: Number of members added

#### `srem(key: string, ...members: string[]): Promise<number>`
**Purpose**: Remove members from set  
**Returns**: Number of members removed

#### `sismember(key: string, member: string): Promise<boolean>`
**Purpose**: Check if member exists in set  
**Returns**: true if exists

#### `smembers(key: string): Promise<string[]>`
**Purpose**: Get all set members  
**Returns**: Array of members

#### `scard(key: string): Promise<number>`
**Purpose**: Get set cardinality (count)  
**Returns**: Number of members

**Example**:
```typescript
// Check if user liked reel
const hasLiked = await this.cacheService.sismember('likes:reel:456', '123');

// Add like
await this.cacheService.sadd('likes:reel:456', '123');

// Get like count
const count = await this.cacheService.scard('likes:reel:456');
```

---

### Hash Operations

#### `hset(key: string, field: string, value: string | number): Promise<number>`
**Purpose**: Set hash field

#### `hget(key: string, field: string): Promise<string | undefined>`
**Purpose**: Get hash field

#### `hgetall(key: string): Promise<Record<string, string>>`
**Purpose**: Get all hash fields

#### `hdel(key: string, ...fields: string[]): Promise<number>`
**Purpose**: Delete hash fields

**Example**:
```typescript
// Store user preferences
await this.cacheService.hset('prefs:user:123', 'theme', 'dark');
await this.cacheService.hset('prefs:user:123', 'language', 'en');

// Retrieve all preferences
const prefs = await this.cacheService.hgetall('prefs:user:123');
// { theme: 'dark', language: 'en' }
```

---

### Counter Operations

#### `incr(key: string): Promise<number>`
**Purpose**: Increment counter by 1

#### `incrby(key: string, amount: number): Promise<number>`
**Purpose**: Increment counter by amount

#### `decr(key: string): Promise<number>`
**Purpose**: Decrement counter by 1

**Example**:
```typescript
// Increment view count
const views = await this.cacheService.incr('views:reel:456');

// Add 10 points
const points = await this.cacheService.incrby('points:user:123', 10);
```

---

### Utility Methods

#### `ttl(key: string): Promise<number>`
**Purpose**: Get time-to-live in seconds  
**Returns**: -2 (doesn't exist), -1 (no expiry), or seconds remaining

#### `expire(key: string, ttlSeconds: number): Promise<boolean>`
**Purpose**: Set expiry on existing key

#### `getClient(): Redis | Cluster | null`
**Purpose**: Get raw Redis client for advanced operations

#### `isRedisAvailable(): boolean`
**Purpose**: Check if Redis is connected

---

## 🎨 Frontend Integration

The Cache module is backend-only. Frontend benefits indirectly through:

1. **Faster API Responses**: Cached endpoints return in <50ms
2. **Rate Limit Errors**: Frontend handles 429 status codes
3. **Lock Conflicts**: Frontend handles 409 status codes

### Handling Rate Limits (Frontend)

```typescript
// api-client/src/lib/orders/orders.hooks.ts
export const useCreateOrder = () => {
  return useMutation({
    mutationFn: (data: CreateOrderDto) => ordersClient.createOrder(data),
    onError: (error) => {
      if (error.response?.status === 429) {
        // Rate limit exceeded
        const retryAfter = error.response.headers['retry-after'];
        toast.error(`Too many requests. Try again in ${retryAfter} seconds.`);
      }
    }
  });
};
```

### Handling Lock Conflicts (Frontend)

```typescript
// api-client/src/lib/orders/orders.hooks.ts
export const useCreateOrder = () => {
  return useMutation({
    mutationFn: (data: CreateOrderDto) => ordersClient.createOrder(data),
    onError: (error) => {
      if (error.response?.status === 409) {
        // Lock conflict
        toast.error('Another operation is in progress. Please wait.');
      }
    },
    retry: (failureCount, error) => {
      // Retry on lock conflicts (409), but not rate limits (429)
      if (error.response?.status === 409 && failureCount < 2) {
        return true;
      }
      return false;
    },
    retryDelay: 1000, // Wait 1 second before retry
  });
};
```

---

## 🔐 Environment Configuration

### Required Variables (Production)

```bash
# Enable Redis/Valkey
VALKEY_ENABLED=true

# AWS ElastiCache Serverless endpoint
VALKEY_HOST=chefooz-caching-emgbcy.serverless.aps1.cache.amazonaws.com
VALKEY_PORT=6379

# Authentication (ACL-based)
VALKEY_USERNAME=default
VALKEY_PASSWORD=your-secure-password

# Connection settings
VALKEY_TLS=true              # Required for AWS ElastiCache
VALKEY_CLUSTER=false         # ElastiCache Serverless = standalone mode

# Optional: Custom cache TTLs
FEED_CACHE_TTL_SECONDS=300   # 5 minutes (default)
```

### Development Configuration

```bash
# Local Redis (no auth)
VALKEY_ENABLED=true
VALKEY_HOST=localhost
VALKEY_PORT=6379
VALKEY_TLS=false
VALKEY_CLUSTER=false

# Or disable for in-memory mode
VALKEY_ENABLED=false
```

### In-Memory Fallback (Zero Config)

If `VALKEY_ENABLED=false` or connection fails:
- All operations use in-memory Map/Set structures
- Pub/sub uses in-process event emitters
- Rate limits/locks NOT shared across instances
- Automatic cleanup every 30 seconds

---

## 🚀 Implementation Guide

### Step 1: Inject CacheService

```typescript
// any.service.ts
import { CacheService } from '../cache/cache.service';

@Injectable()
export class MyService {
  constructor(
    private readonly cacheService: CacheService
  ) {}
}
```

No need to import `CacheModule` — it's global!

### Step 2: Add Caching to Expensive Query

```typescript
async getFeed(userId: string, page: number): Promise<Reel[]> {
  const cacheKey = `feed:user:${userId}:page:${page}`;
  const cacheTtl = 300; // 5 minutes

  return await this.cacheService.getOrSet(
    cacheKey,
    cacheTtl,
    async () => {
      // Expensive DB query only if cache misses
      return await this.reelRepository
        .createQueryBuilder('reel')
        .innerJoin('reel.chef', 'chef')
        .where('chef.id IN (:...followingIds)', { followingIds })
        .orderBy('reel.createdAt', 'DESC')
        .skip((page - 1) * 20)
        .take(20)
        .getMany();
    }
  );
}
```

### Step 3: Add Cache Invalidation

```typescript
async createReel(userId: string, reelData: CreateReelDto): Promise<Reel> {
  const reel = await this.reelRepository.save(reelData);

  // Invalidate feed cache for all followers
  await this.cacheService.publish('invalidate:feed', {
    reelId: reel.id,
    chefId: userId,
    timestamp: Date.now()
  });

  return reel;
}
```

### Step 4: Add Rate Limiting

```typescript
// orders.controller.ts
import { RateLimit } from '../cache/decorators/rate-limit.decorator';

@Post()
@RateLimit(20, 3600) // 20 orders per hour
async createOrder(@Body() dto: CreateOrderDto) {
  return await this.ordersService.createOrder(dto);
}
```

### Step 5: Add Distributed Lock

```typescript
// orders.controller.ts
import { WithLock } from '../cache/decorators/with-lock.decorator';

@Post()
@WithLock('order:create', 10000) // 10-second lock
async createOrder(@Body() dto: CreateOrderDto) {
  // Only one request per user can execute at a time
  return await this.ordersService.createOrder(dto);
}
```

---

## 🧪 Testing

### Unit Testing Cache Service

```typescript
// cache.service.spec.ts
import { Test } from '@nestjs/testing';
import { CacheService } from './cache.service';
import { ConfigService } from '@nestjs/config';

describe('CacheService', () => {
  let service: CacheService;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        CacheService,
        {
          provide: ConfigService,
          useValue: {
            get: jest.fn().mockReturnValue('false'), // In-memory mode
          },
        },
      ],
    }).compile();

    service = module.get<CacheService>(CacheService);
    await service.onModuleInit();
  });

  it('should store and retrieve values', async () => {
    await service.set('test:key', { value: 123 }, 60);
    const result = await service.get('test:key');
    expect(result).toEqual({ value: 123 });
  });

  it('should respect TTL expiration', async () => {
    await service.set('test:ttl', 'value', 1); // 1 second TTL
    await new Promise(resolve => setTimeout(resolve, 1100));
    const result = await service.get('test:ttl');
    expect(result).toBeUndefined();
  });

  it('should acquire and release locks', async () => {
    const token = await service.acquireLock('test:lock', 5000);
    expect(token).toBeTruthy();

    // Second acquire should fail
    const token2 = await service.acquireLock('test:lock', 5000);
    expect(token2).toBeNull();

    // Release and acquire again
    await service.releaseLock('test:lock', token!);
    const token3 = await service.acquireLock('test:lock', 5000);
    expect(token3).toBeTruthy();
  });
});
```

### Testing Rate Limit Decorator

```typescript
// rate-limit.guard.spec.ts
import { Test } from '@nestjs/testing';
import { RateLimitGuard } from './rate-limit.guard';
import { CacheService } from '../cache.service';
import { Reflector } from '@nestjs/core';

describe('RateLimitGuard', () => {
  let guard: RateLimitGuard;
  let cacheService: CacheService;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        RateLimitGuard,
        {
          provide: CacheService,
          useValue: {
            checkRateLimit: jest.fn().mockResolvedValue(true),
          },
        },
        {
          provide: Reflector,
          useValue: {
            get: jest.fn().mockReturnValue({ limit: 10, ttl: 60 }),
          },
        },
      ],
    }).compile();

    guard = module.get<RateLimitGuard>(RateLimitGuard);
    cacheService = module.get<CacheService>(CacheService);
  });

  it('should allow requests within limit', async () => {
    const context = createMockExecutionContext();
    const result = await guard.canActivate(context);
    expect(result).toBe(true);
  });

  it('should block requests exceeding limit', async () => {
    jest.spyOn(cacheService, 'checkRateLimit').mockResolvedValue(false);
    const context = createMockExecutionContext();
    
    await expect(guard.canActivate(context)).rejects.toThrow();
  });
});
```

---

## 🔧 Troubleshooting

### Issue: Redis Connection Fails

**Symptoms**:
```
❌ Redis connection error: NOAUTH Authentication required
```

**Solutions**:
1. Check `VALKEY_USERNAME` and `VALKEY_PASSWORD` are set
2. Verify ACL permissions on Redis
3. Check network connectivity (Security Groups, VPC)
4. Verify `VALKEY_TLS=true` for AWS ElastiCache

### Issue: Cluster Mode Error

**Symptoms**:
```
❌ Failed to refresh slots cache
```

**Solution**:
Set `VALKEY_CLUSTER=false` for AWS ElastiCache Serverless (standalone mode)

### Issue: Lock Deadlock

**Symptoms**:
- Operations never complete
- 409 Conflict errors persist

**Solutions**:
1. Check lock TTL is appropriate (not too long)
2. Ensure locks are always released (use try/finally)
3. Use admin endpoint: `POST /cache/clear-locks`

### Issue: Rate Limit False Positives

**Symptoms**:
- Users blocked unexpectedly
- 429 errors on first request

**Solutions**:
1. Check clock skew between servers
2. Verify rate limit configuration (limit/ttl)
3. Check if multiple instances share Redis
4. Clear rate limit: `DELETE /cache/keys` (admin)

### Issue: Cache Not Invalidating

**Symptoms**:
- Stale data returned after updates
- Changes not reflected immediately

**Solutions**:
1. Verify pub/sub subscription is active
2. Check Redis connection on all instances
3. Use pattern deletion: `delPattern('feed:*')`
4. Check if cache key format changed

---

## 📈 Performance Optimization

### Optimization 1: Use Pattern-Based Cache Keys

**Good**:
```typescript
`feed:user:${userId}:page:${page}` // Easy to invalidate feed:user:123:*
```

**Bad**:
```typescript
`${userId}-feed-${page}` // Hard to pattern match
```

### Optimization 2: Set Appropriate TTLs

| Data Type | Recommended TTL | Reason |
|-----------|-----------------|--------|
| User profiles | 5-10 minutes | Updated infrequently |
| Feed pages | 3-5 minutes | Frequent new content |
| Search results | 10-15 minutes | Query results stable |
| Static content | 1 hour | Rarely changes |
| Real-time data | 30-60 seconds | Needs freshness |

### Optimization 3: Use Pub/Sub for Invalidation

Instead of:
```typescript
// Polling for changes (BAD)
setInterval(async () => {
  await this.cacheService.del('feed:user:123:page:1');
}, 60000);
```

Use:
```typescript
// Event-driven invalidation (GOOD)
await this.cacheService.publish('invalidate:feed', { userId: 123 });
```

### Optimization 4: Batch Operations

Instead of:
```typescript
// Multiple round trips (BAD)
for (const userId of userIds) {
  await this.cacheService.sadd(`followers:${chefId}`, userId);
}
```

Use:
```typescript
// Single operation (GOOD)
await this.cacheService.sadd(`followers:${chefId}`, ...userIds);
```

---

## 🔒 Security Best Practices

### 1. Never Cache Sensitive Data
❌ Don't cache passwords, tokens, credit card numbers  
✅ Cache public profiles, feed pages, search results

### 2. Use User-Scoped Keys
❌ `feed:page:1` (shared across all users)  
✅ `feed:user:123:page:1` (user-specific)

### 3. Validate Lock Ownership
✅ Always use token-based lock release  
✅ Use short TTLs (5-30 seconds)  
✅ Auto-release on response finish

### 4. Rate Limit by Multiple Dimensions
✅ Check both user ID and IP address  
✅ Use tiered limits (Bronze/Silver/Gold/Diamond)  
✅ Log and escalate repeated violations

---

## 📊 Monitoring & Observability

### Key Metrics to Track

1. **Cache Hit Rate**: `cache_hits / (cache_hits + cache_misses)`
   - Target: >60%
   - Alert if: <40%

2. **Redis Availability**: Connection success rate
   - Target: >99.9%
   - Alert if: <99%

3. **Lock Contention**: Blocked requests per second
   - Target: <1% of total requests
   - Alert if: >5%

4. **Rate Limit Violations**: 429 responses per minute
   - Target: <10/minute
   - Alert if: >50/minute

### Logging Best Practices

```typescript
// Log cache operations at DEBUG level
this.logger.debug(`Cache hit: ${key}`);
this.logger.debug(`Cache miss: ${key}`);

// Log failures at ERROR level
this.logger.error(`Redis GET failed for key: ${key}`, error);

// Log security events at WARN level
this.logger.warn(`Rate limit exceeded for ${userId}`);
this.logger.warn(`Failed to acquire lock: ${lockKey}`);
```

---

## 🚀 Deployment

### Production Checklist

- [ ] `VALKEY_ENABLED=true`
- [ ] `VALKEY_HOST` points to production ElastiCache endpoint
- [ ] `VALKEY_USERNAME` and `VALKEY_PASSWORD` configured
- [ ] `VALKEY_TLS=true`
- [ ] `VALKEY_CLUSTER=false` (for ElastiCache Serverless)
- [ ] Security Groups allow traffic from app servers
- [ ] Redis version: 7.x or Valkey 7.x
- [ ] Monitoring alerts configured
- [ ] Backup/restore procedures documented

### Zero-Downtime Deployment

1. **Step 1**: Deploy new code with backward-compatible cache keys
2. **Step 2**: Warm cache with new key format
3. **Step 3**: Monitor cache hit rates
4. **Step 4**: Clean up old cache keys (pattern deletion)

---

## 📚 Additional Resources

- [ioredis Documentation](https://github.com/redis/ioredis)
- [AWS ElastiCache Serverless](https://aws.amazon.com/elasticache/serverless/)
- [Redis Commands Reference](https://redis.io/commands/)
- [Valkey Documentation](https://valkey.io/docs/)

---

**Document Version**: 1.0  
**Last Updated**: 2026-02-23  
**Maintainer**: Backend Team  
**Review Cycle**: Quarterly
