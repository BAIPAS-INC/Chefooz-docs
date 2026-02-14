# Search Module - Technical Guide

**Version:** 1.0  
**Last Updated:** February 14, 2026  
**Module:** `apps/chefooz-apis/src/modules/search-elastic/`  
**Legacy Module:** `apps/chefooz-apis/src/modules/search/`  
**Tech Stack:** Elasticsearch 8+, NestJS, MongoDB, PostgreSQL

---

## 📋 Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [API Endpoints](#api-endpoints)
3. [Service Methods](#service-methods)
4. [Elasticsearch Configuration](#elasticsearch-configuration)
5. [Index Mappings](#index-mappings)
6. [Query Construction](#query-construction)
7. [Fallback Strategy](#fallback-strategy)
8. [Indexing Jobs](#indexing-jobs)
9. [Performance Optimization](#performance-optimization)
10. [Error Handling](#error-handling)

---

## Architecture Overview

### Module Structure

```
apps/chefooz-apis/src/modules/search-elastic/
├── dto/
│   └── search-elastic.dto.ts          # Query DTOs (dishes, menu items, chefs, suggest)
├── elasticsearch.service.ts           # Low-level ES client wrapper
├── search-elastic.service.ts          # High-level search logic
├── search-elastic.controller.ts       # HTTP endpoints
├── search-elastic.module.ts           # Module configuration
└── jobs/
    └── index-sync.job.ts              # Background indexing job

apps/chefooz-apis/src/modules/search/ (Legacy)
├── dto/
│   └── search-users.dto.ts            # Simple user search DTO
├── search.service.ts                  # PostgreSQL-based search
├── search.controller.ts               # Legacy endpoints
└── search.module.ts                   # Module configuration
```

### Technology Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **Search Engine** | Elasticsearch 8+ | Full-text search, fuzzy matching, geospatial |
| **Fallback** | MongoDB + PostgreSQL | When Elasticsearch unavailable |
| **Backend** | NestJS 10+ | REST API framework |
| **Validation** | class-validator | DTO input validation |
| **Logging** | Winston | Structured logging |
| **Cron Jobs** | @nestjs/schedule | Index synchronization |

### Dependencies

```typescript
@Module({
  imports: [
    MongooseModule.forFeature([
      { name: Reel.name, schema: ReelSchema },
    ]),
    TypeOrmModule.forFeature([
      User,
      ChefProfile,
      ChefMenuItem,
    ]),
  ],
  providers: [
    ElasticsearchService,
    SearchElasticService,
  ],
  controllers: [SearchElasticController],
  exports: [SearchElasticService],
})
```

---

## API Endpoints

### 1. Search Dishes/Reels

**Endpoint**: `GET /api/v1/search/dishes`

**Auth**: Required (JWT)

**Rate Limit**: 10 requests/second

**Query Parameters**:
```typescript
{
  query: string;              // Search query (required)
  page?: number;              // Page number (default: 1)
  limit?: number;             // Results per page (default: 20, max: 50)
  sortBy?: SearchSortBy;      // 'relevance' | 'recent' | 'popular' | 'trending'
  hashtags?: string[];        // Filter by hashtags
  chefIds?: string[];         // Filter by specific chefs
  promotedOnly?: boolean;     // Only promoted content
  reelPurpose?: string;       // 'MENU_SHOWCASE' | 'STORY' | etc.
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
        "hashtags": ["butterchicken", "cooking"],
        "videoUrl": "https://...",
        "thumbnailUrl": "https://...",
        "durationSec": 30,
        "stats": { "views": 1234, "likes": 89, "comments": 12, "saves": 45 },
        "author": {
          "userId": "uuid",
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

### 2. Search Menu Items

**Endpoint**: `GET /api/v1/search/menu-items`

**Auth**: Required (JWT)

**Rate Limit**: 10 requests/second

**Query Parameters**:
```typescript
{
  query: string;              // Search query (required)
  page?: number;              // Page number (default: 1)
  limit?: number;             // Results per page (default: 20, max: 50)
  chefId?: string;            // Filter by specific chef
  minPrice?: number;          // Minimum price filter
  maxPrice?: number;          // Maximum price filter
  minRating?: number;         // Minimum rating (0-5)
  cuisines?: string[];        // Filter by cuisine types
  isVegetarian?: boolean;     // Vegetarian only
  isVegan?: boolean;          // Vegan only
  isGlutenFree?: boolean;     // Gluten-free only
  vegOnly?: boolean;          // Vegetarian filter (legacy)
  availableOnly?: boolean;    // Only available items
  lat?: number;               // Latitude
  lng?: number;               // Longitude
  radiusKm?: number;          // Radius in kilometers
}
```

**Response**:
```json
{
  "success": true,
  "message": "Menu items retrieved successfully",
  "data": {
    "items": [
      {
        "id": "menu-item-uuid",
        "name": "Butter Chicken",
        "description": "Creamy tomato-based curry",
        "price": 350,
        "currency": "INR",
        "prepTimeMinutes": 30,
        "rating": 4.5,
        "reviewCount": 120,
        "chef": {
          "userId": "uuid",
          "username": "chef_rakesh",
          "fullName": "Rakesh Kumar",
          "distance": 2.5
        },
        "images": ["https://..."],
        "tags": ["butterchicken", "northindian"],
        "isAvailable": true
      }
    ],
    "total": 45,
    "page": 1,
    "limit": 20,
    "hasMore": true
  }
}
```

### 3. Search Chefs

**Endpoint**: `GET /api/v1/search/chefs`

**Auth**: Required (JWT)

**Rate Limit**: 10 requests/second

**Query Parameters**:
```typescript
{
  query: string;              // Search query (required)
  page?: number;              // Page number (default: 1)
  limit?: number;             // Results per page (default: 20, max: 50)
  minRating?: number;         // Minimum rating (0-5)
  isVerified?: boolean;       // Verified chefs only
  lat?: number;               // Latitude
  lng?: number;               // Longitude
  radiusKm?: number;          // Radius in kilometers
}
```

**Response**:
```json
{
  "success": true,
  "message": "Chefs retrieved successfully",
  "data": {
    "items": [
      {
        "userId": "uuid",
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

### 4. Get Suggestions

**Endpoint**: `GET /api/v1/search/suggest`

**Auth**: Required (JWT)

**Rate Limit**: 20 requests/second

**Query Parameters**:
```typescript
{
  query: string;              // Prefix query (min 3 characters)
  limit?: number;             // Max suggestions (default: 10)
}
```

**Response**:
```json
{
  "success": true,
  "message": "Suggestions retrieved successfully",
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

### 5. Trigger Reindex (Admin Only)

**Endpoint**: `POST /api/v1/search/reindex`

**Auth**: Required (JWT + Admin Role)

**Rate Limit**: No limit (admin only)

**Response**:
```json
{
  "success": true,
  "message": "Re-indexing started in background"
}
```

---

## Service Methods

### SearchElasticService

#### searchDishes()

```typescript
async searchDishes(dto: SearchDishesDto, viewerId?: string): Promise<SearchResultDto>
```

**Flow**:
1. Check if Elasticsearch is enabled
2. If enabled → `searchDishesWithElasticsearch()`
3. If disabled → `searchDishesWithMongo()` (fallback)
4. Enrich results with user data
5. Return paginated results

**Implementation**:
```typescript
async searchDishes(dto: SearchDishesDto, viewerId?: string) {
  const isESEnabled = this.elasticsearchService.isElasticsearchEnabled();
  
  if (isESEnabled) {
    return this.searchDishesWithElasticsearch(dto, viewerId);
  } else {
    this.logger.warn('Elasticsearch unavailable, using MongoDB fallback');
    return this.searchDishesWithMongo(dto, viewerId);
  }
}
```

#### searchDishesWithElasticsearch()

```typescript
private async searchDishesWithElasticsearch(
  dto: SearchDishesDto, 
  viewerId?: string
): Promise<SearchResultDto>
```

**Elasticsearch Query Construction**:

```typescript
const mustClauses = [
  {
    multi_match: {
      query: dto.query,
      fields: ['caption^3', 'hashtags^2'],
      fuzziness: 'AUTO',
      operator: 'or',
    },
  },
];

const filterClauses = [
  { term: { contentType: 'REEL' } },
];

if (dto.hashtags?.length > 0) {
  filterClauses.push({ terms: { hashtags: dto.hashtags } });
}

if (dto.lat && dto.lng && dto.radiusKm) {
  filterClauses.push({
    geo_distance: {
      distance: `${dto.radiusKm}km`,
      location: { lat: dto.lat, lon: dto.lng },
    },
  });
}

const response = await client.search({
  index: 'chefooz_reels',
  body: {
    from: (dto.page - 1) * dto.limit,
    size: dto.limit,
    query: {
      bool: {
        must: mustClauses,
        filter: filterClauses,
        should: [
          { term: { promoted: { value: true, boost: 2.0 } } },
          { range: { createdAt: { gte: 'now-7d/d', boost: 1.2 } } },
        ],
      },
    },
    sort: this.buildSortConfig(dto.sortBy),
    highlight: {
      fields: {
        caption: {},
        'hashtags.text': {},
      },
    },
  },
});
```

**Sort Configuration**:
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
    default: // relevance
      return ['_score', { 'stats.likes': 'desc' }];
  }
}
```

#### searchDishesWithMongo() (Fallback)

```typescript
private async searchDishesWithMongo(
  dto: SearchDishesDto, 
  viewerId?: string
): Promise<SearchResultDto>
```

**MongoDB Query**:
```typescript
const query: any = {};

if (dto.query) {
  query.$or = [
    { caption: { $regex: dto.query, $options: 'i' } },
    { hashtags: { $regex: dto.query, $options: 'i' } },
  ];
}

if (dto.hashtags?.length > 0) {
  query.hashtags = { $in: dto.hashtags };
}

if (dto.lat && dto.lng && dto.radiusKm) {
  query.location = {
    $near: {
      $geometry: {
        type: 'Point',
        coordinates: [dto.lng, dto.lat],
      },
      $maxDistance: dto.radiusKm * 1000, // km to meters
    },
  };
}

const reels = await this.reelModel
  .find(query)
  .sort(this.buildMongoSort(dto.sortBy))
  .skip((dto.page - 1) * dto.limit)
  .limit(dto.limit)
  .lean();
```

**Limitations**:
- ❌ No fuzzy matching (typos)
- ❌ No relevance scoring (TF/IDF)
- ❌ No highlights
- ❌ Slower performance
- ✅ Still functional (graceful degradation)

#### searchMenuItems()

```typescript
async searchMenuItems(dto: SearchMenuItemsDto): Promise<SearchResultDto>
```

**PostgreSQL Query** (No Elasticsearch for menu items yet):
```typescript
const queryBuilder = this.menuItemRepo
  .createQueryBuilder('item')
  .where('item.isActive = :isActive', { isActive: true });

if (dto.query) {
  queryBuilder.andWhere(
    '(item.name ILIKE :query OR item.description ILIKE :query)',
    { query: `%${dto.query}%` }
  );
}

if (dto.minPrice && dto.maxPrice) {
  queryBuilder.andWhere('item.price BETWEEN :min AND :max', {
    min: dto.minPrice,
    max: dto.maxPrice,
  });
}

if (dto.vegOnly) {
  queryBuilder.andWhere("item.tags @> ARRAY['veg']::text[]");
}

if (dto.availableOnly) {
  queryBuilder.andWhere(
    "(item.availability->>'isAvailable')::boolean = true"
  );
}

const [items, total] = await queryBuilder
  .orderBy('item.name', 'ASC')
  .skip((dto.page - 1) * dto.limit)
  .take(dto.limit)
  .getManyAndCount();
```

---

## Elasticsearch Configuration

### ElasticsearchService

**Purpose**: Low-level wrapper around Elasticsearch client

```typescript
@Injectable()
export class ElasticsearchService {
  private client: Client;
  
  readonly REELS_INDEX = 'chefooz_reels';
  readonly MENU_ITEMS_INDEX = 'chefooz_menu_items';
  readonly USERS_INDEX = 'chefooz_users';
  
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
      await this.client.ping();
      return true;
    } catch (error) {
      this.logger.warn('Elasticsearch unavailable');
      return false;
    }
  }
  
  getClient(): Client {
    return this.client;
  }
}
```

**Environment Variables**:
```bash
ELASTICSEARCH_NODE=https://localhost:9200
ELASTICSEARCH_USERNAME=elastic
ELASTICSEARCH_PASSWORD=changeme
ELASTICSEARCH_INDEX_PREFIX=chefooz_
```

---

## Index Mappings

### Reels Index

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
  },
  "mappings": {
    "properties": {
      "reelId": {
        "type": "keyword"
      },
      "userId": {
        "type": "keyword"
      },
      "caption": {
        "type": "text",
        "fields": {
          "keyword": {
            "type": "keyword",
            "ignore_above": 256
          }
        },
        "analyzer": "custom_text_analyzer"
      },
      "hashtags": {
        "type": "keyword",
        "fields": {
          "text": {
            "type": "text",
            "analyzer": "custom_text_analyzer"
          }
        }
      },
      "stats": {
        "properties": {
          "views": { "type": "integer" },
          "likes": { "type": "integer" },
          "comments": { "type": "integer" },
          "saves": { "type": "integer" },
          "orders": { "type": "integer" }
        }
      },
      "location": {
        "type": "geo_point"
      },
      "createdAt": {
        "type": "date"
      },
      "promoted": {
        "type": "boolean"
      },
      "reelPurpose": {
        "type": "keyword"
      },
      "contentType": {
        "type": "keyword"
      }
    }
  }
}
```

### Menu Items Index (Future)

```json
{
  "mappings": {
    "properties": {
      "menuItemId": { "type": "keyword" },
      "chefId": { "type": "keyword" },
      "name": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword" } }
      },
      "description": { "type": "text" },
      "price": { "type": "float" },
      "rating": { "type": "float" },
      "tags": { "type": "keyword" },
      "location": { "type": "geo_point" },
      "isAvailable": { "type": "boolean" }
    }
  }
}
```

### Users Index (Future)

```json
{
  "mappings": {
    "properties": {
      "userId": { "type": "keyword" },
      "username": {
        "type": "text",
        "fields": { "keyword": { "type": "keyword" } }
      },
      "fullName": { "type": "text" },
      "bio": { "type": "text" },
      "reputation": {
        "properties": {
          "tier": { "type": "keyword" },
          "score": { "type": "integer" }
        }
      },
      "stats": {
        "properties": {
          "followers": { "type": "integer" },
          "totalOrders": { "type": "integer" },
          "avgRating": { "type": "float" }
        }
      },
      "location": { "type": "geo_point" },
      "isVerified": { "type": "boolean" }
    }
  }
}
```

---

## Query Construction

### Multi-Match Query

**Use Case**: Search across multiple fields with different boosts

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

**Parameters**:
- `query`: User's search term
- `fields`: Fields to search with boost factors (caption 3×, hashtags 2×)
- `fuzziness`: AUTO (allows 1-2 character typos)
- `operator`: OR (matches any term)

### Boolean Query

**Use Case**: Combine must/should/filter clauses

```json
{
  "bool": {
    "must": [
      {
        "multi_match": {
          "query": "butter chicken",
          "fields": ["caption^3", "hashtags^2"]
        }
      }
    ],
    "filter": [
      { "term": { "contentType": "REEL" } },
      { "terms": { "hashtags": ["butterchicken", "northindian"] } }
    ],
    "should": [
      { "term": { "promoted": { "value": true, "boost": 2.0 } } },
      { "range": { "createdAt": { "gte": "now-7d/d", "boost": 1.2 } } }
    ]
  }
}
```

**Clauses**:
- `must`: Required matches (affects score)
- `filter`: Required matches (no scoring, cached)
- `should`: Optional matches (boost score if present)

### Geospatial Query

**Use Case**: Find content within radius

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

### Highlight Configuration

**Use Case**: Show matching text snippets

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

**Result**:
```json
{
  "highlights": {
    "caption": ["Making the perfect <em>butter chicken</em>!"]
  }
}
```

---

## Fallback Strategy

### Elasticsearch Health Check

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
async searchDishes(dto: SearchDishesDto, viewerId?: string) {
  try {
    if (this.elasticsearchService.isElasticsearchEnabled()) {
      return await this.searchDishesWithElasticsearch(dto, viewerId);
    }
  } catch (error) {
    this.logger.error('Elasticsearch search failed:', error);
  }
  
  // Fallback to MongoDB
  this.logger.warn('Using MongoDB fallback');
  return await this.searchDishesWithMongo(dto, viewerId);
}
```

### Fallback Limitations

| Feature | Elasticsearch | MongoDB Fallback |
|---------|--------------|------------------|
| **Fuzzy Matching** | ✅ Yes (AUTO) | ❌ No |
| **Relevance Scoring** | ✅ TF/IDF | ❌ No scoring |
| **Highlights** | ✅ Yes | ❌ No |
| **Geospatial** | ✅ Optimized | ✅ Supported |
| **Performance** | ✅ Fast (indexed) | ⚠️ Slower (regex) |
| **Typo Tolerance** | ✅ 1-2 chars | ❌ Exact match only |

---

## Indexing Jobs

### Background Index Sync

**Schedule**: Every 15 minutes

```typescript
@Injectable()
export class IndexSyncJob {
  @Cron('0 */15 * * * *')
  async syncRecentReels() {
    const cutoff = new Date(Date.now() - 15 * 60 * 1000);
    
    // Fetch recently updated reels
    const reels = await this.reelModel.find({
      updatedAt: { $gte: cutoff },
    }).lean();
    
    // Bulk index
    const bulkBody = reels.flatMap(reel => [
      { index: { _index: 'chefooz_reels', _id: reel._id.toString() } },
      this.transformReelForIndex(reel),
    ]);
    
    await this.elasticsearchClient.bulk({ body: bulkBody });
    this.logger.log(`Indexed ${reels.length} reels`);
  }
}
```

### Transform for Indexing

```typescript
private transformReelForIndex(reel: any) {
  return {
    reelId: reel._id.toString(),
    userId: reel.userId,
    caption: reel.caption,
    hashtags: reel.hashtags || [],
    stats: reel.stats || {},
    location: reel.location ? {
      lat: reel.location.coordinates[1],
      lon: reel.location.coordinates[0],
    } : undefined,
    createdAt: reel.createdAt,
    promoted: reel.promoted || false,
    reelPurpose: reel.reelPurpose,
    contentType: 'REEL',
  };
}
```

### Manual Bulk Reindex

```typescript
async bulkIndexReels() {
  // Delete old index
  await this.client.indices.delete({
    index: 'chefooz_reels',
    ignore_unavailable: true,
  });
  
  // Create new index with mappings
  await this.client.indices.create({
    index: 'chefooz_reels',
    body: this.reelsIndexMapping,
  });
  
  // Fetch all reels (in batches)
  const batchSize = 1000;
  let skip = 0;
  
  while (true) {
    const reels = await this.reelModel
      .find()
      .skip(skip)
      .limit(batchSize)
      .lean();
    
    if (reels.length === 0) break;
    
    const bulkBody = reels.flatMap(reel => [
      { index: { _index: 'chefooz_reels', _id: reel._id.toString() } },
      this.transformReelForIndex(reel),
    ]);
    
    await this.client.bulk({ body: bulkBody });
    skip += batchSize;
    
    this.logger.log(`Indexed ${skip} reels`);
  }
  
  this.logger.log('Bulk indexing complete');
}
```

---

## Performance Optimization

### Query Optimization

**Use Filter Context (Cached)**:
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

Filter clauses are cached and don't affect scoring (faster).

**Limit Fields Returned**:
```json
{
  "_source": ["reelId", "caption", "hashtags", "stats", "createdAt"]
}
```

Only fetch needed fields to reduce payload size.

**Use Pagination (from + size)**:
```json
{
  "from": 0,
  "size": 20
}
```

Avoid deep pagination (from > 10000) - use search_after instead.

### Index Optimization

**Shard Strategy**:
- Small indices (< 1M docs): 1-3 shards
- Medium indices (1M-10M docs): 3-5 shards
- Large indices (> 10M docs): 5-10 shards

**Replica Strategy**:
- Production: 1-2 replicas (high availability)
- Development: 0 replicas (faster indexing)

**Refresh Interval**:
```json
{
  "settings": {
    "refresh_interval": "30s"
  }
}
```

Default is 1s (near real-time). Increase to 30s for bulk indexing.

---

## Error Handling

### Standard Error Responses

```typescript
// 400 Bad Request
{
  "success": false,
  "message": "Search query must be at least 3 characters",
  "errorCode": "INVALID_QUERY"
}

// 500 Elasticsearch Error
{
  "success": false,
  "message": "Search failed, please try again",
  "errorCode": "SEARCH_FAILED"
}

// 503 Elasticsearch Unavailable (Fallback Active)
{
  "success": true,
  "message": "Search results (fallback mode)",
  "data": { ... },
  "warning": "Limited search features available"
}
```

### Error Logging

```typescript
try {
  const results = await this.searchDishesWithElasticsearch(dto, viewerId);
  return results;
} catch (error) {
  this.logger.error('Elasticsearch search failed:', {
    error: error.message,
    stack: error.stack,
    query: dto.query,
    userId: viewerId,
  });
  
  // Fallback
  return await this.searchDishesWithMongo(dto, viewerId);
}
```

---

## Testing Approach

### Unit Tests

```typescript
describe('SearchElasticService', () => {
  it('should fallback to MongoDB when Elasticsearch unavailable', async () => {
    jest.spyOn(elasticsearchService, 'isElasticsearchEnabled')
      .mockReturnValue(false);
    
    const result = await service.searchDishes({ query: 'butter chicken' });
    
    expect(result.items).toBeDefined();
    // Verify MongoDB was called, not Elasticsearch
  });
  
  it('should apply fuzzy matching in Elasticsearch', async () => {
    const result = await service.searchDishes({ query: 'butr chiken' });
    
    // Should still find "butter chicken" despite typos
    expect(result.items.some(item => 
      item.caption.includes('butter chicken')
    )).toBe(true);
  });
});
```

### Integration Tests

```typescript
describe('SearchElasticController (e2e)', () => {
  it('GET /search/dishes should return results', async () => {
    const response = await request(app.getHttpServer())
      .get('/v1/search/dishes?q=butter chicken')
      .set('Authorization', `Bearer ${userToken}`)
      .expect(200);
    
    expect(response.body.success).toBe(true);
    expect(response.body.data.items).toBeInstanceOf(Array);
    expect(response.body.data.items.length).toBeGreaterThan(0);
  });
  
  it('should apply geospatial filter correctly', async () => {
    const response = await request(app.getHttpServer())
      .get('/v1/search/dishes?q=butter&lat=19.0760&lng=72.8777&radiusKm=5')
      .set('Authorization', `Bearer ${userToken}`)
      .expect(200);
    
    // All results should be within 5km
    response.body.data.items.forEach(item => {
      expect(item.distance).toBeLessThanOrEqual(5);
    });
  });
});
```

---

## Related Documentation

- **Feature Overview**: `FEATURE_OVERVIEW.md` (Business context, search types)
- **QA Test Cases**: `QA_TEST_CASES.md` (Test scenarios)
- **Explore Module**: `../explore/TECHNICAL_GUIDE.md` (Discovery features)

---

**[SLICE_COMPLETE ✅]**  
**Module**: Search  
**Documentation**: Technical Guide  
**Date**: February 14, 2026
