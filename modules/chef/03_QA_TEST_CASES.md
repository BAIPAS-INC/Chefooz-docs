# 🧪 Chef Module - QA Test Cases

## 📋 **Table of Contents**
- [Testing Overview](#testing-overview)
- [Test Environment Setup](#test-environment-setup)
- [API Endpoint Tests](#api-endpoint-tests)
- [Service Layer Tests](#service-layer-tests)
- [Integration Tests](#integration-tests)
- [Performance Tests](#performance-tests)
- [Security Tests](#security-tests)
- [Edge Case Tests](#edge-case-tests)
- [Regression Tests](#regression-tests)

---

## 🎯 **Testing Overview**

### **Test Scope**
This document covers comprehensive QA testing for the Chef Module, focusing on the **public menu retrieval API** and its integration with:
- Media Module (product tagging)
- Cart Module (item validation)
- Feed/Explore modules (product display)

### **Test Categories**
| Category | Test Count | Priority | Automation |
|----------|------------|----------|------------|
| **API Endpoint Tests** | 12 | 🔴 Critical | ✅ Automated |
| **Service Layer Tests** | 10 | 🔴 Critical | ✅ Automated |
| **Integration Tests** | 8 | 🟠 High | ✅ Automated |
| **Performance Tests** | 5 | 🟠 High | ⚠️ Manual + Automated |
| **Security Tests** | 3 | 🔴 Critical | ✅ Automated |
| **Edge Case Tests** | 6 | 🟡 Medium | ✅ Automated |
| **Regression Tests** | 4 | 🟠 High | ✅ Automated |
| **Total** | **48** | - | - |

---

## 🛠️ **Test Environment Setup**

### **Prerequisites**
```bash
# 1. Start PostgreSQL database
docker-compose up -d postgres

# 2. Run database migrations
npm run migration:run

# 3. Seed test data
npm run seed:chef-test-data

# 4. Start API server
npm run start:dev
```

### **Test Data Setup**

```sql
-- Test Chef Users
INSERT INTO users (id, username, role) VALUES
  ('chef-001', 'chef_active', 'chef'),
  ('chef-002', 'chef_empty_menu', 'chef'),
  ('chef-003', 'chef_all_inactive', 'chef'),
  ('chef-004', 'chef_large_menu', 'chef');

-- Test Menu Items (Chef 001 - Active Menu)
INSERT INTO chef_menu_items (id, chef_id, name, price, is_active, availability, image_url) VALUES
  ('item-001', 'chef-001', 'Butter Chicken', 350.00, true, '{"isAvailable": true}', 'https://cdn.chefooz.com/test/butter-chicken.jpg'),
  ('item-002', 'chef-001', 'Paneer Tikka', 280.00, true, '{"isAvailable": true}', 'https://cdn.chefooz.com/test/paneer-tikka.jpg'),
  ('item-003', 'chef-001', 'Dal Makhani', 200.00, false, '{"isAvailable": true}', NULL), -- Inactive
  ('item-004', 'chef-001', 'Biryani', 400.00, true, '{"isAvailable": false}', NULL); -- Unavailable

-- Test Menu Items (Chef 003 - All Inactive)
INSERT INTO chef_menu_items (id, chef_id, name, price, is_active) VALUES
  ('item-010', 'chef-003', 'Old Item 1', 100.00, false),
  ('item-011', 'chef-003', 'Old Item 2', 150.00, false);

-- Test Menu Items (Chef 004 - Large Menu, 50 items)
-- (Generated via script for performance testing)
```

### **Environment Variables**
```env
# .env.test
DATABASE_URL=postgresql://test:test@localhost:5432/chefooz_test
NODE_ENV=test
LOG_LEVEL=debug
```

---

## 🌐 **API Endpoint Tests**

### **TC-CHEF-API-001: Fetch Active Menu Items (Happy Path)**

**Objective**: Verify API returns active menu items for valid chef ID

**Preconditions**:
- Chef `chef-001` has 4 menu items (2 active, 2 inactive/unavailable)

**Test Steps**:
```http
GET /api/v1/chefs/chef-001/menu HTTP/1.1
Host: api.chefooz.com
Accept: application/json
```

**Expected Result**:
```json
{
  "success": true,
  "message": "Chef menu retrieved successfully",
  "data": [
    {
      "id": "item-001",
      "name": "Butter Chicken",
      "price": 35000,
      "chefId": "chef-001",
      "availability": true,
      "imageUrl": "https://cdn.chefooz.com/test/butter-chicken.jpg"
    },
    {
      "id": "item-002",
      "name": "Paneer Tikka",
      "price": 28000,
      "chefId": "chef-001",
      "availability": true,
      "imageUrl": "https://cdn.chefooz.com/test/paneer-tikka.jpg"
    },
    {
      "id": "item-004",
      "name": "Biryani",
      "price": 40000,
      "chefId": "chef-001",
      "availability": false,
      "imageUrl": null
    }
  ]
}
```

**Verification**:
- ✅ Response status: `200 OK`
- ✅ `data` array contains 3 items (excludes `item-003` with `isActive: false`)
- ✅ Prices converted to paise (`350.00 → 35000`)
- ✅ Items sorted by `createdAt DESC`
- ✅ `item-004` included (active but unavailable - availability reflects JSONB)

**Automation Script** (PowerShell):
```powershell
$response = Invoke-RestMethod -Uri "https://api.chefooz.com/api/v1/chefs/chef-001/menu" -Method Get
if ($response.success -eq $true -and $response.data.Count -eq 3) {
    Write-Host "✅ TC-CHEF-API-001 PASSED" -ForegroundColor Green
} else {
    Write-Host "❌ TC-CHEF-API-001 FAILED" -ForegroundColor Red
}
```

---

### **TC-CHEF-API-002: Empty Menu (Chef with No Items)**

**Objective**: Verify API returns empty array for chef with no menu items

**Preconditions**:
- Chef `chef-002` has 0 menu items

**Test Steps**:
```http
GET /api/v1/chefs/chef-002/menu HTTP/1.1
```

**Expected Result**:
```json
{
  "success": true,
  "message": "Chef menu retrieved successfully",
  "data": []
}
```

**Verification**:
- ✅ Response status: `200 OK` (NOT 404)
- ✅ `data` is empty array `[]`
- ✅ `success: true` (no error thrown)

---

### **TC-CHEF-API-003: All Inactive Items (Chef with Only Hidden Items)**

**Objective**: Verify API returns empty array when all items are inactive

**Preconditions**:
- Chef `chef-003` has 2 menu items, both with `isActive: false`

**Test Steps**:
```http
GET /api/v1/chefs/chef-003/menu HTTP/1.1
```

**Expected Result**:
```json
{
  "success": true,
  "message": "Chef menu retrieved successfully",
  "data": []
}
```

**Verification**:
- ✅ Response status: `200 OK`
- ✅ `data` is empty array (inactive items filtered out)
- ✅ No partial data returned

---

### **TC-CHEF-API-004: Invalid Chef ID (Non-UUID Format)**

**Objective**: Verify API rejects invalid UUID format

**Test Steps**:
```http
GET /api/v1/chefs/invalid-id-123/menu HTTP/1.1
```

**Expected Result**:
```json
{
  "success": false,
  "message": "Validation failed",
  "errors": [
    {
      "field": "chefId",
      "message": "chefId must be a valid UUID"
    }
  ],
  "statusCode": 400
}
```

**Verification**:
- ✅ Response status: `400 Bad Request`
- ✅ Error message specifies UUID validation failure
- ✅ No database query executed (validation occurs at controller level)

---

### **TC-CHEF-API-005: Non-Existent Chef (Valid UUID, No Data)**

**Objective**: Verify API returns empty array for valid but non-existent chef ID

**Test Steps**:
```http
GET /api/v1/chefs/00000000-0000-0000-0000-000000000000/menu HTTP/1.1
```

**Expected Result**:
```json
{
  "success": true,
  "message": "Chef menu retrieved successfully",
  "data": []
}
```

**Verification**:
- ✅ Response status: `200 OK` (NOT 404)
- ✅ `data` is empty array
- ✅ Distinguishes between "no items" and "chef not found" (same response for UX simplicity)

---

### **TC-CHEF-API-006: Price Conversion (Rupees to Paise)**

**Objective**: Verify all prices correctly converted from rupees to paise

**Test Steps**:
1. Insert test item: `price: 99.99`
2. Fetch menu for chef
3. Verify price in response

**Expected Result**:
```json
{
  "data": [
    {
      "id": "item-test",
      "name": "Test Item",
      "price": 9999,  // 99.99 × 100 = 9999 paise
      "chefId": "chef-001"
    }
  ]
}
```

**Verification**:
- ✅ `price` is integer (no decimals)
- ✅ Calculation: `99.99 × 100 = 9999` (correct)
- ✅ No floating-point precision errors

**Test Cases for Price Conversion**:
| Database (Rupees) | Expected (Paise) | Result |
|-------------------|------------------|--------|
| `350.00` | `35000` | ✅ Pass |
| `0.50` | `50` | ✅ Pass |
| `1234.56` | `123456` | ✅ Pass |
| `0` | `0` | ✅ Pass |
| `999.99` | `99999` | ✅ Pass |

---

### **TC-CHEF-API-007: Image URL Fallback Chain**

**Objective**: Verify image URL fallback logic (imageUrl → thumbnailUrl → undefined)

**Test Steps**:
1. Create 3 test items:
   - Item A: `imageUrl: 'https://...', thumbnailUrl: null`
   - Item B: `imageUrl: null, thumbnailUrl: 'https://...'`
   - Item C: `imageUrl: null, thumbnailUrl: null`
2. Fetch menu

**Expected Result**:
```json
{
  "data": [
    {
      "id": "item-a",
      "imageUrl": "https://cdn.chefooz.com/full.jpg"  // Primary
    },
    {
      "id": "item-b",
      "imageUrl": "https://cdn.chefooz.com/thumb.jpg"  // Fallback
    },
    {
      "id": "item-c",
      "imageUrl": null  // No image
    }
  ]
}
```

**Verification**:
- ✅ Item A returns `imageUrl` (primary)
- ✅ Item B returns `thumbnailUrl` (fallback)
- ✅ Item C returns `null` (no image available)

---

### **TC-CHEF-API-008: Availability Fallback (Null JSONB)**

**Objective**: Verify availability defaults to `true` when JSONB is null

**Test Steps**:
1. Insert item with `availability: NULL`
2. Fetch menu

**Expected Result**:
```json
{
  "data": [
    {
      "id": "item-null-avail",
      "availability": true  // Defaulted to true
    }
  ]
}
```

**Verification**:
- ✅ `availability: true` (not null or undefined)
- ✅ Fallback logic: `item.availability?.isAvailable ?? true`

---

### **TC-CHEF-API-009: Sorting Order (Chronological DESC)**

**Objective**: Verify items sorted by `createdAt DESC` (newest first)

**Test Steps**:
1. Insert 3 items with timestamps:
   - Item A: `2026-02-01 10:00:00`
   - Item B: `2026-02-01 12:00:00`
   - Item C: `2026-02-01 11:00:00`
2. Fetch menu

**Expected Result**:
```json
{
  "data": [
    { "id": "item-b", "createdAt": "2026-02-01T12:00:00Z" },  // Newest
    { "id": "item-c", "createdAt": "2026-02-01T11:00:00Z" },
    { "id": "item-a", "createdAt": "2026-02-01T10:00:00Z" }   // Oldest
  ]
}
```

**Verification**:
- ✅ Items in DESC order by `createdAt`
- ✅ Newest item appears first in array

---

### **TC-CHEF-API-010: Public Access (No Authentication Required)**

**Objective**: Verify endpoint accessible without JWT token

**Test Steps**:
```http
GET /api/v1/chefs/chef-001/menu HTTP/1.1
# No Authorization header
```

**Expected Result**:
- ✅ Response status: `200 OK`
- ✅ No `401 Unauthorized` error
- ✅ Data returned successfully

**Contrast with Private Endpoint**:
```http
# Protected endpoint (for comparison)
GET /api/v1/chef-kitchen/my-menu HTTP/1.1
# Returns: 401 Unauthorized (requires JWT)
```

---

### **TC-CHEF-API-011: Database Error Handling**

**Objective**: Verify graceful error handling on database failure

**Test Steps**:
1. Simulate database connection failure (stop PostgreSQL)
2. Attempt to fetch menu

**Expected Result**:
```json
{
  "success": false,
  "message": "Failed to fetch chef menu",
  "errorCode": "MENU_FETCH_FAILED",
  "statusCode": 500
}
```

**Verification**:
- ✅ Response status: `500 Internal Server Error`
- ✅ Generic error message (no database details exposed)
- ✅ Error logged to server logs (with stack trace)
- ✅ No sensitive data in response

---

### **TC-CHEF-API-012: Response Envelope Consistency**

**Objective**: Verify all responses follow standard envelope format

**Test Steps**:
1. Fetch menu (success case)
2. Trigger error (invalid UUID)
3. Compare response structures

**Expected Structure**:
```typescript
// Success response
{
  success: true,
  message: string,
  data: any[]
}

// Error response
{
  success: false,
  message: string,
  errorCode?: string,
  statusCode: number
}
```

**Verification**:
- ✅ Both responses have `success` boolean
- ✅ Both responses have `message` string
- ✅ Success responses have `data` field
- ✅ Error responses have `errorCode` and `statusCode`

---

## 🔧 **Service Layer Tests**

### **TC-CHEF-SVC-001: getMenuItems() - Active Filter Logic**

**Objective**: Verify service method filters inactive items at database level

**Test Setup**:
```typescript
// Mock repository data
const mockMenuItems = [
  { id: '1', isActive: true, name: 'Item 1' },
  { id: '2', isActive: false, name: 'Item 2' },
  { id: '3', isActive: true, name: 'Item 3' },
];
```

**Test Code**:
```typescript
it('should only return active items', async () => {
  mockRepository.createQueryBuilder().getMany.mockResolvedValue([
    mockMenuItems[0],
    mockMenuItems[2],
  ]);

  const result = await service.getMenuItems('chef-001');

  expect(result).toHaveLength(2);
  expect(result[0].id).toBe('1');
  expect(result[1].id).toBe('3');
  expect(mockRepository.createQueryBuilder).toHaveBeenCalledWith('item');
  expect(mockRepository.createQueryBuilder().andWhere).toHaveBeenCalledWith(
    'item.isActive = :isActive',
    { isActive: true }
  );
});
```

**Verification**:
- ✅ Only active items returned
- ✅ Database query includes `isActive = true` filter
- ✅ Inactive items never loaded into memory

---

### **TC-CHEF-SVC-002: Price Conversion (Rupees → Paise)**

**Objective**: Verify `Math.round(price * 100)` logic

**Test Cases**:
```typescript
describe('Price conversion', () => {
  const testCases = [
    { dbPrice: 350.00, expected: 35000 },
    { dbPrice: 99.99, expected: 9999 },
    { dbPrice: 0.50, expected: 50 },
    { dbPrice: 1234.56, expected: 123456 },
    { dbPrice: 0, expected: 0 },
  ];

  testCases.forEach(({ dbPrice, expected }) => {
    it(`should convert ${dbPrice} rupees to ${expected} paise`, async () => {
      mockRepository.createQueryBuilder().getMany.mockResolvedValue([
        { id: '1', price: dbPrice, chefId: 'chef-001', isActive: true },
      ]);

      const result = await service.getMenuItems('chef-001');

      expect(result[0].price).toBe(expected);
    });
  });
});
```

**Verification**:
- ✅ All prices correctly converted
- ✅ `Math.round()` handles floating-point precision
- ✅ Result is integer (no decimals)

---

### **TC-CHEF-SVC-003: Availability Fallback (Null JSONB)**

**Objective**: Verify `item.availability?.isAvailable ?? true` logic

**Test Cases**:
```typescript
describe('Availability fallback', () => {
  it('should default to true when availability is null', async () => {
    mockRepository.createQueryBuilder().getMany.mockResolvedValue([
      { id: '1', availability: null, isActive: true },
    ]);

    const result = await service.getMenuItems('chef-001');

    expect(result[0].availability).toBe(true);
  });

  it('should default to true when availability.isAvailable is undefined', async () => {
    mockRepository.createQueryBuilder().getMany.mockResolvedValue([
      { id: '1', availability: {}, isActive: true },
    ]);

    const result = await service.getMenuItems('chef-001');

    expect(result[0].availability).toBe(true);
  });

  it('should respect false availability', async () => {
    mockRepository.createQueryBuilder().getMany.mockResolvedValue([
      { id: '1', availability: { isAvailable: false }, isActive: true },
    ]);

    const result = await service.getMenuItems('chef-001');

    expect(result[0].availability).toBe(false);
  });
});
```

---

### **TC-CHEF-SVC-004: Image URL Fallback Logic**

**Objective**: Verify `imageUrl || thumbnailUrl || undefined` chain

**Test Cases**:
```typescript
describe('Image URL fallback', () => {
  it('should return imageUrl when available', async () => {
    mockRepository.createQueryBuilder().getMany.mockResolvedValue([
      { id: '1', imageUrl: 'https://full.jpg', thumbnailUrl: 'https://thumb.jpg' },
    ]);

    const result = await service.getMenuItems('chef-001');

    expect(result[0].imageUrl).toBe('https://full.jpg');
  });

  it('should fallback to thumbnailUrl when imageUrl is null', async () => {
    mockRepository.createQueryBuilder().getMany.mockResolvedValue([
      { id: '1', imageUrl: null, thumbnailUrl: 'https://thumb.jpg' },
    ]);

    const result = await service.getMenuItems('chef-001');

    expect(result[0].imageUrl).toBe('https://thumb.jpg');
  });

  it('should return undefined when both are null', async () => {
    mockRepository.createQueryBuilder().getMany.mockResolvedValue([
      { id: '1', imageUrl: null, thumbnailUrl: null },
    ]);

    const result = await service.getMenuItems('chef-001');

    expect(result[0].imageUrl).toBeUndefined();
  });
});
```

---

### **TC-CHEF-SVC-005: Debug Logging (Total vs Active Items)**

**Objective**: Verify service logs item counts for debugging

**Test Code**:
```typescript
it('should log total and active item counts', async () => {
  const logSpy = jest.spyOn(service['logger'], 'log');

  mockRepository.find.mockResolvedValue([
    { id: '1', isActive: true },
    { id: '2', isActive: false },
    { id: '3', isActive: true },
  ]);

  mockRepository.createQueryBuilder().getMany.mockResolvedValue([
    { id: '1', isActive: true },
    { id: '3', isActive: true },
  ]);

  await service.getMenuItems('chef-001');

  expect(logSpy).toHaveBeenCalledWith('Total items for chef chef-001: 3');
  expect(logSpy).toHaveBeenCalledWith('Found 2 active menu items for chef chef-001');
});
```

**Verification**:
- ✅ Total items logged (including inactive)
- ✅ Active items count logged
- ✅ Helps debug "missing items" support tickets

---

### **TC-CHEF-SVC-006: Error Handling (Database Failure)**

**Objective**: Verify service throws `MENU_FETCH_FAILED` on database error

**Test Code**:
```typescript
it('should throw MENU_FETCH_FAILED on database error', async () => {
  mockRepository.find.mockRejectedValue(new Error('Connection lost'));

  await expect(service.getMenuItems('chef-001')).rejects.toThrow('MENU_FETCH_FAILED');
});

it('should log error details on failure', async () => {
  const errorSpy = jest.spyOn(service['logger'], 'error');
  mockRepository.find.mockRejectedValue(new Error('Database timeout'));

  try {
    await service.getMenuItems('chef-001');
  } catch {}

  expect(errorSpy).toHaveBeenCalledWith(
    expect.stringContaining('Failed to fetch menu items for chef chef-001'),
    expect.any(String) // Stack trace
  );
});
```

---

### **TC-CHEF-SVC-007: Empty Result Handling**

**Objective**: Verify service returns empty array (not null) when no items found

**Test Code**:
```typescript
it('should return empty array for chef with no items', async () => {
  mockRepository.find.mockResolvedValue([]);
  mockRepository.createQueryBuilder().getMany.mockResolvedValue([]);

  const result = await service.getMenuItems('chef-002');

  expect(result).toEqual([]);
  expect(Array.isArray(result)).toBe(true);
});
```

---

### **TC-CHEF-SVC-008: DTO Mapping Completeness**

**Objective**: Verify all LinkableProductDto fields populated correctly

**Test Code**:
```typescript
it('should map all LinkableProductDto fields', async () => {
  mockRepository.createQueryBuilder().getMany.mockResolvedValue([
    {
      id: 'item-123',
      name: 'Test Item',
      price: 100,
      chefId: 'chef-001',
      availability: { isAvailable: true },
      imageUrl: 'https://test.jpg',
    },
  ]);

  const result = await service.getMenuItems('chef-001');

  expect(result[0]).toEqual({
    id: 'item-123',
    name: 'Test Item',
    price: 10000, // 100 × 100
    chefId: 'chef-001',
    availability: true,
    imageUrl: 'https://test.jpg',
  });

  // Verify no extra fields
  expect(Object.keys(result[0])).toHaveLength(6);
});
```

---

### **TC-CHEF-SVC-009: Query Builder Parameters**

**Objective**: Verify correct SQL query generated with parameters

**Test Code**:
```typescript
it('should build query with correct parameters', async () => {
  const queryBuilder = {
    where: jest.fn().mockReturnThis(),
    andWhere: jest.fn().mockReturnThis(),
    orderBy: jest.fn().mockReturnThis(),
    getMany: jest.fn().mockResolvedValue([]),
  };

  mockRepository.createQueryBuilder.mockReturnValue(queryBuilder);

  await service.getMenuItems('chef-123');

  expect(queryBuilder.where).toHaveBeenCalledWith('item.chefId = :chefId', { chefId: 'chef-123' });
  expect(queryBuilder.andWhere).toHaveBeenCalledWith('item.isActive = :isActive', { isActive: true });
  expect(queryBuilder.orderBy).toHaveBeenCalledWith('item.createdAt', 'DESC');
});
```

---

### **TC-CHEF-SVC-010: Repository Injection**

**Objective**: Verify ChefMenuItem repository correctly injected

**Test Code**:
```typescript
it('should inject ChefMenuItem repository', () => {
  expect(service['chefMenuItemRepository']).toBeDefined();
  expect(service['chefMenuItemRepository']).toHaveProperty('find');
  expect(service['chefMenuItemRepository']).toHaveProperty('createQueryBuilder');
});
```

---

## 🔗 **Integration Tests**

### **TC-CHEF-INT-001: Media Module - Product Tagging Flow**

**Objective**: Verify Media Module can fetch and use menu items for tagging

**Test Steps**:
1. Chef uploads video
2. Frontend fetches chef's menu via `GET /api/v1/chefs/:id/menu`
3. Chef selects "Butter Chicken" from dropdown
4. Submit reel with `linkedProducts: [{ id: "item-001" }]`
5. Verify reel saved with product link

**Expected Result**:
```json
// Reel response
{
  "id": "reel-123",
  "linkedProducts": [
    {
      "id": "item-001",
      "name": "Butter Chicken",
      "price": 35000
    }
  ]
}
```

**Verification**:
- ✅ Menu items fetched successfully
- ✅ Product ID correctly linked to reel
- ✅ Reel displays product tag in feed

---

### **TC-CHEF-INT-002: Cart Module - Item Validation**

**Objective**: Verify Cart Module validates item availability using Chef Module

**Test Steps**:
1. Customer adds "Paneer Tikka" to cart
2. Cart Module calls `ChefService.getMenuItems(chefId)`
3. Cart validates item exists and is available
4. Item added to cart with validated price

**Expected Result**:
```json
// Cart item
{
  "itemId": "item-002",
  "price": 28000,  // Price from Chef Module (validated)
  "quantity": 1
}
```

**Verification**:
- ✅ Cart uses Chef Module for validation (not direct DB query)
- ✅ Price matches Chef Module response
- ✅ Unavailable items rejected with error

---

### **TC-CHEF-INT-003: Feed Module - Product Display**

**Objective**: Verify Feed Module displays product tags from Chef Module data

**Test Steps**:
1. User scrolls feed
2. Reel with linked product appears
3. Frontend fetches product details from Chef Module
4. Product chip displayed on reel

**Expected Result**:
```tsx
<ProductChip
  name="Butter Chicken"
  price="₹350"  // Converted from 35000 paise
  imageUrl="https://cdn.chefooz.com/butter-chicken.jpg"
/>
```

**Verification**:
- ✅ Product data fetched from Chef Module
- ✅ Price correctly displayed (paise → rupees)
- ✅ Image fallback handled (placeholder if null)

---

### **TC-CHEF-INT-004: Cross-Module Dependency (Chef-Kitchen)**

**Objective**: Verify Chef Module reads from Chef-Kitchen entity without issues

**Test Steps**:
1. Chef adds new menu item via Chef-Kitchen Module
2. Verify item visible in Chef Module response
3. Chef updates item (mark inactive)
4. Verify item no longer in Chef Module response

**Expected Behavior**:
```mermaid
sequenceDiagram
    participant CK as Chef-Kitchen Module
    participant DB as PostgreSQL
    participant CM as Chef Module

    CK->>DB: INSERT menu item (active)
    CM->>DB: SELECT active items
    DB-->>CM: Return new item ✅
    
    CK->>DB: UPDATE item (inactive)
    CM->>DB: SELECT active items
    DB-->>CM: Exclude updated item ✅
```

**Verification**:
- ✅ Real-time updates reflected in Chef Module
- ✅ No caching issues (future: invalidate cache on updates)
- ✅ Cross-module entity sharing works correctly

---

### **TC-CHEF-INT-005: Frontend API Client - useChefMenu Hook**

**Objective**: Verify React Query hook integrates correctly with Chef API

**Test Code** (Mobile App):
```typescript
// libs/api-client/src/lib/chef/chef.hooks.ts
import { useQuery } from '@tanstack/react-query';
import { chefClient } from './chef.client';

export const useChefMenu = (chefId: string) => {
  return useQuery({
    queryKey: ['chef', 'menu', chefId],
    queryFn: () => chefClient.getMenu(chefId),
    staleTime: 5 * 60 * 1000, // 5 minutes
  });
};
```

**Test Steps**:
1. Component calls `useChefMenu('chef-001')`
2. Hook triggers API request
3. Verify data returned and cached

**Verification**:
- ✅ API request successful
- ✅ Data cached for 5 minutes
- ✅ Re-renders on data change

---

### **TC-CHEF-INT-006: E2E - Chef to Customer Flow**

**Objective**: End-to-end test from chef menu update to customer view

**Test Steps**:
1. **Chef**: Create menu item "Chicken Biryani" (₹450)
2. **Chef**: Mark item as active
3. **Chef**: Upload reel, tag "Chicken Biryani"
4. **Customer**: View reel in feed
5. **Customer**: See product tag, tap to add to cart
6. **Customer**: Verify price matches (₹450)

**Expected Result**:
- ✅ Menu item visible in product dropdown (step 3)
- ✅ Product tag appears on reel (step 4)
- ✅ Correct price in cart (step 6)
- ✅ No data inconsistencies across modules

---

### **TC-CHEF-INT-007: Database Index Performance**

**Objective**: Verify compound index (`chef_id, is_active`) improves query performance

**Test Steps**:
1. Insert 1000 menu items for `chef-004` (500 active, 500 inactive)
2. Run query: `SELECT * FROM chef_menu_items WHERE chef_id = 'chef-004' AND is_active = true`
3. Analyze execution plan

**Expected Result**:
```sql
EXPLAIN ANALYZE
SELECT * FROM chef_menu_items WHERE chef_id = 'chef-004' AND is_active = true;

-- Expected plan:
-- Index Scan using idx_chef_menu_items_chef_active
-- Execution Time: < 5ms
```

**Verification**:
- ✅ Query uses compound index (not sequential scan)
- ✅ Execution time < 10ms for 1000 items
- ✅ No index merge or extra sorts

---

### **TC-CHEF-INT-008: Swagger Documentation Validation**

**Objective**: Verify API documented correctly in Swagger UI

**Test Steps**:
1. Navigate to `https://api.chefooz.com/api/docs`
2. Find `GET /api/v1/chefs/:chefId/menu`
3. Verify documentation accuracy

**Expected Documentation**:
- ✅ Endpoint path correct
- ✅ Parameter `chefId` documented as UUID
- ✅ Response schema shows `LinkableProductDto[]`
- ✅ Example response matches actual API
- ✅ Error responses documented (400, 500)

---

## ⚡ **Performance Tests**

### **TC-CHEF-PERF-001: Response Time (Single Chef)**

**Objective**: Verify API response time meets SLA (< 200ms p95)

**Test Setup**:
- Chef with 10 active menu items
- 100 concurrent requests

**Test Execution** (Apache Bench):
```bash
ab -n 1000 -c 100 https://api.chefooz.com/api/v1/chefs/chef-001/menu
```

**Expected Metrics**:
| Metric | Target | Result |
|--------|--------|--------|
| **Mean Response Time** | < 150ms | ___ ms |
| **p95 Response Time** | < 200ms | ___ ms |
| **p99 Response Time** | < 300ms | ___ ms |
| **Throughput** | > 500 req/s | ___ req/s |
| **Error Rate** | < 0.5% | ___ % |

**Verification**:
- ✅ p95 response time < 200ms
- ✅ No database connection pool exhaustion
- ✅ No memory leaks during load test

---

### **TC-CHEF-PERF-002: Large Menu Handling**

**Objective**: Verify performance with chef having 100+ menu items

**Test Setup**:
- Insert 100 active menu items for `chef-004`
- Measure response time and payload size

**Expected Result**:
```json
{
  "data": [
    // 100 items...
  ]
}
```

**Verification**:
- ✅ Response time < 300ms (larger payload)
- ✅ Payload size ~30 KB (acceptable for mobile)
- ✅ All items returned (no pagination needed yet)
- ✅ No timeout errors

**Recommendation**: Implement pagination if average menu exceeds 50 items

---

### **TC-CHEF-PERF-003: Database Query Performance**

**Objective**: Measure database query execution time

**Test Setup**:
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM chef_menu_items
WHERE chef_id = 'chef-001' AND is_active = true
ORDER BY created_at DESC;
```

**Expected Result**:
```
Index Scan using idx_chef_menu_items_chef_active
  -> Execution Time: 0.234 ms
  -> Shared Buffers Hit: 4 (no disk I/O)
```

**Verification**:
- ✅ Execution time < 5ms
- ✅ Index used (no sequential scan)
- ✅ All data from buffer cache (no disk reads)

---

### **TC-CHEF-PERF-004: Concurrent Requests (High Load)**

**Objective**: Verify API handles 1000 concurrent requests

**Test Execution**:
```bash
# 10,000 requests with 1000 concurrent connections
ab -n 10000 -c 1000 https://api.chefooz.com/api/v1/chefs/chef-001/menu
```

**Expected Metrics**:
| Metric | Target | Result |
|--------|--------|--------|
| **Successful Requests** | 100% | ___ % |
| **Failed Requests** | 0 | ___ |
| **Connection Errors** | 0 | ___ |
| **Mean Response Time** | < 250ms | ___ ms |

**Verification**:
- ✅ No connection pool exhaustion
- ✅ No `EMFILE` errors (file descriptor limits)
- ✅ Database maintains stable latency

---

### **TC-CHEF-PERF-005: Memory Usage (Sustained Load)**

**Objective**: Verify no memory leaks under sustained load

**Test Steps**:
1. Monitor baseline memory usage
2. Run load test (1 hour, 1000 req/min)
3. Compare memory usage before/after

**Expected Result**:
```
Baseline Memory: 150 MB
After 1 Hour:    152 MB (< 5% increase)
```

**Verification**:
- ✅ Memory usage stable (no linear growth)
- ✅ Garbage collection functioning correctly
- ✅ No memory leaks in service/repository layers

---

## 🔒 **Security Tests**

### **TC-CHEF-SEC-001: SQL Injection Prevention**

**Objective**: Verify parameterized queries prevent SQL injection

**Test Steps**:
```http
# Attempt SQL injection via chefId parameter
GET /api/v1/chefs/'; DROP TABLE chef_menu_items; --/menu HTTP/1.1
```

**Expected Result**:
```json
{
  "success": false,
  "message": "Validation failed",
  "errors": [{ "field": "chefId", "message": "chefId must be a valid UUID" }],
  "statusCode": 400
}
```

**Verification**:
- ✅ Request rejected at validation layer (UUID format)
- ✅ No SQL executed (logs confirm no query)
- ✅ Database tables intact

---

### **TC-CHEF-SEC-002: Data Exposure Prevention**

**Objective**: Verify no sensitive data exposed in responses

**Test Steps**:
1. Fetch menu for chef
2. Inspect response for sensitive fields

**Fields to Check**:
- ❌ Chef email/phone
- ❌ Chef address (unless explicitly public)
- ❌ Internal database IDs (user IDs acceptable)
- ❌ Deleted/archived items

**Expected Result**:
```json
{
  "data": [
    {
      "id": "item-001",  // ✅ Public UUID
      "name": "Butter Chicken",  // ✅ Public info
      "price": 35000,  // ✅ Public info
      "chefId": "chef-001",  // ✅ Public UUID
      // ❌ No chef email, phone, etc.
    }
  ]
}
```

**Verification**:
- ✅ Only whitelisted fields returned (LinkableProductDto)
- ✅ No database metadata exposed
- ✅ No internal error details in production

---

### **TC-CHEF-SEC-003: Rate Limiting (DDoS Protection)**

**Objective**: Verify rate limiting prevents abuse of public endpoint

**Test Steps**:
1. Send 1000 requests from single IP in 10 seconds
2. Verify rate limit triggered

**Expected Result** (After 100 requests):
```json
{
  "success": false,
  "message": "Rate limit exceeded. Try again in 60 seconds.",
  "errorCode": "RATE_LIMIT_EXCEEDED",
  "statusCode": 429
}
```

**Rate Limit Configuration**:
```typescript
// Recommended: 100 requests per minute per IP
@UseGuards(ThrottlerGuard)
@Throttle(100, 60)
@Get(':chefId/menu')
async getMenu() { ... }
```

**Verification**:
- ✅ Rate limit enforced after 100 requests
- ✅ Legitimate users not affected (normal usage < 10 req/min)
- ✅ Rate limit resets after 60 seconds

---

## 🧩 **Edge Case Tests**

### **TC-CHEF-EDGE-001: Extremely Long Item Names**

**Objective**: Verify handling of item names at VARCHAR(255) limit

**Test Steps**:
1. Insert item with 255-character name
2. Fetch menu
3. Verify name not truncated in response

**Expected Result**:
```json
{
  "data": [
    {
      "name": "This is a very long item name that exactly reaches the 255 character limit set by the database VARCHAR constraint and should be returned in full without any truncation issues that might occur in poorly designed systems that do not properly handle maximum field lengths..."
    }
  ]
}
```

**Verification**:
- ✅ Full name returned (no truncation)
- ✅ No database errors
- ✅ JSON response valid

---

### **TC-CHEF-EDGE-002: Zero-Price Items**

**Objective**: Verify handling of free items (price = 0)

**Test Steps**:
1. Insert item with `price: 0.00`
2. Fetch menu

**Expected Result**:
```json
{
  "data": [
    {
      "name": "Free Papad",
      "price": 0  // 0 × 100 = 0 paise
    }
  ]
}
```

**Verification**:
- ✅ Price conversion works correctly (0 → 0)
- ✅ Item included in response (not filtered out)
- ✅ No division-by-zero errors

---

### **TC-CHEF-EDGE-003: Special Characters in Item Names**

**Objective**: Verify handling of Unicode and special characters

**Test Steps**:
1. Insert items with names:
   - `"Paneer Tikka 🔥"`
   - `"Dal Makhani (दाल मखनी)"`
   - `"Chicken 65 <script>alert('xss')</script>"`
2. Fetch menu

**Expected Result**:
```json
{
  "data": [
    { "name": "Paneer Tikka 🔥" },  // ✅ Emoji preserved
    { "name": "Dal Makhani (दाल मखनी)" },  // ✅ Hindi preserved
    { "name": "Chicken 65 &lt;script&gt;alert('xss')&lt;/script&gt;" }  // ✅ HTML escaped
  ]
}
```

**Verification**:
- ✅ Unicode characters preserved
- ✅ HTML/JS escaped (XSS prevention)
- ✅ JSON response valid

---

### **TC-CHEF-EDGE-004: Concurrent Menu Updates**

**Objective**: Verify race condition handling when chef updates menu during API call

**Test Steps**:
1. Start API request for menu
2. Mid-request: Chef updates item (marks inactive)
3. Verify response consistency

**Expected Behavior**:
- ✅ Either includes updated item (if query before update)
- ✅ Or excludes updated item (if query after update)
- ❌ No partial/corrupted data
- ❌ No database lock timeouts

**Verification**:
- ✅ Transaction isolation level appropriate
- ✅ No dirty reads
- ✅ Response time not significantly impacted

---

### **TC-CHEF-EDGE-005: Malformed JSONB Availability Field**

**Objective**: Verify handling of corrupted JSONB data

**Test Steps**:
1. Manually insert item with malformed JSONB:
   ```sql
   INSERT INTO chef_menu_items (availability) VALUES ('{"malformed": }');
   ```
2. Attempt to fetch menu

**Expected Result**:
- ✅ Service handles gracefully (defaults to `availability: true`)
- ✅ Or logs error and excludes item (no crash)
- ❌ No 500 error for entire request

**Verification**:
- ✅ Fallback logic robust
- ✅ Error logged for investigation
- ✅ Other items still returned

---

### **TC-CHEF-EDGE-006: Time Zone Handling (createdAt Sorting)**

**Objective**: Verify sorting works correctly across time zones

**Test Steps**:
1. Insert items with timestamps:
   - `2026-02-01 10:00:00+00:00` (UTC)
   - `2026-02-01 15:30:00+05:30` (IST, equivalent to 10:00 UTC)
   - `2026-02-01 11:00:00+00:00` (UTC)
2. Fetch menu

**Expected Order** (DESC):
1. Item C (11:00 UTC)
2. Item A & B (10:00 UTC, same time, secondary sort by ID)

**Verification**:
- ✅ Timestamps compared in UTC (normalized)
- ✅ No time zone conversion errors
- ✅ Consistent sorting across server restarts

---

## 🔄 **Regression Tests**

### **TC-CHEF-REG-001: Price Conversion Regression**

**Objective**: Verify price conversion not broken by future updates

**Test Steps**:
1. Insert item: `price: 350.00`
2. Fetch menu
3. Verify: `price: 35000` (paise)

**Baseline Result**: ✅ Pass (35000 paise)

**Run After**:
- ✅ Backend code changes
- ✅ Database migrations
- ✅ TypeORM version updates

---

### **TC-CHEF-REG-002: Image Fallback Regression**

**Objective**: Verify image fallback logic not broken

**Test Steps**:
1. Insert item: `imageUrl: null, thumbnailUrl: 'https://...'`
2. Fetch menu
3. Verify: `imageUrl: 'https://...'` (fallback applied)

**Baseline Result**: ✅ Pass (thumbnailUrl returned)

---

### **TC-CHEF-REG-003: Inactive Filter Regression**

**Objective**: Verify inactive items always excluded

**Test Steps**:
1. Insert 2 items: 1 active, 1 inactive
2. Fetch menu
3. Verify: Only 1 item returned

**Baseline Result**: ✅ Pass (1 item)

---

### **TC-CHEF-REG-004: Public Access Regression**

**Objective**: Verify endpoint remains public (no accidental auth added)

**Test Steps**:
1. Send request without JWT token
2. Verify: 200 OK (not 401)

**Baseline Result**: ✅ Pass (200 OK)

**Critical**: Run after security updates or auth middleware changes

---

## 📊 **Test Summary**

### **Test Execution Report Template**

```
═══════════════════════════════════════════════════════════════
                 CHEF MODULE - QA TEST REPORT
═══════════════════════════════════════════════════════════════

Test Execution Date: [YYYY-MM-DD]
Environment: [Staging/Production]
Tested By: [QA Engineer Name]

─────────────────────────────────────────────────────────────────
CATEGORY                    | TOTAL | PASS | FAIL | SKIP | %
─────────────────────────────────────────────────────────────────
API Endpoint Tests          |   12  |  __  |  __  |  __  | ___%
Service Layer Tests         |   10  |  __  |  __  |  __  | ___%
Integration Tests           |    8  |  __  |  __  |  __  | ___%
Performance Tests           |    5  |  __  |  __  |  __  | ___%
Security Tests              |    3  |  __  |  __  |  __  | ___%
Edge Case Tests             |    6  |  __  |  __  |  __  | ___%
Regression Tests            |    4  |  __  |  __  |  __  | ___%
─────────────────────────────────────────────────────────────────
TOTAL                       |   48  |  __  |  __  |  __  | ___%
─────────────────────────────────────────────────────────────────

═══════════════════════════════════════════════════════════════
CRITICAL ISSUES FOUND: [Count]
─────────────────────────────────────────────────────────────────
[List critical issues here]

═══════════════════════════════════════════════════════════════
RECOMMENDATIONS:
─────────────────────────────────────────────────────────────────
[List recommendations here]

═══════════════════════════════════════════════════════════════
SIGN-OFF:
─────────────────────────────────────────────────────────────────
QA Lead: _________________    Date: __________
Dev Lead: ________________    Date: __________
═══════════════════════════════════════════════════════════════
```

---

## ✅ **Completion Criteria**

### **Module Ready for Production When**:
- ✅ All 48 test cases executed
- ✅ 100% pass rate for Critical tests (API, Service, Security)
- ✅ >95% pass rate for High priority tests (Integration, Performance)
- ✅ All P0 bugs resolved
- ✅ Performance metrics meet SLA (p95 < 200ms)
- ✅ Security audit passed (no SQL injection, XSS, data exposure)
- ✅ Load testing completed (1000 concurrent users)
- ✅ Documentation reviewed and approved

---

## 🛡️ **Compliance / KYC — Regression Tests** *(Added March 2026)*

### TC-COMPLIANCE-001: req.user.id bug regression

**Type:** Bug Regression / Automated  
**Feature area:** All compliance endpoints  
**Priority:** P0

**Preconditions:**
- Authenticated chef JWT in request headers

**Steps:**
1. Call any `POST /v1/chef/compliance/*` endpoint with a valid chef JWT

**Expected result:** Chef entity resolved; step saved with correct `chefId`  
**Actual result (before fix):** `Chef undefined` — `req.user.userId` did not exist; all mutations silently failed  
**Fix applied:** Changed all 6 controller methods from `req.user.userId` to `req.user.id`  
**Regression test:** `apps/chefooz-apis/src/modules/chef-compliance/chef-compliance.controller.spec.ts`  
**Status:** Fixed ✅

---

### TC-COMPLIANCE-002: Step 1 not marked complete after submit

**Type:** Bug Regression / Manual  
**Feature area:** compliance/index.tsx checklist  
**Priority:** P1

**Preconditions:**
- Chef is on `compliance/index.tsx`

**Steps:**
1. Enter Step 1 (identity), fill and submit
2. Navigate back to index

**Expected result:** Step 1 card shows green checkmark  
**Actual result (before fix):** Step 1 card still shows unchecked — `setQueryData` was used in hook `onSuccess`, which does not trigger re-render from server  
**Fix applied:** Changed all 4 mutation hooks to use `invalidateQueries` instead of `setQueryData`  
**Regression test:** Manual — confirm checkmark appears after submitting each step  
**Status:** Fixed ✅

---

### TC-COMPLIANCE-003: Identity screen loses state on re-entry

**Type:** Bug Regression / Manual  
**Feature area:** identity.tsx  
**Priority:** P1

**Preconditions:**
- Chef has previously submitted Step 1

**Steps:**
1. Submit Step 1
2. Navigate back to index
3. Tap Step 1 again

**Expected result:** Document type, document number, and upload slots pre-populated from server data  

---

### TC-COMPLIANCE-004: Admin can review pending KYC steps from Chefs list

**Type:** Manual / Automated  
**Feature area:** Admin Portal → Users → Chefs → KYC Review  
**Priority:** P0

**Preconditions:**
- Admin user is logged into the admin portal
- A chef has submitted at least one of identity, bank, or FSSAI

**Steps:**
1. Open `Users → Chefs`
2. Click the action icon in the chef row
3. Verify the KYC review page opens for the selected chef
4. Confirm pending steps are clearly visible in the overall status and step cards
5. Verify or reject one pending step
6. Refresh the page and confirm the updated status persists

**Expected result:**
- The page shows document links, step-by-step pending/verified status, and payout eligibility
- Submitted documents remain in review until an admin explicitly verifies them
- Verify/reject actions update the selected step only
- Rejections require a reason and show that reason after save

**Actual result (before fix):**
- The action icon opened the payouts page instead of a compliance review surface
- Admins had no way to inspect uploaded KYC documents or clear pending steps

**Fix applied:**
- Added `/dashboard/chefs/[chefId]/compliance`
- Added admin compliance endpoints and React Query hooks
- Rewired the chefs table action to the KYC review page

**Regression test:** `apps/chefooz-apis/src/modules/chef-compliance/chef-compliance.service.spec.ts`  
**Status:** Fixed ✅

---

### TC-COMPLIANCE-005: Chef can be verified for operations without bank review

**Type:** Manual / Automated  
**Feature area:** Admin Portal → Users → Chefs → KYC Review  
**Priority:** P0

**Preconditions:**
- Identity and FSSAI are admin-approved
- Legal terms are accepted
- Bank details may still be pending review

**Steps:**
1. Open the chef's KYC review page
2. Verify identity and FSSAI
3. Leave bank unverified
4. Click `Mark Chef Verified`

**Expected result:**
- Chef is marked verified for operations
- Chef can go online and accept orders
- Payout remains disabled until bank review + payout enable action happen

**Fix applied:** Split final admin actions into chef operational verification and payout enablement  
**Regression test:** `apps/chefooz-apis/src/modules/chef-compliance/chef-compliance.service.spec.ts`  
**Status:** Fixed ✅

---

### TC-COMPLIANCE-006: Payout stays disabled until bank review and explicit enablement

**Type:** Manual / Automated  
**Feature area:** Admin Portal → Users → Chefs → KYC Review  
**Priority:** P0

**Preconditions:**
- Chef is already verified for operations
- Bank details are submitted but not admin-approved

**Steps:**
1. Open the chef's KYC review page
2. Confirm `Enable Payout` is disabled while bank review is pending
3. Verify the bank details
4. Click `Enable Payout`

**Expected result:**
- Payout cannot be enabled before bank review
- Payout becomes enabled only after explicit admin action

**Fix applied:** Added `payout_enabled` final admin-controlled switch and payout eligibility rules  
**Regression test:** `apps/chefooz-apis/src/modules/chef-compliance/chef-compliance.service.spec.ts`  
**Status:** Fixed ✅
**Actual result (before fix):** All fields blank on re-entry  
**Fix applied:** Added `useEffect` with `initialized` guard seeding state from `existingCompliance.identityData`  
**Regression test:** Manual  
**Status:** Fixed ✅

---

### TC-COMPLIANCE-004: Bank account confirm number — real-time match

**Type:** Manual / QA  
**Feature area:** bank.tsx  
**Priority:** P1

**Preconditions:**
- Chef on Step 2 (bank.tsx), Bank Transfer mode selected

**Steps:**
1. Enter account number: `12345678901`
2. Type confirm account number: `12345678900` (one digit wrong)
3. Observe inline indicator

**Expected result:** Red "do not match" indicator shown in real-time below confirm field  
**Actual result (before fix):** No real-time indicator — error only shown after submit  
**Fix applied:** `accountsMatch = useMemo(...)` computing `null | true | false`; inline `Ionicons` indicator rendered based on value  
**Status:** Fixed ✅

---

### TC-COMPLIANCE-005: Bank mode toggle changes required fields

**Type:** Manual / QA  
**Feature area:** bank.tsx  
**Priority:** P1

**Preconditions:**
- Chef on Step 2 (bank.tsx)

**Steps:**
1. Select **UPI Transfer** mode
2. Observe Section 2 (Bank Account Details) header

**Expected result:** "Optional" pill shown on bank section; UPI field marked as required  
**Actual result (before fix):** Both sections always appeared required regardless of mode  
**Fix applied:** `bankRequired = preferredMode === 'bank'`; conditional "Optional" pill rendered; validation skips unprovided bank fields when `preferredMode === 'upi'`  
**Status:** Fixed ✅

---

### TC-COMPLIANCE-006: IFSC auto-lookup

**Type:** Manual / QA  
**Feature area:** bank.tsx  
**Priority:** P2

**Preconditions:**
- Chef on Step 2 (bank.tsx), Bank Transfer mode selected

**Steps:**
1. Type a valid 11-character IFSC code (e.g. `HDFC0001234`)
2. Wait for lookup

**Expected result:** `bankName` and `branchName` fields auto-populated; green check icon shown on IFSC field  
**Steps on failure:** Amber warning icon shown; manual entry still possible  
**Fix applied:** `useEffect` fires on `ifscCode` length 11 + regex match; fetches Razorpay IFSC API; `AbortController` via `useRef` cancels stale requests  
**Status:** Fixed ✅

---

### TC-COMPLIANCE-007: FSSAI expiry date validation

**Type:** Manual / QA  
**Feature area:** fssai.tsx  
**Priority:** P1

**Preconditions:**
- Chef on Step 3 (FSSAI)

**Steps:**
1. Enter a past date (e.g. 01/01/2020) in expiry field
2. Tap submit

**Expected result:** Inline error: "Expiry date must be in the future"  
**Status:** Fixed ✅

---

### TC-COMPLIANCE-008: Legal scroll-to-accept gate

**Type:** Manual / QA  
**Feature area:** legal.tsx  
**Priority:** P2

**Preconditions:**
- Chef on Step 4 (legal.tsx), has NOT previously accepted

**Steps:**
1. Without scrolling, tap the accept checkbox

**Expected result:** Screen auto-scrolls to bottom; checkbox remains unchecked until scroll threshold reached  
**Fix applied:** `hasScrolledToBottom` state; `ScrollView.scrollToEnd()` called on premature tap  
**Status:** Fixed ✅

---

**[QA_TEST_CASES_COMPLETE ✅]**

*This document covers comprehensive testing for the Chef Module. For feature overview, see `01_FEATURE_OVERVIEW.md`. For technical implementation details, see `02_TECHNICAL_GUIDE.md`.*

---

**Document Version**: 1.1  
**Last Updated**: March 2026  
**Test Coverage**: 56 test cases (48 original + 8 compliance regression)  
**Automation Coverage**: ~85%  
**Next Review**: Post-QA sign-off (Q2 2026)
