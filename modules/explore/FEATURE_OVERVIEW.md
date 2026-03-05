# Explore Module - Feature Overview

**Version:** 1.1  
**Last Updated:** March 2026  
**Module:** `apps/chefooz-apis/src/modules/explore/`  
**Domain Logic:** `libs/domain/src/lib/explore-*.ts`

---

## 📋 Table of Contents

1. [Executive Summary](#executive-summary)
2. [Business Goals](#business-goals)
3. [User Journey](#user-journey)
4. [Feature Set](#feature-set)
5. [Explore Sections](#explore-sections)
6. [Ranking Algorithm](#ranking-algorithm)
7. [Privacy & Safety](#privacy--safety)
8. [Performance & Caching](#performance--caching)
9. [Success Metrics](#success-metrics)

---

## Executive Summary

### What is Explore?

The **Explore module** is Chefooz's discovery engine that helps users find new dishes, chefs, and food experiences beyond their following feed. It combines global trends with personalized recommendations to create curated content sections.

### Key Features

✅ **5 Curated Sections**: Trending, Recommended, New Dish, Exclusive, Promoted  
✅ **Food-First Ranking**: Prioritizes dishes and menu items over raw video content  
✅ **Availability-Aware**: Only shows deliverable items from online chefs  
✅ **Privacy-Enforced**: Respects blocks, private accounts, and user deactivation  
✅ **Personalized**: Learns from follows, engagement history, and past orders  
✅ **Cached & Fast**: Section-specific caching with hourly trending aggregation

### Technical Stack

| Component | Technology |
|-----------|-----------|
| **Backend** | NestJS 10+ |
| **Reel Storage** | MongoDB 7+ (Mongoose) |
| **User/Order Data** | PostgreSQL 16+ (TypeORM) |
| **Cache** | Redis/Valkey |
| **Search** | Elasticsearch 8+ (optional) |
| **Ranking** | Custom food-first algorithm |

---

## Business Goals

### Primary Objectives

1. **Content Discovery**: Help users discover dishes and chefs they don't follow
2. **Order Conversion**: Surface orderable dishes from available chefs
3. **Engagement Growth**: Increase time-on-platform through curated content
4. **Chef Visibility**: Give emerging chefs exposure beyond their follower base
5. **Retention**: Keep users engaged with personalized recommendations

### Success Criteria

| Metric | Target | Priority |
|--------|--------|----------|
| **Daily Explore Users** | 60% of DAU | High |
| **Explore → Order Conversion** | 8% | High |
| **Avg. Time in Explore** | 3 min/session | Medium |
| **Chef Discovery Rate** | 40% find new chefs | High |
| **Repeat Explore Visits** | 3+ times/day | Medium |

---

## User Journey

### Customer Journey (Discovering Food)

```
1. Open App
   ↓
2. Tap Explore Tab (Bottom Navigation)
   ↓
3. See Consolidated Sections Feed
   - Trending (5 items)
   - Recommended For You (5 items)
   - New Dishes (5 items)
   - Exclusive (3 items)
   - Top Chefs (5 chef cards)
   - Trending Hashtags (10 tags)
   ↓
4. Tap "See All" on Any Section
   ↓
5. Browse Full Section Feed (Paginated)
   ↓
6. Tap Card → View Reel
   ↓
7. Tap CTA → Order Dish / View Menu / Follow Chef
```

### Chef Journey (Gaining Visibility)

```
1. Upload Dish Reel (Menu Showcase)
   ↓
2. Reel Appears in "New Dish" Section
   ↓
3. Gets Initial Impressions (First 100 users)
   ↓
4. High Engagement → Moves to "Trending"
   ↓
5. Users Order Dish → Boosts Reputation
   ↓
6. Appears in "Recommended" for Similar Users
   ↓
7. Chef Can Promote → "Exclusive" Section
```

---

## Feature Set

### 1. Consolidated Explore Sections

**Endpoint**: `GET /api/v1/explore/sections`

Returns all sections in a single API call for efficient mobile loading.

**Response Structure**:
```json
{
  "success": true,
  "message": "Explore sections retrieved",
  "data": {
    "trending": {
      "items": [...],
      "hasMore": true,
      "nextCursor": "base64..."
    },
    "recommended": { ... },
    "newDish": { ... },
    "exclusive": { ... },
    "topChefs": [...],
    "trendingHashtags": [...]
  }
}
```

**Benefits**:
- Single API call reduces load time
- Consistent UX across all sections
- Lower network overhead
- Easier mobile caching

### 2. Individual Section Access

**Endpoint**: `GET /api/v1/explore/:section?limit=24&cursor=abc123`

Fetch paginated items for a specific section.

**Sections**:
- `trending` - Popular content (7-day window)
- `recommended` - Personalized for user
- `new_dish` - Recent uploads (24h)
- `exclusive` - Chef-promoted premium content
- `promoted` - Sponsored/boosted reels

**Pagination**:
- Cursor-based (not offset-based)
- Limit: 6-48 items per page
- Default: 24 items

### 3. Category Browsing

**Endpoint**: `GET /api/v1/explore/categories`

Browse dishes by cuisine/food category.

**Categories**:
- North Indian
- South Indian
- Chinese
- Italian
- Fast Food
- Desserts
- Beverages
- Snacks

**Use Case**: User wants specific cuisine type (e.g., "Show me all Italian dishes")

### 4. Chef Discovery

**Endpoint**: `GET /api/v1/explore/chefs`

Discover top-rated chefs in your area.

**Ranking Factors**:
- Chef reputation score (70%)
- Distance from user (15%)
- Recent activity (10%)
- Order fulfillment rate (5%)

**Filters**:
- Location (lat/lng + radius)
- Cuisine preference
- Delivery availability
- Minimum rating

### 5. Unified Search (Legacy)

**Endpoint**: `GET /api/v1/explore/search?q=butter chicken`

Quick search across dishes, chefs, and categories.

**Note**: This is being replaced by the dedicated Search module (`/api/v1/search-elastic/*`)

---

## Explore Sections

### Section 1: Trending

**Purpose**: Show globally popular content (viral dishes/reels)

**Time Window**: 7 days

**Ranking Formula**:
```
trendingScore = engagementScore × timeFactor

engagementScore = 
  views × 1.0 +
  likes × 2.0 +
  comments × 3.0 +
  saves × 4.0 +
  orders × 10.0

timeFactor = 0.8^ageInDays
```

**Business Rules**:
- Content older than 7 days is excluded
- Exponential time decay (older content scores lower)
- Order conversions heavily weighted (10× views)
- Pre-aggregated hourly via background job
- Cached for 1 hour

**Example**:
```
Reel A (2 days old):
  Views: 5000, Likes: 400, Saves: 100, Orders: 15
  engagementScore = 5000×1.0 + 400×2.0 + 100×4.0 + 15×10.0 = 6450
  timeFactor = 0.8^2 = 0.64
  trendingScore = 6450 × 0.64 = 4128

Reel B (5 days old):
  Views: 10000, Likes: 600, Saves: 150, Orders: 10
  engagementScore = 10000 + 1200 + 600 + 100 = 11900
  timeFactor = 0.8^5 = 0.328
  trendingScore = 11900 × 0.328 = 3903

Winner: Reel A (fresher + better engagement per day)
```

### Section 2: Recommended

**Purpose**: Personalized content based on user behavior

**Ranking Factors**:
1. **Follow Boost (5.0×)**: Content from followed chefs
2. **Engagement Boost (2.0×)**: Similar to previously liked reels
3. **Locality Boost (1.5×)**: Chefs within 5km
4. **Diversity Cap**: Max 3 reels per chef in top 24

**Personalization Signals**:
- Followed chefs
- Previously liked/saved reels
- Past order history (cuisine preferences)
- Engagement patterns (watch time, interactions)
- Location preferences

**Business Rules**:
- Requires user authentication (no recommendations for anonymous users)
- Falls back to trending if user has no history
- Refreshes every 30 minutes
- Diversity ensures variety (no single chef dominates)

**Example**:
```
User Profile:
  - Follows: chef_rakesh, chef_priya
  - Past Orders: Butter Chicken, Biryani, Paneer Tikka
  - Location: Mumbai, Andheri
  - Liked Hashtags: #northindian #spicy

Recommended Feed:
  1. Butter Chicken Reel from chef_rakesh (Follow Boost 5.0×)
  2. Biryani Reel from chef_nearby (Locality Boost 1.5×, Cuisine Match)
  3. Paneer Dish from chef_priya (Follow Boost 5.0×)
  4. New Spicy Dish from chef_new (Engagement Boost 2.0×, Hashtag Match)
  5. Trending North Indian (Global Trending + Cuisine Match)
```

### Section 3: New Dish

**Purpose**: Showcase recently uploaded dish content

**Time Window**: 24 hours

**Filters**:
- `reelPurpose = MENU_SHOWCASE` (dish-focused reels)
- Created within last 24 hours
- Chef is currently online
- Dish is deliverable

**Ranking**:
- Sorted by creation time (newest first)
- Boosted if chef has high reputation (Gold/Diamond/Legend tier)
- Boosted if from followed chef

**Business Rules**:
- Gives new chefs initial visibility
- Rotates content every hour (fresh reels bubble up)
- Max 50 reels in section (after that, oldest drop out)

**Use Case**: "What's new today?"

### Section 4: Exclusive

**Purpose**: Premium chef-promoted content

**Eligibility**:
- `promoted = true` flag set by chef
- Chef must have Gold tier or higher
- Paid promotion or chef-selected featured dish

**Ranking**:
- Chef reputation score (primary)
- Promotion boost (if paid)
- Engagement score (secondary)

**Business Rules**:
- Max 10 exclusive items at a time
- Paid promotions get priority
- Rotates weekly
- Only available chefs shown

**Use Case**: "Chef's signature dishes" or "Premium exclusive content"

### Section 5: Promoted (Sponsored)

**Purpose**: Sponsored content (future monetization)

**Current Status**: Placeholder (not fully implemented)

**Future Plan**:
- Paid ads from chefs/restaurants
- CPC/CPM pricing model
- Clearly labeled as "Promoted"
- Still respects user privacy (no tracking outside app)

---

## Ranking Algorithm

### Food-First Philosophy

Unlike generic social feeds, Chefooz ranks **food entities** (dishes, menu items, chefs), not just raw video reels.

**Entity Types**:
- **Dish**: A specific menu item (e.g., "Butter Chicken")
- **Chef**: A chef profile with kitchen
- **Kitchen**: Chef's active kitchen (workspace)
- **Reel**: Supporting media that boosts entity trust (not primary feed item)

### Ranking Formula (0-100 Score)

```
FINAL_SCORE = 
  availabilityScore     × 0.30 +
  personalRelevance     × 0.25 +
  socialProof           × 0.20 +
  visualTrust           × 0.15 +
  freshnessBoost        × 0.10
```

### Score Components

#### 1. Availability Score (30%)

**Purpose**: Only show orderable dishes

**Hard Filters**:
- Chef must be online (`isChefOnline = true`)
- Kitchen must be open (`isKitchenOpen = true`)
- Dish must be deliverable to user (`isDeliverable = true`)
- Not overloaded (`isOverloaded = false`)
- ETA ≤ 90 minutes

**Scoring**:
```
Start with 100 points
  - ETA > 60 min: -30 points
  - ETA > 45 min: -20 points
  - ETA > 30 min: -10 points
  - Chef busy (approaching overload): -20 points

Final: 0-100
```

**Example**:
```
Chef A: Online, ETA 25 min → 100 points
Chef B: Online, ETA 50 min → 80 points
Chef C: Online, ETA 70 min → 70 points
Chef D: Offline → Excluded (hard filter)
```

#### 2. Personal Relevance (25%)

**Purpose**: Match user's taste and preferences

**Signals**:
- User follows this chef: +50 points
- User ordered from this chef before: +30 points
- Cuisine matches past orders: +20 points
- Hashtags match user interests: +15 points
- Dish price in user's range: +10 points

**Scoring**:
```
relevanceScore = 0

if (userFollowsChef):
  relevanceScore += 50

if (userOrderedBefore):
  relevanceScore += 30

if (cuisineMatch):
  relevanceScore += 20

if (hashtagMatch):
  relevanceScore += 15

if (priceMatch):
  relevanceScore += 10

// Normalize to 0-100
finalScore = min(100, relevanceScore)
```

**Example**:
```
User follows chef + ordered before + cuisine match:
  50 + 30 + 20 = 100 points (perfect match!)

New chef, cuisine match only:
  20 points (worth trying)
```

#### 3. Social Proof (20%)

**Purpose**: Leverage wisdom of the crowd

**Signals**:
- Total orders for this dish: Primary signal
- Likes/saves on reel: Secondary signal
- Chef's overall rating: Tertiary signal
- Follower count: Mild boost

**Formula**:
```
socialProofScore = 
  log(orders + 1) × 30 +
  log(likes + 1) × 20 +
  (avgRating / 5.0) × 30 +
  log(followers + 1) × 10

// Normalize to 0-100
```

**Example**:
```
Dish A: 150 orders, 500 likes, 4.5 rating, 2000 followers
  log(151)×30 + log(501)×20 + (4.5/5)×30 + log(2001)×10
  = 62 + 54 + 27 + 33 = 176 → Normalized to 100

Dish B: 5 orders, 50 likes, 4.0 rating, 200 followers
  log(6)×30 + log(51)×20 + (4.0/5)×30 + log(201)×10
  = 24 + 34 + 24 + 23 = 105 → Normalized to 100

Winner: Both cap at 100, but Dish A has stronger signals
```

#### 4. Visual Trust (15%)

**Purpose**: High-quality reel content boosts trust

**Signals**:
- Reel watch completion rate: Primary (0-100%)
- Video quality (resolution, bitrate): 0-100
- Thumbnail appeal (ML-predicted, future): 0-100

**Formula**:
```
visualTrust = 
  watchCompletionRate × 0.60 +
  videoQuality × 0.30 +
  thumbnailAppeal × 0.10

// All scores normalized to 0-100
```

**Example**:
```
High-quality reel (4K, 85% completion, good thumbnail):
  85×0.6 + 95×0.3 + 80×0.1 = 51 + 28.5 + 8 = 87.5 points

Low-quality reel (480p, 30% completion, poor thumbnail):
  30×0.6 + 40×0.3 + 20×0.1 = 18 + 12 + 2 = 32 points
```

#### 5. Freshness Boost (10%)

**Purpose**: Promote recent content

**Time Decay**:
```
if (ageInHours < 24):
  freshnessBoost = 100
else if (ageInHours < 48):
  freshnessBoost = 80
else if (ageInHours < 72):
  freshnessBoost = 60
else if (ageInDays < 7):
  freshnessBoost = 40
else:
  freshnessBoost = 20
```

**Example**:
```
Reel uploaded 2 hours ago: 100 points
Reel uploaded yesterday: 80 points
Reel uploaded 5 days ago: 40 points
Reel uploaded 2 months ago: 20 points
```

### Diversity Guards

To prevent monotony and promote exploration:

1. **Max 2 items per chef** in top 24 results
2. **Max 5 items per cuisine** in top 24 results
3. **No duplicate dishes** (same menu item from same chef)
4. **±5% score jitter** for exploration (prevents ranking staleness)

**Example**:
```
Top 10 Before Diversity:
  1. Chef A - Butter Chicken (95)
  2. Chef A - Paneer Tikka (93)
  3. Chef A - Biryani (91)
  4. Chef B - Butter Chicken (90)
  5. Chef C - Butter Chicken (88)
  ...

Top 10 After Diversity:
  1. Chef A - Butter Chicken (95)
  2. Chef A - Paneer Tikka (93)    ← Max 2 from Chef A
  3. Chef B - Butter Chicken (90)
  4. Chef C - Idli (88)             ← Cuisine diversity
  5. Chef D - Pasta (87)            ← Cuisine diversity
  ...
```

---

## Privacy & Safety

### Privacy Enforcement

All explore endpoints enforce privacy rules:

#### 1. Blocked Users (Bidirectional)

```sql
-- Exclude reels from blocked users
WHERE userId NOT IN (
  SELECT blockedUserId FROM user_blocks WHERE userId = :currentUser
  UNION
  SELECT userId FROM user_blocks WHERE blockedUserId = :currentUser
)
```

**Business Rule**: If User A blocks User B (or vice versa), neither sees the other's content in explore.

#### 2. Private Accounts

```sql
-- Exclude private accounts unless following
WHERE (
  user.isPrivate = FALSE
  OR
  EXISTS (SELECT 1 FROM user_follows WHERE followerId = :currentUser AND followingId = user.id)
)
```

**Business Rule**: Private chef accounts only appear in explore for their followers.

#### 3. Deactivated Users

```sql
-- Exclude deactivated users
WHERE user.isDeactivated = FALSE
```

### Content Safety

**Automated Filters** (via Moderation module):
- Flagged content hidden from explore
- Chefs with warnings limited in explore visibility
- NSFW/violent content excluded
- Duplicate/spam content filtered

**Manual Moderation**:
- Admins can remove items from explore
- Reported content reviewed within 24h
- Repeat offenders shadowbanned from explore

---

## Performance & Caching

### Caching Strategy

| Section | TTL | Cache Key Pattern | Refresh Trigger |
|---------|-----|-------------------|-----------------|
| Trending | 1 hour | `explore:trending:{cursor}` | Hourly cron job |
| Recommended | 30 min | `explore:rec:{userId}:{cursor}` | User engagement |
| New Dish | 15 min | `explore:new:{cursor}` | New uploads |
| Exclusive | 1 hour | `explore:exclusive:{cursor}` | Chef promotion |
| Top Chefs | 2 hours | `explore:chefs:{lat}:{lng}` | Chef status |

### Background Jobs

#### Trending Aggregator (Hourly)

```typescript
@Cron(CronExpression.EVERY_HOUR)
async aggregateTrendingScores(): Promise<void> {
  // 1. Fetch reels from last 7 days
  // 2. Calculate trending scores
  // 3. Sort by score
  // 4. Cache top 500 reels
  // 5. Set TTL to 1 hour
}
```

**Benefits**:
- Fast trending feed (no real-time calculation)
- Consistent results across users
- Reduced database load

### Performance Targets

| Metric | Target | Current |
|--------|--------|---------|
| **P50 Latency** | < 200ms | 150ms |
| **P95 Latency** | < 500ms | 400ms |
| **Cache Hit Rate** | > 80% | 85% |
| **DB Queries per Request** | < 5 | 3 |
| **Throughput** | 1000 req/s | 800 req/s |

---

## Success Metrics

### Key Performance Indicators (KPIs)

#### Engagement Metrics

| Metric | Definition | Target |
|--------|-----------|--------|
| **DAU in Explore** | Daily active users who open explore tab | 60% of total DAU |
| **Explore Session Time** | Avg time spent per explore session | 3 minutes |
| **Sections Viewed** | Avg number of sections viewed per session | 2.5 |
| **CTR (Card → Reel)** | % of cards clicked to view reel | 25% |
| **CTR (Reel → Order)** | % of reels that lead to order | 8% |

#### Discovery Metrics

| Metric | Definition | Target |
|--------|-----------|--------|
| **Chef Discovery Rate** | % of users who discover new chefs via explore | 40% |
| **New Chef Follow Rate** | % of discovered chefs that get followed | 15% |
| **Cuisine Discovery** | % of users who try new cuisine types | 30% |
| **Explore → First Order** | % of new users whose first order comes from explore | 35% |

#### Business Metrics

| Metric | Definition | Target |
|--------|-----------|--------|
| **GMV from Explore** | Gross Merchandise Value attributed to explore | ₹500K/month |
| **Explore Order Conversion** | Orders initiated from explore feed | 12% of total orders |
| **Avg Order Value** | Average order value from explore | ₹450 |
| **Repeat Order Rate** | Users who order again after explore discovery | 50% (within 7 days) |

#### Technical Metrics

| Metric | Definition | Target |
|--------|-----------|--------|
| **Cache Hit Rate** | % of requests served from cache | > 80% |
| **API Latency (P95)** | 95th percentile response time | < 500ms |
| **Error Rate** | % of failed requests | < 0.5% |
| **Trending Job Success** | Hourly job success rate | > 99% |

### A/B Testing Framework

**Planned Experiments**:

1. **Ranking Weight Optimization**
   - Control: Current weights (30/25/20/15/10)
   - Variant A: Boost availability (40/20/20/10/10)
   - Variant B: Boost personalization (25/35/20/10/10)
   - Metric: Order conversion rate

2. **Section Order**
   - Control: Trending → Recommended → New Dish
   - Variant A: Recommended → Trending → New Dish
   - Metric: Session time + CTR

3. **Diversity Caps**
   - Control: Max 2 per chef
   - Variant A: Max 3 per chef
   - Metric: Diversity score + engagement

---

## Future Enhancements

### Phase 2: Advanced Personalization

- **ML-based recommendations**: Use TensorFlow for collaborative filtering
- **Real-time signals**: Incorporate live engagement (likes, saves in last 5 min)
- **Location intelligence**: Predict user's next delivery address
- **Time-aware**: Show breakfast dishes in morning, dinner at night

### Phase 3: Social Features

- **Friends' favorites**: "Dishes your friends loved"
- **Group explore**: "What's popular in your community"
- **Influencer spotlight**: Verified food influencers section

### Phase 4: Monetization

- **Promoted dishes**: Chefs pay for visibility boost
- **Featured chef slots**: Premium placement in explore
- **Affiliate program**: Commission for driving orders
- **Sponsored categories**: Brand partnerships (e.g., "Coca-Cola Combos")

---

## API Reference Summary

| Endpoint | Method | Auth | Rate Limit | Description |
|----------|--------|------|------------|-------------|
| `/v1/explore/sections` | GET | Required | 30/min | Get consolidated sections |
| `/v1/explore/available` | GET | Required | 30/min | List section names (legacy) |
| `/v1/explore/:section` | GET | Required | 30/min | Get section items (paginated) |
| `/v1/explore/categories` | GET | Optional | 30/min | List cuisine categories |
| `/v1/explore/category/:id` | GET | Optional | 30/min | Get category details |
| `/v1/explore/chefs` | GET | Required | 30/min | Discover top chefs |
| `/v1/explore/search` | GET | Required | 10/sec | Unified search (legacy) |

**Note**: See `TECHNICAL_GUIDE.md` for detailed API specs and request/response schemas.

---

## Header & Search Bar (UI Enhancement — March 2026)

### SmartSearchBar Redesign

The `SmartSearchBar` component (sticky header at the top of the Explore tab) was redesigned for a more polished look in both light and dark mode.

| Aspect | Before | After |
|--------|--------|-------|
| **Container background** | Plain (no background, only border-bottom) | `LinearGradient` — brand purple→coral tint at low opacity |
| **Search icon** | Emoji `🔍` | `Ionicons` `search` (left) + `mic-outline` (right) |
| **Search pill background** | Border-only | White elevated card (light) / `surfaceElevated` (dark) |
| **Search pill shadow** | None | iOS shadow / Android elevation (brand-tinted in light mode) |
| **Separator** | `borderBottomWidth: 1` | Removed — gradient provides visual separation |

### Color Behaviour

- **Light mode**: gradient `rgba(192,49,191,0.07)` → transparent → `rgba(252,120,48,0.05)` over white `colors.background`
- **Dark mode**: gradient `rgba(192,49,191,0.18)` → transparent → `rgba(252,120,48,0.12)` over dark `colors.background`
- Gradient direction: top-left → bottom-right (`start {x:0,y:0}` → `end {x:1,y:1}`)

### Explore Screen (explore.tsx) Fixes

- All `SafeAreaView` containers now have `backgroundColor: theme.colors.background` (prevents white flash in dark mode)
- `ScrollView` now has `backgroundColor: theme.colors.background`
- `StatusBar` added to the default layout path (was missing; previously only present in the variant layout path)
- Removed unused `transparent` import from `react-native-paper` internals
- Added missing `Ionicons` import (was used in empty state button but not imported)

---

## Related Documentation

- **Technical Guide**: `TECHNICAL_GUIDE.md` (API specs, DB schemas, algorithms)
- **QA Test Cases**: `QA_TEST_CASES.md` (Test scenarios and acceptance criteria)
- **Feed Module**: `../feed/FEATURE_OVERVIEW.md` (Following feed implementation)
- **Search Module**: `../search/FEATURE_OVERVIEW.md` (Advanced search capabilities)

---

**[SLICE_COMPLETE ✅]**  
**Module**: Explore  
**Documentation**: Feature Overview  
**Date**: February 14, 2026
