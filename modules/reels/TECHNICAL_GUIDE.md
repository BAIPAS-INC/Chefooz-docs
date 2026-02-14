# Reels Module - Technical Guide

**Module**: `apps/chefooz-apis/src/modules/reels`  
**Tech Stack**: NestJS, MongoDB (Mongoose), PostgreSQL (TypeORM), Redis (Valkey)  
**Version**: 1.0  
**Last Updated**: 2026-02-14

---

## 📑 Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Module Structure](#module-structure)
3. [Database Schema](#database-schema)
4. [API Endpoints](#api-endpoints)
5. [Service Layer](#service-layer)
6. [Data Transfer Objects](#data-transfer-objects)
7. [Integration Patterns](#integration-patterns)
8. [Error Handling](#error-handling)
9. [Testing Strategy](#testing-strategy)
10. [Deployment](#deployment)
11. [Monitoring & Observability](#monitoring--observability)
12. [Performance Optimization](#performance-optimization)
13. [Security Considerations](#security-considerations)
14. [Troubleshooting](#troubleshooting)

---

## 🏗️ Architecture Overview

### High-Level Architecture

```
┌─────────────────┐
│   Mobile App    │ (Expo React Native)
└────────┬────────┘
         │ HTTPS
         ▼
┌─────────────────────────────────────────┐
│         API Gateway / Load Balancer      │
└────────┬────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────┐
│      NestJS Reels Controller            │
│  - GET /detail/:mediaId                 │
│  - DELETE /:reelId                      │
│  - POST /menu                           │
│  - GET /user/:userId                    │
│  - GET /chef/:chefId/menu               │
└────────┬────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────┐
│         ReelsService (Business Logic)    │
│  - getReelDetail()                      │
│  - deleteReel()                         │
│  - createMenuReel()                     │
│  - getUserReels()                       │
│  - getChefMenuReels()                   │
└────┬────────────────────┬───────────────┘
     │                    │
     ▼                    ▼
┌─────────────┐    ┌─────────────────┐
│  MongoDB    │    │   PostgreSQL    │
│   (Reels)   │    │ (Users, Orders) │
└─────────────┘    └─────────────────┘
     │
     ▼
┌─────────────┐
│   Valkey    │ (Like/Save Cache)
│  (Redis)    │
└─────────────┘
```

### Tech Stack

| Component | Technology | Purpose |
|-----------|------------|---------|
| **Framework** | NestJS 10.x | REST API framework with TypeScript |
| **Primary DB** | MongoDB 6.x | Flexible schema for reel documents |
| **Secondary DB** | PostgreSQL 15.x | Relational data (users, orders) |
| **Cache** | Valkey/Redis 7.x | Like/save status caching |
| **ORM (NoSQL)** | Mongoose 8.x | MongoDB object modeling |
| **ORM (SQL)** | TypeORM 0.3.x | PostgreSQL entity management |
| **Validation** | class-validator | DTO validation |
| **Documentation** | Swagger/OpenAPI | API documentation |

---

## 📂 Module Structure

```
apps/chefooz-apis/src/modules/reels/
├── reels.module.ts                 # Module definition
├── reels.controller.ts             # HTTP endpoints (524 lines)
├── reels.service.ts                # Business logic (800 lines)
└── dto/
    ├── get-reel-detail.dto.ts      # Response DTOs (200 lines)
    └── create-menu-reel.dto.ts     # Request DTOs (25 lines)
```

### Dependencies

```typescript
// reels.module.ts
@Module({
  imports: [
    MongooseModule.forFeature([{ name: Reel.name, schema: ReelSchema }]),
    TypeOrmModule.forFeature([User, UserReputationCurrent, Order, ChefMenuItem]),
    CacheModule,
  ],
  controllers: [ReelsController],
  providers: [ReelsService],
  exports: [ReelsService],
})
```

**External Dependencies**:
- `@chefooz-app/domain` → Feature flag utilities, environment config
- `../../database/schemas/reel.schema` → Reel MongoDB schema
- `../../database/entities/user.entity` → User Postgres entity
- `../../database/entities/user-reputation-current.entity` → Reputation entity
- `../../database/entities/order.entity` → Order entity
- `../chef-kitchen/entities/chef-menu-item.entity` → Menu item entity
- `../cache/cache.service` → Valkey/Redis abstraction
- `../../utils/s3-url.util` → S3 URI to HTTPS conversion

---

## 🗄️ Database Schema

### MongoDB - Reel Document

**Collection**: `reels`  
**Indexes**: 9 compound indexes for optimized queries

```typescript
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { Document } from 'mongoose';

@Schema({ timestamps: true, collection: 'reels' })
export class Reel extends Document {
  @Prop({ required: true, index: true })
  userId!: string; // FK to Postgres users.id

  @Prop({ index: true })
  mediaId?: string; // FK to Media document

  @Prop({ type: String, enum: ['REEL', 'POST'], default: 'REEL', index: true })
  contentType!: string;

  @Prop()
  caption?: string;

  @Prop({ type: [String], default: [], index: true })
  hashtags!: string[];

  @Prop({ type: [String], default: [] })
  taggedUserIds!: string[]; // Caption mentions

  @Prop({
    type: [{
      userId: String,
      username: String,
      fullName: String,
      x: Number, // 0-1 normalized
      y: Number  // 0-1 normalized
    }],
    default: []
  })
  positionedTags!: Array<{
    userId: string;
    username: string;
    fullName?: string;
    x: number;
    y: number;
  }>;

  @Prop()
  videoUrl?: string; // HLS adaptive streaming

  @Prop()
  thumbnailUrl?: string;

  @Prop({ required: true })
  durationSec!: number;

  @Prop({ index: true })
  linkedOrderId?: string; // FK to Postgres orders.id

  @Prop()
  creatorOrderValue?: number; // Order total snapshot (paise)

  @Prop({
    type: String,
    enum: ['USER_REVIEW', 'PROMOTIONAL', 'MENU_SHOWCASE'],
    default: 'PROMOTIONAL',
    index: true
  })
  reelPurpose!: string;

  @Prop({ type: Object })
  linkedMenu?: {
    chefId: string;
    menuItemIds: string[];
    estimatedPaise: number;
    previewImage?: string;
  };

  @Prop({ type: ReelStatsSchema, default: () => ({}) })
  stats!: ReelStats;

  @Prop({ default: false, index: true })
  promoted?: boolean;

  @Prop({ type: ReelLocationSchema })
  location?: ReelLocation;

  @Prop()
  coverTimestampSec?: number; // User-selected thumbnail

  @Prop({ type: Array, default: [] })
  textOverlays?: any[]; // FFmpeg burned-in text

  @Prop({ type: Object })
  musicOverlay?: any; // FFmpeg mixed audio

  @Prop({ type: Object })
  filter?: any; // Image filter applied

  @Prop({ index: true })
  deletedAt?: Date; // Soft delete

  @Prop()
  deletedBy?: string; // userId who deleted

  createdAt!: Date; // Auto-generated by timestamps
  updatedAt!: Date; // Auto-generated by timestamps
}

@Schema({ _id: false })
export class ReelStats {
  @Prop({ default: 0 }) views!: number;
  @Prop({ default: 0 }) likes!: number;
  @Prop({ default: 0 }) comments!: number;
  @Prop({ default: 0 }) saves!: number;
  @Prop({ default: 0 }) savedCount!: number;
  @Prop({ default: 0 }) shareCount!: number;
  @Prop({ default: 0 }) copyLinkCount!: number;
  @Prop({ default: 0 }) externalShareCount!: number;
  @Prop({ default: 0 }) directShareCount!: number;
  @Prop({ default: 0 }) storyShareCount!: number;
}

@Schema({ _id: false })
export class ReelLocation {
  @Prop({ required: true }) lat!: number;
  @Prop({ required: true }) lng!: number;
}

export const ReelSchema = SchemaFactory.createForClass(Reel);
```

**Indexes**:
```javascript
reels.createIndex({ createdAt: -1 })
reels.createIndex({ userId: 1, createdAt: -1 })
reels.createIndex({ 'stats.likes': -1, createdAt: -1 })
reels.createIndex({ promoted: 1, createdAt: -1 })
reels.createIndex({ reelPurpose: 1, createdAt: -1 })
reels.createIndex({ 'linkedMenu.chefId': 1, createdAt: -1 })
reels.createIndex({ deletedAt: 1, userId: 1 })
reels.createIndex({ deletedAt: 1, reelPurpose: 1, createdAt: -1 })
reels.createIndex({ location: '2dsphere' })
```

### PostgreSQL - Related Entities

**Users Table** (partial schema):
```sql
CREATE TABLE users (
  id UUID PRIMARY KEY,
  username VARCHAR(30) UNIQUE NOT NULL,
  full_name VARCHAR(100),
  avatar_url TEXT,
  role VARCHAR(20) DEFAULT 'customer',
  created_at TIMESTAMP DEFAULT NOW()
);
```

**User Reputation Current Table**:
```sql
CREATE TABLE user_reputation_current (
  user_id UUID PRIMARY KEY REFERENCES users(id),
  score INTEGER DEFAULT 0,
  updated_at TIMESTAMP DEFAULT NOW()
);
```

**Orders Table** (partial schema):
```sql
CREATE TABLE orders (
  id UUID PRIMARY KEY,
  chef_id UUID REFERENCES users(id),
  customer_id UUID REFERENCES users(id),
  total_paise INTEGER NOT NULL,
  items JSONB NOT NULL, -- Array of order items
  status VARCHAR(50),
  created_at TIMESTAMP DEFAULT NOW()
);
```

**Chef Menu Items Table** (partial schema):
```sql
CREATE TABLE chef_menu_items (
  id UUID PRIMARY KEY,
  chef_id UUID REFERENCES users(id),
  name VARCHAR(200) NOT NULL,
  price DECIMAL(10,2),
  image_url TEXT,
  is_active BOOLEAN DEFAULT true,
  platform_category_id UUID REFERENCES platform_categories(id)
);
```

---

## 🔌 API Endpoints

### 1. GET /api/v1/reels/detail/:mediaId

**Description**: Get complete reel detail with owner profile and engagement stats

**Authentication**: Optional JWT (personalizes isLiked/isSaved)

**Parameters**:
- `mediaId` (path, required): MongoDB ObjectId of the reel

**Response** (200):
```json
{
  "success": true,
  "message": "Reel detail retrieved successfully",
  "data": {
    "id": "507f1f77bcf86cd799439011",
    "mediaId": "507f1f77bcf86cd799439011",
    "userId": "uuid-user-123",
    "author": {
      "id": "uuid-user-123",
      "username": "chef_kumar",
      "fullName": "Kumar Singh",
      "avatarUrl": "https://cdn.chefooz.com/avatars/user123.jpg",
      "reputationLevel": "gold",
      "isVerified": false
    },
    "caption": "Making authentic Dal Makhani! #cooking #indian",
    "hashtags": ["cooking", "indian"],
    "videoUrl": "https://cdn.chefooz.com/reels/video123.m3u8",
    "thumbnailUrl": "https://cdn.chefooz.com/reels/thumb123.jpg",
    "durationSec": 30,
    "stats": {
      "views": 5678,
      "likes": 234,
      "comments": 89,
      "saves": 45,
      "isLiked": false,
      "isSaved": false
    },
    "linkedOrder": {
      "orderId": "uuid-order-456",
      "itemCount": 3,
      "totalPaise": 79900,
      "chefId": "uuid-chef-789",
      "previewImageUrl": "https://cdn.chefooz.com/items/dal-makhani.jpg",
      "itemPreviews": "Dal Makhani (2), Naan, Raita"
    },
    "createdAt": "2024-01-15T10:30:00Z",
    "variants": [
      {
        "url": "https://cdn.chefooz.com/reels/video123.m3u8",
        "quality": "auto"
      }
    ]
  }
}
```

**Response** (404):
```json
{
  "success": false,
  "message": "Reel with ID 507f1f77bcf86cd799439011 not found",
  "errorCode": "REEL_NOT_FOUND"
}
```

**Code Example**:
```typescript
// reels.service.ts
async getReelDetail(mediaId: string, viewerId?: string): Promise<ReelDetailResponseDto> {
  // Try by _id first, fallback to mediaId field
  let reel = await this.reelModel.findById(mediaId).exec();
  if (!reel) {
    reel = await this.reelModel.findOne({ mediaId }).exec();
  }
  
  if (!reel || reel.deletedAt) {
    throw new NotFoundException(`Reel with ID ${mediaId} not found`);
  }

  // Fetch owner profile
  const owner = await this.userRepository.findOne({
    where: { id: reel.userId },
    select: ['id', 'username', 'fullName', 'avatarUrl'],
  });

  // Fetch reputation tier
  const reputation = await this.reputationRepository.findOne({
    where: { userId: reel.userId },
  });
  const reputationTier = reputation ? mapScoreToLevel(reputation.score).level : 'bronze';

  // Check like/save status
  let isLiked = false;
  let isSaved = false;
  if (viewerId) {
    isLiked = await this.cacheService.sismember(`reel:${mediaId}:likes`, viewerId);
    isSaved = await this.cacheService.sismember(`reel:${mediaId}:saves`, viewerId);
  }

  // Build linked order snapshot
  let linkedOrder = null;
  if (reel.linkedOrderId) {
    linkedOrder = await this.buildLinkedOrderSnapshot(reel.linkedOrderId, reel.creatorOrderValue);
  }

  return this.mapToDetailResponse(reel, owner, reputationTier, isLiked, isSaved, linkedOrder);
}
```

---

### 2. DELETE /api/v1/reels/:reelId

**Description**: Soft delete a reel (owner only)

**Authentication**: Required JWT

**Parameters**:
- `reelId` (path, required): MongoDB ObjectId of the reel

**Response** (200):
```json
{
  "success": true,
  "message": "Reel deleted successfully"
}
```

**Response** (403):
```json
{
  "success": false,
  "message": "You can only delete your own reels",
  "errorCode": "FORBIDDEN"
}
```

**Response** (404):
```json
{
  "success": false,
  "message": "Reel not found",
  "errorCode": "REEL_NOT_FOUND"
}
```

**Code Example**:
```typescript
// reels.service.ts
async deleteReel(reelId: string, userId: string): Promise<void> {
  const reel = await this.reelModel.findById(reelId).exec();

  if (!reel) {
    throw new NotFoundException('Reel not found');
  }

  if (reel.deletedAt) {
    throw new BadRequestException('Reel already deleted');
  }

  if (reel.userId !== userId) {
    throw new ForbiddenException('You can only delete your own reels');
  }

  // Soft delete
  reel.deletedAt = new Date();
  reel.deletedBy = userId;
  await reel.save();

  // Clear cache
  await this.cacheService.del(`reel:${reelId}:likes`);
  await this.cacheService.del(`reel:${reelId}:saves`);
}
```

---

### 3. POST /api/v1/reels/menu

**Description**: Create a menu showcase reel (chef-only)

**Authentication**: Required JWT

**Request Body**:
```json
{
  "menuItemIds": ["uuid-item-1", "uuid-item-2"],
  "caption": "Special combo: Dal + Naan + Raita ₹299!"
}
```

**Response** (201):
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

**Response** (403):
```json
{
  "success": false,
  "message": "Only chefs can create menu reels",
  "errorCode": "CHEF_ONLY"
}
```

**Response** (400):
```json
{
  "success": false,
  "message": "Some menu items not found",
  "errorCode": "INVALID_MENU_ITEMS"
}
```

**Code Example**:
```typescript
// reels.service.ts
async createMenuReel(chefId: string, mediaId: string, dto: CreateMenuReelDto): Promise<Reel> {
  // Feature flag check
  const envConfig = getEnvConfig();
  if (!canUploadMenuReel('MENU_SHOWCASE', 'chef', envConfig)) {
    throw new HttpException({
      success: false,
      message: 'Menu reels feature is currently disabled',
      errorCode: 'FEATURE_DISABLED_MENU_REELS',
    }, HttpStatus.FORBIDDEN);
  }

  // Validate chef role
  const chef = await this.userRepository.findOne({ where: { id: chefId } });
  if (!chef || chef.role !== 'chef') {
    throw new ForbiddenException('Only chefs can create menu reels');
  }

  // Validate menu items
  const menuItems = await this.menuItemRepository.find({
    where: { id: In(dto.menuItemIds) },
  });

  if (menuItems.length !== dto.menuItemIds.length) {
    throw new BadRequestException('Some menu items not found');
  }

  const invalidItems = menuItems.filter(m => m.chefId !== chefId);
  if (invalidItems.length > 0) {
    throw new ForbiddenException('You can only create menu reels for your own menu items');
  }

  // Calculate total
  const totalPaise = menuItems.reduce((sum, m) => sum + Math.round((m.price || 0) * 100), 0);
  const previewImage = menuItems[0]?.imageUrl || undefined;

  // Create reel
  const menuReel = new this.reelModel({
    userId: chefId,
    mediaId,
    caption: dto.caption || undefined,
    hashtags: [],
    durationSec: 0, // Updated by media processing
    reelPurpose: 'MENU_SHOWCASE',
    linkedMenu: {
      chefId,
      menuItemIds: dto.menuItemIds,
      estimatedPaise: totalPaise,
      previewImage,
    },
    stats: {
      views: 0,
      likes: 0,
      comments: 0,
      saves: 0,
      savedCount: 0,
      shareCount: 0,
      copyLinkCount: 0,
      externalShareCount: 0,
      directShareCount: 0,
      storyShareCount: 0,
    },
  });

  return await menuReel.save();
}
```

---

### 4. GET /api/v1/reels/user/:userId

**Description**: Get all reels for a specific user (profile reel viewer)

**Authentication**: Optional JWT

**Parameters**:
- `userId` (path, required): User UUID
- `limit` (query, optional): Max reels to return (default 50)
- `skip` (query, optional): Number to skip (default 0)
- `includePromotional` (query, optional): Include promotional/review reels (default true)
- `includeMenu` (query, optional): Include menu showcase reels (default false)

**Response** (200):
```json
{
  "success": true,
  "message": "User reels retrieved successfully",
  "data": [
    {
      "id": "507f1f77bcf86cd799439011",
      "reelId": "507f1f77bcf86cd799439011",
      "mediaId": "507f1f77bcf86cd799439011",
      "userId": "uuid-user-123",
      "author": {
        "userId": "uuid-user-123",
        "username": "chef_kumar",
        "fullName": "Kumar Singh",
        "avatarUrl": "https://cdn.chefooz.com/avatars/user123.jpg"
      },
      "videoUrl": "https://cdn.chefooz.com/reels/video123.m3u8",
      "thumbnailUrl": "https://cdn.chefooz.com/reels/thumb123.jpg",
      "caption": "Making Dal Makhani!",
      "hashtags": ["cooking"],
      "taggedUsers": [
        {
          "userId": "uuid-user-456",
          "username": "john_doe",
          "fullName": "John Doe",
          "avatarUrl": "https://cdn.chefooz.com/avatars/user456.jpg"
        }
      ],
      "durationSec": 30,
      "stats": {
        "views": 5678,
        "likes": 234,
        "comments": 89,
        "saves": 45,
        "savedCount": 45,
        "shareCount": 12
      },
      "createdAt": "2024-01-15T10:30:00Z",
      "isProcessing": false,
      "contentType": "REEL",
      "reelPurpose": "USER_REVIEW"
    }
  ]
}
```

**Code Example**:
```typescript
// reels.service.ts
async getUserReels(
  userId: string,
  limit: number = 50,
  skip: number = 0,
  includePromotional: boolean = true,
  includeMenu: boolean = false
): Promise<Reel[]> {
  const purposes: string[] = [];
  if (includePromotional) {
    purposes.push('PROMOTIONAL', 'USER_REVIEW');
  }
  if (includeMenu) {
    purposes.push('MENU_SHOWCASE');
  }

  if (purposes.length === 0) {
    return [];
  }

  return await this.reelModel
    .find({
      userId,
      reelPurpose: { $in: purposes },
      deletedAt: null,
    })
    .sort({ createdAt: -1 })
    .skip(skip)
    .limit(limit)
    .exec();
}
```

---

### 5. GET /api/v1/reels/chef/:chefId/menu

**Description**: Get all menu reels for a specific chef

**Authentication**: None required (public)

**Parameters**:
- `chefId` (path, required): Chef user UUID

**Response** (200):
```json
{
  "success": true,
  "message": "Chef menu reels retrieved successfully",
  "data": [
    {
      "reelId": "507f1f77bcf86cd799439011",
      "videoUrl": "https://cdn.chefooz.com/reels/video123.m3u8",
      "thumbnailUrl": "https://cdn.chefooz.com/reels/thumb123.jpg",
      "caption": "Special combo deal!",
      "estimatedPaise": 29900,
      "itemCount": 3,
      "viewCount": 1234,
      "likeCount": 56,
      "createdAt": "2024-01-15T10:30:00Z"
    }
  ]
}
```

**Code Example**:
```typescript
// reels.service.ts
async getChefMenuReels(chefId: string, limit: number = 20): Promise<Reel[]> {
  return await this.reelModel
    .find({
      'linkedMenu.chefId': chefId,
      reelPurpose: 'MENU_SHOWCASE',
      deletedAt: null,
    })
    .sort({ createdAt: -1 })
    .limit(limit)
    .exec();
}
```

---

## 🧩 Service Layer

### ReelsService Core Methods

**Method**: `getReelDetail(mediaId, viewerId?)`
```typescript
/**
 * Get detailed reel information
 * @param mediaId - MongoDB ObjectId of the reel
 * @param viewerId - Optional viewer user ID for personalized data
 * @returns Complete reel detail
 */
async getReelDetail(mediaId: string, viewerId?: string): Promise<ReelDetailResponseDto>
```

**Logic Flow**:
1. Query MongoDB for reel by `_id` or `mediaId` field
2. Check if reel exists and not soft-deleted
3. Fetch owner profile from Postgres (username, avatar, fullName)
4. Fetch owner reputation tier from `user_reputation_current` table
5. Check like/save status from Redis cache (if viewerId provided)
6. Build linked order snapshot (if `linkedOrderId` exists)
7. Map to response DTO with S3 URI → HTTPS URL conversion
8. Return complete reel detail

---

**Method**: `buildLinkedOrderSnapshot(orderId, creatorOrderValue?)`
```typescript
/**
 * Build linked order snapshot with item previews
 * @param orderId - Order ID from reel
 * @param creatorOrderValue - Total order value in paise
 * @returns Linked order snapshot or null
 */
private async buildLinkedOrderSnapshot(
  orderId: string,
  creatorOrderValue?: number,
): Promise<LinkedOrderSnapshotDto | null>
```

**Logic Flow**:
1. Query Postgres orders table by orderId
2. If order not found, log warning and return null
3. Build `itemPreviews` string: "Item1 (qty), Item2, Item3"
4. Extract first item's image as preview
5. Return snapshot object with order details

**Example Output**:
```typescript
{
  orderId: "uuid-order-123",
  itemCount: 3,
  totalPaise: 79900,
  chefId: "uuid-chef-456",
  previewImageUrl: "https://cdn.chefooz.com/items/dal-makhani.jpg",
  itemPreviews: "Dal Makhani (2), Naan, Raita"
}
```

---

**Method**: `deleteReel(reelId, userId)`
```typescript
/**
 * Soft delete a reel (owner only)
 * @param reelId - MongoDB ObjectId of the reel
 * @param userId - User ID requesting deletion
 * @throws NotFoundException if reel not found
 * @throws ForbiddenException if user is not the owner
 */
async deleteReel(reelId: string, userId: string): Promise<void>
```

**Logic Flow**:
1. Find reel by `_id` in MongoDB
2. Validate reel exists
3. Check if already deleted (`deletedAt` not null)
4. Verify ownership (`reel.userId === userId`)
5. Set `deletedAt = new Date()` and `deletedBy = userId`
6. Save document
7. Clear Redis cache keys (`reel:{reelId}:likes`, `reel:{reelId}:saves`)

---

**Method**: `createMenuReel(chefId, mediaId, dto)`
```typescript
/**
 * Create a menu showcase reel (chef-only)
 * @param chefId - Chef creating the menu reel
 * @param mediaId - Uploaded video media ID
 * @param dto - Menu reel creation data
 * @returns Created reel document
 */
async createMenuReel(
  chefId: string,
  mediaId: string,
  dto: CreateMenuReelDto,
): Promise<Reel>
```

**Logic Flow**:
1. Check feature flag `MENU_REELS` enabled
2. Validate chef exists and has `role = 'chef'`
3. Fetch all menu items by `dto.menuItemIds`
4. Validate all menu items exist
5. Validate chef owns all menu items (`menuItem.chefId === chefId`)
6. Calculate `estimatedPaise` from menu item prices
7. Extract first menu item's image as preview
8. Create new reel document with `reelPurpose = 'MENU_SHOWCASE'`
9. Save to MongoDB
10. Return saved reel

---

**Method**: `getUserReels(userId, limit, skip, includePromotional, includeMenu)`
```typescript
/**
 * Get all reels for a specific user
 * @param userId - User ID to fetch reels for
 * @param limit - Max reels to return
 * @param skip - Number of reels to skip
 * @param includePromotional - Include promotional/review reels
 * @param includeMenu - Include menu showcase reels
 * @returns Array of user reels
 */
async getUserReels(
  userId: string,
  limit: number = 50,
  skip: number = 0,
  includePromotional: boolean = true,
  includeMenu: boolean = false
): Promise<Reel[]>
```

**Query**:
```javascript
db.reels.find({
  userId: "uuid",
  reelPurpose: { $in: ['PROMOTIONAL', 'USER_REVIEW'] }, // or ['MENU_SHOWCASE']
  deletedAt: null
}).sort({ createdAt: -1 }).skip(0).limit(50)
```

---

**Method**: `getChefMenuReels(chefId, limit)`
```typescript
/**
 * Get all menu reels for a specific chef
 * @param chefId - Chef ID to fetch menu reels for
 * @param limit - Max reels to return
 * @returns Array of menu reels
 */
async getChefMenuReels(chefId: string, limit: number = 20): Promise<Reel[]>
```

**Query**:
```javascript
db.reels.find({
  'linkedMenu.chefId': "uuid-chef",
  reelPurpose: 'MENU_SHOWCASE',
  deletedAt: null
}).sort({ createdAt: -1 }).limit(20)
```

---

## 📦 Data Transfer Objects

### CreateMenuReelDto (Request)

```typescript
import { IsArray, IsString, IsOptional, ArrayMinSize } from 'class-validator';
import { ApiProperty, ApiPropertyOptional } from '@nestjs/swagger';

export class CreateMenuReelDto {
  @ApiProperty({
    description: 'Array of menu item/product IDs to showcase',
    example: ['uuid-item-1', 'uuid-item-2'],
    type: [String],
    minItems: 1,
  })
  @IsArray()
  @ArrayMinSize(1)
  @IsString({ each: true })
  menuItemIds!: string[];

  @ApiPropertyOptional({
    description: 'Optional caption for the menu reel',
    example: 'Special combo deal: Dal + Naan + Raita ₹299!',
    maxLength: 500,
  })
  @IsOptional()
  @IsString()
  caption?: string;
}
```

---

### ReelDetailResponseDto (Response)

```typescript
export class ReelDetailResponseDto {
  @ApiProperty({ description: 'Reel MongoDB ID' })
  mediaId!: string;

  @ApiProperty({ description: 'User ID of reel owner' })
  userId!: string;

  @ApiProperty({ description: 'Reel owner information', type: ReelOwnerDto })
  author!: {
    id: string;
    username: string;
    fullName: string;
    avatarUrl: string | null;
    reputationLevel: string;
    isVerified: boolean;
  };

  @ApiProperty({ description: 'Caption', nullable: true })
  caption?: string | null;

  @ApiProperty({ description: 'Hashtags', type: [String] })
  hashtags!: string[];

  @ApiProperty({ description: 'Video URL (HLS)' })
  videoUrl!: string;

  @ApiProperty({ description: 'Thumbnail URL', nullable: true })
  thumbnailUrl?: string | null;

  @ApiProperty({ description: 'Duration in seconds' })
  durationSec!: number;

  @ApiProperty({ description: 'Engagement stats' })
  stats!: {
    views: number;
    likes: number;
    comments: number;
    saves: number;
    isLiked: boolean;
    isSaved: boolean;
  };

  @ApiProperty({ description: 'Linked order snapshot', nullable: true })
  linkedOrder?: LinkedOrderSnapshotDto | null;

  @ApiProperty({ description: 'Creation timestamp' })
  createdAt!: string;

  @ApiProperty({ description: 'Video variants', type: [ReelVariantDto] })
  variants!: ReelVariantDto[];
}
```

---

### LinkedOrderSnapshotDto (Response)

```typescript
export class LinkedOrderSnapshotDto {
  @ApiProperty({ description: 'Order ID' })
  orderId!: string;

  @ApiProperty({ description: 'Number of items in order' })
  itemCount!: number;

  @ApiProperty({ description: 'Total order value in paise' })
  totalPaise!: number;

  @ApiProperty({ description: 'Chef ID who fulfilled the order' })
  chefId!: string;

  @ApiProperty({ description: 'Preview image URL', nullable: true })
  previewImageUrl?: string | null;

  @ApiProperty({ description: 'Item names preview', nullable: true })
  itemPreviews?: string; // "Dal Makhani (2), Naan, Raita"
}
```

---

## 🔗 Integration Patterns

### S3 URL Conversion

**Utility**: `s3UriToHttps(s3Uri, cdnUrl)`

```typescript
// Input: s3://chefooz-output/reels/video123.m3u8
// Output: https://cdn.chefooz.com/reels/video123.m3u8

import { s3UriToHttps } from '../../utils/s3-url.util';

const videoUrl = s3UriToHttps(reel.videoUrl, process.env.CDN_URL);
```

---

### Cache Integration (Valkey/Redis)

**Like Status Check**:
```typescript
const likeKey = `reel:${mediaId}:likes`;
const isLiked = await this.cacheService.sismember(likeKey, viewerId);
```

**Save Status Check**:
```typescript
const saveKey = `reel:${mediaId}:saves`;
const isSaved = await this.cacheService.sismember(saveKey, viewerId);
```

**Cache Cleanup on Deletion**:
```typescript
await this.cacheService.del(`reel:${reelId}:likes`);
await this.cacheService.del(`reel:${reelId}:saves`);
```

---

### Reputation Tier Mapping

```typescript
import { mapScoreToLevel } from '../reputation/utils/leveling';

const reputation = await this.reputationRepository.findOne({
  where: { userId: reel.userId },
});

const tier = reputation ? mapScoreToLevel(reputation.score).level : 'bronze';
// tier = 'bronze' | 'silver' | 'gold' | 'platinum'
```

---

### Feature Flag Enforcement

```typescript
import { getEnvConfig, canUploadMenuReel } from '@chefooz-app/domain';

const envConfig = getEnvConfig();
if (!canUploadMenuReel('MENU_SHOWCASE', 'chef', envConfig)) {
  throw new HttpException({
    success: false,
    message: 'Menu reels feature is currently disabled',
    errorCode: 'FEATURE_DISABLED_MENU_REELS',
  }, HttpStatus.FORBIDDEN);
}
```

---

## 🚨 Error Handling

### Error Response Envelope

```typescript
interface ErrorResponse {
  success: false;
  message: string;
  errorCode?: string;
}
```

### Common Error Codes

| Error Code | HTTP Status | Trigger | Recovery |
|------------|-------------|---------|----------|
| `REEL_NOT_FOUND` | 404 | Reel doesn't exist or soft-deleted | Check mediaId, ensure reel not deleted |
| `UNAUTHORIZED` | 401 | Missing JWT token | Redirect to login |
| `FORBIDDEN` | 403 | User tries to delete others' reel | Verify ownership |
| `CHEF_ONLY` | 403 | Non-chef tries to create menu reel | Upgrade to chef account |
| `INVALID_MENU_ITEMS` | 400 | Menu items not found | Verify menuItemIds exist |
| `FEATURE_DISABLED_MENU_REELS` | 403 | Feature flag disabled | Wait for feature rollout |
| `REEL_DELETE_ERROR` | 500 | Database/cache error | Retry deletion |
| `USER_REELS_FETCH_ERROR` | 500 | Database error | Retry fetch |
| `MENU_REELS_FETCH_ERROR` | 500 | Database error | Retry fetch |

### Error Handling Pattern

```typescript
try {
  // Business logic
} catch (error) {
  this.logger.error(`Operation failed: ${error.message}`, error.stack);

  if (error instanceof HttpException) {
    throw error; // Re-throw NestJS exceptions
  }

  throw new HttpException(
    {
      success: false,
      message: 'Generic error message',
      errorCode: 'ERROR_CODE',
    },
    HttpStatus.INTERNAL_SERVER_ERROR,
  );
}
```

---

## 🧪 Testing Strategy

### Unit Tests (Jest)

**Test File**: `reels.service.spec.ts`

```typescript
describe('ReelsService', () => {
  let service: ReelsService;
  let reelModel: Model<Reel>;
  let userRepository: Repository<User>;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        ReelsService,
        { provide: getModelToken(Reel.name), useValue: mockReelModel },
        { provide: getRepositoryToken(User), useValue: mockUserRepository },
        // ... other mocks
      ],
    }).compile();

    service = module.get<ReelsService>(ReelsService);
  });

  describe('getReelDetail', () => {
    it('should return reel detail with owner profile', async () => {
      const mockReel = { _id: 'abc123', userId: 'uuid-user', caption: 'Test' };
      const mockUser = { id: 'uuid-user', username: 'john_doe' };

      jest.spyOn(reelModel, 'findById').mockReturnValue({
        exec: jest.fn().mockResolvedValue(mockReel),
      });

      jest.spyOn(userRepository, 'findOne').mockResolvedValue(mockUser);

      const result = await service.getReelDetail('abc123');

      expect(result.author.username).toBe('john_doe');
    });

    it('should throw NotFoundException if reel not found', async () => {
      jest.spyOn(reelModel, 'findById').mockReturnValue({
        exec: jest.fn().mockResolvedValue(null),
      });

      await expect(service.getReelDetail('invalid')).rejects.toThrow(NotFoundException);
    });
  });

  describe('deleteReel', () => {
    it('should soft delete reel and clear cache', async () => {
      const mockReel = { _id: 'abc123', userId: 'uuid-user', save: jest.fn() };
      
      jest.spyOn(reelModel, 'findById').mockReturnValue({
        exec: jest.fn().mockResolvedValue(mockReel),
      });

      await service.deleteReel('abc123', 'uuid-user');

      expect(mockReel.deletedAt).toBeDefined();
      expect(mockReel.save).toHaveBeenCalled();
    });

    it('should throw ForbiddenException if not owner', async () => {
      const mockReel = { _id: 'abc123', userId: 'uuid-other' };
      
      jest.spyOn(reelModel, 'findById').mockReturnValue({
        exec: jest.fn().mockResolvedValue(mockReel),
      });

      await expect(service.deleteReel('abc123', 'uuid-user')).rejects.toThrow(ForbiddenException);
    });
  });

  describe('createMenuReel', () => {
    it('should create menu reel with valid menu items', async () => {
      const dto = { menuItemIds: ['uuid-1', 'uuid-2'], caption: 'Test' };
      const mockChef = { id: 'uuid-chef', role: 'chef' };
      const mockMenuItems = [
        { id: 'uuid-1', chefId: 'uuid-chef', price: 299 },
        { id: 'uuid-2', chefId: 'uuid-chef', price: 99 },
      ];

      jest.spyOn(userRepository, 'findOne').mockResolvedValue(mockChef);
      jest.spyOn(menuItemRepository, 'find').mockResolvedValue(mockMenuItems);

      const result = await service.createMenuReel('uuid-chef', 'media-123', dto);

      expect(result.reelPurpose).toBe('MENU_SHOWCASE');
      expect(result.linkedMenu.estimatedPaise).toBe(39800); // 298*100 + 99*100
    });

    it('should throw ForbiddenException if not chef', async () => {
      const dto = { menuItemIds: ['uuid-1'], caption: 'Test' };
      const mockUser = { id: 'uuid-user', role: 'customer' };

      jest.spyOn(userRepository, 'findOne').mockResolvedValue(mockUser);

      await expect(service.createMenuReel('uuid-user', 'media-123', dto)).rejects.toThrow(ForbiddenException);
    });
  });
});
```

---

### Integration Tests (E2E)

**Test File**: `reels.e2e-spec.ts`

```typescript
describe('ReelsController (E2E)', () => {
  let app: INestApplication;
  let authToken: string;

  beforeAll(async () => {
    const moduleFixture = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = moduleFixture.createNestApplication();
    await app.init();

    // Login to get JWT token
    const loginResponse = await request(app.getHttpServer())
      .post('/api/v1/auth/login')
      .send({ phone: '+919876543210', otp: '123456' });

    authToken = loginResponse.body.data.accessToken;
  });

  it('GET /api/v1/reels/detail/:mediaId should return reel detail', async () => {
    const response = await request(app.getHttpServer())
      .get('/api/v1/reels/detail/507f1f77bcf86cd799439011')
      .set('Authorization', `Bearer ${authToken}`)
      .expect(200);

    expect(response.body.success).toBe(true);
    expect(response.body.data).toHaveProperty('author');
    expect(response.body.data).toHaveProperty('stats');
  });

  it('DELETE /api/v1/reels/:reelId should soft delete own reel', async () => {
    // Create test reel first
    const createResponse = await request(app.getHttpServer())
      .post('/api/v1/reels/menu')
      .set('Authorization', `Bearer ${authToken}`)
      .send({ menuItemIds: ['uuid-1'], caption: 'Test' })
      .expect(201);

    const reelId = createResponse.body.data.reelId;

    // Delete reel
    const deleteResponse = await request(app.getHttpServer())
      .delete(`/api/v1/reels/${reelId}`)
      .set('Authorization', `Bearer ${authToken}`)
      .expect(200);

    expect(deleteResponse.body.success).toBe(true);

    // Verify reel is hidden
    await request(app.getHttpServer())
      .get(`/api/v1/reels/detail/${reelId}`)
      .expect(404);
  });
});
```

---

## 🚀 Deployment

### Environment Variables

```bash
# MongoDB
MONGO_URI=mongodb://localhost:27017/chefooz

# PostgreSQL
DB_HOST=localhost
DB_PORT=5432
DB_USERNAME=chefooz
DB_PASSWORD=secure_password
DB_DATABASE=chefooz_prod

# Redis/Valkey
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=redis_password

# CDN
CDN_URL=https://cdn.chefooz.com

# Feature Flags
FEATURE_FLAG_MENU_REELS=true
```

---

### Docker Deployment

**Dockerfile**:
```dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --production

COPY dist/ ./dist/

EXPOSE 3000

CMD ["node", "dist/apps/chefooz-apis/main.js"]
```

**docker-compose.yml**:
```yaml
version: '3.8'

services:
  api:
    build: .
    ports:
      - "3000:3000"
    environment:
      - MONGO_URI=mongodb://mongo:27017/chefooz
      - DB_HOST=postgres
      - REDIS_HOST=redis
    depends_on:
      - mongo
      - postgres
      - redis

  mongo:
    image: mongo:6
    volumes:
      - mongo_data:/data/db

  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: chefooz_prod
      POSTGRES_USER: chefooz
      POSTGRES_PASSWORD: secure_password
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: valkey/valkey:7
    volumes:
      - redis_data:/data

volumes:
  mongo_data:
  postgres_data:
  redis_data:
```

---

### Kubernetes Deployment

**deployment.yaml**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: reels-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: reels-api
  template:
    metadata:
      labels:
        app: reels-api
    spec:
      containers:
      - name: api
        image: chefooz/api:latest
        ports:
        - containerPort: 3000
        env:
        - name: MONGO_URI
          valueFrom:
            secretKeyRef:
              name: mongo-secret
              key: uri
        - name: DB_HOST
          value: "postgres-service"
        - name: REDIS_HOST
          value: "redis-service"
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
```

---

## 📊 Monitoring & Observability

### Logging

```typescript
// reels.service.ts
this.logger.log(`Reel ${reelId} soft deleted by user ${userId}`);
this.logger.debug(`Fetching reel detail for mediaId: ${mediaId}`);
this.logger.warn(`Linked order ${orderId} not found`);
this.logger.error(`Failed to delete reel: ${error.message}`, error.stack);
```

**Log Format**:
```json
{
  "timestamp": "2024-01-15T10:30:00Z",
  "level": "info",
  "context": "ReelsService",
  "message": "Reel 507f1f77bcf86cd799439011 soft deleted by user uuid-123",
  "userId": "uuid-123",
  "reelId": "507f1f77bcf86cd799439011"
}
```

---

### Metrics (Prometheus)

```typescript
// Track reel detail API latency
const timer = histogram('reel_detail_duration_seconds', 'Reel detail API latency').startTimer();
const result = await this.getReelDetail(mediaId, viewerId);
timer();

// Track deletion events
counter('reel_deletions_total', 'Total reel deletions').inc();

// Track menu reel creations
counter('menu_reel_creations_total', 'Total menu reel creations').inc();
```

**Grafana Dashboard**:
```
- Panel: Reel Detail API Latency (P50, P95, P99)
- Panel: Reel Deletions per Hour
- Panel: Menu Reel Creation Rate
- Panel: MongoDB Query Duration
- Panel: Redis Cache Hit Rate
```

---

### Health Checks

```typescript
@Get('health')
async health() {
  const mongoStatus = await this.reelModel.db.db.admin().ping();
  const redisStatus = await this.cacheService.ping();

  return {
    status: 'ok',
    mongo: mongoStatus ? 'connected' : 'disconnected',
    redis: redisStatus ? 'connected' : 'disconnected',
  };
}
```

---

## ⚡ Performance Optimization

### Database Query Optimization

**Problem**: Fetching user reels with owner profile (N+1 query)

**Solution**: Batch fetch author data after collecting all userId values

```typescript
// ❌ Bad: N+1 queries
for (const reel of reels) {
  const author = await this.userRepository.findOne({ where: { id: reel.userId } });
  // ...
}

// ✅ Good: Single batch query
const userIds = [...new Set(reels.map(r => r.userId))];
const users = await this.userRepository.find({ where: { id: In(userIds) } });
const userMap = new Map(users.map(u => [u.id, u]));

reels.map(reel => {
  const author = userMap.get(reel.userId);
  // ...
});
```

---

### Cache Warming Strategy

**Problem**: Cold cache on deployment causes slow first requests

**Solution**: Warm popular reels cache on startup

```typescript
async onModuleInit() {
  // Fetch top 100 trending reels
  const trendingReels = await this.reelModel
    .find({ deletedAt: null })
    .sort({ 'stats.views': -1 })
    .limit(100)
    .exec();

  // Pre-populate cache
  for (const reel of trendingReels) {
    await this.cacheService.set(
      `reel:${reel._id}:detail`,
      JSON.stringify(reel),
      3600 // 1 hour TTL
    );
  }

  this.logger.log('Cache warmed with 100 trending reels');
}
```

---

### MongoDB Index Optimization

**Query**: Get user reels sorted by createdAt
```javascript
db.reels.find({ userId: "uuid", deletedAt: null }).sort({ createdAt: -1 })
```

**Index**: Compound index on `{ userId: 1, deletedAt: 1, createdAt: -1 }`
```javascript
db.reels.createIndex({ userId: 1, deletedAt: 1, createdAt: -1 })
```

**Performance**:
- Without index: ~500ms (full collection scan)
- With index: ~5ms (index scan)

---

## 🔒 Security Considerations

### Input Validation

```typescript
// DTO validation with class-validator
export class CreateMenuReelDto {
  @IsArray()
  @ArrayMinSize(1)
  @IsString({ each: true })
  menuItemIds!: string[];

  @IsOptional()
  @IsString()
  @MaxLength(500)
  caption?: string;
}
```

---

### Authorization Checks

```typescript
// Ownership validation for deletion
if (reel.userId !== userId) {
  throw new ForbiddenException('You can only delete your own reels');
}

// Role validation for menu reel creation
if (chef.role !== 'chef') {
  throw new ForbiddenException('Only chefs can create menu reels');
}
```

---

### SQL Injection Prevention

**TypeORM** automatically escapes parameters:
```typescript
// ✅ Safe: TypeORM parameterized query
await this.userRepository.findOne({ where: { id: userId } });

// ❌ Unsafe: Raw query without escaping
await this.userRepository.query(`SELECT * FROM users WHERE id = '${userId}'`);
```

---

### NoSQL Injection Prevention

**Mongoose** automatically sanitizes:
```typescript
// ✅ Safe: Mongoose query
await this.reelModel.findById(reelId).exec();

// ❌ Unsafe: Raw MongoDB query
await this.reelModel.collection.findOne({ _id: reelId });
```

---

## 🛠️ Troubleshooting

### Issue 1: Reel detail returns 404 but reel exists

**Symptom**: GET /detail/:mediaId returns 404 even though reel exists in database

**Diagnosis**:
```javascript
db.reels.findOne({ _id: ObjectId("507f1f77bcf86cd799439011") })
```

**Possible Causes**:
1. Reel is soft-deleted (`deletedAt` not null)
2. Using `mediaId` field instead of `_id`
3. Invalid ObjectId format

**Solution**:
```typescript
// Check deletedAt field
db.reels.findOne({ _id: ObjectId("507f1f77bcf86cd799439011"), deletedAt: null })

// Try mediaId field fallback
let reel = await this.reelModel.findById(mediaId).exec();
if (!reel) {
  reel = await this.reelModel.findOne({ mediaId }).exec();
}
```

---

### Issue 2: Menu reel creation fails with "Some menu items not found"

**Symptom**: POST /menu returns 400 error even with valid menu item IDs

**Diagnosis**:
```sql
SELECT id, chef_id, is_active FROM chef_menu_items WHERE id IN ('uuid-1', 'uuid-2');
```

**Possible Causes**:
1. Menu items don't exist (typo in IDs)
2. Menu items marked as `is_active = false`
3. Menu items belong to different chef

**Solution**:
```typescript
// Verify menu items exist and are active
const menuItems = await this.menuItemRepository.find({
  where: { id: In(dto.menuItemIds), isActive: true },
});

// Check ownership
const invalidItems = menuItems.filter(m => m.chefId !== chefId);
if (invalidItems.length > 0) {
  throw new ForbiddenException('You can only use your own menu items');
}
```

---

### Issue 3: Deleted reel still appears in user profile

**Symptom**: Deleted reel visible in GET /user/:userId response

**Diagnosis**:
```javascript
db.reels.findOne({ _id: ObjectId("reelId") })
// Check deletedAt field value
```

**Possible Causes**:
1. Query missing `deletedAt: null` filter
2. Cache not cleared after deletion
3. Frontend caching stale data

**Solution**:
```typescript
// Add deletedAt filter to all queries
const reels = await this.reelModel
  .find({
    userId,
    reelPurpose: { $in: purposes },
    deletedAt: null, // ← Critical filter
  })
  .sort({ createdAt: -1 })
  .exec();

// Clear cache on deletion
await this.cacheService.del(`reel:${reelId}:likes`);
await this.cacheService.del(`reel:${reelId}:saves`);
```

---

### Issue 4: S3 URLs return 403 Forbidden

**Symptom**: Video URLs in response return 403 error when accessed

**Diagnosis**:
```bash
curl -I "https://cdn.chefooz.com/reels/video123.m3u8"
# Returns: 403 Forbidden
```

**Possible Causes**:
1. S3 URI not converted to HTTPS URL
2. CloudFront distribution not configured
3. S3 bucket policy restricts public access

**Solution**:
```typescript
// Ensure S3 URI → HTTPS conversion
import { s3UriToHttps } from '../../utils/s3-url.util';

const videoUrl = s3UriToHttps(reel.videoUrl, process.env.CDN_URL);
// Input: s3://chefooz-output/reels/video123.m3u8
// Output: https://cdn.chefooz.com/reels/video123.m3u8
```

---

### Issue 5: isLiked/isSaved always returns false

**Symptom**: Reel detail always shows `isLiked: false` even after liking

**Diagnosis**:
```bash
# Check Redis set membership
redis-cli SISMEMBER reel:507f1f77bcf86cd799439011:likes uuid-user-123
# Returns: 0 (not a member)
```

**Possible Causes**:
1. Like action not updating Redis set
2. ViewerId not passed to `getReelDetail()`
3. Redis connection failure

**Solution**:
```typescript
// Ensure viewerId is extracted from JWT
const viewerId = (req as any).user?.id;

// Check Redis connection
const isLiked = await this.cacheService.sismember(`reel:${mediaId}:likes`, viewerId);

// Verify Redis set exists
await this.cacheService.sadd(`reel:${mediaId}:likes`, userId); // On like action
await this.cacheService.srem(`reel:${mediaId}:likes`, userId); // On unlike action
```

---

## 📚 Related Documentation

- **Reel Schema**: `apps/chefooz-apis/src/database/schemas/reel.schema.ts`
- **User Entity**: `apps/chefooz-apis/src/database/entities/user.entity.ts`
- **Cache Module**: `apps/chefooz-apis/src/modules/cache/cache.service.ts`
- **S3 URL Util**: `apps/chefooz-apis/src/utils/s3-url.util.ts`
- **Feature Flags**: `libs/domain/src/lib/feature-flags.ts`
- **Media Module**: `docs/modules/media/TECHNICAL_GUIDE.md`
- **Explore Module**: `docs/modules/explore/TECHNICAL_GUIDE.md`

---

## 📞 Support

- **Backend Team**: backend@chefooz.com
- **DevOps**: devops@chefooz.com
- **Slack**: #backend-support

---

**Document Version**: 1.0  
**Last Updated**: 2026-02-14  
**Next Review**: 2026-03-14
