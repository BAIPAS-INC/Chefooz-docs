# Search Module - Feature Overview

**Version:** 1.0  
**Last Updated:** February 14, 2026  
**Module:** `apps/chefooz-apis/src/modules/search-elastic/`  
**Legacy Module:** `apps/chefooz-apis/src/modules/search/` (PostgreSQL-based)  
**Tech Stack:** Elasticsearch 8+, NestJS, MongoDB, PostgreSQL

---

## 📋 Table of Contents

1. [Executive Summary](#executive-summary)
2. [Business Goals](#business-goals)
3. [User Journey](#user-journey)
4. [Feature Set](#feature-set)
5. [Search Types](#search-types)
6. [Ranking & Relevance](#ranking--relevance)
7. [Elasticsearch Architecture](#elasticsearch-architecture)
8. [Fallback Strategy](#fallback-strategy)
9. [Success Metrics](#success-metrics)

---

## Executive Summary

### What is Search?

The **Search module** enables users to quickly find dishes, chefs, and menu items through full-text search with advanced filters. It uses **Elasticsearch** for fast, fuzzy, and multilingual search with fallback to PostgreSQL/MongoDB when Elasticsearch is unavailable.

### Key Features

✅ **Multi-Index Search**: Dishes (reels), menu items, chefs (users)  
✅ **Full-Text Search**: Captions, hashtags, names, bios, descriptions  
✅ **Fuzzy Matching**: Typo-tolerant (e.g., "butr chiken" → "butter chicken")  
✅ **Advanced Filters**: Location, price, rating, cuisine, hashtags  
✅ **Auto-Suggestions**: Real-time search suggestions as user types  
✅ **Fallback Mode**: PostgreSQL/MongoDB when ES unavailable  
✅ **Search Analytics**: Track queries, trending searches, CTR

### Technical Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **Search Engine** | Elasticsearch 8+ | Full-text search, fuzzy matching |
| **Fallback** | PostgreSQL + MongoDB | When ES unavailable |
| **Backend** | NestJS 10+ | REST API endpoints |
| **Indexing** | Background jobs (cron) | Sync DB → Elasticsearch |
| **Analytics** | MongoDB | Search query logs, trending |

---

## Business Goals

### Primary Objectives

1. **Fast Discovery**: Help users find specific dishes/chefs in < 500ms
2. **Order Conversion**: Surface orderable dishes with clear CTAs
3. **Typo Tolerance**: Handle common misspellings ("paner tikka", "byrani")
4. **Local Results**: Prioritize nearby chefs with deliverable items
5. **Trending Insights**: Track what users search for to improve recommendations

### Success Criteria

| Metric | Target | Priority |
|--------|--------|----------|
| **Search Latency (P95)** | < 500ms | Critical |
| **Search → Order Conversion** | 12% | High |
| **Zero Result Rate** | < 10% | High |
| **Typo Correction Rate** | > 85% | Medium |
| **Daily Search Users** | 40% of DAU | Medium |

---

## User Journey

### Customer Journey (Finding Food)

```
1. Tap Search Icon (Top Navigation)
   ↓
2. See Search Bar + Trending Searches
   ↓
3. Start Typing "butter chi..."
   ↓
4. See Auto-Suggestions:
   - "butter chicken"
   - "butter chicken gravy"
   - "butter chicken masala"
   ↓
5. Select Suggestion or Press Search
   ↓
6. See Results (3 Tabs):
   - Dishes (20 reels)
   - Chefs (10 profiles)
   - Menu Items (15 dishes)
   ↓
7. Apply Filters:
   - Price: ₹200-₹400
   - Rating: 4+ stars
   - Distance: < 5 km
   ↓
8. Tap Dish → View Reel → Order
```

### Chef Journey (Appearing in Search)

```
1. Chef Creates Menu Item "Butter Chicken"
   - Name: "Butter Chicken"
   - Description: "Creamy tomato-based curry with tender chicken"
   - Tags: #butterchicken #northindian #curry
   ↓
2. Chef Uploads Reel (Menu Showcase)
   - Caption: "Making the perfect butter chicken!"
   - Hashtags: #butterchicken #cooking
   ↓
3. Background Job Indexes to Elasticsearch
   - Dish indexed in `dishes` index
   - Menu item indexed in `menu_items` index
   - Chef profile indexed in `users` index
   ↓
4. User Searches "butter chicken"
   ↓
5. Chef's Dish Appears in Results (Ranked by relevance + availability)
```

---

## Feature Set

### 1. Dish Search (Reels)

**Endpoint**: `GET /api/v1/search/dishes?q=butter chicken`

**Search Fields**:
- `caption` (3× boost)
- `hashtags` (2× boost)
- `chefName` (1× boost)

**Filters**:
- `hashtags`: Filter by specific hashtags
- `chefIds`: Filter by specific chefs
- `promotedOnly`: Only promoted reels
- `reelPurpose`: `MENU_SHOWCASE`, `STORY`, etc.
- `lat`, `lng`, `radiusKm`: Geospatial filter

**Sort Options**:
- `relevance` (default): Elasticsearch score + boosts
- `recent`: Newest first
- `popular`: Most views/likes
- `trending`: High engagement + recent

**Response**:
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "id": "reel-id",
        "userId": "chef-uuid",
        "caption": "Making the perfect butter chicken!",
        "hashtags": ["butterchicken", "cooking"],
        "thumbnailUrl": "https://...",
        "videoUrl": "https://...",
        "stats": { "views": 5000, "likes": 400 },
        "author": {
          "username": "chef_rakesh",
          "fullName": "Rakesh Kumar"
        },
        "score": 4.2,
        "highlights": {
          "caption": ["Making the perfect <em>butter chicken</em>!"]
        }
      }
    ],
    "total": 156,
    "page": 1,
    "limit": 20,
    "hasMore": true
  }
}
```

### 2. Menu Item Search

**Endpoint**: `GET /api/v1/search/menu-items?q=butter chicken`

**Search Fields**:
- `name` (3× boost)
- `description` (2× boost)
- `tags` (2× boost)
- `chefName` (1× boost)

**Filters**:
- `minPrice`, `maxPrice`: Price range
- `minRating`: Minimum rating (0-5)
- `cuisines`: Filter by cuisine types
- `isVegetarian`, `isVegan`, `isGlutenFree`: Dietary filters
- `lat`, `lng`, `radiusKm`: Geospatial

**Response**:
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "id": "menu-item-uuid",
        "name": "Butter Chicken",
        "description": "Creamy tomato-based curry with tender chicken",
        "price": 350,
        "currency": "INR",
        "prepTimeMinutes": 30,
        "rating": 4.5,
        "reviewCount": 120,
        "chef": {
          "userId": "chef-uuid",
          "username": "chef_rakesh",
          "fullName": "Rakesh Kumar",
          "distance": 2.5
        },
        "images": ["https://..."],
        "tags": ["butterchicken", "northindian"],
        "isAvailable": true,
        "score": 5.8
      }
    ],
    "total": 45,
    "page": 1,
    "limit": 20,
    "hasMore": true
  }
}
```

### 3. Chef Search (Users)

**Endpoint**: `GET /api/v1/search/chefs?q=chef rakesh`

**Search Fields**:
- `username` (3× boost)
- `fullName` (3× boost)
- `bio` (1× boost)
- `cuisineSpecialties` (2× boost)

**Filters**:
- `minRating`: Minimum chef rating
- `isVerified`: Verified chefs only
- `lat`, `lng`, `radiusKm`: Geospatial

**Response**:
```json
{
  "success": true,
  "data": {
    "items": [
      {
        "userId": "chef-uuid",
        "username": "chef_rakesh",
        "fullName": "Rakesh Kumar",
        "avatarUrl": "https://...",
        "bio": "20 years experience in North Indian cuisine",
        "reputation": {
          "tier": "gold",
          "score": 72
        },
        "stats": {
          "followers": 2500,
          "totalOrders": 1200,
          "avgRating": 4.5
        },
        "isOnline": true,
        "isVerified": true,
        "distance": 2.5,
        "score": 6.1
      }
    ],
    "total": 12,
    "page": 1,
    "limit": 20,
    "hasMore": false
  }
}
```

### 4. Auto-Suggest

**Endpoint**: `GET /api/v1/search/suggest?q=butt`

**Purpose**: Real-time search suggestions as user types (minimum 3 characters)

**Response**:
```json
{
  "success": true,
  "data": {
    "suggestions": [
      {
        "text": "butter chicken",
        "type": "dish",
        "frequency": 1500
      },
      {
        "text": "butter chicken gravy",
        "type": "dish",
        "frequency": 350
      },
      {
        "text": "butter naan",
        "type": "dish",
        "frequency": 800
      }
    ]
  }
}
```

**Features**:
- Prefix matching ("butt" → "butter chicken")
- Frequency-based ranking (popular suggestions first)
- Type hints (dish, chef, cuisine)
- Min 3 characters to trigger

### 5. Legacy User Search

**Endpoint**: `GET /api/v1/search/users?q=rakesh`

**Purpose**: Simple PostgreSQL-based user search (fallback)

**Search Fields**:
- `username` (ILIKE match)
- `fullName` (ILIKE match)
- `bio` (ILIKE match)

**Response**: Simplified user objects (no Elasticsearch features)

---

## Search Types

### Type 1: Keyword Search

**Example**: "butter chicken"

**Elasticsearch Query**:
```json
{
  "multi_match": {
    "query": "butter chicken",
    "fields": ["caption^3", "hashtags^2"],
    "fuzziness": "AUTO",
    "operator": "or"
  }
}
```

**Features**:
- Full-text match
- Fuzzy matching (typo tolerance)
- Field boosting (caption 3×, hashtags 2×)
- OR operator (matches "butter" OR "chicken")

### Type 2: Phrase Search

**Example**: `"butter chicken curry"`

**Elasticsearch Query**:
```json
{
  "match_phrase": {
    "caption": "butter chicken curry"
  }
}
```

**Features**:
- Exact phrase match (preserves word order)
- No fuzzy matching
- Higher precision, lower recall

### Type 3: Boolean Search

**Example**: `butter AND chicken`

**Elasticsearch Query**:
```json
{
  "bool": {
    "must": [
      { "match": { "caption": "butter" } },
      { "match": { "caption": "chicken" } }
    ]
  }
}
```

**Features**:
- Both terms required
- AND/OR/NOT operators
- Advanced users only

### Type 4: Hashtag Search

**Example**: `#butterchicken`

**Implementation**: Strip `#` and search `hashtags` field exactly

### Type 5: Geospatial Search

**Example**: "butter chicken near me"

**Elasticsearch Query**:
```json
{
  "bool": {
    "must": { "match": { "caption": "butter chicken" } },
    "filter": {
      "geo_distance": {
        "distance": "5km",
        "location": { "lat": 19.0760, "lon": 72.8777 }
      }
    }
  }
}
```

**Features**:
- Radius filter (5km, 10km, etc.)
- Distance sorting (nearest first)
- Geohash indexing for performance

---

## Ranking & Relevance

### Elasticsearch Scoring Formula

```
_score = (TF/IDF × field_boost) + custom_boosts
```

**Term Frequency (TF)**: How often the search term appears  
**Inverse Document Frequency (IDF)**: How rare the term is (rare = higher score)  
**Field Boost**: Multiply score by boost factor (e.g., caption^3)

### Custom Boosts

#### 1. Promoted Content Boost (2.0×)

```json
{
  "should": [
    { "term": { "promoted": { "value": true, "boost": 2.0 } } }
  ]
}
```

Promoted reels rank higher (paid promotion or chef-selected featured content).

#### 2. Recency Boost (1.2×)

```json
{
  "should": [
    {
      "range": {
        "createdAt": {
          "gte": "now-7d/d",
          "boost": 1.2
        }
      }
    }
  ]
}
```

Content from last 7 days gets +20% boost (freshness signal).

#### 3. Reputation Boost (1.0-2.0×)

```typescript
const reputationBoost = 1.0 + (chefReputation.score / 100);
// Bronze (score 15): 1.15× boost
// Silver (score 45): 1.45× boost
// Gold (score 72): 1.72× boost
// Diamond (score 85): 1.85× boost
// Legend (score 95): 1.95× boost
```

Higher-reputation chefs rank higher (trust signal).

#### 4. Engagement Boost

```typescript
const engagementBoost = Math.log(likes + saves + orders + 1) / 10;
// 10 likes: +0.2 boost
// 100 likes: +0.4 boost
// 1000 likes: +0.6 boost
```

Popular content ranks higher (social proof).

### Final Score Example

```
Search: "butter chicken"

Reel A:
  - TF/IDF: 3.2
  - Field boost (caption^3): 3.2 × 3 = 9.6
  - Promoted: +2.0
  - Recency (3 days old): +1.2
  - Reputation (Gold): +1.72
  - Engagement (500 likes): +0.5
  Final Score: 9.6 + 2.0 + 1.2 + 1.72 + 0.5 = 15.02

Reel B:
  - TF/IDF: 4.5 (higher match)
  - Field boost (hashtags^2): 4.5 × 2 = 9.0
  - Not promoted: 0
  - Recency (30 days old): 0
  - Reputation (Bronze): +1.15
  - Engagement (50 likes): +0.3
  Final Score: 9.0 + 0 + 0 + 1.15 + 0.3 = 10.45

Winner: Reel A (15.02 > 10.45) due to promotion + freshness + reputation
```

---

## Elasticsearch Architecture

### Indices

| Index Name | Document Type | Fields | Shards | Replicas |
|------------|---------------|--------|--------|----------|
| **`chefooz_reels`** | Reel (dish) | `reelId`, `userId`, `caption`, `hashtags`, `stats`, `location`, `createdAt` | 3 | 1 |
| **`chefooz_menu_items`** | Menu Item | `menuItemId`, `chefId`, `name`, `description`, `price`, `rating`, `tags`, `location` | 2 | 1 |
| **`chefooz_users`** | Chef/User | `userId`, `username`, `fullName`, `bio`, `reputation`, `stats`, `location` | 2 | 1 |

### Index Mappings

#### Reels Index

```json
{
  "mappings": {
    "properties": {
      "reelId": { "type": "keyword" },
      "userId": { "type": "keyword" },
      "caption": {
        "type": "text",
        "fields": {
          "keyword": { "type": "keyword" }
        },
        "analyzer": "standard"
      },
      "hashtags": {
        "type": "keyword",
        "fields": {
          "text": { "type": "text" }
        }
      },
      "stats": {
        "properties": {
          "views": { "type": "integer" },
          "likes": { "type": "integer" },
          "saves": { "type": "integer" },
          "orders": { "type": "integer" }
        }
      },
      "location": { "type": "geo_point" },
      "createdAt": { "type": "date" },
      "promoted": { "type": "boolean" }
    }
  }
}
```

### Indexing Strategy

#### Real-Time Indexing (Future)

```typescript
// On reel creation
@EventListener(ReelCreatedEvent)
async indexNewReel(event: ReelCreatedEvent) {
  await this.elasticsearchService.indexReel({
    reelId: event.reel.id,
    userId: event.reel.userId,
    caption: event.reel.caption,
    hashtags: event.reel.hashtags,
    // ... other fields
  });
}
```

#### Batch Indexing (Current)

**Cron Schedule**: Every 15 minutes

```typescript
@Cron('0 */15 * * * *')
async reindexRecentReels() {
  // 1. Fetch reels updated in last 15 minutes
  const reels = await this.reelModel.find({
    updatedAt: { $gte: new Date(Date.now() - 15 * 60 * 1000) },
  });

  // 2. Bulk index to Elasticsearch
  const bulkBody = reels.flatMap(reel => [
    { index: { _index: 'chefooz_reels', _id: reel._id.toString() } },
    {
      reelId: reel._id.toString(),
      userId: reel.userId,
      caption: reel.caption,
      hashtags: reel.hashtags,
      // ... other fields
    },
  ]);

  await this.elasticsearchClient.bulk({ body: bulkBody });
  this.logger.log(`Indexed ${reels.length} reels`);
}
```

#### Full Reindex (Manual)

**Endpoint**: `POST /api/v1/search/reindex` (Admin only)

```typescript
async reindexAll() {
  // 1. Delete old indices
  await this.elasticsearchClient.indices.delete({
    index: 'chefooz_*',
    ignore_unavailable: true,
  });

  // 2. Create new indices with mappings
  await this.createIndices();

  // 3. Fetch all documents from MongoDB/PostgreSQL
  const reels = await this.reelModel.find().lean();
  const menuItems = await this.menuItemRepo.find();
  const users = await this.userRepo.find();

  // 4. Bulk index
  await this.bulkIndexReels(reels);
  await this.bulkIndexMenuItems(menuItems);
  await this.bulkIndexUsers(users);

  this.logger.log('Full reindex complete');
}
```

---

## Fallback Strategy

### Elasticsearch Unavailable

If Elasticsearch is down or unavailable, search automatically falls back to PostgreSQL/MongoDB.

**Fallback Detection**:
```typescript
isElasticsearchEnabled(): boolean {
  try {
    await this.elasticsearchClient.ping();
    return true;
  } catch (error) {
    this.logger.warn('Elasticsearch unavailable, using fallback');
    return false;
  }
}
```

**Fallback Implementation**:

#### Dish Search (MongoDB)

```typescript
async searchDishesWithMongo(dto: SearchDishesDto) {
  const query: any = {
    $or: [
      { caption: { $regex: dto.query, $options: 'i' } },
      { hashtags: { $in: [dto.query.toLowerCase()] } },
    ],
  };

  const reels = await this.reelModel
    .find(query)
    .sort({ 'stats.likes': -1 }) // Simple sort, no relevance scoring
    .limit(dto.limit)
    .skip((dto.page - 1) * dto.limit)
    .lean();

  return {
    items: reels,
    total: await this.reelModel.countDocuments(query),
    page: dto.page,
    limit: dto.limit,
    hasMore: reels.length === dto.limit,
  };
}
```

**Limitations**:
- ❌ No fuzzy matching (typo tolerance)
- ❌ No relevance scoring (TF/IDF)
- ❌ No highlights
- ❌ Slower (no index optimization)
- ✅ Still functional (graceful degradation)

#### Chef Search (PostgreSQL)

```typescript
async searchChefsWithPostgres(dto: SearchChefsDto) {
  const query = this.userRepo
    .createQueryBuilder('user')
    .where('LOWER(user.username) LIKE LOWER(:query)', { query: `%${dto.query}%` })
    .orWhere('LOWER(user.fullName) LIKE LOWER(:query)', { query: `%${dto.query}%` })
    .orderBy('user.followers', 'DESC')
    .take(dto.limit)
    .skip((dto.page - 1) * dto.limit);

  const [users, total] = await query.getManyAndCount();

  return {
    items: users,
    total,
    page: dto.page,
    limit: dto.limit,
    hasMore: users.length === dto.limit,
  };
}
```

---

## Success Metrics

### Key Performance Indicators (KPIs)

#### Search Performance

| Metric | Definition | Target |
|--------|-----------|--------|
| **Latency (P50)** | 50th percentile response time | < 200ms |
| **Latency (P95)** | 95th percentile response time | < 500ms |
| **Latency (P99)** | 99th percentile response time | < 1000ms |
| **Zero Result Rate** | % of searches with no results | < 10% |
| **Typo Correction Rate** | % of typos auto-corrected | > 85% |

#### User Engagement

| Metric | Definition | Target |
|--------|-----------|--------|
| **Search CTR** | % of searches that result in click | 60% |
| **Search → Order** | % of searches that lead to order | 12% |
| **Avg Results Clicked** | Avg number of results clicked per search | 2.5 |
| **Search Refinement Rate** | % of searches refined with filters | 25% |

#### Business Impact

| Metric | Definition | Target |
|--------|-----------|--------|
| **GMV from Search** | Order value attributed to search | ₹300K/month |
| **Search Users (DAU)** | % of daily users who search | 40% |
| **Avg Order Value** | Average order from search results | ₹400 |
| **Repeat Search Users** | Users who search 3+ times/week | 30% |

---

## Future Enhancements

### Phase 2: ML-Powered Search

- **Semantic Search**: Understand intent ("healthy breakfast" → show protein-rich options)
- **Visual Search**: Upload food photo → find similar dishes
- **Voice Search**: "Order butter chicken from nearby chef"
- **Query Understanding**: Detect cuisine ("Italian food" → pasta, pizza, etc.)

### Phase 3: Personalization

- **Search Ranking**: Personalized based on past orders
- **Query Suggestions**: Based on user's taste preferences
- **Location-Aware**: Auto-apply user's location filter
- **Price-Aware**: Respect user's typical price range

### Phase 4: Advanced Features

- **Spell Correction UI**: "Did you mean: butter chicken?"
- **Related Searches**: "People also searched for..."
- **Trending Searches**: Show what's popular right now
- **Search History**: Quick access to past searches

---

## API Reference Summary

| Endpoint | Method | Auth | Rate Limit | Description |
|----------|--------|------|------------|-------------|
| `/v1/search/dishes` | GET | Required | 10/sec | Search reels/dishes |
| `/v1/search/menu-items` | GET | Required | 10/sec | Search menu items |
| `/v1/search/chefs` | GET | Required | 10/sec | Search chefs/users |
| `/v1/search/suggest` | GET | Required | 20/sec | Auto-suggestions |
| `/v1/search/users` | GET | Required | 10/sec | Legacy user search |
| `/v1/search/reindex` | POST | Admin | N/A | Trigger full reindex |

---

## Related Documentation

- **Technical Guide**: `TECHNICAL_GUIDE.md` (API specs, ES configuration)
- **QA Test Cases**: `QA_TEST_CASES.md` (Test scenarios)
- **Explore Module**: `../explore/FEATURE_OVERVIEW.md` (Discovery features)

---

**[SLICE_COMPLETE ✅]**  
**Module**: Search  
**Documentation**: Feature Overview  
**Date**: February 14, 2026
