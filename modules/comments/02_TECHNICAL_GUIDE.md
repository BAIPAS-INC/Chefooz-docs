# Comments Module - Technical Guide

**Version:** 1.0  
**Last Updated:** February 14, 2026  
**Module:** `apps/chefooz-apis/src/modules/comments/`  
**Tech Stack:** NestJS, MongoDB (Mongoose), PostgreSQL (TypeORM)

---

## 📋 Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Database Schema](#database-schema)
3. [API Endpoints](#api-endpoints)
4. [Service Methods](#service-methods)
5. [Business Logic](#business-logic)
6. [Notification Integration](#notification-integration)
7. [Error Handling](#error-handling)
8. [Performance Optimization](#performance-optimization)
9. [Testing](#testing)

---

## Architecture Overview

### Module Structure

```
apps/chefooz-apis/src/modules/comments/
├── dto/
│   ├── create-comment.dto.ts       # Comment creation DTO
│   ├── create-reply.dto.ts         # Reply creation DTO
│   ├── list-comments.dto.ts        # Pagination params
│   ├── list-replies.dto.ts         # Reply pagination
│   ├── delete-comment.dto.ts       # Delete params
│   └── engagement.dto.ts           # Like, pin DTOs
├── comments.controller.ts          # HTTP endpoints
├── comments.service.ts             # Business logic
├── comments.module.ts              # Module configuration
└── comments.service.spec.ts        # Unit tests
```

### Dependencies

```typescript
@Module({
  imports: [
    MongooseModule.forFeature([
      { name: Comment.name, schema: CommentSchema },
      { name: Reel.name, schema: ReelSchema },
    ]),
    TypeOrmModule.forFeature([User]),
    NotificationModule,
  ],
  controllers: [CommentsController],
  providers: [CommentsService],
  exports: [CommentsService],
})
```

**Key Dependencies**:
- **Mongoose**: MongoDB ODM for comment storage
- **TypeORM**: PostgreSQL ORM for user data
- **NotificationModule**: Push/in-app notifications
- **JWT**: Authentication via JwtAuthGuard

---

## Database Schema

### Comment Collection (MongoDB)

```javascript
{
  _id: ObjectId("507f1f77bcf86cd799439011"),
  mediaId: ObjectId("507f1f77bcf86cd799439012"),  // Reel ID
  userId: "550e8400-e29b-41d4-a716-446655440000",  // PostgreSQL user.id
  text: "This recipe looks amazing!",
  parentId: null,  // null = top-level, ObjectId = reply
  replyCount: 5,
  isDeleted: false,
  likesCount: 12,
  likedBy: [
    "550e8400-e29b-41d4-a716-446655440001",
    "550e8400-e29b-41d4-a716-446655440002"
  ],
  isPinned: false,
  mentions: ["chef_rakesh"],  // Deprecated
  taggedUserIds: ["550e8400-e29b-41d4-a716-446655440003"],
  createdAt: ISODate("2026-02-14T10:30:00Z"),
  updatedAt: ISODate("2026-02-14T10:30:00Z")
}
```

### Schema Definition

```typescript
@Schema({ timestamps: true, collection: 'comments' })
export class Comment extends Document {
  @Prop({ required: true, type: Types.ObjectId, index: true })
  mediaId!: Types.ObjectId;

  @Prop({ required: true, index: true })
  userId!: string;

  @Prop({ required: true, maxlength: 500 })
  text!: string;

  @Prop({ type: Types.ObjectId, index: true, default: null })
  parentId?: Types.ObjectId | null;

  @Prop({ default: 0 })
  replyCount!: number;

  @Prop({ default: false })
  isDeleted!: boolean;

  @Prop({ default: 0 })
  likesCount!: number;

  @Prop({ type: [String], default: [] })
  likedBy!: string[];

  @Prop({ default: false })
  isPinned!: boolean;

  @Prop({ type: [String], default: [] })
  mentions!: string[];  // Deprecated

  @Prop({ type: [String], default: [] })
  taggedUserIds!: string[];

  createdAt!: Date;
  updatedAt!: Date;
}
```

### Indexes

```javascript
// Main query: List comments by reel
CommentSchema.index({ mediaId: 1, createdAt: -1 });

// Threading: List replies to comment
CommentSchema.index({ parentId: 1, createdAt: 1 });

// User history: User's comments
CommentSchema.index({ userId: 1, createdAt: -1 });
```

**Index Strategy**:
- **Compound index** on (mediaId, createdAt) for main list query
- **Ascending createdAt** for replies (chronological order)
- **Descending createdAt** for top-level (newest first)

---

## API Endpoints

### 1. Create Comment

**Endpoint**: `POST /api/v1/comments/create`

**Auth**: Required (JWT)

**Request Body**:
```typescript
{
  "mediaId": "507f1f77bcf86cd799439011",  // MongoDB ObjectId
  "text": "This looks delicious! @chef_rakesh",
  "parentId": null,  // Optional: for replies
  "taggedUserIds": ["550e8400-e29b-41d4-a716-446655440000"]  // Optional
}
```

**Response** (201 Created):
```json
{
  "success": true,
  "message": "Comment posted",
  "data": {
    "id": "507f1f77bcf86cd799439013",
    "mediaId": "507f1f77bcf86cd799439011",
    "userId": "550e8400-e29b-41d4-a716-446655440001",
    "username": "john_doe",
    "fullName": "John Doe",
    "avatarUrl": null,
    "text": "This looks delicious! @chef_rakesh",
    "parentId": null,
    "likesCount": 0,
    "isLiked": false,
    "isPinned": false,
    "mentions": ["chef_rakesh"],
    "createdAt": "2026-02-14T10:30:00.000Z",
    "isOwn": true
  }
}
```

**Error Responses**:
```json
// 404 - Reel not found
{
  "success": false,
  "message": "Reel with ID 507f1f77bcf86cd799439011 not found",
  "errorCode": "REEL_NOT_FOUND"
}

// 400 - Empty text
{
  "success": false,
  "message": "Comment text cannot be empty",
  "errorCode": "INVALID_INPUT"
}

// 400 - Cannot reply to reply
{
  "success": false,
  "message": "Cannot reply to a reply. Only 1-level threading supported.",
  "errorCode": "INVALID_NESTING"
}
```

---

### 2. List Comments

**Endpoint**: `GET /api/v1/comments/list`

**Auth**: Required (JWT)

**Query Parameters**:
```typescript
{
  mediaId: string;      // Required: Reel ID
  cursor?: string;      // Optional: Pagination cursor (ObjectId)
  limit?: number;       // Optional: Default 20, max 50
}
```

**Request**:
```
GET /v1/comments/list?mediaId=507f1f77bcf86cd799439011&limit=20
Authorization: Bearer <token>
```

**Response** (200 OK):
```json
{
  "success": true,
  "message": "Comments retrieved",
  "data": {
    "items": [
      {
        "id": "507f1f77bcf86cd799439013",
        "mediaId": "507f1f77bcf86cd799439011",
        "userId": "550e8400-e29b-41d4-a716-446655440001",
        "username": "john_doe",
        "fullName": "John Doe",
        "avatarUrl": null,
        "text": "Amazing recipe!",
        "parentId": null,
        "likesCount": 5,
        "isLiked": false,
        "isPinned": true,
        "mentions": [],
        "createdAt": "2026-02-14T10:30:00.000Z",
        "isOwn": false,
        "replies": [
          {
            "id": "507f1f77bcf86cd799439014",
            "text": "Thanks!",
            "username": "chef_rakesh",
            "createdAt": "2026-02-14T10:35:00.000Z"
          }
        ],
        "repliesCount": 3
      }
    ],
    "nextCursor": "507f1f77bcf86cd799439015"
  }
}
```

**Sorting**:
- Pinned comments first (`isPinned: true`)
- Then by `createdAt` descending (newest first)

**Reply Preview**:
- Shows first 3 replies per comment
- `repliesCount` indicates total replies

---

### 3. Create Reply

**Endpoint**: `POST /api/v1/comments/:commentId/replies`

**Auth**: Required (JWT)

**Path Parameters**:
- `commentId`: MongoDB ObjectId of parent comment

**Request Body**:
```typescript
{
  "mediaId": "507f1f77bcf86cd799439011",  // Must match parent's reel
  "text": "Great question!",
  "taggedUserIds": []  // Optional
}
```

**Response** (201 Created):
```json
{
  "success": true,
  "message": "Reply posted",
  "data": {
    "id": "507f1f77bcf86cd799439016",
    "mediaId": "507f1f77bcf86cd799439011",
    "userId": "550e8400-e29b-41d4-a716-446655440002",
    "username": "chef_rakesh",
    "fullName": "Chef Rakesh",
    "avatarUrl": null,
    "text": "Great question!",
    "parentId": "507f1f77bcf86cd799439013",
    "likesCount": 0,
    "isLiked": false,
    "isPinned": false,
    "mentions": [],
    "createdAt": "2026-02-14T10:35:00.000Z",
    "isOwn": true
  }
}
```

**Side Effects**:
- Increments `parent.replyCount` by 1
- Sends notification to parent comment owner
- Sends mention notifications if tagged users

---

### 4. List Replies

**Endpoint**: `GET /api/v1/comments/:commentId/replies`

**Auth**: Required (JWT)

**Query Parameters**:
```typescript
{
  cursor?: string;   // ISO timestamp
  limit?: number;    // Default 20
}
```

**Request**:
```
GET /v1/comments/507f1f77bcf86cd799439013/replies?limit=20
Authorization: Bearer <token>
```

**Response** (200 OK):
```json
{
  "success": true,
  "message": "Replies retrieved",
  "data": {
    "items": [
      {
        "id": "507f1f77bcf86cd799439016",
        "text": "Great question!",
        "username": "chef_rakesh",
        "createdAt": "2026-02-14T10:35:00.000Z",
        "likesCount": 2,
        "isLiked": false,
        "isOwn": false
      }
    ],
    "nextCursor": "2026-02-14T10:40:00.000Z"
  }
}
```

**Sorting**: Chronological (oldest first, ASC)

---

### 5. Delete Comment

**Endpoint**: `DELETE /api/v1/comments/delete/:commentId`

**Auth**: Required (JWT)

**Path Parameters**:
- `commentId`: MongoDB ObjectId

**Response** (200 OK):
```json
{
  "success": true,
  "message": "Comment deleted",
  "data": null
}
```

**Error Responses**:
```json
// 403 - Not owner
{
  "success": false,
  "message": "You can only delete your own comments",
  "errorCode": "FORBIDDEN"
}

// 404 - Comment not found
{
  "success": false,
  "message": "Comment with ID 507f1f77bcf86cd799439013 not found",
  "errorCode": "COMMENT_NOT_FOUND"
}
```

**Side Effects**:
- Sets `isDeleted: true` (soft delete)
- Decrements `reel.stats.comments` (if top-level)
- Does NOT decrement `parent.replyCount` (preserves count)

---

### 6. Like Comment

**Endpoint**: `POST /api/v1/comments/:commentId/like`

**Auth**: Required (JWT)

**Response** (200 OK):
```json
{
  "success": true,
  "message": "Comment liked",
  "data": {
    "commentId": "507f1f77bcf86cd799439013",
    "likesCount": 6,
    "isLiked": true
  }
}
```

**Idempotency**: If already liked, returns current state (no error)

**Side Effects**:
- Adds userId to `likedBy` array
- Increments `likesCount`
- Sends notification to comment owner (if not self-like)

---

### 7. Unlike Comment

**Endpoint**: `DELETE /api/v1/comments/:commentId/like`

**Auth**: Required (JWT)

**Response** (200 OK):
```json
{
  "success": true,
  "message": "Comment unliked",
  "data": {
    "commentId": "507f1f77bcf86cd799439013",
    "likesCount": 5,
    "isLiked": false
  }
}
```

**Idempotency**: If not liked, returns current state (no error)

**Side Effects**:
- Removes userId from `likedBy` array
- Decrements `likesCount`
- No notification sent

---

### 8. Pin Comment

**Endpoint**: `POST /api/v1/comments/:commentId/pin`

**Auth**: Required (JWT - must be reel creator)

**Request Body**:
```typescript
{
  "mediaId": "507f1f77bcf86cd799439011"  // Reel ID for verification
}
```

**Response** (200 OK):
```json
{
  "success": true,
  "message": "Comment pinned",
  "data": {
    "commentId": "507f1f77bcf86cd799439013",
    "isPinned": true
  }
}
```

**Error Responses**:
```json
// 403 - Not reel creator
{
  "success": false,
  "message": "Only reel creator can pin comments",
  "errorCode": "FORBIDDEN"
}

// 400 - Cannot pin reply
{
  "success": false,
  "message": "Cannot pin a reply, only top-level comments",
  "errorCode": "INVALID_ACTION"
}
```

**Side Effects**:
- Unpins any existing pinned comment on reel
- Sets `isPinned: true` on target comment

---

### 9. Unpin Comment

**Endpoint**: `DELETE /api/v1/comments/pin`

**Auth**: Required (JWT - must be reel creator)

**Request Body**:
```typescript
{
  "mediaId": "507f1f77bcf86cd799439011"
}
```

**Response** (200 OK):
```json
{
  "success": true,
  "message": "Comment unpinned",
  "data": null
}
```

**Side Effects**:
- Sets `isPinned: false` on all comments for reel

---

## Service Methods

### CommentsService Core Methods

#### createComment()

```typescript
async createComment(
  userId: string, 
  dto: CreateCommentDto
): Promise<CommentWithUser>
```

**Flow**:
1. Fetch user info from PostgreSQL
2. Verify reel exists in MongoDB
3. If `parentId`: Verify parent exists, validate same reel, check not reply-to-reply
4. Sanitize text (trim, check not empty)
5. Parse @mentions from text
6. Create comment document
7. If top-level: Increment `reel.stats.comments`
8. Send notification to reel owner
9. Send mention notifications (async)
10. Return formatted comment

**Error Handling**:
- `NotFoundException`: Reel or parent comment not found
- `BadRequestException`: Empty text, nested reply, wrong reel

**Example**:
```typescript
const comment = await commentsService.createComment(
  "550e8400-e29b-41d4-a716-446655440001",
  {
    mediaId: "507f1f77bcf86cd799439011",
    text: "Amazing dish! @chef_rakesh",
    parentId: null,
    taggedUserIds: ["550e8400-e29b-41d4-a716-446655440002"]
  }
);
```

---

#### listComments()

```typescript
async listComments(
  dto: ListCommentsDto, 
  viewerId?: string
): Promise<CommentsListResponse>
```

**Query**:
```typescript
const filter = {
  mediaId: new Types.ObjectId(dto.mediaId),
  parentId: null,  // Only top-level
  isDeleted: false,
};

if (dto.cursor) {
  filter._id = { $lt: new Types.ObjectId(dto.cursor) };
}

const comments = await commentModel
  .find(filter)
  .sort({ isPinned: -1, createdAt: -1 })  // Pinned first, then newest
  .limit(limit + 1)
  .exec();
```

**User Enrichment**:
```typescript
// Batch fetch users
const userIds = [...new Set(comments.map(c => c.userId))];
const users = await userRepository.find({
  where: { id: In(userIds) },
  select: ['id', 'username', 'fullName'],
});
const userMap = new Map(users.map(u => [u.id, u]));
```

**Reply Preview**:
```typescript
for (const comment of items) {
  const replies = await getRepliesPreview(comment.id, viewerId);
  comment.replies = replies;  // First 3 replies
  comment.repliesCount = commentDoc.replyCount;
}
```

**Pagination**:
- Cursor-based using `_id`
- Fetch `limit + 1` items
- If `length > limit`, has more pages
- Return last item's `_id` as `nextCursor`

---

#### createReply()

```typescript
async createReply(
  userId: string, 
  commentId: string, 
  dto: CreateReplyDto
): Promise<CommentWithUser>
```

**Validation**:
```typescript
// Verify parent exists
const parentComment = await commentModel.findById(commentId).exec();
if (!parentComment) {
  throw new NotFoundException('Parent comment not found');
}

// Verify same reel
if (parentComment.mediaId.toString() !== dto.mediaId) {
  throw new BadRequestException('Parent must belong to same reel');
}

// Prevent nested replies
if (parentComment.parentId) {
  throw new BadRequestException('Cannot reply to a reply');
}
```

**Reply Creation**:
```typescript
const reply = new commentModel({
  mediaId: new Types.ObjectId(dto.mediaId),
  userId,
  text: sanitizedText,
  parentId: new Types.ObjectId(commentId),
  taggedUserIds: dto.taggedUserIds?.slice(0, 10) || [],
});

await reply.save();

// Increment parent's replyCount
await commentModel.findByIdAndUpdate(
  commentId,
  { $inc: { replyCount: 1 } },
  { new: true }
).exec();
```

---

#### deleteComment()

```typescript
async deleteComment(
  userId: string, 
  commentId: string
): Promise<void>
```

**Authorization**:
```typescript
const comment = await commentModel.findById(commentId).exec();
if (!comment) {
  throw new NotFoundException('Comment not found');
}

if (comment.userId !== userId) {
  throw new ForbiddenException('Can only delete own comments');
}
```

**Soft Delete**:
```typescript
comment.isDeleted = true;
await comment.save();

// Decrement reel count (top-level only)
if (!comment.parentId) {
  await reelModel.findByIdAndUpdate(
    comment.mediaId,
    { $inc: { 'stats.comments': -1 } }
  ).exec();
}
```

**Why Soft Delete?**
- Preserves reply threading
- Audit trail for moderation
- Potential undelete feature

---

#### likeComment()

```typescript
async likeComment(
  userId: string, 
  commentId: string
): Promise<CommentWithUser>
```

**Logic**:
```typescript
const comment = await commentModel.findById(commentId).exec();

// Check if already liked
if (comment.likedBy.includes(userId)) {
  return mapCommentToResponse(comment, user, userId);  // Idempotent
}

// Add like
comment.likedBy.push(userId);
comment.likesCount += 1;
await comment.save();

// Send notification (if not self-like)
if (comment.userId !== userId) {
  await notificationDispatcher.send(comment.userId, 'engagement.like', {
    username: liker.username,
    type: 'comment',
    commentId,
    mediaId: comment.mediaId.toString(),
  });
}
```

---

#### pinComment()

```typescript
async pinComment(
  userId: string, 
  commentId: string, 
  mediaId: string
): Promise<CommentWithUser>
```

**Authorization**:
```typescript
const reel = await reelModel.findById(mediaId).exec();
if (reel.userId !== userId) {
  throw new ForbiddenException('Only reel creator can pin');
}
```

**Pin Logic**:
```typescript
// Unpin existing
await commentModel.updateMany(
  { mediaId: new Types.ObjectId(mediaId), isPinned: true },
  { isPinned: false }
).exec();

// Pin new
comment.isPinned = true;
await comment.save();
```

**Validation**:
- Must be top-level comment (no `parentId`)
- Must belong to specified reel
- Only 1 pinned comment per reel

---

## Business Logic

### 1-Level Threading Enforcement

```typescript
// When creating reply
if (dto.parentId) {
  const parentComment = await commentModel.findById(dto.parentId).exec();
  
  // Check parent is not itself a reply
  if (parentComment.parentId) {
    throw new BadRequestException(
      'Cannot reply to a reply. Only 1-level threading supported.'
    );
  }
}
```

**Why?**
- Simplifies UI/UX
- Better mobile experience
- Faster queries (no recursion)
- Instagram-like behavior

---

### Mention Parsing & Notifications

```typescript
// Parse @mentions from text
private parseMentions(text: string): string[] {
  const mentionRegex = /@([a-zA-Z0-9_]+)/g;
  const mentions: string[] = [];
  let match;
  
  while ((match = mentionRegex.exec(text)) !== null) {
    mentions.push(match[1]);
  }
  
  return [...new Set(mentions)];  // Deduplicate
}

// Send notifications (async)
private async sendMentionNotifications(
  actorUserId: string,
  commentId: string,
  taggedUserIds: string[],
  actorUsername: string
): Promise<void> {
  // Filter self-tags and duplicates
  const uniqueRecipients = Array.from(new Set(taggedUserIds))
    .filter(id => id !== actorUserId)
    .slice(0, 10);

  for (const recipientId of uniqueRecipients) {
    await notificationOrchestrator.handleEvent({
      type: 'REEL_COMMENTED',
      recipientUserId: recipientId,
      actorUserId,
      entityId: commentId,
      metadata: {
        type: 'mention',
        commentId,
        username: actorUsername,
        mentionType: 'comment',
      },
    });
  }
}
```

**Key Features**:
- Max 10 mentions per comment
- Self-tags filtered out
- Async execution (non-blocking)
- Deduplicated

---

### Denormalized Counts

**Why Denormalize?**
- Avoid expensive `COUNT(*)` queries
- Instant access to counts
- Better performance for lists

**Counters**:
```typescript
// Comment likes count
comment.likesCount = comment.likedBy.length;  // Could use this
// But denormalized for performance:
comment.likesCount = 12;  // Updated on like/unlike

// Reply count
comment.replyCount = 5;  // Updated on reply create/delete
```

**Update Strategy**:
```typescript
// On like
comment.likesCount += 1;

// On unlike
comment.likesCount = Math.max(0, comment.likesCount - 1);

// On reply
await commentModel.findByIdAndUpdate(
  parentId,
  { $inc: { replyCount: 1 } }
);
```

---

## Notification Integration

### Comment Notification

```typescript
await notificationOrchestrator.handleEvent({
  type: 'REEL_COMMENTED',
  recipientUserId: reel.userId,
  actorUserId: userId,
  entityId: mediaId,
  metadata: {
    reelId: mediaId,
    commentId: comment._id.toString(),
    username: user.username,
    commentText: truncatedText,
  },
});
```

**Push Notification**:
```
Title: "New Comment"
Body: "john_doe: This recipe looks amazing!"
Action: Opens reel with comment section
```

---

### Reply Notification

```typescript
await notificationOrchestrator.handleEvent({
  type: 'COMMENT_REPLIED',
  recipientUserId: parentComment.userId,
  actorUserId: userId,
  entityId: mediaId,
  metadata: {
    reelId: mediaId,
    commentId: commentId,
    replyId: reply._id.toString(),
    username: user.username,
    replyText: truncatedText,
  },
});
```

**Push Notification**:
```
Title: "New Reply"
Body: "chef_rakesh replied: Thanks for your comment!"
Action: Opens comment thread
```

---

### Mention Notification

```typescript
await notificationOrchestrator.handleEvent({
  type: 'REEL_COMMENTED',
  recipientUserId: taggedUserId,
  actorUserId: userId,
  entityId: commentId,
  metadata: {
    type: 'mention',
    commentId,
    username: actorUsername,
    mentionType: 'comment',
  },
});
```

**Push Notification**:
```
Title: "New Mention"
Body: "john_doe mentioned you in a comment"
Action: Opens reel with comment highlighted
```

---

### Like Notification

```typescript
await notificationDispatcher.send(
  comment.userId,
  'engagement.like',
  {
    username: liker.username,
    type: 'comment',
    commentId,
    mediaId: comment.mediaId.toString(),
  }
);
```

**Push Notification**:
```
Title: "Comment Liked"
Body: "jane_smith liked your comment"
Action: Opens comment
```

---

## Error Handling

### Standard Error Responses

```typescript
// 400 Bad Request
{
  "success": false,
  "message": "Comment text cannot be empty",
  "errorCode": "INVALID_INPUT"
}

// 403 Forbidden
{
  "success": false,
  "message": "You can only delete your own comments",
  "errorCode": "FORBIDDEN"
}

// 404 Not Found
{
  "success": false,
  "message": "Comment with ID 507f... not found",
  "errorCode": "COMMENT_NOT_FOUND"
}
```

### Error Handling Patterns

```typescript
try {
  const comment = await commentModel.findById(commentId).exec();
  if (!comment) {
    throw new NotFoundException(`Comment with ID ${commentId} not found`);
  }
  
  if (comment.userId !== userId) {
    throw new ForbiddenException('Can only delete own comments');
  }
  
  // ... business logic
} catch (error) {
  logger.error(`Failed to delete comment: ${error.message}`, error.stack);
  throw error;
}
```

### Graceful Degradation

```typescript
// User lookup failure handling
const user = await userRepository.findOne({ where: { id: userId } });

if (!user) {
  logger.warn(`User ${userId} not found. Showing as 'Deleted User'`);
  // Don't block comment creation
}

const username = user?.username || 'deleted_user';
const fullName = user?.fullName || 'Deleted User';
```

---

## Performance Optimization

### Database Indexes

```javascript
// Compound indexes for common queries
{ mediaId: 1, createdAt: -1 }  // List comments DESC
{ parentId: 1, createdAt: 1 }  // List replies ASC
{ userId: 1, createdAt: -1 }   // User comments
```

### Batch User Lookups

```typescript
// Avoid N+1 queries
const userIds = [...new Set(comments.map(c => c.userId))];
const users = await userRepository.find({
  where: { id: In(userIds) },  // Single query
  select: ['id', 'username', 'fullName'],
});
```

### Cursor-Based Pagination

```typescript
// Efficient pagination (no OFFSET)
const filter: any = {
  mediaId: new Types.ObjectId(dto.mediaId),
  isDeleted: false,
};

if (dto.cursor) {
  filter._id = { $lt: new Types.ObjectId(dto.cursor) };
}

const comments = await commentModel
  .find(filter)
  .limit(limit + 1)
  .exec();
```

### Denormalized Counts

```typescript
// No COUNT(*) queries
comment.likesCount = 12;  // Stored value
comment.replyCount = 5;   // Stored value

// vs expensive query:
const likesCount = await commentLikeModel.count({ commentId });  // ❌ Slow
```

### Async Notifications

```typescript
// Don't block comment creation
await comment.save();

// Send notifications asynchronously
this.sendMentionNotifications(...).catch(err => {
  logger.error('Failed to send notifications', err);
});

return comment;  // Return immediately
```

---

## Testing

### Unit Tests

```typescript
describe('CommentsService', () => {
  describe('createComment', () => {
    it('should create comment with mentions', async () => {
      const dto = {
        mediaId: 'reel-id',
        text: 'Great! @chef_rakesh',
        taggedUserIds: ['user-1'],
      };
      
      const comment = await service.createComment('user-2', dto);
      
      expect(comment.text).toBe('Great! @chef_rakesh');
      expect(comment.mentions).toContain('chef_rakesh');
      expect(notificationSpy).toHaveBeenCalledWith('user-1', ...);
    });
    
    it('should prevent nested replies', async () => {
      const reply = await service.createComment('user-1', {
        mediaId: 'reel-id',
        text: 'Reply',
        parentId: 'parent-comment-id',
      });
      
      await expect(
        service.createComment('user-2', {
          mediaId: 'reel-id',
          text: 'Nested reply',
          parentId: reply.id,
        })
      ).rejects.toThrow('Cannot reply to a reply');
    });
  });
  
  describe('likeComment', () => {
    it('should be idempotent', async () => {
      const comment1 = await service.likeComment('user-1', 'comment-id');
      const comment2 = await service.likeComment('user-1', 'comment-id');
      
      expect(comment1.likesCount).toBe(1);
      expect(comment2.likesCount).toBe(1);  // Same count
    });
  });
});
```

### Integration Tests

```typescript
describe('CommentsController (e2e)', () => {
  it('POST /comments/create should create comment', async () => {
    const response = await request(app.getHttpServer())
      .post('/v1/comments/create')
      .set('Authorization', `Bearer ${userToken}`)
      .send({
        mediaId: reelId,
        text: 'Test comment',
      })
      .expect(201);
    
    expect(response.body.success).toBe(true);
    expect(response.body.data.text).toBe('Test comment');
  });
  
  it('should enforce 1-level threading', async () => {
    // Create comment
    const comment = await createComment(reelId, 'Top level');
    
    // Create reply
    const reply = await createReply(comment.id, 'Reply');
    
    // Try nested reply
    await request(app.getHttpServer())
      .post(`/v1/comments/${reply.id}/replies`)
      .set('Authorization', `Bearer ${userToken}`)
      .send({
        mediaId: reelId,
        text: 'Nested reply',
      })
      .expect(400);
  });
});
```

---

## Related Documentation

- **Feature Overview**: `01_FEATURE_OVERVIEW.md` (Business context)
- **QA Test Cases**: `03_QA_TEST_CASES.md` (Test scenarios)
- **Reels Module**: `../reels/02_TECHNICAL_GUIDE.md` (Comment target)
- **Notification Module**: `../notification/02_TECHNICAL_GUIDE.md` (Notifications)

---

**[TECHNICAL_COMPLETE ✅]**  
**Module**: Comments  
**Documentation**: Technical Guide  
**Date**: February 14, 2026
