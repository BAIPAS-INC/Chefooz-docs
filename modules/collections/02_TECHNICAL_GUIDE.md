# Collections Module - Technical Guide

**Version:** 1.0  
**Last Updated:** February 14, 2026  
**Module:** `apps/chefooz-apis/src/modules/collections/`  
**Tech Stack:** NestJS, PostgreSQL (TypeORM), MongoDB (Mongoose)

---

## 📋 Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Database Schema](#database-schema)
3. [API Endpoints](#api-endpoints)
4. [Service Methods](#service-methods)
5. [Business Logic](#business-logic)
6. [Error Handling](#error-handling)
7. [Performance Optimization](#performance-optimization)
8. [Testing](#testing)

---

## Architecture Overview

### Module Structure

```
apps/chefooz-apis/src/modules/collections/
├── dto/
│   └── collections.dto.ts          # All DTOs (create, rename, add, toggle)
├── entities/
│   ├── collection.entity.ts        # Collection table
│   ├── collection-item.entity.ts   # Junction table (collection-reel)
│   └── saved-reel.entity.ts        # Saved reels table
├── collections.controller.ts       # HTTP endpoints
├── collections.service.ts          # Business logic
├── collections.module.ts           # Module configuration
└── collections.service.spec.ts     # Unit tests
```

### Dependencies

```typescript
@Module({
  imports: [
    TypeOrmModule.forFeature([
      Collection,
      CollectionItem,
      SavedReel,
    ]),
    MongooseModule.forFeature([
      { name: Reel.name, schema: ReelSchema },
    ]),
  ],
  controllers: [CollectionsController],
  providers: [CollectionsService],
  exports: [CollectionsService],
})
```

**Key Dependencies**:
- **TypeORM**: PostgreSQL for collections/saved reels
- **Mongoose**: MongoDB for reel stats sync
- **JWT**: Authentication via JwtAuthGuard

---

## Database Schema

### PostgreSQL Tables

#### 1. collections

```sql
CREATE TABLE collections (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL,
  name VARCHAR(60) NOT NULL,
  emoji VARCHAR(10),
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
  
  -- Index for user's collections lookup
  INDEX idx_collections_user_id (user_id)
);
```

**TypeORM Entity**:
```typescript
@Entity('collections')
@Index(['userId'])
export class Collection {
  @PrimaryGeneratedColumn('uuid')
  id!: string;

  @Column({ type: 'uuid' })
  userId!: string;

  @Column({ type: 'varchar', length: 60 })
  name!: string;

  @Column({ type: 'varchar', length: 10, nullable: true })
  emoji!: string | null;

  @OneToMany(() => CollectionItem, (item) => item.collection, { cascade: true })
  items!: CollectionItem[];

  @CreateDateColumn()
  createdAt!: Date;

  @UpdateDateColumn()
  updatedAt!: Date;
}
```

---

#### 2. collection_items

```sql
CREATE TABLE collection_items (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  collection_id UUID NOT NULL REFERENCES collections(id) ON DELETE CASCADE,
  media_id VARCHAR(24) NOT NULL,  -- MongoDB ObjectId string
  added_at TIMESTAMP NOT NULL DEFAULT NOW(),
  
  -- Prevent duplicate reels in same collection
  UNIQUE (collection_id, media_id),
  
  -- Fast lookup of items in collection
  INDEX idx_collection_items_collection_id (collection_id),
  
  -- Find which collections contain a reel
  INDEX idx_collection_items_media_id (media_id)
);
```

**TypeORM Entity**:
```typescript
@Entity('collection_items')
@Unique(['collectionId', 'mediaId'])
@Index(['collectionId'])
@Index(['mediaId'])
export class CollectionItem {
  @PrimaryGeneratedColumn('uuid')
  id!: string;

  @Column({ type: 'uuid' })
  collectionId!: string;

  @ManyToOne(() => Collection, (collection) => collection.items, {
    onDelete: 'CASCADE',
  })
  @JoinColumn({ name: 'collectionId' })
  collection!: Collection;

  @Column({ type: 'varchar', length: 24 })
  mediaId!: string;

  @CreateDateColumn()
  addedAt!: Date;
}
```

**Key Features**:
- **Cascade Delete**: Deleting collection auto-removes all items
- **Unique Constraint**: Prevents duplicate reels in same collection
- **No FK to Reel**: mediaId references MongoDB (flexible, no constraint)

---

#### 3. saved_reels

```sql
CREATE TABLE saved_reels (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL,
  media_id VARCHAR(24) NOT NULL,  -- MongoDB ObjectId string
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  
  -- One save per user per reel
  UNIQUE (user_id, media_id),
  
  -- Fast lookup of user's saved reels
  INDEX idx_saved_reels_user_id (user_id),
  
  -- Count how many users saved a reel
  INDEX idx_saved_reels_media_id (media_id)
);
```

**TypeORM Entity**:
```typescript
@Entity('saved_reels')
@Unique(['userId', 'mediaId'])
@Index(['userId'])
@Index(['mediaId'])
export class SavedReel {
  @PrimaryGeneratedColumn('uuid')
  id!: string;

  @Column({ type: 'uuid' })
  userId!: string;

  @Column({ type: 'varchar', length: 24 })
  mediaId!: string;

  @CreateDateColumn()
  createdAt!: Date;
}
```

**Purpose**:
- Quick toggle save/unsave
- Independent of collections (can save without organizing)
- Used to sync MongoDB `savedCount`

---

### MongoDB Schema

**Reel Stats** (partial):
```javascript
{
  mediaId: ObjectId("691ca197b8be29951a3caf07"),
  userId: "550e8400-e29b-41d4-a716-446655440000",
  stats: {
    viewsCount: 1250,
    likesCount: 89,
    commentsCount: 12,
    sharesCount: 5,
    savedCount: 23  // ← Synced with saved_reels table
  }
}
```

**Sync Strategy**:
- Save action: `$inc: { 'stats.savedCount': 1 }`
- Unsave action: `$inc: { 'stats.savedCount': -1 }`
- Atomic updates prevent race conditions

---

## API Endpoints

### 1. Toggle Save

**Endpoint**: `POST /api/v1/collections/save`

**Auth**: Required (JWT)

**Request Body**:
```typescript
{
  "mediaId": "691ca197b8be29951a3caf07"
}
```

**Response** (200 OK):
```json
{
  "success": true,
  "message": "Reel saved",
  "data": {
    "saved": true,
    "totalSavedCount": 23
  }
}
```

**Behavior**:
- If not saved: Create `SavedReel`, increment MongoDB counter
- If saved: Delete `SavedReel`, decrement MongoDB counter
- Idempotent toggle

**Error Responses**:
```json
// 404 - Reel not found
{
  "success": false,
  "message": "Reel not found",
  "errorCode": "REEL_NOT_FOUND"
}
```

---

### 2. Get Saved Reels

**Endpoint**: `GET /api/v1/collections/saved`

**Auth**: Required (JWT)

**Query Parameters**:
```typescript
{
  cursor?: string;    // ISO timestamp for pagination
  limit?: number;     // Default 20, max 50
}
```

**Request**:
```
GET /v1/collections/saved?limit=20
Authorization: Bearer <token>
```

**Response** (200 OK):
```json
{
  "success": true,
  "message": "Saved reels retrieved",
  "data": {
    "items": [
      {
        "mediaId": "691ca197b8be29951a3caf07",
        "savedAt": "2026-02-14T10:30:00.000Z",
        "collectionsCount": 2
      }
    ],
    "nextCursor": "2026-02-14T09:00:00.000Z"
  }
}
```

**Sorting**: `createdAt DESC` (newest first)

**collectionsCount**: Number of user's collections containing this reel

---

### 3. Get Saved Count

**Endpoint**: `GET /api/v1/collections/saved/count/:mediaId`

**Auth**: Required (JWT)

**Path Parameters**:
- `mediaId`: MongoDB ObjectId (24 chars)

**Response** (200 OK):
```json
{
  "success": true,
  "message": "Saved count retrieved",
  "data": {
    "count": 45
  }
}
```

**Use Case**: Display "45 saves" on reel

---

### 4. Check Save Status

**Endpoint**: `GET /api/v1/collections/saved/status/:mediaId`

**Auth**: Required (JWT)

**Response** (200 OK):
```json
{
  "success": true,
  "message": "Save status retrieved",
  "data": {
    "saved": true
  }
}
```

**Use Case**: Show filled/empty bookmark icon

---

### 5. Get My Collections

**Endpoint**: `GET /api/v1/collections/my`

**Auth**: Required (JWT)

**Response** (200 OK):
```json
{
  "success": true,
  "message": "Collections retrieved",
  "data": {
    "collections": [
      {
        "id": "550e8400-e29b-41d4-a716-446655440000",
        "userId": "550e8400-e29b-41d4-a716-446655440001",
        "name": "Favorite Recipes",
        "emoji": "📌",
        "itemCount": 12,
        "createdAt": "2026-02-10T08:00:00.000Z",
        "updatedAt": "2026-02-14T10:30:00.000Z"
      }
    ]
  }
}
```

**Sorting**: `createdAt DESC`

**itemCount**: Computed from `collection.items.length`

---

### 6. Create Collection

**Endpoint**: `POST /api/v1/collections/create`

**Auth**: Required (JWT)

**Request Body**:
```typescript
{
  "name": "Italian Cuisine",
  "emoji": "🍝"  // Optional
}
```

**Validation**:
- `name`: Required, 1-60 chars
- `emoji`: Optional, max 10 chars

**Response** (201 Created):
```json
{
  "success": true,
  "message": "Collection created",
  "data": {
    "collection": {
      "id": "550e8400-e29b-41d4-a716-446655440002",
      "userId": "550e8400-e29b-41d4-a716-446655440001",
      "name": "Italian Cuisine",
      "emoji": "🍝",
      "itemCount": 0,
      "createdAt": "2026-02-14T10:35:00.000Z",
      "updatedAt": "2026-02-14T10:35:00.000Z"
    }
  }
}
```

**Error Responses**:
```json
// 400 - Collection limit reached
{
  "success": false,
  "message": "Maximum 200 collections allowed",
  "errorCode": "COLLECTION_LIMIT_REACHED"
}
```

---

### 7. Rename Collection

**Endpoint**: `PATCH /api/v1/collections/:id/rename`

**Auth**: Required (JWT)

**Path Parameters**:
- `id`: Collection UUID

**Request Body**:
```typescript
{
  "name": "Quick Dinners",  // Required
  "emoji": "⚡"              // Optional
}
```

**Response** (200 OK):
```json
{
  "success": true,
  "message": "Collection renamed",
  "data": {
    "collection": {
      "id": "550e8400-e29b-41d4-a716-446655440002",
      "name": "Quick Dinners",
      "emoji": "⚡",
      "itemCount": 5,
      "updatedAt": "2026-02-14T10:40:00.000Z"
    }
  }
}
```

**Error Responses**:
```json
// 403 - Not owner
{
  "success": false,
  "message": "Cannot modify another user's collection",
  "errorCode": "FORBIDDEN"
}

// 404 - Collection not found
{
  "success": false,
  "message": "Collection not found",
  "errorCode": "COLLECTION_NOT_FOUND"
}
```

---

### 8. Delete Collection

**Endpoint**: `DELETE /api/v1/collections/:id`

**Auth**: Required (JWT)

**Response** (200 OK):
```json
{
  "success": true,
  "message": "Collection deleted",
  "data": null
}
```

**Side Effects**:
- Cascade delete removes all `collection_items`
- Saved reels remain intact (not unsaved)

---

### 9. Add to Collection

**Endpoint**: `POST /api/v1/collections/:collectionId/add`

**Auth**: Required (JWT)

**Request Body**:
```typescript
{
  "mediaId": "691ca197b8be29951a3caf07"
}
```

**Response** (200 OK):
```json
{
  "success": true,
  "message": "Added to collection",
  "data": null
}
```

**Idempotency**: If reel already in collection, returns success (no error)

**Error Responses**:
```json
// 403 - Not owner
{
  "success": false,
  "message": "Cannot modify another user's collection",
  "errorCode": "FORBIDDEN"
}

// 404 - Reel not found
{
  "success": false,
  "message": "Reel not found",
  "errorCode": "REEL_NOT_FOUND"
}
```

---

### 10. Remove from Collection

**Endpoint**: `DELETE /api/v1/collections/:collectionId/remove/:mediaId`

**Auth**: Required (JWT)

**Path Parameters**:
- `collectionId`: Collection UUID
- `mediaId`: MongoDB Reel ID

**Response** (200 OK):
```json
{
  "success": true,
  "message": "Removed from collection",
  "data": null
}
```

**Behavior**:
- Deletes `collection_items` record
- Does NOT unsave reel (saved status preserved)
- Idempotent (no error if not in collection)

---

### 11. Get Collection Items

**Endpoint**: `GET /api/v1/collections/:collectionId/items`

**Auth**: Required (JWT)

**Query Parameters**:
```typescript
{
  cursor?: string;   // ISO timestamp
  limit?: number;    // Default 20
}
```

**Response** (200 OK):
```json
{
  "success": true,
  "message": "Collection items retrieved",
  "data": {
    "mediaIds": [
      "691ca197b8be29951a3caf07",
      "691ca197b8be29951a3caf08"
    ],
    "nextCursor": "2026-02-14T09:00:00.000Z"
  }
}
```

**Returns**: List of `mediaId` values (not full reel data)

**Frontend**: Must fetch reel details from Reels API

---

### 12. Get Collections for Reel

**Endpoint**: `GET /api/v1/collections/reel/:mediaId/collections`

**Auth**: Required (JWT)

**Response** (200 OK):
```json
{
  "success": true,
  "message": "Collections retrieved",
  "data": {
    "collections": [
      {
        "id": "550e8400-e29b-41d4-a716-446655440002",
        "name": "Favorites",
        "emoji": "⭐",
        "itemCount": 8
      }
    ]
  }
}
```

**Use Case**: "Add to Collection" modal shows checkmarks

---

## Service Methods

### CollectionsService Core Methods

#### toggleSave()

```typescript
async toggleSave(
  userId: string,
  dto: ToggleSaveDto
): Promise<ToggleSaveResponseDto>
```

**Flow**:
1. Verify reel exists in MongoDB
2. Check if already saved (`saved_reels` table)
3. If not saved:
   - Insert `SavedReel` record
   - Increment `reel.stats.savedCount` in MongoDB
4. If saved:
   - Delete `SavedReel` record
   - Decrement `reel.stats.savedCount`
5. Return new state

**Example**:
```typescript
const result = await collectionsService.toggleSave(
  "550e8400-e29b-41d4-a716-446655440001",
  { mediaId: "691ca197b8be29951a3caf07" }
);
// { saved: true, totalSavedCount: 24 }
```

**MongoDB Update** (Atomic):
```typescript
const updatedReel = await this.reelModel.findOneAndUpdate(
  { mediaId: dto.mediaId },
  { $inc: { 'stats.savedCount': 1 } },  // or -1 for unsave
  { new: true }
);
```

---

#### getSavedReels()

```typescript
async getSavedReels(
  userId: string,
  cursor?: string,
  limit: number = 20
): Promise<{ items: SavedReelResponseDto[]; nextCursor: string | null }>
```

**Query**:
```typescript
const queryBuilder = this.savedReelRepository
  .createQueryBuilder('saved')
  .where('saved.userId = :userId', { userId })
  .orderBy('saved.createdAt', 'DESC')
  .limit(limit + 1);

if (cursor) {
  queryBuilder.andWhere('saved.createdAt < :cursor', {
    cursor: new Date(cursor),
  });
}
```

**collectionsCount** Computation:
```typescript
for (const saved of savedReels) {
  const collectionsCount = await this.collectionItemRepository.count({
    where: { mediaId: saved.mediaId },
  });
  
  items.push({
    mediaId: saved.mediaId,
    savedAt: saved.createdAt,
    collectionsCount,
  });
}
```

**Pagination**:
- Cursor-based (ISO timestamp)
- Fetch `limit + 1` items
- If `length > limit`, has more pages

---

#### createCollection()

```typescript
async createCollection(
  userId: string,
  dto: CreateCollectionDto
): Promise<CollectionResponseDto>
```

**Validation**:
```typescript
// Check collection limit
const existingCount = await this.collectionRepository.count({
  where: { userId },
});

if (existingCount >= this.MAX_COLLECTIONS_PER_USER) {
  throw new BadRequestException(
    'Maximum 200 collections allowed'
  );
}
```

**Creation**:
```typescript
const collection = this.collectionRepository.create({
  userId,
  name: dto.name,
  emoji: dto.emoji || null,
});

const saved = await this.collectionRepository.save(collection);
```

---

#### addToCollection()

```typescript
async addToCollection(
  userId: string,
  collectionId: string,
  dto: AddToCollectionDto
): Promise<void>
```

**Authorization**:
```typescript
const collection = await this.collectionRepository.findOne({
  where: { id: collectionId },
});

if (collection.userId !== userId) {
  throw new ForbiddenException('Cannot modify another user\'s collection');
}
```

**Idempotent Add**:
```typescript
// Check if already in collection
const existing = await this.collectionItemRepository.findOne({
  where: { collectionId, mediaId: dto.mediaId },
});

if (existing) {
  return;  // Already added, return success
}

// Add to collection
const item = this.collectionItemRepository.create({
  collectionId,
  mediaId: dto.mediaId,
});

await this.collectionItemRepository.save(item);
```

---

#### getCollectionItems()

```typescript
async getCollectionItems(
  userId: string,
  collectionId: string,
  cursor?: string,
  limit: number = 20
): Promise<{ mediaIds: string[]; nextCursor: string | null }>
```

**Authorization Check**:
```typescript
const collection = await this.collectionRepository.findOne({
  where: { id: collectionId },
});

if (collection.userId !== userId) {
  throw new ForbiddenException('Cannot view another user\'s collection');
}
```

**Query**:
```typescript
const queryBuilder = this.collectionItemRepository
  .createQueryBuilder('item')
  .where('item.collectionId = :collectionId', { collectionId })
  .orderBy('item.addedAt', 'DESC')
  .limit(limit + 1);

if (cursor) {
  queryBuilder.andWhere('item.addedAt < :cursor', {
    cursor: new Date(cursor),
  });
}
```

**Returns**: List of `mediaId` strings (not full reel objects)

---

#### getCollectionsForReel()

```typescript
async getCollectionsForReel(
  userId: string,
  mediaId: string
): Promise<CollectionResponseDto[]>
```

**Query**:
```typescript
const items = await this.collectionItemRepository
  .createQueryBuilder('item')
  .leftJoinAndSelect('item.collection', 'collection')
  .where('item.mediaId = :mediaId', { mediaId })
  .andWhere('collection.userId = :userId', { userId })
  .getMany();
```

**Returns**: Collections containing this reel (for current user)

---

## Business Logic

### Save/Unsave Toggle

```typescript
// Check if already saved
const existingSave = await this.savedReelRepository.findOne({
  where: { userId, mediaId: dto.mediaId },
});

if (existingSave) {
  // UNSAVE
  await this.savedReelRepository.remove(existingSave);
  
  // Decrement MongoDB counter
  const updatedReel = await this.reelModel.findOneAndUpdate(
    { mediaId: dto.mediaId },
    { $inc: { 'stats.savedCount': -1 } },
    { new: true }
  );
  
  return {
    saved: false,
    totalSavedCount: updatedReel?.stats?.savedCount || 0,
  };
} else {
  // SAVE
  const newSave = this.savedReelRepository.create({
    userId,
    mediaId: dto.mediaId,
  });
  await this.savedReelRepository.save(newSave);
  
  // Increment MongoDB counter
  const updatedReel = await this.reelModel.findOneAndUpdate(
    { mediaId: dto.mediaId },
    { $inc: { 'stats.savedCount': 1 } },
    { new: true }
  );
  
  return {
    saved: true,
    totalSavedCount: updatedReel?.stats?.savedCount || 0,
  };
}
```

**Key Points**:
- Atomic MongoDB updates (`$inc`)
- Idempotent (check before insert/delete)
- Returns current state

---

### Collection Limit Enforcement

```typescript
const MAX_COLLECTIONS_PER_USER = 200;

const existingCount = await this.collectionRepository.count({
  where: { userId },
});

if (existingCount >= this.MAX_COLLECTIONS_PER_USER) {
  throw new BadRequestException(
    `Maximum ${this.MAX_COLLECTIONS_PER_USER} collections allowed`
  );
}
```

**Why 200?**
- Reasonable limit for personal organization
- Prevents abuse (spam collections)
- Database performance consideration

---

### Cascade Delete Behavior

**Database-Level**:
```typescript
@ManyToOne(() => Collection, (collection) => collection.items, {
  onDelete: 'CASCADE',  // Database cascade
})
@JoinColumn({ name: 'collectionId' })
collection!: Collection;
```

**What Happens**:
1. User deletes collection
2. Database automatically deletes all `collection_items` with matching `collectionId`
3. No orphaned items
4. Saved status of reels unchanged

**SQL Equivalent**:
```sql
DELETE FROM collections WHERE id = :collectionId;
-- Automatically triggers:
-- DELETE FROM collection_items WHERE collection_id = :collectionId;
```

---

## Error Handling

### Standard Error Responses

```typescript
// 400 Bad Request
{
  "success": false,
  "message": "Maximum 200 collections allowed",
  "errorCode": "COLLECTION_LIMIT_REACHED"
}

// 403 Forbidden
{
  "success": false,
  "message": "Cannot modify another user's collection",
  "errorCode": "FORBIDDEN"
}

// 404 Not Found
{
  "success": false,
  "message": "Reel not found",
  "errorCode": "REEL_NOT_FOUND"
}
```

### Error Handling Patterns

```typescript
try {
  const collection = await collectionRepository.findOne({
    where: { id: collectionId },
  });
  
  if (!collection) {
    throw new NotFoundException('Collection not found');
  }
  
  if (collection.userId !== userId) {
    throw new ForbiddenException('Cannot modify another user\'s collection');
  }
  
  // ... business logic
} catch (error) {
  logger.error(`Failed to update collection: ${error.message}`, error.stack);
  throw error;
}
```

### Graceful Degradation

```typescript
// MongoDB sync failure doesn't block save
try {
  await this.reelModel.findOneAndUpdate(
    { mediaId: dto.mediaId },
    { $inc: { 'stats.savedCount': 1 } }
  );
} catch (error) {
  logger.error('Failed to sync savedCount in MongoDB', error);
  // Continue - PostgreSQL save succeeded
}
```

---

## Performance Optimization

### Database Indexes

```sql
-- Fast lookup of user's collections
CREATE INDEX idx_collections_user_id ON collections(user_id);

-- Fast lookup of user's saved reels
CREATE INDEX idx_saved_reels_user_id ON saved_reels(user_id);

-- Count how many users saved a reel
CREATE INDEX idx_saved_reels_media_id ON saved_reels(media_id);

-- Fast lookup of items in collection
CREATE INDEX idx_collection_items_collection_id ON collection_items(collection_id);

-- Find which collections contain a reel
CREATE INDEX idx_collection_items_media_id ON collection_items(media_id);
```

### Unique Constraints

```sql
-- Prevent duplicate saves
ALTER TABLE saved_reels 
  ADD CONSTRAINT uq_saved_reels_user_media 
  UNIQUE (user_id, media_id);

-- Prevent duplicate items in collection
ALTER TABLE collection_items 
  ADD CONSTRAINT uq_collection_items_collection_media 
  UNIQUE (collection_id, media_id);
```

### Cursor-Based Pagination

```typescript
// Efficient pagination (no OFFSET)
const queryBuilder = savedReelRepository
  .createQueryBuilder('saved')
  .where('saved.userId = :userId', { userId })
  .orderBy('saved.createdAt', 'DESC')
  .limit(limit + 1);

if (cursor) {
  queryBuilder.andWhere('saved.createdAt < :cursor', {
    cursor: new Date(cursor),
  });
}
```

**Why Cursor?**
- No OFFSET (fast for large datasets)
- Consistent results (no skipped items)
- Handles concurrent inserts gracefully

### Denormalized Count

```typescript
// Avoid COUNT(*) query
const itemCount = collection.items?.length || 0;  // From relation

// vs expensive query:
const itemCount = await collectionItemRepository.count({
  where: { collectionId },
});  // ❌ Slow
```

---

## Testing

### Unit Tests

```typescript
describe('CollectionsService', () => {
  describe('toggleSave', () => {
    it('should save reel when not saved', async () => {
      const dto = { mediaId: 'reel-id' };
      
      const result = await service.toggleSave('user-1', dto);
      
      expect(result.saved).toBe(true);
      expect(result.totalSavedCount).toBe(1);
      expect(savedReelRepo.save).toHaveBeenCalled();
      expect(reelModel.findOneAndUpdate).toHaveBeenCalledWith(
        { mediaId: 'reel-id' },
        { $inc: { 'stats.savedCount': 1 } },
        { new: true }
      );
    });
    
    it('should unsave reel when already saved', async () => {
      const existingSave = { id: 'save-id', userId: 'user-1', mediaId: 'reel-id' };
      savedReelRepo.findOne.mockResolvedValue(existingSave);
      
      const result = await service.toggleSave('user-1', { mediaId: 'reel-id' });
      
      expect(result.saved).toBe(false);
      expect(savedReelRepo.remove).toHaveBeenCalledWith(existingSave);
    });
  });
  
  describe('createCollection', () => {
    it('should enforce collection limit', async () => {
      collectionRepo.count.mockResolvedValue(200);
      
      await expect(
        service.createCollection('user-1', { name: 'New Collection' })
      ).rejects.toThrow('Maximum 200 collections allowed');
    });
  });
  
  describe('addToCollection', () => {
    it('should be idempotent', async () => {
      const existing = { id: 'item-id', collectionId: 'coll-1', mediaId: 'reel-1' };
      collectionItemRepo.findOne.mockResolvedValue(existing);
      
      await service.addToCollection('user-1', 'coll-1', { mediaId: 'reel-1' });
      
      expect(collectionItemRepo.save).not.toHaveBeenCalled();
    });
  });
});
```

### Integration Tests

```typescript
describe('CollectionsController (e2e)', () => {
  it('POST /collections/save should save reel', async () => {
    const response = await request(app.getHttpServer())
      .post('/v1/collections/save')
      .set('Authorization', `Bearer ${userToken}`)
      .send({ mediaId: reelId })
      .expect(200);
    
    expect(response.body.success).toBe(true);
    expect(response.body.data.saved).toBe(true);
  });
  
  it('should prevent modifying another user\'s collection', async () => {
    // User1 creates collection
    const { body } = await request(app.getHttpServer())
      .post('/v1/collections/create')
      .set('Authorization', `Bearer ${user1Token}`)
      .send({ name: 'My Collection' })
      .expect(201);
    
    const collectionId = body.data.collection.id;
    
    // User2 tries to rename
    await request(app.getHttpServer())
      .patch(`/v1/collections/${collectionId}/rename`)
      .set('Authorization', `Bearer ${user2Token}`)
      .send({ name: 'Stolen Collection' })
      .expect(403);
  });
});
```

---

## Related Documentation

- **Feature Overview**: `01_FEATURE_OVERVIEW.md` (Business context)
- **QA Test Cases**: `03_QA_TEST_CASES.md` (Test scenarios)
- **Reels Module**: `../reels/02_TECHNICAL_GUIDE.md` (Content being saved)

---

**[TECHNICAL_COMPLETE ✅]**  
**Module**: Collections  
**Documentation**: Technical Guide  
**Date**: February 14, 2026
