# Profile Module – Feature Overview

**Document Type:** Business & Product Requirements  
**Module:** Profile Management  
**Last Updated:** February 14, 2026  
**Status:** Production (First Release – Feb 2026)

---

## 📋 Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Business Context](#2-business-context)
3. [User Personas & Stories](#3-user-personas--stories)
4. [Feature Specifications](#4-feature-specifications)
5. [Business Rules & Constraints](#5-business-rules--constraints)
6. [User Workflows](#6-user-workflows)
7. [Privacy & Security](#7-privacy--security)
8. [Performance Metrics](#8-performance-metrics)
9. [Success Criteria](#9-success-criteria)
10. [Future Roadmap](#10-future-roadmap)

---

## 1. Executive Summary

### 1.1 Purpose

The **Profile Module** is the identity and self-expression layer of the Chefooz platform. It enables users to:

- **Build their personal brand** through customizable profiles (avatar, cover, bio)
- **Control their privacy** (public vs. private accounts)
- **Showcase their content** (reels, reviews, reputation)
- **Operate dual roles** (Customer + Chef profiles)
- **Manage social graph visibility** (who can see their content)

This module serves as the foundation for trust, discovery, and community engagement within Chefooz.

### 1.2 Business Value

| **Metric** | **Target** | **Business Impact** |
|------------|-----------|---------------------|
| Profile completion rate | 85% | Higher engagement + trust signals |
| Username claim rate | 95% | Personalization + brand identity |
| Privacy adoption rate | 30% | User trust + safety compliance |
| Chef profile activations | 15% of users | Creator economy growth |
| Profile view-to-follow rate | 12% | Social discovery effectiveness |

### 1.3 Key Features

1. **Profile Management:** View, edit, and customize user profiles
2. **Username System:** Unique handles with availability checks + suggestions
3. **Profile Images:** Avatar and cover upload via S3 presigned URLs
4. **Privacy Controls:** Public/private account modes with follower-gated content
5. **Social Graph Status:** Real-time relationship state (following, blocked, requested)
6. **Content Grids:** Paginated reels and reviews with privacy filtering
7. **Dual Profiles:** Seamless user-to-chef role switching
8. **Chef Profile Activation:** Business registration with FSSAI compliance
9. **Profile Caching:** 5-minute cache for high-traffic profiles

---

## 2. Business Context

### 2.1 Market Positioning

Chefooz profiles blend **Instagram-style social features** with **food service commerce**:

- **Social Layer:** Profiles act as discovery hubs (like Instagram/TikTok profiles)
- **Commerce Layer:** Chef profiles showcase menus, FSSAI licenses, and delivery zones
- **Trust Layer:** Reputation scores, verification badges, and review history
- **Privacy Layer:** Instagram-like private accounts for casual users

### 2.2 Competitive Advantages

| **Feature** | **Chefooz** | **Swiggy/Zomato** | **Instagram** |
|-------------|-------------|-------------------|---------------|
| Dual profiles (User + Chef) | ✅ | ❌ | ❌ |
| Commerce-integrated profiles | ✅ | ✅ | ❌ |
| Privacy-first content control | ✅ | ❌ | ✅ |
| Reputation-based coin economy | ✅ | ❌ | ❌ |
| FSSAI license display | ✅ | ✅ | ❌ |

### 2.3 Strategic Goals (Q1 2026)

1. **85% profile completion rate** within first 7 days of signup
2. **30% private account adoption** among users (trust signal)
3. **15% chef profile activation rate** (creator economy kickstart)
4. **12% profile-to-follow conversion** (discovery effectiveness)

---

## 3. User Personas & Stories

### 3.1 Persona: Casual Foodie (Primary User)

**Name:** Priya, 28, Marketing Manager  
**Goal:** Discover food, share reviews, maintain privacy

#### User Stories

**US-PROFILE-001: View Own Profile**  
_As a user, I want to view my profile so I can see my content and stats._

- **Acceptance Criteria:**
  - Display username, avatar, cover, bio, location, website
  - Show follower/following counts
  - Show reels count, reviews count
  - Show reputation score and coin balance
  - Display chef badge if activated

**US-PROFILE-002: Edit Profile**  
_As a user, I want to edit my profile to express my personality._

- **Acceptance Criteria:**
  - Update username (with 15-day cooldown)
  - Update display name, full name, bio (300 chars max)
  - Upload avatar (2MB max, .jpg/.png/.webp)
  - Upload cover (6MB max, .jpg/.png/.webp)
  - Set profile visibility (public/private)
  - Add pronouns, location, website, public email

**US-PROFILE-003: Check Username Availability**  
_As a user, I want to check if a username is available before claiming it._

- **Acceptance Criteria:**
  - Real-time availability check
  - Case-insensitive uniqueness
  - Suggest 5 alternatives if taken (numeric/year suffixes)
  - Validate format: 3-30 chars, lowercase alphanumeric + underscore/dot

**US-PROFILE-004: Set Privacy Mode**  
_As a user, I want to make my account private to control who sees my content._

- **Acceptance Criteria:**
  - Toggle public ↔ private mode
  - Private accounts: hide reels/reviews from non-followers
  - Show locked profile to non-followers (limited info)
  - Follow requests required for private accounts

---

### 3.2 Persona: Home Chef (Creator)

**Name:** Amit, 34, Home Chef  
**Goal:** Build professional presence, attract orders

#### User Stories

**US-PROFILE-005: Activate Chef Profile**  
_As a user, I want to activate my chef profile to sell food._

- **Acceptance Criteria:**
  - Provide business name (required)
  - Add description, address, cuisines
  - Set delivery radius and pickup options
  - One chef profile per user (idempotent activation)
  - Defaults to pending verification status

**US-PROFILE-006: Update Chef Profile**  
_As a chef, I want to update my kitchen details._

- **Acceptance Criteria:**
  - Update business name, description
  - Update address, cuisines
  - Update delivery radius (radiusKm)
  - Toggle pickup availability (isPickupAllowed)
  - Toggle profile active/inactive status

**US-PROFILE-007: View Chef Profile Badge**  
_As a user, I want to see who is a verified chef._

- **Acceptance Criteria:**
  - Chef badge visible on profile
  - Show kitchen name (businessName)
  - Show FSSAI number (if provided)
  - Display verification status (pending/verified/rejected)

---

### 3.3 Persona: Profile Viewer (Discovery)

**Name:** Rohan, 25, College Student  
**Goal:** Discover chefs, follow food enthusiasts

#### User Stories

**US-PROFILE-008: View Other User's Profile**  
_As a user, I want to view other profiles to decide if I should follow them._

- **Acceptance Criteria:**
  - View username, avatar, cover, bio, location
  - See follower/following counts
  - See reels/reviews counts
  - View social graph status (isFollowing, isFollowedBy, isRequested)
  - Respect privacy: blocked users see 404
  - Private accounts: show locked projection to non-followers

**US-PROFILE-009: View Profile Reels Grid**  
_As a user, I want to see a profile's reels in a grid._

- **Acceptance Criteria:**
  - Paginated grid (20 items/page)
  - Cursor-based pagination (createdAt)
  - Show thumbnail, view count, like count, duration
  - Filter out soft-deleted reels
  - Respect privacy: block access if private + not following

**US-PROFILE-010: View Profile Reviews**  
_As a user, I want to see a profile's reviews._

- **Acceptance Criteria:**
  - Paginated list (20 items/page)
  - Show rating, comment, order ID, timestamp
  - Respect privacy: block access if private + not following

---

## 4. Feature Specifications

### 4.1 Profile Management

#### 4.1.1 Get Own Profile

**Endpoint:** `GET /api/v1/profile/me`  
**Auth:** JWT required

**Response Fields:**
```typescript
{
  userId: string;
  username: string | null;
  fullName: string | null;
  displayName: string | null;
  avatarUrl: string | null;
  coverUrl: string | null;
  bio: string | null;
  location: string | null;
  website: string | null;
  pronouns: string | null;
  publicEmail: string | null;
  phoneVerified: boolean;
  profileVisibility: 'public' | 'private';
  role: 'user' | 'chef';
  isPrivate: boolean;
  verified: boolean;
  badges: string[];
  reputationScore: number;
  coinsBalance: number;
  followersCount: number;
  followingCount: number;
  reelsCount: number;
  reviewsCount: number;
  isFollowing: boolean;
  isFollowedBy: boolean;
  isRequested: boolean;
  isBlocked: boolean;
  isChef: boolean;
  kitchenName: string | null;
  fssaiNumber: string | null;
  createdAt: string; // ISO 8601
}
```

**Business Rules:**
- Returns full profile data (no privacy filtering for own profile)
- Includes social graph self-state (all false except own profile)
- Displays coin balance and reputation score

---

#### 4.1.2 Update Profile

**Endpoint:** `PATCH /api/v1/profile/me`  
**Auth:** JWT required

**Request Body:**
```typescript
{
  username?: string; // 3-30 chars, lowercase alphanumeric + underscore/dot
  displayName?: string; // 2-50 chars
  fullName?: string; // max 50 chars
  bio?: string; // max 300 chars
  website?: string; // valid URL
  location?: string; // max 100 chars
  pronouns?: string; // max 50 chars
  avatarKey?: string; // S3 key after upload
  coverKey?: string; // S3 key after upload
  profileVisibility?: 'public' | 'private';
  publicEmail?: string; // valid email
  chefName?: string; // Chef profile: business name
  fssaiNumber?: string; // Chef profile: 14-digit license
}
```

**Business Rules:**
- **Username changes:**
  - 15-day cooldown period (`lastUsernameChangeAt`)
  - Case-insensitive uniqueness check
  - Throws `USERNAME_CHANGE_COOLDOWN` error if cooldown active
  - Throws `ConflictException` if username taken
- **Image handling:**
  - Convert S3 keys to CDN URLs: `${CDN_URL}/${s3Key}`
  - Avatar: max 2MB, .jpg/.png/.webp
  - Cover: max 6MB, .jpg/.png/.webp
- **Chef profile updates:**
  - If `chefName` or `fssaiNumber` provided, update `ChefProfile` entity
  - Store FSSAI in description field (temporary: `FSSAI: <number>`)
- **Cache invalidation:**
  - Delete cached profile projection on update: `profile:projection:${username}`

**Response:** Full `ProfileProjection` (updated data)

---

#### 4.1.3 Get Profile by Username

**Endpoint:** `GET /api/v1/profile/:username`  
**Auth:** JWT required

**Privacy Logic:**
1. **Blocked users:** Return 404 (as if profile doesn't exist)
2. **Private accounts:**
   - If viewer not following → return locked projection
   - If viewer following → return full projection
3. **Public accounts:** Return full projection to all

**Locked Projection (Private + Not Following):**
- Shows: `userId`, `username`, `fullName`, `displayName`, `avatarUrl` (only)
- Hides: Cover, bio, location, website, pronouns, email, coin balance, content counts
- Shows: `followersCount`, `followingCount` (for trust signal)
- Shows: Social graph status (`isFollowing`, `isRequested`, etc.)

**Cache Strategy:**
- Key: `profile:projection:${username}`
- TTL: 5 minutes (300 seconds)
- Invalidated on profile edit, follow/unfollow

---

#### 4.1.4 Get Profile Social Graph Status

**Endpoint:** `GET /api/v1/profile/:username/status`  
**Auth:** JWT required

**Response:**
```typescript
{
  isFollowing: boolean;
  isFollowedBy: boolean;
  isRequested: boolean; // Pending follow request
  isBlocked: boolean;
  isPrivate: boolean;
}
```

**Business Rules:**
- Used by frontend to render action buttons (Follow, Unfollow, Requested, Blocked)
- Checked in parallel with profile fetch for performance
- Respects bidirectional blocking

---

### 4.2 Username System

#### 4.2.1 Check Username Availability

**Endpoint:** `GET /api/v1/profile/username-check?username=chef_john`  
**Auth:** Optional JWT (OptionalJwtAuthGuard)

**Response:**
```typescript
{
  available: boolean;
  suggestions?: string[]; // If unavailable, returns 5 alternatives
}
```

**Suggestion Algorithm:**
1. **Numeric suffixes:** `chef_john1`, `chef_john2`, ... up to 3 suggestions
2. **Year-based suffixes:** `chef_john_2026`, `chef_john_2025`, up to 2 suggestions
3. Total: 5 suggestions maximum
4. Skip if suggestion already taken

**Business Rules:**
- No rate limiting (basic validation only)
- Case-insensitive uniqueness check
- If current user owns username → return `available: true`

---

### 4.3 Profile Image Upload

#### 4.3.1 Get Avatar Signed URL

**Endpoint:** `POST /api/v1/profile/avatar/signed-url`  
**Auth:** JWT required

**Request Body:**
```typescript
{
  filename: string; // e.g., "profile-pic.jpg"
  contentType: 'image/jpeg' | 'image/png' | 'image/webp';
}
```

**Response:**
```typescript
{
  uploadUrl: string; // Presigned PUT URL (valid 15 minutes)
  s3Key: string; // e.g., "profiles/user123/avatar-1733659200.webp"
  cdnUrl: string; // Final CDN URL after upload
  expiresIn: number; // 900 seconds
}
```

**Technical Details:**
- Upload directly to **OUTPUT bucket** (no MediaConvert needed)
- Client resizes image before upload (mobile app responsibility)
- S3 path pattern: `profiles/{userId}/avatar-{timestamp}.{ext}`
- Max size: 2MB (enforced client-side)
- Presigned URL generated via AWS SDK v3 (`@aws-sdk/s3-request-presigner`)

**Upload Flow:**
1. Client calls `POST /profile/avatar/signed-url`
2. Backend generates presigned PUT URL (15 min expiry)
3. Client uploads image directly to S3 using `uploadUrl`
4. Client sends `s3Key` to `PATCH /profile/me` to update profile

---

#### 4.3.2 Get Cover Signed URL

**Endpoint:** `POST /api/v1/profile/cover/signed-url`  
**Auth:** JWT required

**Request Body:**
```typescript
{
  filename: string;
  contentType: 'image/jpeg' | 'image/png' | 'image/webp';
}
```

**Response:** Same as avatar (see 4.3.1)

**Technical Details:**
- Same flow as avatar
- S3 path pattern: `profiles/{userId}/cover-{timestamp}.{ext}`
- Max size: 6MB (larger for wide aspect ratio)

---

### 4.4 Content Grids

#### 4.4.1 Get User Reels

**Endpoint:** `GET /api/v1/profile/:username/reels?limit=20&cursor=<ISO-date>`  
**Auth:** JWT required

**Response:**
```typescript
{
  reels: [
    {
      id: string;
      thumbnailUrl: string;
      playbackUrl: string;
      duration: number; // seconds
      viewCount: number;
      likeCount: number;
      createdAt: string; // ISO 8601
      linkedOrderId: string | null;
      reelPurpose: string | null;
    }
  ],
  nextCursor: string | null; // ISO date for pagination
}
```

**Business Rules:**
- Returns 20 reels per page (configurable via `limit`, max 50)
- Cursor-based pagination (descending `createdAt`)
- **Privacy check:**
  - If account private + viewer not following → throw `BadRequestException`
- **Soft-delete filtering:**
  - Exclude reels with `deletedAt != null` (critical for UX)
- **URL conversion:**
  - Convert S3 URIs to HTTPS URLs via `s3UriToHttps` utility

---

#### 4.4.2 Get User Reviews

**Endpoint:** `GET /api/v1/profile/:username/reviews?limit=20&cursor=<ISO-date>`  
**Auth:** JWT required

**Response:**
```typescript
{
  reviews: [
    {
      id: string;
      rating: number; // 1-5
      comment: string;
      orderId: string;
      createdAt: string; // ISO 8601
    }
  ],
  nextCursor: string | null;
}
```

**Business Rules:**
- Same pagination and privacy logic as reels
- Returns reviews from `order_review` table (PostgreSQL)

---

### 4.5 Dual Profile System

#### 4.5.1 Activate Chef Profile

**Endpoint:** `POST /api/v1/profile/chef/activate`  
**Auth:** JWT required

**Request Body:**
```typescript
{
  businessName: string; // Required, max 100 chars
  description?: string; // Optional
  address?: string; // Kitchen address
  cuisines?: string[]; // e.g., ["Italian", "Continental"]
  radiusKm?: number; // Delivery radius
  isPickupAllowed?: boolean; // Allow self-pickup
}
```

**Response:**
```typescript
{
  id: string; // Chef profile ID
  userId: string;
  businessName: string;
  isActive: boolean;
  verificationStatus: 'pending' | 'verified' | 'rejected';
  description: string | null;
  address: string | null;
  cuisines: string[];
  deliveryDetails: {
    radiusKm: number;
    isPickupAllowed: boolean;
  };
  createdAt: string;
  updatedAt: string;
}
```

**Business Rules:**
- **One chef profile per user** (idempotent: throws `ConflictException` if already exists)
- Defaults to `verificationStatus: 'pending'`
- Defaults to `isActive: true`
- Does NOT change user role automatically (role is informational only)

---

#### 4.5.2 Update Chef Profile

**Endpoint:** `PATCH /api/v1/profile/chef`  
**Auth:** JWT required

**Request Body:**
```typescript
{
  businessName?: string;
  description?: string;
  address?: string;
  cuisines?: string[];
  isActive?: boolean;
  radiusKm?: number;
  isPickupAllowed?: boolean;
}
```

**Response:** Same as activation (see 4.5.1)

**Business Rules:**
- Throws `NotFoundException` if chef profile doesn't exist
- Partial updates supported (all fields optional)

---

#### 4.5.3 Get My Chef Profile

**Endpoint:** `GET /api/v1/profile/chef/me`  
**Auth:** JWT required

**Response:** Same as activation (see 4.5.1) or `null` if not activated

---

#### 4.5.4 Get Profile Role

**Endpoint:** `GET /api/v1/profile/role`  
**Auth:** JWT required

**Response:**
```typescript
{
  isChef: boolean; // True if chef profile exists + isActive
  activeProfile: 'user' | 'chef'; // Current context
}
```

**Business Rules:**
- Used by mobile app to toggle between user/chef modes
- `activeProfile` defaults to `'user'` unless explicitly set to `'chef'`

---

## 5. Business Rules & Constraints

### 5.1 Username Rules

| **Rule** | **Value** | **Rationale** |
|----------|----------|---------------|
| Length | 3-30 characters | Balance uniqueness + memorability |
| Format | Lowercase alphanumeric + `_` `.` | Instagram-like handles |
| Uniqueness | Case-insensitive | Prevent confusion (John vs john) |
| Change cooldown | 15 days | Prevent abuse + maintain brand identity |
| Validation regex | `^[a-z0-9_.]{3,30}$` | Consistent format |

---

### 5.2 Profile Image Rules

| **Type** | **Max Size** | **Formats** | **Upload Bucket** | **Rationale** |
|----------|-------------|-------------|-------------------|---------------|
| Avatar | 2MB | .jpg, .png, .webp | OUTPUT (direct) | Client-side resize, no processing |
| Cover | 6MB | .jpg, .png, .webp | OUTPUT (direct) | Wide aspect ratio, larger file |

**Technical Note:**  
Unlike reels (INPUT → MediaConvert → OUTPUT), profile images skip MediaConvert and upload directly to OUTPUT bucket. Mobile app resizes images before upload.

---

### 5.3 Privacy Rules

| **Scenario** | **Viewer** | **Behavior** |
|-------------|-----------|-------------|
| Public account | Anyone | Full profile access |
| Private account | Non-follower | Locked projection (avatar + basic info only) |
| Private account | Follower | Full profile access |
| Blocked user | Blocked by target | 404 (as if profile doesn't exist) |
| Blocked user | Blocker | 404 (as if profile doesn't exist) |

---

### 5.4 Chef Profile Rules

| **Rule** | **Value** | **Rationale** |
|----------|----------|---------------|
| Profiles per user | 1 | Simplify onboarding, prevent fragmentation |
| Verification status | `pending` by default | Manual admin approval required |
| Public visibility | Always public | Business profiles must be discoverable |
| FSSAI requirement | Optional | Stored in description field (temporary) |
| Delivery radius | 0-50 km | Prevent unrealistic delivery zones |

---

### 5.5 Caching Rules

| **Cache Key** | **TTL** | **Invalidation Trigger** |
|--------------|--------|--------------------------|
| `profile:projection:{username}` | 5 minutes | Profile edit, follow/unfollow |

**Cache Strategy:**
- Store full `ProfileProjection` in Valkey/Redis
- Check cache before DB query
- Reduce DB load for high-traffic profiles (e.g., top chefs)
- Social module calls `invalidateProfileCache()` on follow/unfollow

---

## 6. User Workflows

### 6.1 New User Profile Setup

**Flow:** Signup → Profile Completion → Username Claim → First Follow

**Steps:**
1. User signs up via OTP (Auth module)
2. User redirected to profile setup screen
3. User uploads avatar (optional)
4. User claims username (required)
   - Checks availability via `GET /username-check`
   - Selects from suggestions if taken
5. User fills bio, location (optional)
6. User sets privacy mode (defaults to public)
7. User browses Explore feed
8. User follows first chef/foodie

**Success Metric:** 85% complete username + avatar within 7 days

---

### 6.2 Chef Profile Activation

**Flow:** User Mode → Activate Chef → Add Details → Submit for Verification

**Steps:**
1. User taps "Become a Chef" in settings
2. User fills chef profile form:
   - Business name (required)
   - Description, address, cuisines (optional)
   - Delivery radius, pickup options
3. User taps "Activate Chef Profile"
4. Backend creates chef profile (status: `pending`)
5. User sees "Pending Verification" badge
6. Admin reviews FSSAI, verifies profile
7. User gets notification: "Chef profile approved!"
8. User switches to chef mode to post menu items

**Success Metric:** 15% user-to-chef activation rate in Q1 2026

---

### 6.3 Private Account Usage

**Flow:** Public Profile → Set Private → Follow Request → Approve/Deny

**Steps:**
1. User goes to Profile → Settings
2. User toggles "Private Account" ON
3. Profile visibility changes to `private`
4. New visitors see locked profile (avatar + username only)
5. Visitor taps "Follow"
6. Follow request sent (status: `pending`)
7. User receives notification
8. User approves request
9. Visitor now sees full profile + content grids

**Success Metric:** 30% private account adoption (trust signal)

---

### 6.4 Profile Discovery

**Flow:** Explore Feed → Tap Profile → View Grid → Follow

**Steps:**
1. User scrolls Explore feed (reels)
2. User taps creator's avatar on reel
3. Profile screen opens with:
   - Avatar, cover, bio
   - Follower/following counts
   - Reels grid (3 columns)
4. User taps a reel thumbnail → Reel player opens
5. User returns to profile
6. User taps "Follow"
7. Profile added to user's Following list

**Success Metric:** 12% profile-to-follow conversion rate

---

## 7. Privacy & Security

### 7.1 Data Privacy Compliance

**GDPR/DPDPA Considerations:**
- **Profile visibility:** User controls public/private mode
- **Data minimization:** Optional fields (pronouns, location, website)
- **Right to be forgotten:** Soft-delete profiles (future: hard delete)
- **Data portability:** Export profile data via settings (future)

---

### 7.2 Blocking Logic

**Bidirectional Blocking:**
- If User A blocks User B → B cannot view A's profile (404)
- If User B blocks User A → A cannot view B's profile (404)
- Both checks performed in `isUserBlocked()` helper

**Database Query:**
```sql
SELECT * FROM user_blocks
WHERE (userId = :targetId AND blockedUserId = :viewerId)
   OR (userId = :viewerId AND blockedUserId = :targetId);
```

---

### 7.3 Sensitive Data Handling

| **Field** | **Visibility** | **Storage** |
|-----------|---------------|-------------|
| Phone number | Never exposed | Hashed in DB, not in profiles |
| Coin balance | Own profile only | Exposed to authenticated user |
| Reputation score | Public | Displayed on profiles |
| FSSAI number | Public (if chef) | Stored in chef profile description |
| Email | Opt-in public email | User controls visibility |

---

### 7.4 Rate Limiting

**Endpoints:**
- **Username check:** No rate limit (basic validation)
- **Profile edits:** No rate limit (user action, not spammy)
- **Image uploads:** No rate limit (S3 presigned URLs, client-side upload)

**Rationale:** Profile operations are infrequent and user-initiated, not prone to abuse.

---

## 8. Performance Metrics

### 8.1 Technical KPIs

| **Metric** | **Target** | **Measurement** |
|-----------|-----------|----------------|
| Profile fetch latency | <200ms (p95) | CloudWatch API Gateway latency |
| Cache hit rate | >70% | Valkey/Redis metrics |
| Reels grid load time | <500ms | Mobile app analytics |
| Image upload success rate | >98% | S3 upload success logs |
| Username availability check | <100ms | DB query latency |

---

### 8.2 Business KPIs

| **Metric** | **Target** | **Measurement** |
|-----------|-----------|----------------|
| Profile completion rate | 85% | Users with username + avatar / total signups |
| Chef activation rate | 15% | Chef profiles / total users |
| Private account adoption | 30% | Private profiles / total users |
| Profile-to-follow conversion | 12% | Follows / profile views |
| Average session time on profile | 45 seconds | Mobile app analytics |

---

### 8.3 Scalability Targets

| **Metric** | **Target** | **Strategy** |
|-----------|-----------|-------------|
| Concurrent profile views | 10,000/min | Caching + CDN |
| Image uploads per day | 50,000 | S3 presigned URLs (offload to client) |
| DB query load | <1000 QPS | Caching + read replicas |

---

## 9. Success Criteria

### 9.1 Launch Readiness (Feb 2026)

**Must-Have Features:**
- ✅ Profile CRUD (view, edit, delete)
- ✅ Username availability check + suggestions
- ✅ Avatar/cover upload via S3 presigned URLs
- ✅ Privacy controls (public/private modes)
- ✅ Social graph status integration
- ✅ Reels and reviews content grids
- ✅ Chef profile activation + management
- ✅ Profile caching (5-min TTL)

**Nice-to-Have (Deferred to Q2):**
- ⏳ Verification badges
- ⏳ Reputation badge system
- ⏳ Profile analytics (views, follows over time)
- ⏳ Profile templates for chefs

---

### 9.2 Post-Launch Validation (30 Days)

**Week 1:**
- 85% users complete username + avatar
- <5% username conflict errors

**Week 2:**
- 12% profile-to-follow conversion
- <1% blocked user access attempts

**Week 4:**
- 15% chef profile activations
- 30% private account adoption

---

### 9.3 Quality Gates

| **Gate** | **Criteria** | **Status** |
|---------|-------------|-----------|
| API stability | <0.5% error rate | ✅ Ready |
| Mobile UI polish | 4.5+ App Store rating | ✅ Ready |
| Privacy compliance | DPDPA audit passed | ✅ Ready |
| Performance | <200ms p95 latency | ✅ Ready |
| Cache effectiveness | >70% hit rate | ✅ Ready |

---

## 10. Future Roadmap

### 10.1 Q2 2026: Profile Enhancements

**Verification Badges:**
- Manual admin verification for top chefs
- Blue checkmark badge on profiles
- Verification criteria: FSSAI + order count + reviews

**Reputation Badge System:**
- Bronze/Silver/Gold/Platinum badges based on reputation score
- Display badges on profile (e.g., "Gold Foodie")

**Profile Analytics:**
- View counts, follower growth trends
- Best-performing reels/reviews
- Available to chef profiles only

---

### 10.2 Q3 2026: Commerce Integration

**Menu Preview on Profile:**
- Show top 3 menu items on chef profiles
- Tap to view full menu → Order flow

**Order History Badge:**
- Show total orders placed (for users)
- Show total orders fulfilled (for chefs)

**Loyalty Program Integration:**
- Display coin earning history
- Show redeemed rewards on profile

---

### 10.3 Q4 2026: Social Features

**Profile Highlights (Instagram Stories-like):**
- Pin top reels to profile as highlights
- Categorize by cuisine/theme

**Profile Mentions:**
- Tag chefs/users in reels and reviews
- Backlink to tagged profiles

**Profile Sharing:**
- Share profile links (deep links)
- QR code generation for offline discovery

---

## 11. Document Metadata

| **Field** | **Value** |
|-----------|----------|
| **Document Version** | 1.0 |
| **Last Updated** | February 14, 2026 |
| **Owner** | Product Team (Chefooz) |
| **Stakeholders** | Engineering, Design, QA, Legal |
| **Review Cycle** | Quarterly |
| **Related Modules** | Auth, Social, Reels, Reviews, Chef |

---

## 12. Glossary

| **Term** | **Definition** |
|----------|---------------|
| **Profile Projection** | Computed view of user profile with social graph state |
| **Locked Projection** | Limited profile view for private accounts (non-followers) |
| **Dual Profile** | User + Chef profiles under one account |
| **Social Graph Status** | Relationship state (following, blocked, requested) |
| **Username Cooldown** | 15-day waiting period between username changes |
| **Presigned URL** | Time-limited S3 upload URL for client-side uploads |
| **Cache TTL** | Time-to-live for cached profile data (5 minutes) |

---

**[END OF FEATURE OVERVIEW]**

---

**Next Document:** `TECHNICAL_GUIDE.md` (Developer Reference)
