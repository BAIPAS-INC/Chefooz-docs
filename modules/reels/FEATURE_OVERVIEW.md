# Reels Module - Feature Overview

**Module**: `apps/chefooz-apis/src/modules/reels`  
**Purpose**: Short-form video content management and retrieval  
**Version**: 1.0  
**Last Updated**: 2026-02-14

---

## 📋 Executive Summary

The Reels module manages short-form video content on the Chefooz platform, handling reel retrieval, user engagement tracking, and content lifecycle management. It supports three distinct reel types (User Review, Promotional, Menu Showcase) with different monetization strategies and business purposes.

### Key Business Value

- **Content Discovery**: Powers explore, profile, and chef showcase experiences
- **Revenue Generation**: Links promotional reels to orders for creator commission tracking
- **Chef Marketing**: Enables menu showcase reels for direct product promotion
- **User Engagement**: Tracks views, likes, comments, saves for algorithmic ranking
- **Content Safety**: Implements soft deletion for compliance while hiding inappropriate content

---

## 🎯 Module Capabilities

### Core Features

1. **Reel Detail Retrieval**
   - Complete reel metadata with owner profile
   - Engagement statistics (views, likes, comments, saves)
   - Video variants for adaptive streaming
   - Linked order information for review reels
   - Personalized status (isLiked, isSaved) for authenticated users

2. **Reel Lifecycle Management**
   - Soft deletion (owner-initiated)
   - Content preservation for legal compliance
   - Automatic cache invalidation on deletion

3. **Menu Showcase Reels** (Chef-only)
   - Non-monetized product promotion
   - Multi-item linking (menu combos)
   - Feature flag enforcement
   - Chef ownership validation

4. **User Profile Reels**
   - Reverse chronological retrieval
   - Separate filtering for promotional vs menu reels
   - Paginated results with skip/limit
   - Avatar and user details injection

5. **Chef Menu Reels**
   - Dedicated endpoint for chef public page
   - Optimized for menu showcase strip
   - Recent-first ordering

---

## 🏗️ System Architecture

### Database Design

**Primary Storage**: MongoDB (NoSQL)
- **Collection**: `reels`
- **Schema**: Flexible document structure with embedded stats
- **Indexes**: Optimized for userId, createdAt, reelPurpose, deletedAt queries

**Secondary Storage**: PostgreSQL (Relational)
- **Tables**: `users`, `user_reputation_current`, `orders`, `chef_menu_items`
- **Purpose**: Owner profiles, reputation tiers, order linking, menu item validation

### Reel Types

```typescript
enum ReelPurpose {
  USER_REVIEW = 'USER_REVIEW',        // Has linkedOrderId, monetizable
  PROMOTIONAL = 'PROMOTIONAL',         // No order linking, organic content
  MENU_SHOWCASE = 'MENU_SHOWCASE'      // Chef menu items, non-monetizable
}
```

### Document Structure

```typescript
{
  _id: ObjectId,
  userId: string,                     // FK to users.id (Postgres)
  mediaId: string,                    // FK to Media document
  contentType: 'REEL' | 'POST',
  caption: string,
  hashtags: string[],
  taggedUserIds: string[],            // Caption mentions
  positionedTags: Array<{             // Instagram-style positioned tags
    userId: string,
    username: string,
    x: number,                        // 0-1 normalized
    y: number                         // 0-1 normalized
  }>,
  videoUrl: string,                   // HLS adaptive streaming URL
  thumbnailUrl: string,
  durationSec: number,
  linkedOrderId?: string,             // For USER_REVIEW reels
  creatorOrderValue?: number,         // Order total snapshot (paise)
  reelPurpose: string,
  linkedMenu?: {                      // For MENU_SHOWCASE reels
    chefId: string,
    menuItemIds: string[],
    estimatedPaise: number,
    previewImage: string
  },
  stats: {
    views: number,
    likes: number,
    comments: number,
    saves: number,
    savedCount: number,
    shareCount: number,
    copyLinkCount: number,
    externalShareCount: number,
    directShareCount: number,
    storyShareCount: number
  },
  coverTimestampSec?: number,         // User-selected thumbnail frame
  textOverlays?: Array<any>,          // FFmpeg burned-in text
  musicOverlay?: object,              // FFmpeg mixed audio
  filter?: object,                    // Image filter applied
  deletedAt?: Date,                   // Soft delete timestamp
  deletedBy?: string,                 // User who deleted (owner or admin)
  createdAt: Date,
  updatedAt: Date
}
```

---

## 🔗 Integration Points

### 1. Media Module
- **Dependency**: Reels are created during media upload processing
- **Flow**: Upload → Processing → Reel document creation
- **Fields**: `mediaId` links to Media document

### 2. User Module
- **Dependency**: Owner profile data (username, avatar, fullName)
- **Flow**: Reel detail API joins users table for author info
- **Fields**: `userId` FK to `users.id`

### 3. Reputation Module
- **Dependency**: Chef reputation tier display
- **Flow**: Reel detail API joins `user_reputation_current` table
- **Fields**: Maps reputation score to tier (Bronze/Silver/Gold/Platinum)

### 4. Order Module (Review Reels)
- **Dependency**: Order linking for creator commission tracking
- **Flow**: Reel created with `linkedOrderId` during post-order review flow
- **Fields**: `linkedOrderId` FK to `orders.id`, `creatorOrderValue` snapshot

### 5. Chef-Kitchen Module (Menu Reels)
- **Dependency**: Menu item validation and ownership checks
- **Flow**: Chef creates menu reel → validates menu items exist and belong to chef
- **Fields**: `linkedMenu.menuItemIds` FK to `chef_menu_items.id`

### 6. Cache Module (Valkey/Redis)
- **Dependency**: Like/save status retrieval
- **Flow**: Reel detail API checks Redis sets for viewer engagement
- **Keys**: `reel:{reelId}:likes`, `reel:{reelId}:saves`

### 7. Social Module
- **Dependency**: Follow relationships for feed personalization
- **Flow**: Explore/feed modules query reels and filter by social graph

### 8. Explore Module
- **Dependency**: Reel ranking and discovery algorithms
- **Flow**: Explore service queries reels with complex scoring
- **Purpose**: Trending, recommended, category-based feeds

---

## 📊 Business Rules

### Reel Purpose Rules

| Purpose | Order Linking | Monetization | Commission | Coins | Who Can Create |
|---------|---------------|--------------|------------|-------|----------------|
| **USER_REVIEW** | ✅ Required | ✅ Yes | ✅ Yes | ✅ Yes | Any user after order delivery |
| **PROMOTIONAL** | ❌ No | ❌ No | ❌ No | ❌ No | Any user |
| **MENU_SHOWCASE** | ❌ No (uses menuItemIds) | ❌ No | ❌ No | ❌ No | Chefs only |

### Soft Deletion Rules

1. **Ownership**: Only reel owner can delete their own content
2. **Preservation**: `deletedAt` timestamp set, document not removed
3. **Visibility**: Excluded from all public feeds and queries
4. **Compliance**: Preserved for legal/moderation review
5. **Cache Cleanup**: Like/save Redis sets cleared on deletion
6. **Admin Override**: Admins can delete any reel (future feature)

### Menu Reel Creation Rules

1. **Role Check**: User must have `role = 'chef'`
2. **Menu Item Validation**: All `menuItemIds` must exist in database
3. **Ownership Validation**: All menu items must belong to requesting chef
4. **Feature Flag**: `MENU_REELS` feature flag must be enabled
5. **Estimated Price**: Calculated from menu item prices (converted to paise)
6. **Preview Image**: First menu item's image used as reel preview

---

## 🎬 User Flows

### Flow 1: View Reel Detail (Customer)

```
1. User taps reel in feed/explore/profile
2. App calls GET /api/v1/reels/detail/:mediaId
3. Backend retrieves:
   - Reel document from MongoDB
   - Owner profile from Postgres (username, avatar)
   - Reputation tier from user_reputation_current
   - isLiked/isSaved from Redis cache (if authenticated)
   - Linked order data (if USER_REVIEW reel)
4. Response includes:
   - Reel metadata (caption, hashtags, stats)
   - Owner profile (username, avatar, reputation tier)
   - Video variants (HLS adaptive streaming)
   - Engagement status (isLiked, isSaved)
   - Linked order snapshot (if applicable)
5. App displays full-screen reel with:
   - Video player
   - Caption and hashtags
   - Action buttons (like, comment, save, share)
   - Owner profile chip
   - Linked product card (if review reel)
```

### Flow 2: Delete Own Reel (Owner)

```
1. User long-presses own reel or taps "..." menu
2. Selects "Delete" option
3. App shows confirmation dialog
4. User confirms deletion
5. App calls DELETE /api/v1/reels/:reelId
6. Backend validates:
   - JWT authentication
   - Reel exists and not already deleted
   - User is the owner (userId matches)
7. Backend soft deletes:
   - Sets deletedAt = current timestamp
   - Sets deletedBy = userId
   - Clears Redis cache (like/save sets)
8. Response returns success
9. App removes reel from UI
10. Reel hidden from all public feeds immediately
```

### Flow 3: Create Menu Reel (Chef)

```
1. Chef uploads video via media upload flow
2. Chef navigates to menu management
3. Selects menu items to showcase (e.g., combo deal)
4. Adds caption (optional)
5. Taps "Create Menu Reel"
6. App calls POST /api/v1/reels/menu
   Body: {
     menuItemIds: ["uuid-1", "uuid-2"],
     caption: "Special combo: Dal + Naan + Raita ₹299!"
   }
7. Backend validates:
   - Feature flag MENU_REELS enabled
   - User role is 'chef'
   - Menu items exist in database
   - Chef owns all menu items
8. Backend creates reel document:
   - reelPurpose = 'MENU_SHOWCASE'
   - linkedMenu = { chefId, menuItemIds, estimatedPaise, previewImage }
   - stats initialized to zeros
9. Response returns reelId
10. App shows success toast
11. Menu reel appears on chef's public page
```

### Flow 4: Browse Chef Menu Reels (Customer)

```
1. Customer visits chef public page
2. App calls GET /api/v1/reels/chef/:chefId/menu
3. Backend queries:
   - reels collection
   - Filter: linkedMenu.chefId = chefId, reelPurpose = 'MENU_SHOWCASE', deletedAt = null
   - Sort: createdAt DESC
   - Limit: 20
4. Response returns array of menu reels:
   - reelId, videoUrl, thumbnailUrl
   - caption, menuItemIds
   - estimatedPaise (for display)
5. App displays horizontal scrollable strip:
   - Video thumbnails with duration badge
   - Caption preview
   - Estimated price
6. User taps reel → plays in full-screen viewer
7. Reel shows "Add to Cart" button
8. Tapping button adds all linked menu items to cart
```

---

## 🔒 Security & Privacy

### Authentication

- **GET /detail/:mediaId**: Optional JWT (personalizes isLiked/isSaved)
- **DELETE /:reelId**: Required JWT + ownership validation
- **POST /menu**: Required JWT + chef role validation
- **GET /user/:userId**: Optional JWT (future: private profile enforcement)
- **GET /chef/:chefId/menu**: No auth required (public showcase)

### Authorization Rules

1. **Delete Reel**: Only owner can delete (userId === reel.userId)
2. **Create Menu Reel**: Only users with `role = 'chef'`
3. **Menu Item Ownership**: Chef must own all menuItemIds

### Privacy Considerations

- Soft-deleted reels excluded from all queries via `deletedAt: null` filter
- Blocked users excluded in feed/explore modules (not enforced in reels module directly)
- Private account content filtering handled by Social module (future enhancement)

---

## 📈 Performance & Scalability

### Database Indexes

```javascript
// Compound indexes for optimized queries
reels.createIndex({ userId: 1, createdAt: -1 })                      // User profile reels
reels.createIndex({ reelPurpose: 1, createdAt: -1 })                 // Purpose-based feeds
reels.createIndex({ 'linkedMenu.chefId': 1, createdAt: -1 })         // Chef menu reels
reels.createIndex({ deletedAt: 1, userId: 1 })                       // Soft delete filtering
reels.createIndex({ deletedAt: 1, reelPurpose: 1, createdAt: -1 })   // Purpose + soft delete
reels.createIndex({ createdAt: -1 })                                 // Default scroll
reels.createIndex({ 'stats.likes': -1, createdAt: -1 })              // Trending
reels.createIndex({ promoted: 1, createdAt: -1 })                    // Promoted content
reels.createIndex({ location: '2dsphere' })                          // Geospatial queries
```

### Caching Strategy

**Redis Keys**:
- `reel:{reelId}:likes` → Redis Set of userIds who liked
- `reel:{reelId}:saves` → Redis Set of userIds who saved

**Cache Invalidation**:
- On reel deletion: Clear like/save sets
- On like/unlike: Add/remove userId from like set
- On save/unsave: Add/remove userId from save set

**Cache TTL**:
- Like/save sets: No expiration (persistent engagement)

### Query Optimization

**User Profile Reels**:
```javascript
db.reels.find({
  userId: "uuid",
  reelPurpose: { $in: ['PROMOTIONAL', 'USER_REVIEW'] },
  deletedAt: null
}).sort({ createdAt: -1 }).skip(0).limit(50)
```
- **Index Used**: `{ userId: 1, createdAt: -1 }`
- **Performance**: <10ms for 10K reels per user

**Chef Menu Reels**:
```javascript
db.reels.find({
  'linkedMenu.chefId': "uuid",
  reelPurpose: 'MENU_SHOWCASE',
  deletedAt: null
}).sort({ createdAt: -1 }).limit(20)
```
- **Index Used**: `{ 'linkedMenu.chefId': 1, createdAt: -1 }`
- **Performance**: <15ms for 1K menu reels per chef

### Scalability Considerations

1. **Horizontal Scaling**: MongoDB sharding by userId (future)
2. **Read Replicas**: Direct read-heavy queries to replicas
3. **CDN Caching**: Video URLs served via CloudFront CDN
4. **Lazy Loading**: Owner profile data joined only on demand (not pre-cached)

---

## 🧪 Feature Flags

### MENU_REELS

**Status**: Production-ready (Phase 3.3.2)  
**Purpose**: Enable/disable menu reel creation feature

**Enforcement**:
```typescript
if (!canUploadMenuReel('MENU_SHOWCASE', 'chef', envConfig)) {
  throw new HttpException({
    success: false,
    message: 'Menu reels feature is currently disabled',
    errorCode: 'FEATURE_DISABLED_MENU_REELS',
  }, HttpStatus.FORBIDDEN);
}
```

**Rollout Strategy**:
- **Alpha**: 10 selected chefs (manual enable)
- **Beta**: All chefs with >100 followers
- **GA**: All chefs globally

---

## 📊 Analytics & Metrics

### Key Metrics Tracked

**Reel-Level Metrics** (stored in `stats` subdocument):
- `views`: Total view count (incremented on play)
- `likes`: Total likes (updated via Social module)
- `comments`: Total comments (updated via Comments module)
- `saves`: Total saves (updated via Collections module)
- `savedCount`: Duplicate of saves (legacy field)
- `shareCount`: Total shares (all types combined)
- `copyLinkCount`: Share via link copy
- `externalShareCount`: Share to external apps (WhatsApp, Instagram)
- `directShareCount`: Share to Chefooz DMs
- `storyShareCount`: Share to Chefooz stories

**Business Metrics** (calculated):
- Conversion rate: `linkedOrders / views` (for menu reels)
- Engagement rate: `(likes + comments + saves) / views`
- Retention rate: `shares / views`

### Monitoring Queries

```javascript
// Top performing reels by engagement rate
db.reels.aggregate([
  { $match: { deletedAt: null } },
  { $project: {
      userId: 1,
      caption: 1,
      engagementRate: {
        $divide: [
          { $add: ['$stats.likes', '$stats.comments', '$stats.saves'] },
          { $max: ['$stats.views', 1] }
        ]
      }
  }},
  { $sort: { engagementRate: -1 } },
  { $limit: 100 }
])

// Menu reel performance by chef
db.reels.aggregate([
  { $match: { reelPurpose: 'MENU_SHOWCASE', deletedAt: null } },
  { $group: {
      _id: '$linkedMenu.chefId',
      totalReels: { $sum: 1 },
      totalViews: { $sum: '$stats.views' },
      avgViews: { $avg: '$stats.views' }
  }},
  { $sort: { totalViews: -1 } }
])
```

---

## 🚨 Error Handling

### Common Error Codes

| Error Code | HTTP Status | Trigger | User-Facing Message |
|------------|-------------|---------|---------------------|
| `REEL_NOT_FOUND` | 404 | Reel doesn't exist or soft-deleted | "This reel is no longer available" |
| `UNAUTHORIZED` | 401 | Missing JWT token | "Please log in to continue" |
| `CHEF_ONLY` | 403 | Non-chef tries to create menu reel | "Only chefs can create menu reels" |
| `INVALID_MENU_ITEMS` | 400 | Menu items not found or don't belong to chef | "Some menu items are invalid" |
| `FEATURE_DISABLED_MENU_REELS` | 403 | Feature flag disabled | "Menu reels feature is currently unavailable" |
| `REEL_DELETE_ERROR` | 500 | Database/cache error during deletion | "Failed to delete reel. Please try again" |
| `USER_REELS_FETCH_ERROR` | 500 | Database error fetching user reels | "Failed to load reels. Please try again" |

### Error Response Format

```json
{
  "success": false,
  "message": "Human-readable error message",
  "errorCode": "MACHINE_READABLE_CODE"
}
```

---

## 🔄 Future Enhancements

### Planned Features

1. **Admin Moderation**
   - Admin override for reel deletion
   - Moderation queue for reported reels
   - Automated content filtering integration

2. **Private Account Support**
   - Hide reels from non-followers if account is private
   - Integration with UserPrivacy module

3. **Reel Analytics Dashboard**
   - Detailed engagement breakdown
   - Audience demographics
   - Peak viewing times

4. **Advanced Menu Reels**
   - Multi-reel carousel for combo deals
   - Interactive product tags (tap to view details)
   - Dynamic pricing (real-time availability)

5. **Geospatial Features**
   - "Reels Near You" based on location
   - Location-tagged reels on map view

6. **Reel Series/Playlists**
   - Group related reels (e.g., recipe series)
   - Auto-play next reel in series

---

## 📚 Related Documentation

- **Technical Guide**: `TECHNICAL_GUIDE.md` (API specs, database schema, code examples)
- **QA Test Cases**: `QA_TEST_CASES.md` (Test scenarios, automation coverage)
- **Media Module**: `docs/modules/media/FEATURE_OVERVIEW.md` (Upload flow)
- **Media-Processing Module**: `docs/modules/media-processing/FEATURE_OVERVIEW.md` (Video transcoding)
- **Explore Module**: `docs/modules/explore/FEATURE_OVERVIEW.md` (Discovery algorithms)

---

## 🏷️ Glossary

- **HLS**: HTTP Live Streaming, adaptive bitrate video protocol
- **Soft Delete**: Marking content as deleted without removing from database
- **Creator Commission**: Revenue share for users who create promotional content
- **Menu Reel**: Non-monetized reel showcasing chef's menu items
- **Review Reel**: Monetized reel linked to a completed order
- **Promotional Reel**: Organic content without order linking
- **CDN**: Content Delivery Network for fast global video delivery
- **Valkey/Redis**: In-memory cache for like/save status
- **Reputation Tier**: User ranking based on engagement and quality (Bronze/Silver/Gold/Platinum)

---

## 📞 Support Contacts

- **Backend Team**: backend@chefooz.com
- **Content Moderation**: moderation@chefooz.com
- **Product Manager**: pm@chefooz.com

---

**Document Version**: 1.0  
**Last Reviewed**: 2026-02-14  
**Next Review**: 2026-03-14
