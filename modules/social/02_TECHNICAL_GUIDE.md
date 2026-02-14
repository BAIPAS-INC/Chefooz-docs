# Social Module - Technical Guide

**Module**: `social`  
**Type**: Core Feature  
**Category**: Social & Discovery  
**Status**: Production  
**Last Updated**: February 14, 2026

---

## 📋 Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Database Schema](#database-schema)
3. [API Endpoints](#api-endpoints)
4. [Service Layer](#service-layer)
5. [Data Transfer Objects (DTOs)](#data-transfer-objects-dtos)
6. [Entity Relationships](#entity-relationships)
7. [Caching Strategy](#caching-strategy)
8. [Notification Integration](#notification-integration)
9. [Error Handling](#error-handling)
10. [Performance Optimization](#performance-optimization)
11. [Code Examples](#code-examples)

---

## Architecture Overview

### Module Structure

```
apps/chefooz-apis/src/modules/social/
├── dto/
│   ├── social.dto.ts              # Follow, block, privacy DTOs
│   └── share-reel.dto.ts          # Reel sharing DTOs
├── entities/
│   ├── user-follow.entity.ts      # Follow relationships
│   ├── user-privacy.entity.ts     # Privacy settings + counters
│   └── user-block.entity.ts       # Block relationships
├── social.controller.ts           # Social graph endpoints
├── social.service.ts              # Social graph business logic
├── reels-share.controller.ts      # Reel sharing endpoints
├── reels-share.service.ts         # Share tracking logic
└── social.module.ts               # Module configuration
```

### Technology Stack

**Backend**:
- NestJS 10+ (TypeScript)
- PostgreSQL 15+ (via TypeORM)
- MongoDB 7+ (via Mongoose, for reels)
- Valkey/Redis (caching)

**Key Libraries**:
- TypeORM (PostgreSQL ORM)
- Mongoose (MongoDB ODM)
- class-validator (DTO validation)
- @nestjs/cache-manager (caching)

### Module Dependencies

```typescript
@Module({
  imports: [
    // PostgreSQL entities
    TypeOrmModule.forFeature([
      UserFollow,
      UserPrivacy,
      UserBlock,
      User,
      ChefProfile,
    ]),
    
    // MongoDB schema
    MongooseModule.forFeature([
      { name: Reel.name, schema: ReelSchema },
    ]),
    
    // External modules
    CacheModule,
    NotificationModule,
    DeeplinkModule,
    ActivityModule,
  ],
  controllers: [SocialController, ReelsShareController],
  providers: [SocialService, ReelsShareService],
  exports: [SocialService, ReelsShareService],
})
export class SocialModule {}
```

---

## Database Schema

### 1. UserFollow Entity

**Table**: `user_follow`  
**Purpose**: Tracks follow relationships with Instagram-style pending/accepted flow

```typescript
@Entity('user_follow')
@Unique(['followerId', 'targetId'])
@Index(['targetId'])
@Index(['followerId'])
@Index(['status'])
export class UserFollow {
  @PrimaryGeneratedColumn('uuid')
  id!: string;

  // User who initiated the follow
  @Column({ type: 'uuid' })
  followerId!: string;

  // User being followed
  @Column({ type: 'uuid' })
  targetId!: string;

  // Status: 'accepted' (active follow) or 'pending' (awaiting approval)
  @Column({
    type: 'enum',
    enum: ['accepted', 'pending'],
    default: 'pending',
  })
  status!: 'accepted' | 'pending';

  @CreateDateColumn()
  createdAt!: Date;

  @UpdateDateColumn()
  updatedAt!: Date;

  // Relations
  @ManyToOne(() => User)
  @JoinColumn({ name: 'followerId' })
  follower!: User;

  @ManyToOne(() => User)
  @JoinColumn({ name: 'targetId' })
  target!: User;
}
```

**Indexes**:
- **UNIQUE (followerId, targetId)**: One follow per pair, prevents duplicates
- **INDEX (targetId)**: Fast follower list retrieval
- **INDEX (followerId)**: Fast following list retrieval
- **INDEX (status)**: Filter by pending/accepted

**Status Flow**:
- Public account: `null` → `'accepted'` (immediate)
- Private account: `null` → `'pending'` → `'accepted'` (approval required)

---

### 2. UserPrivacy Entity

**Table**: `user_privacy`  
**Purpose**: Stores privacy settings and denormalized follower/following counters

```typescript
@Entity('user_privacy')
@Index(['userId'], { unique: true })
export class UserPrivacy {
  @PrimaryGeneratedColumn('uuid')
  id!: string;

  // User this privacy record belongs to
  @Column({ type: 'uuid', unique: true })
  userId!: string;

  // Account privacy setting (false = public, true = private)
  @Column({ type: 'boolean', default: false })
  isPrivate!: boolean;

  // Denormalized counter: number of accepted followers
  @Column({ type: 'int', default: 0 })
  followersCount!: number;

  // Denormalized counter: number of accepted follows
  @Column({ type: 'int', default: 0 })
  followingCount!: number;

  @CreateDateColumn()
  createdAt!: Date;

  @UpdateDateColumn()
  updatedAt!: Date;
}
```

**Indexes**:
- **UNIQUE (userId)**: One privacy record per user

**Default Values**:
- `isPrivate = false` (public account)
- `followersCount = 0`
- `followingCount = 0`

**Counter Management**:
- Incremented on follow accept (status → 'accepted')
- Decremented on unfollow (status = 'accepted')
- Decremented on block (removes accepted follows)

---

### 3. UserBlock Entity

**Table**: `user_block`  
**Purpose**: Tracks blocked users with bidirectional effect

```typescript
@Entity('user_block')
@Unique(['userId', 'blockedUserId'])
@Index(['userId'])
@Index(['blockedUserId'])
export class UserBlock {
  @PrimaryGeneratedColumn('uuid')
  id!: string;

  // User who initiated the block
  @Column({ type: 'uuid' })
  userId!: string;

  // User being blocked
  @Column({ type: 'uuid' })
  blockedUserId!: string;

  @CreateDateColumn()
  createdAt!: Date;
}
```

**Indexes**:
- **UNIQUE (userId, blockedUserId)**: One block per pair, prevents duplicates
- **INDEX (userId)**: Fast block list retrieval
- **INDEX (blockedUserId)**: Check if user is blocked by someone

**Block Effects**:
- Remove all follow relationships (both directions)
- Prevent future follows
- Hide profile/content
- Exclude from search results

---

## API Endpoints

### 1. Follow Management

#### POST /api/v1/social/follow/:targetId

**Purpose**: Follow a user (auto-accept public, pending private)

**Authentication**: JWT required

**Rate Limit**: 30 requests/minute

**Request**:
```typescript
// Path parameter
targetId: string (UUID or username)

// Headers
Authorization: Bearer <jwt_token>
```

**Response (Public Account)**:
```json
{
  "success": true,
  "message": "Successfully followed user",
  "data": {
    "status": "accepted",
    "targetUserId": "d1c2d2f6-6823-4ae3-99b8-f7e31c2bbbe5",
    "isPrivate": false
  }
}
```

**Response (Private Account)**:
```json
{
  "success": true,
  "message": "Follow request sent",
  "data": {
    "status": "pending",
    "targetUserId": "a3b4c5d6-7890-4321-8765-4321fedcba98",
    "isPrivate": true
  }
}
```

**Error Codes**:
- `400`: SOCIAL_ALREADY_FOLLOWING, SOCIAL_SELF_FOLLOW_NOT_ALLOWED
- `403`: SOCIAL_USER_BLOCKED
- `404`: SOCIAL_TARGET_USER_NOT_FOUND
- `429`: FOLLOW_RATE_LIMIT_EXCEEDED

**Implementation**:
```typescript
@Post('follow/:targetId')
@HttpCode(HttpStatus.OK)
@RateLimit(30, 60) // 30 requests per 60 seconds
@ApiOperation({ summary: 'Follow a user' })
async followUser(
  @Request() req: any,
  @Param('targetId') targetId: string,
) {
  const userId = req.user.id;
  const result = await this.socialService.followUser(userId, targetId);
  return {
    success: true,
    message: result.status === 'accepted'
      ? 'Successfully followed user'
      : 'Follow request sent',
    data: result,
  };
}
```

---

#### POST /api/v1/social/unfollow/:targetId

**Purpose**: Unfollow a user

**Authentication**: JWT required

**Rate Limit**: 30 requests/minute

**Request**:
```typescript
// Path parameter
targetId: string (UUID or username)

// Headers
Authorization: Bearer <jwt_token>
```

**Response**:
```json
{
  "success": true,
  "message": "Successfully unfollowed user",
  "data": {
    "targetUserId": "d1c2d2f6-6823-4ae3-99b8-f7e31c2bbbe5"
  }
}
```

**Error Codes**:
- `400`: SOCIAL_NOT_FOLLOWING
- `404`: SOCIAL_TARGET_USER_NOT_FOUND
- `429`: UNFOLLOW_RATE_LIMIT_EXCEEDED

---

#### POST /api/v1/social/requests/:requestId/accept

**Purpose**: Accept a pending follow request

**Authentication**: JWT required

**Request**:
```typescript
// Path parameter
requestId: string (UUID of UserFollow record)

// Headers
Authorization: Bearer <jwt_token>
```

**Response**:
```json
{
  "success": true,
  "message": "Follow request accepted",
  "data": {
    "requestId": "f7e31c2b-bbe5-4321-8765-d1c2d2f66823",
    "followerId": "a3b4c5d6-7890-4321-8765-4321fedcba98"
  }
}
```

**Error Codes**:
- `404`: SOCIAL_REQUEST_NOT_FOUND

**Implementation**:
```typescript
@Post('requests/:requestId/accept')
@HttpCode(HttpStatus.OK)
@ApiOperation({ summary: 'Accept a follow request' })
async acceptFollowRequest(
  @Request() req: any,
  @Param('requestId') requestId: string,
) {
  const userId = req.user.id;
  const result = await this.socialService.acceptFollowRequest(
    userId,
    requestId,
  );
  return {
    success: true,
    message: 'Follow request accepted',
    data: result,
  };
}
```

---

#### POST /api/v1/social/requests/:requestId/reject

**Purpose**: Reject a pending follow request

**Authentication**: JWT required

**Request**:
```typescript
// Path parameter
requestId: string (UUID)

// Headers
Authorization: Bearer <jwt_token>
```

**Response**:
```json
{
  "success": true,
  "message": "Follow request rejected",
  "data": {
    "requestId": "f7e31c2b-bbe5-4321-8765-d1c2d2f66823"
  }
}
```

**Error Codes**:
- `404`: SOCIAL_REQUEST_NOT_FOUND

---

#### GET /api/v1/social/relationship/:targetId

**Purpose**: Get relationship state with target user (6 boolean flags)

**Authentication**: JWT required

**Request**:
```typescript
// Path parameter
targetId: string (UUID or username)

// Headers
Authorization: Bearer <jwt_token>
```

**Response**:
```json
{
  "success": true,
  "message": "Relationship retrieved",
  "data": {
    "isFollowing": true,
    "isFollowedBy": true,
    "isRequested": false,
    "isPrivate": false,
    "isBlockedByMe": false,
    "hasBlockedMe": false
  }
}
```

**Field Definitions**:
- `isFollowing`: Current user follows target (status='accepted')
- `isFollowedBy`: Target follows current user (status='accepted')
- `isRequested`: Current user sent pending follow request (status='pending')
- `isPrivate`: Target account is private
- `isBlockedByMe`: Current user blocked target
- `hasBlockedMe`: Target blocked current user

---

#### POST /api/v1/social/remove-follower/:followerId

**Purpose**: Remove a follower (Instagram-style)

**Authentication**: JWT required

**Request**:
```typescript
// Path parameter
followerId: string (UUID)

// Headers
Authorization: Bearer <jwt_token>
```

**Response**:
```json
{
  "success": true,
  "message": "Follower removed",
  "data": {
    "followerId": "a3b4c5d6-7890-4321-8765-4321fedcba98"
  }
}
```

**Error Codes**:
- `400`: SOCIAL_NOT_FOLLOWING (follower doesn't follow you)
- `404`: SOCIAL_TARGET_USER_NOT_FOUND

---

### 2. Follow Lists

#### GET /api/v1/social/requests/pending

**Purpose**: Get pending follow requests

**Authentication**: JWT required

**Request**:
```typescript
// Query parameters
cursor?: string (ISO date string)
limit?: number (default: 20, max: 100)

// Headers
Authorization: Bearer <jwt_token>
```

**Response**:
```json
{
  "success": true,
  "message": "Pending requests retrieved",
  "data": {
    "items": [
      {
        "id": "f7e31c2b-bbe5-4321-8765-d1c2d2f66823",
        "followerId": "a3b4c5d6-7890-4321-8765-4321fedcba98",
        "createdAt": "2026-02-14T10:30:00.000Z",
        "follower": {
          "id": "a3b4c5d6-7890-4321-8765-4321fedcba98",
          "username": "john_chef",
          "fullName": "John Doe",
          "avatar": "https://cdn.chefooz.app/avatars/john.jpg"
        }
      }
    ],
    "nextCursor": "2026-02-13T08:15:00.000Z",
    "hasMore": true
  }
}
```

**Pagination**:
- Cursor-based (createdAt DESC)
- Returns `nextCursor` for next page
- `hasMore` indicates more results available

---

#### GET /api/v1/social/followers

**Purpose**: Get own followers list

**Authentication**: JWT required

**Request**:
```typescript
// Query parameters
cursor?: string (ISO date string)
limit?: number (default: 20, max: 100)

// Headers
Authorization: Bearer <jwt_token>
```

**Response**:
```json
{
  "success": true,
  "message": "Followers retrieved",
  "data": {
    "items": [
      {
        "id": "a3b4c5d6-7890-4321-8765-4321fedcba98",
        "username": "john_chef",
        "fullName": "John Doe",
        "avatar": "https://cdn.chefooz.app/avatars/john.jpg",
        "isFollowing": true,
        "isFollowedBy": true
      }
    ],
    "nextCursor": "2026-02-13T08:15:00.000Z",
    "hasMore": true
  }
}
```

---

#### GET /api/v1/social/followers/:userIdOrUsername

**Purpose**: Get user's followers list

**Authentication**: JWT required

**Request**:
```typescript
// Path parameter
userIdOrUsername: string (UUID or username)

// Query parameters
cursor?: string (ISO date string)
limit?: number (default: 20, max: 100)

// Headers
Authorization: Bearer <jwt_token>
```

**Response**: Same as GET /followers

**Error Codes**:
- `404`: SOCIAL_TARGET_USER_NOT_FOUND

---

#### GET /api/v1/social/following

**Purpose**: Get own following list

**Authentication**: JWT required

**Request**:
```typescript
// Query parameters
cursor?: string (ISO date string)
limit?: number (default: 20, max: 100)

// Headers
Authorization: Bearer <jwt_token>
```

**Response**:
```json
{
  "success": true,
  "message": "Following retrieved",
  "data": {
    "items": [
      {
        "id": "d1c2d2f6-6823-4ae3-99b8-f7e31c2bbbe5",
        "username": "chef_maria",
        "fullName": "Maria Chef",
        "avatar": "https://cdn.chefooz.app/avatars/maria.jpg",
        "isFollowing": true,
        "isFollowedBy": false
      }
    ],
    "nextCursor": "2026-02-12T15:45:00.000Z",
    "hasMore": false
  }
}
```

---

#### GET /api/v1/social/following/:userIdOrUsername

**Purpose**: Get user's following list

**Authentication**: JWT required

**Request**:
```typescript
// Path parameter
userIdOrUsername: string (UUID or username)

// Query parameters
cursor?: string (ISO date string)
limit?: number (default: 20, max: 100)

// Headers
Authorization: Bearer <jwt_token>
```

**Response**: Same as GET /following

---

### 3. Blocking

#### POST /api/v1/social/block/:targetId

**Purpose**: Block a user (removes all follows, prevents interactions)

**Authentication**: JWT required

**Rate Limit**: 20 requests/minute

**Request**:
```typescript
// Path parameter
targetId: string (UUID)

// Headers
Authorization: Bearer <jwt_token>
```

**Response**:
```json
{
  "success": true,
  "message": "User blocked successfully",
  "data": {
    "blockedUserId": "a3b4c5d6-7890-4321-8765-4321fedcba98"
  }
}
```

**Error Codes**:
- `400`: SOCIAL_SELF_BLOCK_NOT_ALLOWED
- `404`: SOCIAL_TARGET_USER_NOT_FOUND
- `429`: BLOCK_RATE_LIMIT_EXCEEDED

**Side Effects**:
- Removes all follow relationships (both directions)
- Decrements counters for removed follows
- Invalidates cache for both users
- No notification sent (soft block)

---

#### POST /api/v1/social/unblock/:targetId

**Purpose**: Unblock a user

**Authentication**: JWT required

**Request**:
```typescript
// Path parameter
targetId: string (UUID)

// Headers
Authorization: Bearer <jwt_token>
```

**Response**:
```json
{
  "success": true,
  "message": "User unblocked successfully",
  "data": {
    "unblockedUserId": "a3b4c5d6-7890-4321-8765-4321fedcba98"
  }
}
```

**Error Codes**:
- `400`: SOCIAL_NOT_BLOCKED (user not blocked)
- `404`: SOCIAL_TARGET_USER_NOT_FOUND

---

#### GET /api/v1/social/blocked/check/:targetId

**Purpose**: Check block status with target user

**Authentication**: JWT required

**Request**:
```typescript
// Path parameter
targetId: string (UUID)

// Headers
Authorization: Bearer <jwt_token>
```

**Response**:
```json
{
  "success": true,
  "message": "Block status retrieved",
  "data": {
    "isBlocked": true,
    "hasBlocked": true,
    "isBlockedBy": false
  }
}
```

**Field Definitions**:
- `isBlocked`: Blocked in either direction
- `hasBlocked`: Current user blocked target
- `isBlockedBy`: Target blocked current user

---

#### GET /api/v1/social/blocked

**Purpose**: Get blocked users list

**Authentication**: JWT required

**Request**:
```typescript
// Query parameters
cursor?: string (ISO date string)
limit?: number (default: 20, max: 100)

// Headers
Authorization: Bearer <jwt_token>
```

**Response**:
```json
{
  "success": true,
  "message": "Blocked users retrieved",
  "data": {
    "items": [
      {
        "id": "a3b4c5d6-7890-4321-8765-4321fedcba98",
        "username": "spammer123",
        "fullName": "Spammer User",
        "avatar": null,
        "blockedAt": "2026-02-10T12:00:00.000Z"
      }
    ],
    "nextCursor": "2026-02-09T09:30:00.000Z",
    "hasMore": false
  }
}
```

---

### 4. Privacy Settings

#### GET /api/v1/social/profile/privacy

**Purpose**: Get own privacy settings

**Authentication**: JWT required

**Request**:
```typescript
// Headers
Authorization: Bearer <jwt_token>
```

**Response**:
```json
{
  "success": true,
  "message": "Privacy settings retrieved",
  "data": {
    "isPrivate": false,
    "followersCount": 142,
    "followingCount": 85
  }
}
```

---

#### POST /api/v1/social/profile/privacy

**Purpose**: Update privacy settings

**Authentication**: JWT required

**Request**:
```typescript
// Headers
Authorization: Bearer <jwt_token>

// Body
{
  "isPrivate": true
}
```

**Response**:
```json
{
  "success": true,
  "message": "Privacy settings updated",
  "data": {
    "isPrivate": true,
    "followersCount": 142,
    "followingCount": 85
  }
}
```

**Side Effects**:
- Invalidates cache: `social:privacy:{userId}`
- Existing followers remain unaffected
- Future follows require approval if toggled to private

---

#### GET /api/v1/social/profile/:userId

**Purpose**: Get social profile projection (enriched user data)

**Authentication**: JWT required

**Request**:
```typescript
// Path parameter
userId: string (UUID)

// Headers
Authorization: Bearer <jwt_token>
```

**Response**:
```json
{
  "success": true,
  "message": "Social profile retrieved",
  "data": {
    "user": {
      "id": "d1c2d2f6-6823-4ae3-99b8-f7e31c2bbbe5",
      "username": "chef_maria",
      "fullName": "Maria Chef",
      "avatar": "https://cdn.chefooz.app/avatars/maria.jpg",
      "bio": "Home chef specializing in Italian cuisine",
      "createdAt": "2025-01-15T10:00:00.000Z"
    },
    "privacy": {
      "isPrivate": false,
      "followersCount": 1250,
      "followingCount": 320
    },
    "relationship": {
      "isFollowing": true,
      "isFollowedBy": false,
      "isRequested": false,
      "isPrivate": false,
      "isBlockedByMe": false,
      "hasBlockedMe": false
    },
    "chefProfile": {
      "businessName": "Maria's Kitchen",
      "isActive": true,
      "cuisineTypes": ["Italian", "Mediterranean"]
    },
    "verification": null,
    "reputation": null
  }
}
```

**Error Codes**:
- `404`: SOCIAL_TARGET_USER_NOT_FOUND

---

### 5. Social Search

#### GET /api/v1/social/search

**Purpose**: Search users by name, username, or chef business name

**Authentication**: JWT required

**Request**:
```typescript
// Query parameters
query: string (search query)
cursor?: string (ISO date string)
limit?: number (default: 20, max: 100)

// Headers
Authorization: Bearer <jwt_token>
```

**Response**:
```json
{
  "success": true,
  "message": "Search results retrieved",
  "data": {
    "items": [
      {
        "id": "d1c2d2f6-6823-4ae3-99b8-f7e31c2bbbe5",
        "username": "chef_maria",
        "fullName": "Maria Chef",
        "avatar": "https://cdn.chefooz.app/avatars/maria.jpg",
        "isChef": true,
        "chefBusinessName": "Maria's Kitchen"
      }
    ],
    "nextCursor": "2026-02-10T14:20:00.000Z",
    "hasMore": true
  }
}
```

**Search Algorithm**:
1. ILIKE search on `fullName`, `username`, `chef.businessName`
2. Exclude blocked users (both directions)
3. Join `chef_profiles` table
4. Order by `createdAt DESC`
5. Paginate with cursor

---

### 6. Reel Sharing

#### POST /api/v1/reels/:reelId/share

**Purpose**: Record a share action

**Authentication**: JWT required

**Rate Limit**: 20 shares/minute (in-memory, production: use Redis)

**Request**:
```typescript
// Path parameter
reelId: string (MongoDB ObjectId)

// Headers
Authorization: Bearer <jwt_token>

// Body
{
  "type": "copy-link" | "external" | "direct" | "story",
  "targetUserId"?: "uuid" // Required for type='direct'
}
```

**Response**:
```json
{
  "success": true,
  "message": "Share recorded successfully",
  "data": {
    "success": true,
    "reelId": "691ca198b8be29951a3caf6b",
    "type": "copy-link",
    "updatedShareCount": 43,
    "deepLink": {
      "short": "https://chefooz.app/r/aB3xY7Zq",
      "universal": "https://chefooz.app/r/aB3xY7Zq",
      "app": "chefooz://reel/691ca198b8be29951a3caf6b"
    }
  }
}
```

**Error Codes**:
- `400`: SHARE_TARGET_USER_REQUIRED (for direct shares), SHARE_SELF_NOT_ALLOWED
- `403`: SHARE_USER_BLOCKED
- `404`: SHARE_REEL_NOT_FOUND, SHARE_TARGET_USER_NOT_FOUND
- `429`: SHARE_RATE_LIMIT_EXCEEDED
- `500`: SHARE_UPDATE_FAILED

---

#### GET /api/v1/reels/:reelId/share/stats

**Purpose**: Get share statistics for a reel

**Authentication**: JWT required

**Request**:
```typescript
// Path parameter
reelId: string (MongoDB ObjectId)

// Headers
Authorization: Bearer <jwt_token>
```

**Response**:
```json
{
  "success": true,
  "message": "Share stats retrieved",
  "data": {
    "reelId": "691ca198b8be29951a3caf6b",
    "shareCount": 43,
    "copyLinkCount": 15,
    "externalShareCount": 20,
    "directShareCount": 6,
    "storyShareCount": 2
  }
}
```

---

#### GET /api/v1/reels/:reelId/share/targetable-users

**Purpose**: Get list of users to directly share with (followers + following)

**Authentication**: JWT required

**Request**:
```typescript
// Path parameter
reelId: string (MongoDB ObjectId, not used in logic but part of route)

// Query parameters
cursor?: string (ISO date string)
limit?: number (default: 20, max: 100)

// Headers
Authorization: Bearer <jwt_token>
```

**Response**:
```json
{
  "success": true,
  "message": "Targetable users retrieved",
  "data": {
    "items": [
      {
        "id": "a3b4c5d6-7890-4321-8765-4321fedcba98",
        "username": "john_chef",
        "fullName": "John Doe",
        "avatar": "https://cdn.chefooz.app/avatars/john.jpg",
        "isFollowing": true,
        "isFollowedBy": true
      }
    ],
    "nextCursor": "2026-02-13T11:00:00.000Z",
    "hasMore": true
  }
}
```

---

## Service Layer

### SocialService

**Location**: `apps/chefooz-apis/src/modules/social/social.service.ts`

#### 1. followUser()

**Purpose**: Follow a user (auto-accept public, pending private)

**Signature**:
```typescript
async followUser(
  userId: string,
  targetIdentifier: string,
): Promise<{
  status: 'accepted' | 'pending';
  targetUserId: string;
  isPrivate: boolean;
}>
```

**Implementation**:
```typescript
async followUser(userId: string, targetIdentifier: string) {
  // 1. Resolve target identifier (username or UUID)
  const targetId = await this.resolveUserIdentifier(targetIdentifier);

  // 2. Prevent self-follow
  if (userId === targetId) {
    throw new HttpException(
      {
        success: false,
        message: 'Cannot follow yourself',
        errorCode: 'SOCIAL_SELF_FOLLOW_NOT_ALLOWED',
      },
      HttpStatus.BAD_REQUEST,
    );
  }

  // 3. Check for existing follow
  const existingFollow = await this.followRepository.findOne({
    where: { followerId: userId, targetId },
  });

  if (existingFollow) {
    throw new HttpException(
      {
        success: false,
        message: 'Already following or request pending',
        errorCode: 'SOCIAL_ALREADY_FOLLOWING',
      },
      HttpStatus.BAD_REQUEST,
    );
  }

  // 4. Check block status (both directions)
  const isBlocked = await this.isBlocked(userId, targetId);
  if (isBlocked) {
    throw new HttpException(
      {
        success: false,
        message: 'Cannot follow this user',
        errorCode: 'SOCIAL_USER_BLOCKED',
      },
      HttpStatus.FORBIDDEN,
    );
  }

  // 5. Get target's privacy settings
  const targetPrivacy = await this.getOrCreatePrivacy(targetId);

  // 6. Create follow record with appropriate status
  const status = targetPrivacy.isPrivate ? 'pending' : 'accepted';
  const follow = this.followRepository.create({
    followerId: userId,
    targetId,
    status,
  });
  await this.followRepository.save(follow);

  // 7. Increment counters if auto-accepted
  if (status === 'accepted') {
    await this.incrementFollowCounts(userId, targetId);
  }

  // 8. Invalidate cache
  await this.invalidateSocialCache(userId, targetId);

  // 9. Send notification
  await this.notificationOrchestrator.createWithContext({
    type: 'FOLLOWED',
    userId: targetId,
    actorId: userId,
    contextData: {
      isPending: status === 'pending',
    },
  });

  return {
    status,
    targetUserId: targetId,
    isPrivate: targetPrivacy.isPrivate,
  };
}
```

---

#### 2. unfollowUser()

**Purpose**: Unfollow a user

**Signature**:
```typescript
async unfollowUser(
  userId: string,
  targetIdentifier: string,
): Promise<{ targetUserId: string }>
```

**Implementation**:
```typescript
async unfollowUser(userId: string, targetIdentifier: string) {
  // 1. Resolve target identifier
  const targetId = await this.resolveUserIdentifier(targetIdentifier);

  // 2. Find existing follow
  const follow = await this.followRepository.findOne({
    where: { followerId: userId, targetId },
  });

  if (!follow) {
    throw new HttpException(
      {
        success: false,
        message: 'Not following this user',
        errorCode: 'SOCIAL_NOT_FOLLOWING',
      },
      HttpStatus.BAD_REQUEST,
    );
  }

  // 3. Delete follow record
  await this.followRepository.remove(follow);

  // 4. Decrement counters (only if was accepted)
  if (follow.status === 'accepted') {
    await this.decrementFollowCounts(userId, targetId);
  }

  // 5. Invalidate cache
  await this.invalidateSocialCache(userId, targetId);

  return { targetUserId: targetId };
}
```

---

#### 3. acceptFollowRequest()

**Purpose**: Accept a pending follow request

**Signature**:
```typescript
async acceptFollowRequest(
  userId: string,
  requestId: string,
): Promise<{ requestId: string; followerId: string }>
```

**Implementation**:
```typescript
async acceptFollowRequest(userId: string, requestId: string) {
  // 1. Find pending request
  const follow = await this.followRepository.findOne({
    where: {
      id: requestId,
      targetId: userId,
      status: 'pending',
    },
  });

  if (!follow) {
    throw new HttpException(
      {
        success: false,
        message: 'Follow request not found',
        errorCode: 'SOCIAL_REQUEST_NOT_FOUND',
      },
      HttpStatus.NOT_FOUND,
    );
  }

  // 2. Update status to accepted
  follow.status = 'accepted';
  await this.followRepository.save(follow);

  // 3. Increment counters
  await this.incrementFollowCounts(follow.followerId, userId);

  // 4. Invalidate cache
  await this.invalidateSocialCache(follow.followerId, userId);

  // 5. Send notification
  await this.notificationOrchestrator.createWithContext({
    type: 'FOLLOW_REQUEST_ACCEPTED',
    userId: follow.followerId,
    actorId: userId,
  });

  return {
    requestId,
    followerId: follow.followerId,
  };
}
```

---

#### 4. rejectFollowRequest()

**Purpose**: Reject a pending follow request

**Signature**:
```typescript
async rejectFollowRequest(
  userId: string,
  requestId: string,
): Promise<{ requestId: string }>
```

**Implementation**:
```typescript
async rejectFollowRequest(userId: string, requestId: string) {
  // 1. Find pending request
  const follow = await this.followRepository.findOne({
    where: {
      id: requestId,
      targetId: userId,
      status: 'pending',
    },
  });

  if (!follow) {
    throw new HttpException(
      {
        success: false,
        message: 'Follow request not found',
        errorCode: 'SOCIAL_REQUEST_NOT_FOUND',
      },
      HttpStatus.NOT_FOUND,
    );
  }

  // 2. Delete pending request (no notification, soft rejection)
  await this.followRepository.remove(follow);

  // 3. Invalidate cache
  await this.invalidateSocialCache(follow.followerId, userId);

  return { requestId };
}
```

---

#### 5. blockUser()

**Purpose**: Block a user (removes all follows, prevents interactions)

**Signature**:
```typescript
async blockUser(
  userId: string,
  targetId: string,
): Promise<{ blockedUserId: string }>
```

**Implementation**:
```typescript
async blockUser(userId: string, targetId: string) {
  // 1. Prevent self-block
  if (userId === targetId) {
    throw new HttpException(
      {
        success: false,
        message: 'Cannot block yourself',
        errorCode: 'SOCIAL_SELF_BLOCK_NOT_ALLOWED',
      },
      HttpStatus.BAD_REQUEST,
    );
  }

  // 2. Check if already blocked
  const existingBlock = await this.blockRepository.findOne({
    where: { userId, blockedUserId: targetId },
  });

  if (existingBlock) {
    throw new HttpException(
      {
        success: false,
        message: 'User already blocked',
        errorCode: 'SOCIAL_ALREADY_BLOCKED',
      },
      HttpStatus.BAD_REQUEST,
    );
  }

  // 3. Find all follow relationships (both directions)
  const follows = await this.followRepository.find({
    where: [
      { followerId: userId, targetId },
      { followerId: targetId, targetId: userId },
    ],
  });

  // 4. Decrement counters for accepted follows
  for (const follow of follows) {
    if (follow.status === 'accepted') {
      await this.decrementFollowCounts(follow.followerId, follow.targetId);
    }
  }

  // 5. Remove all follows
  if (follows.length > 0) {
    await this.followRepository.remove(follows);
  }

  // 6. Create block record
  const block = this.blockRepository.create({
    userId,
    blockedUserId: targetId,
  });
  await this.blockRepository.save(block);

  // 7. Invalidate cache
  await this.invalidateSocialCache(userId, targetId);

  // 8. Log notification (no negative notification sent)
  this.logger.log(
    `❌ Block notification not sent (soft block for privacy)`,
  );

  return { blockedUserId: targetId };
}
```

---

#### 6. getRelationship()

**Purpose**: Get relationship state with target user (6 boolean flags)

**Signature**:
```typescript
async getRelationship(
  userId: string,
  targetId: string,
): Promise<{
  isFollowing: boolean;
  isFollowedBy: boolean;
  isRequested: boolean;
  isPrivate: boolean;
  isBlockedByMe: boolean;
  hasBlockedMe: boolean;
}>
```

**Implementation**:
```typescript
async getRelationship(userId: string, targetId: string) {
  // 1. Get follow status (both directions)
  const [following, followedBy] = await Promise.all([
    this.isFollowing(userId, targetId),
    this.isFollowing(targetId, userId),
  ]);

  // 2. Check pending request
  const isRequested = await this.isPendingRequest(userId, targetId);

  // 3. Get privacy settings
  const targetPrivacy = await this.getOrCreatePrivacy(targetId);

  // 4. Check block status (both directions)
  const [isBlockedByMe, hasBlockedMe] = await Promise.all([
    this.hasBlocked(userId, targetId),
    this.hasBlocked(targetId, userId),
  ]);

  return {
    isFollowing: following,
    isFollowedBy: followedBy,
    isRequested,
    isPrivate: targetPrivacy.isPrivate,
    isBlockedByMe,
    hasBlockedMe,
  };
}
```

---

#### 7. searchUsers()

**Purpose**: Search users by name, username, or chef business name

**Signature**:
```typescript
async searchUsers(
  userId: string,
  dto: SearchUsersDto,
): Promise<{
  items: Array<{
    id: string;
    username: string;
    fullName: string | null;
    avatar: string | null;
    isChef: boolean;
    chefBusinessName: string | null;
  }>;
  nextCursor: string | null;
  hasMore: boolean;
}>
```

**Implementation**:
```typescript
async searchUsers(userId: string, dto: SearchUsersDto) {
  const limit = dto.limit || 20;

  // 1. Build query
  const queryBuilder = this.userRepository
    .createQueryBuilder('user')
    .leftJoinAndSelect('user.chefProfile', 'chef')
    .where(
      new Brackets((qb) => {
        qb.where('user.fullName ILIKE :query', {
          query: `%${dto.query}%`,
        })
          .orWhere('user.username ILIKE :query', {
            query: `%${dto.query}%`,
          })
          .orWhere('chef.businessName ILIKE :query', {
            query: `%${dto.query}%`,
          });
      }),
    )
    .orderBy('user.createdAt', 'DESC')
    .take(limit + 1);

  // 2. Add cursor pagination
  if (dto.cursor) {
    queryBuilder.andWhere('user.createdAt < :cursor', {
      cursor: dto.cursor,
    });
  }

  // 3. Exclude blocked users (both directions)
  const blocks = await this.blockRepository.find({
    where: [
      { userId },
      { blockedUserId: userId },
    ],
  });

  const blockedIds = blocks.map((b) =>
    b.userId === userId ? b.blockedUserId : b.userId,
  );

  if (blockedIds.length > 0) {
    queryBuilder.andWhere('user.id NOT IN (:...blockedIds)', {
      blockedIds,
    });
  }

  // 4. Execute query
  const users = await queryBuilder.getMany();

  // 5. Pagination
  const hasMore = users.length > limit;
  const items = hasMore ? users.slice(0, limit) : users;
  const nextCursor = hasMore
    ? items[items.length - 1].createdAt.toISOString()
    : null;

  // 6. Map to response format
  const results = items.map((user) => ({
    id: user.id,
    username: user.username,
    fullName: user.fullName || null,
    avatar: user.avatar || null,
    isChef: !!user.chefProfile,
    chefBusinessName: user.chefProfile?.businessName || null,
  }));

  return { items: results, nextCursor, hasMore };
}
```

---

### ReelsShareService

**Location**: `apps/chefooz-apis/src/modules/social/reels-share.service.ts`

#### 1. recordShare()

**Purpose**: Record a share action and increment counters

**Signature**:
```typescript
async recordShare(
  userId: string,
  reelId: string,
  type: ShareType,
  targetUserId?: string,
): Promise<{
  reel: Reel;
  updatedShareCount: number;
  deepLink?: { short: string; universal: string; app: string };
}>
```

**Implementation**:
```typescript
async recordShare(
  userId: string,
  reelId: string,
  type: ShareType,
  targetUserId?: string,
) {
  // 1. Rate limiting (in-memory, production: use Redis)
  const now = Date.now();
  const oneMinuteAgo = now - 60 * 1000;
  const userShares = this.shareRateLimits.get(userId) || [];
  const recentShares = userShares.filter((ts) => ts > oneMinuteAgo);

  if (recentShares.length >= 20) {
    throw new HttpException(
      {
        success: false,
        message: 'Too many shares. Please wait a moment.',
        errorCode: 'SHARE_RATE_LIMIT_EXCEEDED',
      },
      HttpStatus.TOO_MANY_REQUESTS,
    );
  }

  recentShares.push(now);
  this.shareRateLimits.set(userId, recentShares);

  // 2. Verify reel exists
  const reel = await this.reelModel.findById(reelId);
  if (!reel) {
    throw new HttpException(
      {
        success: false,
        message: 'Reel not found',
        errorCode: 'SHARE_REEL_NOT_FOUND',
      },
      HttpStatus.NOT_FOUND,
    );
  }

  // 3. Validate target user (for direct shares)
  if (type === ShareType.DIRECT) {
    if (!targetUserId) {
      throw new HttpException(
        {
          success: false,
          message: 'Target user ID required for direct share',
          errorCode: 'SHARE_TARGET_USER_REQUIRED',
        },
        HttpStatus.BAD_REQUEST,
      );
    }

    // Prevent self-sharing
    if (targetUserId === userId) {
      throw new HttpException(
        {
          success: false,
          message: 'Cannot share to yourself',
          errorCode: 'SHARE_SELF_NOT_ALLOWED',
        },
        HttpStatus.BAD_REQUEST,
      );
    }

    // Check blocking
    const isBlocked = await this.blockRepository.findOne({
      where: [
        { userId, blockedUserId: targetUserId },
        { userId: targetUserId, blockedUserId: userId },
      ],
    });

    if (isBlocked) {
      throw new HttpException(
        {
          success: false,
          message: 'Cannot share to this user',
          errorCode: 'SHARE_USER_BLOCKED',
        },
        HttpStatus.FORBIDDEN,
      );
    }
  }

  // 4. Increment counters atomically
  const updateFields: Record<string, number> = {
    'stats.shareCount': 1,
  };

  switch (type) {
    case ShareType.COPY_LINK:
      updateFields['stats.copyLinkCount'] = 1;
      break;
    case ShareType.EXTERNAL:
      updateFields['stats.externalShareCount'] = 1;
      break;
    case ShareType.DIRECT:
      updateFields['stats.directShareCount'] = 1;
      break;
    case ShareType.STORY:
      updateFields['stats.storyShareCount'] = 1;
      break;
  }

  const updatedReel = await this.reelModel.findByIdAndUpdate(
    reelId,
    { $inc: updateFields },
    { new: true },
  );

  // 5. Generate deep link
  const deepLink = await this.deeplinkService.generateReelLink(reelId);

  // 6. Track analytics
  this.trackShareAnalytics(userId, reelId, type, targetUserId);

  // 7. Send notification (for direct shares)
  if (type === ShareType.DIRECT && targetUserId) {
    this.logger.log(
      `🔔 TODO: Send REEL_SHARED_DIRECT notification to ${targetUserId}`,
    );
  }

  return {
    reel: updatedReel!,
    updatedShareCount: updatedReel!.stats.shareCount,
    deepLink,
  };
}
```

---

## Data Transfer Objects (DTOs)

### 1. Social DTOs

**Location**: `apps/chefooz-apis/src/modules/social/dto/social.dto.ts`

```typescript
export class UpdatePrivacyDto {
  @ApiProperty({ description: 'Set account to private' })
  @IsBoolean()
  isPrivate!: boolean;
}

export class GetFollowListDto {
  @ApiProperty({ description: 'User ID', required: false })
  @IsOptional()
  @IsUUID()
  userId?: string;

  @ApiProperty({ description: 'Pagination cursor', required: false })
  @IsOptional()
  @IsString()
  cursor?: string;

  @ApiProperty({ description: 'Items per page', required: false, default: 20 })
  @IsOptional()
  @IsInt()
  @Min(1)
  @Type(() => Number)
  limit?: number = 20;
}

export class SearchUsersDto {
  @ApiProperty({ description: 'Search query' })
  @IsString()
  query!: string;

  @ApiProperty({ description: 'Pagination cursor', required: false })
  @IsOptional()
  @IsString()
  cursor?: string;

  @ApiProperty({ description: 'Items per page', required: false, default: 20 })
  @IsOptional()
  @IsInt()
  @Min(1)
  @Type(() => Number)
  limit?: number = 20;
}
```

---

### 2. Share DTOs

**Location**: `apps/chefooz-apis/src/modules/social/dto/share-reel.dto.ts`

```typescript
export enum ShareType {
  COPY_LINK = 'copy-link',
  EXTERNAL = 'external',
  DIRECT = 'direct',
  STORY = 'story',
}

export class ShareReelDto {
  @ApiProperty({
    description: 'Type of share action',
    enum: ShareType,
    example: ShareType.COPY_LINK,
    required: true,
  })
  @IsEnum(ShareType, {
    message: 'type must be one of: copy-link, external, direct, story',
  })
  type!: ShareType;

  @ApiPropertyOptional({
    description: 'Target user ID (required only for direct share)',
    example: 'd1c2d2f6-6823-4ae3-99b8-f7e31c2bbbe5',
  })
  @IsString({ message: 'targetUserId must be a valid UUID string' })
  @ValidateIf((o) => o.type === ShareType.DIRECT, {
    message: 'targetUserId is required for direct shares',
  })
  targetUserId?: string;
}

export class GetTargetableUsersDto {
  @ApiPropertyOptional({
    description: 'Cursor for pagination',
    example: '691ca198b8be29951a3caf6b',
  })
  @IsString()
  @IsOptional()
  cursor?: string;

  @ApiPropertyOptional({
    description: 'Number of users to return',
    example: 20,
    default: 20,
  })
  @IsOptional()
  limit?: number = 20;
}
```

---

## Entity Relationships

### Relationship Diagram

```
┌─────────────┐
│    User     │
└──────┬──────┘
       │
       ├──────────────────────────────────┐
       │                                  │
       │ followerId                       │ targetId
       │                                  │
       ▼                                  ▼
┌─────────────────┐              ┌──────────────────┐
│   UserFollow    │              │   UserPrivacy    │
│                 │              │                  │
│ - followerId    │              │ - userId         │
│ - targetId      │              │ - isPrivate      │
│ - status        │              │ - followersCount │
│ - createdAt     │              │ - followingCount │
└─────────────────┘              └──────────────────┘

       │
       │ userId / blockedUserId
       │
       ▼
┌─────────────────┐              ┌──────────────────┐
│   UserBlock     │              │   ChefProfile    │
│                 │              │                  │
│ - userId        │              │ - userId         │
│ - blockedUserId │              │ - businessName   │
│ - createdAt     │              │ - isActive       │
└─────────────────┘              └──────────────────┘

       │
       │ reelId (MongoDB)
       │
       ▼
┌─────────────────┐
│      Reel       │
│   (MongoDB)     │
│                 │
│ - stats:        │
│   - shareCount  │
│   - copyLink... │
│   - external... │
│   - direct...   │
│   - story...    │
└─────────────────┘
```

---

## Caching Strategy

### Cache Keys

**Social Profile**:
```
social:profile:{userId}
```
- **TTL**: 30 minutes
- **Data**: Enriched profile projection (user + privacy + chef + relationship)
- **Invalidation**: Follow/unfollow/block/privacy changes

**Privacy Settings**:
```
social:privacy:{userId}
```
- **TTL**: 60 minutes
- **Data**: isPrivate, followersCount, followingCount
- **Invalidation**: Privacy toggle, follow/unfollow/block

### Cache Invalidation

**Implementation**:
```typescript
private async invalidateSocialCache(
  userId: string,
  targetId: string,
): Promise<void> {
  const keys = [
    `social:profile:${userId}`,
    `social:profile:${targetId}`,
    `social:privacy:${userId}`,
    `social:privacy:${targetId}`,
  ];

  await Promise.all(
    keys.map((key) => this.cacheManager.del(key)),
  );

  this.logger.debug(`🗑️ Cache invalidated: ${keys.join(', ')}`);
}
```

**Triggers**:
- Follow: Invalidate both users (follower + target)
- Unfollow: Invalidate both users
- Block: Invalidate both users
- Privacy toggle: Invalidate own user
- Accept/reject request: Invalidate both users

---

## Notification Integration

### Notification Events

**1. FOLLOWED**:
```typescript
await this.notificationOrchestrator.createWithContext({
  type: 'FOLLOWED',
  userId: targetId,
  actorId: userId,
  contextData: {
    isPending: status === 'pending',
  },
});
```
- **Sent when**: User follows another user
- **Recipient**: Followed user
- **Context**: isPending flag (true for private accounts)
- **Text**: "X followed you" or "X requested to follow you"

**2. FOLLOW_REQUEST_ACCEPTED**:
```typescript
await this.notificationOrchestrator.createWithContext({
  type: 'FOLLOW_REQUEST_ACCEPTED',
  userId: followerId,
  actorId: userId,
});
```
- **Sent when**: User accepts follow request
- **Recipient**: Original requester
- **Text**: "X accepted your follow request"

**3. REEL_SHARED_DIRECT (TODO)**:
```typescript
// TODO: Implement when notification service is integrated
this.logger.log(
  `🔔 TODO: Send REEL_SHARED_DIRECT notification to ${targetUserId}`,
);
```
- **Sent when**: User shares reel directly to another user
- **Recipient**: Share target user
- **Text**: "X shared a reel with you"

---

## Error Handling

### Error Response Format

```json
{
  "success": false,
  "message": "Human-readable error message",
  "errorCode": "ERROR_CODE_IDENTIFIER"
}
```

### Error Codes

| Code | Message | HTTP Status | Cause |
|------|---------|-------------|-------|
| SOCIAL_TARGET_USER_NOT_FOUND | Target user not found | 404 | User ID/username doesn't exist |
| SOCIAL_ALREADY_FOLLOWING | Already following or request pending | 400 | Duplicate follow attempt |
| SOCIAL_NOT_FOLLOWING | Not following this user | 400 | Unfollow without follow |
| SOCIAL_SELF_FOLLOW_NOT_ALLOWED | Cannot follow yourself | 400 | Follow targetId == userId |
| SOCIAL_USER_BLOCKED | Cannot follow this user | 403 | Block exists (either direction) |
| SOCIAL_REQUEST_NOT_FOUND | Follow request not found | 404 | Invalid requestId or already processed |
| SOCIAL_SELF_BLOCK_NOT_ALLOWED | Cannot block yourself | 400 | Block targetId == userId |
| FOLLOW_RATE_LIMIT_EXCEEDED | Too many follow requests. Please wait. | 429 | >30 follows/min |
| UNFOLLOW_RATE_LIMIT_EXCEEDED | Too many unfollow requests. Please wait. | 429 | >30 unfollows/min |
| BLOCK_RATE_LIMIT_EXCEEDED | Too many block requests. Please wait. | 429 | >20 blocks/min |
| SHARE_RATE_LIMIT_EXCEEDED | Too many shares. Please wait a moment. | 429 | >20 shares/min |
| SHARE_REEL_NOT_FOUND | Reel not found | 404 | Invalid reelId |
| SHARE_TARGET_USER_REQUIRED | Target user ID required for direct share | 400 | Missing targetUserId for direct share |
| SHARE_TARGET_USER_NOT_FOUND | Target user not found | 404 | Invalid targetUserId |
| SHARE_SELF_NOT_ALLOWED | Cannot share to yourself | 400 | Share targetUserId == userId |
| SHARE_USER_BLOCKED | Cannot share to this user | 403 | Block exists (either direction) |
| SHARE_UPDATE_FAILED | Failed to update reel | 500 | MongoDB update failed |

---

## Performance Optimization

### 1. Database Indexing

**Indexes Used**:
```sql
-- user_follow
CREATE UNIQUE INDEX idx_user_follow_unique ON user_follow (follower_id, target_id);
CREATE INDEX idx_user_follow_target ON user_follow (target_id); -- Follower lists
CREATE INDEX idx_user_follow_follower ON user_follow (follower_id); -- Following lists
CREATE INDEX idx_user_follow_status ON user_follow (status); -- Pending requests

-- user_privacy
CREATE UNIQUE INDEX idx_user_privacy_user ON user_privacy (user_id);

-- user_block
CREATE UNIQUE INDEX idx_user_block_unique ON user_block (user_id, blocked_user_id);
CREATE INDEX idx_user_block_user ON user_block (user_id); -- Block lists
CREATE INDEX idx_user_block_blocked ON user_block (blocked_user_id); -- Blocked by check
```

**Query Performance**:
- Follower list: Uses `idx_user_follow_target` (target_id)
- Following list: Uses `idx_user_follow_follower` (follower_id)
- Pending requests: Uses `idx_user_follow_status` + `idx_user_follow_target`
- Block check: Uses `idx_user_block_user` + `idx_user_block_blocked`

---

### 2. Denormalized Counters

**Rationale**:
- Avoid COUNT queries on large follow tables (10M+ rows)
- Single SELECT vs COUNT aggregation (100x faster)

**Implementation**:
```typescript
async incrementFollowCounts(followerId: string, targetId: string) {
  await this.userPrivacyRepository.increment(
    { userId: targetId },
    'followersCount',
    1,
  );
  await this.userPrivacyRepository.increment(
    { userId: followerId },
    'followingCount',
    1,
  );
}

async decrementFollowCounts(followerId: string, targetId: string) {
  await this.userPrivacyRepository.decrement(
    { userId: targetId },
    'followersCount',
    1,
  );
  await this.userPrivacyRepository.decrement(
    { userId: followerId },
    'followingCount',
    1,
  );
}
```

**Trade-offs**:
- ✅ Faster reads (single SELECT)
- ✅ No DB load from COUNT queries
- ⚠️ Slightly more complex writes
- ⚠️ Risk of drift (mitigated by atomic updates)

---

### 3. Cursor-Based Pagination

**Implementation**:
```typescript
const queryBuilder = this.followRepository
  .createQueryBuilder('follow')
  .where('follow.targetId = :userId', { userId })
  .andWhere('follow.status = :status', { status: 'accepted' })
  .orderBy('follow.createdAt', 'DESC')
  .take(limit + 1);

if (cursor) {
  queryBuilder.andWhere('follow.createdAt < :cursor', { cursor });
}

const follows = await queryBuilder.getMany();
const hasMore = follows.length > limit;
const items = hasMore ? follows.slice(0, limit) : follows;
const nextCursor = hasMore
  ? items[items.length - 1].createdAt.toISOString()
  : null;
```

**Benefits**:
- Consistent performance (no OFFSET penalty)
- No "phantom reads" (data changes between pages)
- Supports infinite scroll UX

---

### 4. Atomic Operations

**MongoDB (Reel Share Counters)**:
```typescript
await this.reelModel.findByIdAndUpdate(
  reelId,
  {
    $inc: {
      'stats.shareCount': 1,
      'stats.copyLinkCount': 1,
    },
  },
  { new: true },
);
```

**PostgreSQL (Follow Counters)**:
```typescript
await this.userPrivacyRepository.increment(
  { userId: targetId },
  'followersCount',
  1,
);
```

**Benefits**:
- Prevents race conditions (concurrent updates)
- Database-level atomicity (no locks)
- Ensures counter accuracy

---

## Code Examples

### Example 1: Follow a Public Account

```typescript
// Client-side
const response = await axios.post(
  `/api/v1/social/follow/chef_maria`,
  {},
  {
    headers: {
      Authorization: `Bearer ${jwtToken}`,
    },
  },
);

// Response
{
  "success": true,
  "message": "Successfully followed user",
  "data": {
    "status": "accepted",
    "targetUserId": "d1c2d2f6-6823-4ae3-99b8-f7e31c2bbbe5",
    "isPrivate": false
  }
}

// UI Update
setButtonState('Following');
setFollowersCount((prev) => prev + 1);
```

---

### Example 2: Follow a Private Account and Accept Request

```typescript
// Step 1: User A follows private Chef C
const followResponse = await axios.post(
  `/api/v1/social/follow/chef_carlos`,
  {},
  {
    headers: {
      Authorization: `Bearer ${userAToken}`,
    },
  },
);

// Response
{
  "success": true,
  "message": "Follow request sent",
  "data": {
    "status": "pending",
    "targetUserId": "a3b4c5d6-7890-4321-8765-4321fedcba98",
    "isPrivate": true
  }
}

// UI Update (User A's view)
setButtonState('Requested');

// ---

// Step 2: Chef C receives notification and opens requests
const requestsResponse = await axios.get(
  `/api/v1/social/requests/pending`,
  {
    headers: {
      Authorization: `Bearer ${chefCToken}`,
    },
  },
);

// Response
{
  "success": true,
  "message": "Pending requests retrieved",
  "data": {
    "items": [
      {
        "id": "f7e31c2b-bbe5-4321-8765-d1c2d2f66823",
        "followerId": "userA_uuid",
        "follower": {
          "username": "user_a",
          "fullName": "User A",
          "avatar": "..."
        }
      }
    ]
  }
}

// ---

// Step 3: Chef C accepts request
const acceptResponse = await axios.post(
  `/api/v1/social/requests/f7e31c2b-bbe5-4321-8765-d1c2d2f66823/accept`,
  {},
  {
    headers: {
      Authorization: `Bearer ${chefCToken}`,
    },
  },
);

// Response
{
  "success": true,
  "message": "Follow request accepted"
}

// User A receives FOLLOW_REQUEST_ACCEPTED notification
// UI Update (User A's view)
setButtonState('Following');
```

---

### Example 3: Block a User

```typescript
// User A blocks disruptive User D
const response = await axios.post(
  `/api/v1/social/block/${userDId}`,
  {},
  {
    headers: {
      Authorization: `Bearer ${userAToken}`,
    },
  },
);

// Response
{
  "success": true,
  "message": "User blocked successfully",
  "data": {
    "blockedUserId": "userD_uuid"
  }
}

// Side Effects (Backend):
// - Remove all follows (A ↔ D)
// - Decrement counters
// - Create block record
// - Invalidate cache
// - No notification sent

// UI Update
setButtonState('Blocked');
setFollowersCount((prev) => prev - 1); // If D was following A
```

---

### Example 4: Share Reel Directly to User

```typescript
// Step 1: Get targetable users
const usersResponse = await axios.get(
  `/api/v1/reels/${reelId}/share/targetable-users`,
  {
    headers: {
      Authorization: `Bearer ${jwtToken}`,
    },
  },
);

// Response
{
  "success": true,
  "data": {
    "items": [
      {
        "id": "userE_uuid",
        "username": "user_e",
        "fullName": "User E",
        "avatar": "...",
        "isFollowing": true,
        "isFollowedBy": true
      }
    ]
  }
}

// ---

// Step 2: User selects User E and shares
const shareResponse = await axios.post(
  `/api/v1/reels/${reelId}/share`,
  {
    type: 'direct',
    targetUserId: 'userE_uuid',
  },
  {
    headers: {
      Authorization: `Bearer ${jwtToken}`,
    },
  },
);

// Response
{
  "success": true,
  "message": "Share recorded successfully",
  "data": {
    "reelId": "691ca198b8be29951a3caf6b",
    "type": "direct",
    "updatedShareCount": 43,
    "deepLink": {
      "short": "https://chefooz.app/r/aB3xY7Zq",
      "universal": "https://chefooz.app/r/aB3xY7Zq",
      "app": "chefooz://reel/691ca198b8be29951a3caf6b"
    }
  }
}

// User E receives REEL_SHARED_DIRECT notification (TODO)
// UI Update
showToast('Shared to User E');
setShareCount(43);
```

---

### Example 5: Search Users

```typescript
// User searches for "Maria"
const response = await axios.get(
  `/api/v1/social/search`,
  {
    params: {
      query: 'Maria',
      limit: 20,
    },
    headers: {
      Authorization: `Bearer ${jwtToken}`,
    },
  },
);

// Response
{
  "success": true,
  "message": "Search results retrieved",
  "data": {
    "items": [
      {
        "id": "chef_maria_uuid",
        "username": "chef_maria",
        "fullName": "Maria Chef",
        "avatar": "https://cdn.chefooz.app/avatars/maria.jpg",
        "isChef": true,
        "chefBusinessName": "Maria's Kitchen"
      },
      {
        "id": "maria_user_uuid",
        "username": "maria123",
        "fullName": "Maria Smith",
        "avatar": null,
        "isChef": false,
        "chefBusinessName": null
      }
    ],
    "nextCursor": "2026-02-10T14:20:00.000Z",
    "hasMore": true
  }
}

// UI Update
renderSearchResults(response.data.items);

// User scrolls down → Fetch next page
const nextPageResponse = await axios.get(
  `/api/v1/social/search`,
  {
    params: {
      query: 'Maria',
      cursor: response.data.nextCursor,
      limit: 20,
    },
    headers: {
      Authorization: `Bearer ${jwtToken}`,
    },
  },
);
```

---

**[SLICE_COMPLETE ✅]**
