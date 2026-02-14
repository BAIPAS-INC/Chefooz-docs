# Social Module - Feature Overview

**Module**: `social`  
**Type**: Core Feature  
**Category**: Social & Discovery  
**Status**: Production  
**Last Updated**: February 14, 2026

---

## 📋 Table of Contents

1. [Executive Summary](#executive-summary)
2. [Module Purpose](#module-purpose)
3. [Key Features](#key-features)
4. [User Flows](#user-flows)
5. [Business Rules](#business-rules)
6. [Integration Points](#integration-points)
7. [Security & Privacy](#security--privacy)
8. [Performance Considerations](#performance-considerations)
9. [Future Enhancements](#future-enhancements)

---

## Executive Summary

The **Social Module** implements Instagram-style social graph functionality for the Chefooz platform, enabling users to follow chefs, manage privacy settings, block unwanted interactions, and share content with their network. This module serves as the foundation for social discovery, content distribution, and community building.

### Core Capabilities

- **Follow System**: Public/private account distinction with follow request approval
- **Blocking**: Comprehensive user blocking with bidirectional relationship cleanup
- **Privacy Controls**: Per-user privacy settings with follower/following counters
- **Reel Sharing**: Multi-channel sharing (copy-link, external, direct, story)
- **Social Search**: Multi-field user search excluding blocked relationships
- **Relationship State**: Rich relationship data for UI rendering

### Technical Highlights

- **Database**: PostgreSQL (TypeORM) for relationships, MongoDB for reels
- **Caching**: Valkey/Redis for social profile and privacy data
- **Pagination**: Cursor-based (createdAt DESC) for infinite scroll
- **Rate Limiting**: 30 follows/min, 30 unfollows/min, 20 blocks/min, 20 shares/min
- **Notifications**: Integrated with NotificationOrchestrator for social events

---

## Module Purpose

### Problem Statement

Food content platforms require robust social graph functionality to:
- Enable users to follow chefs and discover content from their network
- Allow private accounts with gated content access
- Prevent harassment through comprehensive blocking
- Track content sharing and virality
- Provide rich relationship data for personalized UI

### Solution Approach

The Social Module implements:
1. **Instagram-Style Follow System**: Public accounts auto-accept, private accounts require approval
2. **Follow Request Flow**: Pending requests → manual accept/reject → notification
3. **Comprehensive Blocking**: Removes all follows (both directions), prevents future interactions
4. **Privacy Settings**: isPrivate flag controls content visibility and auto-accept behavior
5. **Multi-Channel Sharing**: Track shares by type (copy-link, external, direct, story)
6. **Relationship State**: 6 boolean flags (isFollowing, isFollowedBy, isRequested, isPrivate, isBlockedByMe, hasBlockedMe)

### Success Metrics

- Follow conversion rate (public: 100%, private: 60-80%)
- Average followers per chef (target: 50+)
- Block rate (<5% of total relationships)
- Share rate (>15% of reel views)
- Search discoverability (>70% users findable)

---

## Key Features

### 1. Follow System

**Public Accounts** (isPrivate=false):
- Follow requests auto-accepted immediately
- Followers can view all reels and stories
- Status: 'accepted' from the start
- Notification: FOLLOWED event sent instantly

**Private Accounts** (isPrivate=true):
- Follow requests pending until approved
- Status: 'pending' → 'accepted' (after approval)
- Followers only see content after acceptance
- Notification: FOLLOWED (isPending=true) → FOLLOW_REQUEST_ACCEPTED

**Unfollow**:
- Removes follow relationship
- Decrements counters (only if was 'accepted')
- Invalidates cache for both users
- No notification sent

**Remove Follower** (Instagram-style):
- Target user removes their follower
- Decrements counters
- Soft removal (no notification, no drama)

### 2. Follow Requests

**Pending Requests**:
- List all pending follow requests
- Sorted by createdAt DESC (newest first)
- Includes follower user data (username, fullName, avatar)
- Paginated (cursor-based, limit 20)

**Accept Request**:
- Updates status to 'accepted'
- Increments followersCount, followingCount
- Sends FOLLOW_REQUEST_ACCEPTED notification
- Invalidates cache

**Reject Request**:
- Removes pending request entirely
- No counter changes
- No notification sent
- Soft rejection (no negative feedback)

### 3. Blocking System

**Block User**:
- Removes all follow relationships (both directions)
- Creates block record (UNIQUE constraint on userId + blockedUserId)
- Decrements counters for removed follows
- Prevents future follows
- Invalidates cache for both users
- Bidirectional effect: neither user can interact

**Unblock User**:
- Removes block record
- Invalidates cache
- Re-enables following (user must re-follow manually)

**Block Effects**:
- Blocked user cannot view blocker's profile
- Blocked user cannot view blocker's reels
- Blocked user cannot follow blocker
- Blocked user cannot search for blocker
- Blocker does not see blocked user in search results

**Check Block Status**:
- GET /blocked/check/:targetId returns 3 flags:
  - `isBlocked`: Blocked in either direction
  - `hasBlocked`: Current user blocked target
  - `isBlockedBy`: Target blocked current user

### 4. Privacy Settings

**Privacy Controls**:
- `isPrivate` boolean (default: false)
- Stored in `user_privacy` table
- Auto-created on first access (getOrCreatePrivacy)

**Privacy Effects**:
- Public (false): Anyone can follow, reels are public
- Private (true): Follows require approval, reels hidden from non-followers

**Counters**:
- `followersCount`: Number of accepted followers
- `followingCount`: Number of accepted follows
- Denormalized for performance (avoid COUNT queries)
- Updated atomically on follow/unfollow/block

### 5. Relationship State

**getRelationship** returns 6 flags:
- `isFollowing`: Current user follows target
- `isFollowedBy`: Target follows current user
- `isRequested`: Current user sent pending follow request
- `isPrivate`: Target account is private
- `isBlockedByMe`: Current user blocked target
- `hasBlockedMe`: Target blocked current user

**UI Use Cases**:
- Show "Follow" vs "Requested" vs "Following" button
- Show "Follows You" badge
- Hide content if blocked or private without access
- Disable interactions if blocked

### 6. Social Lists

**Followers**:
- GET /followers → Own followers list
- GET /followers/:userIdOrUsername → User's followers list
- Paginated (cursor-based, limit 20)
- Includes relationship status for viewer (isFollowing, isFollowedBy)
- Filtered to 'accepted' status only

**Following**:
- GET /following → Own following list
- GET /following/:userIdOrUsername → User's following list
- Paginated (cursor-based, limit 20)
- Includes relationship status for viewer
- Filtered to 'accepted' status only

**Blocked Users**:
- GET /blocked → Own blocked users list
- Paginated (cursor-based, limit 20)
- Includes user data (username, fullName, avatar)
- Filters out deleted users

### 7. Social Search

**Search Users**:
- Searches fullName, username, chef businessName
- Uses ILIKE (case-insensitive pattern matching)
- Excludes blocked users (both directions)
- Joins `chef_profiles` table
- Paginated (cursor-based, limit 20)
- Returns user data + isChef flag

**Search Query Examples**:
- "John" → matches fullName ILIKE '%John%'
- "@john" → matches username ILIKE '%john%'
- "Tasty Kitchen" → matches chef businessName ILIKE '%Tasty%'

### 8. Social Profile

**GET /profile/:userId**:
- Returns enriched projection with:
  - User data (id, username, fullName, avatar, bio, etc.)
  - Privacy settings (isPrivate, followersCount, followingCount)
  - Relationship status (6 flags)
  - Chef profile (if chef)
  - Verification status (TODO)
  - Reputation score (TODO)

**Use Cases**:
- Profile screen data fetch
- Render follow button state
- Show follower/following counts
- Display chef badge
- Hide content if blocked/private

### 9. Reel Sharing

**Share Types**:
- `copy-link`: User copies reel link to clipboard
- `external`: User shares to external app (WhatsApp, Instagram, etc.)
- `direct`: User shares directly to another Chefooz user (in-app)
- `story`: User reposts reel to their story

**Share Flow**:
1. POST /reels/:reelId/share with type and optional targetUserId
2. Validate reel exists
3. Check rate limit (20 shares/min)
4. Validate target user (for direct shares)
5. Check blocking (cannot share to blocked users)
6. Increment counters: shareCount + type-specific counter
7. Generate deep link (short, universal, app)
8. Send notification (for direct shares)
9. Return updated share count + deep link

**Share Stats**:
- GET /reels/:reelId/share/stats returns:
  - `shareCount`: Total shares
  - `copyLinkCount`: Copy link shares
  - `externalShareCount`: External app shares
  - `directShareCount`: Direct user shares
  - `storyShareCount`: Story reposts

**Targetable Users**:
- GET /reels/:reelId/share/targetable-users
- Returns followers + following (mutual connections prioritized)
- Excludes blocked users
- Includes relationship status (isFollowing, isFollowedBy)

---

## User Flows

### Flow 1: Follow a Public Account

**Actors**: User A (follower), User B (public chef)

**Steps**:
1. User A navigates to User B's profile
2. App fetches relationship state: `GET /social/relationship/:userBId`
3. App shows "Follow" button (isFollowing=false, isPrivate=false)
4. User A taps "Follow"
5. App calls `POST /social/follow/:userBId`
6. Backend:
   - Checks if User B is private → No (isPrivate=false)
   - Creates UserFollow record with status='accepted'
   - Increments followersCount (User B), followingCount (User A)
   - Invalidates cache: social:profile:A, social:profile:B, social:privacy:A, social:privacy:B
   - Sends FOLLOWED notification to User B (isPending=false)
7. Response: `{ success: true, status: 'accepted' }`
8. App updates button to "Following"

**Result**: User A immediately follows User B, sees all content, User B receives notification.

---

### Flow 2: Follow a Private Account

**Actors**: User A (follower), Chef C (private chef)

**Steps**:
1. User A navigates to Chef C's profile
2. App fetches relationship state: `GET /social/relationship/:chefCId`
3. App shows "Follow" button (isFollowing=false, isPrivate=true)
4. User A taps "Follow"
5. App calls `POST /social/follow/:chefCId`
6. Backend:
   - Checks if Chef C is private → Yes (isPrivate=true)
   - Creates UserFollow record with status='pending'
   - Does NOT increment counters
   - Invalidates cache
   - Sends FOLLOWED notification to Chef C (isPending=true)
7. Response: `{ success: true, status: 'pending' }`
8. App updates button to "Requested"

**Chef C's Actions**:
9. Chef C receives notification: "User A requested to follow you"
10. Chef C opens follow requests: `GET /social/requests/pending`
11. Chef C sees User A's request with user data
12. Chef C taps "Accept"
13. App calls `POST /social/requests/:requestId/accept`
14. Backend:
   - Updates status to 'accepted'
   - Increments followersCount (Chef C), followingCount (User A)
   - Invalidates cache
   - Sends FOLLOW_REQUEST_ACCEPTED notification to User A
15. Response: `{ success: true }`

**Result**: User A now follows Chef C, sees all content, receives acceptance notification.

---

### Flow 3: Unfollow a User

**Actors**: User A (follower), User B (followed)

**Steps**:
1. User A navigates to User B's profile
2. App shows "Following" button (isFollowing=true)
3. User A taps "Following" → confirmation modal
4. User A confirms unfollow
5. App calls `POST /social/unfollow/:userBId`
6. Backend:
   - Finds UserFollow record (followerId=A, targetId=B, status='accepted')
   - Deletes record
   - Decrements followersCount (User B), followingCount (User A)
   - Invalidates cache
   - No notification sent
7. Response: `{ success: true }`
8. App updates button to "Follow"

**Result**: User A no longer follows User B, counters updated, no notification.

---

### Flow 4: Block a User

**Actors**: User A (blocker), User D (disruptive user)

**Steps**:
1. User A navigates to User D's profile
2. User A taps "Block User" (in overflow menu)
3. App shows confirmation modal: "Block @userD? They won't be able to follow you or view your profile."
4. User A confirms block
5. App calls `POST /social/block/:userDId`
6. Backend:
   - Checks self-blocking → No (targetId ≠ userId)
   - Finds all follow relationships (both directions):
     - UserFollow(followerId=A, targetId=D) → Deleted
     - UserFollow(followerId=D, targetId=A) → Deleted
   - Decrements counters for removed follows
   - Creates UserBlock record (userId=A, blockedUserId=D)
   - Invalidates cache for both users
   - Logs notification error (no negative notification sent)
7. Response: `{ success: true }`
8. App updates UI: "Blocked" status, hides content

**Result**: User A blocks User D, all follows removed, User D cannot interact with User A.

---

### Flow 5: Share Reel to Another User

**Actors**: User A (sharer), User E (recipient), Reel R (shared content)

**Steps**:
1. User A views Reel R
2. User A taps "Share" button
3. App shows share modal with options: Copy Link, WhatsApp, Direct Share, Story
4. User A selects "Direct Share"
5. App opens targetable users list: `GET /reels/:reelRId/share/targetable-users`
6. Backend returns followers + following (excludes blocked users)
7. User A selects User E from list
8. App calls `POST /reels/:reelRId/share` with type='direct', targetUserId=E
9. Backend:
   - Checks rate limit (20 shares/min) → OK
   - Validates Reel R exists → Yes
   - Validates User E exists → Yes
   - Checks blocking (A ↔ E) → No blocks
   - Increments shareCount, directShareCount atomically
   - Generates deep link (short, universal, app)
   - Sends REEL_SHARED_DIRECT notification to User E (TODO)
   - Tracks analytics: REEL_SHARED event
10. Response: `{ success: true, updatedShareCount: 43, deepLink: {...} }`
11. User E receives notification: "User A shared a reel with you"
12. User E taps notification → opens Reel R

**Result**: Reel R shared to User E, share count incremented, deep link generated, notification sent.

---

### Flow 6: Search for Users

**Actors**: User A (searcher)

**Steps**:
1. User A navigates to Explore → Search tab
2. User A types "John" in search bar
3. App calls `GET /social/search?query=John&limit=20`
4. Backend:
   - Searches fullName ILIKE '%John%', username ILIKE '%John%', chef businessName ILIKE '%John%'
   - Joins `chef_profiles` table
   - Excludes blocked users:
     - UserBlock(userId=A, blockedUserId=X) → Exclude X
     - UserBlock(userId=X, blockedUserId=A) → Exclude X
   - Orders by createdAt DESC
   - Returns first 20 results with nextCursor
5. Response: `{ success: true, data: { items: [...], nextCursor: "..." } }`
6. App displays search results with user avatars, usernames, "Follow" buttons
7. User A scrolls down → App fetches next page with cursor

**Result**: User A discovers users matching "John", excluding blocked relationships, with pagination.

---

### Flow 7: Remove a Follower (Instagram-style)

**Actors**: Chef C (target), User F (follower to remove)

**Steps**:
1. Chef C navigates to their followers list: `GET /social/followers`
2. Chef C sees User F in the list
3. Chef C taps "..." on User F's row → "Remove Follower"
4. App shows confirmation: "Remove @userF as a follower?"
5. Chef C confirms
6. App calls `POST /social/remove-follower/:userFId`
7. Backend:
   - Finds UserFollow record (followerId=F, targetId=C, status='accepted')
   - Deletes record
   - Decrements followersCount (Chef C), followingCount (User F)
   - Invalidates cache
   - No notification sent (soft removal)
8. Response: `{ success: true }`
9. App removes User F from followers list

**Result**: User F no longer follows Chef C, counters updated, no notification (soft removal for privacy).

---

## Business Rules

### Follow System Rules

1. **Public Account Auto-Accept**:
   - If target.isPrivate = false → status = 'accepted' immediately
   - Counters incremented on follow
   - FOLLOWED notification sent (isPending=false)

2. **Private Account Pending**:
   - If target.isPrivate = true → status = 'pending'
   - Counters NOT incremented until acceptance
   - FOLLOWED notification sent (isPending=true)
   - Pending request visible in target's requests list

3. **Duplicate Follow Prevention**:
   - UNIQUE constraint on (followerId, targetId)
   - Attempting duplicate follow returns existing relationship

4. **Self-Follow Prevention**:
   - Backend validates targetId ≠ userId
   - Returns 400 Bad Request if attempted

5. **Block Prevents Follow**:
   - Cannot follow if blocked in either direction
   - Returns 403 Forbidden

6. **Unfollow Counter Rules**:
   - Decrements counters only if status was 'accepted'
   - Pending requests deleted without counter changes

7. **Follow Request Expiration**:
   - Pending requests never expire (manual rejection required)
   - TODO: Implement 90-day auto-expiration

### Blocking Rules

1. **Bidirectional Block Effect**:
   - Block applies in both directions
   - Neither user can interact after block

2. **Follow Cleanup on Block**:
   - Remove all follow relationships (both directions)
   - Decrement counters for removed follows
   - Prevents future follows

3. **Block Prevents Interactions**:
   - Cannot view profile
   - Cannot view reels/stories
   - Cannot follow
   - Cannot search
   - Cannot share content
   - Cannot send messages (TODO)

4. **Self-Block Prevention**:
   - Backend validates targetId ≠ userId
   - Returns 400 Bad Request if attempted

5. **Unblock Re-Enable**:
   - Unblock removes block record
   - Users must re-follow manually (not automatic)

### Privacy Rules

1. **Default Privacy**:
   - New accounts: isPrivate = false
   - Auto-created on first access (getOrCreatePrivacy)

2. **Privacy Toggle**:
   - Users can toggle isPrivate at any time
   - Existing followers remain unaffected
   - Future follows require approval if toggled to private

3. **Content Visibility**:
   - Public accounts: All content visible to everyone
   - Private accounts: Content visible to accepted followers only
   - Blocked users: No content visible regardless of privacy

4. **Counter Visibility**:
   - followersCount, followingCount always visible
   - Privacy setting does not hide counters

### Sharing Rules

1. **Share Rate Limiting**:
   - Max 20 shares per minute per user
   - Enforced in-memory (production: use Redis)
   - Returns 429 Too Many Requests if exceeded

2. **Direct Share Requirements**:
   - Requires targetUserId
   - Target user must exist
   - Cannot share to self
   - Cannot share to blocked users

3. **Share Counter Increments**:
   - Atomic increments: shareCount + type-specific counter
   - MongoDB $inc operator for atomic updates

4. **Share Notifications**:
   - Direct shares → REEL_SHARED_DIRECT notification (TODO)
   - Other share types → no notification

5. **Share Analytics**:
   - Track all shares for analytics (TODO)
   - Event: REEL_SHARED with type and targetUserId

### Search Rules

1. **Multi-Field Search**:
   - Searches fullName, username, chef businessName
   - Uses ILIKE (case-insensitive, pattern matching)

2. **Block Filtering**:
   - Excludes blocked users (both directions)
   - Applies to both search query and pagination

3. **Pagination**:
   - Cursor-based (createdAt DESC)
   - Default limit: 20
   - Returns nextCursor for next page

4. **Chef Profile Join**:
   - Joins `chef_profiles` table for chef data
   - Returns isChef flag

---

## Integration Points

### 1. User Module

**Dependencies**:
- User entity for user data lookups
- Username validation
- UUID resolution

**Integration**:
```typescript
import { User } from '../../database/entities/user.entity';

// Lookup user by username or UUID
const user = await this.userRepository.findOne({ where: { username } });
const user = await this.userRepository.findOne({ where: { id: userId } });
```

**Use Cases**:
- Resolve username to userId
- Validate target user exists
- Fetch user data for search results, followers lists, etc.

### 2. Chef Profile Module

**Dependencies**:
- ChefProfile entity for chef data
- Chef businessName for search

**Integration**:
```typescript
import { ChefProfile } from '../chef/entities/chef-profile.entity';

// Join chef profile for search
.leftJoinAndSelect('user.chefProfile', 'chef')
.where('chef.businessName ILIKE :query', { query: `%${dto.query}%` })
```

**Use Cases**:
- Search by chef kitchen name
- Display chef badge in search results
- Show chef profile in social profile projection

### 3. Notification Module

**Dependencies**:
- NotificationOrchestrator for social events
- Notification types: FOLLOWED, FOLLOW_REQUEST_ACCEPTED, REEL_SHARED_DIRECT

**Integration**:
```typescript
import { NotificationOrchestrator } from '../notification/services/notification-orchestrator.service';

// Send follow notification
await this.notificationOrchestrator.createWithContext({
  type: 'FOLLOWED',
  userId: targetId,
  actorId: userId,
  contextData: {
    isPending: isPrivate,
  },
});
```

**Events Sent**:
- **FOLLOWED**: User X followed you (with isPending flag for private accounts)
- **FOLLOW_REQUEST_ACCEPTED**: User X accepted your follow request
- **REEL_SHARED_DIRECT**: User X shared a reel with you (TODO)

### 4. Deeplink Module

**Dependencies**:
- DeeplinkService for reel share links
- Generate short, universal, and app links

**Integration**:
```typescript
import { DeeplinkService } from '../deeplink/deeplink.service';

// Generate reel deep link
const deepLink = await this.deeplinkService.generateReelLink(reelId);
// Returns: { short, universal, app }
```

**Use Cases**:
- Generate share links for copy-link, external shares
- Return deep link in share response
- Enable direct navigation to reel from external apps

### 5. Activity Module

**Dependencies**:
- Activity feed tracking for social events
- Track follows, shares for feed generation

**Integration**:
```typescript
import { ActivityModule } from '../activity/activity.module';

// Activity feed tracks:
// - FOLLOWED event → Show in follower's feed
// - REEL_SHARED event → Show in recipient's feed
```

**Use Cases**:
- Generate activity feed from social events
- Show "X followed you" in activity timeline
- Show "X shared a reel with you" in activity timeline

### 6. Cache Module

**Dependencies**:
- Valkey/Redis for social profile caching
- Cache keys: social:profile:{userId}, social:privacy:{userId}

**Integration**:
```typescript
import { CACHE_MANAGER } from '@nestjs/cache-manager';
import { Cache } from 'cache-manager';

// Invalidate cache on relationship changes
await this.cacheManager.del(`social:profile:${userId}`);
await this.cacheManager.del(`social:privacy:${userId}`);
```

**Cache Strategy**:
- Cache social profile projections (30 min TTL)
- Cache privacy settings (60 min TTL)
- Invalidate on follow/unfollow/block/privacy changes

### 7. Reels Module (MongoDB)

**Dependencies**:
- Reel schema for share tracking
- Share counters: shareCount, copyLinkCount, externalShareCount, directShareCount, storyShareCount

**Integration**:
```typescript
import { Reel } from '../../database/schemas/reel.schema';

// Increment share counters atomically
await this.reelModel.findByIdAndUpdate(
  reelId,
  { $inc: { 'stats.shareCount': 1, 'stats.copyLinkCount': 1 } },
  { new: true },
);
```

**Use Cases**:
- Track share analytics per reel
- Display share counts in reel UI
- Rank reels by virality (share count)

---

## Security & Privacy

### Authentication

**JWT Guard**:
- All endpoints protected by `@UseGuards(JwtAuthGuard)`
- JWT token required in Authorization header
- User ID extracted from `req.user.id`

**Endpoint Security**:
```typescript
@UseGuards(JwtAuthGuard)
@ApiBearerAuth()
export class SocialController {
  // All endpoints require authentication
}
```

### Rate Limiting

**Follow/Unfollow**:
- 30 requests per minute per user
- Enforced via `@RateLimit(30, 60)` decorator
- Prevents follow spam, bot abuse

**Block/Unblock**:
- 20 requests per minute per user
- Enforced via `@RateLimit(20, 60)` decorator
- Prevents block spam

**Share**:
- 20 shares per minute per user
- Enforced in-memory (production: use Redis)
- Prevents share spam, virality manipulation

### Privacy Protection

**Block Effects**:
- Bidirectional: Neither user can interact
- Removes all follow relationships
- Hides profile from blocked user
- Hides content from blocked user
- Excludes blocked users from search

**Private Account Protection**:
- Follow requests require manual approval
- Content hidden from non-followers
- Pending requests visible only to target

**Profile Visibility**:
- Public accounts: Everyone can view
- Private accounts: Followers only
- Blocked users: No access regardless of privacy

### Data Protection

**Password/Sensitive Data**:
- User entity fields excluded from responses
- Never expose JWT tokens, passwords, emails in social endpoints

**Personal Information**:
- Only public profile data returned (username, fullName, avatar, bio)
- Phone numbers, emails excluded from social responses

**Block Privacy**:
- Blocked users list private (only own list visible)
- Block status check requires authentication
- No notification sent on block (soft block for privacy)

### Spam Prevention

**Duplicate Prevention**:
- UNIQUE constraints on follow, block relationships
- Prevents duplicate follows, duplicate blocks

**Self-Interaction Prevention**:
- Cannot follow self
- Cannot block self
- Cannot share to self

**Rate Limiting**:
- Follow: 30/min
- Unfollow: 30/min
- Block: 20/min
- Share: 20/min

**Block Validation**:
- Cannot follow blocked users
- Cannot share to blocked users
- Cannot search for blocked users

---

## Performance Considerations

### Database Optimization

**Indexes**:
```sql
-- user_follow table
CREATE UNIQUE INDEX idx_user_follow_unique ON user_follow (follower_id, target_id);
CREATE INDEX idx_user_follow_target ON user_follow (target_id);
CREATE INDEX idx_user_follow_follower ON user_follow (follower_id);
CREATE INDEX idx_user_follow_status ON user_follow (status);

-- user_privacy table
CREATE UNIQUE INDEX idx_user_privacy_user ON user_privacy (user_id);

-- user_block table
CREATE UNIQUE INDEX idx_user_block_unique ON user_block (user_id, blocked_user_id);
CREATE INDEX idx_user_block_user ON user_block (user_id);
CREATE INDEX idx_user_block_blocked ON user_block (blocked_user_id);
```

**Query Optimization**:
- Use indexes for follower/following lists (targetId, followerId)
- Use indexes for block checks (userId, blockedUserId)
- Use UNIQUE constraints to prevent duplicates at DB level

### Denormalized Counters

**Why Denormalize**:
- Avoid COUNT queries on large follow tables
- Fast counter retrieval (single SELECT vs COUNT aggregation)
- Counters updated atomically on relationship changes

**Counter Management**:
```typescript
// Increment on follow accept
await this.incrementFollowCounts(followerId, targetId);

// Decrement on unfollow
await this.decrementFollowCounts(followerId, targetId);
```

**Trade-offs**:
- Faster reads (single SELECT)
- Slightly more complex writes (update counters)
- Risk of counter drift (mitigated by atomic updates)

### Caching Strategy

**Cache Keys**:
- `social:profile:{userId}`: Social profile projection (TTL: 30 min)
- `social:privacy:{userId}`: Privacy settings (TTL: 60 min)

**Cache Invalidation**:
- Follow/unfollow → Invalidate both users' profile + privacy
- Block/unblock → Invalidate both users' profile + privacy
- Privacy toggle → Invalidate user's privacy cache

**Cache Hit Rate**:
- Target: >80% for profile projections
- Target: >90% for privacy settings
- Reduces DB load for frequent profile views

### Pagination Strategy

**Cursor-Based Pagination**:
- Avoids OFFSET performance issues on large datasets
- Uses createdAt field for cursor
- Limit+1 pattern for hasMore detection

**Example**:
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
const nextCursor = hasMore ? items[items.length - 1].createdAt.toISOString() : null;
```

**Benefits**:
- Consistent performance regardless of page depth
- No "phantom reads" (data changes between pages)
- Supports infinite scroll UX

### Atomic Operations

**Follow Counter Updates**:
```typescript
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
```

**Share Counter Updates**:
```typescript
await this.reelModel.findByIdAndUpdate(
  reelId,
  { $inc: { 'stats.shareCount': 1, 'stats.copyLinkCount': 1 } },
  { new: true },
);
```

**Benefits**:
- Prevents race conditions (concurrent updates)
- Ensures counter accuracy
- Database-level atomicity (no application-level locks)

---

## Future Enhancements

### Phase 1: Enhanced Social Features (Q2 2026)

1. **Close Friends**:
   - Create private list of close friends
   - Share reels/stories to close friends only
   - Visibility option: Public, Followers, Close Friends

2. **Mute Users**:
   - Mute without unfollowing
   - Hide user's content from feed
   - No notification sent (soft mute)

3. **Follow Request Expiration**:
   - Auto-expire pending requests after 90 days
   - Send reminder notification at 60 days
   - Clean up stale requests

4. **Suggested Users**:
   - Recommend users to follow based on:
     - Mutual connections (friends of friends)
     - Similar interests (cuisine types, location)
     - Popular chefs (high follower count)
   - Personalized "Discover" tab

### Phase 2: Advanced Analytics (Q3 2026)

1. **Follow Insights**:
   - Follow conversion rate (follow requests → accepted)
   - Follower growth over time
   - Top sources of new followers (search, reels, profile views)

2. **Share Analytics**:
   - Share-to-view ratio per reel
   - Top shared reels
   - Share channel breakdown (copy-link, WhatsApp, Instagram, etc.)

3. **Engagement Metrics**:
   - Average interactions per follower
   - Most engaged followers
   - Least engaged followers (candidates for removal)

### Phase 3: Advanced Privacy (Q4 2026)

1. **Restricted Accounts**:
   - Restrict without blocking
   - User doesn't know they're restricted
   - Comments hidden from public, DMs go to request folder
   - Instagram-style soft moderation

2. **Privacy Granularity**:
   - Hide followers list
   - Hide following list
   - Hide like counts
   - Hide share counts

3. **Story Privacy**:
   - Hide stories from specific users
   - Show stories to close friends only
   - Story privacy independent of account privacy

### Phase 4: Social Commerce (Q1 2027)

1. **Shoppable Profiles**:
   - Link menu items to profile
   - "Shop" tab on chef profiles
   - Direct ordering from social profile

2. **Affiliate Sharing**:
   - Track orders from shared reels
   - Revenue sharing for influencers
   - Affiliate links in shares

3. **Collaborative Content**:
   - Tag collaborators in reels (multiple chefs)
   - Co-authored content (appears on both profiles)
   - Split engagement metrics

---

## Appendix

### API Endpoints Summary

| Method | Endpoint | Description | Rate Limit |
|--------|----------|-------------|------------|
| POST | /follow/:targetId | Follow user | 30/min |
| POST | /unfollow/:targetId | Unfollow user | 30/min |
| POST | /requests/:requestId/accept | Accept follow request | - |
| POST | /requests/:requestId/reject | Reject follow request | - |
| GET | /relationship/:targetId | Get relationship state | - |
| POST | /remove-follower/:followerId | Remove follower | - |
| GET | /requests/pending | Get pending requests | - |
| GET | /followers | Get own followers | - |
| GET | /followers/:userIdOrUsername | Get user followers | - |
| GET | /following | Get own following | - |
| GET | /following/:userIdOrUsername | Get user following | - |
| POST | /block/:targetId | Block user | 20/min |
| POST | /unblock/:targetId | Unblock user | 20/min |
| GET | /blocked/check/:targetId | Check block status | - |
| GET | /blocked | Get blocked users | - |
| GET | /profile/privacy | Get privacy settings | - |
| POST | /profile/privacy | Update privacy settings | - |
| GET | /profile/:userId | Get social profile | - |
| GET | /search | Search users | - |
| POST | /reels/:reelId/share | Record share | 20/min |
| GET | /reels/:reelId/share/stats | Get share stats | - |
| GET | /reels/:reelId/share/targetable-users | Get targetable users | - |

### Database Schema Summary

**user_follow**:
```sql
CREATE TABLE user_follow (
  id UUID PRIMARY KEY,
  follower_id UUID NOT NULL REFERENCES users(id),
  target_id UUID NOT NULL REFERENCES users(id),
  status VARCHAR(10) NOT NULL CHECK (status IN ('accepted', 'pending')),
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
  UNIQUE (follower_id, target_id)
);
```

**user_privacy**:
```sql
CREATE TABLE user_privacy (
  id UUID PRIMARY KEY,
  user_id UUID NOT NULL UNIQUE REFERENCES users(id),
  is_private BOOLEAN NOT NULL DEFAULT FALSE,
  followers_count INT NOT NULL DEFAULT 0,
  following_count INT NOT NULL DEFAULT 0,
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);
```

**user_block**:
```sql
CREATE TABLE user_block (
  id UUID PRIMARY KEY,
  user_id UUID NOT NULL REFERENCES users(id),
  blocked_user_id UUID NOT NULL REFERENCES users(id),
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  UNIQUE (user_id, blocked_user_id)
);
```

### Error Codes

| Code | Description | HTTP Status |
|------|-------------|-------------|
| SOCIAL_TARGET_USER_NOT_FOUND | Target user not found | 404 |
| SOCIAL_ALREADY_FOLLOWING | Already following user | 400 |
| SOCIAL_NOT_FOLLOWING | Not following user | 400 |
| SOCIAL_SELF_FOLLOW_NOT_ALLOWED | Cannot follow yourself | 400 |
| SOCIAL_USER_BLOCKED | User is blocked | 403 |
| SOCIAL_REQUEST_NOT_FOUND | Follow request not found | 404 |
| SOCIAL_SELF_BLOCK_NOT_ALLOWED | Cannot block yourself | 400 |
| FOLLOW_RATE_LIMIT_EXCEEDED | Too many follow requests | 429 |
| UNFOLLOW_RATE_LIMIT_EXCEEDED | Too many unfollow requests | 429 |
| BLOCK_RATE_LIMIT_EXCEEDED | Too many block requests | 429 |
| SHARE_RATE_LIMIT_EXCEEDED | Too many shares | 429 |
| SHARE_REEL_NOT_FOUND | Reel not found | 404 |
| SHARE_TARGET_USER_REQUIRED | Target user required for direct share | 400 |
| SHARE_TARGET_USER_NOT_FOUND | Share target user not found | 404 |
| SHARE_SELF_NOT_ALLOWED | Cannot share to yourself | 400 |
| SHARE_USER_BLOCKED | Cannot share to blocked user | 403 |
| SHARE_UPDATE_FAILED | Failed to update share counters | 500 |

---

**[SLICE_COMPLETE ✅]**
