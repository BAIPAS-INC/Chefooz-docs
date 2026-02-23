# Search & Ranking Pipeline — Complete Integration Guide

**Version:** 1.0  
**Last Updated:** February 2026  
**Scope:** Full search pipeline — Query → Elasticsearch → Ranking → Fallback → Response  
**Business Rule Summary:** Search defaults to Elasticsearch for full-text search with fuzzy matching and relevance scoring. If Elasticsearch is unavailable, the system gracefully degrades to MongoDB (reels) or PostgreSQL (menu items, users) without throwing errors. The index is synced every 15 minutes via background job. Manual bulk reindex is available for disaster recovery.  
**Rate Limits:** All search endpoints: 10 req/s. Autocomplete suggest: 20 req/s.  
**Key Constraints:** Menu items and users are NOT yet in Elasticsearch (PostgreSQL only). Only the `chefooz_reels` index is fully operational. Future indices (`chefooz_menu_items`, `chefooz_users`) are planned.

---

## 📋 Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Module Structure](#module-structure)
3. [API Endpoints](#api-endpoints)
4. [Elasticsearch Configuration](#elasticsearch-configuration)
5. [Index Mappings](#index-mappings)
6. [Query Construction](#query-construction)
7. [Ranking Algorithm](#ranking-algorithm)
8. [Sort Strategies](#sort-strategies)
9. [Geospatial Search](#geospatial-search)
10. [Fallback Strategy](#fallback-strategy)
11. [Index Sync Jobs](#index-sync-jobs)
12. [Performance Optimization](#performance-optimization)
13. [Frontend Integration](#frontend-integration)
14. [Testing Checklist](#testing-checklist)

---

## Architecture Overview

### Search Pipeline Flow

```
Frontend Query
    │
    ▼
GET /api/v1/search/dishes?query=butter+chicken&lat=19.07&lng=72.87&sortBy=relevance
    │
    ▼
SearchElasticService.searchDishes()
    │
    ├── isElasticsearchEnabled()? ─────────────────────────┐
    │           │ YES                                        │ NO
    │           ▼                                           ▼
    │   Build Bool Query                           MongoDB fallback
    │   ┌─────────────────────┐                   (regex search)
    │   │ must:               │
    │   │   multi_match       │
    │   │   - caption^3       │
    │   │   - hashtags^2      │
    │   │   fuzziness: AUTO   │
    │   ├─────────────────────┤
    │   │ filter:             │
    │   │   - contentType     │
    │   │   - hashtags filter │
    │   │   - geo_distance    │
    │   ├─────────────────────┤
    │   │ should:             │
    │   │   - promoted boost  │
    │   │   - recency boost   │
    │   └─────────────────────┘
    │           │
    │           ▼
    │   Sort + Highlight + Paginate
    │           │
    └───────────▼
    Map ES hits → SearchResultDto
    Return { items[], total, page, hasMore }
```

### Technology Stack

| Component | Technology | Status |
|-----------|-----------|--------|
| **Search Engine** | Elasticsearch 8+ | ✅ Production |
| **Fallback (Reels)** | MongoDB (`$regex` + `$near`) | ✅ Operational |
| **Fallback (Menu Items)** | PostgreSQL (`ILIKE` + JSON filters) | ✅ Operational |
| **Fallback (Users)** | PostgreSQL (`ILIKE` + full-text) | ✅ Operational |
| **Index Sync** | NestJS `@Cron` job (every 15 min) | ✅ Operational |
| **Autocomplete** | Elasticsearch completion suggester | ✅ Operational |

---

## Module Structure

```
apps/chefooz-apis/src/modules/search-elastic/
├── dto/
│   └── search-elastic.dto.ts          # SearchDishesDto, SearchMenuItemsDto, SearchChefsDto, SuggestDto
├── elasticsearch.service.ts           # ES client wrapper (health check, raw client)
├── search-elastic.service.ts          # High-level search logic (queries, ranking, fallback)
├── search-elastic.controller.ts       # HTTP endpoints
├── search-elastic.module.ts           # Module configuration
└── jobs/
    └── index-sync.job.ts              # Background indexing (every 15 min)

apps/chefooz-apis/src/modules/search/ (Legacy — simple user search)
├── dto/
│   └── search-users.dto.ts
├── search.service.ts                  # PostgreSQL-based user search
├── search.controller.ts
└── search.module.ts
```

---

## API Endpoints

### 1. Search Dishes / Reels

**Endpoint**: `GET /api/v1/search/dishes`  
**Auth**: JWT required  
**Rate Limit**: 10 req/s

**Query Parameters**:
```typescript
interface SearchDishesDto {
  query: string;              // Search query (required)
  page?: number;              // Default: 1
  limit?: number;             // Default: 20, max: 50
  sortBy?: 'relevance' | 'recent' | 'popular' | 'trending'; // Default: relevance
  hashtags?: string[];        // Filter by hashtags
  chefIds?: string[];         // Filter by specific chefs
  promotedOnly?: boolean;     // Only promoted content
  reelPurpose?: string;       // 'MENU_SHOWCASE' | 'STORY' | 'TUTORIAL' | 'GENERAL'
  lat?: number;               // Latitude for geospatial
  lng?: number;               // Longitude for geospatial
  radiusKm?: number;          // Radius in kilometers
}
```

**Response**:
```json
{
  "success": true,
  "message": "Search results retrieved successfully",
  "data": {
    "items": [
      {
        "id": "reel-id",
        "userId": "chef-uuid",
        "caption": "Making the perfect butter chicken!",
        "hashtags": ["butterchicken", "cooking", "northindian"],
        "videoUrl": "https://cdn.chefooz.com/reels/...",
        "thumbnailUrl": "https://cdn.chefooz.com/thumbnails/...",
        "durationSec": 30,
        "stats": { "views": 12340, "likes": 890, "comments": 120, "saves": 450 },
        "author": {
          "userId": "uuid",
          "username": "chef_rakesh",
          "fullName": "Rakesh Kumar",
          "avatarUrl": "https://..."
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
    "hasMore": true,
    "source": "elasticsearch"
  }
}
```

**`source` field values**: `"elasticsearch"` | `"mongodb"` (indicates which engine served the result)

---

### 2. Search Menu Items

**Endpoint**: `GET /api/v1/search/menu-items`  
**Auth**: JWT required  
**Rate Limit**: 10 req/s  
**Engine**: PostgreSQL (`ILIKE`) — Elasticsearch index planned

**Query Parameters**:
```typescript
interface SearchMenuItemsDto {
  query: string;
  page?: number;
  limit?: number;
  chefId?: string;            // Filter by specific chef
  minPrice?: number;          // Minimum price in INR
  maxPrice?: number;          // Maximum price in INR
  minRating?: number;         // Minimum rating (0–5)
  cuisines?: string[];        // Filter by cuisine types
  isVegetarian?: boolean;     // Vegetarian only
  isVegan?: boolean;          // Vegan only
  isGlutenFree?: boolean;     // Gluten-free only
  vegOnly?: boolean;          // Legacy vegetarian filter
  availableOnly?: boolean;    // Only currently available items
  lat?: number;
  lng?: number;
  radiusKm?: number;
}
```

---

### 3. Search Chefs

**Endpoint**: `GET /api/v1/search/chefs`  
**Auth**: JWT required  
**Rate Limit**: 10 req/s  
**Engine**: PostgreSQL

**Query Parameters**:
```typescript
interface SearchChefsDto {
  query: string;
  page?: number;
  limit?: number;
  minRating?: number;         // Minimum average rating
  isVerified?: boolean;       // Verified chefs only
  lat?: number;
  lng?: number;
  radiusKm?: number;
}
```

**Response includes**:
- Chef `reputation.tier` (bronze/silver/gold/platinum)
- `reputation.score` (numeric CRS score)
- `distance` in km (if lat/lng provided)
- `isOnline` status

---

### 4. Autocomplete Suggestions

**Endpoint**: `GET /api/v1/search/suggest`  
**Auth**: JWT required  
**Rate Limit**: 20 req/s  
**Min Query Length**: 3 characters

**Query Parameters**:
```typescript
interface SuggestDto {
  query: string;    // Prefix (min 3 chars)
  limit?: number;   // Default: 10
}
```

**Response**:
```json
{
  "success": true,
  "data": {
    "suggestions": [
      { "text": "butter chicken", "type": "dish", "frequency": 1500 },
      { "text": "butter chicken gravy", "type": "dish", "frequency": 820 },
      { "text": "butter naan", "type": "dish", "frequency": 340 }
    ]
  }
}
```

---

## Elasticsearch Configuration

### ElasticsearchService

```typescript
@Injectable()
export class ElasticsearchService {
  private client: Client;

  readonly REELS_INDEX = 'chefooz_reels';
  readonly MENU_ITEMS_INDEX = 'chefooz_menu_items';   // Future
  readonly USERS_INDEX = 'chefooz_users';              // Future

  constructor(private readonly configService: ConfigService) {
    this.client = new Client({
      node: configService.get('ELASTICSEARCH_NODE'),
      auth: {
        username: configService.get('ELASTICSEARCH_USERNAME'),
        password: configService.get('ELASTICSEARCH_PASSWORD'),
      },
    });
  }

  isElasticsearchEnabled(): boolean {
    try {
      await this.client.ping({ requestTimeout: 1000 });
      return true;
    } catch (error) {
      this.logger.warn('Elasticsearch unavailable:', error.message);
      return false;
    }
  }

  getClient(): Client {
    return this.client;
  }
}
```

### Environment Variables

```bash
ELASTICSEARCH_NODE=https://your-es-cluster:9200
ELASTICSEARCH_USERNAME=elastic
ELASTICSEARCH_PASSWORD=changeme
ELASTICSEARCH_INDEX_PREFIX=chefooz_
```

### Cluster Configuration

```json
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "analysis": {
      "analyzer": {
        "custom_text_analyzer": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase", "asciifolding"]
        }
      }
    }
  }
}
```

**`asciifolding`**: Normalizes accented characters (é → e, ñ → n) for better matching across transliterations of Indian dish names.

---

## Index Mappings

### Reels Index (`chefooz_reels`) — Production

```json
{
  "mappings": {
    "properties": {
      "reelId":      { "type": "keyword" },
      "userId":      { "type": "keyword" },
      "caption": {
        "type": "text",
        "analyzer": "custom_text_analyzer",
        "fields": {
          "keyword": { "type": "keyword", "ignore_above": 256 }
        }
      },
      "hashtags": {
        "type": "keyword",
        "fields": {
          "text": { "type": "text", "analyzer": "custom_text_analyzer" }
        }
      },
      "stats": {
        "properties": {
          "views":    { "type": "integer" },
          "likes":    { "type": "integer" },
          "comments": { "type": "integer" },
          "saves":    { "type": "integer" },
          "orders":   { "type": "integer" }
        }
      },
      "location":    { "type": "geo_point" },
      "createdAt":   { "type": "date" },
      "promoted":    { "type": "boolean" },
      "reelPurpose": { "type": "keyword" },
      "contentType": { "type": "keyword" }
    }
  }
}
```

### Menu Items Index (`chefooz_menu_items`) — Planned

```json
{
  "mappings": {
    "properties": {
      "menuItemId":  { "type": "keyword" },
      "chefId":      { "type": "keyword" },
      "name": {
        "type": "text",
        "analyzer": "custom_text_analyzer",
        "fields": { "keyword": { "type": "keyword" } }
      },
      "description": { "type": "text", "analyzer": "custom_text_analyzer" },
      "price":       { "type": "float" },
      "rating":      { "type": "float" },
      "tags":        { "type": "keyword" },
      "cuisines":    { "type": "keyword" },
      "location":    { "type": "geo_point" },
      "isAvailable": { "type": "boolean" }
    }
  }
}
```

### Users Index (`chefooz_users`) — Planned

```json
{
  "mappings": {
    "properties": {
      "userId":    { "type": "keyword" },
      "username": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword" } }
      },
      "fullName":  { "type": "text" },
      "bio":       { "type": "text" },
      "reputation": {
        "properties": {
          "tier":  { "type": "keyword" },
          "score": { "type": "integer" }
        }
      },
      "stats": {
        "properties": {
          "followers":   { "type": "integer" },
          "totalOrders": { "type": "integer" },
          "avgRating":   { "type": "float" }
        }
      },
      "location":   { "type": "geo_point" },
      "isVerified": { "type": "boolean" }
    }
  }
}
```

---

## Query Construction

### Multi-Match Query (Text Search)

Used for all text-based dish/reel searches:

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

**Boost factors**:
- `caption^3` — Caption matches are 3× more important
- `hashtags^2` — Hashtag matches are 2× more important

**Fuzziness AUTO** allows:
- 0 edits for queries ≤ 2 chars
- 1 edit for queries 3–5 chars
- 2 edits for queries ≥ 6 chars

→ "butr chicken" matches "butter chicken" ✅

### Boolean Query (Full Query Structure)

```typescript
const buildDishQuery = (dto: SearchDishesDto) => ({
  bool: {
    must: [
      {
        multi_match: {
          query: dto.query,
          fields: ['caption^3', 'hashtags^2'],
          fuzziness: 'AUTO',
          operator: 'or',
        },
      },
    ],
    filter: buildFilterClauses(dto),    // Exact filters (no score impact, cached)
    should: buildBoostClauses(dto),     // Optional boosts (score enhancement)
  },
});
```

### Filter Clauses (Exact Match, Cached)

```typescript
const buildFilterClauses = (dto: SearchDishesDto): any[] => {
  const filters: any[] = [
    { term: { contentType: 'REEL' } },
  ];

  if (dto.hashtags?.length > 0) {
    filters.push({ terms: { hashtags: dto.hashtags } });
  }

  if (dto.chefIds?.length > 0) {
    filters.push({ terms: { userId: dto.chefIds } });
  }

  if (dto.promotedOnly) {
    filters.push({ term: { promoted: true } });
  }

  if (dto.reelPurpose) {
    filters.push({ term: { reelPurpose: dto.reelPurpose } });
  }

  if (dto.lat && dto.lng && dto.radiusKm) {
    filters.push({
      geo_distance: {
        distance: `${dto.radiusKm}km`,
        location: { lat: dto.lat, lon: dto.lng },
      },
    });
  }

  return filters;
};
```

> **Why filter (not must)?** Filter clauses don't affect relevance score and are aggressively cached by Elasticsearch. This makes repeated searches with the same filters much faster.

### Boost Clauses (Optional, Score Enhancement)

```typescript
const buildBoostClauses = (): any[] => [
  { term: { promoted: { value: true, boost: 2.0 } } },    // Promoted: 2× score
  { range: { createdAt: { gte: 'now-7d/d', boost: 1.2 } } }, // Recent (7d): 1.2× score
];
```

---

## Ranking Algorithm

### Elasticsearch Score (Relevance Mode)

When `sortBy = 'relevance'` (default), ES uses **TF/IDF-based scoring** plus boost modifiers:

```
final_score = base_tf_idf_score
            × promoted_boost (×2.0 if promoted)
            × recency_boost  (×1.2 if created within 7 days)
            + engagement_signal (likes count as tiebreaker)
```

**Sort configuration**:
```typescript
// sortBy = 'relevance'
return ['_score', { 'stats.likes': 'desc' }];
// Primary: ES relevance score
// Tiebreaker: Total likes (higher engagement wins)
```

### Popularity Mode

```typescript
// sortBy = 'popular'
return [
  { 'stats.views': 'desc' },
  { 'stats.likes': 'desc' }
];
```

Views are the primary signal; likes are tiebreaker.

### Trending Mode

```typescript
// sortBy = 'trending'
return [
  { 'stats.likes': 'desc' },
  { 'stats.saves': 'desc' },
  { createdAt: 'desc' }
];
```

Trending combines recent engagement signals: likes + saves + recency.

### Recent Mode

```typescript
// sortBy = 'recent'
return [{ createdAt: 'desc' }];
```

Strict chronological order.

### Ranking Summary Table

| Mode | Primary Signal | Secondary | Tertiary | Best For |
|------|---------------|-----------|----------|---------|
| `relevance` | ES TF/IDF score | Promoted boost | Likes | Keyword searches |
| `popular` | Total views | Likes | — | Discover top content |
| `trending` | Recent likes | Recent saves | Recency | Viral / trending |
| `recent` | Created date | — | — | Latest uploads |

---

## Sort Strategies

```typescript
private buildSortConfig(sortBy: SearchSortBy): any[] {
  switch (sortBy) {
    case 'recent':
      return [{ createdAt: 'desc' }];

    case 'popular':
      return [{ 'stats.views': 'desc' }, { 'stats.likes': 'desc' }];

    case 'trending':
      return [
        { 'stats.likes': 'desc' },
        { 'stats.saves': 'desc' },
        { createdAt: 'desc' },
      ];

    default: // 'relevance'
      return ['_score', { 'stats.likes': 'desc' }];
  }
}
```

---

## Geospatial Search

### Elasticsearch Geospatial Query

```json
{
  "geo_distance": {
    "distance": "5km",
    "location": {
      "lat": 19.0760,
      "lon": 72.8777
    }
  }
}
```

This is added to the **filter** clause (not must) for performance — geo filters are cached.

### MongoDB Geospatial Fallback

```typescript
if (dto.lat && dto.lng && dto.radiusKm) {
  query.location = {
    $near: {
      $geometry: {
        type: 'Point',
        coordinates: [dto.lng, dto.lat], // MongoDB: [lng, lat] order
      },
      $maxDistance: dto.radiusKm * 1000, // Convert km → meters
    },
  };
}
```

**Note**: MongoDB requires a `2dsphere` index on the `location` field for `$near` to work.

### Location Field Format

**Elasticsearch**:
```json
{ "lat": 19.0760, "lon": 72.8777 }
```

**MongoDB** (GeoJSON):
```json
{
  "type": "Point",
  "coordinates": [72.8777, 19.0760]  // [longitude, latitude]
}
```

**Important**: The order is reversed between ES (`lat, lon`) and MongoDB (`lon, lat`). The transform function handles this:

```typescript
private transformForIndex(reel: any) {
  return {
    location: reel.location ? {
      lat: reel.location.coordinates[1], // MongoDB [1] = lat
      lon: reel.location.coordinates[0], // MongoDB [0] = lng
    } : undefined,
    // ...
  };
}
```

---

## Fallback Strategy

### Health Check

```typescript
isElasticsearchEnabled(): boolean {
  try {
    await this.client.ping({ requestTimeout: 1000 });
    return true;
  } catch (error) {
    this.logger.warn('Elasticsearch unavailable:', error.message);
    return false;
  }
}
```

### Automatic Fallback

```typescript
async searchDishes(dto: SearchDishesDto, viewerId?: string): Promise<SearchResultDto> {
  try {
    if (this.elasticsearchService.isElasticsearchEnabled()) {
      return await this.searchDishesWithElasticsearch(dto, viewerId);
    }
  } catch (error) {
    this.logger.error('Elasticsearch search failed:', error);
    // Continue to fallback — never throws to caller
  }

  this.logger.warn('Using MongoDB fallback for dish search');
  return await this.searchDishesWithMongo(dto, viewerId);
}
```

### Capability Comparison

| Capability | Elasticsearch | MongoDB Fallback | PostgreSQL Fallback |
|-----------|--------------|-----------------|-------------------|
| **Fuzzy Matching** | ✅ AUTO (1-2 chars) | ❌ Exact only | ❌ Exact only |
| **Relevance Score** | ✅ TF/IDF | ❌ No scoring | ❌ No scoring |
| **Highlights** | ✅ `<em>` tags | ❌ None | ❌ None |
| **Geospatial** | ✅ `geo_distance` | ✅ `$near` (2dsphere) | ❌ Not supported |
| **Typo Tolerance** | ✅ 1-2 char edits | ❌ None | ❌ None |
| **Performance** | ✅ Inverted index | ⚠️ Regex scans | ⚠️ Sequential scan |
| **Promoted Boost** | ✅ `should` boost | ❌ None | ❌ None |

---

## Index Sync Jobs

### Incremental Sync (Every 15 Minutes)

```typescript
@Injectable()
export class IndexSyncJob {
  @Cron('0 */15 * * * *')
  async syncRecentReels(): Promise<void> {
    const cutoff = new Date(Date.now() - 15 * 60 * 1000); // 15 min ago

    const reels = await this.reelModel.find({
      updatedAt: { $gte: cutoff },
    }).lean();

    if (reels.length === 0) return;

    const bulkBody = reels.flatMap(reel => [
      { index: { _index: 'chefooz_reels', _id: reel._id.toString() } },
      this.transformReelForIndex(reel),
    ]);

    await this.elasticsearchClient.bulk({ body: bulkBody });
    this.logger.log(`Synced ${reels.length} reels to Elasticsearch`);
  }
}
```

**Sync covers**:
- New reels published
- Reels with updated stats (likes, views, comments)
- Reels with moderation status changes (removed from public index)

### Manual Bulk Reindex

Used for disaster recovery or after index schema changes:

```typescript
async bulkIndexReels(): Promise<void> {
  // 1. Delete and recreate index with correct mappings
  await this.client.indices.delete({ index: 'chefooz_reels', ignore_unavailable: true });
  await this.client.indices.create({ index: 'chefooz_reels', body: this.reelsIndexMapping });

  // 2. Batch process all reels (1000 per batch)
  const batchSize = 1000;
  let skip = 0;

  while (true) {
    const reels = await this.reelModel.find().skip(skip).limit(batchSize).lean();
    if (reels.length === 0) break;

    const bulkBody = reels.flatMap(reel => [
      { index: { _index: 'chefooz_reels', _id: reel._id.toString() } },
      this.transformReelForIndex(reel),
    ]);

    await this.client.bulk({ body: bulkBody });
    skip += batchSize;
    this.logger.log(`Bulk indexed ${skip} reels`);
  }

  this.logger.log('Bulk reindex complete');
}
```

### Transform for Indexing

```typescript
private transformReelForIndex(reel: any) {
  return {
    reelId: reel._id.toString(),
    userId: reel.userId,
    caption: reel.caption || '',
    hashtags: reel.hashtags || [],
    stats: {
      views:    reel.stats?.views ?? 0,
      likes:    reel.stats?.likes ?? 0,
      comments: reel.stats?.comments ?? 0,
      saves:    reel.stats?.saves ?? 0,
      orders:   reel.stats?.orders ?? 0,
    },
    location: reel.location ? {
      lat: reel.location.coordinates[1],
      lon: reel.location.coordinates[0],
    } : undefined,
    createdAt: reel.createdAt,
    promoted: reel.promoted ?? false,
    reelPurpose: reel.reelPurpose,
    contentType: 'REEL',
  };
}
```

---

## Performance Optimization

### Use Filter Context (Cached)

Filter clauses don't affect scoring and are cached by Elasticsearch between queries:

```json
{
  "bool": {
    "filter": [
      { "term": { "contentType": "REEL" } },
      { "range": { "createdAt": { "gte": "now-7d/d" } } }
    ]
  }
}
```

✅ Repeated searches with the same filters are fast — results come from ES query cache.

### Limit Fields Returned (`_source`)

```json
{
  "_source": ["reelId", "caption", "hashtags", "stats", "createdAt", "userId", "promoted"]
}
```

Only fetch the fields needed for the response. This reduces network transfer and deserialization overhead.

### Pagination via `from` + `size`

```typescript
const esQuery = {
  from: (dto.page - 1) * dto.limit,  // Offset
  size: dto.limit,                    // Page size (max 50)
  // ...
};
```

**Important**: Elasticsearch deep pagination (`from > 10,000`) becomes slow. Use `search_after` with a cursor for pages beyond 10,000 results (not a concern for current scale).

### Index Shard Configuration

- **3 primary shards**: Parallel search across shards
- **1 replica shard**: High availability (reads from replicas too)

```json
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1
  }
}
```

### Highlights

```json
{
  "highlight": {
    "fields": {
      "caption": {
        "pre_tags": ["<em>"],
        "post_tags": ["</em>"],
        "number_of_fragments": 1
      },
      "hashtags.text": {}
    }
  }
}
```

Returns only 1 fragment per field (`number_of_fragments: 1`) to keep response size small.

---

## Frontend Integration

### React Query Hook Pattern

```typescript
// libs/api-client/src/search.hooks.ts

export const useSearchDishes = (params: SearchDishesDto) => {
  return useQuery({
    queryKey: ['search', 'dishes', params],
    queryFn: () => searchClient.searchDishes(params),
    enabled: params.query.length >= 3,  // Min 3 chars
    staleTime: 1000 * 60,               // Cache results for 1 minute
  });
};

export const useSearchSuggest = (query: string) => {
  return useQuery({
    queryKey: ['search', 'suggest', query],
    queryFn: () => searchClient.getSuggestions(query),
    enabled: query.length >= 3,
    staleTime: 1000 * 30,              // Suggestions: 30s cache
  });
};
```

### Search Screen Integration Pattern

```typescript
// apps/chefooz-app/src/app/(tabs)/search.tsx

const SearchScreen = () => {
  const [query, setQuery] = useState('');
  const [debouncedQuery] = useDebounce(query, 300); // 300ms debounce
  const [sortBy, setSortBy] = useState<SearchSortBy>('relevance');
  const { latitude, longitude } = useUserLocation();

  const { data, isLoading, fetchNextPage } = useInfiniteSearchDishes({
    query: debouncedQuery,
    sortBy,
    lat: latitude,
    lng: longitude,
    radiusKm: 10, // 10km radius default
  });
};
```

**Debounce**: 300ms debounce prevents an API call on every keystroke.  
**Infinite scroll**: Use `useInfiniteQuery` for paginated results with "Load more".

---

## Testing Checklist

### Unit Tests

- [ ] `buildDishQuery()` — correct must/filter/should clauses generated
- [ ] `buildSortConfig('relevance')` — returns `['_score', { 'stats.likes': 'desc' }]`
- [ ] `buildSortConfig('trending')` — returns likes + saves + recency sort
- [ ] `buildFilterClauses()` — geo_distance added when lat/lng/radiusKm provided
- [ ] `transformReelForIndex()` — MongoDB `[lng, lat]` → ES `{lat, lon}` conversion
- [ ] `isElasticsearchEnabled()` — returns false when ping fails

### Integration Tests

```powershell
# Search dishes
Invoke-WebRequest -Method GET `
  -Uri "http://localhost:3000/api/v1/search/dishes?query=butter+chicken&lat=19.07&lng=72.87&radiusKm=5&sortBy=relevance" `
  -Headers @{ Authorization = "Bearer $jwt" }

# Search menu items
Invoke-WebRequest -Method GET `
  -Uri "http://localhost:3000/api/v1/search/menu-items?query=biryani&vegOnly=true&maxPrice=300" `
  -Headers @{ Authorization = "Bearer $jwt" }

# Get suggestions
Invoke-WebRequest -Method GET `
  -Uri "http://localhost:3000/api/v1/search/suggest?query=but" `
  -Headers @{ Authorization = "Bearer $jwt" }

# Search chefs
Invoke-WebRequest -Method GET `
  -Uri "http://localhost:3000/api/v1/search/chefs?query=rakesh&isVerified=true&minRating=4" `
  -Headers @{ Authorization = "Bearer $jwt" }
```

### Scenario Tests

- [ ] Fuzzy search: "butr chikin" matches "butter chicken"
- [ ] Hashtag filter: `?hashtags[]=biryani` returns only biryani reels
- [ ] Geo filter: Results within 5km of provided lat/lng
- [ ] Promoted boost: Promoted reels appear higher in relevance sort
- [ ] Trending: Recently liked/saved content ranks higher
- [ ] Elasticsearch down → MongoDB fallback returns results
- [ ] `source: 'mongodb'` in response when ES unavailable
- [ ] Pagination: `page=2&limit=20` returns correct offset
- [ ] `promotedOnly=true` returns only promoted content
- [ ] Suggestion: minimum 3 chars enforced (2 chars returns empty)

### Performance Tests

- [ ] Search response time < 200ms with Elasticsearch
- [ ] Search response time < 1s with MongoDB fallback
- [ ] Index sync completes within 15-minute window (for typical volumes)
- [ ] Bulk reindex: ~1000 reels/second throughput

---

## Roadmap

| Feature | Status | Notes |
|---------|--------|-------|
| Reels search (Elasticsearch) | ✅ Production | Fully operational |
| Menu items search (PostgreSQL) | ✅ Production | No ES yet |
| Chefs search (PostgreSQL) | ✅ Production | No ES yet |
| Autocomplete suggestions | ✅ Production | — |
| Menu items Elasticsearch index | 🔄 Planned | Week 11+ |
| Users Elasticsearch index | 🔄 Planned | Week 11+ |
| Personalized search ranking | 🔄 Planned | Based on user history |
| Search analytics dashboard | 🔄 Planned | Top queries, zero-result queries |

---

[SLICE_COMPLETE ✅]
