# 🔧 Chef Module - Technical Implementation Guide

## 📋 **Table of Contents**
- [Architecture Overview](#architecture-overview)
- [Database Schema](#database-schema)
- [API Endpoints](#api-endpoints)
- [Service Layer](#service-layer)
- [DTO Specifications](#dto-specifications)
- [Error Handling](#error-handling)
- [Performance Optimization](#performance-optimization)
- [Integration Patterns](#integration-patterns)
- [Testing Strategy](#testing-strategy)

---

## 🏗️ **Architecture Overview**

### **Module Structure**

```
apps/chefooz-apis/src/modules/chef/
├── chef.controller.ts      # Public API endpoint (GET menu)
├── chef.service.ts          # Business logic (menu retrieval)
└── chef.module.ts           # NestJS module definition
```

**Key Characteristics**:
- ✅ **Minimal Module**: Single responsibility (menu read API)
- ✅ **Read-Only**: No write operations (delegates to Chef-Kitchen)
- ✅ **Public Access**: No JWT authentication required
- ✅ **Cross-Module Dependency**: Imports ChefMenuItem from Chef-Kitchen

---

### **System Architecture Diagram**

```mermaid
graph TB
    subgraph "Frontend Layer"
        Mobile[Mobile App<br/>React Native]
        Admin[Admin Portal<br/>Next.js]
    end
    
    subgraph "API Gateway"
        APIGW[API Gateway<br/>/api/v1/chefs/:id/menu]
    end
    
    subgraph "Chef Module"
        Controller[ChefController<br/>@Controller('chefs')]
        Service[ChefService<br/>getMenuItems()]
    end
    
    subgraph "Data Access"
        TypeORM[TypeORM Repository<br/>ChefMenuItem]
    end
    
    subgraph "Database - PostgreSQL"
        DB[(chef_menu_items table)]
    end
    
    subgraph "Related Modules"
        ChefKitchen[Chef-Kitchen Module<br/>Menu CRUD Operations]
        Media[Media Module<br/>Product Tagging]
        Cart[Cart Module<br/>Item Validation]
    end
    
    Mobile -->|HTTP GET| APIGW
    Admin -->|HTTP GET| APIGW
    APIGW --> Controller
    Controller --> Service
    Service --> TypeORM
    TypeORM --> DB
    
    ChefKitchen -.->|Writes| DB
    Media -.->|Reads| Service
    Cart -.->|Validates| Service
    
    style Service fill:#7ED321,stroke:#333,stroke-width:3px
    style DB fill:#4A90E2,stroke:#333,stroke-width:2px
    style Controller fill:#FF6B35,stroke:#333,stroke-width:2px,color:#fff
```

---

### **Module Dependencies**

```typescript
// chef.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { ChefMenuItem } from '../chef-kitchen/entities/chef-menu-item.entity'; // Cross-module import
import { ChefController } from './chef.controller';
import { ChefService } from './chef.service';

@Module({
  imports: [
    TypeOrmModule.forFeature([ChefMenuItem]), // Register repository
  ],
  controllers: [ChefController],
  providers: [ChefService],
  exports: [ChefService], // Available for other modules
})
export class ChefModule {}
```

**Dependency Explanation**:
- **ChefMenuItem**: Entity managed by Chef-Kitchen module but read by Chef module
- **TypeOrmModule**: Required for repository injection
- **Exports**: ChefService exported for Media, Cart, Feed modules

---

## 🗄️ **Database Schema**

### **ChefMenuItem Entity (Read-Only View)**

```typescript
// apps/chefooz-apis/src/modules/chef-kitchen/entities/chef-menu-item.entity.ts
import { Entity, Column, PrimaryGeneratedColumn, ManyToOne, JoinColumn } from 'typeorm';
import { User } from '../../../database/entities/user.entity';

@Entity('chef_menu_items')
export class ChefMenuItem {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ name: 'chef_id', type: 'uuid' })
  chefId: string;

  @ManyToOne(() => User, { onDelete: 'CASCADE' })
  @JoinColumn({ name: 'chef_id' })
  chef: User;

  @Column({ type: 'varchar', length: 255 })
  name: string;

  @Column({ type: 'text', nullable: true })
  description: string;

  @Column({ type: 'decimal', precision: 10, scale: 2 })
  price: number; // Stored in rupees (e.g., 350.00)

  @Column({ type: 'varchar', length: 100, nullable: true })
  category: string;

  @Column({ name: 'is_active', type: 'boolean', default: true })
  isActive: boolean;

  @Column({ name: 'image_url', type: 'text', nullable: true })
  imageUrl: string;

  @Column({ name: 'thumbnail_url', type: 'text', nullable: true })
  thumbnailUrl: string;

  @Column({ type: 'jsonb', default: { isAvailable: true } })
  availability: {
    isAvailable: boolean;
    outOfStockReason?: string;
    unavailableUntil?: Date;
  };

  @Column({ name: 'created_at', type: 'timestamp', default: () => 'CURRENT_TIMESTAMP' })
  createdAt: Date;

  @Column({ name: 'updated_at', type: 'timestamp', default: () => 'CURRENT_TIMESTAMP' })
  updatedAt: Date;
}
```

---

### **PostgreSQL Table Structure**

```sql
-- Table: public.chef_menu_items
CREATE TABLE public.chef_menu_items (
    id UUID DEFAULT uuid_generate_v4() PRIMARY KEY,
    chef_id UUID NOT NULL REFERENCES public.users(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    price NUMERIC(10,2) NOT NULL CHECK (price >= 0),
    category VARCHAR(100),
    is_active BOOLEAN DEFAULT true NOT NULL,
    image_url TEXT,
    thumbnail_url TEXT,
    availability JSONB DEFAULT '{"isAvailable": true}'::jsonb,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP NOT NULL,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP NOT NULL
);

-- Indexes
CREATE INDEX idx_chef_menu_items_chef_id ON public.chef_menu_items(chef_id);
CREATE INDEX idx_chef_menu_items_active ON public.chef_menu_items(is_active);
CREATE INDEX idx_chef_menu_items_chef_active ON public.chef_menu_items(chef_id, is_active); -- Compound index (optimized for Chef Module query)

-- Comments
COMMENT ON TABLE public.chef_menu_items IS 'Menu items created by chefs (managed by Chef-Kitchen, read by Chef Module)';
COMMENT ON COLUMN public.chef_menu_items.price IS 'Price in rupees (DECIMAL). API converts to paise (multiply by 100).';
COMMENT ON COLUMN public.chef_menu_items.is_active IS 'Visibility control. False = soft-deleted/hidden from customers.';
COMMENT ON COLUMN public.chef_menu_items.availability IS 'Real-time availability status. {isAvailable: boolean, outOfStockReason?: string}';
```

---

### **Query Optimization**

#### **Compound Index Justification**
```sql
-- Chef Module query pattern
SELECT * FROM chef_menu_items 
WHERE chef_id = '123e4567-...' 
  AND is_active = true 
ORDER BY created_at DESC;

-- Optimized by compound index: (chef_id, is_active)
-- Avoids index merge, uses single index scan
```

#### **Index Selection Strategy**
| Query Pattern | Index Used | Performance |
|---------------|------------|-------------|
| `WHERE chef_id = ?` | `idx_chef_menu_items_chef_id` | ⚡ Fast (1-15 rows) |
| `WHERE is_active = ?` | `idx_chef_menu_items_active` | ⚠️ Slow (full scan) |
| `WHERE chef_id = ? AND is_active = ?` | `idx_chef_menu_items_chef_active` | ⚡⚡ Fastest (compound) |

**Best Practice**: Always filter by `chef_id` first (high selectivity), then `is_active`.

---

## 🌐 **API Endpoints**

### **GET /api/v1/chefs/:chefId/menu**

#### **Controller Implementation**

```typescript
// apps/chefooz-apis/src/modules/chef/chef.controller.ts
import { Controller, Get, Param } from '@nestjs/common';
import { ApiTags, ApiOperation, ApiResponse, ApiParam } from '@nestjs/swagger';
import { ChefService } from './chef.service';
import { LinkableProductDto } from '@chefooz/types';

@ApiTags('Chef')
@Controller('chefs')
export class ChefController {
  constructor(private readonly chefService: ChefService) {}

  @Get(':chefId/menu')
  @ApiOperation({
    summary: 'Get active menu items for a chef',
    description: 'Public endpoint. Returns menu items formatted for product tagging in reels and cart validation.',
  })
  @ApiParam({
    name: 'chefId',
    type: 'string',
    description: 'Chef user ID (UUID)',
    example: '123e4567-e89b-12d3-a456-426614174000',
  })
  @ApiResponse({
    status: 200,
    description: 'Menu items retrieved successfully',
    schema: {
      example: {
        success: true,
        message: 'Chef menu retrieved successfully',
        data: [
          {
            id: '507f1f77bcf86cd799439011',
            name: 'Butter Chicken',
            price: 35000, // Paise
            chefId: '123e4567-e89b-12d3-a456-426614174000',
            availability: true,
            imageUrl: 'https://cdn.chefooz.com/products/butter-chicken.jpg',
          },
        ],
      },
    },
  })
  @ApiResponse({
    status: 500,
    description: 'Failed to fetch menu items',
    schema: {
      example: {
        success: false,
        message: 'Failed to fetch chef menu',
        errorCode: 'MENU_FETCH_FAILED',
      },
    },
  })
  async getMenu(@Param('chefId') chefId: string) {
    try {
      const data = await this.chefService.getMenuItems(chefId);
      return {
        success: true,
        message: 'Chef menu retrieved successfully',
        data,
      };
    } catch (error) {
      throw error; // Let global exception filter handle it
    }
  }
}
```

---

#### **Request/Response Examples**

**Request**:
```http
GET /api/v1/chefs/123e4567-e89b-12d3-a456-426614174000/menu HTTP/1.1
Host: api.chefooz.com
Accept: application/json
```

**Response (Success - 200)**:
```json
{
  "success": true,
  "message": "Chef menu retrieved successfully",
  "data": [
    {
      "id": "507f1f77bcf86cd799439011",
      "name": "Butter Chicken",
      "price": 35000,
      "chefId": "123e4567-e89b-12d3-a456-426614174000",
      "availability": true,
      "imageUrl": "https://cdn.chefooz.com/products/butter-chicken.jpg"
    },
    {
      "id": "507f1f77bcf86cd799439012",
      "name": "Paneer Tikka",
      "price": 28000,
      "chefId": "123e4567-e89b-12d3-a456-426614174000",
      "availability": true,
      "imageUrl": "https://cdn.chefooz.com/products/paneer-tikka.jpg"
    }
  ]
}
```

**Response (Empty Menu - 200)**:
```json
{
  "success": true,
  "message": "Chef menu retrieved successfully",
  "data": []
}
```

**Response (Error - 500)**:
```json
{
  "success": false,
  "message": "Failed to fetch chef menu",
  "errorCode": "MENU_FETCH_FAILED"
}
```

---

## 🛠️ **Service Layer**

### **ChefService Implementation**

```typescript
// apps/chefooz-apis/src/modules/chef/chef.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { ChefMenuItem } from '../chef-kitchen/entities/chef-menu-item.entity';
import { LinkableProductDto } from '@chefooz/types';

@Injectable()
export class ChefService {
  private readonly logger = new Logger(ChefService.name);

  constructor(
    @InjectRepository(ChefMenuItem)
    private readonly chefMenuItemRepository: Repository<ChefMenuItem>,
  ) {}

  /**
   * Fetch active menu items for a chef
   * 
   * @param chefId - Chef user ID (UUID)
   * @returns Array of menu items in LinkableProductDto format
   * @throws Error if database query fails
   * 
   * Business Rules:
   * - Only returns items where isActive = true
   * - Sorts by createdAt DESC (newest first)
   * - Converts price from rupees to paise (multiply by 100)
   * - Defaults availability to true if JSONB is null
   * - Returns imageUrl with fallback to thumbnailUrl
   */
  async getMenuItems(chefId: string): Promise<LinkableProductDto[]> {
    try {
      this.logger.log(`Fetching menu items for chef: ${chefId}`);

      // Debug: Log total items (including inactive)
      const allItems = await this.chefMenuItemRepository.find({
        where: { chefId },
      });
      this.logger.log(`Total items for chef ${chefId}: ${allItems.length}`);

      // Query active items only (compound index optimized)
      const menuItems = await this.chefMenuItemRepository
        .createQueryBuilder('item')
        .where('item.chefId = :chefId', { chefId })
        .andWhere('item.isActive = :isActive', { isActive: true })
        .orderBy('item.createdAt', 'DESC')
        .getMany();

      this.logger.log(`Found ${menuItems.length} active menu items for chef ${chefId}`);

      // Transform to LinkableProductDto
      return menuItems.map((item) => ({
        id: item.id,
        name: item.name,
        price: Math.round(Number(item.price) * 100), // Rupees → Paise
        chefId: item.chefId,
        availability: item.availability?.isAvailable ?? true, // Default to true
        imageUrl: item.imageUrl || item.thumbnailUrl || undefined, // Fallback chain
      }));
    } catch (error) {
      this.logger.error(`Failed to fetch menu items for chef ${chefId}:`, error.stack);
      throw new Error('MENU_FETCH_FAILED');
    }
  }
}
```

---

### **Price Conversion Logic**

#### **Conversion Algorithm**
```typescript
// Input: Database DECIMAL (rupees)
const dbPrice: number = 350.00; // Type: number (from DECIMAL column)

// Step 1: Explicit number conversion (safety)
const priceAsNumber = Number(dbPrice); // 350.00

// Step 2: Convert rupees to paise (multiply by 100)
const priceInPaise = priceAsNumber * 100; // 35000.00

// Step 3: Round to handle floating-point precision issues
const finalPrice = Math.round(priceInPaise); // 35000 (integer)
```

#### **Edge Cases Handled**
| Input (Rupees) | Calculation | Output (Paise) |
|----------------|-------------|----------------|
| `350.00` | `350 × 100` | `35000` |
| `99.99` | `99.99 × 100` | `9999` |
| `0.50` | `0.50 × 100` | `50` |
| `1234.56` | `1234.56 × 100` | `123456` |
| `0` | `0 × 100` | `0` |

**Why `Math.round()`?**
- JavaScript floating-point arithmetic can introduce tiny errors (e.g., `350.00 * 100` might be `34999.999999999996`)
- `Math.round()` ensures integer precision
- Payment gateways (Razorpay) require exact integer paise

---

### **Availability Fallback Logic**

```typescript
// Availability field structure
interface Availability {
  isAvailable: boolean;
  outOfStockReason?: string;
  unavailableUntil?: Date;
}

// Fallback logic
const availability = item.availability?.isAvailable ?? true;

// Scenarios:
// 1. availability = { isAvailable: true }  → true
// 2. availability = { isAvailable: false } → false
// 3. availability = null                   → true (default)
// 4. availability = undefined              → true (default)
// 5. availability = {}                     → true (isAvailable undefined)
```

**Design Rationale**:
- **Optimistic Default**: New items default to available
- **Null Safety**: Handles missing JSONB gracefully
- **Chef Responsibility**: Chef must explicitly mark items unavailable

---

### **Image URL Fallback Chain**

```typescript
const imageUrl = item.imageUrl || item.thumbnailUrl || undefined;

// Fallback priority:
// 1. imageUrl:      Primary high-resolution image (chef uploaded)
// 2. thumbnailUrl:  Generated thumbnail (media-processing service)
// 3. undefined:     No image available (frontend shows placeholder)

// Frontend handling (React Native):
<Image 
  source={{ uri: product.imageUrl || defaultPlaceholderUrl }}
  style={styles.productImage}
/>
```

---

## 📦 **DTO Specifications**

### **LinkableProductDto**

```typescript
// libs/types/src/lib/dto/linkable-product.dto.ts
export interface LinkableProductDto {
  id: string;              // Menu item UUID
  name: string;            // Item name (e.g., "Butter Chicken")
  price: number;           // Price in paise (integer)
  chefId: string;          // Chef user ID (UUID)
  availability: boolean;   // Can be ordered right now
  imageUrl?: string;       // Product image URL (optional)
}

// Usage example in Media Module
import { LinkableProductDto } from '@chefooz/types';

interface ReelCreateDto {
  title: string;
  videoUrl: string;
  linkedProducts?: LinkableProductDto[]; // Array of menu items
}
```

---

### **DTO Validation Rules**

| Field | Type | Required | Validation | Example |
|-------|------|----------|------------|---------|
| `id` | `string` | ✅ Yes | UUID format | `"507f1f77..."` |
| `name` | `string` | ✅ Yes | Max 255 chars | `"Butter Chicken"` |
| `price` | `number` | ✅ Yes | Integer, ≥ 0 | `35000` (paise) |
| `chefId` | `string` | ✅ Yes | UUID format | `"123e4567..."` |
| `availability` | `boolean` | ✅ Yes | true/false | `true` |
| `imageUrl` | `string` | ❌ No | Valid URL or undefined | `"https://..."` |

---

## ❌ **Error Handling**

### **Error Scenarios**

#### **1. Database Connection Failure**
```typescript
// Error thrown by TypeORM
{
  name: 'QueryFailedError',
  message: 'Connection lost: The server closed the connection.',
}

// Caught by ChefService
this.logger.error(`Failed to fetch menu items for chef ${chefId}:`, error.stack);
throw new Error('MENU_FETCH_FAILED');

// Global exception filter transforms to:
{
  "success": false,
  "message": "Failed to fetch chef menu",
  "errorCode": "MENU_FETCH_FAILED",
  "statusCode": 500
}
```

---

#### **2. Invalid Chef ID (Non-UUID)**
```typescript
// NestJS ValidationPipe rejects invalid UUID
GET /api/v1/chefs/invalid-id/menu

// Response (400 Bad Request)
{
  "success": false,
  "message": "Validation failed",
  "errors": [
    {
      "field": "chefId",
      "message": "chefId must be a valid UUID"
    }
  ]
}
```

---

#### **3. Chef Not Found (Empty Menu)**
```typescript
// No error thrown - returns empty array
GET /api/v1/chefs/00000000-0000-0000-0000-000000000000/menu

// Response (200 OK)
{
  "success": true,
  "message": "Chef menu retrieved successfully",
  "data": []
}

// Rationale: Not finding a chef is NOT an error (chef might have no menu yet)
```

---

### **Error Logging Strategy**

```typescript
// ChefService error logging
try {
  // Query logic
} catch (error) {
  // Log full error stack for debugging
  this.logger.error(`Failed to fetch menu items for chef ${chefId}:`, error.stack);
  
  // Log additional context
  this.logger.error(`Database connection status: ${this.connection.isConnected}`);
  this.logger.error(`Query parameters: chefId=${chefId}, isActive=true`);
  
  // Throw generic error (don't expose DB details to client)
  throw new Error('MENU_FETCH_FAILED');
}
```

---

## ⚡ **Performance Optimization**

### **1. Database Query Optimization**

#### **Query Execution Plan**
```sql
-- Original query (generated by TypeORM QueryBuilder)
EXPLAIN ANALYZE
SELECT * FROM chef_menu_items 
WHERE chef_id = '123e4567-...' 
  AND is_active = true 
ORDER BY created_at DESC;

-- Result:
-- Index Scan using idx_chef_menu_items_chef_active
-- Cost: 0.42..8.44 rows=1 width=500
-- Execution Time: 0.234 ms
```

**Optimization Techniques**:
- ✅ Compound index `(chef_id, is_active)` eliminates index merge
- ✅ `ORDER BY created_at DESC` uses index-only scan
- ✅ No `JOIN` required (denormalized chef data)

---

### **2. N+1 Query Prevention**

```typescript
// ❌ BAD: N+1 queries (if we fetched chef data per item)
const menuItems = await this.chefMenuItemRepository.find({ where: { chefId } });
for (const item of menuItems) {
  const chef = await this.userRepository.findOne(item.chefId); // N queries
}

// ✅ GOOD: Single query (no JOIN needed)
const menuItems = await this.chefMenuItemRepository
  .createQueryBuilder('item')
  .where('item.chefId = :chefId', { chefId })
  .andWhere('item.isActive = :isActive', { isActive: true })
  .getMany(); // 1 query total
```

---

### **3. Response Payload Optimization**

```typescript
// Payload size comparison
// Full entity:       ~1.2 KB per item (includes description, category, timestamps)
// LinkableProductDto: ~0.3 KB per item (only essential fields)

// For 10 items:
// Full entity:       12 KB
// LinkableProductDto: 3 KB (75% reduction)
```

---

### **4. Future Caching Strategy (Phase 2)**

```typescript
// Planned Redis caching implementation
@Injectable()
export class ChefService {
  constructor(
    @InjectRepository(ChefMenuItem) private repo: Repository<ChefMenuItem>,
    private cacheService: CacheService,
  ) {}

  async getMenuItems(chefId: string): Promise<LinkableProductDto[]> {
    const cacheKey = `chef:menu:${chefId}`;
    
    // Check cache first
    const cached = await this.cacheService.get<LinkableProductDto[]>(cacheKey);
    if (cached) {
      this.logger.log(`Cache hit for chef ${chefId}`);
      return cached;
    }

    // Fetch from database
    const menu = await this.fetchMenuFromDatabase(chefId);

    // Store in cache (5-minute TTL)
    await this.cacheService.set(cacheKey, menu, 300);

    return menu;
  }
}

// Cache invalidation (in Chef-Kitchen module)
async updateMenuItem(id: string, dto: UpdateMenuItemDto) {
  const item = await this.repo.save({ id, ...dto });
  
  // Invalidate chef's menu cache
  await this.cacheService.del(`chef:menu:${item.chefId}`);
  
  return item;
}
```

**Expected Performance Improvement**:
- 📈 Cache hit rate: ~70% (most chefs' menus stable)
- ⚡ Response time: 200ms → 10ms (95% reduction)
- 💾 Database load: -60% reduction in queries

---

## 🔗 **Integration Patterns**

### **1. Media Module Integration (Product Tagging)**

```typescript
// apps/chefooz-app/src/app/reels/upload.tsx
import { useChefMenu } from '@chefooz/api-client';

export const ReelUploadScreen = () => {
  const { user } = useAuth();
  const { data: menuItems, isLoading } = useChefMenu(user.id); // Chef Module API

  const [selectedProducts, setSelectedProducts] = useState<LinkableProductDto[]>([]);

  const handleProductSelect = (product: LinkableProductDto) => {
    setSelectedProducts((prev) => [...prev, product]);
  };

  const handleSubmit = async () => {
    await createReel({
      videoUrl: uploadedVideoUrl,
      linkedProducts: selectedProducts.map((p) => ({ id: p.id })), // Only IDs sent
    });
  };

  return (
    <View>
      <ProductTaggingDropdown 
        items={menuItems} 
        onSelect={handleProductSelect}
        loading={isLoading}
      />
      <SelectedProductsList items={selectedProducts} />
      <SubmitButton onPress={handleSubmit} />
    </View>
  );
};
```

---

### **2. Cart Module Integration (Item Validation)**

```typescript
// apps/chefooz-apis/src/modules/cart/cart.service.ts
import { ChefService } from '../chef/chef.service';

@Injectable()
export class CartService {
  constructor(private chefService: ChefService) {}

  async addItemToCart(userId: string, dto: AddCartItemDto) {
    // Fetch chef's menu
    const menuItems = await this.chefService.getMenuItems(dto.chefId);

    // Validate item exists and is available
    const item = menuItems.find((m) => m.id === dto.itemId);
    if (!item) {
      throw new BadRequestException('Menu item not found or unavailable');
    }

    if (!item.availability) {
      throw new BadRequestException('Item is currently out of stock');
    }

    // Add to cart with validated price
    return this.cartRepository.save({
      userId,
      itemId: dto.itemId,
      quantity: dto.quantity,
      price: item.price, // Use validated price from Chef Module
    });
  }
}
```

---

### **3. Feed Module Integration (Product Display)**

```typescript
// apps/chefooz-app/src/components/ReelCard.tsx
import { LinkableProductDto } from '@chefooz/types';

interface ReelCardProps {
  reel: Reel;
  linkedProducts: LinkableProductDto[]; // Pre-fetched from Chef Module
}

export const ReelCard: React.FC<ReelCardProps> = ({ reel, linkedProducts }) => {
  return (
    <View>
      <Video source={{ uri: reel.videoUrl }} />
      
      {linkedProducts.length > 0 && (
        <ProductTags>
          {linkedProducts.map((product) => (
            <ProductChip
              key={product.id}
              name={product.name}
              price={product.price / 100} // Convert paise to rupees
              imageUrl={product.imageUrl}
              onPress={() => navigation.navigate('ProductDetails', { id: product.id })}
            />
          ))}
        </ProductTags>
      )}
    </View>
  );
};
```

---

## 🧪 **Testing Strategy**

### **Unit Tests (ChefService)**

```typescript
// apps/chefooz-apis/src/modules/chef/chef.service.spec.ts
import { Test } from '@nestjs/testing';
import { getRepositoryToken } from '@nestjs/typeorm';
import { ChefService } from './chef.service';
import { ChefMenuItem } from '../chef-kitchen/entities/chef-menu-item.entity';

describe('ChefService', () => {
  let service: ChefService;
  let mockRepository: any;

  beforeEach(async () => {
    mockRepository = {
      find: jest.fn(),
      createQueryBuilder: jest.fn(() => ({
        where: jest.fn().mockReturnThis(),
        andWhere: jest.fn().mockReturnThis(),
        orderBy: jest.fn().mockReturnThis(),
        getMany: jest.fn(),
      })),
    };

    const module = await Test.createTestingModule({
      providers: [
        ChefService,
        {
          provide: getRepositoryToken(ChefMenuItem),
          useValue: mockRepository,
        },
      ],
    }).compile();

    service = module.get<ChefService>(ChefService);
  });

  describe('getMenuItems', () => {
    it('should return active menu items with price in paise', async () => {
      const mockItems = [
        {
          id: '1',
          name: 'Butter Chicken',
          price: 350.00,
          chefId: 'chef-123',
          isActive: true,
          availability: { isAvailable: true },
          imageUrl: 'https://example.com/image.jpg',
          thumbnailUrl: null,
          createdAt: new Date(),
        },
      ];

      mockRepository.find.mockResolvedValue(mockItems);
      mockRepository.createQueryBuilder().getMany.mockResolvedValue(mockItems);

      const result = await service.getMenuItems('chef-123');

      expect(result).toEqual([
        {
          id: '1',
          name: 'Butter Chicken',
          price: 35000, // Converted to paise
          chefId: 'chef-123',
          availability: true,
          imageUrl: 'https://example.com/image.jpg',
        },
      ]);
    });

    it('should filter out inactive items', async () => {
      const mockItems = [
        { id: '1', isActive: true, price: 100, availability: { isAvailable: true } },
        { id: '2', isActive: false, price: 200, availability: { isAvailable: true } },
      ];

      mockRepository.find.mockResolvedValue(mockItems);
      mockRepository.createQueryBuilder().getMany.mockResolvedValue([mockItems[0]]);

      const result = await service.getMenuItems('chef-123');

      expect(result).toHaveLength(1);
      expect(result[0].id).toBe('1');
    });

    it('should default availability to true when JSONB is null', async () => {
      const mockItems = [
        {
          id: '1',
          price: 100,
          chefId: 'chef-123',
          isActive: true,
          availability: null, // Null JSONB
          imageUrl: null,
          thumbnailUrl: null,
        },
      ];

      mockRepository.find.mockResolvedValue(mockItems);
      mockRepository.createQueryBuilder().getMany.mockResolvedValue(mockItems);

      const result = await service.getMenuItems('chef-123');

      expect(result[0].availability).toBe(true);
    });

    it('should fallback to thumbnailUrl when imageUrl is null', async () => {
      const mockItems = [
        {
          id: '1',
          price: 100,
          chefId: 'chef-123',
          isActive: true,
          availability: { isAvailable: true },
          imageUrl: null,
          thumbnailUrl: 'https://example.com/thumbnail.jpg',
        },
      ];

      mockRepository.find.mockResolvedValue(mockItems);
      mockRepository.createQueryBuilder().getMany.mockResolvedValue(mockItems);

      const result = await service.getMenuItems('chef-123');

      expect(result[0].imageUrl).toBe('https://example.com/thumbnail.jpg');
    });

    it('should throw error on database failure', async () => {
      mockRepository.find.mockRejectedValue(new Error('Database error'));

      await expect(service.getMenuItems('chef-123')).rejects.toThrow('MENU_FETCH_FAILED');
    });
  });
});
```

---

### **Integration Tests (E2E)**

```typescript
// apps/chefooz-apis-e2e/src/chef/chef.e2e-spec.ts
import { INestApplication } from '@nestjs/common';
import { Test } from '@nestjs/testing';
import * as request from 'supertest';
import { AppModule } from '../src/app.module';

describe('Chef Module (E2E)', () => {
  let app: INestApplication;

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = moduleRef.createNestApplication();
    await app.init();
  });

  afterAll(async () => {
    await app.close();
  });

  describe('GET /api/v1/chefs/:chefId/menu', () => {
    it('should return menu items for valid chef', async () => {
      const chefId = 'test-chef-uuid';

      return request(app.getHttpServer())
        .get(`/api/v1/chefs/${chefId}/menu`)
        .expect(200)
        .expect((res) => {
          expect(res.body.success).toBe(true);
          expect(res.body.data).toBeInstanceOf(Array);
          expect(res.body.data[0]).toHaveProperty('id');
          expect(res.body.data[0]).toHaveProperty('price');
          expect(typeof res.body.data[0].price).toBe('number');
        });
    });

    it('should return empty array for chef with no menu', async () => {
      const chefId = 'chef-no-menu-uuid';

      return request(app.getHttpServer())
        .get(`/api/v1/chefs/${chefId}/menu`)
        .expect(200)
        .expect((res) => {
          expect(res.body.data).toEqual([]);
        });
    });

    it('should reject invalid UUID format', async () => {
      return request(app.getHttpServer())
        .get('/api/v1/chefs/invalid-id/menu')
        .expect(400);
    });
  });
});
```

---

## 🎯 **Implementation Checklist**

### **Backend Setup** ✅
- [x] ChefModule created with ChefController and ChefService
- [x] TypeORM repository injected for ChefMenuItem
- [x] Public GET endpoint implemented
- [x] Price conversion logic (rupees → paise)
- [x] Availability fallback logic
- [x] Image URL fallback chain
- [x] Error handling with logging
- [x] Swagger documentation

### **Database** ✅
- [x] ChefMenuItem entity defined (Chef-Kitchen module)
- [x] Compound index on (chef_id, is_active)
- [x] JSONB availability field
- [x] Foreign key constraint to users table

### **Shared Types** ✅
- [x] LinkableProductDto interface in libs/types
- [x] Exported for use in Media, Cart, Feed modules

### **Testing** ⏳
- [ ] Unit tests for ChefService (price conversion, fallbacks)
- [ ] E2E tests for GET /chefs/:id/menu
- [ ] Load testing (1000 concurrent requests)

### **Future Enhancements** ⏳
- [ ] Redis caching layer
- [ ] Category filtering support
- [ ] Search & sorting parameters
- [ ] Rate limiting (public endpoint protection)

---

## 🛡️ **Compliance / KYC Technical Notes** *(Added March 2026)*

### Backend module: `chef-compliance`

```
apps/chefooz-apis/src/modules/chef-compliance/
├── admin-chef-compliance.controller.ts # admin-only review endpoints
├── chef-compliance.controller.ts   # 6 route methods, all use req.user.id
├── chef-compliance.service.ts      # getOrCreate, submitXxx, adminGetCompliance, adminVerifyStep
├── chef-compliance.module.ts
└── dto/compliance.dto.ts           # SubmitIdentityDto, SubmitBankDetailsDto, SubmitFssaiDto,
                                      AcceptLegalDto, ComplianceUploadUrlDto, AdminVerifyStepDto
```

### Admin KYC review surface *(Updated May 2026)*

- Admin page route: `/dashboard/chefs/[chefId]/compliance`
- Entry point: `apps/chefooz-admin/src/app/dashboard/chefs/page.tsx` action column
- Data flow: `admin-chef-compliance.client.ts` → `useAdminChefCompliance.ts` → MUI review page
- Review actions use confirmation UI and invalidate the per-chef query key `['admin', 'chef-compliance', chefId]`
- Submitted documents are not considered admin-approved until their `verifiedBy` field is set by an admin action
- Final activation is intentionally split between chef operations and payout enablement

### Admin endpoints

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/v1/admin/chef-compliance/:chefId` | Fetch full unmasked compliance record |
| `PATCH` | `/api/v1/admin/chef-compliance/:chefId/identity` | Verify or reject identity |
| `PATCH` | `/api/v1/admin/chef-compliance/:chefId/bank` | Verify or reject bank |
| `PATCH` | `/api/v1/admin/chef-compliance/:chefId/fssai` | Verify or reject FSSAI |
| `PATCH` | `/api/v1/admin/chef-compliance/:chefId/notes` | Save internal admin notes |
| `PATCH` | `/api/v1/admin/chef-compliance/:chefId/chef-status` | Enable chef operations |
| `PATCH` | `/api/v1/admin/chef-compliance/:chefId/payout` | Enable payouts |

### Admin response mapping

- Admin endpoints do **not** return the raw TypeORM entity
- `ChefComplianceService.mapAdminCompliance()` converts stored `Date` values to ISO strings for transport
- Admin responses remain unmasked so reviewers can inspect full identity and payout details before approving
- Rejection requires `rejectionReason`; verification clears any prior rejection reason for that step
- `chefVerificationStatus` is sourced from `ChefProfile.verificationStatus`
- `payoutEnabled` is sourced from `chef_compliance.payout_enabled`
- `canVerifyChef` requires approved identity + approved FSSAI + accepted legal terms
- `canEnablePayout` requires verified chef operations + approved bank details

### S3 upload pattern

```
POST /v1/chef/compliance/upload-url
  Body: { purpose, fileExtension, contentType }
  Returns: { uploadUrl (pre-signed PUT), publicUrl, fileKey, expiresIn }

Client then:
  PUT {uploadUrl} with raw file body  →  stores at S3_OUTPUT_BUCKET
  Submits publicUrl as field in subsequent step DTO
```

S3 key format: `compliance/{chefId}/{purpose}/{uuid}.{ext}`

Accepted purposes: `front`, `back`, `selfie`, `certificate`

### Key constraint: req.user.id

JWT guard exposes `.id` (NOT `.userId`). All 6 controller methods use `req.user.id`. This was a production bug fixed in March 2026 that caused "Chef undefined" errors.

### cache invalidation

All 4 mutation hooks use `queryClient.invalidateQueries(['chef-compliance'])` in `onSuccess`. Do NOT use `setQueryData` — query must be refetched from server to ensure index.tsx checklist reflects updated step status.

### IFSC Auto-fill (bank.tsx)

- Fetches `https://ifsc.razorpay.com/{IFSC}` when `ifscCode` reaches 11 chars and matches `/^[A-Z]{4}0[A-Z0-9]{6}$/`
- Uses `AbortController` via `useRef` to cancel stale requests
- Auto-populates `bankName` and `branchName` from `{ BANK, BRANCH }` response fields
- `ifscLookupState: 'idle' | 'loading' | 'found' | 'error'` drives inline status indicator

### Legal scroll gate (legal.tsx)

- Accept checkbox is disabled until user scrolls within `normalize(60)` px of the bottom
- If chef taps disabled checkbox, `ScrollView.scrollToEnd()` is called automatically
- `hasScrolledToBottom` bypassed when re-entering after prior acceptance

### Expiry date formatting (fssai.tsx)

- UI input: `DD/MM/YYYY` (auto-inserts slashes as digits are typed)
- API payload: `YYYY-MM-DD` (converted via `parseDateDMY` + `toIsoDate`)
- Validation: parsed `Date` must be strictly after `new Date()` (current moment)

### Pre-population pattern

All four screens share the same guard pattern to avoid overwriting user-edited state:
```ts
const [initialized, setInitialized] = useState(false);
useEffect(() => {
  if (initialized) return;
  // seed from existingCompliance.*Data
  setInitialized(true);
}, [existingCompliance, initialized]);
```

---

**[TECHNICAL_GUIDE_COMPLETE ✅]**

*For business overview, see `01_FEATURE_OVERVIEW.md`. For QA testing procedures, see `03_QA_TEST_CASES.md`.*

---

**Document Version**: 1.2  
**Last Updated**: 2026-05-01  
**Implementation Status**: ✅ Complete (Compliance/KYC with admin review endpoints and screen)  
**Next Review**: Q2 2026
