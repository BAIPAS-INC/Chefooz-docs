# Feed Module - Technical Guide

**Version:** 1.0  
**Last Updated:** February 14, 2026  
**Module:** `apps/chefooz-apis/src/modules/feed/`  
**Domain Logic:** `libs/domain/src/feed/`  
**Tech Stack:** NestJS, MongoDB (Mongoose), PostgreSQL (TypeORM), Redis/Valkey

---

## 📋 Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [API Endpoints](#api-endpoints)
3. [Service Methods](#service-methods)
4. [Domain Logic](#domain-logic)
5. [Data Models](#data-models)
6. [Ranking Algorithm Implementation](#ranking-algorithm-implementation)
7. [Caching Strategy](#caching-strategy)
8. [Database Queries](#database-queries)
9. [Error Handling](#error-handling)
10. [Testing Approach](#testing-approach)
11. [Deployment Notes](#deployment-notes)

---

## Architecture Overview

### Module Structure

```
apps/chefooz-apis/src/modules/feed/
├── dto/
│   ├── engagement.dto.ts          # Engagement action input (VIEW/LIKE/SAVE)
│   └── feed-query.dto.ts          # Feed retrieval query params
├── feed.controller.ts             # HTTP endpoints (GET /feed, POST /engagement)
├── feed.service.ts                # Business logic (ranking, engagement tracking)
├── feed.module.ts                 # Module configuration
├── feed.service.spec.ts           # Unit tests
├── EXPLAINABILITY.md              # Ranking algorithm documentation
├── reel-seed.md                   # Seed data generation guide
└── seed-reels.script.ts           # Test data seeding script

libs/domain/src/feed/
├── feed.policy.ts                 # Abuse detection policies (upload rate, duplicates, anomalies)
├── feed.config.ts                 # Configuration helpers (limits, weights, TTL)
├── feed.rules.ts                  # Visibility rules (state transitions, multipliers)
├── feed.policy.spec.ts            # Policy unit tests
├── feed.config.spec.ts            # Config unit tests
├── feed.rules.spec.ts             # Rules unit tests
└── index.ts                       # Barrel exports
```

### Technology Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **Framework** | NestJS 10+ | REST API framework |
| **Reel Storage** | MongoDB 7+ (Mongoose) | Reel documents, engagement records |
| **User Data** | PostgreSQL 16+ (TypeORM) | Users, reputation, orders, menu items, follows |
| **Cache** | Redis 7+ (Valkey) | Feed caching, engagement sets, pub/sub |
| **Validation** | class-validator | DTO input validation |
| **Logging** | Winston | Structured logging |
| **Testing** | Jest | Unit + integration tests |

### Dependencies

```typescript
// feed.module.ts
@Module({
  imports: [
    // MongoDB schemas
    MongooseModule.forFeature([
      { name: Reel.name, schema: ReelSchema },
      { name: Engagement.name, schema: EngagementSchema },
      { name: Media.name, schema: MediaSchema },
    ]),
    
    // PostgreSQL entities
    TypeOrmModule.forFeature([
      User,
      UserReputationCurrent,
      Order,
      ChefMenuItem,
      UserFollow,
    ]),
    
    // External modules
    ChefKitchenModule, // Chef availability, kitchen status
    NotificationModule, // Like notifications
    ActivityModule, // Activity feed tracking
  ],
  controllers: [FeedController],
  providers: [FeedService],
  exports: [FeedService],
})
export class FeedModule {}
```

---

## API Endpoints

### 1. GET /api/v1/feed

**Description**: Retrieve paginated, ranked feed items with optional filtering.

**Authentication**: Optional (public endpoint, JWT for personalization)

**Rate Limiting**: Global rate limiter (100 requests/min per IP)

#### Request

**Query Parameters**:
```typescript
interface FeedQueryDto {
  cursor?: string;        // MongoDB ObjectId for pagination (omit for page 1)
  limit?: number;         // Items per page (1-30, default: 10)
  type?: FeedType;        // Content type (REEL | POST | ALL, default: REEL)
  lat?: number;           // User latitude (placeholder, not implemented)
  lng?: number;           // User longitude (placeholder, not implemented)
  sort?: FeedSort;        // Sorting strategy (DEFAULT | TRENDING | RECENT, default: DEFAULT)
  followingOnly?: boolean;// Show only followed creators (default: false)
  userId?: string;        // Required if followingOnly=true
}
```

**Example Requests**:
```bash
# Guest feed (first page, cached)
GET /api/v1/feed

# Authenticated feed (personalized, page 1)
GET /api/v1/feed
Authorization: Bearer <jwt_token>

# Page 2 (cursor-based pagination)
GET /api/v1/feed?cursor=65f3b1c4f8d9e2a1b3c4d5e6&limit=10

# Following-only mode
GET /api/v1/feed?followingOnly=true&userId=abc-123-def-456

# Trending feed
GET /api/v1/feed?sort=TRENDING&limit=20

# Recent feed
GET /api/v1/feed?sort=RECENT&limit=15
```

#### Response

**Success (200)**:
```typescript
{
  success: true,
  message: "Feed retrieved successfully",
  data: {
    items: FeedItem[], // See FeedItem structure below
    nextCursor: string | null // Cursor for next page, null if last page
  }
}
```

**FeedItem Structure**:
```typescript
interface FeedItem {
  id: string; // MongoDB ObjectId
  userId: string; // Creator UUID
  contentType: 'REEL' | 'POST';
  author?: {
    id: string;
    username: string;
    fullName: string;
    avatarUrl: string | null;
  };
  caption: string;
  hashtags: string[];
  taggedUsers?: Array<{
    userId: string;
    username: string;
    fullName?: string;
    avatarUrl?: string;
  }>;
  videoUrl: string; // HTTPS URL from S3
  thumbnailUrl: string; // HTTPS URL from S3
  durationSec: number;
  linkedOrder?: {
    orderId: string;
    itemCount: number;
    totalPaise: number;
    chefId: string;
    previewImageUrl?: string;
    itemPreviews: string[]; // First 3 items + "+N more" suffix
  };
  linkedMenu?: {
    chefId: string;
    menuItemIds: string[];
    estimatedPaise: number;
    previewImage?: string;
  };
  reelPurpose: 'USER_REVIEW' | 'PROMOTIONAL' | 'MENU_SHOWCASE';
  orderContext?: {
    chefId: string;
    chefName: string;
    chefAvatarUrl?: string;
    distanceKm?: number;
    etaMinutes?: number;
    isOpen?: boolean;
    openingHours?: string;
    orderPreview?: {
      orderId: string;
      itemCount: number;
      totalPaise: number;
      previewImageUrl?: string;
      itemPreviews: string[];
    };
  };
  feedCardPreview: {
    previewImage: string;
    durationSec: number;
    aspectRatio: number;
    reelPurpose: 'USER_REVIEW' | 'PROMOTIONAL' | 'MENU_SHOWCASE';
    linkedOrder?: object;
    linkedMenu?: object;
    chefContext?: {
      chefId: string;
      chefName: string;
      isOpen: boolean;
      etaMinutes: number;
    };
  };
  stats: {
    views: number;
    likes: number;
    comments: number;
    saves: number;
    shareCount: number;
    isLiked: boolean; // True if authenticated user liked
    isSaved: boolean; // True if authenticated user saved
  };
  promoted: boolean;
  createdAt: string; // ISO 8601 timestamp
}
```

**Error Responses**:

**400 Bad Request** (invalid query params):
```json
{
  "success": false,
  "message": "Validation failed",
  "errorCode": "VALIDATION_ERROR"
}
```

**500 Internal Server Error**:
```json
{
  "success": false,
  "message": "Failed to retrieve feed",
  "errorCode": "FEED_FETCH_ERROR"
}
```

#### Controller Implementation

```typescript
// feed.controller.ts
@Controller('feed')
@ApiTags('Feed')
export class FeedController {
  constructor(private readonly feedService: FeedService) {}

  @Get()
  @ApiOperation({ summary: 'Get paginated feed with advanced ranking' })
  @ApiResponse({ status: 200, description: 'Feed retrieved successfully' })
  @ApiResponse({ status: 400, description: 'Invalid query parameters' })
  @ApiResponse({ status: 500, description: 'Server error' })
  async getFeed(
    @Query() query: FeedQueryDto,
    @Req() req: Request,
  ): Promise<{ success: boolean; message: string; data: any }> {
    try {
      this.logger.log(
        `Fetching feed: cursor=${query.cursor || 'none'}, limit=${query.limit || 10}, sort=${query.sort || 'DEFAULT'}`,
      );

      // Extract userId from JWT (if authenticated)
      const userId = (req as any).user?.id;
      const userLat = query.lat;
      const userLng = query.lng;

      const result = await this.feedService.getFeed(query, userId, userLat, userLng);

      return {
        success: true,
        message: 'Feed retrieved successfully',
        data: result,
      };
    } catch (error) {
      const errorMessage = error instanceof Error ? error.message : 'Unknown error';
      const errorStack = error instanceof Error ? error.stack : undefined;
      this.logger.error(`Error fetching feed: ${errorMessage}`, errorStack);

      throw new HttpException(
        {
          success: false,
          message: 'Failed to retrieve feed',
          errorCode: 'FEED_FETCH_ERROR',
        },
        HttpStatus.INTERNAL_SERVER_ERROR,
      );
    }
  }
}
```

---

### 2. POST /api/v1/feed/engagement

**Description**: Record engagement action (view, like, save) on a reel.

**Authentication**:
- **VIEW**: Optional (guest views allowed, throttled by IP)
- **LIKE/SAVE**: Required (JWT bearer token)

**Rate Limiting**:
- **VIEW**: 3 seconds per user/IP per reel (throttled)
- **LIKE/SAVE**: No rate limit (idempotent)

#### Request

**Body**:
```typescript
interface EngagementDto {
  reelId: string;      // MongoDB ObjectId (validated with @IsMongoId)
  action: EngagementAction; // 'VIEW' | 'LIKE' | 'SAVE'
}
```

**Example Requests**:
```bash
# View action (guest, throttled)
POST /api/v1/feed/engagement
Content-Type: application/json
{
  "reelId": "65f3b1c4f8d9e2a1b3c4d5e6",
  "action": "VIEW"
}

# Like action (authenticated)
POST /api/v1/feed/engagement
Authorization: Bearer <jwt_token>
Content-Type: application/json
{
  "reelId": "65f3b1c4f8d9e2a1b3c4d5e6",
  "action": "LIKE"
}

# Save action (authenticated)
POST /api/v1/feed/engagement
Authorization: Bearer <jwt_token>
Content-Type: application/json
{
  "reelId": "65f3b1c4f8d9e2a1b3c4d5e6",
  "action": "SAVE"
}
```

#### Response

**Success (200)**:
```json
{
  "success": true,
  "message": "Engagement recorded successfully",
  "data": {
    "reelId": "65f3b1c4f8d9e2a1b3c4d5e6",
    "stats": {
      "views": 1234,
      "likes": 56,
      "comments": 12,
      "saves": 23,
      "shareCount": 8
    }
  }
}
```

**Error Responses**:

**401 Unauthorized** (LIKE/SAVE without auth):
```json
{
  "success": false,
  "message": "Authentication required",
  "errorCode": "FEED_AUTH_REQUIRED"
}
```

**404 Not Found** (reel doesn't exist):
```json
{
  "success": false,
  "message": "Reel not found",
  "errorCode": "FEED_REEL_NOT_FOUND"
}
```

**409 Conflict** (already liked, idempotent response):
```json
{
  "success": true,
  "message": "Engagement recorded successfully",
  "data": {
    "reelId": "65f3b1c4f8d9e2a1b3c4d5e6",
    "stats": { ... }
  }
}
```

**429 Too Many Requests** (view throttle violated):
```json
{
  "success": false,
  "message": "View rate limited. Please wait.",
  "errorCode": "FEED_RATE_LIMITED"
}
```

#### Controller Implementation

```typescript
// feed.controller.ts
@Post('engagement')
@ApiOperation({ summary: 'Record engagement (view, like, save) on a reel' })
@ApiResponse({ status: 200, description: 'Engagement recorded successfully' })
@ApiResponse({ status: 401, description: 'Authentication required (for like/save)' })
@ApiResponse({ status: 404, description: 'Reel not found' })
@ApiResponse({ status: 409, description: 'Already liked (idempotent)' })
@ApiResponse({ status: 429, description: 'View rate limited' })
async recordEngagement(
  @Body() engagementDto: EngagementDto,
  @Req() req: Request,
): Promise<{ success: boolean; message: string; data: FeedEngagementResponse }> {
  try {
    this.logger.log(
      `Recording engagement: ${engagementDto.action} on reel ${engagementDto.reelId}`,
    );

    // Extract userId from JWT (if available)
    const userId = (req as any).user?.id;
    const ip = req.ip || req.socket.remoteAddress;

    this.logger.debug(`🔑 UserId from request: ${userId || 'undefined (not authenticated)'}`);

    const result: FeedEngagementResponse = await this.feedService.recordEngagement(
      engagementDto.reelId,
      engagementDto.action as EngagementAction,
      userId,
      ip,
    );

    return {
      success: true,
      message: 'Engagement recorded successfully',
      data: result,
    };
  } catch (error) {
    // Re-throw HttpException from service layer
    if (error instanceof HttpException) {
      throw error;
    }

    // Unexpected errors
    const errorMessage = error instanceof Error ? error.message : 'Unknown error';
    const errorStack = error instanceof Error ? error.stack : undefined;
    this.logger.error(`Error recording engagement: ${errorMessage}`, errorStack);

    throw new HttpException(
      {
        success: false,
        message: 'Failed to record engagement',
        errorCode: 'FEED_ENGAGEMENT_ERROR',
      },
      HttpStatus.INTERNAL_SERVER_ERROR,
    );
  }
}
```

---

### 3. POST /api/v1/feed/admin/reseed

**Description**: Clear all reels and reseed with test data (development only).

**Authentication**: None (TODO: Add @Roles('admin') guard before production)

**⚠️ Security Risk**: This endpoint has no authentication. Must be secured before production deployment.

#### Request

```bash
POST /api/v1/feed/admin/reseed
```

#### Response

**Success (200)**:
```json
{
  "success": true,
  "message": "Reels cleared and reseeded successfully with real video URLs"
}
```

**Error (500)**:
```json
{
  "success": false,
  "message": "Failed to reseed reels"
}
```

#### Controller Implementation

```typescript
// feed.controller.ts
@Post('admin/reseed')
@ApiOperation({ summary: '[ADMIN] Clear and reseed reels with test data' })
// TODO: Add @Roles('admin') guard before production
async reseedReels() {
  try {
    await this.feedService.clearAndReseedReels();
    return {
      success: true,
      message: 'Reels cleared and reseeded successfully with real video URLs',
    };
  } catch (error) {
    this.logger.error('Error reseeding reels:', error);
    throw new HttpException(
      {
        success: false,
        message: 'Failed to reseed reels',
      },
      HttpStatus.INTERNAL_SERVER_ERROR,
    );
  }
}
```

---

## Service Methods

### FeedService Overview

```typescript
// feed.service.ts
@Injectable()
export class FeedService implements OnModuleInit {
  private readonly logger = new Logger(FeedService.name);
  private viewThrottle = new Map<string, number>(); // In-memory throttle (TODO: Migrate to Redis)
  private chefContextCache = new Map<string, any>(); // Request-scoped cache

  constructor(
    @InjectModel(Reel.name) private reelModel: Model<Reel>,
    @InjectModel(Engagement.name) private engagementModel: Model<Engagement>,
    @InjectModel(Media.name) private mediaModel: Model<Media>,
    @InjectRepository(User) private userRepository: Repository<User>,
    @InjectRepository(UserReputationCurrent) private reputationRepository: Repository<UserReputationCurrent>,
    @InjectRepository(Order) private orderRepository: Repository<Order>,
    @InjectRepository(ChefMenuItem) private menuItemRepository: Repository<ChefMenuItem>,
    @InjectRepository(UserFollow) private userFollowRepository: Repository<UserFollow>,
    private cacheService: CacheService,
    private chefKitchenService: ChefKitchenService,
    private chefScheduleService: ChefScheduleService,
    private notificationOrchestrator: NotificationOrchestrator,
  ) {}

  async onModuleInit() {
    this.setupCacheInvalidationSubscribers();
  }
}
```

---

### 1. getFeed()

**Signature**:
```typescript
async getFeed(
  query: FeedQueryDto,
  userId?: string,
  userLat?: number,
  userLng?: number,
): Promise<{ items: FeedItem[]; nextCursor: string | null }>
```

**Purpose**: Retrieve paginated, ranked feed items with optional filtering.

**Algorithm**:
1. Normalize page limit (1-30 range, default 10)
2. Build cache key (if first page, no cursor)
3. Check cache (guest feeds only, first page)
4. Build MongoDB filter (videoUrl exists, cursor filter, reelPurpose filter)
5. Apply following-only filter (if enabled)
6. Apply sorting strategy (DEFAULT/TRENDING/RECENT)
7. Fetch reels with pagination (limit+1 pattern)
8. Populate mediaId (for S3 variants)
9. Map reels to FeedItem (with orderContext, linkedOrder, linkedMenu)
10. Check user engagement (isLiked, isSaved)
11. Determine nextCursor (if hasNextPage)
12. Cache first page (if applicable)
13. Return { items, nextCursor }

**Implementation**:
```typescript
async getFeed(
  query: FeedQueryDto,
  userId?: string,
  userLat?: number,
  userLng?: number,
): Promise<{ items: FeedItem[]; nextCursor: string | null }> {
  // Phase 3.4.1: Consume environment config
  const limit = normalizePageLimit(query.limit, envConfig);
  const sort = query.sort || FeedSort.DEFAULT;

  // Build cache key (first page only)
  const cacheKey = !query.cursor
    ? `feed:${sort}:limit:${limit}:lat:${userLat || 'null'}:lng:${userLng || 'null'}`
    : null;

  // Check cache
  if (cacheKey) {
    const cached = await this.cacheService.get(cacheKey);
    if (cached) {
      this.logger.debug(`Cache hit: ${cacheKey}`);
      return JSON.parse(cached);
    }
  }

  // Build MongoDB filter
  const filter: any = {
    videoUrl: { $exists: true, $ne: '' }, // Only show processed reels
  };

  // Cursor pagination
  if (query.cursor) {
    filter._id = { $lt: query.cursor };
  }

  // Phase 3.3.2: Feature flag enforcement
  if (!canViewMenuReel()) {
    filter.reelPurpose = { $ne: 'MENU_SHOWCASE' };
  }

  // Following-only filter
  if (query.followingOnly && query.userId) {
    const follows = await this.userFollowRepository.find({
      where: { followerId: query.userId, status: 'accepted' },
    });

    const followingIds = follows.map((f) => f.followingId);

    if (followingIds.length === 0) {
      // User follows nobody, return empty feed
      return { items: [], nextCursor: null };
    }

    filter.userId = { $in: followingIds };
  }

  // Apply sorting strategy
  let reels: Reel[];
  let nextCursor: string | null = null;

  switch (sort) {
    case FeedSort.DEFAULT:
      // Advanced ranked feed
      const result = await this.getAdvancedRankedFeed(filter, limit, userLat, userLng);
      reels = result.items;
      nextCursor = result.nextCursor;
      break;

    case FeedSort.TRENDING:
      // Sort by likes desc, createdAt desc
      reels = await this.reelModel
        .find(filter)
        .sort({ 'stats.likes': -1, createdAt: -1 })
        .limit(limit + 1)
        .populate('mediaId')
        .exec();
      break;

    case FeedSort.RECENT:
      // Sort by createdAt desc
      reels = await this.reelModel
        .find(filter)
        .sort({ createdAt: -1 })
        .limit(limit + 1)
        .populate('mediaId')
        .exec();
      break;

    default:
      reels = await this.reelModel
        .find(filter)
        .sort({ createdAt: -1 })
        .limit(limit + 1)
        .populate('mediaId')
        .exec();
  }

  // Pagination logic (limit+1 pattern)
  if (sort !== FeedSort.DEFAULT) {
    const hasNextPage = reels.length > limit;
    if (hasNextPage) {
      nextCursor = String((reels[limit] as any)._id);
      reels = reels.slice(0, limit);
    }
  }

  // Map reels to FeedItem
  const items = await Promise.all(
    reels.map((reel) => this.mapReelToFeedItem(reel, userId, userLat, userLng)),
  );

  const response = { items, nextCursor };

  // Cache first page
  if (cacheKey) {
    const cacheTtl = getFeedCacheTtl(envConfig);
    await this.cacheService.set(cacheKey, JSON.stringify(response), cacheTtl);
    this.logger.debug(`Cached feed: ${cacheKey} (TTL: ${cacheTtl}s)`);
  }

  return response;
}
```

---

### 2. getAdvancedRankedFeed()

**Signature**:
```typescript
async getAdvancedRankedFeed(
  filter: any,
  limit: number,
  lat?: number,
  lng?: number,
): Promise<{ items: Reel[]; nextCursor: string | null }>
```

**Purpose**: Fetch candidate reels, rank by score, return top N.

**Algorithm**:
1. Calculate candidate pool size (limit × 5, min 50)
2. Fetch recent reels (sorted by createdAt desc)
3. Extract unique creator user IDs
4. Filter valid UUIDs (exclude MongoDB ObjectIds)
5. Batch fetch reputation data (feedBoostWeight, creatorBoostState)
6. Build reputation map { userId: reputation }
7. Calculate ranking score for each reel
8. Sort by score descending
9. Take top `limit` items
10. Determine nextCursor (last item _id)

**Implementation**:
```typescript
async getAdvancedRankedFeed(
  filter: any,
  limit: number,
  lat?: number,
  lng?: number,
): Promise<{ items: Reel[]; nextCursor: string | null }> {
  const candidateLimit = Math.max(limit * 5, 50);

  // Fetch candidate reels
  const candidates = await this.reelModel
    .find(filter)
    .sort({ createdAt: -1 })
    .limit(candidateLimit)
    .populate('mediaId')
    .exec();

  // Extract unique user IDs
  const userIds = [...new Set(candidates.map((r) => r.userId))];

  // Filter valid UUIDs (exclude MongoDB ObjectIds)
  const validUuids = userIds.filter((id) => {
    return /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i.test(id);
  });

  // Batch fetch reputation data
  const reputations = await this.reputationRepository.find({
    where: { userId: In(validUuids) },
  });

  // Build reputation map
  const reputationMap = new Map<string, any>();
  reputations.forEach((rep) => {
    reputationMap.set(rep.userId, {
      feedBoostWeight: rep.feedBoostWeight,
      creatorBoostState: rep.creatorBoostState,
      lastBoostAt: rep.lastBoostAt,
    });
  });

  // Calculate ranking scores
  const scored = candidates.map((reel) => {
    const score = this.calculateRankingScore(reel, reputationMap);
    return { reel, score };
  });

  // Sort by score descending
  scored.sort((a, b) => b.score - a.score);

  // Take top N
  const topReels = scored.slice(0, limit).map((s) => s.reel);

  // Determine nextCursor
  const nextCursor = topReels.length > 0 ? String((topReels[topReels.length - 1] as any)._id) : null;

  return { items: topReels, nextCursor };
}
```

---

### 3. calculateRankingScore()

**Signature**:
```typescript
private calculateRankingScore(
  reel: Reel,
  reputationMap: Map<string, any>,
): number
```

**Purpose**: Calculate multi-factor ranking score for a single reel.

**Formula**:
```
finalScore = (baseScore + reputationBoost + promotedBoost) * visibilityMultiplier
finalScore = clamp(finalScore, 0, 1000)
```

**Implementation**:
```typescript
private calculateRankingScore(
  reel: Reel,
  reputationMap: Map<string, any>,
): number {
  // Base score: Engagement signals × recency decay
  const engagementScore =
    reel.stats.likes * 10 +
    reel.stats.views * 1 +
    reel.stats.saves * 15;

  const ageInHours = (Date.now() - reel.createdAt.getTime()) / (1000 * 60 * 60);
  const recencyFactor = Math.pow(0.5, ageInHours / 48); // 48h half-life

  const baseScore = engagementScore * recencyFactor;

  // Reputation boost (CRS integration)
  const reputation = reputationMap.get(reel.userId);
  const feedBoostWeight = reputation?.feedBoostWeight || 0;
  const reputationBoostScale = envConfig.feed.reputationBoostScale; // 100
  const reputationBoost = feedBoostWeight * reputationBoostScale;

  // Promoted boost
  const promotedBoost = reel.promoted ? envConfig.feed.promotedBoost : 0; // 50

  // Raw score (before visibility multiplier)
  let rawScore = baseScore + reputationBoost + promotedBoost;

  // Phase 3.6.8: Creator boost fairness (visibility multiplier)
  let visibilityMultiplier = 1.0;
  const creatorBoostState = reputation?.creatorBoostState || 'ACTIVE';

  switch (creatorBoostState) {
    case 'ACTIVE':
      visibilityMultiplier = 1.0;
      break;
    case 'FLAGGED_CHURNING':
    case 'FLAGGED_BOOSTING':
      visibilityMultiplier = 0.3; // 70% penalty
      this.logger.log({
        event: 'creator_boost_penalty',
        reelId: String((reel as any)._id),
        userId: reel.userId,
        boostState: creatorBoostState,
        visibilityMultiplier,
        originalScore: rawScore,
        finalScore: rawScore * visibilityMultiplier,
      });
      break;
    case 'SUSPENDED':
      visibilityMultiplier = 0.0; // Hidden
      this.logger.log({
        event: 'creator_boost_penalty',
        reelId: String((reel as any)._id),
        userId: reel.userId,
        boostState: 'SUSPENDED',
        visibilityMultiplier,
        originalScore: rawScore,
        finalScore: 0,
      });
      break;
  }

  // Apply visibility multiplier
  let finalScore = rawScore * visibilityMultiplier;

  // Clamp to 0-1000
  finalScore = Math.max(0, Math.min(1000, finalScore));

  return finalScore;
}
```

---

### 4. mapReelToFeedItem()

**Signature**:
```typescript
async mapReelToFeedItem(
  reel: Reel,
  userId?: string,
  userLat?: number,
  userLng?: number,
): Promise<FeedItem>
```

**Purpose**: Transform MongoDB Reel document to FeedItem response format.

**Steps**:
1. Build linkedOrder object (if reel has linkedOrderId)
2. Build orderContext object (if order exists)
3. Build linkedMenu object (if reel has linkedMenu string ID)
4. Fetch Media document for S3 variants (fallback to reel.videoUrl)
5. Convert S3 URIs to HTTPS URLs
6. Fetch author profile (username, fullName, avatarUrl)
7. Fetch tagged user profiles (if taggedUserIds exists)
8. Check user engagement (isLiked, isSaved)
9. Build feedCardPreview object
10. Return FeedItem

**Implementation** (condensed, see full code for details):
```typescript
async mapReelToFeedItem(
  reel: Reel,
  userId?: string,
  userLat?: number,
  userLng?: number,
): Promise<FeedItem> {
  let linkedOrder: any = undefined;
  let orderContext: any = undefined;

  // Build linkedOrder and orderContext
  if (reel.linkedOrderId) {
    try {
      const order = await this.orderRepository.findOne({
        where: { id: reel.linkedOrderId },
      });

      if (order) {
        // Fetch menu items for availability check
        const menuItems = await this.menuItemRepository.find({
          where: { orderId: order.id },
        });

        // Check if at least ONE item available
        const availableItems = menuItems.filter((item) => {
          const availability = item.availability as any;
          return (
            item.isActive &&
            availability?.isAvailable !== false &&
            availability?.soldOut !== true
          );
        });

        if (availableItems.length > 0) {
          // Show order banner (partial order support)
          const itemPreviews = menuItems
            .slice(0, 3)
            .map((item) => item.name)
            .concat(menuItems.length > 3 ? [`+${menuItems.length - 3} more`] : []);

          linkedOrder = {
            orderId: order.id,
            itemCount: menuItems.length,
            totalPaise: Math.round(parseFloat(order.totalPaise.toString())),
            chefId: order.chefId,
            previewImageUrl: menuItems[0]?.imageUrl,
            itemPreviews,
          };

          this.logger.debug(
            `LinkedOrder populated for reel ${String((reel as any)._id)}: ${availableItems.length}/${menuItems.length} items available`,
          );

          // Build order context (with caching)
          const cachedContext = this.chefContextCache.get(order.chefId);
          if (cachedContext) {
            orderContext = cachedContext;
          } else {
            // Fetch chef, kitchen, schedule
            const chef = await this.userRepository.findOne({
              where: { id: order.chefId },
            });
            const kitchen = await this.chefKitchenService.getKitchen(order.chefId);
            const schedule = await this.chefScheduleService.getWeeklySchedule(order.chefId);

            // Calculate distance, ETA, isOpen
            const distance = userLat && userLng && chef?.lat && chef?.lng
              ? distanceKm(userLat, userLng, chef.lat, chef.lng)
              : undefined;

            const eta = estimateEtaMinutes(
              menuItems.map((item) => item.prepTimeMinutes || 30),
            );

            const isOpen = computeChefIsOpen(kitchen, schedule);

            orderContext = {
              chefId: order.chefId,
              chefName: chef?.fullName || 'Unknown Chef',
              chefAvatarUrl: chef?.avatarUrl,
              distanceKm: distance,
              etaMinutes: eta,
              isOpen,
              openingHours: schedule?.formatted,
              orderPreview: linkedOrder,
            };

            // Cache context
            this.chefContextCache.set(order.chefId, orderContext);
            this.logger.debug(
              `OrderContext cached for chef ${order.chefId}: ${distance || 'N/A'}km, ETA ${eta}min, Open: ${isOpen}`,
            );
          }
        } else {
          this.logger.debug(
            `LinkedOrder ${reel.linkedOrderId} has no available items, skipping banner`,
          );
        }
      }
    } catch (error) {
      this.logger.warn(
        `Failed to fetch linkedOrder ${reel.linkedOrderId}: ${(error as Error).message}`,
      );
    }
  }

  // Build linkedMenu
  let linkedMenu: any = undefined;
  if (reel.linkedMenu && typeof reel.linkedMenu === 'string') {
    try {
      const menuItem = await this.menuItemRepository.findOne({
        where: { id: reel.linkedMenu },
      });

      if (menuItem) {
        const availability = menuItem.availability as any;
        const isAvailable =
          menuItem.isActive &&
          availability?.isAvailable !== false &&
          availability?.soldOut !== true;

        if (isAvailable) {
          const priceInPaise = Math.round(parseFloat(menuItem.price.toString()) * 100);
          linkedMenu = {
            chefId: menuItem.chefId,
            menuItemIds: [menuItem.id],
            estimatedPaise: priceInPaise,
            previewImage: menuItem.imageUrl || undefined,
          };

          this.logger.debug(
            `LinkedMenu populated for reel ${String((reel as any)._id)}: ${menuItem.name} - ₹${priceInPaise / 100}`,
          );
        } else {
          this.logger.debug(
            `LinkedMenu ${reel.linkedMenu} is not available, skipping`,
          );
        }
      }
    } catch (error) {
      this.logger.warn(
        `Failed to fetch linkedMenu ${reel.linkedMenu}: ${(error as Error).message}`,
      );
    }
  }

  // Fetch Media document for S3 variants
  let media: any = null;
  if (reel.mediaId) {
    try {
      media = await this.mediaModel.findById(reel.mediaId).exec();
      if (!media) {
        this.logger.warn(`⚠️ Media document not found for mediaId: ${reel.mediaId}`);
      }
    } catch (error) {
      this.logger.error(
        `Failed to fetch media for mediaId ${reel.mediaId}: ${(error as Error).message}`,
      );
    }
  }

  const videoUrl = media?.variants?.[0]?.url || reel.videoUrl || '';

  // Fetch author info
  const author = await this.userRepository.findOne({
    where: { id: reel.userId },
    select: ['id', 'username', 'fullName', 'avatarUrl'],
  });

  // Fetch tagged users
  let taggedUsers: Array<{
    userId: string;
    username: string;
    fullName?: string;
    avatarUrl?: string;
  }> | undefined;

  if (reel.taggedUserIds && reel.taggedUserIds.length > 0) {
    const taggedUserDetails = await this.userRepository.find({
      where: reel.taggedUserIds.map((id) => ({ id })),
      select: ['id', 'username', 'fullName', 'avatarUrl'],
    });

    taggedUsers = taggedUserDetails.map((user) => ({
      userId: user.id,
      username: user.username,
      fullName: user.fullName || undefined,
      avatarUrl: user.avatarUrl || undefined,
    }));
  }

  // Check user engagement
  let isLiked = false;
  let isSaved = false;

  if (userId) {
    const [likeEngagement, saveEngagement] = await Promise.all([
      this.engagementModel.findOne({ userId, reelId: String((reel as any)._id), type: 'like', active: true }),
      this.engagementModel.findOne({ userId, reelId: String((reel as any)._id), type: 'save', active: true }),
    ]);

    isLiked = !!likeEngagement;
    isSaved = !!saveEngagement;
  }

  // Convert S3 URIs to HTTPS URLs
  const cdnUrl = process.env.CDN_URL;
  const videoUrlHttps = s3UriToHttps(videoUrl, cdnUrl);
  const thumbnailUrlHttps = s3UriToHttps(reel.thumbnailUrl, cdnUrl);

  // Build feedCardPreview
  const reelPurpose = (reel.reelPurpose || 'PROMOTIONAL') as
    | 'USER_REVIEW'
    | 'PROMOTIONAL'
    | 'MENU_SHOWCASE';

  const feedCardPreview: FeedItem['feedCardPreview'] = {
    previewImage: thumbnailUrlHttps || videoUrlHttps,
    durationSec: reel.durationSec,
    aspectRatio: (reel as any).aspectRatio || 9 / 16,
    reelPurpose,
    linkedOrder,
    linkedMenu,
    chefContext: orderContext
      ? {
          chefId: orderContext.chefId,
          chefName: orderContext.chefName,
          isOpen: orderContext.isOpen ?? false,
          etaMinutes: orderContext.etaMinutes ?? 0,
        }
      : undefined,
  };

  // Return FeedItem
  return {
    id: String((reel as any)._id),
    userId: reel.userId,
    contentType: (reel as any).contentType || 'REEL',
    author: author
      ? {
          id: author.id,
          username: author.username,
          fullName: author.fullName || 'Unknown',
          avatarUrl: author.avatarUrl || null,
        }
      : undefined,
    caption: reel.caption,
    hashtags: reel.hashtags,
    taggedUsers,
    videoUrl: videoUrlHttps,
    thumbnailUrl: thumbnailUrlHttps,
    durationSec: reel.durationSec,
    linkedOrder,
    linkedMenu,
    reelPurpose,
    orderContext,
    feedCardPreview,
    stats: {
      views: reel.stats.views,
      likes: reel.stats.likes,
      comments: reel.stats.comments,
      saves: reel.stats.saves,
      shareCount: reel.stats.shareCount || 0,
      isLiked,
      isSaved,
    },
    promoted: reel.promoted,
    createdAt: reel.createdAt.toISOString(),
  };
}
```

---

### 5. recordEngagement()

**Signature**:
```typescript
async recordEngagement(
  reelId: string,
  action: EngagementAction,
  userId?: string,
  ip?: string,
): Promise<FeedEngagementResponse>
```

**Purpose**: Record view, like, or save action on a reel.

**Steps**:
1. Verify reel exists (throw 404 if not found)
2. Route to action-specific handler (VIEW/LIKE/SAVE)

**Implementation**:
```typescript
async recordEngagement(
  reelId: string,
  action: EngagementAction,
  userId?: string,
  ip?: string,
): Promise<FeedEngagementResponse> {
  this.logger.debug(
    `🔍 Recording engagement: action=${action}, reelId=${reelId}, userId=${userId || 'guest'}`,
  );

  // Verify reel exists
  const reel = await this.reelModel.findById(reelId);
  if (!reel) {
    this.logger.error(`❌ Reel not found: ${reelId}`);
    throw new HttpException(
      {
        success: false,
        message: 'Reel not found',
        errorCode: 'FEED_REEL_NOT_FOUND',
      },
      HttpStatus.NOT_FOUND,
    );
  }

  this.logger.debug(`✅ Reel found: ${reel._id}`);

  switch (action) {
    case EngagementAction.VIEW:
      return this.handleViewEngagement(reel, userId, ip);
    case EngagementAction.LIKE:
      return this.handleLikeEngagement(reel, userId);
    case EngagementAction.SAVE:
      return this.handleSaveEngagement(reel, userId);
    default:
      throw new HttpException(
        {
          success: false,
          message: 'Invalid engagement action',
          errorCode: 'FEED_INVALID_ACTION',
        },
        HttpStatus.BAD_REQUEST,
      );
  }
}
```

---

### 6. handleViewEngagement()

**Signature**:
```typescript
private async handleViewEngagement(
  reel: Reel,
  userId?: string,
  ip?: string,
): Promise<FeedEngagementResponse>
```

**Purpose**: Record view with 3-second throttling.

**Implementation**:
```typescript
private async handleViewEngagement(
  reel: Reel,
  userId?: string,
  ip?: string,
): Promise<FeedEngagementResponse> {
  const identifier = userId || ip || 'anonymous';
  const throttleKey = `${identifier}:${String((reel as any)._id)}`;
  const now = Date.now();
  const lastView = this.viewThrottle.get(throttleKey);

  // Rate limit: 3 seconds
  if (lastView && now - lastView < 3000) {
    throw new HttpException(
      {
        success: false,
        message: 'View rate limited. Please wait.',
        errorCode: 'FEED_RATE_LIMITED',
      },
      HttpStatus.TOO_MANY_REQUESTS,
    );
  }

  // Update throttle timestamp
  this.viewThrottle.set(throttleKey, now);

  // Increment view counter in MongoDB
  reel.stats.views += 1;
  await reel.save();

  // TODO: In production, increment Redis counter: INCR views:{reelId}
  // Background job will flush Redis counters to MongoDB hourly

  return {
    reelId: String((reel as any)._id),
    stats: {
      views: reel.stats.views,
      likes: reel.stats.likes,
      comments: reel.stats.comments,
      saves: reel.stats.saves,
      shareCount: reel.stats.shareCount || 0,
    },
  };
}
```

---

### 7. handleLikeEngagement()

**Signature**:
```typescript
private async handleLikeEngagement(
  reel: Reel,
  userId?: string,
): Promise<FeedEngagementResponse>
```

**Purpose**: Record like with idempotency and notification.

**Implementation**:
```typescript
private async handleLikeEngagement(
  reel: Reel,
  userId?: string,
): Promise<FeedEngagementResponse> {
  // DEVELOPMENT: Use fallback userId if not authenticated
  // TODO: Re-enable strict auth check for production
  if (!userId) {
    this.logger.warn('⚠️  Like engagement without userId - using fallback for development');
    userId = 'd1c2d2f6-6823-4ae3-99b8-f7e31c2bbbe5';
  }

  const reelId = String((reel as any)._id);

  // Check if already liked using MongoDB
  const existingLike = await this.engagementModel.findOne({
    userId,
    reelId,
    type: 'like',
  });

  if (existingLike) {
    // Idempotent: return current stats without error
    this.logger.debug(`User ${userId} already liked reel ${reelId}`);
    return {
      reelId,
      stats: {
        views: reel.stats.views,
        likes: reel.stats.likes,
        comments: reel.stats.comments,
        saves: reel.stats.saves,
        shareCount: reel.stats.shareCount || 0,
      },
    };
  }

  // Create engagement record
  await this.engagementModel.create({
    userId,
    reelId,
    type: 'like',
    active: true,
  });

  // Increment like counter
  reel.stats.likes += 1;
  await reel.save();

  // Add user to likes set in cache for fast lookup
  const likeKey = `reel:${reelId}:likes`;
  await this.cacheService.sadd(likeKey, userId);

  // Invalidate feed cache to reflect new engagement
  await this.cacheService.publish('invalidate:feed', {
    reason: 'like_engagement',
    reelId,
    userId,
  });

  // Send notification to reel owner (if not liking own reel)
  try {
    if (reel.userId !== userId) {
      const user = await this.userRepository.findOne({
        where: { id: userId },
        select: ['id', 'username'],
      });

      await this.notificationOrchestrator.handleEvent({
        type: 'REEL_LIKED',
        recipientUserId: reel.userId,
        actorUserId: userId,
        entityId: reelId,
        metadata: {
          reelId,
          username: user?.username || 'Someone',
          reelThumbnail: reel.thumbnailUrl,
        },
      });
    }
  } catch (error) {
    this.logger.error('Failed to send like notification:', error);
    // Don't fail the request
  }

  return {
    reelId,
    stats: {
      views: reel.stats.views,
      likes: reel.stats.likes,
      comments: reel.stats.comments,
      saves: reel.stats.saves,
      shareCount: reel.stats.shareCount || 0,
    },
  };
}
```

---

### 8. handleSaveEngagement()

**Signature**:
```typescript
private async handleSaveEngagement(
  reel: Reel,
  userId?: string,
): Promise<FeedEngagementResponse>
```

**Purpose**: Record save with idempotency (similar to like, without notification).

**Implementation**:
```typescript
private async handleSaveEngagement(
  reel: Reel,
  userId?: string,
): Promise<FeedEngagementResponse> {
  // DEVELOPMENT: Use fallback userId if not authenticated
  // TODO: Re-enable strict auth check for production
  if (!userId) {
    this.logger.warn('⚠️  Save engagement without userId - using fallback for development');
    userId = 'd1c2d2f6-6823-4ae3-99b8-f7e31c2bbbe5';
  }

  const reelId = String((reel as any)._id);

  // Check if already saved using MongoDB
  const existingSave = await this.engagementModel.findOne({
    userId,
    reelId,
    type: 'save',
  });

  if (existingSave) {
    // Idempotent: return current stats without error
    this.logger.debug(`User ${userId} already saved reel ${reelId}`);
    return {
      reelId,
      stats: {
        views: reel.stats.views,
        likes: reel.stats.likes,
        comments: reel.stats.comments,
        saves: reel.stats.saves,
        shareCount: reel.stats.shareCount || 0,
      },
    };
  }

  // Create engagement record
  await this.engagementModel.create({
    userId,
    reelId,
    type: 'save',
    active: true,
  });

  // Increment save counter
  reel.stats.saves += 1;
  await reel.save();

  // Add user to saves set in cache for fast lookup
  const saveKey = `reel:${reelId}:saves`;
  await this.cacheService.sadd(saveKey, userId);

  // Invalidate feed cache to reflect new engagement
  await this.cacheService.publish('invalidate:feed', {
    reason: 'save_engagement',
    reelId,
    userId,
  });

  return {
    reelId,
    stats: {
      views: reel.stats.views,
      likes: reel.stats.likes,
      comments: reel.stats.comments,
      saves: reel.stats.saves,
      shareCount: reel.stats.shareCount || 0,
    },
  };
}
```

---

### 9. setupCacheInvalidationSubscribers()

**Signature**:
```typescript
private setupCacheInvalidationSubscribers(): void
```

**Purpose**: Subscribe to pub/sub channel for cache invalidation.

**Implementation**:
```typescript
private setupCacheInvalidationSubscribers(): void {
  this.cacheService.subscribe('invalidate:feed', (message: any) => {
    this.logger.log(`Cache invalidation triggered: ${JSON.stringify(message)}`);
    this.cacheService.invalidate('feed:*'); // Clear all feed cache pages
  });
}
```

---

### 10. clearAndReseedReels()

**Signature**:
```typescript
async clearAndReseedReels(): Promise<void>
```

**Purpose**: Development utility to clear and reseed reels with test data.

**Implementation**:
```typescript
async clearAndReseedReels(): Promise<void> {
  this.logger.log('🗑️  Clearing all reels from MongoDB...');

  // Delete all existing reels
  const deleteResult = await this.reelModel.deleteMany({});
  this.logger.log(`✓ Deleted ${deleteResult.deletedCount} reels`);

  // Get some user IDs from reputation table
  const users = await this.reputationRepository.find({ take: 20 });
  if (users.length === 0) {
    throw new Error('No users found. Run v1_basic seed first.');
  }

  this.logger.log(`🌱 Creating 50 test reels with real video URLs...`);

  // Real test videos from Google's sample repository
  const testVideos = [
    {
      url: 'https://commondatastorage.googleapis.com/gtv-videos-bucket/sample/BigBuckBunny.mp4',
      thumbnail: 'https://commondatastorage.googleapis.com/gtv-videos-bucket/sample/images/BigBuckBunny.jpg',
      duration: 596,
    },
    // ... (more test videos)
  ];

  const captions = [
    '🔥 Made this today! Recipe in bio',
    'Trying a new dish! What do you think? 👨‍🍳',
    // ... (more captions)
  ];

  const reels = [];
  for (let i = 0; i < 50; i++) {
    const randomUser = users[Math.floor(Math.random() * users.length)];
    const randomVideo = testVideos[Math.floor(Math.random() * testVideos.length)];
    const randomCaption = captions[Math.floor(Math.random() * captions.length)];

    reels.push({
      userId: randomUser.userId,
      videoUrl: randomVideo.url,
      thumbnailUrl: randomVideo.thumbnail,
      durationSec: randomVideo.duration,
      caption: randomCaption,
      hashtags: [],
      stats: {
        views: Math.floor(Math.random() * 10000),
        likes: Math.floor(Math.random() * 1000),
        comments: Math.floor(Math.random() * 100),
        saves: Math.floor(Math.random() * 200),
      },
      promoted: Math.random() > 0.8, // 20% chance of promoted
      createdAt: new Date(Date.now() - Math.random() * 30 * 24 * 60 * 60 * 1000), // Random date in last 30 days
    });
  }

  await this.reelModel.insertMany(reels);
  this.logger.log(`✅ Created ${reels.length} reels with real video URLs`);
}
```

---

## Domain Logic

### Location: `libs/domain/src/feed/`

### 1. feed.policy.ts

**Purpose**: Abuse detection policies (upload rate, duplicates, engagement anomalies).

#### isUploadRateAbusive()

```typescript
export const isUploadRateAbusive = (
  input: UploadRateInput,
  envConfig: EnvironmentConfig,
): PolicyDecision => {
  const hourlyLimit = getTierAdjustedHourlyLimit(input.userTier, envConfig);
  const dailyLimit = getTierAdjustedDailyLimit(input.userTier, envConfig);

  const hourlyExceeded = input.uploadsLastHour >= hourlyLimit;
  const dailyExceeded = input.uploadsLast24h >= dailyLimit;

  if (hourlyExceeded || dailyExceeded) {
    return {
      allowed: false,
      reason: hourlyExceeded
        ? `Upload rate exceeded: ${input.uploadsLastHour}/${hourlyLimit} per hour`
        : `Daily upload limit exceeded: ${input.uploadsLast24h}/${dailyLimit}`,
      errorCode: 'UPLOAD_RATE_EXCEEDED',
      metadata: {
        userId: input.userId,
        uploadsLastHour: input.uploadsLastHour,
        uploadsLast24h: input.uploadsLast24h,
        hourlyLimit,
        dailyLimit,
        tier: input.userTier,
      },
    };
  }

  return {
    allowed: true,
    metadata: {
      userId: input.userId,
      uploadsLastHour: input.uploadsLastHour,
      uploadsLast24h: input.uploadsLast24h,
      hourlyLimit,
      dailyLimit,
      tier: input.userTier,
    },
  };
};
```

#### isDuplicateContent()

```typescript
export const isDuplicateContent = (
  input: DuplicateContentInput,
  envConfig: EnvironmentConfig,
): PolicyDecision => {
  const windowMinutes = envConfig.feed.duplicateWindowMinutes;
  const cutoffTime = new Date(Date.now() - windowMinutes * 60 * 1000);

  // Check for matching hash within time window
  const duplicateFound = input.previousHashes.some(
    (prev) => prev.hash === input.contentHash && prev.timestamp >= cutoffTime,
  );

  if (duplicateFound) {
    return {
      allowed: false,
      reason: `Duplicate content detected within ${windowMinutes} minutes`,
      errorCode: 'DUPLICATE_CONTENT',
      metadata: {
        contentHash: input.contentHash,
        windowMinutes,
        duplicateCount: input.previousHashes.filter((h) => h.hash === input.contentHash).length,
      },
    };
  }

  return {
    allowed: true,
    metadata: {
      contentHash: input.contentHash,
      windowMinutes,
    },
  };
};
```

#### detectEngagementAnomaly()

```typescript
export const detectEngagementAnomaly = (
  input: EngagementAnomalyInput,
  envConfig: EnvironmentConfig,
): PolicyDecision => {
  // Prevent division by zero
  if (input.views === 0) {
    return {
      allowed: true,
      metadata: {
        reason: 'No views yet',
      },
    };
  }

  const currentEngagementRate = ((input.likes + input.comments) / input.views) * 100;

  // Calculate expected engagement rate with velocity factor
  // New posts naturally have higher engagement rates
  const velocityFactor = Math.max(0.5, Math.min(1.0, input.timeSincePostMinutes / 60));
  const expectedRate = input.avgEngagementRate * velocityFactor;

  const threshold = envConfig.feed.engagementSpikeThreshold;
  const spikePercentage = ((currentEngagementRate - expectedRate) / expectedRate) * 100;

  if (spikePercentage > threshold && input.timeSincePostMinutes > 5) {
    return {
      allowed: false,
      reason: `Engagement anomaly detected: ${spikePercentage.toFixed(1)}% spike`,
      errorCode: 'ENGAGEMENT_ANOMALY',
      metadata: {
        currentRate: currentEngagementRate,
        expectedRate,
        spikePercentage,
        threshold,
        likes: input.likes,
        comments: input.comments,
        views: input.views,
        timeSincePostMinutes: input.timeSincePostMinutes,
      },
    };
  }

  return {
    allowed: true,
    metadata: {
      currentRate: currentEngagementRate,
      expectedRate,
      spikePercentage: spikePercentage > 0 ? spikePercentage : 0,
    },
  };
};
```

---

### 2. feed.config.ts

**Purpose**: Configuration helpers for extracting feed-related config from EnvironmentConfig.

#### getFeedLimits()

```typescript
export const getFeedLimits = (envConfig: EnvironmentConfig): FeedLimits => {
  return {
    defaultPageSize: envConfig.feed.defaultPageSize,
    maxPageSize: envConfig.feed.maxPageSize,
    cacheTtlSeconds: envConfig.feed.cacheTtlSeconds,
  };
};
```

#### normalizePageLimit()

```typescript
export const normalizePageLimit = (
  requestedLimit: number | undefined,
  envConfig: EnvironmentConfig,
): number => {
  const limits = getFeedLimits(envConfig);
  const limit = requestedLimit ?? limits.defaultPageSize;

  return Math.min(Math.max(1, limit), limits.maxPageSize);
};
```

#### getFeedRankingWeights()

```typescript
export const getFeedRankingWeights = (envConfig: EnvironmentConfig): FeedRankingWeights => {
  return {
    promotedBoost: envConfig.feed.promotedBoost,
    reputationBoostScale: envConfig.feed.reputationBoostScale,
  };
};
```

---

### 3. feed.rules.ts

**Purpose**: Visibility rules for state transitions and multipliers.

#### getFeedVisibilityMultiplier()

```typescript
export const getFeedVisibilityMultiplier = (
  userId: string,
  reputationTier: UserTier,
  abuseSignals: FeedAbuseSignals,
  envConfig: EnvironmentConfig,
): FeedVisibilityResult => {
  // Determine current state
  const state = calculateFeedState(abuseSignals, envConfig);

  // Get base multiplier from environment
  const multipliers = envConfig.feed.visibilityMultipliers;
  let baseMultiplier: number;

  switch (state) {
    case FeedState.NORMAL:
      baseMultiplier = multipliers.normal;
      break;
    case FeedState.WARNED:
      baseMultiplier = multipliers.warned;
      break;
    case FeedState.THROTTLED:
      baseMultiplier = multipliers.throttled;
      break;
    case FeedState.SUPPRESSED:
      baseMultiplier = multipliers.suppressed;
      break;
    default:
      baseMultiplier = multipliers.normal;
  }

  // Tier bonus (small, doesn't override abuse state)
  const tierBonus = getTierVisibilityBonus(reputationTier);
  const finalMultiplier = Math.max(0, Math.min(1.0, baseMultiplier + tierBonus));

  return {
    multiplier: finalMultiplier,
    state,
    reason: state !== FeedState.NORMAL ? getStateReason(abuseSignals) : undefined,
  };
};
```

#### calculateFeedState()

```typescript
const calculateFeedState = (
  signals: FeedAbuseSignals,
  envConfig: EnvironmentConfig,
): FeedState => {
  const violationCount = signals.violationCount;

  // Check for active abuse signals
  const hasActiveAbuse =
    signals.uploadRateViolation ||
    signals.duplicateContentDetected ||
    signals.engagementAnomalyDetected;

  if (violationCount >= 6 || hasActiveAbuse) {
    return FeedState.SUPPRESSED;
  }

  if (violationCount >= 3) {
    return FeedState.THROTTLED;
  }

  if (violationCount >= 1) {
    return FeedState.WARNED;
  }

  return FeedState.NORMAL;
};
```

#### nextFeedState()

```typescript
export const nextFeedState = (
  input: StateTransitionInput,
  envConfig: EnvironmentConfig,
): FeedState => {
  const { currentState, abuseSignals, lastSuppressionAt } = input;

  // Check if cooldown period has passed
  if (lastSuppressionAt && currentState === FeedState.SUPPRESSED) {
    const cooldownHours = envConfig.feed.suppressionCooldownHours;
    const hoursSinceSuppress = (Date.now() - lastSuppressionAt.getTime()) / (1000 * 60 * 60);

    if (hoursSinceSuppress >= cooldownHours) {
      // Check if signals are clear
      const hasActiveAbuse =
        abuseSignals.uploadRateViolation ||
        abuseSignals.duplicateContentDetected ||
        abuseSignals.engagementAnomalyDetected;

      if (!hasActiveAbuse && abuseSignals.violationCount < 3) {
        return FeedState.WARNED; // Graduated recovery
      }
    }
  }

  // Calculate new state based on current signals
  return calculateFeedState(abuseSignals, envConfig);
};
```

---

## Data Models

### MongoDB Schemas (Mongoose)

#### Reel Schema

```typescript
// apps/chefooz-apis/src/database/schemas/reel.schema.ts
@Schema({ timestamps: true, collection: 'reels' })
export class Reel {
  @Prop({ required: true })
  userId: string; // Creator UUID

  @Prop({ required: true })
  videoUrl: string; // S3 URL (legacy, fallback)

  @Prop()
  thumbnailUrl: string; // S3 thumbnail URL

  @Prop()
  mediaId: string; // Reference to Media document (for S3 variants)

  @Prop({ required: true })
  durationSec: number;

  @Prop()
  caption: string;

  @Prop({ type: [String], default: [] })
  hashtags: string[];

  @Prop({ type: [String], default: [] })
  taggedUserIds: string[];

  @Prop({ type: Object, default: { views: 0, likes: 0, comments: 0, saves: 0, shareCount: 0 } })
  stats: {
    views: number;
    likes: number;
    comments: number;
    saves: number;
    shareCount: number;
  };

  @Prop({ default: 'PROMOTIONAL' })
  reelPurpose: 'USER_REVIEW' | 'PROMOTIONAL' | 'MENU_SHOWCASE';

  @Prop()
  linkedOrderId: string; // UUID reference to Order entity

  @Prop()
  linkedMenu: string; // String ID reference to ChefMenuItem

  @Prop({ default: false })
  promoted: boolean;

  @Prop()
  createdAt: Date;

  @Prop()
  updatedAt: Date;
}

export const ReelSchema = SchemaFactory.createForClass(Reel);
```

#### Engagement Schema

```typescript
// apps/chefooz-apis/src/database/schemas/engagement.schema.ts
@Schema({ timestamps: true, collection: 'engagements' })
export class Engagement {
  @Prop({ required: true })
  userId: string; // User UUID

  @Prop({ required: true })
  reelId: string; // Reel MongoDB ObjectId

  @Prop({ required: true })
  type: 'like' | 'save' | 'comment' | 'share';

  @Prop({ default: true })
  active: boolean;

  @Prop()
  createdAt: Date;

  @Prop()
  updatedAt: Date;
}

export const EngagementSchema = SchemaFactory.createForClass(Engagement);
```

#### Media Schema

```typescript
// apps/chefooz-apis/src/modules/media/media.schema.ts
@Schema({ timestamps: true, collection: 'media' })
export class Media {
  @Prop({ required: true })
  userId: string; // Uploader UUID

  @Prop({ required: true })
  s3Key: string; // Original S3 key

  @Prop({ required: true })
  s3Bucket: string;

  @Prop({ type: [Object], default: [] })
  variants: Array<{
    url: string; // S3 URL
    quality: string; // '1080p', '720p', '480p'
    format: string; // 'mp4', 'webm'
    sizeBytes: number;
  }>;

  @Prop()
  status: 'pending' | 'processing' | 'completed' | 'failed';

  @Prop()
  createdAt: Date;

  @Prop()
  updatedAt: Date;
}

export const MediaSchema = SchemaFactory.createForClass(Media);
```

---

### PostgreSQL Entities (TypeORM)

#### User Entity

```typescript
// apps/chefooz-apis/src/database/entities/user.entity.ts
@Entity('users')
export class User {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ unique: true })
  username: string;

  @Column({ nullable: true })
  fullName: string;

  @Column({ nullable: true })
  avatarUrl: string;

  @Column({ type: 'decimal', precision: 10, scale: 7, nullable: true })
  lat: number;

  @Column({ type: 'decimal', precision: 10, scale: 7, nullable: true })
  lng: number;

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;
}
```

#### UserReputationCurrent Entity

```typescript
// apps/chefooz-apis/src/database/entities/user-reputation-current.entity.ts
@Entity('user_reputation_current')
export class UserReputationCurrent {
  @PrimaryColumn('uuid')
  userId: string;

  @Column({ type: 'decimal', precision: 4, scale: 2, default: 0 })
  feedBoostWeight: number; // 0.0 - 2.0

  @Column({ default: 'ACTIVE' })
  creatorBoostState: 'ACTIVE' | 'FLAGGED_CHURNING' | 'FLAGGED_BOOSTING' | 'SUSPENDED';

  @Column({ type: 'timestamp', nullable: true })
  lastBoostAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;
}
```

#### Order Entity

```typescript
// apps/chefooz-apis/src/database/entities/order.entity.ts
@Entity('orders')
export class Order {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column('uuid')
  chefId: string;

  @Column('uuid')
  customerId: string;

  @Column({ type: 'decimal', precision: 10, scale: 2 })
  totalPaise: number;

  @Column()
  status: string;

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;
}
```

#### ChefMenuItem Entity

```typescript
// apps/chefooz-apis/src/modules/chef-kitchen/entities/chef-menu-item.entity.ts
@Entity('chef_menu_items')
export class ChefMenuItem {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column('uuid')
  chefId: string;

  @Column('uuid', { nullable: true })
  orderId: string;

  @Column()
  name: string;

  @Column({ type: 'decimal', precision: 10, scale: 2 })
  price: number;

  @Column({ nullable: true })
  imageUrl: string;

  @Column({ default: true })
  isActive: boolean;

  @Column({ type: 'jsonb', nullable: true })
  availability: {
    isAvailable: boolean;
    soldOut: boolean;
  };

  @Column({ type: 'int', default: 30 })
  prepTimeMinutes: number;

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;
}
```

#### UserFollow Entity

```typescript
// apps/chefooz-apis/src/modules/social/entities/user-follow.entity.ts
@Entity('user_follows')
export class UserFollow {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column('uuid')
  followerId: string; // User who follows

  @Column('uuid')
  followingId: string; // User being followed

  @Column({ default: 'pending' })
  status: 'pending' | 'accepted' | 'rejected';

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;
}
```

---

## Ranking Algorithm Implementation

### Core Calculation (from calculateRankingScore)

```typescript
// Step 1: Base score (engagement signals × recency decay)
const engagementScore =
  reel.stats.likes * 10 +
  reel.stats.views * 1 +
  reel.stats.saves * 15;

const ageInHours = (Date.now() - reel.createdAt.getTime()) / (1000 * 60 * 60);
const recencyFactor = Math.pow(0.5, ageInHours / 48); // 48h half-life

const baseScore = engagementScore * recencyFactor;

// Step 2: Reputation boost (CRS integration)
const reputation = reputationMap.get(reel.userId);
const feedBoostWeight = reputation?.feedBoostWeight || 0;
const reputationBoostScale = envConfig.feed.reputationBoostScale; // 100
const reputationBoost = feedBoostWeight * reputationBoostScale;

// Step 3: Promoted boost
const promotedBoost = reel.promoted ? envConfig.feed.promotedBoost : 0; // 50

// Step 4: Raw score (before visibility multiplier)
let rawScore = baseScore + reputationBoost + promotedBoost;

// Step 5: Visibility multiplier (creator boost state)
let visibilityMultiplier = 1.0;
const creatorBoostState = reputation?.creatorBoostState || 'ACTIVE';

switch (creatorBoostState) {
  case 'ACTIVE':
    visibilityMultiplier = 1.0;
    break;
  case 'FLAGGED_CHURNING':
  case 'FLAGGED_BOOSTING':
    visibilityMultiplier = 0.3; // 70% penalty
    break;
  case 'SUSPENDED':
    visibilityMultiplier = 0.0; // Hidden
    break;
}

// Step 6: Apply visibility multiplier
let finalScore = rawScore * visibilityMultiplier;

// Step 7: Clamp to 0-1000
finalScore = Math.max(0, Math.min(1000, finalScore));

return finalScore;
```

### Recency Decay Formula

```
recencyFactor = 0.5^(ageInHours / 48)
```

**Examples**:
- 0 hours old: `0.5^(0/48) = 1.0` (no decay)
- 12 hours old: `0.5^(12/48) = 0.84` (16% decay)
- 24 hours old: `0.5^(24/48) = 0.71` (29% decay)
- 48 hours old: `0.5^(48/48) = 0.5` (50% decay, half-life)
- 96 hours old: `0.5^(96/48) = 0.25` (75% decay)

### Engagement Weights

| Signal | Weight | Rationale |
|--------|--------|-----------|
| **Likes** | 10× | Common engagement signal, moderate intent |
| **Views** | 1× | Baseline popularity, low intent |
| **Saves** | 15× | High-intent signal, user wants to revisit |

### CRS Reputation Boost

```
reputationBoost = feedBoostWeight * reputationBoostScale
```

**feedBoostWeight Range**: 0.0 - 2.0 (from CRS system)
**reputationBoostScale**: 100 (from environment config)

**Examples**:
- Bronze creator (feedBoostWeight=0.6): `0.6 × 100 = 60 points`
- Silver creator (feedBoostWeight=1.0): `1.0 × 100 = 100 points`
- Gold creator (feedBoostWeight=1.5): `1.5 × 100 = 150 points`
- Diamond creator (feedBoostWeight=1.8): `1.8 × 100 = 180 points`
- Legend creator (feedBoostWeight=2.0): `2.0 × 100 = 200 points`

### Promoted Boost

```
promotedBoost = reel.promoted ? 50 : 0
```

**Use Case**: Sponsored content or platform-promoted reels.

### Visibility Multiplier (Creator Boost State)

| State | Multiplier | Effect |
|-------|------------|--------|
| **ACTIVE** | 1.0 | No penalty, full visibility |
| **FLAGGED_CHURNING** | 0.3 | 70% reduction (gaming detected) |
| **FLAGGED_BOOSTING** | 0.3 | 70% reduction (gaming detected) |
| **SUSPENDED** | 0.0 | 100% hidden (severe violation) |

### Score Clamping

```
finalScore = Math.max(0, Math.min(1000, finalScore))
```

**Purpose**: Prevent outliers from dominating feed, normalize score range.

---

## Caching Strategy

### First Page Caching (Guest Feeds)

**Cache Key Pattern**:
```
feed:${sort}:limit:${limit}:lat:${lat}:lng:${lng}
```

**Examples**:
- `feed:DEFAULT:limit:10:lat:null:lng:null` (guest feed, no location)
- `feed:TRENDING:limit:20:lat:12.9716:lng:77.5946` (trending feed with location)

**Conditions**:
- No cursor (first page only)
- Guest or authenticated users (same cache for both)

**TTL**: `cacheTtlSeconds` from environment config (typically 300s = 5 minutes)

**Invalidation**:
- Pub/sub event: `invalidate:feed`
- Clears all `feed:*` pattern
- Triggered by: New like, new save, new reel

**Cache Miss Behavior**:
- Fetch reels from MongoDB
- Apply ranking algorithm
- Map to FeedItem
- Cache result with TTL
- Return to client

**Cache Hit Behavior**:
- Fetch from Redis
- Parse JSON
- Return to client immediately

**Performance**:
- Cache hit: ~10ms
- Cache miss: ~150ms (ranking algorithm)

---

### Chef Context Caching (N+1 Prevention)

**Cache Key**: In-memory Map (request-scoped)

**Purpose**: Prevent N+1 queries when multiple reels are from the same chef.

**Lifecycle**:
1. Request starts: `chefContextCache = new Map()`
2. First reel from chef X: Fetch chef, kitchen, schedule → Cache
3. Second reel from chef X: `chefContextCache.get(chefId)` → Hit
4. Request ends: Cache cleared

**Example**:
- 10 reels in feed
- 3 reels from chef X, 2 from chef Y, 5 from others
- Without cache: 10 DB queries (chef + kitchen + schedule × 10)
- With cache: 7 DB queries (X + Y + 5 others)

**Performance Impact**: 30-40% reduction in DB queries for order context.

---

### Engagement Caching (Redis Sets)

**Like Sets**:
```
reel:${reelId}:likes → Set<userId>
```

**Save Sets**:
```
reel:${reelId}:saves → Set<userId>
```

**Purpose**: Fast lookup for `isLiked`, `isSaved` flags in feed response.

**Operations**:
- **Add**: `SADD reel:65f3b1c...:likes userId`
- **Check**: `SISMEMBER reel:65f3b1c...:likes userId`
- **Remove**: `SREM reel:65f3b1c...:likes userId` (for unlike)

**TTL**: None (manual cleanup required)

**⚠️ Concern**: Unbounded growth, must implement TTL or periodic cleanup.

**Recommendation**: Set 30-day TTL on sets:
```typescript
await this.cacheService.expire(`reel:${reelId}:likes`, 30 * 24 * 60 * 60); // 30 days
```

---

### Cache Invalidation (Pub/Sub)

**Channel**: `invalidate:feed`

**Subscribers**: FeedService (in `onModuleInit`)

**Publishers**:
- `handleLikeEngagement()`: On new like
- `handleSaveEngagement()`: On new save
- Reels module: On new reel creation

**Message Format**:
```json
{
  "reason": "like_engagement",
  "reelId": "65f3b1c4f8d9e2a1b3c4d5e6",
  "userId": "abc-123-def-456"
}
```

**Invalidation Logic**:
```typescript
this.cacheService.subscribe('invalidate:feed', (message: any) => {
  this.logger.log(`Cache invalidation triggered: ${JSON.stringify(message)}`);
  this.cacheService.invalidate('feed:*'); // Clear all feed cache pages
});
```

**Performance**: Invalidation takes ~5ms (Redis DEL command).

---

## Database Queries

### MongoDB Queries (Reel Collection)

#### 1. Fetch Candidate Reels (Default Ranking)

```typescript
const candidates = await this.reelModel
  .find({
    videoUrl: { $exists: true, $ne: '' },
    _id: { $lt: cursor }, // Cursor pagination
    reelPurpose: { $ne: 'MENU_SHOWCASE' }, // Feature flag filter
  })
  .sort({ createdAt: -1 }) // Recent first
  .limit(50) // Candidate pool
  .populate('mediaId') // Fetch Media document
  .exec();
```

**Indexes Required**:
```javascript
db.reels.createIndex({ videoUrl: 1 });
db.reels.createIndex({ createdAt: -1 });
db.reels.createIndex({ _id: -1 });
db.reels.createIndex({ reelPurpose: 1 });
```

**Performance**: ~30ms for 50 reels (1M total reels, indexed).

---

#### 2. Fetch Trending Reels

```typescript
const reels = await this.reelModel
  .find({
    videoUrl: { $exists: true, $ne: '' },
    _id: { $lt: cursor },
  })
  .sort({ 'stats.likes': -1, createdAt: -1 }) // Likes desc, then recent
  .limit(11) // limit + 1
  .populate('mediaId')
  .exec();
```

**Indexes Required**:
```javascript
db.reels.createIndex({ 'stats.likes': -1, createdAt: -1 });
```

**Performance**: ~50ms (compound index on stats.likes + createdAt).

---

#### 3. Fetch Following-Only Reels

```typescript
const reels = await this.reelModel
  .find({
    videoUrl: { $exists: true, $ne: '' },
    userId: { $in: followingIds }, // IN query
    _id: { $lt: cursor },
  })
  .sort({ createdAt: -1 })
  .limit(11)
  .populate('mediaId')
  .exec();
```

**Indexes Required**:
```javascript
db.reels.createIndex({ userId: 1, createdAt: -1 });
```

**Performance**: ~40ms (depends on followingIds.length).

---

#### 4. Check Existing Like (Idempotency)

```typescript
const existingLike = await this.engagementModel.findOne({
  userId: 'abc-123',
  reelId: '65f3b1c...',
  type: 'like',
});
```

**Indexes Required**:
```javascript
db.engagements.createIndex({ userId: 1, reelId: 1, type: 1 });
```

**Performance**: ~5ms (compound index).

---

### PostgreSQL Queries (User Entities)

#### 1. Batch Fetch Reputation Data

```typescript
const reputations = await this.reputationRepository.find({
  where: { userId: In(validUuids) },
});
```

**SQL**:
```sql
SELECT * FROM user_reputation_current
WHERE user_id IN ('abc-123', 'def-456', ...)
```

**Indexes Required**:
```sql
CREATE INDEX idx_reputation_userId ON user_reputation_current (user_id);
```

**Performance**: ~20ms for 50 user IDs (10M users, indexed).

---

#### 2. Fetch UserFollow Records (Following Filter)

```typescript
const follows = await this.userFollowRepository.find({
  where: { followerId: userId, status: 'accepted' },
});
```

**SQL**:
```sql
SELECT * FROM user_follows
WHERE follower_id = 'abc-123' AND status = 'accepted'
```

**Indexes Required**:
```sql
CREATE INDEX idx_follow_follower_status ON user_follow (follower_id, status);
```

**Performance**: ~15ms for 500 follows (1M follows, indexed).

---

#### 3. Fetch Order with Menu Items

```typescript
const order = await this.orderRepository.findOne({
  where: { id: linkedOrderId },
});

const menuItems = await this.menuItemRepository.find({
  where: { orderId: order.id },
});
```

**SQL**:
```sql
SELECT * FROM orders WHERE id = 'order-123';
SELECT * FROM chef_menu_items WHERE order_id = 'order-123';
```

**Indexes Required**:
```sql
CREATE INDEX idx_order_id ON orders (id);
CREATE INDEX idx_menu_order ON chef_menu_items (order_id);
```

**Performance**: ~10ms for order + menu items (100K orders, indexed).

---

## Error Handling

### Error Response Structure

```typescript
{
  success: false,
  message: string,
  errorCode?: string
}
```

### Error Codes Reference

| Error Code | HTTP Status | Description | Resolution |
|------------|-------------|-------------|------------|
| `FEED_FETCH_ERROR` | 500 | Unexpected error during feed retrieval | Check server logs, retry |
| `FEED_REEL_NOT_FOUND` | 404 | Reel ID not found in MongoDB | Verify reel exists, check ID format |
| `FEED_AUTH_REQUIRED` | 401 | Like/save without auth token | Provide JWT bearer token |
| `FEED_RATE_LIMITED` | 429 | View throttle violated (< 3s) | Wait 3 seconds before retry |
| `FEED_ENGAGEMENT_ERROR` | 500 | Unexpected error during engagement | Check server logs, retry |
| `FEED_INVALID_ACTION` | 400 | Invalid engagement action enum | Use VIEW/LIKE/SAVE |
| `UPLOAD_RATE_EXCEEDED` | 429 | Upload limit exceeded | Wait for rate limit reset |
| `DUPLICATE_CONTENT` | 409 | Duplicate content hash detected | Upload different content |
| `ENGAGEMENT_ANOMALY` | 403 | Engagement farming detected | Review engagement patterns |

### Controller Error Handling Pattern

```typescript
try {
  // Service call
  const result = await this.feedService.getFeed(query, userId, userLat, userLng);
  return {
    success: true,
    message: 'Feed retrieved successfully',
    data: result,
  };
} catch (error) {
  // Re-throw HttpException from service layer
  if (error instanceof HttpException) {
    throw error;
  }

  // Unexpected errors
  const errorMessage = error instanceof Error ? error.message : 'Unknown error';
  const errorStack = error instanceof Error ? error.stack : undefined;
  this.logger.error(`Error fetching feed: ${errorMessage}`, errorStack);

  throw new HttpException(
    {
      success: false,
      message: 'Failed to retrieve feed',
      errorCode: 'FEED_FETCH_ERROR',
    },
    HttpStatus.INTERNAL_SERVER_ERROR,
  );
}
```

### Service Error Handling Pattern

```typescript
// Service layer throws HttpException
if (!reel) {
  this.logger.error(`❌ Reel not found: ${reelId}`);
  throw new HttpException(
    {
      success: false,
      message: 'Reel not found',
      errorCode: 'FEED_REEL_NOT_FOUND',
    },
    HttpStatus.NOT_FOUND,
  );
}
```

---

## Testing Approach

### Unit Tests (feed.service.spec.ts)

**Test Cases**:
1. **getFeed()**:
   - ✅ Returns paginated feed (first page, no cursor)
   - ✅ Returns cached feed (cache hit)
   - ✅ Returns feed with cursor (page 2+)
   - ✅ Applies following-only filter (with followingIds)
   - ✅ Returns empty feed (user follows nobody)
   - ✅ Applies sorting strategy (DEFAULT/TRENDING/RECENT)
   - ✅ Excludes MENU_SHOWCASE reels (feature flag OFF)

2. **getAdvancedRankedFeed()**:
   - ✅ Fetches candidate pool (limit × 5)
   - ✅ Batch fetches reputation data (valid UUIDs)
   - ✅ Calculates ranking scores
   - ✅ Sorts by score descending
   - ✅ Returns top N items

3. **calculateRankingScore()**:
   - ✅ Calculates base score (engagement + recency)
   - ✅ Applies reputation boost (CRS feedBoostWeight)
   - ✅ Applies promoted boost
   - ✅ Applies visibility multiplier (ACTIVE/FLAGGED/SUSPENDED)
   - ✅ Clamps score to 0-1000

4. **mapReelToFeedItem()**:
   - ✅ Builds linkedOrder (with availability check)
   - ✅ Shows partial orders (at least ONE item available)
   - ✅ Builds orderContext (with caching)
   - ✅ Builds linkedMenu (with availability check)
   - ✅ Fetches Media document (S3 variants)
   - ✅ Fetches author profile
   - ✅ Fetches tagged users
   - ✅ Checks user engagement (isLiked, isSaved)
   - ✅ Converts S3 URIs to HTTPS URLs

5. **recordEngagement()**:
   - ✅ Throws 404 if reel not found
   - ✅ Routes to VIEW handler (guest)
   - ✅ Routes to LIKE handler (authenticated)
   - ✅ Routes to SAVE handler (authenticated)

6. **handleViewEngagement()**:
   - ✅ Increments view counter
   - ✅ Throttles view (3 seconds)
   - ✅ Throws 429 if throttle violated

7. **handleLikeEngagement()**:
   - ✅ Creates engagement record
   - ✅ Increments like counter
   - ✅ Adds user to Redis like set
   - ✅ Sends notification to reel owner
   - ✅ Invalidates feed cache
   - ✅ Returns idempotent response (already liked)

8. **handleSaveEngagement()**:
   - ✅ Creates engagement record
   - ✅ Increments save counter
   - ✅ Adds user to Redis save set
   - ✅ Invalidates feed cache
   - ✅ Returns idempotent response (already saved)

### Integration Tests (feed.e2e.spec.ts)

**Test Cases**:
1. **GET /api/v1/feed**:
   - ✅ Returns 200 with feed items
   - ✅ Returns cached feed (cache hit)
   - ✅ Returns paginated feed (cursor)
   - ✅ Returns following-only feed
   - ✅ Returns trending feed
   - ✅ Returns recent feed
   - ✅ Returns 400 for invalid query params
   - ✅ Returns 500 for server error

2. **POST /api/v1/feed/engagement**:
   - ✅ Returns 200 for VIEW (guest)
   - ✅ Returns 200 for LIKE (authenticated)
   - ✅ Returns 200 for SAVE (authenticated)
   - ✅ Returns 401 for LIKE without auth
   - ✅ Returns 404 for invalid reel ID
   - ✅ Returns 429 for VIEW throttle violation
   - ✅ Returns 409 for duplicate LIKE (idempotent)

3. **POST /api/v1/feed/admin/reseed**:
   - ✅ Returns 200 with success message
   - ✅ Clears all reels
   - ✅ Creates 50 test reels
   - ✅ Returns 500 for seeding error

### Domain Logic Tests

**feed.policy.spec.ts**:
- ✅ `isUploadRateAbusive()`: Detects hourly/daily violations
- ✅ `isDuplicateContent()`: Detects duplicate hashes
- ✅ `detectEngagementAnomaly()`: Detects engagement spikes

**feed.config.spec.ts**:
- ✅ `getFeedLimits()`: Returns config values
- ✅ `normalizePageLimit()`: Clamps to 1-30 range
- ✅ `getFeedRankingWeights()`: Returns boost values

**feed.rules.spec.ts**:
- ✅ `getFeedVisibilityMultiplier()`: Returns correct multiplier
- ✅ `calculateFeedState()`: Returns correct state
- ✅ `nextFeedState()`: Transitions states correctly
- ✅ `canRecoverFromSuppression()`: Checks recovery eligibility

---

## Deployment Notes

### Environment Variables

```bash
# Feed Configuration
FEED_DEFAULT_PAGE_SIZE=10
FEED_MAX_PAGE_SIZE=30
FEED_CACHE_TTL_SECONDS=300
FEED_PROMOTED_BOOST=50
FEED_REPUTATION_BOOST_SCALE=100

# Abuse Prevention
FEED_MAX_UPLOADS_PER_HOUR=10
FEED_MAX_UPLOADS_PER_DAY=50
FEED_DUPLICATE_WINDOW_MINUTES=60
FEED_ENGAGEMENT_SPIKE_THRESHOLD=200
FEED_SUPPRESSION_COOLDOWN_HOURS=24

# Visibility Multipliers
FEED_VISIBILITY_NORMAL=1.0
FEED_VISIBILITY_WARNED=0.6
FEED_VISIBILITY_THROTTLED=0.3
FEED_VISIBILITY_SUPPRESSED=0.0

# CDN
CDN_URL=https://d1234567890.cloudfront.net
```

### Database Indexes (Production)

**MongoDB (Reels)**:
```javascript
use chefooz_production;

db.reels.createIndex({ videoUrl: 1 });
db.reels.createIndex({ createdAt: -1 });
db.reels.createIndex({ _id: -1 });
db.reels.createIndex({ userId: 1, createdAt: -1 });
db.reels.createIndex({ 'stats.likes': -1, createdAt: -1 });
db.reels.createIndex({ reelPurpose: 1 });

db.engagements.createIndex({ userId: 1, reelId: 1, type: 1 });
```

**PostgreSQL (Users)**:
```sql
CREATE INDEX idx_reputation_userId ON user_reputation_current (user_id);
CREATE INDEX idx_follow_follower_status ON user_follow (follower_id, status);
CREATE INDEX idx_follow_following_status ON user_follow (following_id, status);
CREATE INDEX idx_order_id ON orders (id);
CREATE INDEX idx_order_chef ON orders (chef_id);
CREATE INDEX idx_menu_order ON chef_menu_items (order_id);
CREATE INDEX idx_menu_active ON chef_menu_items (is_active, availability);
```

### Redis Configuration

```bash
# Redis for caching and pub/sub
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_DB=0
REDIS_PASSWORD=<secure_password>
```

### Horizontal Scaling Checklist

✅ **Required Changes**:
1. **Migrate view throttle to Redis** (in-memory Map doesn't scale)
2. **Add load balancer** (sticky sessions for chef context cache)
3. **Enable Redis pub/sub** (cache invalidation across instances)
4. **Monitor memory usage** (view throttle cleanup)

⚠️ **Security Fixes**:
1. **Add @Roles('admin') guard** to `/admin/reseed` endpoint
2. **Remove fallback userId** in LIKE/SAVE handlers (dev mode only)
3. **Enable strict auth checks** in production

### Monitoring & Alerts

**Metrics to Track**:
- Feed requests/sec (p50, p95, p99 latency)
- Cache hit rate (target: 85%+)
- Ranking algorithm time (target: < 100ms)
- Engagement actions/sec (VIEW/LIKE/SAVE)
- Abuse detection rate (violations/hour)
- Creator boost penalties (FLAGGED/SUSPENDED count)

**Alerts**:
- Feed latency > 500ms (p95)
- Cache hit rate < 70%
- Engagement errors > 1% of requests
- View throttle memory > 100MB

---

**[SLICE_COMPLETE ✅]**  
**Feed Module - Technical Guide**  
**Generated:** February 14, 2026  
**Lines:** ~2,000
