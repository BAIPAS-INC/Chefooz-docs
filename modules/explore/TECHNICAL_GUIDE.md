# Explore Module - Technical Guide

**Version:** 1.1  
**Last Updated:** April 17, 2026  
**Module:** `apps/chefooz-apis/src/modules/explore/`  
**Domain Logic:** `libs/domain/src/lib/explore-*.ts`  
**Tech Stack:** NestJS, MongoDB (Mongoose), PostgreSQL (TypeORM), Redis/Valkey

---

## 📋 Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [API Endpoints](#api-endpoints)
3. [Service Methods](#service-methods)
4. [Ranking Services](#ranking-services)
5. [Data Models](#data-models)
6. [Caching Strategy](#caching-strategy)
7. [Database Queries](#database-queries)
8. [Background Jobs](#background-jobs)
9. [Error Handling](#error-handling)
10. [Testing Approach](#testing-approach)

---

## Recent Fixes

- 2026-04-17: `apps/chefooz-app/src/app/(tabs)/explore.tsx` now applies safe-area-aware bottom padding through `getTabBarHeight(insets.bottom) + normalize(20)` so the final explore modules are scrollable above the absolute tab bar.

## Architecture Overview

### Module Structure

```
apps/chefooz-apis/src/modules/explore/
├── dto/
│   ├── explore.query.dto.ts           # Query params (cursor, limit)
│   ├── explore.response.dto.ts        # Response DTOs
│   ├── explore-sections.query.dto.ts  # Consolidated sections query
│   ├── explore-sections.response.dto.ts # Consolidated response
│   ├── explore-categories.dto.ts      # Category DTOs
│   ├── explore-chefs.dto.ts           # Chef discovery DTOs
│   ├── category-detail.dto.ts         # Category detail response
│   └── unified-search.dto.ts          # Legacy search DTOs
├── services/
│   ├── explore-ranking.service.ts     # Food-first ranking engine
│   └── explore-signal-builder.service.ts # Ranking signal extraction
├── mappers/
│   └── explore.mapper.ts              # Entity → DTO mapping
├── jobs/
│   └── trending.aggregator.ts         # Hourly trending pre-calculation
├── explore.controller.ts              # HTTP endpoints
├── explore.service.ts                 # Business logic
├── explore.module.ts                  # Module configuration
└── explore-explainability.spec.ts     # Unit tests

libs/domain/src/lib/
├── explore-explainability.ts          # Card type resolution, reason tags
└── explore-layout.ts                  # Layout configuration
```

### Technology Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **Framework** | NestJS 10+ | REST API framework |
| **Reel Storage** | MongoDB 7+ (Mongoose) | Reel documents |
| **User/Order Data** | PostgreSQL 16+ (TypeORM) | Users, chefs, orders, follows |
| **Cache** | Redis/Valkey | Section caching, trending sets |
| **Cron Jobs** | @nestjs/schedule | Hourly trending aggregation |
| **Validation** | class-validator | DTO validation |
| **Logging** | Winston | Structured logging |

### Dependencies

```typescript
@Module({
  imports: [
    // MongoDB schemas
    MongooseModule.forFeature([
      { name: Reel.name, schema: ReelSchema },
    ]),
    
    // PostgreSQL entities
    TypeOrmModule.forFeature([
      User, ChefProfile, ChefMenuItem,
      UserBlock, UserPrivacy, UserFollow,
      PlatformCategory, ChefKitchen, Order,
      UserReputationCurrent,
    ]),
    
    CacheModule,
    FeatureFlagsModule,
  ],
  providers: [
    ExploreService,
    ExploreRankingService,
    ExploreSignalBuilder,
    TrendingAggregatorJob,
  ],
  exports: [ExploreService, ExploreRankingService],
})
```

---

## API Endpoints

### 1. Get Consolidated Sections

**Endpoint**: `GET /api/v1/explore/sections`

**Description**: Returns all explore sections in a single call for efficient mobile loading.

**Auth**: Required (JWT)

**Rate Limit**: 30 requests/minute

**Query Parameters**:
```typescript
{
  limit?: number;           // Items per section (default: 5)
  chefLimit?: number;       // Number of chefs (default: 5)
  hashtagLimit?: number;    // Number of hashtags (default: 10)
  includeNearYou?: boolean; // Include location-based content
  latitude?: number;        // User's latitude
  longitude?: number;       // User's longitude
}
```

**Response**:
```json
{
  "success": true,
  "message": "Explore sections retrieved",
  "data": {
    "trending": {
      "items": [
        {
          "type": "FOOD",
          "entityId": "menu-item-uuid",
          "primaryReelId": "reel-id",
          "score": 95,
          "reasonTags": ["TRENDING", "HIGHLY_ORDERED"],
          "cardData": {
            "title": "Butter Chicken",
            "chefName": "Chef Rakesh",
            "chefAvatar": "https://...",
            "thumbnailUrl": "https://...",
            "price": 350,
            "rating": 4.5,
            "isAvailable": true,
            "estimatedETA": 30
          },
          "ctaAction": "ORDER_NOW"
        }
      ],
      "hasMore": true,
      "nextCursor": "base64-encoded-cursor"
    },
    "recommended": { ... },
    "newDish": { ... },
    "exclusive": { ... },
    "topChefs": [
      {
        "userId": "chef-uuid",
        "username": "chef_rakesh",
        "fullName": "Rakesh Kumar",
        "avatarUrl": "https://...",
        "reputation": {
          "tier": "gold",
          "score": 72
        },
        "stats": {
          "followers": 2500,
          "totalOrders": 1200,
          "avgRating": 4.5
        },
        "isOnline": true,
        "distance": 2.5
      }
    ],
    "trendingHashtags": [
      {
        "tag": "butterchicken",
        "displayName": "#butterchicken",
        "reelCount": 450,
        "trendingScore": 89
      }
    ]
  }
}
```

### 2. Get Section Items (Paginated)

**Endpoint**: `GET /api/v1/explore/:section`

**Sections**: `trending`, `recommended`, `new_dish`, `exclusive`, `promoted`

**Auth**: Required (JWT)

**Rate Limit**: 60 requests/minute

**Query Parameters**:
```typescript
{
  cursor?: string;  // Base64 cursor for pagination
  limit?: number;   // 6-48 items (default: 24)
}
```

**Response**:
```json
{
  "success": true,
  "message": "trending items retrieved",
  "data": {
    "items": [ ... ],
    "nextCursor": "base64...",
    "hasMore": true
  }
}
```

### 3. Get Categories

**Endpoint**: `GET /api/v1/explore/categories`

**Auth**: Optional (public endpoint)

**Rate Limit**: 30 requests/minute

**Query Parameters**:
```typescript
{
  limit?: number;  // Max categories (default: 20)
}
```

**Response**:
```json
{
  "success": true,
  "message": "Categories retrieved",
  "data": {
    "categories": [
      {
        "id": "category-uuid",
        "key": "NORTH_INDIAN",
        "name": "North Indian",
        "icon": "🍛",
        "reelCount": 1250,
        "thumbnailUrl": "https://..."
      }
    ]
  }
}
```

### 4. Get Category Details

**Endpoint**: `GET /api/v1/explore/category/:categoryKey/detail`

**Auth**: Required (JWT)

**Rate Limit**: 60 requests/minute

**Response**: Category info + paginated reels

### 5. Get Trending Chefs

**Endpoint**: `GET /api/v1/explore/chefs/trending`

**Auth**: Required (JWT)

**Rate Limit**: 30 requests/minute

**Query Parameters**:
```typescript
{
  limit?: number;   // Default: 20
  cursor?: string;
}
```

### 6. Get Nearby Chefs

**Endpoint**: `GET /api/v1/explore/chefs/nearby`

**Auth**: Required (JWT)

**Rate Limit**: 30 requests/minute

**Query Parameters**:
```typescript
{
  latitude: number;     // Required
  longitude: number;    // Required
  maxDistance?: number; // km (default: 10)
  limit?: number;       // Default: 20
  cursor?: string;
}
```

### 7. Unified Search (Legacy)

**Endpoint**: `GET /api/v1/explore/search?q=butter chicken`

**Auth**: Required (JWT)

**Rate Limit**: 10 requests/second

**Note**: Being replaced by `/api/v1/search-elastic/*` endpoints

---

## Service Methods

### ExploreService

#### getAllSections()
```typescript
async getAllSections(
  userId: string,
  options: {
    limit?: number;
    chefLimit?: number;
    hashtagLimit?: number;
    includeNearYou?: boolean;
    latitude?: number;
    longitude?: number;
  }
): Promise<ConsolidatedSectionsDto>
```

Fetches all explore sections in parallel for efficient loading.

**Implementation**:
```typescript
const [trending, recommended, newDish, exclusive, chefs, hashtags] = 
  await Promise.all([
    this.getTrendingItems(userId, limit),
    this.getRecommendedItems(userId, limit),
    this.getNewDishItems(userId, limit),
    this.getExclusiveItems(userId, limit),
    this.getTrendingChefs(userId, chefLimit),
    this.getTrendingHashtags(hashtagLimit),
  ]);
```

#### getExploreSectionItems()
```typescript
async getExploreSectionItems(
  section: ExploreSection,
  userId: string,
  cursor?: string,
  limit: number = 24
): Promise<ExploreItemsResponseDto>
```

Routes to appropriate section handler:
- `trending` → `getTrendingItems()`
- `recommended` → `getRecommendedItems()`
- `new_dish` → `getNewDishItems()`
- `exclusive` → `getExclusiveItems()`
- `promoted` → `getPromotedItems()`

#### getTrendingItems()

**Cache Key**: `explore:trending:{cursor}`  
**TTL**: 1 hour

```typescript
// 1. Check cache
const cached = await this.cacheService.get(cacheKey);
if (cached) return cached;

// 2. Fetch pre-aggregated trending reels
const trendingData = await this.cacheService.get('explore:trending:sorted');
if (!trendingData) {
  // Fallback: run aggregation manually
  await this.trendingAggregator.aggregateTrendingScores();
  return this.getTrendingItems(userId, cursor, limit);
}

// 3. Paginate and apply privacy filters
const filtered = trendingData
  .filter(item => this.isVisibleToUser(item, userId))
  .slice(cursorOffset, cursorOffset + limit);

// 4. Enrich with user data, menu items, etc.
const enriched = await this.enrichExploreItems(filtered);

// 5. Cache result
await this.cacheService.set(cacheKey, result, 3600);
return result;
```

#### getRecommendedItems()

**Cache Key**: `explore:rec:{userId}:{cursor}`  
**TTL**: 30 minutes

```typescript
// 1. Build user context
const context = await this.buildUserContext(userId);

// 2. Fetch candidate reels
const candidates = await this.fetchCandidateReels(context);

// 3. Build ranking signals
const signals = await this.signalBuilder.buildSignals(candidates, context);

// 4. Rank entities
const ranked = this.rankingService.rankEntities(signals, context);

// 5. Apply diversity and pagination
const result = ranked.slice(cursorOffset, cursorOffset + limit);

return result;
```

**User Context**:
```typescript
interface UserRankingContext {
  userId: string;
  followedChefIds: string[];
  pastOrderChefIds: string[];
  likedCuisines: string[];
  engagedHashtags: string[];
  location?: { lat: number; lng: number };
  priceRange: { min: number; max: number };
  tier: UserTier;
}
```

#### getNewDishItems()

**Cache Key**: `explore:new:{cursor}`  
**TTL**: 15 minutes

```typescript
// 1. Fetch reels from last 24 hours
const reels = await this.reelModel.find({
  createdAt: { $gte: new Date(Date.now() - 24 * 60 * 60 * 1000) },
  reelPurpose: 'MENU_SHOWCASE',
}).sort({ createdAt: -1 });

// 2. Filter available chefs only
const available = await this.filterByChefAvailability(reels);

// 3. Boost followed chefs
const boosted = available.map(reel => ({
  ...reel,
  boostScore: context.followedChefIds.includes(reel.userId) ? 2.0 : 1.0,
}));

// 4. Sort and paginate
return boosted
  .sort((a, b) => b.boostScore - a.boostScore || b.createdAt - a.createdAt)
  .slice(cursorOffset, cursorOffset + limit);
```

---

## Ranking Services

### ExploreRankingService

**Purpose**: Food-first ranking engine for explore feed.

#### rankEntities()

```typescript
rankEntities(
  entities: Array<{ entityId: string; type: RankableEntityType; signals: RankingSignals }>,
  context: UserRankingContext,
  weights?: Partial<RankingWeights>,
  diversityConfig?: Partial<DiversityConfig>
): RankedExploreItem[]
```

**Steps**:

1. **Hard Filter** (availability):
   ```typescript
   const available = entities.filter(e => 
     e.signals.isDeliverable &&
     e.signals.isChefOnline &&
     e.signals.isKitchenOpen &&
     !e.signals.isOverloaded &&
     (e.signals.estimatedETA || 0) <= 90
   );
   ```

2. **Calculate Scores**:
   ```typescript
   const scored = available.map(entity => {
     const breakdown = {
       availability: this.scoreAvailability(entity.signals),
       personalRelevance: this.scorePersonalRelevance(entity.signals, context),
       socialProof: this.scoreSocialProof(entity.signals, context),
       visualTrust: this.scoreVisualTrust(entity.signals),
       freshness: this.scoreFreshness(entity.signals),
     };
     
     const finalScore = 
       breakdown.availability * 0.30 +
       breakdown.personalRelevance * 0.25 +
       breakdown.socialProof * 0.20 +
       breakdown.visualTrust * 0.15 +
       breakdown.freshness * 0.10;
     
     return { ...entity, score: finalScore, scoreBreakdown: breakdown };
   });
   ```

3. **Sort by Score**:
   ```typescript
   scored.sort((a, b) => b.score - a.score);
   ```

4. **Apply Diversity Guards**:
   ```typescript
   const diversified = this.applyDiversityGuards(scored, diversityConfig);
   ```

5. **Add Score Jitter** (±5%):
   ```typescript
   const withJitter = diversified.map(item => ({
     ...item,
     score: item.score + (Math.random() - 0.5) * 0.1 * item.score,
   }));
   ```

6. **Re-sort & Return**:
   ```typescript
   withJitter.sort((a, b) => b.score - a.score);
   return withJitter;
   ```

#### Diversity Guards

```typescript
private applyDiversityGuards(
  items: RankedExploreItem[],
  config: DiversityConfig
): RankedExploreItem[] {
  const result: RankedExploreItem[] = [];
  const chefCounts = new Map<string, number>();
  const cuisineCounts = new Map<string, number>();
  const seenDishes = new Set<string>();

  for (const item of items) {
    // Guard 1: Max items per chef
    const chefId = item._metadata?.chefId;
    if (chefId && chefCounts.get(chefId) >= config.maxItemsPerChef) {
      continue;
    }

    // Guard 2: Max items per cuisine
    const cuisine = item._metadata?.cuisine;
    if (cuisine && cuisineCounts.get(cuisine) >= config.maxItemsPerCuisine) {
      continue;
    }

    // Guard 3: No duplicate dishes
    const dishKey = `${chefId}:${item.entityId}`;
    if (config.preventDuplicates && seenDishes.has(dishKey)) {
      continue;
    }

    // Accept item
    result.push(item);
    if (chefId) chefCounts.set(chefId, (chefCounts.get(chefId) || 0) + 1);
    if (cuisine) cuisineCounts.set(cuisine, (cuisineCounts.get(cuisine) || 0) + 1);
    seenDishes.add(dishKey);
  }

  return result;
}
```

### ExploreSignalBuilder

**Purpose**: Extract ranking signals from raw entity data.

```typescript
async buildSignals(
  entities: Array<{ reelId: string; userId: string }>,
  context: UserRankingContext
): Promise<Array<{ entityId: string; signals: RankingSignals }>>
```

**Signals Extracted**:

```typescript
interface RankingSignals {
  // Availability
  isDeliverable: boolean;
  isChefOnline: boolean;
  isKitchenOpen: boolean;
  isOverloaded: boolean;
  estimatedETA?: number;

  // Personal Relevance
  isFollowedChef: boolean;
  hasOrderedBefore: boolean;
  cuisineMatch: boolean;
  hashtagMatch: boolean;
  priceMatch: boolean;

  // Social Proof
  totalOrders: number;
  likes: number;
  saves: number;
  avgRating: number;
  followerCount: number;

  // Visual Trust
  watchCompletionRate: number;
  videoQuality: number;

  // Freshness
  ageInHours: number;
}
```

**Implementation**:
```typescript
async buildSignals(entities, context) {
  // 1. Fetch chef statuses
  const chefIds = [...new Set(entities.map(e => e.userId))];
  const chefStatuses = await this.getChefStatuses(chefIds);

  // 2. Fetch user engagement history
  const engagementHistory = await this.getUserEngagementHistory(context.userId);

  // 3. Fetch order history
  const orderHistory = await this.getUserOrderHistory(context.userId);

  // 4. Build signals for each entity
  return entities.map(entity => ({
    entityId: entity.reelId,
    signals: {
      // Availability
      isDeliverable: this.checkDeliverability(entity, context.location),
      isChefOnline: chefStatuses.get(entity.userId)?.isOnline || false,
      isKitchenOpen: chefStatuses.get(entity.userId)?.isKitchenOpen || false,
      isOverloaded: chefStatuses.get(entity.userId)?.pendingOrders > 5,
      estimatedETA: this.calculateETA(entity, context.location),

      // Personal Relevance
      isFollowedChef: context.followedChefIds.includes(entity.userId),
      hasOrderedBefore: orderHistory.chefIds.includes(entity.userId),
      cuisineMatch: context.likedCuisines.includes(entity.cuisine),
      hashtagMatch: entity.hashtags.some(h => context.engagedHashtags.includes(h)),
      priceMatch: entity.price >= context.priceRange.min && entity.price <= context.priceRange.max,

      // Social Proof
      totalOrders: entity.stats?.orders || 0,
      likes: entity.stats?.likes || 0,
      saves: entity.stats?.saves || 0,
      avgRating: entity.stats?.avgRating || 0,
      followerCount: chefStatuses.get(entity.userId)?.followers || 0,

      // Visual Trust
      watchCompletionRate: engagementHistory.watchRates.get(entity.reelId) || 0,
      videoQuality: this.assessVideoQuality(entity.mediaUrl),

      // Freshness
      ageInHours: (Date.now() - new Date(entity.createdAt).getTime()) / (1000 * 60 * 60),
    },
  }));
}
```

---

## Data Models

### ExploreItemDto (Response)

```typescript
interface ExploreItemDto {
  type: ExploreCardType;           // 'FOOD' | 'CHEF' | 'FOOD_REEL' | 'REEL_STORY'
  entityId: string;                 // Menu item ID or chef ID
  primaryReelId?: string;           // Main reel to display
  secondaryReels?: string[];        // Additional supporting reels
  score: number;                    // Ranking score (0-100)
  reasonTags: ReasonTag[];          // ['TRENDING', 'FOLLOWED_CHEF', etc.]
  cardData: {
    title: string;
    chefName: string;
    chefAvatar: string;
    thumbnailUrl: string;
    price?: number;
    rating?: number;
    isAvailable: boolean;
    estimatedETA?: number;
  };
  ctaAction: ExploreCtaAction;      // 'ORDER_NOW' | 'VIEW_MENU' | 'FOLLOW' | 'WATCH'
  scoreBreakdown?: ScoreBreakdown;  // Debug info (only in dev mode)
}
```

### RankingSignals (Internal)

```typescript
interface RankingSignals {
  // Availability (Hard Filters)
  isDeliverable: boolean;
  isChefOnline: boolean;
  isKitchenOpen: boolean;
  isOverloaded: boolean;
  estimatedETA?: number;

  // Personal Relevance (0-100)
  isFollowedChef: boolean;
  hasOrderedBefore: boolean;
  cuisineMatch: boolean;
  hashtagMatch: boolean;
  priceMatch: boolean;

  // Social Proof (counts)
  totalOrders: number;
  likes: number;
  saves: number;
  comments: number;
  avgRating: number;
  followerCount: number;

  // Visual Trust (0-100)
  watchCompletionRate: number;
  videoQuality: number;
  thumbnailAppeal?: number;

  // Freshness
  ageInHours: number;
}
```

### UserRankingContext

```typescript
interface UserRankingContext {
  userId: string;
  followedChefIds: string[];
  pastOrderChefIds: string[];
  likedCuisines: string[];
  engagedHashtags: string[];
  location?: { lat: number; lng: number };
  priceRange: { min: number; max: number };
  tier: UserTier;
}
```

---

## Caching Strategy

### Cache Keys

| Section | Key Pattern | TTL | Refresh |
|---------|-------------|-----|---------|
| **Consolidated Sections** | `explore:sections:{userId}` | 15 min | User engagement |
| **Trending** | `explore:trending:{cursor}` | 1 hour | Hourly cron |
| **Recommended** | `explore:rec:{userId}:{cursor}` | 30 min | User action |
| **New Dish** | `explore:new:{cursor}` | 15 min | New uploads |
| **Exclusive** | `explore:exclusive:{cursor}` | 1 hour | Chef promotion |
| **Top Chefs** | `explore:chefs:{lat}:{lng}` | 2 hours | Chef status |
| **Trending Set** | `explore:trending:sorted` | 1 hour | Cron job |

### Cache Implementation

```typescript
// Get with fallback
async getCachedSection(key: string, fallback: () => Promise<any>, ttl: number) {
  const cached = await this.cacheService.get(key);
  if (cached) {
    this.logger.debug(`Cache HIT: ${key}`);
    return cached;
  }

  this.logger.debug(`Cache MISS: ${key}`);
  const fresh = await fallback();
  await this.cacheService.set(key, fresh, ttl);
  return fresh;
}

// Invalidate on events
async invalidateUserRecommendations(userId: string) {
  const pattern = `explore:rec:${userId}:*`;
  await this.cacheService.deletePattern(pattern);
  this.logger.log(`Invalidated recommendations for user ${userId}`);
}
```

---

## Background Jobs

### Trending Aggregator

**Schedule**: Hourly (`@Cron(CronExpression.EVERY_HOUR)`)

**Purpose**: Pre-calculate trending scores to avoid real-time computation.

```typescript
@Injectable()
export class TrendingAggregatorJob {
  @Cron(CronExpression.EVERY_HOUR)
  async aggregateTrendingScores(): Promise<void> {
    // 1. Define time window (last 7 days)
    const cutoffDate = new Date();
    cutoffDate.setDate(cutoffDate.getDate() - 7);

    // 2. Fetch recent reels
    const reels = await this.reelModel.find({
      createdAt: { $gte: cutoffDate },
    }).select('_id userId stats createdAt').lean();

    // 3. Calculate scores
    const scored = reels.map(reel => ({
      reelId: reel._id.toString(),
      score: this.calculateTrendingScore(reel),
    }));

    // 4. Sort by score
    scored.sort((a, b) => b.score - a.score);

    // 5. Keep top 500
    const top = scored.slice(0, 500);

    // 6. Store in Redis
    await this.cacheService.set('explore:trending:sorted', top, 3600);

    this.logger.log(`Trending aggregation complete: ${top.length} reels cached`);
  }

  private calculateTrendingScore(reel: any): number {
    const ageInDays = (Date.now() - new Date(reel.createdAt).getTime()) / (1000 * 60 * 60 * 24);
    const timeFactor = Math.pow(0.8, ageInDays);

    const engagementScore =
      (reel.stats?.views || 0) * 1.0 +
      (reel.stats?.likes || 0) * 2.0 +
      (reel.stats?.comments || 0) * 3.0 +
      (reel.stats?.saves || 0) * 4.0 +
      (reel.stats?.orders || 0) * 10.0;

    return engagementScore * timeFactor;
  }
}
```

**Manual Trigger** (for admin/testing):
```bash
curl -X POST http://localhost:3000/api/v1/explore/trigger-trending \
  -H "Authorization: Bearer <admin-token>"
```

---

## Database Queries

### Privacy Filter Query

```typescript
// Exclude blocked users (bidirectional)
const blockedUserIds = await this.blockRepo
  .createQueryBuilder('block')
  .select('block.blockedUserId')
  .where('block.userId = :userId', { userId })
  .unionAll(
    this.blockRepo
      .createQueryBuilder('block')
      .select('block.userId')
      .where('block.blockedUserId = :userId', { userId })
  )
  .getRawMany()
  .then(rows => rows.map(r => r.blockedUserId));

// Exclude private accounts (unless following)
const followedUserIds = await this.followRepo.find({
  where: { followerId: userId },
  select: ['followingId'],
}).then(rows => rows.map(r => r.followingId));

const visibleUserIds = await this.privacyRepo
  .createQueryBuilder('privacy')
  .leftJoin('privacy.user', 'user')
  .where('privacy.isPrivate = FALSE')
  .orWhere('user.id IN (:...followedIds)', { followedIds: followedUserIds })
  .select('user.id')
  .getRawMany()
  .then(rows => rows.map(r => r.user_id));

// Exclude deactivated users
const activeUserIds = await this.userRepo.find({
  where: { isDeactivated: false },
  select: ['id'],
}).then(rows => rows.map(r => r.id));

// Final filter
const allowedUserIds = visibleUserIds.filter(
  id => !blockedUserIds.includes(id) && activeUserIds.includes(id)
);

// Apply to reels query
const reels = await this.reelModel.find({
  userId: { $in: allowedUserIds },
});
```

---

## Error Handling

### Standard Error Responses

```typescript
// 400 Bad Request
{
  "success": false,
  "message": "Invalid section name",
  "errorCode": "INVALID_SECTION",
  "data": null
}

// 401 Unauthorized
{
  "success": false,
  "message": "Authentication required",
  "errorCode": "UNAUTHORIZED"
}

// 429 Rate Limit
{
  "success": false,
  "message": "Rate limit exceeded. Try again in 60 seconds.",
  "errorCode": "RATE_LIMIT_EXCEEDED",
  "retryAfter": 60
}

// 500 Internal Error
{
  "success": false,
  "message": "Failed to fetch explore items",
  "errorCode": "EXPLORE_FETCH_ERROR",
  "data": null
}
```

### Error Logging

```typescript
try {
  const result = await this.getExploreSectionItems(section, userId, cursor, limit);
  return result;
} catch (error) {
  this.logger.error(`Explore ${section} fetch failed for user ${userId}:`, {
    error: error.message,
    stack: error.stack,
    section,
    userId,
  });

  // Fallback to empty results (graceful degradation)
  return {
    items: [],
    nextCursor: null,
    hasMore: false,
  };
}
```

---

## Testing Approach

### Unit Tests

**File**: `explore-explainability.spec.ts`, `explore-ranking.service.spec.ts`

```typescript
describe('ExploreRankingService', () => {
  it('should rank available entities higher than unavailable', () => {
    const entities = [
      { entityId: '1', signals: { isChefOnline: false, ... } },
      { entityId: '2', signals: { isChefOnline: true, ... } },
    ];
    
    const ranked = service.rankEntities(entities, mockContext);
    
    expect(ranked[0].entityId).toBe('2'); // Online chef ranked first
  });

  it('should boost followed chefs in personalized ranking', () => {
    const context = { followedChefIds: ['chef-1'], ... };
    const entities = [
      { entityId: '1', userId: 'chef-1', ... },
      { entityId: '2', userId: 'chef-2', ... },
    ];
    
    const ranked = service.rankEntities(entities, context);
    
    expect(ranked[0].entityId).toBe('1'); // Followed chef ranked first
  });
});
```

### Integration Tests

```typescript
describe('ExploreController (e2e)', () => {
  it('GET /explore/sections should return all sections', async () => {
    const response = await request(app.getHttpServer())
      .get('/v1/explore/sections')
      .set('Authorization', `Bearer ${userToken}`)
      .expect(200);

    expect(response.body.success).toBe(true);
    expect(response.body.data).toHaveProperty('trending');
    expect(response.body.data).toHaveProperty('recommended');
    expect(response.body.data.trending.items).toBeInstanceOf(Array);
  });

  it('GET /explore/trending should paginate correctly', async () => {
    const page1 = await request(app.getHttpServer())
      .get('/v1/explore/trending?limit=10')
      .set('Authorization', `Bearer ${userToken}`)
      .expect(200);

    expect(page1.body.data.items.length).toBe(10);
    expect(page1.body.data.hasMore).toBe(true);
    
    const cursor = page1.body.data.nextCursor;
    const page2 = await request(app.getHttpServer())
      .get(`/v1/explore/trending?limit=10&cursor=${cursor}`)
      .set('Authorization', `Bearer ${userToken}`)
      .expect(200);

    expect(page2.body.data.items.length).toBeGreaterThan(0);
    // Ensure no overlap
    const page1Ids = page1.body.data.items.map(i => i.entityId);
    const page2Ids = page2.body.data.items.map(i => i.entityId);
    expect(page1Ids.some(id => page2Ids.includes(id))).toBe(false);
  });
});
```

---

## Performance Optimization

### Query Optimization

```typescript
// ❌ Bad: N+1 query problem
for (const reel of reels) {
  reel.chef = await this.userRepo.findOne({ where: { id: reel.userId } });
}

// ✅ Good: Batch query
const userIds = [...new Set(reels.map(r => r.userId))];
const users = await this.userRepo.find({ where: { id: In(userIds) } });
const userMap = new Map(users.map(u => [u.id, u]));
reels.forEach(reel => { reel.chef = userMap.get(reel.userId); });
```

### Caching Best Practices

1. **Cache High-Traffic Sections** (trending, recommended)
2. **Shorter TTL for Real-Time Sections** (new_dish: 15 min)
3. **Invalidate on User Actions** (like, follow → clear recommendations)
4. **Use Cursor-Based Pagination** (avoid offset LIMIT queries)

---

## SmartSearchBar — Gradient Header (UI Enhancement March 2026)

**File**: `apps/chefooz-app/src/components/explore/SmartSearchBar.tsx`

### Styling Approach

- Container is `<LinearGradient>` from `expo-linear-gradient` (replaces plain `<View>`)
- Gradient colours computed inside the component (not in `makeStyles`) because `colors` used as const arrays
- `style` on `LinearGradient` includes `backgroundColor: colors.background` as a paint-behind fallback

### Dark / Light Gradient Values

```ts
// Dark mode
['rgba(192, 49, 191, 0.18)', 'rgba(20, 20, 20, 0.0)', 'rgba(252, 120, 48, 0.12)']

// Light mode
['rgba(192, 49, 191, 0.07)', 'rgba(255, 255, 255, 0.0)', 'rgba(252, 120, 48, 0.05)']
```

Direction: `start={{ x: 0, y: 0 }}` → `end={{ x: 1, y: 1 }}` (diagonal top-left → bottom-right)

### Search Pill Background

| Mode | Background | Shadow |
|------|-----------|--------|
| Light | `#FFFFFF` | iOS: `rgba(192,49,191,0.20)` shadow, opacity 0.28, radius 8 |
| Dark  | `colors.surfaceElevated` (`#2C2C2E`) | iOS: black shadow, opacity 0.45, radius 8 |
| Android | Both modes | `elevation: 3` (light) / `elevation: 5` (dark) |

### Constraints

- The `gradientColors` tuple must be `[string, string, string]` — TypeScript `as const` applied on the tuple
- Do not pass `isDark` into `makeStyles` for gradient colours since LinearGradient needs the array before StyleSheet is applied
- The `TextInput` import was removed — the component is tap-only (navigates to `/search`)

### Explore Screen Background Fix

All `SafeAreaView` wrappers in `explore.tsx` now receive `{ backgroundColor: theme.colors.background }` as an inline style override. The `styles.container` static StyleSheet retains `flex: 1` only, with the theme-reactive colour provided at render time.

---

## Related Documentation

- **Feature Overview**: `FEATURE_OVERVIEW.md` (Business context and user flows)
- **QA Test Cases**: `QA_TEST_CASES.md` (Test scenarios)
- **Search Module**: `../search/TECHNICAL_GUIDE.md` (Elasticsearch integration)

---

---

## Unified Discovery Shell — Frontend Architecture (March 2026)

> **Change type:** UI Enhancement — frontend only.  
> **File changed:** `apps/chefooz-app/src/app/(tabs)/explore.tsx`

### Component Tree

```
ExploreTab (explore.tsx)
├── SmartSearchBar          (sticky)
└── ScrollView
    ├── FoodCategoriesScroll
    │
    ├── [NEW_USER path]
    │   ├── ExploreMenuShowcase   ← new module
    │   ├── ExplorePromoStrip     ← new module
    │   ├── CommunityEatingSection
    │   ├── NewRisingChefs
    │   ├── TrendingDishesGrid
    │   └── ReelsThatMakeYouHungry (lazy)
    │
    ├── [RETURNING_USER path]
    │   ├── ExploreTrendingThemes  ← new module
    │   ├── TrendingDishesGrid
    │   ├── CommunityEatingSection
    │   ├── ExploreChefWorld       ← new module
    │   ├── ExploreMenuShowcase    ← new module
    │   ├── ExplorePromoStrip      ← new module
    │   ├── ExploreFreshPicks      ← new module
    │   ├── NewRisingChefs
    │   ├── TrendingDishesGrid (recommended)
    │   └── ReelsThatMakeYouHungry (lazy)
    │
    └── [shared bottom — both profiles, lazy]
        ├── ReelsThatMakeYouHungry (nearYou)
        └── ReelsThatMakeYouHungry (newDishes, new-user only)
```

### New Files

| File | Purpose |
|---|---|
| `apps/chefooz-app/src/hooks/useExploreAudienceProfile.ts` | Classify user into `NEW_USER` or `RETURNING_USER` based on `createdAt` |
| `apps/chefooz-app/src/components/explore/modules/ExploreMenuShowcase.tsx` | Horizontal scroll of orderable food cards from `mapReelsToFoodItems` |
| `apps/chefooz-app/src/components/explore/modules/ExplorePromoStrip.tsx` | Featured PROMOTIONAL reel strip |
| `apps/chefooz-app/src/components/explore/modules/ExploreTrendingThemes.tsx` | Hashtag discovery chips from `sections.hashtags` |
| `apps/chefooz-app/src/components/explore/modules/ExploreChefWorld.tsx` | Deep chef discovery cards with thumbnail collage, bio, follow action |
| `apps/chefooz-app/src/components/explore/modules/ExploreFreshPicks.tsx` | Horizontal reel cards from `sections.newDishes` |
| `libs/utils/src/lib/explore-data-mapper.ts` | Added `classifyReelBuckets()` + `ExploreReelBuckets` |

### Audience Profile Detection

```typescript
// apps/chefooz-app/src/hooks/useExploreAudienceProfile.ts
export type ExploreAudienceProfile = 'NEW_USER' | 'RETURNING_USER';

const NEW_USER_THRESHOLD_DAYS = 14;

export function useExploreAudienceProfile(user): ExploreAudienceResult {
  const isNewUser = !user || !user.createdAt
    || (Date.now() - new Date(user.createdAt).getTime()) < NEW_USER_THRESHOLD_DAYS * 86400000;
  return { profile: isNewUser ? 'NEW_USER' : 'RETURNING_USER', isNewUser };
}
```

- Relies only on `auth.store` `user.createdAt` — no additional API call
- 14-day threshold is configurable via the constant `NEW_USER_THRESHOLD_DAYS`

### Content Bucketing

```typescript
// libs/utils/src/lib/explore-data-mapper.ts
export interface ExploreReelBuckets {
  menu: Reel[];   // MENU_SHOWCASE or has linkedMenu
  promo: Reel[];  // PROMOTIONAL
  review: Reel[]; // USER_REVIEW or has linkedOrder
}

export function classifyReelBuckets(reels: Reel[]): ExploreReelBuckets {
  return {
    menu:   reels.filter(r => r.reelPurpose === 'MENU_SHOWCASE' || (r.linkedMenu && typeof r.linkedMenu === 'object')),
    promo:  reels.filter(r => r.reelPurpose === 'PROMOTIONAL'),
    review: reels.filter(r => r.reelPurpose === 'USER_REVIEW' || !!r.linkedOrder),
  };
}
```

Only the 3 `ReelPurpose` values available in `libs/types` are used (`USER_REVIEW | PROMOTIONAL | MENU_SHOWCASE`). There is no `RECIPE`, `COOKING_CLASS`, or `BEHIND_THE_SCENES` value — do not filter for these.

### Data Sources per Module

| Module | Data source | Field |
|---|---|---|
| ExploreMenuShowcase | `mapReelsToFoodItems(trending + recommended)` | `ExploreFoodItem[]` |
| ExplorePromoStrip | `classifyReelBuckets().promo` | `Reel[]` |
| ExploreTrendingThemes | `sections.hashtags` | `HashtagSummary[]` |
| ExploreChefWorld | `trendingChefsData.chefs` | `ExploreChef[]` |
| ExploreFreshPicks | `sections.newDishes` | `Reel[]` |

### Lazy Loading

`ReelsThatMakeYouHungry` rows are conditionally rendered only after `reelsVisible === true`, which is set when `ScrollView.onScroll` fires with `contentOffset.y > 800`. This prevents reel rows from blocking above-the-fold food content on first render.

### Analytics Backward Compatibility

`layoutVariant` on `exploreAnalytics.trackInteraction()` is now passed as the fixed string `'FOOD_GRID'`. The layout variant concept is deprecated and this field should be treated as a legacy key in any analytics dashboard. The real audience profile is now the `profile` value from `useExploreAudienceProfile`.

### Responsive Approach

All spacing, padding, icon sizes, and font sizes in the new modules use `normalize()` and `normalizeFontSize()` from `@chefooz-app/utils/responsive`. Baseline is 375px. Fixed dimensions (avatar, card width, image height): `normalize(value)`.

---

**[UI_ENHANCEMENT ✅ Explore Unified Discovery Shell]**  
**Module**: Explore  
**Documentation**: Technical Guide  
**Last Updated**: March 2026

---

## Moderation Filter Constraint (Critical — March 2026)

**Rule**: Every `reelModel.find()` call that serves public content MUST include `moderationStatus: { $ne: 'rejected' }`.

**Background**: The `moderationStatus` field was added after initial deployment. Legacy reels have `moderationStatus: undefined`. Using `moderationStatus: 'approved'` would exclude all legacy reels. The correct filter is `{ $ne: 'rejected' }` which includes `undefined`, `'approved'`, `'pending'`, `'reviewing'` and excludes only banned content.

**Affected files**:

| File | Method | Filter added |
|------|---------|--------------|
| `explore.service.ts` | `getTrendingItems` | `moderationStatus: { $ne: 'rejected' }` |
| `explore.service.ts` | `getRecommendedItems` | `moderationStatus: { $ne: 'rejected' }` |
| `explore.service.ts` | `getNewDishItems` | `moderationStatus: { $ne: 'rejected' }` |
| `explore.service.ts` | `getPromotedItems` | `moderationStatus: { $ne: 'rejected' }` |
| `explore.service.ts` | `getVirtualSectionDetail` (base `reelQuery`) | `moderationStatus: { $ne: 'rejected' }` |
| `explore.service.ts` | `getReelsByCategory` | `moderationStatus: { $ne: 'rejected' }` |
| `explore.service.ts` | `getRankedExploreItems` | `moderationStatus: { $ne: 'rejected' }` |
| `chef-public.service.ts` | `getChefReels` | `moderationStatus: { $ne: 'rejected' }` |

**Already protected**: `feed.service.ts` (uses `$or` pattern), `search-elastic.service.ts` Mongo fallback (uses `moderationStatus: 'approved'`), `getTrendingHashtags` aggregation (uses this filter since initial implementation).

**DO NOT** add this filter to chef-owned management endpoints (`getUserReels`, `getChefMenuReels`, `reels.service.ts` listing) — owners need to see their own pending/rejected reels for awareness.

## TrendingTagsStrip — "See All" Merged from ExploreTrendingThemes (March 2026)

The previously separate `ExploreTrendingThemes` component (scroll-section hashtag strip with a "See All" button) has been merged into `TrendingTagsStrip`.

- `TrendingTagsStrip` now accepts an optional `onSeeAll?: () => void` prop
- When provided, a "See All →" label appears at the right of the "Trending" header row
- `ExploreTrendingThemes` is no longer rendered in `explore.tsx` for the returning-user scroll section (Module 3b removed)
- The `ExploreTrendingThemes.tsx` file remains on disk (not deleted) but is no longer imported in any active screen

---

## Near-You Distance Filtering (implemented March 2026)

### Problem
`ExploreMenuShowcase` ("Dishes Near You" / "Order Something Delicious") was showing global trending and recommended reels with **zero location awareness**. The backend `getNearYouItems()` was a TODO stub that always returned empty.

### Architecture

**Backend (`explore.service.ts`)**
- `getNearYouItems()` now implements Haversine-based proximity filtering:
  1. Loads all `ChefKitchen` records with non-null `latitude`/`longitude`
  2. Computes great-circle distance from user's coordinates to each chef kitchen
  3. Filters to chefs within `min(EXPLORE_NEAR_YOU_RADIUS_KM, kitchen.deliveryRadiusKm)` km
  4. Maps `ChefProfile.id → User.id` to match reel `userId` for `MENU_SHOWCASE` reels
  5. Queries MongoDB reels where poster is a nearby chef (`userId IN nearbyChefUserIds`) OR `linkedMenu.chefId` is a nearby chefId
  6. Applies the same enrichment pipeline (users, orders, linkedMenu, privacy filter) as `getTrendingItems`
- Env var `EXPLORE_NEAR_YOU_RADIUS_KM` (default: `25`) — platform-wide max search radius
- `ChefKitchen.deliveryRadiusKm` further restricts per-chef; effective radius = `min(platform max, chef radius)`
- Cache key now includes rounded lat/lng (`round(lat * 100) / 100`) so nearby users share the same cache bucket
- When coordinates are present in the request, `nearYou` is always fetched (regardless of `includeNearYou` flag)

**Frontend (`explore.tsx`)**
- Reads `defaultAddress.lat` / `defaultAddress.lng` from `useAddressStore`
- Passes them to `useConsolidatedExploreSections` when available
- `menuFoodItems` source: prefers `sections.nearYou` when it has results; falls back to `trending + recommended` when no location or no nearby chefs found
- Both `ExploreMenuShowcase` titles/subtitles update dynamically: "Dishes Near You / near you" vs "Popular Dishes / online chefs" depending on location availability

### Edge cases
- No default address set → location params omitted → `nearYou` section is empty → fallback to global trending
- Chef has no `latitude`/`longitude` in their `chef_kitchen` record → excluded from near-you results (does not appear in "near you", may appear in trending/recommended fallback)
- Chef's `deliveryRadiusKm = 0` → treated as no restriction, uses platform max
- User located further than `EXPLORE_NEAR_YOU_RADIUS_KM` from all chefs → `nearYou` empty → fallback to trending

---

## PAN-India Packaged Food Delivery Enrichment (March 2026)

### Problem
Packaged food items (`isPackagedFood=true`, `nationalDeliveryEnabled=true`) can be shipped across India via courier. However, if a product's shelf life is short (e.g., 3 days), it may expire in transit before reaching a user far from the kitchen. The explore feed needed to surface this constraint so users could see at a glance whether a national item can actually reach them.

### Domain Rule
```
isOrderable = (dispatchBufferDays + estimatedDeliveryDays) < shelfLifeDays
```
**STRICT**: equal is NOT orderable — a product arriving on its expiry day is unacceptable.
See `libs/domain/src/lib/packaged-food-delivery.ts`.

### Data Model Additions

| Column | Table | Type | Default | Purpose |
|---|---|---|---|---|
| `nationalDeliveryEnabled` | `chef_menu_item` | `BOOLEAN` | `false` | Per-item PAN-India shipping toggle |
| `dispatchBufferDays` | `chef_menu_item` | `SMALLINT` | `1` | Packing days before courier pickup |
| `pincode` | `chef_kitchen` | `VARCHAR(10)` | `NULL` | Kitchen PIN code for Shiprocket serviceability |

Migration: `1775000000000-AddPackagedDeliveryFields.ts` (idempotent, uses `IF NOT EXISTS`).

### Backend Enrichment Flow (`explore.service.ts`)

1. `getAllSections()` calls `enrichWithDeliveryFeasibility(result, userId)` **after** full section mapping, **before** Redis caching
2. `enrichWithDeliveryFeasibility()`:
   - Collects all `menuItemIds` from items with a `linkedMenu` property
   - Queries TypeORM for those items where `nationalDeliveryEnabled = true` (one batched query)
   - Fetches user's default `UserAddress` (pincode, lat, lng)
   - Fetches `ChefKitchen.pincode` for relevant chef IDs
   - For each matching reel item, calls `PackagedDeliveryService.checkFeasibility()`
   - Attaches result to `item.linkedMenu.deliveryFeasibility`
3. If the `chef_menu_item` table doesn't yet have the `nationalDeliveryEnabled` column (pre-migration), the query failure is caught silently and enrichment is skipped

### PackagedDeliveryService

Located at `apps/chefooz-apis/src/modules/packaged-delivery/packaged-delivery.service.ts`.

- `checkFeasibility(params)`: Shiprocket serviceability → domain rule → `DeliveryFeasibility`
- `checkFeasibilityBatch(request)`: runs all items via `Promise.all`
- When `noLocationData=true` (user has no address/coords): returns `isOrderable=true` (optimistic — cannot gate without data)

### ShiprocketClient

Located at `apps/chefooz-apis/src/modules/packaged-delivery/shiprocket.client.ts`.

| Operation | Endpoint | Cache TTL |
|---|---|---|
| Auth | `POST /v1/external/auth/login` | 9 days (`shiprocket:token`) |
| Serviceability | `GET /v1/external/courier/serviceability` | 6 hours (`courier:svc:{origin}:{dest}`) |

- On HTTP 401: invalidates cached token, retries once
- Fallback (Shiprocket unavailable or unconfigured): distance-based zone estimate
  - 0–200 km → 2 days, 200–600 km → 3 days, 600–1500 km → 4 days, 1500+ km → 5 days

### Required Environment Variables

| Variable | Example | Description |
|---|---|---|
| `SHIPROCKET_BASE_URL` | `https://apiv2.shiprocket.in` | Shiprocket API base |
| `SHIPROCKET_EMAIL` | `ops@chefooz.com` | Shiprocket login email |
| `SHIPROCKET_PASSWORD` | `***` | Shiprocket login password |

### Frontend Implementation (`ExploreMenuShowcase.tsx`)

| State | UI Behaviour |
|---|---|
| `isNational` | "🇮🇳 Ships India" badge overlaid on item image (bottom-right) |
| `isNational` + orderable | "Delivers in N days" label instead of ETA minutes |
| `isNotOrderable` | Card opacity 65%, CTA = "Not Available" (grey, disabled), red reason text below price |
| `noLocationData=true` | CTA remains "Order Now" (optimistic — user hasn't set address yet) |

### Edge Cases

| Scenario | Behaviour |
|---|---|
| User has no default address | `noLocationData=true` → CTA optimistically enabled |
| Chef kitchen has no `pincode` | Falls back to Haversine distance estimate |
| Shiprocket returns no available couriers | `isServiceable=false` → `isOrderable=false` |
| `shelfLifeDays=0` or unset | Domain rule always fails → not orderable |
| `dispatchBuffer + deliveryDays === shelfLifeDays` | NOT orderable (strict boundary) |
| Pre-migration table state | TypeORM query fails gracefully; enrichment skipped |

**Last Updated**: March 2026
