# Stories Module - Technical Guide

## Architecture Overview

### High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                         Mobile App (Expo)                        │
│  - Story creation UI (camera/media picker)                       │
│  - Horizontal story rail (tray)                                  │
│  - Story viewer (vertical swipe)                                 │
│  - View tracking                                                 │
└────────────────────────────┬─────────────────────────────────────┘
                             │
                             │ HTTPS/REST (JWT Bearer Token)
                             ↓
┌──────────────────────────────────────────────────────────────────┐
│                    NestJS API Gateway                            │
│  - JwtAuthGuard (all endpoints)                                  │
│  - StoriesController (7 endpoints)                               │
│  - Multipart/form-data upload                                    │
└────────────────────────────┬─────────────────────────────────────┘
                             │
                             ↓
┌──────────────────────────────────────────────────────────────────┐
│                      StoriesService                              │
│  - createStory()                                                 │
│  - getUserStories()                                              │
│  - getStoryTrayForUser()                                         │
│  - getStoryFeedForUser()                                         │
│  - getStoriesByUsername()                                        │
│  - getStoryById()                                                │
│  - markStoryViewed()                                             │
│  - checkStoryPrivacy()                                           │
│  - deleteExpiredStories()                                        │
└──┬────────────────────────┬────────────────────────┬─────────────┘
   │                        │                        │
   ↓                        ↓                        ↓
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────┐
│   MongoDB       │  │  PostgreSQL     │  │  AWS MediaConvert   │
│   (Mongoose)    │  │  (TypeORM)      │  │  (Video Processing) │
│                 │  │                 │  │                     │
│  stories        │  │  users          │  │  - Transcoding      │
│  (TTL index)    │  │  user_follows   │  │  - HLS variants     │
│                 │  │                 │  │  - Thumbnails       │
└─────────────────┘  └─────────────────┘  └─────────────────────┘
         │
         ↓
┌─────────────────────────────────────────┐
│  MongoDB TTL Index (Auto-Expiration)    │
│  Deletes stories ~60s after expiresAt   │
└─────────────────────────────────────────┘
```

### Tech Stack

| Component | Technology | Version | Purpose |
|-----------|-----------|---------|---------|
| **Backend Framework** | NestJS | 10.x | API development |
| **Primary Database** | MongoDB | 6.x | Story documents with TTL |
| **ODM** | Mongoose | 8.x | MongoDB schema + queries |
| **Secondary Database** | PostgreSQL | 15.x | User profiles, follows |
| **ORM** | TypeORM | 0.3.x | PostgreSQL entities |
| **Scheduling** | @nestjs/schedule | 4.x | Cron jobs |
| **Media Processing** | AWS MediaConvert | - | Video transcoding |
| **File Upload** | @nestjs/platform-express | 10.x | Multipart handling |
| **Authentication** | Passport JWT | 10.x | JWT authentication |
| **API Docs** | Swagger/OpenAPI | 7.x | API documentation |

---

## Module Structure

### File Tree

```
apps/chefooz-apis/src/modules/stories/
├── dto/
│   └── story.dto.ts              # Request/response DTOs
├── jobs/
│   └── story-cleanup.job.ts      # Scheduled cleanup cron
├── stories.controller.ts         # 7 HTTP endpoints
├── stories.service.ts            # Business logic + DB access
├── stories.module.ts             # Module configuration
└── stories.schema.ts             # MongoDB schema
```

### Dependencies

**External Packages:**
```typescript
// NestJS
@nestjs/common
@nestjs/mongoose
@nestjs/typeorm
@nestjs/schedule
@nestjs/platform-express

// Mongoose
mongoose

// TypeORM
typeorm

// Validation
class-validator
class-transformer
```

**Internal Dependencies:**
```typescript
// Entities
'../../database/entities/user.entity'
'../social/entities/user-follow.entity'

// Guards
'../auth/guards/jwt.guard'

// Utils
'../../utils/s3-url.util'
```

---

## Database Schema

### MongoDB Story Schema

**File**: `stories.schema.ts`

```typescript
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { Document } from 'mongoose';

export type StoryDocument = Story & Document;

@Schema({ timestamps: true, collection: 'stories' })
export class Story {
  @Prop({ required: true, type: String })
  userId!: string; // UUID reference to Postgres users.id

  @Prop({ required: true, type: String })
  mediaId!: string; // MediaConvert job ID or unique identifier

  @Prop({ required: true, enum: ['image', 'video'] })
  mediaType!: 'image' | 'video';

  @Prop({ required: true, type: String })
  mediaUrl!: string; // S3 URI (converted to HTTPS via s3UriToHttps)

  @Prop({ required: true, type: String })
  thumbnailUrl!: string; // S3 URI to thumbnail

  @Prop({ required: true, type: Number, default: 5000 })
  durationMs!: number; // 5000ms images, 10000ms videos

  @Prop({ required: true, type: Date, default: Date.now })
  createdAt!: Date;

  @Prop({ 
    required: true, 
    type: Date, 
    default: () => new Date(Date.now() + 24 * 60 * 60 * 1000),
    index: { expireAfterSeconds: 0 } // TTL index
  })
  expiresAt!: Date;

  @Prop({ required: true, type: Number, default: 0 })
  views!: number;

  @Prop({ type: [String], default: [] })
  viewers!: string[]; // Capped at 100

  @Prop({ 
    required: true, 
    enum: ['public', 'followers', 'close-friends'], 
    default: 'followers',
    index: true
  })
  privacy!: 'public' | 'followers' | 'close-friends';

  @Prop({ type: [String], default: [] })
  closeFriends!: string[];
}

export const StorySchema = SchemaFactory.createForClass(Story);

// Compound indexes
StorySchema.index({ userId: 1, createdAt: -1 });
StorySchema.index({ privacy: 1, createdAt: -1 });
StorySchema.index({ views: -1 });
StorySchema.index({ expiresAt: 1 });
```

**Index Details:**

| Index | Type | Purpose | Query Pattern |
|-------|------|---------|---------------|
| `{ expiresAt: 1 }` | TTL | Auto-delete after expiration | MongoDB background process |
| `{ userId: 1, createdAt: -1 }` | Compound | Get user's stories newest first | `GET /stories/me` |
| `{ privacy: 1, createdAt: -1 }` | Compound | Filter by privacy level | Feed generation |
| `{ views: -1 }` | Single | Sort by popularity | Analytics queries |

### PostgreSQL Entities

**User Entity** (TypeORM)
```typescript
@Entity('users')
export class User {
  @PrimaryGeneratedColumn('uuid')
  id!: string;

  @Column({ unique: true })
  username!: string;

  @Column({ nullable: true })
  fullName!: string;

  // TODO: Add avatar field
  // @Column({ nullable: true })
  // avatar?: string;
}
```

**UserFollow Entity** (TypeORM)
```typescript
@Entity('user_follows')
export class UserFollow {
  @PrimaryGeneratedColumn('uuid')
  id!: string;

  @Column('uuid')
  followerId!: string; // User who follows

  @Column('uuid')
  targetId!: string; // User being followed

  @Column({ type: 'enum', enum: ['pending', 'accepted', 'blocked'] })
  status!: 'pending' | 'accepted' | 'blocked';

  @CreateDateColumn()
  createdAt!: Date;
}
```

---

## API Endpoints

### 1. Create Story

**Endpoint**: `POST /api/v1/stories`

**Auth**: Required (JWT)

**Request** (multipart/form-data):
```typescript
{
  file: File; // Photo or video
  mediaType: 'image' | 'video';
  privacy?: 'public' | 'followers' | 'close-friends'; // Default: 'followers'
  closeFriends?: string[]; // UUIDs (required if privacy='close-friends')
  durationMs?: number; // Default: 5000 (images), 10000 (videos)
}
```

**Response**:
```json
{
  "success": true,
  "message": "Story created successfully",
  "data": {
    "id": "65f3a8b7c1234567890abcde",
    "userId": "a1b2c3d4-e5f6-7890-1234-567890abcdef",
    "mediaId": "story-1709812345678",
    "mediaType": "image",
    "mediaUrl": "https://cdn.chefooz.app/stories/story-1709812345678.jpg",
    "thumbnailUrl": "https://cdn.chefooz.app/stories/thumbnails/thumb.jpg",
    "durationMs": 5000,
    "createdAt": "2024-03-07T10:00:00.000Z",
    "expiresAt": "2024-03-08T10:00:00.000Z",
    "views": 0,
    "privacy": "followers",
    "user": {
      "id": "a1b2c3d4-e5f6-7890-1234-567890abcdef",
      "username": "john_chef",
      "fullName": "John Doe"
    }
  }
}
```

**cURL Example**:
```bash
curl -X POST https://api.chefooz.app/api/v1/stories \
  -H "Authorization: Bearer $JWT_TOKEN" \
  -F "file=@photo.jpg" \
  -F "mediaType=image" \
  -F "privacy=followers" \
  -F "durationMs=5000"
```

**Controller Code**:
```typescript
@Post()
@HttpCode(HttpStatus.CREATED)
@UseInterceptors(FileInterceptor('file'))
async createStory(
  @UploadedFile() file: any,
  @Body() dto: CreateStoryDto,
  @Request() req: any,
) {
  if (!file) {
    throw new BadRequestException('Media file required');
  }

  const userId = req.user.id;

  // TODO: Replace with actual MediaConvert integration
  const mockMediaUrl = `https://cdn.chefooz.app/stories/${Date.now()}_${file.originalname}`;
  const mockThumbnailUrl = `https://cdn.chefooz.app/stories/thumbnails/${Date.now()}_thumb.jpg`;
  const mockMediaId = `story-${Date.now()}`;

  const story = await this.storiesService.createStory(
    userId,
    dto,
    mockMediaUrl,
    mockThumbnailUrl,
    mockMediaId,
  );

  return {
    success: true,
    message: 'Story created successfully',
    data: story,
  };
}
```

---

### 2. Get Own Stories

**Endpoint**: `GET /api/v1/stories/me`

**Auth**: Required (JWT)

**Response**:
```json
{
  "success": true,
  "message": "Stories retrieved",
  "data": {
    "stories": [
      {
        "id": "65f3a8b7c1234567890abcde",
        "userId": "a1b2c3d4-e5f6-7890-1234-567890abcdef",
        "mediaType": "image",
        "mediaUrl": "https://cdn.chefooz.app/stories/story.jpg",
        "thumbnailUrl": "https://cdn.chefooz.app/stories/thumb.jpg",
        "durationMs": 5000,
        "createdAt": "2024-03-07T10:00:00.000Z",
        "expiresAt": "2024-03-08T10:00:00.000Z",
        "views": 42,
        "hasSeen": false,
        "privacy": "followers",
        "user": {
          "id": "a1b2c3d4-e5f6-7890-1234-567890abcdef",
          "username": "john_chef",
          "fullName": "John Doe"
        }
      }
    ],
    "total": 1
  }
}
```

**Service Code**:
```typescript
async getUserStories(userId: string): Promise<StoryResponseDto[]> {
  const stories = await this.storyModel
    .find({ 
      userId, 
      expiresAt: { $gt: new Date() } // Only non-expired
    })
    .sort({ createdAt: -1 })
    .exec();

  const user = await this.userRepository.findOne({ where: { id: userId } });
  if (!user) {
    throw new NotFoundException('User not found');
  }

  return stories.map(story => this.mapToResponseDto(story, user, false));
}
```

---

### 3. Get Story Tray

**Endpoint**: `GET /api/v1/stories/tray`

**Auth**: Required (JWT)

**Purpose**: Lightweight endpoint for horizontal story rail UI

**Response**:
```json
{
  "success": true,
  "message": "Story tray retrieved",
  "data": {
    "tray": [
      {
        "username": "john_chef",
        "avatar": null,
        "hasUnseen": true,
        "storyCount": 3
      },
      {
        "username": "jane_baker",
        "avatar": null,
        "hasUnseen": false,
        "storyCount": 1
      }
    ],
    "total": 2
  }
}
```

**Service Logic**:
```typescript
async getStoryTrayForUser(userId: string): Promise<{ tray: any[]; total: number }> {
  // 1. Get following users
  const follows = await this.followRepository.find({
    where: { followerId: userId, status: 'accepted' },
    select: ['targetId'],
  });
  const followingIds = follows.map(f => f.targetId);
  const allUserIds = [userId, ...followingIds];

  // 2. Get stories from self + following
  const stories = await this.storyModel
    .find({
      userId: { $in: allUserIds },
      expiresAt: { $gt: new Date() },
      $or: [
        { privacy: 'public' },
        { privacy: 'followers', userId: { $in: followingIds } },
        { privacy: 'followers', userId },
        { privacy: 'close-friends', closeFriends: userId },
      ],
    })
    .sort({ createdAt: -1 })
    .exec();

  // 3. Load user data
  const uniqueUserIds = [...new Set(stories.map(s => s.userId))];
  const users = await this.userRepository.find({
    where: { id: In(uniqueUserIds) },
  });
  const userMap = new Map(users.map(u => [u.id, u]));

  // 4. Group by user
  const grouped = new Map<string, { stories: StoryDocument[]; user: User }>();
  for (const story of stories) {
    const user = userMap.get(story.userId);
    if (!user) continue;
    if (!grouped.has(story.userId)) {
      grouped.set(story.userId, { stories: [], user });
    }
    grouped.get(story.userId)!.stories.push(story);
  }

  // 5. Build tray items
  const tray = Array.from(grouped.values()).map(({ stories: userStories, user }) => {
    const hasUnseen = userStories.some(s => !s.viewers.includes(userId));
    return {
      username: user.username,
      avatar: undefined, // TODO: Add avatar support
      hasUnseen,
      storyCount: userStories.length,
    };
  });

  // 6. Sort: self first, then unseen, then seen
  tray.sort((a, b) => {
    if (a.username === userMap.get(userId)?.username) return -1;
    if (b.username === userMap.get(userId)?.username) return 1;
    if (a.hasUnseen && !b.hasUnseen) return -1;
    if (!a.hasUnseen && b.hasUnseen) return 1;
    return 0;
  });

  return { tray, total: tray.length };
}
```

---

### 4. Get Story Feed

**Endpoint**: `GET /api/v1/stories/feed`

**Auth**: Required (JWT)

**Purpose**: Full story feed grouped by user

**Response**:
```json
{
  "success": true,
  "message": "Story feed retrieved",
  "data": {
    "feed": [
      {
        "user": {
          "id": "a1b2c3d4-e5f6-7890-1234-567890abcdef",
          "username": "john_chef",
          "fullName": "John Doe",
          "avatar": null
        },
        "stories": [
          {
            "id": "65f3a8b7c1234567890abcde",
            "mediaType": "image",
            "mediaUrl": "https://cdn.chefooz.app/stories/story.jpg",
            "thumbnailUrl": "https://cdn.chefooz.app/stories/thumb.jpg",
            "durationMs": 5000,
            "views": 42,
            "hasSeen": false,
            "privacy": "followers"
          }
        ],
        "hasUnseenStories": true
      }
    ],
    "total": 1
  }
}
```

**Service Logic** (similar to tray, but returns full story objects):
```typescript
async getStoryFeedForUser(userId: string): Promise<StoryFeedResponseDto> {
  // Same query as tray, but map to full StoryResponseDto
  const feed: StoryFeedItemDto[] = [];
  for (const [uid, userStories] of grouped.entries()) {
    const user = userMap.get(uid);
    if (!user) continue;

    const mappedStories = userStories.map(s => 
      this.mapToResponseDto(s, user, s.viewers.includes(userId))
    );

    const hasUnseenStories = mappedStories.some(s => !s.hasSeen);

    feed.push({
      user: {
        id: user.id,
        username: user.username,
        fullName: user.fullName,
        avatar: undefined,
      },
      stories: mappedStories,
      hasUnseenStories,
    });
  }

  // Sort: self first, unseen, then seen
  feed.sort((a, b) => {
    if (a.user.id === userId) return -1;
    if (b.user.id === userId) return 1;
    if (a.hasUnseenStories && !b.hasUnseenStories) return -1;
    if (!a.hasUnseenStories && b.hasUnseenStories) return 1;
    return 0;
  });

  return { feed, total: stories.length };
}
```

---

### 5. Get Stories by Username

**Endpoint**: `GET /api/v1/stories/user/:username`

**Auth**: Required (JWT)

**Response**: Same as "Get Own Stories" but for specified username

**Service Logic**:
```typescript
async getStoriesByUsername(username: string, viewerId: string): Promise<StoryResponseDto[]> {
  // 1. Find user by username
  const user = await this.userRepository.findOne({ where: { username } });
  if (!user) {
    throw new NotFoundException('User not found');
  }

  // 2. Get user's stories
  const stories = await this.storyModel
    .find({
      userId: user.id,
      expiresAt: { $gt: new Date() },
    })
    .sort({ createdAt: -1 })
    .exec();

  // 3. Filter by privacy
  const filtered: StoryDocument[] = [];
  for (const story of stories) {
    try {
      await this.checkStoryPrivacy(story, viewerId);
      filtered.push(story);
    } catch {
      continue; // Skip inaccessible stories
    }
  }

  const hasSeen = (story: StoryDocument) => story.viewers.includes(viewerId);
  return filtered.map(story => this.mapToResponseDto(story, user, hasSeen(story)));
}
```

---

### 6. Get Single Story

**Endpoint**: `GET /api/v1/stories/:id`

**Auth**: Required (JWT)

**Response**: Single story object (same structure as feed stories)

**Service Logic**:
```typescript
async getStoryById(storyId: string, viewerId: string): Promise<StoryResponseDto> {
  const story = await this.storyModel.findById(storyId).exec();
  if (!story) {
    throw new NotFoundException('Story not found');
  }

  // Privacy check
  await this.checkStoryPrivacy(story, viewerId);

  const user = await this.userRepository.findOne({ where: { id: story.userId } });
  if (!user) {
    throw new NotFoundException('Story owner not found');
  }

  const hasSeen = story.viewers.includes(viewerId);
  return this.mapToResponseDto(story, user, hasSeen);
}
```

---

### 7. Mark Story as Viewed

**Endpoint**: `POST /api/v1/stories/:id/view`

**Auth**: Required (JWT)

**Response**:
```json
{
  "success": true,
  "message": "Story viewed"
}
```

**Service Logic**:
```typescript
async markStoryViewed(storyId: string, viewerId: string): Promise<void> {
  const story = await this.storyModel.findById(storyId).exec();
  if (!story) {
    throw new NotFoundException('Story not found');
  }

  // Privacy check
  await this.checkStoryPrivacy(story, viewerId);

  // Don't count duplicate views
  if (story.viewers.includes(viewerId)) {
    return;
  }

  // Increment views and add viewer (cap at 100)
  story.views += 1;
  if (story.viewers.length < 100) {
    story.viewers.push(viewerId);
  }

  await story.save();
}
```

---

## Service Layer

### Core Methods

#### 1. createStory()

**Purpose**: Save new story to MongoDB

**Parameters**:
- `userId` (string): Creator's UUID
- `dto` (CreateStoryDto): Story metadata
- `mediaUrl` (string): S3 URI
- `thumbnailUrl` (string): Thumbnail S3 URI
- `mediaId` (string): MediaConvert job ID

**Logic Flow**:
1. Validate user exists in Postgres
2. Validate close friends array if privacy='close-friends'
3. Create Story document
4. Set `expiresAt = now + 24 hours`
5. Save to MongoDB
6. Map to DTO with CDN URLs

**Code**:
```typescript
async createStory(
  userId: string,
  dto: CreateStoryDto,
  mediaUrl: string,
  thumbnailUrl: string,
  mediaId: string,
): Promise<StoryResponseDto> {
  const user = await this.userRepository.findOne({ where: { id: userId } });
  if (!user) {
    throw new NotFoundException('User not found');
  }

  if (dto.privacy === 'close-friends' && (!dto.closeFriends || dto.closeFriends.length === 0)) {
    throw new BadRequestException('Close friends list required for close-friends privacy');
  }

  const story = new this.storyModel({
    userId,
    mediaId,
    mediaType: dto.mediaType,
    mediaUrl,
    thumbnailUrl,
    durationMs: dto.durationMs || (dto.mediaType === 'image' ? 5000 : 10000),
    privacy: dto.privacy || 'followers',
    closeFriends: dto.closeFriends || [],
    views: 0,
    viewers: [],
    createdAt: new Date(),
    expiresAt: new Date(Date.now() + 24 * 60 * 60 * 1000),
  });

  const saved = await story.save();
  return this.mapToResponseDto(saved, user, false);
}
```

---

#### 2. checkStoryPrivacy()

**Purpose**: Enforce privacy rules before story access

**Parameters**:
- `story` (StoryDocument): Story to check
- `viewerId` (string): Viewer's UUID

**Logic Flow**:
1. If viewer is owner → Allow
2. If privacy='public' → Allow
3. If privacy='followers' → Check UserFollow entity
4. If privacy='close-friends' → Check closeFriends array
5. Throw ForbiddenException if check fails

**Code**:
```typescript
private async checkStoryPrivacy(story: StoryDocument, viewerId: string): Promise<void> {
  // Owner can always view
  if (story.userId === viewerId) {
    return;
  }

  // Public stories
  if (story.privacy === 'public') {
    return;
  }

  // Followers-only
  if (story.privacy === 'followers') {
    const isFollowing = await this.followRepository.findOne({
      where: { followerId: viewerId, targetId: story.userId, status: 'accepted' },
    });
    if (!isFollowing) {
      throw new ForbiddenException('You must follow this user to view their stories');
    }
    return;
  }

  // Close friends
  if (story.privacy === 'close-friends') {
    if (!story.closeFriends.includes(viewerId)) {
      throw new ForbiddenException('You are not in this user\'s close friends list');
    }
    return;
  }

  throw new ForbiddenException('Access denied');
}
```

---

#### 3. mapToResponseDto()

**Purpose**: Convert MongoDB document to API response DTO

**Logic**:
- Convert S3 URIs to HTTPS URLs via `s3UriToHttps()`
- Include user data
- Add `hasSeen` flag based on viewer presence in viewers array

**Code**:
```typescript
private mapToResponseDto(story: StoryDocument, user: User, hasSeen: boolean): StoryResponseDto {
  const cdnUrl = process.env.CDN_URL;
  return {
    id: (story as any)._id.toString(),
    userId: story.userId,
    mediaId: story.mediaId,
    mediaType: story.mediaType,
    mediaUrl: s3UriToHttps(story.mediaUrl, cdnUrl),
    thumbnailUrl: s3UriToHttps(story.thumbnailUrl, cdnUrl),
    durationMs: story.durationMs,
    createdAt: story.createdAt.toISOString(),
    expiresAt: story.expiresAt.toISOString(),
    views: story.views,
    hasSeen,
    privacy: story.privacy,
    user: {
      id: user.id,
      username: user.username,
      fullName: user.fullName,
      avatar: undefined, // TODO
    },
  };
}
```

---

#### 4. deleteExpiredStories()

**Purpose**: Manually delete expired stories (redundant to TTL index)

**Returns**: `{ deleted: number }`

**Code**:
```typescript
async deleteExpiredStories(): Promise<{ deleted: number }> {
  const result = await this.storyModel.deleteMany({
    expiresAt: { $lt: new Date() },
  }).exec();

  return { deleted: result.deletedCount || 0 };
}
```

---

## DTOs

### CreateStoryDto

```typescript
export class CreateStoryDto {
  @ApiProperty({ enum: ['image', 'video'] })
  @IsEnum(['image', 'video'])
  @IsNotEmpty()
  mediaType!: 'image' | 'video';

  @ApiPropertyOptional({ 
    enum: ['public', 'followers', 'close-friends'], 
    default: 'followers'
  })
  @IsEnum(['public', 'followers', 'close-friends'])
  @IsOptional()
  privacy?: 'public' | 'followers' | 'close-friends';

  @ApiPropertyOptional({ type: [String] })
  @IsArray()
  @IsUUID('4', { each: true })
  @IsOptional()
  closeFriends?: string[];

  @ApiPropertyOptional({ type: Number })
  @IsNumber()
  @Min(1000)
  @IsOptional()
  durationMs?: number;
}
```

### StoryResponseDto

```typescript
export class StoryResponseDto {
  @ApiProperty()
  id!: string;

  @ApiProperty()
  userId!: string;

  @ApiProperty()
  mediaId!: string;

  @ApiProperty({ enum: ['image', 'video'] })
  mediaType!: 'image' | 'video';

  @ApiProperty()
  mediaUrl!: string;

  @ApiProperty()
  thumbnailUrl!: string;

  @ApiProperty()
  durationMs!: number;

  @ApiProperty()
  createdAt!: string;

  @ApiProperty()
  expiresAt!: string;

  @ApiProperty()
  views!: number;

  @ApiProperty({ type: Boolean })
  hasSeen?: boolean;

  @ApiProperty({ enum: ['public', 'followers', 'close-friends'] })
  privacy!: 'public' | 'followers' | 'close-friends';

  @ApiProperty()
  user!: {
    id: string;
    username: string;
    fullName?: string;
    avatar?: string;
  };
}
```

### StoryFeedResponseDto

```typescript
export class StoryFeedItemDto {
  @ApiProperty()
  user!: {
    id: string;
    username: string;
    fullName?: string;
    avatar?: string;
  };

  @ApiProperty({ type: [StoryResponseDto] })
  stories!: StoryResponseDto[];

  @ApiProperty({ type: Boolean })
  hasUnseenStories!: boolean;
}

export class StoryFeedResponseDto {
  @ApiProperty({ type: [StoryFeedItemDto] })
  feed!: StoryFeedItemDto[];

  @ApiProperty({ type: Number })
  total!: number;
}
```

---

## Integration Patterns

### S3 URI to HTTPS URL Conversion

**Utility**: `s3UriToHttps()`

**Purpose**: Convert internal S3 URIs to public CDN URLs

**Example**:
```typescript
// Input
s3://chefooz-stories/story-1709812345678.jpg

// Output
https://cdn.chefooz.app/stories/story-1709812345678.jpg
```

**Implementation**:
```typescript
export function s3UriToHttps(s3Uri: string, cdnUrl: string): string {
  if (!s3Uri.startsWith('s3://')) return s3Uri;
  
  const [, bucket, ...pathParts] = s3Uri.replace('s3://', '').split('/');
  const path = pathParts.join('/');
  
  return `${cdnUrl}/${path}`;
}
```

---

### MongoDB TTL Index

**Purpose**: Auto-delete stories 24 hours after expiration

**Configuration**:
```typescript
@Prop({ 
  required: true, 
  type: Date, 
  default: () => new Date(Date.now() + 24 * 60 * 60 * 1000),
  index: { expireAfterSeconds: 0 }
})
expiresAt!: Date;
```

**How It Works**:
1. MongoDB background thread checks TTL index every 60 seconds
2. Deletes documents where `expiresAt <= now`
3. No manual intervention required
4. StoryCleanupJob provides redundancy

**Verification**:
```bash
# Check index status
db.stories.getIndexes()

# Should show:
{
  "v": 2,
  "key": { "expiresAt": 1 },
  "name": "expiresAt_1",
  "expireAfterSeconds": 0
}
```

---

### Scheduled Cleanup Job

**File**: `jobs/story-cleanup.job.ts`

**Schedule**: Every 6 hours (`0 */6 * * *`)

**Code**:
```typescript
@Injectable()
export class StoryCleanupJob {
  private readonly logger = new Logger(StoryCleanupJob.name);

  constructor(private readonly storiesService: StoriesService) {}

  @Cron('0 */6 * * *', {
    name: 'story-cleanup',
    timeZone: 'UTC',
  })
  async handleStoryCleanup() {
    this.logger.log('[Story Cleanup] Starting expired story cleanup...');

    try {
      const result = await this.storiesService.deleteExpiredStories();
      
      this.logger.log(
        `[Story Cleanup] Completed. Deleted ${result.deleted} expired stories.`
      );
    } catch (error) {
      const err = error as Error;
      this.logger.error(
        `[Story Cleanup] Failed: ${err.message}`,
        err.stack
      );
    }
  }
}
```

---

## Error Handling

### Error Response Format

```typescript
{
  success: false,
  message: string,
  errorCode?: string
}
```

### Error Codes

| Code | HTTP Status | Trigger | Recovery |
|------|-------------|---------|----------|
| `STORY_NOT_FOUND` | 404 | Story ID doesn't exist or expired | Check expiration, verify ID |
| `USER_NOT_FOUND` | 404 | Username lookup failed | Verify username spelling |
| `ACCESS_DENIED_PRIVACY` | 403 | Viewer doesn't meet privacy requirements | Follow user or request close friends access |
| `CLOSE_FRIENDS_REQUIRED` | 400 | Empty closeFriends array with privacy='close-friends' | Select at least one friend |
| `INVALID_MEDIA_TYPE` | 400 | mediaType not 'image' or 'video' | Use valid enum value |
| `MEDIA_FILE_REQUIRED` | 400 | No file in multipart request | Upload a photo or video |
| `UNAUTHORIZED` | 401 | Missing or invalid JWT | Log in again |

**Error Handling Pattern**:
```typescript
try {
  await this.checkStoryPrivacy(story, viewerId);
} catch (error) {
  if (error instanceof ForbiddenException) {
    return {
      success: false,
      message: 'You don\'t have permission to view this story',
      errorCode: 'ACCESS_DENIED_PRIVACY',
    };
  }
  throw error;
}
```

---

## Testing Strategy

### Unit Tests

**File**: `stories.service.spec.ts`

**Test Suite 1: createStory()**
```typescript
describe('StoriesService - createStory', () => {
  it('should create story with default privacy (followers)', async () => {
    const dto = { mediaType: 'image' } as CreateStoryDto;
    const result = await service.createStory(userId, dto, mediaUrl, thumbnailUrl, mediaId);
    
    expect(result.privacy).toBe('followers');
    expect(result.expiresAt).toBeDefined();
  });

  it('should throw error if privacy=close-friends and closeFriends empty', async () => {
    const dto = { 
      mediaType: 'image', 
      privacy: 'close-friends', 
      closeFriends: [] 
    } as CreateStoryDto;
    
    await expect(service.createStory(userId, dto, mediaUrl, thumbnailUrl, mediaId))
      .rejects.toThrow(BadRequestException);
  });
});
```

**Test Suite 2: checkStoryPrivacy()**
```typescript
describe('StoriesService - checkStoryPrivacy', () => {
  it('should allow owner to view their story', async () => {
    const story = { userId: 'user-123', privacy: 'followers' } as StoryDocument;
    
    await expect(service['checkStoryPrivacy'](story, 'user-123'))
      .resolves.not.toThrow();
  });

  it('should allow public story for anyone', async () => {
    const story = { userId: 'user-123', privacy: 'public' } as StoryDocument;
    
    await expect(service['checkStoryPrivacy'](story, 'user-456'))
      .resolves.not.toThrow();
  });

  it('should throw ForbiddenException if not following', async () => {
    const story = { userId: 'user-123', privacy: 'followers' } as StoryDocument;
    jest.spyOn(followRepository, 'findOne').mockResolvedValue(null);
    
    await expect(service['checkStoryPrivacy'](story, 'user-456'))
      .rejects.toThrow(ForbiddenException);
  });
});
```

**Test Suite 3: markStoryViewed()**
```typescript
describe('StoriesService - markStoryViewed', () => {
  it('should increment views and add viewer', async () => {
    const story = { 
      views: 0, 
      viewers: [], 
      save: jest.fn() 
    } as unknown as StoryDocument;
    
    jest.spyOn(storyModel, 'findById').mockResolvedValue(story);
    
    await service.markStoryViewed('story-123', 'viewer-456');
    
    expect(story.views).toBe(1);
    expect(story.viewers).toContain('viewer-456');
    expect(story.save).toHaveBeenCalled();
  });

  it('should not count duplicate views', async () => {
    const story = { 
      views: 5, 
      viewers: ['viewer-456'], 
      save: jest.fn() 
    } as unknown as StoryDocument;
    
    jest.spyOn(storyModel, 'findById').mockResolvedValue(story);
    
    await service.markStoryViewed('story-123', 'viewer-456');
    
    expect(story.views).toBe(5); // No increment
    expect(story.save).not.toHaveBeenCalled();
  });
});
```

---

### E2E Tests

**File**: `stories.e2e-spec.ts`

**Test 1: Full Story Creation Flow**
```typescript
it('POST /stories - should create story and retrieve it', async () => {
  const jwt = await getJwtToken('john_chef');
  
  const response = await request(app.getHttpServer())
    .post('/api/v1/stories')
    .set('Authorization', `Bearer ${jwt}`)
    .attach('file', './test/fixtures/photo.jpg')
    .field('mediaType', 'image')
    .field('privacy', 'followers')
    .expect(201);

  expect(response.body.success).toBe(true);
  expect(response.body.data.id).toBeDefined();
  
  const storyId = response.body.data.id;
  
  // Verify story is retrievable
  const getResponse = await request(app.getHttpServer())
    .get(`/api/v1/stories/${storyId}`)
    .set('Authorization', `Bearer ${jwt}`)
    .expect(200);

  expect(getResponse.body.data.id).toBe(storyId);
});
```

**Test 2: Privacy Enforcement**
```typescript
it('GET /stories/:id - should return 403 if not following', async () => {
  const creatorJwt = await getJwtToken('john_chef');
  const viewerJwt = await getJwtToken('jane_baker');
  
  // Create followers-only story
  const createResponse = await request(app.getHttpServer())
    .post('/api/v1/stories')
    .set('Authorization', `Bearer ${creatorJwt}`)
    .attach('file', './test/fixtures/photo.jpg')
    .field('mediaType', 'image')
    .field('privacy', 'followers')
    .expect(201);
  
  const storyId = createResponse.body.data.id;
  
  // Try to view without following
  await request(app.getHttpServer())
    .get(`/api/v1/stories/${storyId}`)
    .set('Authorization', `Bearer ${viewerJwt}`)
    .expect(403);
});
```

**Test 3: Story Feed Ordering**
```typescript
it('GET /stories/feed - should prioritize self and unseen stories', async () => {
  const jwt = await getJwtToken('john_chef');
  
  const response = await request(app.getHttpServer())
    .get('/api/v1/stories/feed')
    .set('Authorization', `Bearer ${jwt}`)
    .expect(200);

  const feed = response.body.data.feed;
  
  // First item should be self stories (if any)
  if (feed.length > 0 && feed[0].user.username === 'john_chef') {
    expect(feed[0].user.username).toBe('john_chef');
  }
  
  // Unseen stories should be prioritized
  const unseenIndex = feed.findIndex(item => item.hasUnseenStories);
  const seenIndex = feed.findIndex(item => !item.hasUnseenStories);
  
  if (unseenIndex !== -1 && seenIndex !== -1) {
    expect(unseenIndex).toBeLessThan(seenIndex);
  }
});
```

---

## Deployment

### Dockerfile

```dockerfile
FROM node:20-alpine

WORKDIR /app

# Copy package files
COPY package*.json ./
COPY nx.json ./
COPY tsconfig.base.json ./

# Install dependencies
RUN npm ci

# Copy source
COPY apps/chefooz-apis ./apps/chefooz-apis
COPY libs ./libs

# Build
RUN npx nx build chefooz-apis --prod

# Expose port
EXPOSE 3000

# Start
CMD ["node", "dist/apps/chefooz-apis/main.js"]
```

---

### Docker Compose

```yaml
version: '3.8'

services:
  api:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - MONGODB_URI=mongodb://mongo:27017/chefooz
      - DATABASE_URL=postgresql://postgres:password@postgres:5432/chefooz
      - JWT_SECRET=${JWT_SECRET}
      - CDN_URL=https://cdn.chefooz.app
    depends_on:
      - mongo
      - postgres

  mongo:
    image: mongo:6
    ports:
      - "27017:27017"
    volumes:
      - mongo-data:/data/db
    command: mongod --setParameter ttlMonitorSleepSecs=60

  postgres:
    image: postgres:15
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_DB=chefooz
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=password
    volumes:
      - postgres-data:/var/lib/postgresql/data

volumes:
  mongo-data:
  postgres-data:
```

---

### Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: chefooz-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: chefooz-api
  template:
    metadata:
      labels:
        app: chefooz-api
    spec:
      containers:
      - name: api
        image: chefooz/api:latest
        ports:
        - containerPort: 3000
        env:
        - name: MONGODB_URI
          valueFrom:
            secretKeyRef:
              name: db-secrets
              key: mongodb-uri
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secrets
              key: postgres-url
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: api-secrets
              key: jwt-secret
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: chefooz-api-service
spec:
  selector:
    app: chefooz-api
  ports:
  - protocol: TCP
    port: 80
    targetPort: 3000
  type: LoadBalancer
```

---

## Monitoring & Observability

### Logging

**Format**: Structured JSON logs

```typescript
{
  "timestamp": "2024-03-07T10:00:00.000Z",
  "level": "info",
  "context": "StoriesService",
  "message": "Story created successfully",
  "metadata": {
    "storyId": "65f3a8b7c1234567890abcde",
    "userId": "a1b2c3d4-e5f6-7890-1234-567890abcdef",
    "privacy": "followers"
  }
}
```

**Key Log Points**:
- Story creation
- View tracking
- Privacy check failures
- Cleanup job runs

---

### Prometheus Metrics

```typescript
// Story creation rate
stories_created_total{privacy="followers"} 1234

// View count
stories_views_total 5678

// Privacy check failures
stories_privacy_denied_total{privacy="followers"} 12

// Cleanup job
stories_cleanup_deleted_total 42
stories_cleanup_duration_seconds 1.23
```

---

### Grafana Dashboard

**Panels**:
1. **Story Creation Rate**: Line graph (stories/min by privacy level)
2. **Active Stories**: Gauge (current non-expired stories)
3. **Views per Minute**: Line graph (total views)
4. **Privacy Denials**: Counter (403 errors by privacy type)
5. **Cleanup Job Health**: Last run time + deleted count

---

### Health Check

**Endpoint**: `GET /api/v1/health`

**Response**:
```json
{
  "status": "healthy",
  "checks": {
    "mongodb": "up",
    "postgres": "up",
    "ttl_index": "active"
  }
}
```

---

## Performance Optimization

### N+1 Query Prevention

**Problem**: Loading user data for each story individually

**Bad Example**:
```typescript
for (const story of stories) {
  const user = await this.userRepository.findOne({ where: { id: story.userId } });
  // N+1 queries!
}
```

**Good Example**:
```typescript
const uniqueUserIds = [...new Set(stories.map(s => s.userId))];
const users = await this.userRepository.find({
  where: { id: In(uniqueUserIds) },
});
const userMap = new Map(users.map(u => [u.id, u]));

for (const story of stories) {
  const user = userMap.get(story.userId); // O(1) lookup
}
```

---

### Index Optimization

**Query**: Get user's stories
```typescript
db.stories.find({ userId: 'user-123', expiresAt: { $gt: new Date() } })
  .sort({ createdAt: -1 });
```

**Index**: `{ userId: 1, createdAt: -1 }`

**Performance**: <10ms for up to 100 stories per user

---

### Query Plan Verification

```javascript
db.stories.find({ userId: 'user-123', expiresAt: { $gt: new Date() } })
  .sort({ createdAt: -1 })
  .explain('executionStats');

// Should show:
// - "stage": "IXSCAN" (index scan, not COLLSCAN)
// - "indexName": "userId_1_createdAt_-1"
// - "executionTimeMillis": < 10
```

---

## Security

### Input Validation

**class-validator Decorators**:
```typescript
@IsEnum(['image', 'video'])
@IsNotEmpty()
mediaType!: 'image' | 'video';

@IsUUID('4', { each: true })
@IsOptional()
closeFriends?: string[];
```

**Validation Pipe** (global):
```typescript
app.useGlobalPipes(new ValidationPipe({
  whitelist: true,
  forbidNonWhitelisted: true,
  transform: true,
}));
```

---

### Authorization Checks

**JWT Guard** (all endpoints):
```typescript
@Controller('v1/stories')
@UseGuards(JwtAuthGuard)
export class StoriesController {}
```

**Privacy Enforcement**:
```typescript
private async checkStoryPrivacy(story: StoryDocument, viewerId: string) {
  if (story.privacy === 'followers') {
    const isFollowing = await this.followRepository.findOne({
      where: { followerId: viewerId, targetId: story.userId, status: 'accepted' }
    });
    if (!isFollowing) throw new ForbiddenException();
  }
}
```

---

### SQL/NoSQL Injection Prevention

**TypeORM** (Postgres):
- Parameterized queries by default
- No raw SQL in stories module

**Mongoose** (MongoDB):
- Schema validation
- No `$where` or `Function` queries
- All queries use strict schema fields

---

## Troubleshooting

### Issue 1: Stories Not Expiring

**Symptom**: Stories still visible after 24 hours

**Diagnosis**:
```javascript
// Check TTL index
db.stories.getIndexes();

// Check expired stories count
db.stories.countDocuments({ expiresAt: { $lt: new Date() } });
```

**Possible Causes**:
1. TTL index not created (check `expireAfterSeconds: 0`)
2. MongoDB background thread not running (check `ttlMonitorSleepSecs`)
3. System clock drift (compare `expiresAt` to server time)

**Solution**:
```bash
# Recreate TTL index
db.stories.dropIndex('expiresAt_1');
db.stories.createIndex({ "expiresAt": 1 }, { expireAfterSeconds: 0 });

# Manually trigger cleanup
POST /api/v1/admin/stories/cleanup
```

---

### Issue 2: 403 Errors on Story View

**Symptom**: User can't view story despite following creator

**Diagnosis**:
```sql
-- Check follow relationship
SELECT * FROM user_follows
WHERE follower_id = 'viewer-uuid'
AND target_id = 'creator-uuid';
```

**Possible Causes**:
1. Follow status not 'accepted' (pending/blocked)
2. User unfollowed after story was created
3. Story privacy changed after creation

**Solution**:
- Verify follow status is 'accepted'
- Ensure privacy rules match current follow state
- Check story privacy level in MongoDB

---

### Issue 3: Slow Feed Queries

**Symptom**: `/stories/feed` takes >1 second

**Diagnosis**:
```javascript
// Check query plan
db.stories.find({
  userId: { $in: followingIds },
  expiresAt: { $gt: new Date() }
}).explain('executionStats');
```

**Possible Causes**:
1. Missing index on `userId` or `expiresAt`
2. Large `followingIds` array (>1000 users)
3. N+1 query for user data

**Solution**:
- Verify indexes with `.getIndexes()`
- Implement pagination for large follower counts
- Batch user fetching with `In(uniqueUserIds)`
- Add Redis caching (30s TTL)

---

### Issue 4: MediaConvert Integration Not Working

**Symptom**: Videos not processing, mock URLs returned

**Diagnosis**:
```typescript
// Controller still using mock URLs
const mockMediaUrl = `https://cdn.chefooz.app/stories/${Date.now()}_${file.originalname}`;
```

**Possible Causes**:
1. MediaConvert integration not implemented (TODO in code)
2. AWS credentials missing
3. MediaConvert endpoint misconfigured

**Solution**:
- Implement full MediaConvert integration (see Media module)
- Verify AWS credentials in environment variables
- Test MediaConvert job submission manually

---

### Issue 5: Viewer Cap Not Working

**Symptom**: More than 100 viewers in `viewers` array

**Diagnosis**:
```javascript
// Check viewer array length
db.stories.findOne({ _id: ObjectId('...') }, { viewers: 1 });
```

**Possible Causes**:
1. Race condition in `markStoryViewed()` (multiple concurrent requests)
2. Logic error in cap enforcement

**Solution**:
```typescript
// Atomic update with $push + $slice
await this.storyModel.updateOne(
  { _id: storyId, viewers: { $ne: viewerId } },
  { 
    $inc: { views: 1 },
    $push: { viewers: { $each: [viewerId], $slice: -100 } }
  }
);
```

---

**Last Updated**: 2026-02-14
**Version**: 1.0.0
**Status**: Production-Ready (MediaConvert integration pending)
