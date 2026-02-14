# Stories Module - Feature Overview

## Executive Summary

The **Stories Module** provides Instagram-style ephemeral content that disappears after 24 hours. Users can share photos and videos visible to their followers or close friends, with built-in privacy controls and view tracking. Stories are stored in MongoDB with automatic TTL-based expiration, ensuring zero manual cleanup overhead. The module supports video processing via AWS MediaConvert, view count tracking (with capped viewer lists), and privacy-based feed generation (public, followers-only, close friends).

**Key Differentiators:**
- **24-Hour Auto-Expiration**: MongoDB TTL index automatically deletes expired stories without manual intervention
- **3-Tier Privacy System**: Public (everyone), Followers-only (mutual follows), Close Friends (custom list)
- **View Tracking**: Increments view count + stores first 100 viewers per story
- **Story Feed**: Grouped by user, prioritizes unseen content, respects privacy rules
- **MediaConvert Integration**: AWS MediaConvert handles video transcoding, HLS variants, thumbnails

---

## Module Capabilities

### Core Features

1. **Story Creation**
   - Upload photos or videos via multipart/form-data
   - Choose privacy level: public, followers, close-friends
   - Set custom display duration (default: 5s images, 10s videos)
   - AWS MediaConvert processes videos (HLS adaptive streaming)
   - Automatic expiration after 24 hours

2. **Story Viewing**
   - View single story by ID (with privacy enforcement)
   - Get story feed (self + following users, unseen prioritized)
   - Get story tray (lightweight horizontal rail for UI)
   - Get user's own stories
   - Get stories by username (respects privacy)
   - Mark story as viewed (increments count, adds viewer to list)

3. **Privacy Enforcement**
   - **Public**: Anyone can view
   - **Followers**: Only users who follow the creator
   - **Close Friends**: Only specific users in closeFriends array
   - Privacy checks applied at both service and controller levels

4. **View Tracking**
   - Increments `views` counter on each view
   - Stores viewer UUIDs in `viewers` array (capped at 100 to prevent bloat)
   - Prevents duplicate views (same user viewing multiple times counted once)
   - Returns `hasSeen` flag in API responses for UI state

5. **Automatic Expiration**
   - Stories expire 24 hours after creation
   - MongoDB TTL index on `expiresAt` field handles deletion
   - Scheduled cleanup job runs every 6 hours (redundancy + logging)
   - No manual cleanup required

---

## System Architecture

### High-Level Components

```
┌─────────────────┐
│  Mobile App     │
│  (Expo)         │
└────────┬────────┘
         │ POST /api/v1/stories (multipart)
         │ GET  /api/v1/stories/tray
         │ GET  /api/v1/stories/feed
         │ POST /api/v1/stories/:id/view
         ↓
┌─────────────────────────────────────────┐
│  NestJS API Gateway                     │
│  JWT Auth + StoriesController           │
└────────┬────────────────────────────────┘
         │
         ↓
┌─────────────────────────────────────────┐
│  StoriesService                         │
│  - createStory()                        │
│  - getStoryTrayForUser()                │
│  - getStoryFeedForUser()                │
│  - markStoryViewed()                    │
│  - checkStoryPrivacy()                  │
└────┬────────────────────────────────────┘
     │
     ├─────────────────────────────────────┐
     │                                     │
     ↓                                     ↓
┌─────────────────┐              ┌─────────────────┐
│  MongoDB        │              │  PostgreSQL     │
│  stories        │              │  users          │
│  (TTL index)    │              │  user_follows   │
└─────────────────┘              └─────────────────┘
     │
     ↓
┌─────────────────────────────────┐
│  AWS MediaConvert               │
│  (Video transcoding)            │
└─────────────────────────────────┘
```

### Technology Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **Backend** | NestJS 10.x | API framework |
| **Primary Storage** | MongoDB 6.x (Mongoose) | Story documents with TTL expiration |
| **Secondary Storage** | PostgreSQL 15.x (TypeORM) | User profiles + follow relationships |
| **Media Processing** | AWS MediaConvert | Video transcoding, HLS, thumbnails |
| **CDN** | CloudFront / S3 | Media delivery |
| **Scheduling** | @nestjs/schedule | Cron jobs for cleanup |
| **Authentication** | JWT | User authentication via JwtAuthGuard |

---

## Database Design

### MongoDB Story Document

```json
{
  "_id": "65f3a8b7c1234567890abcde",
  "userId": "a1b2c3d4-e5f6-7890-1234-567890abcdef",
  "mediaId": "story-1709812345678",
  "mediaType": "video",
  "mediaUrl": "s3://chefooz-stories/output/story-1709812345678.m3u8",
  "thumbnailUrl": "s3://chefooz-stories/thumbnails/story-1709812345678-thumb.jpg",
  "durationMs": 10000,
  "createdAt": "2024-03-07T10:00:00.000Z",
  "expiresAt": "2024-03-08T10:00:00.000Z",
  "views": 42,
  "viewers": [
    "b2c3d4e5-f6a7-8901-2345-678901bcdefg",
    "c3d4e5f6-a7b8-9012-3456-789012cdefgh"
  ],
  "privacy": "followers",
  "closeFriends": []
}
```

**Field Descriptions:**

| Field | Type | Description |
|-------|------|-------------|
| `_id` | ObjectId | MongoDB unique identifier |
| `userId` | string (UUID) | Story creator's UUID (FK to Postgres `users.id`) |
| `mediaId` | string | MediaConvert job ID or unique identifier |
| `mediaType` | enum | 'image' or 'video' |
| `mediaUrl` | string | S3 URI to final media (HLS for videos) |
| `thumbnailUrl` | string | S3 URI to thumbnail or compressed image |
| `durationMs` | number | Display duration (5000ms default images, 10000ms videos) |
| `createdAt` | Date | Story creation timestamp |
| `expiresAt` | Date | Expiration timestamp (24 hours after createdAt) |
| `views` | number | Total view count |
| `viewers` | string[] | UUID array of viewers (capped at 100) |
| `privacy` | enum | 'public', 'followers', 'close-friends' |
| `closeFriends` | string[] | UUID array (only used if privacy='close-friends') |

**Indexes:**

1. **TTL Index**: `expiresAt` (expires documents 0 seconds after `expiresAt` date)
2. **User Stories**: `{ userId: 1, createdAt: -1 }` (get user's stories newest first)
3. **Privacy Filter**: `{ privacy: 1, createdAt: -1 }` (filter by privacy level)
4. **Popularity Sort**: `{ views: -1 }` (sort by view count)

### PostgreSQL Entities

**users** (TypeORM)
```typescript
{
  id: string (UUID primary key)
  username: string (unique)
  fullName: string
  // Avatar support pending (TODO in service layer)
}
```

**user_follows** (TypeORM)
```typescript
{
  id: string (UUID primary key)
  followerId: string (UUID, FK to users.id)
  targetId: string (UUID, FK to users.id)
  status: enum ('pending', 'accepted', 'blocked')
  createdAt: Date
}
```

---

## Integration Points

### 1. Media Module
**Purpose**: Handle S3 uploads and MediaConvert job triggers

**Integration Flow:**
1. Client calls Media service to get presigned S3 URL
2. Client uploads media directly to S3
3. Backend copies to MediaConvert input bucket
4. MediaConvert processes video (transcoding, thumbnails, HLS)
5. Webhook updates story document with final URLs

**Current State**: TODO - Using mock URLs in `createStory()` controller

**Code Location:**
```typescript
// StoriesController.createStory()
// TODO: Replace mock URLs with actual MediaConvert integration
const mockMediaUrl = `https://cdn.chefooz.app/stories/${Date.now()}_${file.originalname}`;
const mockThumbnailUrl = `https://cdn.chefooz.app/stories/thumbnails/${Date.now()}_thumb.jpg`;
```

### 2. User Module
**Purpose**: Retrieve user profiles for story feed

**Service Method:**
```typescript
async getStoryTrayForUser(userId: string) {
  const users = await this.userRepository.find({
    where: { id: In(uniqueUserIds) },
  });
  // Map users to stories
}
```

**Data Retrieved:**
- `username`, `fullName`, `avatar` (TODO: avatar field not yet in User entity)

### 3. Social Module (UserFollow)
**Purpose**: Determine who can view stories based on follow relationships

**Service Method:**
```typescript
async checkStoryPrivacy(story: StoryDocument, viewerId: string) {
  if (story.privacy === 'followers') {
    const isFollowing = await this.followRepository.findOne({
      where: { followerId: viewerId, targetId: story.userId, status: 'accepted' },
    });
    if (!isFollowing) throw new ForbiddenException();
  }
}
```

**Used In:**
- Story feed generation (only show stories from followed users)
- Privacy enforcement (check if viewer can access story)

### 4. Schedule Module
**Purpose**: Run periodic cleanup jobs

**Cron Job:**
```typescript
@Cron('0 */6 * * *', { name: 'story-cleanup', timeZone: 'UTC' })
async handleStoryCleanup() {
  const result = await this.storiesService.deleteExpiredStories();
  // Logs deleted count
}
```

**Schedule**: Every 6 hours
**Purpose**: Redundant cleanup (MongoDB TTL already handles auto-deletion)

---

## Business Rules

### Story Expiration Rules

1. **24-Hour Lifespan**: Stories expire exactly 24 hours after creation
2. **Automatic Deletion**: MongoDB TTL index deletes expired documents (no manual intervention)
3. **Grace Period**: None - TTL index deletes within ~60 seconds of expiration
4. **Redundant Cleanup**: Cron job runs every 6 hours for logging and metrics
5. **No Soft Delete**: Stories are permanently deleted (ephemeral by design)

### Privacy Rules

| Privacy Level | Who Can View | Close Friends Required? |
|---------------|-------------|-------------------------|
| `public` | Everyone (authenticated users) | No |
| `followers` | Users who follow the creator (status='accepted') | No |
| `close-friends` | Only users in `closeFriends` array | Yes (400 error if empty) |

**Privacy Enforcement:**
- Checked in `checkStoryPrivacy()` service method
- Applied to: single story view, feed generation, username lookup
- Owner can always view their own stories
- 403 error if privacy check fails

### View Tracking Rules

1. **Duplicate Views**: Same user viewing story multiple times counted once
2. **Viewer Cap**: Only first 100 viewers stored in `viewers` array
3. **View Count**: Always incremented (no cap)
4. **Anonymous Views**: Not supported - JWT required for all endpoints
5. **Owner Views**: Not counted (owner can view own story without incrementing)

### Feed Ordering Rules

**Story Feed** (`GET /feed`):
1. Self stories (if any) appear first
2. Close friends stories (unseen prioritized)
3. Following users stories (unseen prioritized)
4. Within each group, sorted by newest first

**Story Tray** (`GET /tray`):
1. Self stories (if any) appear first with "+" icon
2. Unseen stories prioritized
3. Seen stories at the end
4. Returns minimal data: `username`, `avatar`, `hasUnseen`, `storyCount`

---

## User Flows

### Flow 1: Create Story (User)

**Actors**: User (authenticated)

**Preconditions**:
- User logged in (JWT valid)
- Media file ready (photo or video)

**Steps**:
1. User taps "+" icon in story tray
2. Mobile app opens camera or media picker
3. User selects/captures media
4. User chooses privacy level (public / followers / close-friends)
5. If close-friends selected, user picks friends from list
6. User taps "Share" button
7. Mobile app uploads media to S3 (via presigned URL from Media service)
8. Mobile app calls `POST /api/v1/stories` with metadata
9. Backend validates JWT and media type
10. Backend saves story document with `expiresAt = now + 24h`
11. If video, triggers MediaConvert job (TODO: integration pending)
12. Backend returns story object
13. Mobile app shows success toast
14. Story appears in user's profile tray

**Success Criteria**:
- Story document created in MongoDB
- `expiresAt` set to 24 hours from now
- Story visible to appropriate audience based on privacy
- Video processing starts (if applicable)

### Flow 2: View Story Feed (User)

**Actors**: User (authenticated)

**Preconditions**:
- User logged in
- User follows at least one other user

**Steps**:
1. User opens home screen
2. Mobile app calls `GET /api/v1/stories/tray`
3. Backend queries MongoDB for stories from self + following users
4. Backend checks privacy rules for each story
5. Backend groups stories by user
6. Backend returns tray items with `hasUnseen` flag
7. Mobile app renders horizontal story rail
8. User taps a story avatar
9. Mobile app calls `GET /api/v1/stories/user/:username`
10. Backend returns all stories for that user (respecting privacy)
11. Mobile app displays stories in vertical swipe viewer
12. For each story viewed, mobile app calls `POST /api/v1/stories/:id/view`
13. Backend increments `views` and adds user to `viewers` array
14. Story circle changes from gradient (unseen) to gray (seen)

**Success Criteria**:
- Story tray shows self + following users
- Unseen stories prioritized
- View counts updated in real-time
- Privacy rules enforced (no unauthorized access)

### Flow 3: View Specific User's Stories

**Actors**: User A (viewer), User B (creator)

**Preconditions**:
- Both users authenticated
- User A follows User B (if privacy='followers')
- User B has active stories

**Steps**:
1. User A visits User B's profile
2. User A taps story ring around profile picture
3. Mobile app calls `GET /api/v1/stories/user/{username}`
4. Backend retrieves User B's UUID from username
5. Backend queries MongoDB for User B's stories where `expiresAt > now`
6. Backend checks privacy for each story:
   - Public: Allow
   - Followers: Check if User A follows User B
   - Close-friends: Check if User A in `closeFriends` array
7. Backend filters out inaccessible stories
8. Backend returns story list
9. Mobile app displays stories
10. User A views each story
11. Backend increments `views` and adds User A to `viewers`

**Success Criteria**:
- Only accessible stories returned (privacy enforced)
- View count incremented for each viewed story
- No 403 errors for inaccessible stories (filtered silently)

### Flow 4: Story Expires (System)

**Actors**: MongoDB TTL Index, StoryCleanupJob (cron)

**Preconditions**:
- Story created 24 hours ago
- `expiresAt` timestamp reached

**Steps**:
1. Story's `expiresAt` timestamp passes
2. MongoDB TTL index detects expired document (~60s delay)
3. MongoDB deletes story document automatically
4. Every 6 hours, StoryCleanupJob runs
5. Job calls `StoriesService.deleteExpiredStories()`
6. Service queries `{ expiresAt: { $lt: new Date() } }`
7. Service deletes any remaining expired stories
8. Service logs deleted count
9. Prometheus metrics updated
10. Story no longer appears in feeds or queries

**Success Criteria**:
- Story deleted within ~60 seconds of expiration
- No manual intervention required
- Cleanup job provides redundancy and logging

---

## Security & Privacy

### Authentication

All endpoints require JWT authentication via `@UseGuards(JwtAuthGuard)`.

| Endpoint | Auth Required | Role Required |
|----------|---------------|---------------|
| `POST /stories` | ✅ Yes | User (any) |
| `GET /stories/me` | ✅ Yes | User (any) |
| `GET /stories/tray` | ✅ Yes | User (any) |
| `GET /stories/feed` | ✅ Yes | User (any) |
| `GET /stories/user/:username` | ✅ Yes | User (any) |
| `GET /stories/:id` | ✅ Yes | User (any) |
| `POST /stories/:id/view` | ✅ Yes | User (any) |

**JWT Payload:**
```typescript
{
  id: string;        // User UUID
  username: string;
  email: string;
  role: 'user' | 'chef' | 'admin';
}
```

### Privacy Enforcement

**3-Tier Privacy System:**

1. **Public Stories**
   - Accessible by all authenticated users
   - No follow relationship required
   - Useful for business/chef accounts promoting content

2. **Followers-Only Stories** (Default)
   - Viewer must follow creator with status='accepted'
   - Checked via `UserFollow` entity
   - Most common privacy setting

3. **Close Friends Stories**
   - Viewer must be in `closeFriends` array
   - Creator selects specific users
   - Array stored per-story (not global close friends list)

**Privacy Check Flow:**
```typescript
private async checkStoryPrivacy(story: StoryDocument, viewerId: string) {
  // Owner bypass
  if (story.userId === viewerId) return;
  
  // Public stories
  if (story.privacy === 'public') return;
  
  // Followers-only
  if (story.privacy === 'followers') {
    const follow = await this.followRepository.findOne({
      where: { followerId: viewerId, targetId: story.userId, status: 'accepted' }
    });
    if (!follow) throw new ForbiddenException();
  }
  
  // Close friends
  if (story.privacy === 'close-friends') {
    if (!story.closeFriends.includes(viewerId)) throw new ForbiddenException();
  }
}
```

### Authorization Rules

1. **Story Creation**: Any authenticated user can create stories
2. **Story Viewing**: Privacy rules enforced at service layer
3. **View Marking**: Only authenticated users can mark views
4. **Feed Access**: JWT required, follows relationships respected

### Data Protection

- **S3 URIs → HTTPS URLs**: `s3UriToHttps()` utility converts internal URIs to CDN URLs
- **Viewer Privacy**: Only first 100 viewers stored (prevents profile scraping)
- **No PII Leakage**: Story documents don't store sensitive user data
- **TTL Expiration**: Ephemeral by design - no long-term storage

---

## Performance & Scalability

### Database Performance

**MongoDB Indexes:**

```typescript
// 1. TTL Index (automatic expiration)
{ expiresAt: 1 }  // expireAfterSeconds: 0

// 2. User stories (newest first)
{ userId: 1, createdAt: -1 }

// 3. Privacy filtering
{ privacy: 1, createdAt: -1 }

// 4. Popularity sorting
{ views: -1 }
```

**Query Optimization:**
- **N+1 Prevention**: Batch user fetching with `In(uniqueUserIds)`
- **Projection**: Return only necessary fields in tray endpoint
- **Pagination**: Not yet implemented (TODO for large follower counts)

**Performance Targets:**
- Story creation: <100ms (excludes MediaConvert processing)
- Story feed: <200ms (up to 100 following users)
- Story tray: <100ms (lightweight, minimal data)
- Single story view: <50ms (indexed query)

### Caching Strategy

**Current State**: No caching implemented (TODO)

**Recommended Strategy:**
- Cache story tray in Valkey/Redis with 30-second TTL
- Cache key: `story:tray:{userId}`
- Invalidate on new story creation or story expiration
- Cache feed results for 60 seconds

**Implementation Example:**
```typescript
const cacheKey = `story:tray:${userId}`;
let tray = await this.cacheManager.get(cacheKey);

if (!tray) {
  tray = await this.getStoryTrayForUser(userId);
  await this.cacheManager.set(cacheKey, tray, 30); // 30s TTL
}
```

### Scalability Considerations

1. **Horizontal Scaling**: Stateless API design allows multiple NestJS instances
2. **MongoDB Sharding**: Shard by `userId` for large user bases
3. **CDN**: Serve media from CloudFront (low-latency global delivery)
4. **MediaConvert**: Auto-scales video processing jobs
5. **Viewer Cap**: 100-viewer limit prevents document bloat

---

## Analytics & Metrics

### Story-Level Metrics

| Metric | Description | Source Field |
|--------|-------------|--------------|
| **Views** | Total view count | `story.views` |
| **Unique Viewers** | Number of unique viewers | `story.viewers.length` |
| **Completion Rate** | % viewers who watched full story | TODO (requires client-side tracking) |
| **Shares** | TODO | Not implemented |
| **Replies** | TODO | Not implemented |
| **Exits** | Number of users who skipped story | TODO |

### User-Level Metrics

| Metric | Description | Query |
|--------|-------------|-------|
| **Stories Posted** | Total stories created by user | `db.stories.countDocuments({ userId })` |
| **Avg Views per Story** | Average views across all stories | `db.stories.aggregate([{ $match: { userId }}, { $group: { _id: null, avg: { $avg: '$views' }}}])` |
| **Story Frequency** | Stories posted per day | TODO (time-series analysis) |
| **Privacy Preferences** | Distribution of privacy settings | `db.stories.aggregate([{ $group: { _id: '$privacy', count: { $sum: 1 }}}])` |

### Business Metrics

1. **Daily Active Story Creators**: Users who posted ≥1 story in last 24h
2. **Story Engagement Rate**: (Total views / Total stories) × 100
3. **Retention Rate**: % users who post stories weekly

**MongoDB Aggregation Example:**
```javascript
db.stories.aggregate([
  {
    $match: {
      createdAt: { $gte: new Date(Date.now() - 24 * 60 * 60 * 1000) }
    }
  },
  {
    $group: {
      _id: '$userId',
      storyCount: { $sum: 1 },
      totalViews: { $sum: '$views' }
    }
  },
  {
    $group: {
      _id: null,
      uniqueCreators: { $sum: 1 },
      avgViewsPerStory: { $avg: '$totalViews' }
    }
  }
])
```

---

## Error Handling

### Error Codes

| Code | HTTP Status | Trigger | User-Facing Message |
|------|-------------|---------|---------------------|
| `STORY_NOT_FOUND` | 404 | Story ID doesn't exist or expired | "This story is no longer available" |
| `USER_NOT_FOUND` | 404 | Username doesn't exist in Postgres | "User not found" |
| `ACCESS_DENIED_PRIVACY` | 403 | Viewer doesn't meet privacy requirements | "You don't have permission to view this story" |
| `CLOSE_FRIENDS_REQUIRED` | 400 | privacy='close-friends' but closeFriends array empty | "Please select at least one close friend" |
| `INVALID_MEDIA_TYPE` | 400 | mediaType not 'image' or 'video' | "Invalid media type. Use 'image' or 'video'" |
| `MEDIA_FILE_REQUIRED` | 400 | No file uploaded in multipart request | "Please upload a photo or video" |
| `UNAUTHORIZED` | 401 | JWT missing or invalid | "Please log in to view stories" |

**Error Response Format:**
```json
{
  "success": false,
  "message": "You don't have permission to view this story",
  "errorCode": "ACCESS_DENIED_PRIVACY"
}
```

---

## Future Enhancements

### Planned Features

1. **Story Replies (Direct Messages)**
   - User taps reply button on story
   - Opens DM thread with story creator
   - Story context included in message

2. **Story Reactions**
   - Quick emoji reactions (❤️, 😂, 😮)
   - Stored in `reactions` subdocument array
   - Notifications sent to creator

3. **Story Highlights (Persistent Stories)**
   - Save stories to profile permanently
   - Group highlights by theme (e.g., "Events", "Products")
   - Stored separately from ephemeral stories

4. **Advanced Analytics**
   - Completion rate tracking (% of story watched)
   - Drop-off points for multi-story sequences
   - Audience demographics breakdown

5. **Story Ads (Chef Promotions)**
   - Sponsored stories in feed
   - Targeting based on location, interests
   - CPC/CPM billing model

6. **Story Mentions & Tags**
   - Tag users in stories (similar to reels)
   - Mentioned users receive notification
   - Click mention to view tagged user's profile

---

## Testing Strategy

### Unit Tests

**Service Layer** (`stories.service.spec.ts`):
- `createStory()`: Validates story creation with various privacy levels
- `getStoryTrayForUser()`: Verifies tray data structure and unseen prioritization
- `getStoryFeedForUser()`: Tests feed ordering and privacy filtering
- `checkStoryPrivacy()`: Ensures privacy rules enforced correctly
- `markStoryViewed()`: Validates view count increment and viewer cap

**Controller Layer** (`stories.controller.spec.ts`):
- Mocks `StoriesService` methods
- Validates request/response formats
- Tests JWT authentication guards

### Integration Tests

**E2E Tests** (`stories.e2e-spec.ts`):
- Full story creation flow (upload → save → retrieve)
- Feed generation with multiple users
- Privacy enforcement scenarios
- TTL expiration verification (wait 24h, verify deletion)

**Test Data:**
```typescript
const mockUser = {
  id: 'a1b2c3d4-e5f6-7890-1234-567890abcdef',
  username: 'john_chef',
  fullName: 'John Doe',
};

const mockStory = {
  userId: mockUser.id,
  mediaId: 'story-1709812345678',
  mediaType: 'image',
  mediaUrl: 's3://chefooz-stories/story.jpg',
  thumbnailUrl: 's3://chefooz-stories/thumb.jpg',
  durationMs: 5000,
  privacy: 'followers',
  closeFriends: [],
  views: 0,
  viewers: [],
};
```

### Performance Tests

**Load Testing:**
- 1000 concurrent users fetching story feed
- 500 stories/second creation rate
- 10,000 simultaneous story views

**Targets:**
- P95 response time <200ms for feed endpoint
- Story creation <100ms (excluding MediaConvert)
- MongoDB TTL deletion within 60s of expiration

---

## Deployment & Configuration

### Environment Variables

```bash
# MongoDB
MONGODB_URI=mongodb://localhost:27017/chefooz

# PostgreSQL
DATABASE_URL=postgresql://user:pass@localhost:5432/chefooz

# AWS MediaConvert
AWS_REGION=us-east-1
AWS_ACCESS_KEY_ID=AKIA...
AWS_SECRET_ACCESS_KEY=...
MEDIACONVERT_ENDPOINT=https://abc123.mediaconvert.us-east-1.amazonaws.com
MEDIACONVERT_ROLE_ARN=arn:aws:iam::123456789012:role/MediaConvertRole

# S3
S3_BUCKET_STORIES=chefooz-stories
S3_BUCKET_OUTPUT=chefooz-stories-output

# CDN
CDN_URL=https://cdn.chefooz.app

# JWT
JWT_SECRET=your_jwt_secret
JWT_EXPIRY=7d
```

### MongoDB TTL Index Setup

```javascript
// Ensure TTL index exists (auto-created by Mongoose schema)
db.stories.createIndex(
  { "expiresAt": 1 },
  { expireAfterSeconds: 0 }
);
```

**Verification:**
```javascript
db.stories.getIndexes();
// Should show:
// { "v": 2, "key": { "expiresAt": 1 }, "name": "expiresAt_1", "expireAfterSeconds": 0 }
```

### Cron Job Configuration

**Schedule**: Every 6 hours (0 */6 * * *)

**Purpose**: Redundant cleanup + logging (TTL index is primary deletion mechanism)

**Manual Trigger:**
```bash
# From admin panel or CLI
POST /api/v1/admin/stories/cleanup
```

---

## Related Documentation

- [Reels Module](../reels/FEATURE_OVERVIEW.md) - Long-form video content
- [Media Module](../media/FEATURE_OVERVIEW.md) - S3 upload + MediaConvert integration
- [Social Module](../social/FEATURE_OVERVIEW.md) - Follow relationships
- [User Module](../user/FEATURE_OVERVIEW.md) - User profiles and authentication

---

## Support & Contact

**Module Owner**: Content Team
**Slack Channel**: `#content-stories`
**On-Call Rotation**: PagerDuty "Content Services"

**Common Issues:**
- Stories not expiring → Check MongoDB TTL index status
- 403 errors on view → Verify follow relationships in Postgres
- Slow feed queries → Check MongoDB indexes with `.explain()`
- MediaConvert failures → Review AWS CloudWatch logs

---

**Last Updated**: 2026-02-14
**Version**: 1.0.0
**Status**: Production-Ready (MediaConvert integration pending)
