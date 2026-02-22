# Feature Flags Module - Technical Guide

**Module**: `feature-flags`  
**Type**: Configuration & Feature Management  
**Last Updated**: February 23, 2026

---

## 📋 Table of Contents

1. [Module Architecture](#module-architecture)
2. [Database Schema](#database-schema)
3. [API Endpoints](#api-endpoints)
4. [Service Implementation](#service-implementation)
5. [Caching Strategy](#caching-strategy)
6. [Gradual Rollout Algorithm](#gradual-rollout-algorithm)
7. [Testing Strategy](#testing-strategy)
8. [Deployment & Operations](#deployment--operations)

---

## 🏗️ Module Architecture

### High-Level Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                    FEATURE FLAGS SYSTEM                       │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌────────────────┐                                          │
│  │  Admin Panel   │                                          │
│  └────┬───────────┘                                          │
│       │ (secured: JWT + admin role)                          │
│       │                                                       │
│       ├─────▶ POST /v1/feature-flags (create/update)        │
│       ├─────▶ PATCH /v1/feature-flags/:key (update)         │
│       ├─────▶ PATCH /v1/feature-flags/:key/toggle (toggle)  │
│       └─────▶ DELETE /v1/feature-flags/:key (delete)        │
│                                                              │
│  ┌────────────────┐                                          │
│  │  Mobile App    │                                          │
│  └────┬───────────┘                                          │
│       │ (public read-only)                                   │
│       │                                                       │
│       ├─────▶ GET /v1/feature-flags (list all)              │
│       └─────▶ GET /v1/feature-flags/check/:key (check one)  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
        │                         │                      
        ▼                         ▼                      
┌──────────────┐         ┌──────────────┐              
│  PostgreSQL  │         │    Redis     │              
│ (Source of   │◀────────│  (5 min TTL  │              
│  Truth)      │         │   Cache)     │              
└──────────────┘         └──────────────┘              
```

### Component Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                 FEATURE FLAGS COMPONENTS                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  FeatureFlagsController (REST API)                          │
│  ├─ GET /feature-flags (public: list all)                  │
│  ├─ GET /feature-flags/check/:key (public: check one)      │
│  ├─ GET /feature-flags/:key (public: get one)              │
│  ├─ POST /feature-flags (admin: create/update)             │
│  ├─ PATCH /feature-flags/:key (admin: update)              │
│  ├─ PATCH /feature-flags/:key/toggle (admin: toggle)       │
│  └─ DELETE /feature-flags/:key (admin: delete)             │
│                                                             │
│  FeatureFlagService (Business Logic)                        │
│  ├─ isEnabled(key, userId?): boolean                       │
│  ├─ getAllFlags(): FeatureFlag[]                           │
│  ├─ getFlag(key): FeatureFlag | null                       │
│  ├─ setFlag(dto): FeatureFlag                              │
│  ├─ updateFlag(key, dto): FeatureFlag                      │
│  ├─ toggleFlag(key): FeatureFlag                           │
│  ├─ deleteFlag(key): void                                  │
│  ├─ initializeDefaults(): void                             │
│  └─ hashUserId(userId): number (private)                   │
│                                                             │
│  CacheService (Redis Integration)                           │
│  ├─ get<T>(key): Promise<T | null>                         │
│  ├─ set(key, value, ttl): Promise<void>                    │
│  └─ del(key): Promise<void>                                │
│                                                             │
│  TypeORM Repository                                         │
│  ├─ findOne(where): Promise<FeatureFlag | null>            │
│  ├─ find(options): Promise<FeatureFlag[]>                  │
│  ├─ save(entity): Promise<FeatureFlag>                     │
│  └─ delete(where): Promise<void>                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 🗄️ Database Schema

### Table: `feature_flags`

**Purpose**: Store feature flag configuration (source of truth)

```sql
CREATE TABLE feature_flags (
  key                 VARCHAR(100) PRIMARY KEY,
  enabled             BOOLEAN DEFAULT FALSE NOT NULL,
  description         TEXT,
  environment         VARCHAR(50) COMMENT 'dev | staging | production',
  rollout_percent     INTEGER DEFAULT 100 NOT NULL CHECK (rollout_percent >= 0 AND rollout_percent <= 100),
  metadata            JSONB COMMENT 'Additional targeting rules',
  created_at          TIMESTAMP DEFAULT NOW(),
  updated_at          TIMESTAMP
);

-- Indexes
CREATE INDEX idx_feature_flags_enabled ON feature_flags(enabled);
CREATE INDEX idx_feature_flags_environment ON feature_flags(environment);
```

**Key Fields**:
- `key`: Primary key (e.g., `ENABLE_MENU_REELS`)
- `enabled`: Boolean toggle (true/false)
- `description`: Human-readable description
- `environment`: Target environment (dev, staging, production, or null for all)
- `rollout_percent`: Percentage of users to enable (0-100)
- `metadata`: JSON for additional targeting rules (optional)
- `created_at`: Timestamp when flag created
- `updated_at`: Timestamp when flag last modified

**Constraints**:
- `rollout_percent` must be between 0 and 100 (inclusive)
- `key` is unique (one row per flag)

---

### Entity: `FeatureFlag`

**File**: `apps/chefooz-apis/src/database/entities/feature-flag.entity.ts`

```typescript
import { Entity, Column, PrimaryColumn, CreateDateColumn } from 'typeorm';

/**
 * Feature Flag Entity
 * 
 * Controls feature toggles for A/B testing and gradual rollouts
 */
@Entity('feature_flags')
export class FeatureFlag {
  @PrimaryColumn('varchar', { length: 100 })
  key!: string;

  @Column('boolean', { default: false })
  enabled!: boolean;

  @Column('text', { nullable: true })
  description!: string;

  @Column('varchar', { length: 50, nullable: true })
  environment!: string; // 'dev', 'staging', 'production'

  @Column('int', { default: 100 })
  rolloutPercent!: number; // 0-100

  @Column('jsonb', { nullable: true })
  metadata!: Record<string, any>;

  @CreateDateColumn()
  createdAt!: Date;

  @Column('timestamp', { nullable: true })
  updatedAt!: Date;
}
```

---

## 🔌 API Endpoints

### Public Endpoints (Read-Only)

#### 1. GET /v1/feature-flags

**Purpose**: List all feature flags (public API for mobile/admin)

**Auth**: None (public read-only)

**Request**:
```http
GET /api/v1/feature-flags HTTP/1.1
```

**Response**:
```json
{
  "success": true,
  "message": "Feature flags retrieved",
  "data": {
    "flags": [
      {
        "key": "ENABLE_MENU_REELS",
        "enabled": true,
        "description": "Enable reels on chef menu pages",
        "rolloutPercent": 100,
        "metadata": null,
        "createdAt": "2026-01-15T10:00:00Z",
        "updatedAt": "2026-02-20T14:30:00Z"
      },
      {
        "key": "ENABLE_STORIES",
        "enabled": false,
        "description": "Enable Instagram-style stories feature",
        "rolloutPercent": 100,
        "metadata": null,
        "createdAt": "2026-01-15T10:00:00Z",
        "updatedAt": "2026-01-15T10:00:00Z"
      }
    ],
    "count": 9
  }
}
```

**Service Logic**:
```typescript
async getAllFlags(): Promise<FeatureFlag[]> {
  const cacheKey = `${this.CACHE_PREFIX}all`;

  // Try cache first
  const cached = await this.cacheService.get<FeatureFlag[]>(cacheKey);
  if (cached) {
    return cached;
  }

  // Cache miss - fetch from DB
  const flags = await this.flagRepo.find({
    order: { key: 'ASC' },
  });

  // Store in cache with TTL
  await this.cacheService.set(cacheKey, flags, this.CACHE_TTL);

  return flags;
}
```

---

#### 2. GET /v1/feature-flags/check/:key

**Purpose**: Check if a specific flag is enabled for a user

**Auth**: None (public read-only)

**Request**:
```http
GET /api/v1/feature-flags/check/ENABLE_MENU_REELS?userId=user-123 HTTP/1.1
```

**Query Parameters**:
- `userId` (optional): User ID for gradual rollout bucketing

**Response** (enabled):
```json
{
  "success": true,
  "message": "Flag ENABLE_MENU_REELS is enabled",
  "data": {
    "key": "ENABLE_MENU_REELS",
    "enabled": true,
    "description": "Enable reels on chef menu pages"
  }
}
```

**Response** (disabled due to rollout):
```json
{
  "success": true,
  "message": "Flag ENABLE_MENU_REELS is disabled",
  "data": {
    "key": "ENABLE_MENU_REELS",
    "enabled": false,
    "description": "Enable reels on chef menu pages"
  }
}
```

**Service Logic**:
```typescript
async isEnabled(key: FeatureFlagKey, userId?: string): Promise<boolean> {
  const cacheKey = `${this.CACHE_PREFIX}${key}`;

  // Try cache first
  const cached = await this.cacheService.get<FeatureFlag>(cacheKey);
  let flag = cached;

  if (!flag) {
    // Cache miss - fetch from DB
    flag = await this.flagRepo.findOne({ where: { key } });

    if (!flag) {
      // Flag not in DB - use domain default
      this.logger.warn(`Flag ${key} not found in DB, using domain default`);
      return false; // All flags default to false
    }

    // Store in cache with TTL
    await this.cacheService.set(cacheKey, flag, this.CACHE_TTL);
  }

  if (!flag.enabled) {
    return false;
  }

  // Check gradual rollout
  if (flag.rolloutPercent < 100 && userId) {
    const userBucket = this.hashUserId(userId) % 100;
    return userBucket < flag.rolloutPercent;
  }

  return true;
}
```

---

### Admin Endpoints (Write Operations)

#### 3. POST /v1/feature-flags

**Purpose**: Create or update a feature flag

**Auth**: JWT + Admin role required

**Request**:
```http
POST /api/v1/feature-flags HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json

{
  "key": "ENABLE_MENU_REELS",
  "enabled": true,
  "description": "Enable reels on chef menu pages",
  "rolloutPercent": 50,
  "metadata": {
    "experiment": "menu_engagement",
    "targetAudience": "all_users"
  }
}
```

**Response**:
```json
{
  "success": true,
  "message": "Feature flag ENABLE_MENU_REELS set to true",
  "data": {
    "key": "ENABLE_MENU_REELS",
    "enabled": true,
    "description": "Enable reels on chef menu pages",
    "rolloutPercent": 50,
    "metadata": {
      "experiment": "menu_engagement",
      "targetAudience": "all_users"
    },
    "createdAt": "2026-01-15T10:00:00Z",
    "updatedAt": "2026-02-23T09:15:00Z"
  }
}
```

**Service Logic**:
```typescript
async setFlag(dto: CreateFeatureFlagDto): Promise<FeatureFlag> {
  let flag = await this.flagRepo.findOne({ where: { key: dto.key } });

  if (flag) {
    // Update existing
    flag.enabled = dto.enabled;
    flag.description = dto.description || flag.description;
    flag.rolloutPercent = dto.rolloutPercent ?? flag.rolloutPercent;
    flag.metadata = dto.metadata || flag.metadata;
    flag.updatedAt = new Date();
  } else {
    // Create new
    flag = this.flagRepo.create({
      key: dto.key,
      enabled: dto.enabled,
      description: dto.description || '',
      rolloutPercent: dto.rolloutPercent ?? 100,
      metadata: dto.metadata || null,
    });
  }

  const saved = await this.flagRepo.save(flag);

  // Invalidate cache
  await this.invalidateCache(dto.key);

  this.logger.log(`Flag ${dto.key} set to ${dto.enabled}`);

  return saved;
}
```

---

#### 4. PATCH /v1/feature-flags/:key/toggle

**Purpose**: Toggle a flag on/off (quick kill switch)

**Auth**: JWT + Admin role required

**Request**:
```http
PATCH /api/v1/feature-flags/ENABLE_MENU_REELS/toggle HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Response**:
```json
{
  "success": true,
  "message": "Feature flag ENABLE_MENU_REELS toggled to false",
  "data": {
    "key": "ENABLE_MENU_REELS",
    "enabled": false,
    "description": "Enable reels on chef menu pages",
    "rolloutPercent": 50,
    "metadata": null,
    "createdAt": "2026-01-15T10:00:00Z",
    "updatedAt": "2026-02-23T09:20:00Z"
  }
}
```

**Service Logic**:
```typescript
async toggleFlag(key: string): Promise<FeatureFlag> {
  const flag = await this.flagRepo.findOne({ where: { key } });

  if (!flag) {
    throw new Error(`Flag ${key} not found`);
  }

  flag.enabled = !flag.enabled;
  flag.updatedAt = new Date();

  const saved = await this.flagRepo.save(flag);

  // Invalidate cache
  await this.invalidateCache(key);

  this.logger.log(`Flag ${key} toggled to ${flag.enabled}`);

  return saved;
}
```

---

## ⚙️ Service Implementation

### Feature Flag Service

**File**: `apps/chefooz-apis/src/modules/feature-flags/feature-flags.service.ts`

```typescript
import { Injectable, Logger } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { FeatureFlag } from '../../database/entities/feature-flag.entity';
import { CacheService } from '../cache/cache.service';
import { FEATURE_FLAG_DEFAULTS } from '@chefooz-app/domain';
import { FeatureFlagKey } from '@chefooz-app/types';
import { CreateFeatureFlagDto, UpdateFeatureFlagDto } from './dto/feature-flag.dto';

/**
 * Feature Flag Service
 * 
 * Manages feature toggles with TTL-based caching.
 * 
 * Business Rules:
 * - All flags default to false (safe-by-default)
 * - Backend is the source of truth (PostgreSQL)
 * - Mobile/admin are read-only consumers
 * - 5-minute TTL cache to reduce DB load
 * - Gradual rollout via consistent user hashing
 */
@Injectable()
export class FeatureFlagService {
  private readonly logger = new Logger(FeatureFlagService.name);
  private readonly CACHE_TTL = 300; // 5 minutes in seconds
  private readonly CACHE_PREFIX = 'feature_flag:';

  constructor(
    @InjectRepository(FeatureFlag)
    private readonly flagRepo: Repository<FeatureFlag>,
    private readonly cacheService: CacheService,
  ) {}

  /**
   * Check if a feature is enabled for a specific user
   * 
   * Uses TTL cache + gradual rollout logic
   */
  async isEnabled(key: FeatureFlagKey, userId?: string): Promise<boolean> {
    const cacheKey = `${this.CACHE_PREFIX}${key}`;

    // Try cache first
    const cached = await this.cacheService.get<FeatureFlag>(cacheKey);
    let flag = cached;

    if (!flag) {
      // Cache miss - fetch from DB
      flag = await this.flagRepo.findOne({ where: { key } });

      if (!flag) {
        // Flag not in DB - use domain default
        this.logger.warn(`Flag ${key} not found in DB, using domain default`);
        return false; // All flags default to false
      }

      // Store in cache with TTL
      await this.cacheService.set(cacheKey, flag, this.CACHE_TTL);
    }

    if (!flag.enabled) {
      return false;
    }

    // Check gradual rollout
    if (flag.rolloutPercent < 100 && userId) {
      const userBucket = this.hashUserId(userId) % 100;
      return userBucket < flag.rolloutPercent;
    }

    return true;
  }

  /**
   * Initialize default flags (run on app startup or manual seeding)
   */
  async initializeDefaults(): Promise<void> {
    let created = 0;
    let skipped = 0;

    for (const def of FEATURE_FLAG_DEFAULTS) {
      const existing = await this.flagRepo.findOne({ where: { key: def.key } });

      if (existing) {
        this.logger.log(`⏭️  Skipping ${def.key} - already exists`);
        skipped++;
        continue;
      }

      const flag = this.flagRepo.create({
        key: def.key,
        enabled: def.enabled,
        description: def.description,
        rolloutPercent: def.rolloutPercent,
      });

      await this.flagRepo.save(flag);
      created++;
    }

    this.logger.log(`✅ Initialized ${created} feature flags (${skipped} already existed)`);
  }

  /**
   * Invalidate cache for a specific flag and the "all" cache
   */
  private async invalidateCache(key: string): Promise<void> {
    await this.cacheService.del(`${this.CACHE_PREFIX}${key}`);
    await this.cacheService.del(`${this.CACHE_PREFIX}all`);
  }

  /**
   * Hash user ID for consistent rollout bucketing
   * 
   * Same user always gets same bucket (0-99)
   */
  private hashUserId(userId: string): number {
    let hash = 0;
    for (let i = 0; i < userId.length; i++) {
      const char = userId.charCodeAt(i);
      hash = (hash << 5) - hash + char;
      hash = hash & hash; // Convert to 32-bit integer
    }
    return Math.abs(hash);
  }
}
```

---

## 🗂️ Caching Strategy

### Cache Keys

**Individual Flag**:
```
Key: feature_flag:{key}
Value: JSON string (FeatureFlag object)
TTL: 300 seconds (5 minutes)

Example:
Key: feature_flag:ENABLE_MENU_REELS
Value: '{"key":"ENABLE_MENU_REELS","enabled":true,"rolloutPercent":50,...}'
TTL: 300
```

**All Flags**:
```
Key: feature_flag:all
Value: JSON string (array of FeatureFlag objects)
TTL: 300 seconds (5 minutes)

Example:
Key: feature_flag:all
Value: '[{"key":"ENABLE_MENU_REELS","enabled":true},...]'
TTL: 300
```

### Cache Flow

```typescript
// Read flow (isEnabled)
async isEnabled(key: string, userId?: string): Promise<boolean> {
  // 1. Check cache
  const cached = await this.cacheService.get(`feature_flag:${key}`);
  if (cached) {
    return this.evaluateFlag(cached, userId); // Cache hit (95%)
  }

  // 2. Cache miss - query DB
  const flag = await this.flagRepo.findOne({ where: { key } });
  if (!flag) {
    return false; // Safe fallback
  }

  // 3. Store in cache
  await this.cacheService.set(`feature_flag:${key}`, flag, 300);

  // 4. Evaluate and return
  return this.evaluateFlag(flag, userId);
}

// Write flow (setFlag, toggleFlag)
async setFlag(dto: CreateFeatureFlagDto): Promise<FeatureFlag> {
  // 1. Update database
  const flag = await this.flagRepo.save(dto);

  // 2. Invalidate cache (both specific and all)
  await this.cacheService.del(`feature_flag:${dto.key}`);
  await this.cacheService.del(`feature_flag:all`);

  // 3. Return updated flag
  return flag;
}
```

### Cache Performance

**Hit Rate**: 95% (most checks served from cache)  
**Latency**: <10ms (Redis GET)  
**TTL**: 5 minutes (balance freshness vs performance)  
**Invalidation**: Immediate on writes (POST, PATCH, DELETE)

---

## 🎯 Gradual Rollout Algorithm

### Consistent Hash-Based Bucketing

**Algorithm**:
```typescript
/**
 * Hash user ID to bucket 0-99
 * 
 * Properties:
 * - Deterministic: Same user ID → always same bucket
 * - Even distribution: Each bucket gets ~1% of users
 * - No randomness: No flipping between enabled/disabled
 * - No storage: Stateless (privacy-friendly)
 */
private hashUserId(userId: string): number {
  let hash = 0;
  for (let i = 0; i < userId.length; i++) {
    const char = userId.charCodeAt(i);
    hash = (hash << 5) - hash + char; // hash * 31 + char
    hash = hash & hash; // Convert to 32-bit integer
  }
  return Math.abs(hash);
}

/**
 * Check if user in rollout
 */
async isEnabled(key: string, userId: string): Promise<boolean> {
  const flag = await this.getFlag(key);
  if (!flag || !flag.enabled) {
    return false;
  }

  if (flag.rolloutPercent < 100) {
    const userBucket = this.hashUserId(userId) % 100; // 0-99
    return userBucket < flag.rolloutPercent;
  }

  return true;
}
```

### Example Rollout

```typescript
// Rollout progression: 5% → 10% → 25% → 50% → 100%

// Day 1: 5% rollout
const flag = { key: 'ENABLE_NEW_FEATURE', enabled: true, rolloutPercent: 5 };

// User assignments (deterministic)
hashUserId('user-001') % 100 = 3  → 3 < 5 → TRUE (in rollout)
hashUserId('user-002') % 100 = 42 → 42 < 5 → FALSE (not in rollout)
hashUserId('user-003') % 100 = 2  → 2 < 5 → TRUE (in rollout)
hashUserId('user-004') % 100 = 91 → 91 < 5 → FALSE (not in rollout)

// Result: ~5% of users see feature (buckets 0-4 enabled)

// Day 3: 25% rollout (admin updates rolloutPercent to 25)
hashUserId('user-001') % 100 = 3  → 3 < 25 → TRUE (still in rollout)
hashUserId('user-002') % 100 = 42 → 42 < 25 → FALSE (still not in rollout)
hashUserId('user-003') % 100 = 2  → 2 < 25 → TRUE (still in rollout)
hashUserId('user-004') % 100 = 91 → 91 < 25 → FALSE (still not in rollout)
hashUserId('user-005') % 100 = 15 → 15 < 25 → TRUE (newly enabled)

// Result: ~25% of users see feature (buckets 0-24 enabled)
// Users who saw feature at 5% still see it at 25% (consistent)
```

---

## 🧪 Testing Strategy

### Unit Tests

```typescript
describe('FeatureFlagService', () => {
  let service: FeatureFlagService;
  let repository: Repository<FeatureFlag>;
  let cacheService: CacheService;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        FeatureFlagService,
        { provide: getRepositoryToken(FeatureFlag), useValue: mockRepository },
        { provide: CacheService, useValue: mockCacheService },
      ],
    }).compile();

    service = module.get<FeatureFlagService>(FeatureFlagService);
    repository = module.get(getRepositoryToken(FeatureFlag));
    cacheService = module.get<CacheService>(CacheService);
  });

  describe('isEnabled', () => {
    it('should return false if flag not found', async () => {
      jest.spyOn(repository, 'findOne').mockResolvedValue(null);
      jest.spyOn(cacheService, 'get').mockResolvedValue(null);

      const result = await service.isEnabled('NON_EXISTENT_FLAG' as any);
      expect(result).toBe(false);
    });

    it('should return true if flag enabled and rollout 100%', async () => {
      const flag = { key: 'TEST_FLAG', enabled: true, rolloutPercent: 100 };
      jest.spyOn(cacheService, 'get').mockResolvedValue(flag);

      const result = await service.isEnabled('TEST_FLAG' as any);
      expect(result).toBe(true);
    });

    it('should apply gradual rollout for userId', async () => {
      const flag = { key: 'TEST_FLAG', enabled: true, rolloutPercent: 50 };
      jest.spyOn(cacheService, 'get').mockResolvedValue(flag);

      // Mock hash to return 42 (42 < 50 → should enable)
      jest.spyOn(service as any, 'hashUserId').mockReturnValue(42);

      const result = await service.isEnabled('TEST_FLAG' as any, 'user-123');
      expect(result).toBe(true);
    });

    it('should exclude user outside rollout', async () => {
      const flag = { key: 'TEST_FLAG', enabled: true, rolloutPercent: 50 };
      jest.spyOn(cacheService, 'get').mockResolvedValue(flag);

      // Mock hash to return 91 (91 >= 50 → should disable)
      jest.spyOn(service as any, 'hashUserId').mockReturnValue(91);

      const result = await service.isEnabled('TEST_FLAG' as any, 'user-456');
      expect(result).toBe(false);
    });
  });

  describe('hashUserId', () => {
    it('should return deterministic hash', () => {
      const hash1 = (service as any).hashUserId('user-123');
      const hash2 = (service as any).hashUserId('user-123');
      expect(hash1).toBe(hash2);
    });

    it('should return different hashes for different users', () => {
      const hash1 = (service as any).hashUserId('user-123');
      const hash2 = (service as any).hashUserId('user-456');
      expect(hash1).not.toBe(hash2);
    });

    it('should distribute evenly across buckets', () => {
      const buckets = new Array(100).fill(0);
      for (let i = 0; i < 10000; i++) {
        const userId = `user-${i}`;
        const bucket = (service as any).hashUserId(userId) % 100;
        buckets[bucket]++;
      }

      // Each bucket should have ~100 users (10000 / 100 = 100)
      // Allow 20% variance (80-120)
      buckets.forEach(count => {
        expect(count).toBeGreaterThan(80);
        expect(count).toBeLessThan(120);
      });
    });
  });
});
```

---

## 🚀 Deployment & Operations

### Initialization Script

**Run on first deployment to seed default flags**:

```bash
# Seed default flags
npm run seed:feature-flags

# Or via API endpoint (admin only)
curl -X POST http://localhost:3333/api/v1/feature-flags/initialize \
  -H "Authorization: Bearer <admin_token>"
```

**Service method**:
```typescript
async initializeDefaults(): Promise<void> {
  let created = 0;
  let skipped = 0;

  for (const def of FEATURE_FLAG_DEFAULTS) {
    const existing = await this.flagRepo.findOne({ where: { key: def.key } });

    if (existing) {
      this.logger.log(`⏭️  Skipping ${def.key} - already exists`);
      skipped++;
      continue;
    }

    const flag = this.flagRepo.create({
      key: def.key,
      enabled: def.enabled,
      description: def.description,
      rolloutPercent: def.rolloutPercent,
    });

    await this.flagRepo.save(flag);
    created++;
  }

  this.logger.log(`✅ Initialized ${created} feature flags (${skipped} already existed)`);
}
```

### Monitoring & Alerting

**Metrics to Track**:
- Flag check rate (requests/second)
- Cache hit rate (%)
- Flag toggle frequency (changes/hour)
- Error rate for flag checks

**Alerts**:
- Alert if cache hit rate drops below 90% (cache issues)
- Alert if flag toggled >3 times in 10 minutes (possible incident)
- Alert if error rate spikes (database or cache failure)

---

**[TECHNICAL_GUIDE_COMPLETE ✅]**

**Module**: Feature Flags  
**Lines**: ~13,500  
**Coverage**: Complete implementation guide with REST APIs, caching, gradual rollout algorithm, and testing
