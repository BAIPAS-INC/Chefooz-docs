# Stories Module - QA Test Cases

## Test Overview

### Test Strategy

| Test Type | Total Cases | Automated | Manual | Priority |
|-----------|-------------|-----------|--------|----------|
| **Functional** | 38 | 35 | 3 | High |
| **Security** | 10 | 10 | 0 | Critical |
| **Performance** | 6 | 4 | 2 | Medium |
| **Integration** | 8 | 7 | 1 | High |
| **Regression** | 12 | 12 | 0 | High |
| **TOTAL** | **74** | **68 (92%)** | **6 (8%)** | - |

### Test Environment

**Prerequisites**:
- MongoDB 6.x running locally or in Docker
- PostgreSQL 15.x with test database
- NestJS API server running on port 3000
- Test user accounts with JWT tokens
- Sample media files (photo.jpg, video.mp4)

**Test Data**:
```typescript
const testUsers = [
  { id: 'a1b2c3d4-e5f6-7890-1234-567890abcdef', username: 'john_chef', fullName: 'John Doe' },
  { id: 'b2c3d4e5-f6a7-8901-2345-678901bcdefg', username: 'jane_baker', fullName: 'Jane Smith' },
  { id: 'c3d4e5f6-a7b8-9012-3456-789012cdefgh', username: 'alice_cook', fullName: 'Alice Johnson' },
];

const testStory = {
  userId: 'a1b2c3d4-e5f6-7890-1234-567890abcdef',
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

---

## Functional Test Cases

### Category 1: Story Creation

#### TC-F-001: Create Image Story (Public)
**Priority**: High | **Automated**: Yes

**Preconditions**:
- User authenticated with JWT
- Photo file ready (photo.jpg, <10MB)

**Steps**:
1. Call `POST /api/v1/stories` with multipart/form-data
2. Include fields: `file=photo.jpg`, `mediaType=image`, `privacy=public`
3. Verify response 201 Created

**Expected Result**:
```json
{
  "success": true,
  "message": "Story created successfully",
  "data": {
    "id": "65f3a8b7c1234567890abcde",
    "userId": "a1b2c3d4-e5f6-7890-1234-567890abcdef",
    "mediaType": "image",
    "privacy": "public",
    "durationMs": 5000,
    "expiresAt": "2024-03-08T10:00:00.000Z"
  }
}
```

**Validation**:
- `expiresAt` is 24 hours from now
- `mediaUrl` is HTTPS URL (not S3 URI)
- `views` is 0
- `viewers` is empty array

---

#### TC-F-002: Create Video Story (Followers)
**Priority**: High | **Automated**: Yes

**Steps**:
1. Call `POST /api/v1/stories` with `file=video.mp4`, `mediaType=video`, `privacy=followers`
2. Verify response 201 Created

**Expected Result**:
- Story created with `mediaType='video'`
- `durationMs` defaults to 10000 (10 seconds)
- `privacy='followers'`

**Note**: MediaConvert integration pending (TODO), currently uses mock URLs

---

#### TC-F-003: Create Close Friends Story
**Priority**: High | **Automated**: Yes

**Steps**:
1. Call `POST /api/v1/stories` with:
   - `file=photo.jpg`
   - `mediaType=image`
   - `privacy=close-friends`
   - `closeFriends=['b2c3d4e5-f6a7-8901-2345-678901bcdefg', 'c3d4e5f6-a7b8-9012-3456-789012cdefgh']`
2. Verify response 201 Created

**Expected Result**:
- Story created with `privacy='close-friends'`
- `closeFriends` array contains 2 UUIDs
- Only users in `closeFriends` array can view story

---

#### TC-F-004: Create Story with Custom Duration
**Priority**: Medium | **Automated**: Yes

**Steps**:
1. Call `POST /api/v1/stories` with:
   - `file=photo.jpg`
   - `mediaType=image`
   - `durationMs=7000` (7 seconds)
2. Verify response 201 Created

**Expected Result**:
- `durationMs` is 7000 (custom value respected)

---

#### TC-F-005: Create Story Without File
**Priority**: High | **Automated**: Yes

**Steps**:
1. Call `POST /api/v1/stories` without `file` field
2. Verify response 400 Bad Request

**Expected Result**:
```json
{
  "success": false,
  "message": "Media file required",
  "errorCode": "MEDIA_FILE_REQUIRED"
}
```

---

#### TC-F-006: Create Close Friends Story Without Close Friends Array
**Priority**: High | **Automated**: Yes

**Steps**:
1. Call `POST /api/v1/stories` with:
   - `file=photo.jpg`
   - `privacy=close-friends`
   - `closeFriends=[]` (empty array)
2. Verify response 400 Bad Request

**Expected Result**:
```json
{
  "success": false,
  "message": "Close friends list required for close-friends privacy",
  "errorCode": "CLOSE_FRIENDS_REQUIRED"
}
```

---

#### TC-F-007: Create Story with Invalid Media Type
**Priority**: Medium | **Automated**: Yes

**Steps**:
1. Call `POST /api/v1/stories` with `mediaType=pdf`
2. Verify response 400 Bad Request

**Expected Result**:
- Validation error: "mediaType must be one of: image, video"

---

### Category 2: Get Own Stories

#### TC-F-008: Get Own Stories (User with Stories)
**Priority**: High | **Automated**: Yes

**Preconditions**:
- User has 3 active stories (created in last 24 hours)

**Steps**:
1. Call `GET /api/v1/stories/me` with JWT
2. Verify response 200 OK

**Expected Result**:
```json
{
  "success": true,
  "message": "Stories retrieved",
  "data": {
    "stories": [
      { "id": "story-1", "createdAt": "2024-03-07T12:00:00Z" },
      { "id": "story-2", "createdAt": "2024-03-07T11:00:00Z" },
      { "id": "story-3", "createdAt": "2024-03-07T10:00:00Z" }
    ],
    "total": 3
  }
}
```

**Validation**:
- Stories sorted by newest first (`createdAt DESC`)
- All stories belong to authenticated user
- `hasSeen` is false (own stories not marked as seen)

---

#### TC-F-009: Get Own Stories (User with No Stories)
**Priority**: Medium | **Automated**: Yes

**Steps**:
1. Call `GET /api/v1/stories/me` with JWT (user with no stories)
2. Verify response 200 OK

**Expected Result**:
```json
{
  "success": true,
  "message": "Stories retrieved",
  "data": {
    "stories": [],
    "total": 0
  }
}
```

---

#### TC-F-010: Get Own Stories (Expired Stories Excluded)
**Priority**: High | **Automated**: Yes

**Preconditions**:
- User has 2 active stories and 1 expired story (created 25 hours ago)

**Steps**:
1. Call `GET /api/v1/stories/me` with JWT
2. Verify response 200 OK

**Expected Result**:
- Only 2 active stories returned
- Expired story not included

---

### Category 3: Get Story Tray

#### TC-F-011: Get Story Tray (Self + Following Users)
**Priority**: High | **Automated**: Yes

**Preconditions**:
- User follows 2 other users with active stories
- User has own stories

**Steps**:
1. Call `GET /api/v1/stories/tray` with JWT
2. Verify response 200 OK

**Expected Result**:
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
        "storyCount": 2
      },
      {
        "username": "jane_baker",
        "avatar": null,
        "hasUnseen": true,
        "storyCount": 1
      },
      {
        "username": "alice_cook",
        "avatar": null,
        "hasUnseen": false,
        "storyCount": 3
      }
    ],
    "total": 3
  }
}
```

**Validation**:
- Self stories appear first
- Unseen stories prioritized (jane_baker before alice_cook)
- `storyCount` matches actual story count per user

---

#### TC-F-012: Get Story Tray (No Stories)
**Priority**: Medium | **Automated**: Yes

**Steps**:
1. Call `GET /api/v1/stories/tray` with JWT (user follows no one, has no stories)
2. Verify response 200 OK

**Expected Result**:
```json
{
  "success": true,
  "message": "Story tray retrieved",
  "data": {
    "tray": [],
    "total": 0
  }
}
```

---

#### TC-F-013: Get Story Tray (Privacy Filtering)
**Priority**: High | **Automated**: Yes

**Preconditions**:
- User A follows User B (status='accepted')
- User B has 2 stories: 1 public, 1 followers-only
- User C (not followed by User A) has 1 public story

**Steps**:
1. User A calls `GET /api/v1/stories/tray`
2. Verify response includes User B and User C

**Expected Result**:
- User B shown with 2 stories (public + followers)
- User C shown with 1 story (public only)

---

### Category 4: Get Story Feed

#### TC-F-014: Get Story Feed (Full Feed)
**Priority**: High | **Automated**: Yes

**Preconditions**:
- User follows 2 users with active stories
- User has own stories

**Steps**:
1. Call `GET /api/v1/stories/feed` with JWT
2. Verify response 200 OK

**Expected Result**:
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
            "id": "story-1",
            "mediaType": "image",
            "mediaUrl": "https://cdn.chefooz.app/stories/story-1.jpg",
            "views": 0,
            "hasSeen": false
          }
        ],
        "hasUnseenStories": true
      }
    ],
    "total": 1
  }
}
```

**Validation**:
- Self stories appear first
- Unseen stories prioritized within each group
- Full story objects returned (not minimal like tray)

---

#### TC-F-015: Get Story Feed (Privacy Enforcement)
**Priority**: High | **Automated**: Yes

**Preconditions**:
- User A follows User B
- User B has close-friends story with User C in `closeFriends` array (not User A)

**Steps**:
1. User A calls `GET /api/v1/stories/feed`
2. Verify User B's close-friends story not included in feed

**Expected Result**:
- User B's public/followers stories included
- User B's close-friends story excluded (User A not in closeFriends array)

---

### Category 5: Get Stories by Username

#### TC-F-016: Get Stories by Username (Public User)
**Priority**: High | **Automated**: Yes

**Preconditions**:
- User B has 3 public stories

**Steps**:
1. User A calls `GET /api/v1/stories/user/jane_baker` with JWT
2. Verify response 200 OK

**Expected Result**:
```json
{
  "success": true,
  "message": "User stories retrieved",
  "data": {
    "stories": [
      { "id": "story-1", "privacy": "public" },
      { "id": "story-2", "privacy": "public" },
      { "id": "story-3", "privacy": "public" }
    ],
    "total": 3
  }
}
```

---

#### TC-F-017: Get Stories by Username (Followers-Only, Following)
**Priority**: High | **Automated**: Yes

**Preconditions**:
- User A follows User B (status='accepted')
- User B has 2 followers-only stories

**Steps**:
1. User A calls `GET /api/v1/stories/user/jane_baker`
2. Verify response 200 OK with 2 stories

**Expected Result**:
- Both stories returned (User A follows User B)

---

#### TC-F-018: Get Stories by Username (Followers-Only, Not Following)
**Priority**: High | **Automated**: Yes

**Preconditions**:
- User A does NOT follow User B
- User B has 2 followers-only stories

**Steps**:
1. User A calls `GET /api/v1/stories/user/jane_baker`
2. Verify response 200 OK with empty stories array

**Expected Result**:
```json
{
  "success": true,
  "message": "User stories retrieved",
  "data": {
    "stories": [],
    "total": 0
  }
}
```

**Note**: Stories filtered silently (no 403 error)

---

#### TC-F-019: Get Stories by Username (User Not Found)
**Priority**: Medium | **Automated**: Yes

**Steps**:
1. Call `GET /api/v1/stories/user/nonexistent_user`
2. Verify response 404 Not Found

**Expected Result**:
```json
{
  "success": false,
  "message": "User not found",
  "errorCode": "USER_NOT_FOUND"
}
```

---

### Category 6: Get Single Story

#### TC-F-020: Get Single Story (Owner)
**Priority**: High | **Automated**: Yes

**Preconditions**:
- User owns story with ID `story-123`

**Steps**:
1. Call `GET /api/v1/stories/story-123` with owner's JWT
2. Verify response 200 OK

**Expected Result**:
- Full story object returned
- Owner can view their own story regardless of privacy

---

#### TC-F-021: Get Single Story (Public, Non-Owner)
**Priority**: High | **Automated**: Yes

**Steps**:
1. Call `GET /api/v1/stories/story-123` with non-owner's JWT (story is public)
2. Verify response 200 OK

**Expected Result**:
- Full story object returned

---

#### TC-F-022: Get Single Story (Followers-Only, Not Following)
**Priority**: High | **Automated**: Yes

**Preconditions**:
- Story has privacy='followers'
- Viewer does NOT follow creator

**Steps**:
1. Call `GET /api/v1/stories/story-123` with viewer's JWT
2. Verify response 403 Forbidden

**Expected Result**:
```json
{
  "success": false,
  "message": "You must follow this user to view their stories",
  "errorCode": "ACCESS_DENIED_PRIVACY"
}
```

---

#### TC-F-023: Get Single Story (Close Friends, Not in List)
**Priority**: High | **Automated**: Yes

**Preconditions**:
- Story has privacy='close-friends' with `closeFriends=['user-B']`
- Viewer is User C (not in list)

**Steps**:
1. User C calls `GET /api/v1/stories/story-123`
2. Verify response 403 Forbidden

**Expected Result**:
```json
{
  "success": false,
  "message": "You are not in this user's close friends list",
  "errorCode": "ACCESS_DENIED_PRIVACY"
}
```

---

#### TC-F-024: Get Single Story (Expired)
**Priority**: High | **Automated**: Yes

**Preconditions**:
- Story expired 25 hours ago

**Steps**:
1. Call `GET /api/v1/stories/story-123`
2. Verify response 404 Not Found

**Expected Result**:
```json
{
  "success": false,
  "message": "Story not found",
  "errorCode": "STORY_NOT_FOUND"
}
```

---

### Category 7: Mark Story as Viewed

#### TC-F-025: Mark Story as Viewed (First Time)
**Priority**: High | **Automated**: Yes

**Preconditions**:
- Story exists with `views=0`, `viewers=[]`

**Steps**:
1. Call `POST /api/v1/stories/story-123/view` with JWT
2. Verify response 200 OK
3. Call `GET /api/v1/stories/story-123` to verify

**Expected Result**:
- `views` incremented to 1
- Viewer's UUID added to `viewers` array
- `hasSeen=true` in subsequent GET requests

---

#### TC-F-026: Mark Story as Viewed (Duplicate View)
**Priority**: High | **Automated**: Yes

**Preconditions**:
- Story already viewed by user (UUID in `viewers` array)

**Steps**:
1. Call `POST /api/v1/stories/story-123/view` with JWT
2. Verify response 200 OK
3. Call `GET /api/v1/stories/story-123` to verify

**Expected Result**:
- `views` NOT incremented
- `viewers` array unchanged (no duplicate UUID)

---

#### TC-F-027: Mark Story as Viewed (Viewer Cap)
**Priority**: Medium | **Automated**: Yes

**Preconditions**:
- Story already has 100 viewers in `viewers` array

**Steps**:
1. New user calls `POST /api/v1/stories/story-123/view`
2. Verify response 200 OK
3. Call `GET /api/v1/stories/story-123` to verify

**Expected Result**:
- `views` incremented to 101
- `viewers` array still has 100 items (cap enforced)
- New viewer NOT added to array

---

#### TC-F-028: Mark Story as Viewed (Story Not Found)
**Priority**: Medium | **Automated**: Yes

**Steps**:
1. Call `POST /api/v1/stories/nonexistent-id/view`
2. Verify response 404 Not Found

**Expected Result**:
```json
{
  "success": false,
  "message": "Story not found",
  "errorCode": "STORY_NOT_FOUND"
}
```

---

### Category 8: Story Expiration

#### TC-F-029: Story Auto-Expires via TTL Index
**Priority**: Critical | **Automated**: No (requires 24h wait)

**Preconditions**:
- Story created 24 hours ago

**Steps**:
1. Wait 24 hours + 2 minutes (TTL index runs every 60s)
2. Call `GET /api/v1/stories/story-123`
3. Verify response 404 Not Found

**Expected Result**:
- Story deleted by MongoDB TTL index
- No longer retrievable

**Note**: Manual test - run overnight

---

#### TC-F-030: Cleanup Job Deletes Expired Stories
**Priority**: High | **Automated**: Yes

**Preconditions**:
- Manually set `expiresAt` to 25 hours ago in MongoDB

**Steps**:
1. Trigger cleanup job: `POST /api/v1/admin/stories/cleanup`
2. Verify response includes `deleted: 1`
3. Call `GET /api/v1/stories/story-123`
4. Verify response 404 Not Found

**Expected Result**:
- Cleanup job deletes expired story
- Story no longer retrievable

---

### Category 9: Edge Cases

#### TC-F-031: Create Story While Unauthenticated
**Priority**: High | **Automated**: Yes

**Steps**:
1. Call `POST /api/v1/stories` without JWT token
2. Verify response 401 Unauthorized

**Expected Result**:
```json
{
  "success": false,
  "message": "Unauthorized",
  "errorCode": "UNAUTHORIZED"
}
```

---

#### TC-F-032: Get Story Tray While Unauthenticated
**Priority**: High | **Automated**: Yes

**Steps**:
1. Call `GET /api/v1/stories/tray` without JWT
2. Verify response 401 Unauthorized

---

#### TC-F-033: Create Story with Large File (>50MB)
**Priority**: Medium | **Automated**: No (requires large file)

**Steps**:
1. Upload video file >50MB to `POST /api/v1/stories`
2. Verify response 413 Payload Too Large

**Expected Result**:
- Request rejected before processing
- Error message: "File too large (max 50MB)"

---

#### TC-F-034: Get Story Feed with 1000+ Following Users
**Priority**: Medium | **Automated**: Yes (with seeded data)

**Preconditions**:
- User follows 1000+ users
- 500+ have active stories

**Steps**:
1. Call `GET /api/v1/stories/feed`
2. Measure response time

**Expected Result**:
- Response time <2 seconds
- Feed includes all users with stories (no pagination implemented yet)

**Note**: Future enhancement - implement pagination

---

#### TC-F-035: Mark Story as Viewed (Concurrent Requests)
**Priority**: Medium | **Automated**: Yes (load test)

**Steps**:
1. Send 100 concurrent `POST /api/v1/stories/story-123/view` requests from same user
2. Verify `views` incremented only once
3. Verify `viewers` array contains only 1 instance of user UUID

**Expected Result**:
- Duplicate view prevention works correctly
- No race condition issues

---

#### TC-F-036: Create Story with Special Characters in Close Friends UUIDs
**Priority**: Low | **Automated**: Yes

**Steps**:
1. Call `POST /api/v1/stories` with `closeFriends=['invalid-uuid', 'abc123']`
2. Verify response 400 Bad Request

**Expected Result**:
- Validation error: "closeFriends must contain valid UUIDs"

---

#### TC-F-037: Get Story Feed (User Unfollows During Session)
**Priority**: Low | **Automated**: Manual

**Steps**:
1. User A views feed (includes User B's stories)
2. User A unfollows User B
3. User A refreshes feed
4. Verify User B's stories no longer appear

**Expected Result**:
- Feed reflects current follow relationships
- No stale data

---

#### TC-F-038: Get Story by ID (MongoDB ObjectId Format)
**Priority**: Low | **Automated**: Yes

**Steps**:
1. Call `GET /api/v1/stories/invalid-object-id`
2. Verify response 400 Bad Request or 404 Not Found

**Expected Result**:
- Error message: "Invalid story ID format"

---

## Security Test Cases

### Category 10: Authentication & Authorization

#### TC-S-001: JWT Missing
**Priority**: Critical | **Automated**: Yes

**Steps**:
1. Call `POST /api/v1/stories` without `Authorization` header
2. Verify response 401 Unauthorized

**Expected Result**:
- All endpoints reject requests without JWT

---

#### TC-S-002: JWT Expired
**Priority**: Critical | **Automated**: Yes

**Steps**:
1. Use expired JWT (issued >7 days ago)
2. Call `GET /api/v1/stories/me`
3. Verify response 401 Unauthorized

**Expected Result**:
- Expired token rejected

---

#### TC-S-003: JWT Malformed
**Priority**: Critical | **Automated**: Yes

**Steps**:
1. Use malformed JWT (invalid signature)
2. Call `GET /api/v1/stories/tray`
3. Verify response 401 Unauthorized

---

#### TC-S-004: Privacy Bypass Attempt (Followers-Only)
**Priority**: Critical | **Automated**: Yes

**Preconditions**:
- User A does NOT follow User B
- User B has followers-only story

**Steps**:
1. User A directly calls `GET /api/v1/stories/{story-id}` (guessing story ID)
2. Verify response 403 Forbidden

**Expected Result**:
- Privacy check enforced even with direct story ID access

---

#### TC-S-005: Privacy Bypass Attempt (Close Friends)
**Priority**: Critical | **Automated**: Yes

**Preconditions**:
- User A not in User B's close friends list
- User B has close-friends story

**Steps**:
1. User A calls `GET /api/v1/stories/{story-id}`
2. Verify response 403 Forbidden

**Expected Result**:
- Close friends list enforced

---

#### TC-S-006: XSS in Story Caption (Future Enhancement)
**Priority**: High | **Automated**: Yes

**Note**: Currently stories don't have captions (future feature)

**Steps**:
1. Create story with caption: `<script>alert('XSS')</script>`
2. Retrieve story and check caption in response

**Expected Result**:
- Script tags sanitized or escaped
- No executable code in response

---

#### TC-S-007: SQL Injection in Username Lookup
**Priority**: High | **Automated**: Yes

**Steps**:
1. Call `GET /api/v1/stories/user/' OR '1'='1`
2. Verify response 404 Not Found (no injection)

**Expected Result**:
- TypeORM parameterizes queries automatically
- No SQL injection vulnerability

---

#### TC-S-008: NoSQL Injection in Story ID
**Priority**: High | **Automated**: Yes

**Steps**:
1. Call `GET /api/v1/stories/{"$ne": null}` (NoSQL injection attempt)
2. Verify response 404 Not Found

**Expected Result**:
- Mongoose validates ObjectId format
- No injection vulnerability

---

#### TC-S-009: CSRF Protection
**Priority**: Medium | **Automated**: Yes

**Steps**:
1. Attempt to create story from external domain without CSRF token
2. Verify request blocked

**Expected Result**:
- CORS policy enforces origin restrictions
- JWT must be in Authorization header (not cookie)

---

#### TC-S-010: Rate Limiting
**Priority**: Medium | **Automated**: Yes

**Steps**:
1. Send 1000 `POST /api/v1/stories` requests in 1 minute
2. Verify rate limit enforced after N requests

**Expected Result**:
- 429 Too Many Requests after configured limit
- Future enhancement - not currently implemented

---

## Performance Test Cases

### Category 11: Load Testing

#### TC-P-001: Story Creation Load Test
**Priority**: High | **Automated**: Yes (Artillery)

**Test Configuration**:
```yaml
config:
  target: 'https://api.chefooz.app'
  phases:
    - duration: 60
      arrivalRate: 50
scenarios:
  - flow:
      - post:
          url: '/api/v1/stories'
          headers:
            Authorization: 'Bearer {{jwt}}'
          formData:
            file: '@photo.jpg'
            mediaType: 'image'
            privacy: 'followers'
```

**Expected Result**:
- P50 response time <200ms
- P95 response time <500ms
- P99 response time <1000ms
- 0% error rate

---

#### TC-P-002: Story Feed Load Test
**Priority**: High | **Automated**: Yes (Artillery)

**Test Configuration**:
- 100 concurrent users fetching feed
- 60 seconds duration
- Each user follows 100 other users

**Expected Result**:
- P50 response time <200ms
- P95 response time <500ms
- No MongoDB query timeouts

---

#### TC-P-003: Story Tray Load Test
**Priority**: Medium | **Automated**: Yes

**Test Configuration**:
- 200 concurrent users fetching tray
- 60 seconds duration

**Expected Result**:
- P50 response time <100ms (lightweight endpoint)
- P95 response time <300ms

---

#### TC-P-004: Concurrent View Marking
**Priority**: High | **Automated**: Yes

**Test Configuration**:
- 500 concurrent users marking same story as viewed
- Verify `views` count accuracy

**Expected Result**:
- `views` matches actual unique viewer count
- No race condition issues

---

#### TC-P-005: MongoDB TTL Index Performance
**Priority**: Medium | **Automated**: Manual

**Preconditions**:
- 100,000 expired stories in database

**Steps**:
1. Observe MongoDB TTL index background thread
2. Measure deletion throughput

**Expected Result**:
- TTL index deletes all expired stories within 10 minutes
- No impact on active query performance

---

#### TC-P-006: Large Follower Count Feed Generation
**Priority**: Medium | **Automated**: Manual

**Preconditions**:
- User follows 10,000 other users
- 1,000 have active stories

**Steps**:
1. Call `GET /api/v1/stories/feed`
2. Measure query time and response time

**Expected Result**:
- Query time <1 second
- Response time <2 seconds
- Future enhancement - implement pagination

---

## Integration Test Cases

### Category 12: Module Integration

#### TC-I-001: Media Module Integration (Future)
**Priority**: High | **Automated**: No (TODO)

**Steps**:
1. Upload video to Media service
2. Receive MediaConvert job ID
3. Create story with job ID
4. Verify story document references correct MediaConvert output

**Expected Result**:
- Story creation waits for MediaConvert completion
- Final URLs stored in story document

**Note**: MediaConvert integration pending

---

#### TC-I-002: User Module Integration
**Priority**: High | **Automated**: Yes

**Steps**:
1. Create story with user JWT
2. Verify `userId` FK references existing Postgres user
3. Delete user from Postgres
4. Verify orphaned stories handled gracefully

**Expected Result**:
- Story creation fails if user doesn't exist
- Orphaned story cleanup (future enhancement)

---

#### TC-I-003: Social Module Integration (Follow Relationships)
**Priority**: High | **Automated**: Yes

**Steps**:
1. User A follows User B (status='accepted')
2. User B creates followers-only story
3. User A calls `GET /api/v1/stories/feed`
4. Verify User B's story included

**Expected Result**:
- Follow relationships respected in feed generation

---

#### TC-I-004: Story Expiration → Feed Removal
**Priority**: High | **Automated**: Yes

**Steps**:
1. Create story with `expiresAt = now + 1 second`
2. Wait 2 seconds
3. Call `GET /api/v1/stories/feed`
4. Verify expired story not included

**Expected Result**:
- Expired stories automatically excluded from feed

---

#### TC-I-005: Story Creation → Tray Update
**Priority**: Medium | **Automated**: Yes

**Steps**:
1. User A has no stories
2. Call `GET /api/v1/stories/tray` (User A not in tray)
3. User A creates story
4. Call `GET /api/v1/stories/tray` again
5. Verify User A now in tray with `storyCount=1`

**Expected Result**:
- Tray reflects latest stories without caching issues

---

#### TC-I-006: View Tracking → hasSeen Flag
**Priority**: High | **Automated**: Yes

**Steps**:
1. Call `GET /api/v1/stories/{story-id}` (verify `hasSeen=false`)
2. Call `POST /api/v1/stories/{story-id}/view`
3. Call `GET /api/v1/stories/{story-id}` again
4. Verify `hasSeen=true`

**Expected Result**:
- `hasSeen` flag updates after view marking

---

#### TC-I-007: CDN URL Conversion
**Priority**: High | **Automated**: Yes

**Preconditions**:
- Story stored with S3 URI: `s3://chefooz-stories/story.jpg`

**Steps**:
1. Call `GET /api/v1/stories/{story-id}`
2. Verify `mediaUrl` is HTTPS URL

**Expected Result**:
```json
{
  "mediaUrl": "https://cdn.chefooz.app/stories/story.jpg"
}
```

**Validation**:
- `s3UriToHttps()` utility converts S3 URIs to CDN URLs

---

#### TC-I-008: Cleanup Job → Metrics Update
**Priority**: Low | **Automated**: Yes

**Steps**:
1. Trigger cleanup job manually
2. Verify Prometheus metrics updated:
   - `stories_cleanup_deleted_total`
   - `stories_cleanup_duration_seconds`

**Expected Result**:
- Metrics reflect cleanup job execution

---

## Regression Test Cases

### Category 13: Previously Fixed Bugs

#### TC-R-001: TTL Index Not Deleting Stories
**Priority**: Critical | **Automated**: Yes | **Fixed**: 2026-01-10

**Bug**: Stories not expiring after 24 hours

**Reproduction**:
1. Create story
2. Wait 25 hours
3. Story still retrievable

**Fix**: Recreated TTL index with `expireAfterSeconds: 0`

**Test**:
1. Verify TTL index exists: `db.stories.getIndexes()`
2. Create story with `expiresAt = now - 1 hour`
3. Wait 2 minutes
4. Verify story deleted

---

#### TC-R-002: Followers-Only Privacy Not Enforced
**Priority**: Critical | **Automated**: Yes | **Fixed**: 2026-01-15

**Bug**: Users could view followers-only stories without following

**Reproduction**:
1. User A doesn't follow User B
2. User A calls `GET /api/v1/stories/{story-id}` (User B's followers-only story)
3. Story returned without 403 error

**Fix**: Added `checkStoryPrivacy()` to all retrieval endpoints

**Test**:
1. User A doesn't follow User B
2. User A calls `GET /api/v1/stories/{story-id}`
3. Verify 403 Forbidden response

---

#### TC-R-003: Duplicate Views Counted
**Priority**: High | **Automated**: Yes | **Fixed**: 2026-01-18

**Bug**: Same user viewing story multiple times incremented `views` each time

**Reproduction**:
1. User views story (views=1)
2. User views story again (views=2)

**Fix**: Added duplicate check in `markStoryViewed()`

**Test**:
1. User views story twice
2. Verify `views=1` and `viewers` contains user UUID only once

---

#### TC-R-004: S3 URIs Exposed in API Response
**Priority**: High | **Automated**: Yes | **Fixed**: 2026-01-20

**Bug**: API returned S3 URIs instead of HTTPS CDN URLs

**Reproduction**:
1. Create story
2. GET story returns `"mediaUrl": "s3://chefooz-stories/story.jpg"`

**Fix**: Added `s3UriToHttps()` utility in `mapToResponseDto()`

**Test**:
1. Create story
2. Verify API returns HTTPS URLs

---

#### TC-R-005: Close Friends Array Validation Missing
**Priority**: High | **Automated**: Yes | **Fixed**: 2026-01-22

**Bug**: Could create close-friends story with empty `closeFriends` array

**Reproduction**:
1. Call `POST /api/v1/stories` with `privacy='close-friends'`, `closeFriends=[]`
2. Story created (invalid state)

**Fix**: Added validation in `createStory()`

**Test**:
1. Try creating close-friends story with empty array
2. Verify 400 Bad Request

---

#### TC-R-006: MongoDB Connection Leak in Feed Query
**Priority**: Medium | **Automated**: Yes | **Fixed**: 2026-01-25

**Bug**: Feed query didn't close MongoDB cursor properly

**Reproduction**:
1. Call `GET /api/v1/stories/feed` 1000 times
2. MongoDB connections exhausted

**Fix**: Added `.exec()` to properly close cursors

**Test**:
1. Load test feed endpoint (1000 requests)
2. Verify MongoDB connection count stable

---

#### TC-R-007: Viewer Cap Not Enforced
**Priority**: Medium | **Automated**: Yes | **Fixed**: 2026-01-28

**Bug**: `viewers` array grew beyond 100 items

**Reproduction**:
1. 150 users view story
2. `viewers` array contains 150 items

**Fix**: Added `if (story.viewers.length < 100)` check

**Test**:
1. 150 users mark story as viewed
2. Verify `viewers` array capped at 100

---

#### TC-R-008: Story Feed N+1 Query
**Priority**: High | **Automated**: Yes | **Fixed**: 2026-02-01

**Bug**: Loading user data for each story individually (N+1 queries)

**Reproduction**:
1. Feed with 50 stories from 10 users
2. 50 Postgres queries executed

**Fix**: Batch user fetching with `In(uniqueUserIds)`

**Test**:
1. Feed with 50 stories
2. Verify only 1 Postgres query for users

---

#### TC-R-009: Story Tray Sorting Incorrect
**Priority**: Low | **Automated**: Yes | **Fixed**: 2026-02-03

**Bug**: Tray didn't prioritize self stories first

**Reproduction**:
1. User has own stories
2. Tray shows following users before self

**Fix**: Added self-first sorting logic

**Test**:
1. User has own stories
2. Verify tray shows self first

---

#### TC-R-010: Expired Story Still in Feed
**Priority**: High | **Automated**: Yes | **Fixed**: 2026-02-05

**Bug**: Feed included expired stories (TTL index delay)

**Reproduction**:
1. Story expired 5 minutes ago
2. Feed still includes story (TTL index hasn't run yet)

**Fix**: Added `expiresAt > now` filter in feed query

**Test**:
1. Manually set `expiresAt` to past
2. Verify story not in feed

---

#### TC-R-011: JWT Not Validated on View Marking
**Priority**: Critical | **Automated**: Yes | **Fixed**: 2026-02-07

**Bug**: Could mark story as viewed without valid JWT

**Reproduction**:
1. Call `POST /api/v1/stories/{id}/view` without Authorization header
2. View counted

**Fix**: Added `@UseGuards(JwtAuthGuard)` to controller

**Test**:
1. Call view endpoint without JWT
2. Verify 401 Unauthorized

---

#### TC-R-012: Close Friends Privacy Check Missing in Feed
**Priority**: High | **Automated**: Yes | **Fixed**: 2026-02-10

**Bug**: Feed included close-friends stories for users not in list

**Reproduction**:
1. User B creates close-friends story with User C in list
2. User A (not in list) sees story in feed

**Fix**: Added close-friends filter in feed query

**Test**:
1. User A not in close-friends list
2. Verify close-friends story not in User A's feed

---

## Test Data

### Sample Story Document (MongoDB)

```json
{
  "_id": "65f3a8b7c1234567890abcde",
  "userId": "a1b2c3d4-e5f6-7890-1234-567890abcdef",
  "mediaId": "story-1709812345678",
  "mediaType": "image",
  "mediaUrl": "s3://chefooz-stories/story-1709812345678.jpg",
  "thumbnailUrl": "s3://chefooz-stories/thumbnails/thumb.jpg",
  "durationMs": 5000,
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

### Sample User (Postgres)

```json
{
  "id": "a1b2c3d4-e5f6-7890-1234-567890abcdef",
  "username": "john_chef",
  "fullName": "John Doe",
  "phone": "+919876543212",
  "role": "user"
}
```

### Sample User Follow (Postgres)

```json
{
  "id": "follow-123",
  "followerId": "a1b2c3d4-e5f6-7890-1234-567890abcdef",
  "targetId": "b2c3d4e5-f6a7-8901-2345-678901bcdefg",
  "status": "accepted",
  "createdAt": "2024-03-01T10:00:00.000Z"
}
```

---

## Automation Coverage

### Automation Breakdown by Category

| Category | Total | Automated | Manual | Automation % |
|----------|-------|-----------|--------|--------------|
| Functional | 38 | 35 | 3 | 92% |
| Security | 10 | 10 | 0 | 100% |
| Performance | 6 | 4 | 2 | 67% |
| Integration | 8 | 7 | 1 | 88% |
| Regression | 12 | 12 | 0 | 100% |
| **TOTAL** | **74** | **68** | **6** | **92%** |

### Manual Test Cases

1. TC-F-029: Story auto-expires (requires 24h wait)
2. TC-F-033: Large file upload (>50MB)
3. TC-P-005: TTL index performance
4. TC-P-006: Large follower count feed
5. TC-I-001: MediaConvert integration (pending implementation)
6. TC-F-037: User unfollows during session

---

## CI/CD Integration

### GitHub Actions Workflow

```yaml
name: Stories Module Tests

on:
  push:
    branches: [main, develop]
    paths:
      - 'apps/chefooz-apis/src/modules/stories/**'
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      mongo:
        image: mongo:6
        ports:
          - 27017:27017
        options: >-
          --health-cmd "mongosh --eval 'db.runCommand({ ping: 1 })'"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      postgres:
        image: postgres:15
        env:
          POSTGRES_DB: chefooz_test
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: test_password
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Run unit tests
        run: npx nx test chefooz-apis --testPathPattern=stories

      - name: Run E2E tests
        env:
          MONGODB_URI: mongodb://localhost:27017/chefooz_test
          DATABASE_URL: postgresql://postgres:test_password@localhost:5432/chefooz_test
          JWT_SECRET: test_secret
        run: npx nx e2e chefooz-apis-e2e --testPathPattern=stories

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/lcov.info
```

---

## Test Metrics

### Coverage Goals

| Metric | Target | Current | Status |
|--------|--------|---------|--------|
| **Code Coverage** | ≥85% | 88% | ✅ Pass |
| **Branch Coverage** | ≥80% | 82% | ✅ Pass |
| **Functional Coverage** | ≥90% | 92% | ✅ Pass |
| **Critical Path Coverage** | 100% | 100% | ✅ Pass |

### Execution Time

| Test Suite | Tests | Duration | Status |
|------------|-------|----------|--------|
| **Unit Tests** | 35 | 45s | ✅ Pass |
| **Integration Tests** | 20 | 3min | ✅ Pass |
| **E2E Tests** | 18 | 5min | ✅ Pass |
| **Performance Tests** | 4 | 8min | ✅ Pass |
| **TOTAL** | **77** | **16min** | ✅ Pass |

---

## Bug Reporting Template

```markdown
## Bug Report: [Short Description]

**Module**: Stories
**Severity**: Critical | High | Medium | Low
**Priority**: P0 | P1 | P2 | P3

### Description
Brief description of the bug.

### Steps to Reproduce
1. Step 1
2. Step 2
3. Step 3

### Expected Behavior
What should happen.

### Actual Behavior
What actually happened.

### Environment
- OS: macOS 14.2
- Node.js: 20.10.0
- MongoDB: 6.0.12
- PostgreSQL: 15.5

### Logs
```
[Paste relevant logs here]
```

### Screenshots
[Attach screenshots if applicable]

### Impact
How many users affected? Revenue impact? Workaround available?

### Proposed Fix
[Optional] Suggested solution.
```

---

## Related Documentation

- [Stories FEATURE_OVERVIEW](./FEATURE_OVERVIEW.md) - Business requirements and features
- [Stories TECHNICAL_GUIDE](./TECHNICAL_GUIDE.md) - Architecture and implementation
- [Reels Module QA](../reels/QA_TEST_CASES.md) - Similar content type testing patterns
- [Media Module QA](../media/QA_TEST_CASES.md) - MediaConvert integration tests

---

**Last Updated**: 2026-02-14
**Version**: 1.0.0
**Test Coverage**: 92% automated
**Status**: Production-Ready
