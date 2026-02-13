# Profile Module – Technical Guide

**Document Type:** Developer Reference  
**Module:** Profile Management  
**Last Updated:** February 14, 2026  
**Audience:** Backend Engineers, Mobile Developers, DevOps

---

## 📋 Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Database Schema](#2-database-schema)
3. [API Reference](#3-api-reference)
4. [Service Layer](#4-service-layer)
5. [Data Flow](#5-data-flow)
6. [Caching Strategy](#6-caching-strategy)
7. [Security & Authentication](#7-security--authentication)
8. [Error Handling](#8-error-handling)
9. [Integration Points](#9-integration-points)
10. [Code Examples](#10-code-examples)
11. [Testing Guide](#11-testing-guide)
12. [Deployment & Operations](#12-deployment--operations)

---

## 1. Architecture Overview

### 1.1 Module Structure

```
apps/chefooz-apis/src/modules/profile/
├── dto/
│   ├── activate-chef-profile.dto.ts
│   ├── chef-profile-response.dto.ts
│   ├── edit-profile.dto.ts
│   ├── get-profile.dto.ts
│   ├── signed-url.dto.ts
│   ├── update-chef-profile.dto.ts
│   └── username-check.dto.ts
├── profile.controller.ts         # User profile endpoints
├── profile-roles.controller.ts   # Chef profile endpoints
├── profile.service.ts             # User profile logic
├── profile-roles.service.ts       # Chef profile logic
├── profile.mapper.ts              # DTO mapping
└── profile.module.ts              # NestJS module
```

---

### 1.2 Technology Stack

| **Component** | **Technology** | **Purpose** |
|--------------|---------------|-------------|
| Backend Framework | NestJS 10+ | API layer + dependency injection |
| Primary Database | PostgreSQL | Transactional data (users, chef profiles, reviews) |
| NoSQL Database | MongoDB | Reels metadata |
| Cache Layer | Valkey/Redis | Profile projection caching (5-min TTL) |
| Storage | AWS S3 | Profile images (avatar/cover) |
| CDN | CloudFront | Image delivery |
| Authentication | JWT (7-day tokens) | Secured endpoints |

---

### 1.3 Dependencies

**Internal Modules:**
- `auth` → JWT guards (`JwtAuthGuard`, `OptionalJwtAuthGuard`)
- `social` → Follow/block relationships (`UserFollow`, `UserPrivacy`, `UserBlock`)
- `cache` → Valkey/Redis service (`CacheService`)
- `media` → Reel metadata (`Media` schema)
- `reels` → Reel content (`Reel` schema)
- `reviews` → Order reviews (`OrderReview` entity)

**External Libraries:**
- `@aws-sdk/client-s3` → S3 client
- `@aws-sdk/s3-request-presigner` → Presigned URL generation
- `class-validator` → DTO validation
- `class-transformer` → DTO transformation

---

## 2. Database Schema

### 2.1 User Entity (PostgreSQL)

**Table:** `user`

```sql
CREATE TABLE "user" (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  phone VARCHAR(15) NOT NULL UNIQUE,
  username VARCHAR(30) UNIQUE,
  full_name VARCHAR(50),
  display_name VARCHAR(50),
  avatar_url TEXT,
  cover_url TEXT,
  bio TEXT,
  location VARCHAR(100),
  website TEXT,
  pronouns VARCHAR(50),
  public_email VARCHAR(255),
  phone_verified BOOLEAN DEFAULT FALSE,
  profile_visibility VARCHAR(10) DEFAULT 'public', -- 'public' | 'private'
  role VARCHAR(10) DEFAULT 'user', -- 'user' | 'chef'
  coins INTEGER DEFAULT 0,
  reputation_score INTEGER DEFAULT 0,
  last_username_change_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_user_username ON "user"(username);
CREATE INDEX idx_user_phone ON "user"(phone);
CREATE INDEX idx_user_role ON "user"(role);
```

**Key Fields:**
- `username`: Unique handle (3-30 chars, lowercase alphanumeric + `_` `.`)
- `profile_visibility`: `'public'` or `'private'`
- `last_username_change_at`: Tracks 15-day cooldown
- `role`: Informational (does not enforce permissions)

---

### 2.2 ChefProfile Entity (PostgreSQL)

**Table:** `chef_profile`

```sql
CREATE TABLE chef_profile (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID NOT NULL REFERENCES "user"(id) ON DELETE CASCADE,
  business_name VARCHAR(100) NOT NULL,
  description TEXT,
  address TEXT,
  cuisines TEXT[], -- Array of strings
  delivery_details JSONB, -- { radiusKm: number, isPickupAllowed: boolean }
  is_active BOOLEAN DEFAULT TRUE,
  verification_status VARCHAR(20) DEFAULT 'pending', -- 'pending' | 'verified' | 'rejected'
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  
  CONSTRAINT chef_profile_user_id_unique UNIQUE(user_id)
);

CREATE INDEX idx_chef_profile_user_id ON chef_profile(user_id);
CREATE INDEX idx_chef_profile_verification_status ON chef_profile(verification_status);
```

**Key Fields:**
- `user_id`: One-to-one relationship with `user`
- `delivery_details`: JSON object with delivery radius and pickup options
- `verification_status`: Admin-controlled approval state

---

### 2.3 Social Graph Entities (PostgreSQL)

**UserFollow Table:**
```sql
CREATE TABLE user_follow (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  follower_id UUID NOT NULL REFERENCES "user"(id),
  target_id UUID NOT NULL REFERENCES "user"(id),
  status VARCHAR(20) DEFAULT 'accepted', -- 'accepted' | 'pending'
  created_at TIMESTAMP DEFAULT NOW(),
  
  CONSTRAINT user_follow_unique UNIQUE(follower_id, target_id)
);

CREATE INDEX idx_user_follow_follower ON user_follow(follower_id);
CREATE INDEX idx_user_follow_target ON user_follow(target_id);
CREATE INDEX idx_user_follow_status ON user_follow(status);
```

**UserBlock Table:**
```sql
CREATE TABLE user_block (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID NOT NULL REFERENCES "user"(id),
  blocked_user_id UUID NOT NULL REFERENCES "user"(id),
  created_at TIMESTAMP DEFAULT NOW(),
  
  CONSTRAINT user_block_unique UNIQUE(user_id, blocked_user_id)
);

CREATE INDEX idx_user_block_user ON user_block(user_id);
CREATE INDEX idx_user_block_blocked_user ON user_block(blocked_user_id);
```

**UserPrivacy Table:**
```sql
CREATE TABLE user_privacy (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID NOT NULL UNIQUE REFERENCES "user"(id),
  is_private BOOLEAN DEFAULT FALSE,
  followers_count INTEGER DEFAULT 0,
  following_count INTEGER DEFAULT 0,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_user_privacy_user_id ON user_privacy(user_id);
```

---

### 2.4 Reel Schema (MongoDB)

**Collection:** `reels`

```typescript
interface Reel {
  _id: ObjectId;
  userId: string; // User UUID
  videoUrl: string; // S3 URI (s3://bucket/key)
  thumbnailUrl: string; // S3 URI
  durationSec: number;
  reelPurpose: 'promotional' | 'exploratory' | null;
  linkedOrderId: string | null;
  stats: {
    views: number;
    likes: number;
    shares: number;
  };
  createdAt: Date;
  updatedAt: Date;
  deletedAt: Date | null; // Soft delete
}
```

**Key Indexes:**
```javascript
db.reels.createIndex({ userId: 1, createdAt: -1 });
db.reels.createIndex({ deletedAt: 1 }); // For filtering soft-deleted
```

---

### 2.5 OrderReview Entity (PostgreSQL)

**Table:** `order_review`

```sql
CREATE TABLE order_review (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID NOT NULL REFERENCES "user"(id),
  order_item_id UUID NOT NULL,
  overall FLOAT NOT NULL, -- 1.0 - 5.0
  comment TEXT,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_order_review_user_id ON order_review(user_id);
CREATE INDEX idx_order_review_created_at ON order_review(created_at);
```

---

## 3. API Reference

### 3.1 Profile Management Endpoints

#### 3.1.1 Get Own Profile

```http
GET /api/v1/profile/me
Authorization: Bearer <jwt>
```

**Response (200 OK):**
```json
{
  "success": true,
  "message": "Profile retrieved",
  "data": {
    "userId": "550e8400-e29b-41d4-a716-446655440000",
    "username": "chef_john",
    "fullName": "John Doe",
    "displayName": "Chef John",
    "avatarUrl": "https://cdn.chefooz.com/profiles/user123/avatar-1733659200.webp",
    "coverUrl": "https://cdn.chefooz.com/profiles/user123/cover-1733659200.webp",
    "bio": "Professional chef specializing in Italian cuisine 🍝",
    "location": "New York, USA",
    "website": "https://chefjohn.com",
    "pronouns": "he/him",
    "publicEmail": "chef@example.com",
    "phoneVerified": true,
    "profileVisibility": "public",
    "role": "user",
    "isPrivate": false,
    "verified": false,
    "badges": [],
    "reputationScore": 0,
    "coinsBalance": 150,
    "followersCount": 1250,
    "followingCount": 340,
    "reelsCount": 45,
    "reviewsCount": 78,
    "isFollowing": false,
    "isFollowedBy": false,
    "isRequested": false,
    "isBlocked": false,
    "isChef": true,
    "kitchenName": "The Golden Spoon",
    "fssaiNumber": "12345678901234",
    "createdAt": "2025-11-28T10:00:00Z"
  }
}
```

---

#### 3.1.2 Update Profile

```http
PATCH /api/v1/profile/me
Authorization: Bearer <jwt>
Content-Type: application/json
```

**Request Body:**
```json
{
  "username": "chef_john_new",
  "displayName": "Chef John",
  "fullName": "John Doe",
  "bio": "Updated bio",
  "website": "https://chefjohn.com",
  "location": "New York, USA",
  "pronouns": "he/him",
  "avatarKey": "profiles/user123/avatar-1733659200.webp",
  "coverKey": "profiles/user123/cover-1733659200.webp",
  "profileVisibility": "private",
  "publicEmail": "chef@example.com",
  "chefName": "The Golden Spoon Bistro",
  "fssaiNumber": "12345678901234"
}
```

**Response (200 OK):**
```json
{
  "success": true,
  "message": "Profile updated successfully",
  "data": { /* Full ProfileProjection */ }
}
```

**Error (409 Conflict - Username Taken):**
```json
{
  "success": false,
  "message": "Username already taken",
  "errorCode": "CONFLICT"
}
```

**Error (400 Bad Request - Username Cooldown):**
```json
{
  "success": false,
  "message": "You can change your username again in 10 days.",
  "errorCode": "USERNAME_CHANGE_COOLDOWN",
  "data": {
    "daysLeft": 10
  }
}
```

---

#### 3.1.3 Check Username Availability

```http
GET /api/v1/profile/username-check?username=chef_john
Authorization: Bearer <jwt> (optional)
```

**Response (200 OK - Available):**
```json
{
  "success": true,
  "message": "Username availability checked",
  "data": {
    "available": true
  }
}
```

**Response (200 OK - Taken):**
```json
{
  "success": true,
  "message": "Username availability checked",
  "data": {
    "available": false,
    "suggestions": [
      "chef_john1",
      "chef_john2",
      "chef_john3",
      "chef_john_2026",
      "chef_john_2025"
    ]
  }
}
```

---

#### 3.1.4 Get Avatar Signed URL

```http
POST /api/v1/profile/avatar/signed-url
Authorization: Bearer <jwt>
Content-Type: application/json
```

**Request Body:**
```json
{
  "filename": "profile-pic.jpg",
  "contentType": "image/jpeg"
}
```

**Response (200 OK):**
```json
{
  "success": true,
  "message": "Avatar upload URL generated",
  "data": {
    "uploadUrl": "https://chefooz-media-output.s3.ap-south-1.amazonaws.com/profiles/user123/avatar-1733659200.webp?X-Amz-...",
    "s3Key": "profiles/user123/avatar-1733659200.webp",
    "cdnUrl": "https://cdn.chefooz.com/profiles/user123/avatar-1733659200.webp",
    "expiresIn": 900
  }
}
```

**Upload Flow:**
1. Client calls `POST /avatar/signed-url`
2. Client uploads image to `uploadUrl` via `PUT` request
3. Client sends `s3Key` to `PATCH /profile/me` with `avatarKey` field

---

#### 3.1.5 Get Cover Signed URL

```http
POST /api/v1/profile/cover/signed-url
Authorization: Bearer <jwt>
Content-Type: application/json
```

**Request/Response:** Same as avatar (see 3.1.4)

---

#### 3.1.6 Get Profile by Username

```http
GET /api/v1/profile/:username
Authorization: Bearer <jwt>
```

**Response (200 OK - Public Profile):**
```json
{
  "success": true,
  "message": "Profile retrieved",
  "data": { /* Full ProfileProjection */ }
}
```

**Response (200 OK - Locked Private Profile):**
```json
{
  "success": true,
  "message": "Profile retrieved",
  "data": {
    "userId": "550e8400-e29b-41d4-a716-446655440000",
    "username": "private_user",
    "fullName": "Jane Doe",
    "displayName": "Jane",
    "avatarUrl": "https://cdn.chefooz.com/profiles/user456/avatar.webp",
    "coverUrl": null,
    "bio": null,
    "location": null,
    "website": null,
    "pronouns": null,
    "publicEmail": null,
    "phoneVerified": false,
    "profileVisibility": "private",
    "isPrivate": true,
    "verified": false,
    "badges": [],
    "reputationScore": 0,
    "coinsBalance": 0,
    "followersCount": 500,
    "followingCount": 200,
    "reelsCount": 0,
    "reviewsCount": 0,
    "isFollowing": false,
    "isFollowedBy": false,
    "isRequested": false,
    "isBlocked": false,
    "isChef": false,
    "kitchenName": null,
    "fssaiNumber": null,
    "createdAt": "2025-10-15T08:00:00Z"
  }
}
```

**Error (404 Not Found - User Not Found or Blocked):**
```json
{
  "success": false,
  "message": "User @unknown_user not found",
  "errorCode": "NOT_FOUND"
}
```

---

#### 3.1.7 Get Profile Status

```http
GET /api/v1/profile/:username/status
Authorization: Bearer <jwt>
```

**Response (200 OK):**
```json
{
  "success": true,
  "message": "Status retrieved",
  "data": {
    "isFollowing": true,
    "isFollowedBy": false,
    "isRequested": false,
    "isBlocked": false,
    "isPrivate": false
  }
}
```

---

#### 3.1.8 Get User Reels

```http
GET /api/v1/profile/:username/reels?limit=20&cursor=2024-01-01T00:00:00.000Z
Authorization: Bearer <jwt>
```

**Response (200 OK):**
```json
{
  "success": true,
  "message": "Reels retrieved",
  "data": {
    "reels": [
      {
        "id": "507f1f77bcf86cd799439011",
        "thumbnailUrl": "https://cdn.chefooz.com/reels/user123/thumb.webp",
        "playbackUrl": "https://cdn.chefooz.com/reels/user123/video.mp4",
        "duration": 15,
        "viewCount": 1250,
        "likeCount": 340,
        "createdAt": "2025-11-28T10:00:00Z",
        "linkedOrderId": null,
        "reelPurpose": "exploratory"
      }
    ],
    "nextCursor": "2025-11-20T08:00:00Z"
  }
}
```

**Error (400 Bad Request - Private Account):**
```json
{
  "success": false,
  "message": "This account is private",
  "errorCode": "PRIVATE_ACCOUNT"
}
```

---

#### 3.1.9 Get User Reviews

```http
GET /api/v1/profile/:username/reviews?limit=20&cursor=2024-01-01T00:00:00.000Z
Authorization: Bearer <jwt>
```

**Response (200 OK):**
```json
{
  "success": true,
  "message": "Reviews retrieved",
  "data": {
    "reviews": [
      {
        "id": "550e8400-e29b-41d4-a716-446655440001",
        "rating": 4.5,
        "comment": "Amazing pasta!",
        "orderId": "550e8400-e29b-41d4-a716-446655440002",
        "createdAt": "2025-11-28T10:00:00Z"
      }
    ],
    "nextCursor": "2025-11-20T08:00:00Z"
  }
}
```

---

### 3.2 Chef Profile Endpoints

#### 3.2.1 Activate Chef Profile

```http
POST /api/v1/profile/chef/activate
Authorization: Bearer <jwt>
Content-Type: application/json
```

**Request Body:**
```json
{
  "businessName": "The Golden Spoon",
  "description": "Authentic Italian cuisine",
  "address": "123 Main St, New York",
  "cuisines": ["Italian", "Continental"],
  "radiusKm": 10,
  "isPickupAllowed": true
}
```

**Response (201 Created):**
```json
{
  "success": true,
  "message": "Chef profile activated successfully",
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "userId": "550e8400-e29b-41d4-a716-446655440001",
    "businessName": "The Golden Spoon",
    "isActive": true,
    "verificationStatus": "pending",
    "description": "Authentic Italian cuisine",
    "address": "123 Main St, New York",
    "cuisines": ["Italian", "Continental"],
    "deliveryDetails": {
      "radiusKm": 10,
      "isPickupAllowed": true
    },
    "createdAt": "2025-11-28T10:00:00Z",
    "updatedAt": "2025-11-28T10:00:00Z"
  }
}
```

**Error (409 Conflict - Already Exists):**
```json
{
  "success": false,
  "message": "Chef profile already activated for this user",
  "errorCode": "CHEF_PROFILE_EXISTS"
}
```

---

#### 3.2.2 Update Chef Profile

```http
PATCH /api/v1/profile/chef
Authorization: Bearer <jwt>
Content-Type: application/json
```

**Request Body:**
```json
{
  "businessName": "The Golden Spoon Bistro",
  "description": "Updated description",
  "cuisines": ["Italian", "Continental", "Mediterranean"],
  "isActive": true,
  "radiusKm": 15
}
```

**Response (200 OK):**
```json
{
  "success": true,
  "message": "Chef profile updated successfully",
  "data": { /* Full ChefProfileResponseDto */ }
}
```

**Error (404 Not Found):**
```json
{
  "success": false,
  "message": "Chef profile not found",
  "errorCode": "CHEF_PROFILE_NOT_FOUND"
}
```

---

#### 3.2.3 Get My Chef Profile

```http
GET /api/v1/profile/chef/me
Authorization: Bearer <jwt>
```

**Response (200 OK):**
```json
{
  "success": true,
  "message": "Chef profile retrieved successfully",
  "data": { /* Full ChefProfileResponseDto */ }
}
```

**Response (200 OK - No Chef Profile):**
```json
{
  "success": true,
  "message": "No chef profile found",
  "data": null
}
```

---

#### 3.2.4 Get Profile Role

```http
GET /api/v1/profile/role
Authorization: Bearer <jwt>
```

**Response (200 OK):**
```json
{
  "success": true,
  "message": "Profile role retrieved successfully",
  "data": {
    "isChef": true,
    "activeProfile": "user"
  }
}
```

---

## 4. Service Layer

### 4.1 ProfileService (profile.service.ts)

#### 4.1.1 Core Methods

**getProfile(username, viewerId)**
```typescript
/**
 * Get profile by username
 * Respects privacy and blocking rules
 * 
 * @param username - Username to fetch
 * @param viewerId - ID of viewing user
 * @returns ProfileProjection or Partial<ProfileProjection> (locked)
 * @throws NotFoundException if user not found or blocked
 */
async getProfile(
  username: string, 
  viewerId: string
): Promise<ProfileProjection | Partial<ProfileProjection>>
```

**Logic Flow:**
1. Check cache (`profile:projection:${username}`)
2. Fetch user from DB
3. Check if viewer is blocked → throw 404
4. Get social state (following, blocked, privacy)
5. If private + not following → return locked projection
6. Else return full projection + cache

---

**getMyProfile(userId)**
```typescript
/**
 * Get own profile (full access, no privacy filtering)
 * 
 * @param userId - User ID
 * @returns ProfileProjection
 * @throws NotFoundException if user not found
 */
async getMyProfile(userId: string): Promise<ProfileProjection>
```

---

**updateProfile(userId, dto)**
```typescript
/**
 * Update profile
 * Validates username uniqueness, resolves S3 keys to CDN URLs
 * 
 * @param userId - User ID
 * @param dto - EditProfileDto
 * @returns ProfileProjection
 * @throws BadRequestException if username cooldown active
 * @throws ConflictException if username taken
 */
async updateProfile(
  userId: string, 
  dto: EditProfileDto
): Promise<ProfileProjection>
```

**Logic Flow:**
1. Fetch user from DB
2. **Username change validation:**
   - Check cooldown: `lastUsernameChangeAt + 15 days > now`
   - Check uniqueness (case-insensitive)
3. **Image handling:**
   - Convert `avatarKey` → `avatarUrl`: `${CDN_URL}/${avatarKey}`
   - Convert `coverKey` → `coverUrl`: `${CDN_URL}/${coverKey}`
4. **Chef profile updates:**
   - If `chefName` or `fssaiNumber` provided → update `ChefProfile`
5. **Cache invalidation:**
   - Delete `profile:projection:${username}`
6. Return updated profile

---

**checkUsernameAvailability(username, currentUserId)**
```typescript
/**
 * Check username availability and generate suggestions
 * 
 * @param username - Username to check
 * @param currentUserId - Optional: ID of current user (allows self-check)
 * @returns UsernameCheckResponseDto
 */
async checkUsernameAvailability(
  username: string, 
  currentUserId?: string
): Promise<UsernameCheckResponseDto>
```

**Suggestion Algorithm:**
```typescript
// Numeric suffixes
for (let i = 1; i <= 5; i++) {
  const suggestion = `${username}${i}`;
  // Check availability, add to suggestions
}

// Year-based suffixes
const currentYear = new Date().getFullYear();
for (let i = 0; i < 2; i++) {
  const suggestion = `${username}_${currentYear - i}`;
  // Check availability, add to suggestions
}

return { available: false, suggestions: suggestions.slice(0, 5) };
```

---

**getAvatarSignedUrl(userId, dto)**
```typescript
/**
 * Generate presigned S3 URL for avatar upload
 * 
 * @param userId - User ID
 * @param dto - GetAvatarSignedUrlDto { filename, contentType }
 * @returns ProfileImageSignedUrlResponseDto
 */
async getAvatarSignedUrl(
  userId: string, 
  dto: GetAvatarSignedUrlDto
): Promise<ProfileImageSignedUrlResponseDto>
```

**S3 Key Format:** `profiles/{userId}/avatar-{timestamp}.{ext}`  
**Max Size:** 2MB (enforced client-side)  
**Presigned URL Expiry:** 15 minutes (900 seconds)

---

**getUserReels(username, viewerId, limit, cursor)**
```typescript
/**
 * Get user's reels (paginated)
 * 
 * @param username - Username
 * @param viewerId - Viewing user ID
 * @param limit - Items per page (max 50)
 * @param cursor - ISO date string for pagination
 * @returns { reels, nextCursor }
 * @throws BadRequestException if account private + not following
 */
async getUserReels(
  username: string, 
  viewerId: string, 
  limit: number, 
  cursor?: string
)
```

**MongoDB Query:**
```typescript
const query: any = { 
  userId: user.id,
  deletedAt: null, // CRITICAL: Exclude soft-deleted reels
};
if (cursor) {
  query.createdAt = { $lt: new Date(cursor) };
}

const reels = await this.reelModel
  .find(query)
  .sort({ createdAt: -1 })
  .limit(limit + 1)
  .exec();
```

**Privacy Check:**
```typescript
const socialState = await this.getSocialState(user.id, viewerId);
if (socialState.isPrivate && !socialState.isFollowing && user.id !== viewerId) {
  throw new BadRequestException('This account is private');
}
```

---

#### 4.1.2 Helper Methods

**getSocialState(targetId, viewerId)**
```typescript
/**
 * Get social graph state between two users
 * 
 * @param targetId - Target user ID
 * @param viewerId - Viewing user ID
 * @returns Social state object
 */
private async getSocialState(targetId: string, viewerId: string) {
  const [isFollowingRow, isFollowedByRow, isRequestedRow, isBlockedRow, privacy] = 
    await Promise.all([
      this.followRepo.findOne({
        where: { followerId: viewerId, targetId, status: 'accepted' },
      }),
      this.followRepo.findOne({
        where: { followerId: targetId, targetId: viewerId, status: 'accepted' },
      }),
      this.followRepo.findOne({
        where: { followerId: viewerId, targetId, status: 'pending' },
      }),
      this.blockRepo.findOne({
        where: [
          { userId: targetId, blockedUserId: viewerId },
          { userId: viewerId, blockedUserId: targetId },
        ],
      }),
      this.privacyRepo.findOne({ where: { userId: targetId } }),
    ]);

  return {
    isFollowing: !!isFollowingRow,
    isFollowedBy: !!isFollowedByRow,
    isRequested: !!isRequestedRow,
    isBlocked: !!isBlockedRow,
    isPrivate: privacy?.isPrivate ?? false,
  };
}
```

---

**getCounts(userId)**
```typescript
/**
 * Get aggregated counts (followers, following, reels, reviews)
 * 
 * @param userId - User ID
 * @returns Counts object
 */
private async getCounts(userId: string) {
  const [privacy, reelsCount, reviewsCount] = await Promise.all([
    this.privacyRepo.findOne({ where: { userId } }),
    this.mediaModel.countDocuments({ userId, status: 'ready' }),
    this.reviewRepo.count({ where: { userId } }),
  ]);

  return {
    followersCount: privacy?.followersCount ?? 0,
    followingCount: privacy?.followingCount ?? 0,
    reelsCount,
    reviewsCount,
  };
}
```

---

**isUserBlocked(targetId, viewerId)**
```typescript
/**
 * Check if user is blocked (bidirectional)
 * 
 * @param targetId - Target user ID
 * @param viewerId - Viewing user ID
 * @returns true if blocked, false otherwise
 */
private async isUserBlocked(targetId: string, viewerId: string): Promise<boolean> {
  const block = await this.blockRepo.findOne({
    where: [
      { userId: targetId, blockedUserId: viewerId },
      { userId: viewerId, blockedUserId: targetId },
    ],
  });
  return !!block;
}
```

---

### 4.2 ProfileMapper (profile.mapper.ts)

**buildProjection(user, socialState, counts, chefProfile)**
```typescript
/**
 * Build full profile projection
 * 
 * @param user - User entity
 * @param socialState - Social graph state
 * @param counts - Aggregated counts
 * @param chefProfile - Optional chef profile
 * @returns ProfileProjection
 */
buildProjection(
  user: User,
  socialState: {
    isFollowing: boolean;
    isFollowedBy: boolean;
    isRequested: boolean;
    isBlocked: boolean;
    isPrivate: boolean;
  },
  counts: {
    followersCount: number;
    followingCount: number;
    reelsCount: number;
    reviewsCount: number;
  },
  chefProfile?: any,
): ProfileProjection
```

**Logic:**
- Extract FSSAI from chef profile description: `/FSSAI:\s*(\d+)/`
- Map all user fields to projection
- Include social graph state
- Include counts

---

**buildLockedProjection(user, socialState, counts, chefProfile)**
```typescript
/**
 * Build locked profile projection (private accounts)
 * 
 * @param user - User entity
 * @param socialState - Social graph state
 * @param counts - Partial counts (followers/following only)
 * @param chefProfile - Optional chef profile
 * @returns Partial<ProfileProjection>
 */
buildLockedProjection(
  user: User,
  socialState: {
    isFollowing: boolean;
    isFollowedBy: boolean;
    isRequested: boolean;
    isBlocked: boolean;
    isPrivate: boolean;
  },
  counts: {
    followersCount: number;
    followingCount: number;
  },
  chefProfile?: any,
): Partial<ProfileProjection>
```

**Locked Fields (Hidden):**
- `coverUrl`, `bio`, `location`, `website`, `pronouns`, `publicEmail`
- `coinsBalance`, `reelsCount`, `reviewsCount`
- `fssaiNumber`

**Visible Fields:**
- `userId`, `username`, `fullName`, `displayName`, `avatarUrl`
- `followersCount`, `followingCount`
- Social graph state
- `isChef`, `kitchenName` (business info)

---

### 4.3 ProfileRolesService (profile-roles.service.ts)

**activateChefProfile(userId, dto)**
```typescript
/**
 * Activate chef profile for a user
 * 
 * @param userId - User ID
 * @param dto - ActivateChefProfileDto
 * @returns ChefProfileResponseDto
 * @throws ConflictException if chef profile already exists
 */
async activateChefProfile(
  userId: string,
  dto: ActivateChefProfileDto,
): Promise<ChefProfileResponseDto>
```

**Logic:**
1. Check if chef profile exists
2. If exists → throw `ConflictException`
3. Create `ChefProfile` entity with:
   - `businessName` (required)
   - `description`, `address`, `cuisines` (optional)
   - `deliveryDetails: { radiusKm, isPickupAllowed }`
   - `verificationStatus: 'pending'`
   - `isActive: true`
4. Save to DB
5. Return mapped DTO

---

**updateChefProfile(userId, dto)**
```typescript
/**
 * Update chef profile
 * 
 * @param userId - User ID
 * @param dto - UpdateChefProfileDto
 * @returns ChefProfileResponseDto
 * @throws NotFoundException if chef profile doesn't exist
 */
async updateChefProfile(
  userId: string,
  dto: UpdateChefProfileDto,
): Promise<ChefProfileResponseDto>
```

---

**isChef(userId)**
```typescript
/**
 * Check if user has activated chef profile
 * 
 * @param userId - User ID
 * @returns true if chef profile exists and is active
 */
async isChef(userId: string): Promise<boolean>
```

**Query:**
```typescript
const chefProfile = await this.chefProfileRepo.findOne({
  where: { userId, isActive: true },
});
return !!chefProfile;
```

---

## 5. Data Flow

### 5.1 Profile View Flow

```mermaid
sequenceDiagram
    participant Client
    participant API
    participant ProfileService
    participant Cache
    participant DB
    participant SocialModule

    Client->>API: GET /profile/:username (JWT)
    API->>ProfileService: getProfile(username, viewerId)
    ProfileService->>Cache: get('profile:projection:username')
    
    alt Cache Hit
        Cache-->>ProfileService: Return cached projection
        ProfileService-->>API: Return projection
    else Cache Miss
        ProfileService->>DB: Find user by username
        DB-->>ProfileService: User entity
        
        ProfileService->>SocialModule: Check if blocked
        SocialModule-->>ProfileService: isBlocked
        
        alt User Blocked
            ProfileService-->>API: throw NotFoundException
        else Not Blocked
            ProfileService->>SocialModule: Get social state
            SocialModule-->>ProfileService: { isFollowing, isPrivate, ... }
            
            alt Private + Not Following
                ProfileService->>ProfileMapper: buildLockedProjection()
                ProfileMapper-->>ProfileService: Locked projection
            else Public or Following
                ProfileService->>DB: Get counts (followers, reels, reviews)
                DB-->>ProfileService: Counts
                ProfileService->>ProfileMapper: buildProjection()
                ProfileMapper-->>ProfileService: Full projection
                ProfileService->>Cache: set('profile:projection:username', 300s)
            end
        end
    end
    
    API-->>Client: { success: true, data: projection }
```

---

### 5.2 Profile Update Flow

```mermaid
sequenceDiagram
    participant Client
    participant API
    participant ProfileService
    participant DB
    participant Cache

    Client->>API: PATCH /profile/me (JWT + EditProfileDto)
    API->>ProfileService: updateProfile(userId, dto)
    
    ProfileService->>DB: Find user by ID
    DB-->>ProfileService: User entity
    
    alt Username Changed
        ProfileService->>DB: Check username cooldown
        
        alt Cooldown Active
            ProfileService-->>API: throw BadRequestException
        else Cooldown Passed
            ProfileService->>DB: Check username uniqueness
            
            alt Username Taken
                ProfileService-->>API: throw ConflictException
            else Username Available
                ProfileService->>ProfileService: Update username + lastUsernameChangeAt
            end
        end
    end
    
    ProfileService->>ProfileService: Update basic fields (bio, location, etc.)
    
    alt Avatar/Cover Updated
        ProfileService->>ProfileService: Convert S3 keys to CDN URLs
    end
    
    ProfileService->>DB: Save user entity
    
    alt Chef Fields Updated
        ProfileService->>DB: Update ChefProfile
    end
    
    ProfileService->>Cache: delete('profile:projection:username')
    ProfileService->>ProfileService: getMyProfile(userId)
    
    API-->>Client: { success: true, data: updated projection }
```

---

### 5.3 Image Upload Flow

```mermaid
sequenceDiagram
    participant Client
    participant API
    participant ProfileService
    participant S3

    Client->>API: POST /profile/avatar/signed-url (JWT + filename)
    API->>ProfileService: getAvatarSignedUrl(userId, dto)
    
    ProfileService->>ProfileService: Generate S3 key: profiles/{userId}/avatar-{timestamp}.ext
    ProfileService->>S3: Generate presigned PUT URL (15 min expiry)
    S3-->>ProfileService: Presigned URL
    
    ProfileService-->>API: { uploadUrl, s3Key, cdnUrl, expiresIn }
    API-->>Client: Return signed URL
    
    Client->>S3: PUT image to uploadUrl (direct upload)
    S3-->>Client: 200 OK
    
    Client->>API: PATCH /profile/me (avatarKey: s3Key)
    API->>ProfileService: updateProfile(userId, { avatarKey })
    ProfileService->>ProfileService: Convert s3Key to CDN URL
    ProfileService-->>API: Updated profile
    API-->>Client: { success: true, data: profile }
```

---

### 5.4 Chef Profile Activation Flow

```mermaid
sequenceDiagram
    participant Client
    participant API
    participant ProfileRolesService
    participant DB

    Client->>API: POST /profile/chef/activate (JWT + ActivateChefProfileDto)
    API->>ProfileRolesService: activateChefProfile(userId, dto)
    
    ProfileRolesService->>DB: Find existing chef profile
    
    alt Chef Profile Exists
        ProfileRolesService-->>API: throw ConflictException
    else No Chef Profile
        ProfileRolesService->>DB: Create ChefProfile (verificationStatus: pending)
        DB-->>ProfileRolesService: Saved chef profile
        ProfileRolesService-->>API: Return ChefProfileResponseDto
    end
    
    API-->>Client: { success: true, data: chef profile }
```

---

## 6. Caching Strategy

### 6.1 Cache Configuration

**Cache Key:** `profile:projection:{username}`  
**TTL:** 5 minutes (300 seconds)  
**Storage:** Valkey/Redis

**CacheService Methods:**
```typescript
// Get cached profile
const cached = await this.cacheService.get<ProfileProjection>(cacheKey);

// Set cache
await this.cacheService.set(cacheKey, projection, this.CACHE_TTL);

// Delete cache
await this.cacheService.del(cacheKey);
```

---

### 6.2 Cache Invalidation Triggers

| **Event** | **Action** | **Trigger** |
|-----------|----------|-------------|
| Profile edit | Delete cache | `PATCH /profile/me` |
| Username change | Delete cache | `PATCH /profile/me` (username field) |
| Follow user | Delete cache | Social module calls `invalidateProfileCache(username)` |
| Unfollow user | Delete cache | Social module calls `invalidateProfileCache(username)` |
| Block user | Delete cache | Social module calls `invalidateProfileCache(username)` |

**External Invalidation Method:**
```typescript
/**
 * Invalidate profile cache (called from social module)
 */
async invalidateProfileCache(username: string) {
  await this.cacheService.del(`profile:projection:${username}`);
  this.logger.debug(`Invalidated cache for profile: ${username}`);
}
```

---

### 6.3 Cache Performance

**Expected Hit Rate:** >70%  
**Cache Miss Scenarios:**
- First profile view after signup
- Cache expiry (5 minutes)
- Profile edited recently
- Follow/unfollow events

**Monitoring:**
- Track cache hit/miss ratio via Valkey metrics
- Alert if hit rate drops below 60%

---

## 7. Security & Authentication

### 7.1 JWT Authentication

**Guards:**
- `JwtAuthGuard`: All secured endpoints (extracts `req.user.id`)
- `OptionalJwtAuthGuard`: Username availability check (allows unauthenticated)

**JWT Payload:**
```typescript
interface JwtPayload {
  id: string; // User ID
  phone: string;
  iat: number; // Issued at
  exp: number; // Expires at (7 days)
}
```

**Token Extraction:**
```typescript
@UseGuards(JwtAuthGuard)
async getMyProfile(@Request() req: any) {
  const userId = req.user.id; // Extracted by JwtAuthGuard
  return this.profileService.getMyProfile(userId);
}
```

---

### 7.2 Authorization Rules

| **Endpoint** | **Auth Required** | **Additional Check** |
|-------------|------------------|---------------------|
| `GET /profile/me` | ✅ Yes | None |
| `PATCH /profile/me` | ✅ Yes | None (self-update) |
| `GET /profile/:username` | ✅ Yes | Block check, privacy check |
| `GET /username-check` | ⚠️ Optional | None |
| `POST /avatar/signed-url` | ✅ Yes | None |
| `GET /profile/:username/reels` | ✅ Yes | Privacy check |
| `POST /chef/activate` | ✅ Yes | None |

---

### 7.3 Privacy Enforcement

**Blocked User Access:**
```typescript
const isBlocked = await this.isUserBlocked(targetId, viewerId);
if (isBlocked) {
  throw new NotFoundException(`User @${username} not found`); // 404, not 403
}
```

**Private Account Access:**
```typescript
if (socialState.isPrivate && !socialState.isFollowing && user.id !== viewerId) {
  return this.profileMapper.buildLockedProjection(user, socialState, counts, chefProfile);
}
```

**Content Grid Access:**
```typescript
if (socialState.isPrivate && !socialState.isFollowing && user.id !== viewerId) {
  throw new BadRequestException('This account is private');
}
```

---

### 7.4 Input Validation

**DTO Validation (class-validator):**
```typescript
export class EditProfileDto {
  @IsOptional()
  @IsString()
  @MaxLength(30)
  @Matches(/^[a-z0-9_.]{3,30}$/, {
    message: 'Username must be 3-30 characters, lowercase alphanumeric with underscore/dot only',
  })
  username?: string;

  @IsOptional()
  @IsUrl()
  website?: string;

  @IsOptional()
  @IsEmail()
  publicEmail?: string;
}
```

**Presigned URL Validation:**
```typescript
@IsEnum(['image/jpeg', 'image/png', 'image/webp'], {
  message: 'Only image/jpeg, image/png, and image/webp are allowed',
})
contentType!: string;
```

---

## 8. Error Handling

### 8.1 Standard Error Responses

**404 Not Found (User Not Found or Blocked):**
```json
{
  "success": false,
  "message": "User @unknown_user not found",
  "errorCode": "NOT_FOUND"
}
```

**409 Conflict (Username Taken):**
```json
{
  "success": false,
  "message": "Username already taken",
  "errorCode": "CONFLICT"
}
```

**400 Bad Request (Username Cooldown):**
```json
{
  "success": false,
  "message": "You can change your username again in 10 days.",
  "errorCode": "USERNAME_CHANGE_COOLDOWN",
  "data": { "daysLeft": 10 }
}
```

**400 Bad Request (Private Account):**
```json
{
  "success": false,
  "message": "This account is private",
  "errorCode": "PRIVATE_ACCOUNT"
}
```

**400 Bad Request (S3 Presign Failed):**
```json
{
  "success": false,
  "message": "Failed to generate avatar upload URL",
  "errorCode": "S3_PRESIGN_FAILED"
}
```

**409 Conflict (Chef Profile Exists):**
```json
{
  "success": false,
  "message": "Chef profile already activated for this user",
  "errorCode": "CHEF_PROFILE_EXISTS"
}
```

---

### 8.2 Error Handling Best Practices

**1. Return 404 for Blocked Users (Not 403):**
```typescript
// ✅ Correct: Hides block status
throw new NotFoundException(`User @${username} not found`);

// ❌ Wrong: Reveals block status
throw new ForbiddenException('User is blocked');
```

**2. Log Errors with Context:**
```typescript
this.logger.error(`Failed to generate avatar signed URL: ${errorMessage}`);
this.logger.debug(`   S3 Key: ${s3Key}`);
```

**3. Graceful Degradation (Cache Errors):**
```typescript
try {
  return !!block;
} catch (error) {
  this.logger.error(`Error in isUserBlocked: ${error.message}`, error.stack);
  return false; // Fail open to not block profile access
}
```

---

## 9. Integration Points

### 9.1 Social Module

**Imports:**
- `UserFollow` entity → Check following status
- `UserPrivacy` entity → Check privacy settings, get follower counts
- `UserBlock` entity → Check block status

**Exports:**
- `invalidateProfileCache(username)` → Called on follow/unfollow events

---

### 9.2 Reels Module

**Imports:**
- `Reel` schema (MongoDB) → Fetch user reels

**Query Example:**
```typescript
const reels = await this.reelModel
  .find({ userId, deletedAt: null })
  .sort({ createdAt: -1 })
  .limit(20)
  .exec();
```

---

### 9.3 Reviews Module

**Imports:**
- `OrderReview` entity (PostgreSQL) → Fetch user reviews

**Query Example:**
```typescript
const reviews = await this.reviewRepo
  .createQueryBuilder('review')
  .where('review.userId = :userId', { userId })
  .orderBy('review.createdAt', 'DESC')
  .take(20)
  .getMany();
```

---

### 9.4 Media Processing Module

**Imports:**
- `Media` schema (MongoDB) → Count user reels

**Query Example:**
```typescript
const reelsCount = await this.mediaModel.countDocuments({ 
  userId, 
  status: 'ready' 
});
```

---

### 9.5 Chef Module

**Imports:**
- `ChefProfile` entity (PostgreSQL) → Display chef badge, business name

**Query Example:**
```typescript
const chefProfile = await this.chefProfileRepo.findOne({ 
  where: { userId, isActive: true } 
});
```

---

## 10. Code Examples

### 10.1 Mobile App (React Native + React Query)

**Fetch Own Profile:**
```typescript
import { useGetMyProfile } from '@chefooz/api-client/profile';

export const ProfileScreen = () => {
  const { data, isLoading } = useGetMyProfile();

  if (isLoading) return <LoadingSpinner />;

  return (
    <View>
      <Avatar source={{ uri: data.data.avatarUrl }} />
      <Text>{data.data.username}</Text>
      <Text>{data.data.bio}</Text>
      <Text>Followers: {data.data.followersCount}</Text>
      <Text>Following: {data.data.followingCount}</Text>
    </View>
  );
};
```

---

**Update Profile:**
```typescript
import { useUpdateProfile } from '@chefooz/api-client/profile';

export const EditProfileScreen = () => {
  const { mutate: updateProfile, isLoading } = useUpdateProfile();

  const handleSave = () => {
    updateProfile({
      username: 'new_username',
      bio: 'Updated bio',
      location: 'New York',
    }, {
      onSuccess: () => {
        Alert.alert('Success', 'Profile updated!');
      },
      onError: (error) => {
        Alert.alert('Error', error.message);
      },
    });
  };

  return <Button title="Save" onPress={handleSave} disabled={isLoading} />;
};
```

---

**Upload Avatar:**
```typescript
import { useGetAvatarSignedUrl, useUpdateProfile } from '@chefooz/api-client/profile';
import * as ImagePicker from 'expo-image-picker';
import * as FileSystem from 'expo-file-system';

export const AvatarUploadButton = () => {
  const { mutateAsync: getSignedUrl } = useGetAvatarSignedUrl();
  const { mutate: updateProfile } = useUpdateProfile();

  const handleUpload = async () => {
    // Pick image
    const result = await ImagePicker.launchImageLibraryAsync({
      mediaTypes: ImagePicker.MediaTypeOptions.Images,
      allowsEditing: true,
      aspect: [1, 1],
      quality: 0.8,
    });

    if (result.canceled) return;

    const uri = result.assets[0].uri;
    const filename = uri.split('/').pop();
    const contentType = 'image/jpeg';

    // Get signed URL
    const { data: signedUrlData } = await getSignedUrl({ filename, contentType });
    const { uploadUrl, s3Key } = signedUrlData.data;

    // Upload to S3
    await FileSystem.uploadAsync(uploadUrl, uri, {
      httpMethod: 'PUT',
      headers: { 'Content-Type': contentType },
    });

    // Update profile with S3 key
    updateProfile({ avatarKey: s3Key }, {
      onSuccess: () => Alert.alert('Success', 'Avatar updated!'),
    });
  };

  return <Button title="Upload Avatar" onPress={handleUpload} />;
};
```

---

**Check Username Availability:**
```typescript
import { useCheckUsername } from '@chefooz/api-client/profile';
import { useState, useEffect } from 'react';

export const UsernameInput = () => {
  const [username, setUsername] = useState('');
  const { data, isLoading } = useCheckUsername(username, {
    enabled: username.length >= 3, // Only check if valid length
  });

  useEffect(() => {
    if (data && !data.data.available) {
      console.log('Suggestions:', data.data.suggestions);
    }
  }, [data]);

  return (
    <View>
      <TextInput
        value={username}
        onChangeText={setUsername}
        placeholder="Enter username"
      />
      {isLoading && <ActivityIndicator />}
      {data && !data.data.available && (
        <Text style={{ color: 'red' }}>Username taken</Text>
      )}
      {data && data.data.available && (
        <Text style={{ color: 'green' }}>Username available</Text>
      )}
    </View>
  );
};
```

---

### 10.2 Backend Testing (Jest)

**Unit Test: Username Cooldown:**
```typescript
describe('ProfileService - Username Change', () => {
  it('should enforce 15-day username cooldown', async () => {
    const user = await createTestUser({
      username: 'old_username',
      lastUsernameChangeAt: new Date(Date.now() - 10 * 24 * 60 * 60 * 1000), // 10 days ago
    });

    await expect(
      profileService.updateProfile(user.id, { username: 'new_username' })
    ).rejects.toThrow('You can change your username again in 5 days');
  });

  it('should allow username change after cooldown', async () => {
    const user = await createTestUser({
      username: 'old_username',
      lastUsernameChangeAt: new Date(Date.now() - 20 * 24 * 60 * 60 * 1000), // 20 days ago
    });

    const result = await profileService.updateProfile(user.id, { 
      username: 'new_username' 
    });

    expect(result.username).toBe('new_username');
  });
});
```

---

**Integration Test: Private Profile Access:**
```typescript
describe('ProfileController - Privacy', () => {
  it('should return locked projection for private account (non-follower)', async () => {
    const privateUser = await createTestUser({ profileVisibility: 'private' });
    const viewer = await createTestUser();

    const response = await request(app.getHttpServer())
      .get(`/api/v1/profile/${privateUser.username}`)
      .set('Authorization', `Bearer ${viewerToken}`)
      .expect(200);

    expect(response.body.data.bio).toBeNull(); // Hidden
    expect(response.body.data.followersCount).toBeDefined(); // Visible
  });

  it('should return full projection for private account (follower)', async () => {
    const privateUser = await createTestUser({ profileVisibility: 'private' });
    const follower = await createTestUser();
    await createFollow(follower.id, privateUser.id, 'accepted');

    const response = await request(app.getHttpServer())
      .get(`/api/v1/profile/${privateUser.username}`)
      .set('Authorization', `Bearer ${followerToken}`)
      .expect(200);

    expect(response.body.data.bio).toBeDefined(); // Visible
  });
});
```

---

## 11. Testing Guide

### 11.1 Unit Tests

**Coverage Target:** >80%

**Test Files:**
- `profile.service.spec.ts` → ProfileService methods
- `profile-roles.service.spec.ts` → ProfileRolesService methods
- `profile.mapper.spec.ts` → Projection mapping logic

**Key Test Cases:**
1. Username availability check with suggestions
2. Username cooldown enforcement
3. S3 signed URL generation
4. Privacy filtering (locked vs. full projection)
5. Block detection (bidirectional)
6. Cache hit/miss scenarios
7. Chef profile activation (duplicate prevention)

---

### 11.2 Integration Tests

**Coverage Target:** >70%

**Test Files:**
- `profile.controller.e2e.spec.ts` → API endpoint tests

**Key Test Cases:**
1. GET /profile/me (authenticated)
2. PATCH /profile/me (username change, image upload)
3. GET /profile/:username (public, private, blocked)
4. GET /profile/:username/reels (privacy check)
5. POST /chef/activate (success, conflict)
6. PATCH /chef (update fields)

---

### 11.3 Load Testing

**Tool:** Apache JMeter or k6

**Scenarios:**
1. **Profile View Storm:** 1000 concurrent GET /profile/:username
2. **Cache Effectiveness:** Measure hit rate under load
3. **Image Upload:** 500 concurrent presigned URL requests

**Acceptance Criteria:**
- p95 latency <200ms for profile views
- Cache hit rate >70%
- Error rate <0.5%

---

## 12. Deployment & Operations

### 12.1 Environment Variables

```env
# AWS S3 Configuration
AWS_REGION=ap-south-1
AWS_ACCESS_KEY_ID=<redacted>
AWS_SECRET_ACCESS_KEY=<redacted>
S3_OUTPUT_BUCKET=chefooz-media-output
CDN_URL=https://cdn.chefooz.com

# Username Cooldown (days)
USERNAME_COOLDOWN_DAYS=15

# Cache Configuration
REDIS_HOST=valkey-cluster.example.com
REDIS_PORT=6379
REDIS_PASSWORD=<redacted>

# Database URLs
DATABASE_URL=postgresql://user:pass@db.example.com:5432/chefooz
MONGO_URI=mongodb://user:pass@mongo.example.com:27017/chefooz
```

---

### 12.2 Monitoring & Alerts

**CloudWatch Metrics:**
- `ProfileView_Count` → Profile view rate
- `ProfileUpdate_Count` → Profile edit rate
- `CacheHitRate` → Valkey cache hit rate
- `S3PresignError_Count` → Presigned URL failures

**Alerts:**
- Profile view latency >500ms (p95) → Page DevOps
- Cache hit rate <60% → Investigate cache config
- S3 presign error rate >1% → Check AWS credentials

---

### 12.3 Database Migrations

**Migration Files:**
```
migrations/
├── 001_create_user_table.sql
├── 002_create_chef_profile_table.sql
├── 003_add_username_cooldown_column.sql
└── 004_create_social_graph_tables.sql
```

**Run Migrations:**
```bash
npm run typeorm migration:run
```

---

### 12.4 Rollback Plan

**Database Rollback:**
```bash
npm run typeorm migration:revert
```

**Cache Invalidation (Full Flush):**
```bash
redis-cli -h valkey-cluster.example.com -p 6379 FLUSHDB
```

**S3 Cleanup (Orphaned Images):**
```bash
aws s3 rm s3://chefooz-media-output/profiles/ --recursive --exclude "*" --include "avatar-*" --older-than 30d
```

---

## 13. Document Metadata

| **Field** | **Value** |
|-----------|----------|
| **Document Version** | 1.0 |
| **Last Updated** | February 14, 2026 |
| **Owner** | Engineering Team (Chefooz) |
| **Stakeholders** | Backend, Mobile, DevOps, QA |
| **Review Cycle** | Quarterly |
| **Related Modules** | Auth, Social, Reels, Reviews, Chef |

---

**[END OF TECHNICAL GUIDE]**

---

**Next Document:** `QA_TEST_CASES.md` (Quality Assurance Reference)
