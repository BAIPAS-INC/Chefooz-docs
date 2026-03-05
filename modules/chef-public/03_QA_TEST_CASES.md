# 🧪 Chef-Public Module - QA Test Cases

## 📋 **Table of Contents**
- [Test Environment Setup](#test-environment-setup)
- [Test Data Preparation](#test-data-preparation)
- [Chef Profile Tests](#chef-profile-tests)
- [Menu Tests](#menu-tests)
- [Reels Tests](#reels-tests)
- [Reorder Preview Tests](#reorder-preview-tests)
- [Integration Tests](#integration-tests)
- [Performance Tests](#performance-tests)
- [Security Tests](#security-tests)
- [Automation Scripts](#automation-scripts)

---

## 🛠️ **Test Environment Setup**

### **Prerequisites**

1. **Backend Running** (NestJS on port 3000)
2. **PostgreSQL Running** (port 5432)
3. **MongoDB Running** (port 27017)
4. **Environment Variables**:
   ```bash
   DATABASE_URL=postgresql://postgres:password@localhost:5432/chefooz_test
   MONGODB_URI=mongodb://localhost:27017/chefooz_test
   CDN_URL=https://cdn.chefooz.com
   JWT_SECRET=test_secret_key_12345
   ```

### **Database Seeding**

```bash
# Seed test data
npm run db:seed:test

# Verify data
npm run db:verify
```

### **Expected Seeded Data**

| Entity | Count | Details |
|--------|-------|---------|
| Users (Chefs) | 5 | 3 with kitchen setup, 2 without |
| ChefKitchen | 3 | Various statuses (online/offline) |
| ChefMenuItem | 30+ | Across 5 categories, mixed veg/non-veg |
| PlatformCategory | 10 | Appetizers, Main Course, Desserts, etc. |
| Orders | 20+ | Mixed statuses (DELIVERED, PENDING, CANCELLED) |
| Reels | 15 | 10 active, 5 soft-deleted |

---

## 📊 **Test Data Preparation**

### **Test Chef Profiles**

```typescript
// Test Chef 1: Complete Setup
const CHEF_COMPLETE = {
  id: '550e8400-e29b-41d4-a716-446655440001',
  fullName: 'Chef John Doe',
  avatarUrl: 'https://storage.chefooz.com/avatars/john.jpg',
  kitchen: {
    kitchenName: 'The Golden Spoon',
    isOnline: true,
    deliveryRadiusKm: 5,
    acceptingOrders: true,
  },
  menuItems: 15, // 8 veg, 7 non-veg
  orders: 50, // DELIVERED orders
  reels: 8,
};

// Test Chef 2: Incomplete Setup (No Kitchen)
const CHEF_INCOMPLETE = {
  id: '550e8400-e29b-41d4-a716-446655440002',
  fullName: 'Jane Smith',
  avatarUrl: 'https://storage.chefooz.com/avatars/jane.jpg',
  kitchen: null, // No kitchen entity
  menuItems: 0,
  orders: 0,
  reels: 0,
};

// Test Chef 3: Kitchen Offline
const CHEF_OFFLINE = {
  id: '550e8400-e29b-41d4-a716-446655440003',
  fullName: 'Chef Mike Ross',
  avatarUrl: 'https://storage.chefooz.com/avatars/mike.jpg',
  kitchen: {
    kitchenName: 'Mike\'s Bistro',
    isOnline: false, // Offline
    deliveryRadiusKm: 3,
    acceptingOrders: false,
  },
  menuItems: 12,
  orders: 25,
  reels: 5,
};

// Test Chef 4: Veg Only
const CHEF_VEG_ONLY = {
  id: '550e8400-e29b-41d4-a716-446655440004',
  fullName: 'Chef Priya Sharma',
  avatarUrl: 'https://storage.chefooz.com/avatars/priya.jpg',
  kitchen: {
    kitchenName: 'Sattvic Kitchen',
    isOnline: true,
    deliveryRadiusKm: 7,
    acceptingOrders: true,
  },
  menuItems: 20, // All veg
  orders: 75,
  reels: 12,
};

// Test Chef 5: Non-Existent
const CHEF_NOT_FOUND = '550e8400-e29b-41d4-a716-446655440999';
```

---

## 🧑‍🍳 **Chef Profile Tests**

### **Test Case 1.1: Get Complete Chef Profile**

**ID**: `CHEF_PUB_PROFILE_001`

**Description**: Retrieve profile for chef with complete kitchen setup.

**Endpoint**: `GET /api/v1/chefs/:chefId/public`

**Test Data**: `CHEF_COMPLETE` (chef with full setup)

**Steps**:
```bash

# PowerShell Script
$chefId = "550e8400-e29b-41d4-a716-446655440001"
RESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/chefs/$chefId/public")

# Parse response
$data = ($response.Content | jq .)

# Assertions
echo "✅ Test Case 1.1: Get Complete Chef Profile"
echo "Status Code: $($response.StatusCode)" # Expected: 200
echo "Success: $($data.success)" # Expected: true
echo "Chef ID: $($data.data.chefId)" # Expected: chef ID
echo "Kitchen Name: $($data.data.kitchenName)" # Expected: "The Golden Spoon"
echo "Is Open: $($data.data.isOpen)" # Expected: true
echo "Veg Type: $($data.data.vegType)" # Expected: "both"
echo "Total Orders: $($data.data.totalOrders)" # Expected: 50
```

**Expected Response**:
```json
{
  "success": true,
  "message": "Chef profile retrieved successfully",
  "data": {
    "chefId": "550e8400-e29b-41d4-a716-446655440001",
    "kitchenName": "The Golden Spoon",
    "avatar": "https://storage.chefooz.com/avatars/john.jpg",
    "coverImage": null,
    "rating": 4.5,
    "vegType": "both",
    "isOpen": true,
    "deliveryRadiusKm": 5,
    "etaMinutes": 30,
    "description": null,
    "cuisines": [],
    "acceptingOrders": true,
    "totalOrders": 50
  }
}
```

**Validation Criteria**:
- ✅ Status code: 200
- ✅ `success`: true
- ✅ `data.chefId` matches request
- ✅ `data.kitchenName` present
- ✅ `data.isOpen` is boolean
- ✅ `data.vegType` is one of: "veg", "non-veg", "both"
- ✅ `data.totalOrders` ≥ 0
- ✅ `data.deliveryRadiusKm` > 0
- ✅ Response time < 500ms

---

### **Test Case 1.2: Get Minimal Profile (No Kitchen Setup)**

**ID**: `CHEF_PUB_PROFILE_002`

**Description**: Retrieve profile for chef without kitchen setup (graceful degradation).

**Endpoint**: `GET /api/v1/chefs/:chefId/public`

**Test Data**: `CHEF_INCOMPLETE` (chef without kitchen)

**Steps**:
```bash

$chefId = "550e8400-e29b-41d4-a716-446655440002"
RESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/chefs/$chefId/public")
$data = ($response.Content | jq .)

echo "✅ Test Case 1.2: Get Minimal Profile"
echo "Status Code: $($response.StatusCode)" # Expected: 200 (not 404)
echo "Kitchen Name: $($data.data.kitchenName)" # Expected: Chef's fullName
echo "Is Open: $($data.data.isOpen)" # Expected: false
echo "Total Orders: $($data.data.totalOrders)" # Expected: 0
echo "Description: $($data.data.description)" # Expected: "still setting up"
```

**Expected Response**:
```json
{
  "success": true,
  "message": "Chef profile retrieved successfully",
  "data": {
    "chefId": "550e8400-e29b-41d4-a716-446655440002",
    "kitchenName": "Jane Smith",
    "avatar": "https://storage.chefooz.com/avatars/jane.jpg",
    "coverImage": null,
    "rating": 0,
    "vegType": "both",
    "isOpen": false,
    "deliveryRadiusKm": 0,
    "etaMinutes": 0,
    "description": "This chef is still setting up their kitchen",
    "cuisines": [],
    "acceptingOrders": false,
    "totalOrders": 0
  }
}
```

**Validation Criteria**:
- ✅ Status code: 200 (not 404 - graceful degradation)
- ✅ `data.kitchenName` = chef's fullName (fallback)
- ✅ `data.isOpen` = false
- ✅ `data.totalOrders` = 0
- ✅ `data.deliveryRadiusKm` = 0
- ✅ `data.description` contains "still setting up"

---

### **Test Case 1.3: Chef Not Found (404)**

**ID**: `CHEF_PUB_PROFILE_003`

**Description**: Request profile for non-existent chef ID.

**Endpoint**: `GET /api/v1/chefs/:chefId/public`

**Test Data**: `CHEF_NOT_FOUND` (invalid UUID)

**Steps**:
```bash

$chefId = "550e8400-e29b-41d4-a716-446655440999"
try {
RESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/chefs/$chefId/public")
    echo "❌ Test Failed: Expected 404 error"
} catch {
    $statusCode = $_.Exception.Response.StatusCode.value__
    $errorBody = $_.ErrorDetails.Message | jq .
    
    echo "✅ Test Case 1.3: Chef Not Found"
    echo "Status Code: $statusCode" # Expected: 404
    echo "Error Code: $($errorBody.errorCode)" # Expected: "CHEF_NOT_FOUND"
}
```

**Expected Response**:
```json
{
  "statusCode": 404,
  "success": false,
  "message": "Chef not found",
  "errorCode": "CHEF_NOT_FOUND"
}
```

**Validation Criteria**:
- ✅ Status code: 404
- ✅ `errorCode`: "CHEF_NOT_FOUND"
- ✅ `success`: false

---

### **Test Case 1.4: VegType Calculation - Veg Only**

**ID**: `CHEF_PUB_PROFILE_004`

**Description**: Verify vegType calculation for chef with only veg items.

**Endpoint**: `GET /api/v1/chefs/:chefId/public`

**Test Data**: `CHEF_VEG_ONLY` (all menu items are veg)

**Steps**:
```bash

$chefId = "550e8400-e29b-41d4-a716-446655440004"
RESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/chefs/$chefId/public")
$data = ($response.Content | jq .)

echo "✅ Test Case 1.4: VegType Calculation - Veg Only"
echo "Veg Type: $($data.data.vegType)" # Expected: "veg"
```

**Validation Criteria**:
- ✅ `data.vegType` = "veg" (all menu items are veg)

---

### **Test Case 1.5: VegType Calculation - Mixed**

**ID**: `CHEF_PUB_PROFILE_005`

**Description**: Verify vegType calculation for chef with mixed veg/non-veg items.

**Endpoint**: `GET /api/v1/chefs/:chefId/public`

**Test Data**: `CHEF_COMPLETE` (8 veg + 7 non-veg items)

**Steps**:
```bash

$chefId = "550e8400-e29b-41d4-a716-446655440001"
RESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/chefs/$chefId/public")
$data = ($response.Content | jq .)

echo "✅ Test Case 1.5: VegType Calculation - Mixed"
echo "Veg Type: $($data.data.vegType)" # Expected: "both"
```

**Validation Criteria**:
- ✅ `data.vegType` = "both" (mixed veg and non-veg)

---

### **Test Case 1.6: Order Count Verification**

**ID**: `CHEF_PUB_PROFILE_006`

**Description**: Verify total orders count (DELIVERED orders only).

**Endpoint**: `GET /api/v1/chefs/:chefId/public`

**Test Data**: `CHEF_COMPLETE` (50 DELIVERED orders in database)

**Pre-Test Database Query**:
```sql
SELECT COUNT(DISTINCT ord.id) AS delivered_count
FROM "Order" ord
WHERE ord."chefId" = '550e8400-e29b-41d4-a716-446655440001'
  AND ord.status = 'DELIVERED'
  AND EXISTS (
    SELECT 1 FROM jsonb_array_elements(ord.items) AS item
    WHERE (item->>'menuItemId')::uuid IN (
      SELECT id FROM "ChefMenuItem" WHERE "chefId" = '550e8400-e29b-41d4-a716-446655440001'
    )
  );
-- Expected: 50
```

**Steps**:
```bash

$chefId = "550e8400-e29b-41d4-a716-446655440001"
RESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/chefs/$chefId/public")
$data = ($response.Content | jq .)

echo "✅ Test Case 1.6: Order Count Verification"
echo "Total Orders: $($data.data.totalOrders)" # Expected: 50
```

**Validation Criteria**:
- ✅ `data.totalOrders` = 50 (matches database count)
- ✅ Only DELIVERED orders counted (not PENDING/CANCELLED)

---

### **Test Case 1.7: Offline Kitchen Status**

**ID**: `CHEF_PUB_PROFILE_007`

**Description**: Verify profile shows offline status correctly.

**Endpoint**: `GET /api/v1/chefs/:chefId/public`

**Test Data**: `CHEF_OFFLINE` (kitchen isOnline = false)

**Steps**:
```bash

$chefId = "550e8400-e29b-41d4-a716-446655440003"
RESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/chefs/$chefId/public")
$data = ($response.Content | jq .)

echo "✅ Test Case 1.7: Offline Kitchen Status"
echo "Is Open: $($data.data.isOpen)" # Expected: false
echo "Accepting Orders: $($data.data.acceptingOrders)" # Expected: false
```

**Validation Criteria**:
- ✅ `data.isOpen` = false
- ✅ `data.acceptingOrders` = false

---

## 🍽️ **Menu Tests**

### **Test Case 2.1: Get Grouped Menu**

**ID**: `CHEF_PUB_MENU_001`

**Description**: Retrieve menu grouped by platform categories.

**Endpoint**: `GET /api/v1/chefs/:chefId/menu`

**Test Data**: `CHEF_COMPLETE` (15 menu items across 5 categories)

**Steps**:
```bash

$chefId = "550e8400-e29b-41d4-a716-446655440001"
RESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/chefs/$chefId/menu")
$data = ($response.Content | jq .)

echo "✅ Test Case 2.1: Get Grouped Menu"
echo "Total Items: $($data.data.totalItems)" # Expected: 15
echo "Categorized Groups: $($data.data.categorized.Count)" # Expected: 5
echo "Uncategorized Items: $($data.data.uncategorized.Count)" # Expected: 0

# Check category structure
foreach ($category in $data.data.categorized) {
    echo "Category: $($category.categoryName), Items: $($category.items.Count)"
}
```

**Expected Response Structure**:
```json
{
  "success": true,
  "message": "Menu retrieved successfully",
  "data": {
    "categorized": [
      {
        "categoryId": "cat-appetizers-001",
        "categoryName": "Appetizers",
        "items": [...]
      },
      {
        "categoryId": "cat-main-course-001",
        "categoryName": "Main Course",
        "items": [...]
      }
    ],
    "uncategorized": [],
    "totalItems": 15
  }
}
```

**Validation Criteria**:
- ✅ Status code: 200
- ✅ `data.totalItems` = 15
- ✅ `data.categorized` is array with 5 elements
- ✅ Each category has `categoryId`, `categoryName`, `items`
- ✅ All items have `basePricePaise` field

---

### **Test Case 2.2: Price Conversion (Rupees to Paise)**

**ID**: `CHEF_PUB_MENU_002`

**Description**: Verify price field conversion (price in rupees, basePricePaise in paise).

**Endpoint**: `GET /api/v1/chefs/:chefId/menu`

**Test Data**: Menu item with `price = 280.00` rupees

**Steps**:
```bash

$chefId = "550e8400-e29b-41d4-a716-446655440001"
RESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/chefs/$chefId/menu")
$data = ($response.Content | jq .)

# Find specific item
$item = $data.data.categorized[0].items[0]

echo "✅ Test Case 2.2: Price Conversion"
echo "Price (Rupees): $($item.price)" # Expected: 280.00
echo "Base Price (Paise): $($item.basePricePaise)" # Expected: 28000
```

**Validation Criteria**:
- ✅ `item.price` = 280.00 (rupees, DECIMAL field)
- ✅ `item.basePricePaise` = 28000 (paise, calculated)
- ✅ `basePricePaise = Math.round(price * 100)`

---

### **Test Case 2.3: Active Items Only**

**ID**: `CHEF_PUB_MENU_003`

**Description**: Verify only active items returned (isActive = true).

**Endpoint**: `GET /api/v1/chefs/:chefId/menu`

**Pre-Test Database Query**:
```sql
-- Total menu items (including inactive)
SELECT COUNT(*) FROM "ChefMenuItem" WHERE "chefId" = '550e8400-e29b-41d4-a716-446655440001';
-- Expected: 18 (15 active + 3 inactive)

-- Active items only
SELECT COUNT(*) FROM "ChefMenuItem" WHERE "chefId" = '550e8400-e29b-41d4-a716-446655440001' AND "isActive" = true;
-- Expected: 15
```

**Steps**:
```bash

$chefId = "550e8400-e29b-41d4-a716-446655440001"
RESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/chefs/$chefId/menu")
$data = ($response.Content | jq .)

echo "✅ Test Case 2.3: Active Items Only"
echo "Total Items: $($data.data.totalItems)" # Expected: 15 (not 18)

# Verify all items are active
foreach ($category in $data.data.categorized) {
    foreach ($item in $category.items) {
        if ($item.isActive -ne $true) {
            echo "❌ Found inactive item: $($item.id)"
        }
    }
}
```

**Validation Criteria**:
- ✅ `data.totalItems` = 15 (only active items)
- ✅ All returned items have `isActive = true`

---

### **Test Case 2.4: Category Name Resolution**

**ID**: `CHEF_PUB_MENU_004`

**Description**: Verify platform category names fetched correctly.

**Endpoint**: `GET /api/v1/chefs/:chefId/menu`

**Steps**:
```bash

$chefId = "550e8400-e29b-41d4-a716-446655440001"
RESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/chefs/$chefId/menu")
$data = ($response.Content | jq .)

echo "✅ Test Case 2.4: Category Name Resolution"
foreach ($category in $data.data.categorized) {
    echo "Category ID: $($category.categoryId), Name: $($category.categoryName)"
    # Expected: Real category names (not IDs)
}
```

**Validation Criteria**:
- ✅ Each category has human-readable `categoryName`
- ✅ Category names match PlatformCategory entity (e.g., "Appetizers", "Main Course")

---

### **Test Case 2.5: Empty Menu (No Items)**

**ID**: `CHEF_PUB_MENU_005`

**Description**: Verify response for chef with no menu items.

**Endpoint**: `GET /api/v1/chefs/:chefId/menu`

**Test Data**: `CHEF_INCOMPLETE` (no menu items)

**Steps**:
```bash

$chefId = "550e8400-e29b-41d4-a716-446655440002"
RESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/chefs/$chefId/menu")
$data = ($response.Content | jq .)

echo "✅ Test Case 2.5: Empty Menu"
echo "Total Items: $($data.data.totalItems)" # Expected: 0
echo "Categorized: $($data.data.categorized.Count)" # Expected: 0
echo "Uncategorized: $($data.data.uncategorized.Count)" # Expected: 0
```

**Validation Criteria**:
- ✅ Status code: 200 (not 404)
- ✅ `data.totalItems` = 0
- ✅ `data.categorized` = [] (empty array)
- ✅ `data.uncategorized` = [] (empty array)

---

### **Test Case 2.6: Uncategorized Items Handling**

**ID**: `CHEF_PUB_MENU_006`

**Description**: Verify items without platformCategoryId go to uncategorized array.

**Pre-Test Database Setup**:
```sql
-- Create test item without category
INSERT INTO "ChefMenuItem" (id, "chefId", name, price, "isActive", "platformCategoryId")
VALUES (
  '550e8400-e29b-41d4-a716-446655440101',
  '550e8400-e29b-41d4-a716-446655440001',
  'Mystery Dish',
  250.00,
  true,
  NULL -- No category
);
```

**Steps**:
```bash

$chefId = "550e8400-e29b-41d4-a716-446655440001"
RESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/chefs/$chefId/menu")
$data = ($response.Content | jq .)

echo "✅ Test Case 2.6: Uncategorized Items"
echo "Uncategorized Items: $($data.data.uncategorized.Count)" # Expected: 1
echo "Item Name: $($data.data.uncategorized[0].name)" # Expected: "Mystery Dish"
```

**Validation Criteria**:
- ✅ `data.uncategorized` contains item without platformCategoryId
- ✅ Uncategorized items still have `basePricePaise` field

---

## 🎥 **Reels Tests**

### **Test Case 3.1: Get Chef Reels (Default Limit)**

**ID**: `CHEF_PUB_REELS_001`

**Description**: Retrieve chef reels with default limit (20).

**Endpoint**: `GET /api/v1/chefs/:chefId/reels`

**Test Data**: `CHEF_COMPLETE` (8 active reels)

**Steps**:
```bash

$chefId = "550e8400-e29b-41d4-a716-446655440001"
RESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/chefs/$chefId/reels")
$data = ($response.Content | jq .)

echo "✅ Test Case 3.1: Get Chef Reels"
echo "Total Reels: $($data.data.total)" # Expected: 8
echo "Reels Count: $($data.data.reels.Count)" # Expected: 8
```

**Validation Criteria**:
- ✅ Status code: 200
- ✅ `data.reels` is array with 8 elements
- ✅ `data.total` = 8

---

### **Test Case 3.2: S3 URI to HTTPS Conversion**

**ID**: `CHEF_PUB_REELS_002`

**Description**: Verify S3 URIs converted to HTTPS CDN URLs.

**Endpoint**: `GET /api/v1/chefs/:chefId/reels`

**Pre-Test Database Data**:
```javascript
// MongoDB Reel document
{
  userId: "550e8400-e29b-41d4-a716-446655440001",
  videoUrl: "s3://chefooz-media-bucket/reels/chef123-reel1.mp4",
  thumbnailUrl: "s3://chefooz-media-bucket/thumbnails/chef123-reel1.jpg",
  deletedAt: null
}
```

**Steps**:
```bash

$chefId = "550e8400-e29b-41d4-a716-446655440001"
RESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/chefs/$chefId/reels")
$data = ($response.Content | jq .)

$reel = $data.data.reels[0]

echo "✅ Test Case 3.2: S3 URI to HTTPS Conversion"
echo "Video URL: $($reel.videoUrl)" # Expected: https://cdn.chefooz.com/...
echo "Thumbnail URL: $($reel.thumbnailUrl)" # Expected: https://cdn.chefooz.com/...
```

**Validation Criteria**:
- ✅ `videoUrl` starts with "https://" (not "s3://")
- ✅ `thumbnailUrl` starts with "https://" (not "s3://")
- ✅ URLs contain CDN_URL from environment

---

### **Test Case 3.3: Soft-Delete Filter (CRITICAL)**

**ID**: `CHEF_PUB_REELS_003`

**Description**: Verify soft-deleted reels excluded from response.

**Endpoint**: `GET /api/v1/chefs/:chefId/reels`

**Pre-Test Database Data**:
```javascript
// Chef has 10 reels total:
// - 8 active (deletedAt = null)
// - 2 soft-deleted (deletedAt = "2024-11-15T10:00:00Z")
```

**Steps**:
```bash

$chefId = "550e8400-e29b-41d4-a716-446655440001"
RESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/chefs/$chefId/reels")
$data = ($response.Content | jq .)

echo "✅ Test Case 3.3: Soft-Delete Filter"
echo "Returned Reels: $($data.data.total)" # Expected: 8 (not 10)

# Verify no soft-deleted reels
foreach ($reel in $data.data.reels) {
    echo "Reel ID: $($reel.id)" # Should not include deleted reel IDs
}
```

**Validation Criteria**:
- ✅ Only active reels returned (`deletedAt === null`)
- ✅ Total = 8 (excludes 2 soft-deleted reels)
- ✅ No soft-deleted reel IDs in response

---

### **Test Case 3.4: Custom Limit Parameter**

**ID**: `CHEF_PUB_REELS_004`

**Description**: Verify custom limit parameter works correctly.

**Endpoint**: `GET /api/v1/chefs/:chefId/reels?limit=5`

**Test Data**: `CHEF_VEG_ONLY` (12 active reels)

**Steps**:
```bash

$chefId = "550e8400-e29b-41d4-a716-446655440004"
$limit = 5
RESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/chefs/$chefId/reels?limit=$limit")
$data = ($response.Content | jq .)

echo "✅ Test Case 3.4: Custom Limit"
echo "Requested Limit: $limit"
echo "Returned Reels: $($data.data.reels.Count)" # Expected: 5
```

**Validation Criteria**:
- ✅ `data.reels.Count` = 5 (respects limit)
- ✅ Total available reels = 12 (not limited)

---

### **Test Case 3.5: Reels Ordering (Newest First)**

**ID**: `CHEF_PUB_REELS_005`

**Description**: Verify reels sorted by createdAt DESC (newest first).

**Endpoint**: `GET /api/v1/chefs/:chefId/reels`

**Steps**:
```bash

$chefId = "550e8400-e29b-41d4-a716-446655440001"
RESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/chefs/$chefId/reels")
$data = ($response.Content | jq .)

echo "✅ Test Case 3.5: Reels Ordering"

# Check ordering
$previousDate = [DateTime]::MaxValue
foreach ($reel in $data.data.reels) {
    $currentDate = [DateTime]::Parse($reel.createdAt)
    if ($currentDate -gt $previousDate) {
        echo "❌ Order violation: Older reel before newer reel"
    }
    $previousDate = $currentDate
    echo "Reel: $($reel.id), Created: $($reel.createdAt)"
}
```

**Validation Criteria**:
- ✅ Reels sorted by `createdAt` DESC (newest first)
- ✅ Each reel's `createdAt` ≤ previous reel's `createdAt`

---

### **Test Case 3.6: No Reels Available**

**ID**: `CHEF_PUB_REELS_006`

**Description**: Verify response when chef has no reels.

**Endpoint**: `GET /api/v1/chefs/:chefId/reels`

**Test Data**: `CHEF_INCOMPLETE` (0 reels)

**Steps**:
```bash

$chefId = "550e8400-e29b-41d4-a716-446655440002"
RESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/chefs/$chefId/reels")
$data = ($response.Content | jq .)

echo "✅ Test Case 3.6: No Reels Available"
echo "Total Reels: $($data.data.total)" # Expected: 0
echo "Reels Array: $($data.data.reels.Count)" # Expected: 0
```

**Validation Criteria**:
- ✅ Status code: 200 (not 404)
- ✅ `data.total` = 0
- ✅ `data.reels` = [] (empty array)

---

### **Test Case 3.7: Reel Stats Verification**

**ID**: `CHEF_PUB_REELS_007`

**Description**: Verify reel stats (views, likes) returned correctly.

**Endpoint**: `GET /api/v1/chefs/:chefId/reels`

**Pre-Test Database Data**:
```javascript
// MongoDB Reel document
{
  userId: "550e8400-e29b-41d4-a716-446655440001",
  stats: {
    views: 1500,
    likes: 250
  }
}
```

**Steps**:
```bash

$chefId = "550e8400-e29b-41d4-a716-446655440001"
RESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/chefs/$chefId/reels")
$data = ($response.Content | jq .)

$reel = $data.data.reels[0]

echo "✅ Test Case 3.7: Reel Stats"
echo "View Count: $($reel.viewCount)" # Expected: 1500
echo "Like Count: $($reel.likeCount)" # Expected: 250
```

**Validation Criteria**:
- ✅ `viewCount` matches database (1500)
- ✅ `likeCount` matches database (250)
- ✅ Stats default to 0 if null/undefined

---

## 🔄 **Reorder Preview Tests**

### **Test Case 4.1: Reorder Preview (Authenticated User with Orders)**

**ID**: `CHEF_PUB_REORDER_001`

**Description**: Retrieve reorder preview for authenticated user with previous orders.

**Endpoint**: `GET /api/v1/chefs/:chefId/reorder-preview`

**Test Data**: User with last DELIVERED order containing 3 items

**Steps**:
```bash

# Get JWT via OTP auth (Chefooz: no passwords, OTP-only via WhatsApp/Twilio SMS)
OTPRESPONSE=$(curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/auth/v2/send-otp" \
  -H "Content-Type: application/json" \
  -d '{"phoneNumber": "+919876543210"}')
$requestId = ($otpResponse.Content | jq .).data.requestId
$otp = Read-Host "Enter OTP received via WhatsApp or Twilio SMS"
VERIFYRESPONSE=$(curl -s \
  -X POST \
  "https://api-staging.chefooz.com/api/v1/auth/v2/verify-otp" \
  -H "Content-Type: application/json" \
  -d '{`')
$token = ($verifyResponse.Content | jq .).data.accessToken# Request reorder preview
$chefId = "550e8400-e29b-41d4-a716-446655440001"
RESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/chefs/$chefId/reorder-preview")
$data = ($response.Content | jq .)

echo "✅ Test Case 4.1: Reorder Preview (Authenticated)"
echo "Reorderable: $($data.data.reorderable)" # Expected: true
echo "Last Order Date: $($data.data.lastOrderDate)"
echo "Items Count: $($data.data.items.Count)" # Expected: 3
```

**Expected Response**:
```json
{
  "success": true,
  "message": "Reorder preview retrieved successfully",
  "data": {
    "reorderable": true,
    "lastOrderDate": "2024-11-20T10:30:00Z",
    "items": [
      {
        "itemId": "item-123",
        "name": "Butter Chicken",
        "imageUrl": "https://storage.chefooz.com/menu/butter-chicken.jpg",
        "pricePaise": 35000,
        "quantity": 2
      },
      {
        "itemId": "item-456",
        "name": "Garlic Naan",
        "imageUrl": "https://storage.chefooz.com/menu/naan.jpg",
        "pricePaise": 5000,
        "quantity": 4
      }
    ]
  }
}
```

**Validation Criteria**:
- ✅ Status code: 200
- ✅ `data.reorderable` = true
- ✅ `data.lastOrderDate` present
- ✅ `data.items` array with 3 elements
- ✅ Each item has `itemId`, `name`, `pricePaise`, `quantity`

---

### **Test Case 4.2: Reorder Preview (No Previous Orders)**

**ID**: `CHEF_PUB_REORDER_002`

**Description**: Verify null response when user has no previous orders.

**Endpoint**: `GET /api/v1/chefs/:chefId/reorder-preview`

**Test Data**: Authenticated user with no orders from this chef

**Steps**:
```bash

$token = "..." # Valid JWT token
$chefId = "550e8400-e29b-41d4-a716-446655440003" # Chef user never ordered from
RESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/chefs/$chefId/reorder-preview")
$data = ($response.Content | jq .)

echo "✅ Test Case 4.2: No Previous Orders"
echo "Data: $($data.data)" # Expected: null
```

**Expected Response**:
```json
{
  "success": true,
  "message": "No previous orders found",
  "data": null
}
```

**Validation Criteria**:
- ✅ Status code: 200 (not 404)
- ✅ `data` = null (not error)

---

### **Test Case 4.3: Reorder Preview (No Authentication)**

**ID**: `CHEF_PUB_REORDER_003`

**Description**: Verify null response when request has no JWT token.

**Endpoint**: `GET /api/v1/chefs/:chefId/reorder-preview`

**Steps**:
```bash

$chefId = "550e8400-e29b-41d4-a716-446655440001"
RESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/chefs/$chefId/reorder-preview")
# No Authorization header
$data = ($response.Content | jq .)

echo "✅ Test Case 4.3: No Authentication"
echo "Data: $($data.data)" # Expected: null
```

**Validation Criteria**:
- ✅ Status code: 200 (not 401 Unauthorized)
- ✅ `data` = null (graceful degradation)

---

### **Test Case 4.4: Reorder Snapshot Data Verification**

**ID**: `CHEF_PUB_REORDER_004`

**Description**: Verify reorder uses order snapshots (not live menu).

**Endpoint**: `GET /api/v1/chefs/:chefId/reorder-preview`

**Pre-Test Database Data**:
```sql
-- Order item snapshot
{
  "menuItemId": "item-123",
  "titleSnapshot": "Butter Chicken (Old Name)",
  "unitPricePaise": 30000, -- ₹300 at order time
  "quantity": 2
}

-- Current menu item (price changed)
UPDATE "ChefMenuItem" 
SET name = 'Butter Chicken (New Name)', price = 350.00
WHERE id = 'item-123';
```

**Steps**:
```bash

$token = "..." # Valid JWT token
$chefId = "550e8400-e29b-41d4-a716-446655440001"
RESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/chefs/$chefId/reorder-preview")
$data = ($response.Content | jq .)

$item = $data.data.items[0]

echo "✅ Test Case 4.4: Snapshot Data"
echo "Item Name: $($item.name)" # Expected: "Butter Chicken (Old Name)"
echo "Price Paise: $($item.pricePaise)" # Expected: 60000 (30000 × 2)
```

**Validation Criteria**:
- ✅ Item name = "Butter Chicken (Old Name)" (snapshot, not current)
- ✅ Price = 60000 paise (snapshot × quantity, not current price)

---

### **Test Case 4.5: Last Order Only (Not All Orders)**

**ID**: `CHEF_PUB_REORDER_005`

**Description**: Verify only last DELIVERED order shown (not all orders).

**Endpoint**: `GET /api/v1/chefs/:chefId/reorder-preview`

**Pre-Test Database Data**:
```sql
-- User has 3 DELIVERED orders from this chef:
-- Order 1: 2024-11-10 (3 items)
-- Order 2: 2024-11-15 (5 items)
-- Order 3: 2024-11-20 (2 items) <- Most recent
```

**Steps**:
```bash

$token = "..." # Valid JWT token
$chefId = "550e8400-e29b-41d4-a716-446655440001"
RESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/chefs/$chefId/reorder-preview")
$data = ($response.Content | jq .)

echo "✅ Test Case 4.5: Last Order Only"
echo "Last Order Date: $($data.data.lastOrderDate)" # Expected: 2024-11-20
echo "Items Count: $($data.data.items.Count)" # Expected: 2 (not 3 or 5)
```

**Validation Criteria**:
- ✅ `lastOrderDate` = "2024-11-20" (most recent)
- ✅ `items.Count` = 2 (only last order items)

---

### **Test Case 4.6: DELIVERED Orders Only (Not PENDING/CANCELLED)**

**ID**: `CHEF_PUB_REORDER_006`

**Description**: Verify only DELIVERED orders considered (not PENDING/CANCELLED).

**Endpoint**: `GET /api/v1/chefs/:chefId/reorder-preview`

**Pre-Test Database Data**:
```sql
-- User has 3 orders from this chef:
-- Order 1: 2024-11-15, status = DELIVERED
-- Order 2: 2024-11-18, status = PENDING (more recent)
-- Order 3: 2024-11-20, status = CANCELLED (most recent)
```

**Steps**:
```bash

$token = "..." # Valid JWT token
$chefId = "550e8400-e29b-41d4-a716-446655440001"
RESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/chefs/$chefId/reorder-preview")
$data = ($response.Content | jq .)

echo "✅ Test Case 4.6: DELIVERED Orders Only"
echo "Last Order Date: $($data.data.lastOrderDate)" # Expected: 2024-11-15 (DELIVERED)
```

**Validation Criteria**:
- ✅ `lastOrderDate` = "2024-11-15" (last DELIVERED order, not most recent overall)
- ✅ PENDING/CANCELLED orders ignored

---

## 🔗 **Integration Tests**

### **Test Case 5.1: End-to-End Flow - Discover Chef from Reel**

**ID**: `CHEF_PUB_INT_001`

**Description**: Complete flow: View Reel → Visit Chef Public Page → Browse Menu → Add to Cart (future).

**Steps**:
```bash

# Step 1: Get chef reels (discover chef)
$chefId = "550e8400-e29b-41d4-a716-446655440001"
REELSRESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/chefs/$chefId/reels")
$firstReel = (($reelsResponse.Content | jq .).data.reels[0])

echo "Step 1: Discovered Chef via Reel: $($firstReel.id)"

# Step 2: Get chef public profile
PROFILERESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/chefs/$chefId/public")
$profile = ($profileResponse.Content | jq .).data

echo "Step 2: Chef Profile: $($profile.kitchenName), IsOpen: $($profile.isOpen)"

# Step 3: Get chef menu
MENURESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/chefs/$chefId/menu")
$menu = ($menuResponse.Content | jq .).data

echo "Step 3: Menu Items: $($menu.totalItems)"

# Step 4: (Future) Add to Cart
echo "Step 4: Add to Cart (integration pending)"

echo "✅ Test Case 5.1: End-to-End Flow Complete"
```

**Validation Criteria**:
- ✅ All API calls return 200 status
- ✅ Data consistency across endpoints (same chefId)
- ✅ Flow completes without errors

---

### **Test Case 5.2: Cross-Module Data Consistency**

**ID**: `CHEF_PUB_INT_002`

**Description**: Verify data consistency between Chef-Kitchen and Chef-Public modules.

**Steps**:
```bash

$chefId = "550e8400-e29b-41d4-a716-446655440001"

# Get data from Chef-Kitchen API (admin/owner access)
$token = "..." # Chef's JWT token
KITCHENRESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/chefs/kitchen")
$kitchenData = ($kitchenResponse.Content | jq .).data

# Get data from Chef-Public API
PUBLICRESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/chefs/$chefId/public")
$publicData = ($publicResponse.Content | jq .).data

echo "✅ Test Case 5.2: Data Consistency"
echo "Kitchen Name Match: $($kitchenData.kitchenName -eq $publicData.kitchenName)"
echo "IsOnline Match: $($kitchenData.isOnline -eq $publicData.isOpen)"
echo "Delivery Radius Match: $($kitchenData.deliveryRadiusKm -eq $publicData.deliveryRadiusKm)"
```

**Validation Criteria**:
- ✅ Kitchen name matches between modules
- ✅ `isOnline` (Chef-Kitchen) = `isOpen` (Chef-Public)
- ✅ Delivery radius matches

---

## ⚡ **Performance Tests**

### **Test Case 6.1: Chef Profile Load Time**

**ID**: `CHEF_PUB_PERF_001`

**Description**: Verify chef profile loads within 500ms (p95).

**Endpoint**: `GET /api/v1/chefs/:chefId/public`

**Steps**:
```bash

$chefId = "550e8400-e29b-41d4-a716-446655440001"

# Run 10 iterations
$times = @()
for ($i = 1; $i -le 10; $i++) {
    $start = Get-Date
RESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/chefs/$chefId/public")
    $end = Get-Date
    $duration = ($end - $start).TotalMilliseconds
    $times += $duration
    echo "Iteration $i: $duration ms"
}

# Calculate p95
$sorted = $times | Sort-Object
$p95Index = [Math]::Ceiling($sorted.Count * 0.95) - 1
$p95 = $sorted[$p95Index]

echo "✅ Test Case 6.1: Chef Profile Performance"
echo "P95: $p95 ms" # Expected: < 500ms
```

**Validation Criteria**:
- ✅ P95 response time < 500ms
- ✅ All requests return 200 status

---

### **Test Case 6.2: Menu Load Time**

**ID**: `CHEF_PUB_PERF_002`

**Description**: Verify menu loads within 800ms (p95).

**Endpoint**: `GET /api/v1/chefs/:chefId/menu`

**Steps**: (Same as 6.1, different endpoint)

**Validation Criteria**:
- ✅ P95 response time < 800ms

---

### **Test Case 6.3: Reels Load Time**

**ID**: `CHEF_PUB_PERF_003`

**Description**: Verify reels load within 600ms (p95).

**Endpoint**: `GET /api/v1/chefs/:chefId/reels`

**Validation Criteria**:
- ✅ P95 response time < 600ms

---

## 🔒 **Security Tests**

### **Test Case 7.1: Public Access Verification**

**ID**: `CHEF_PUB_SEC_001`

**Description**: Verify public endpoints accessible without JWT token.

**Endpoints**: 
- `GET /api/v1/chefs/:chefId/public`
- `GET /api/v1/chefs/:chefId/menu`
- `GET /api/v1/chefs/:chefId/reels`

**Steps**:
```bash

$chefId = "550e8400-e29b-41d4-a716-446655440001"

# No Authorization header
PROFILERESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/chefs/$chefId/public")
MENURESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/chefs/$chefId/menu")
REELSRESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/chefs/$chefId/reels")

echo "✅ Test Case 7.1: Public Access"
echo "Profile: $($profileResponse.StatusCode)" # Expected: 200
echo "Menu: $($menuResponse.StatusCode)" # Expected: 200
echo "Reels: $($reelsResponse.StatusCode)" # Expected: 200
```

**Validation Criteria**:
- ✅ All endpoints return 200 without JWT
- ✅ No 401 Unauthorized errors

---

### **Test Case 7.2: No Sensitive Data Exposure**

**ID**: `CHEF_PUB_SEC_002`

**Description**: Verify no sensitive chef data exposed in public profile.

**Endpoint**: `GET /api/v1/chefs/:chefId/public`

**Steps**:
```bash

$chefId = "550e8400-e29b-41d4-a716-446655440001"
RESPONSE=$(curl -s \
  -X GET \
  "https://api-staging.chefooz.com/api/v1/chefs/$chefId/public")
$data = ($response.Content | jq .).data

echo "✅ Test Case 7.2: No Sensitive Data"

# Check for sensitive fields (should not be present)
$sensitiveFields = @("email", "phoneNumber", "fssaiNumber", "bankAccount", "address")
foreach ($field in $sensitiveFields) {
    if ($data.PSObject.Properties.Name -contains $field) {
        echo "❌ SECURITY ISSUE: Sensitive field '$field' exposed"
    }
}
```

**Validation Criteria**:
- ✅ No email, phone, FSSAI, bank account fields in response
- ✅ Only public-safe data returned

---

## 🤖 **Automation Scripts**

### **Full Test Suite (PowerShell)**

```bash

# full-test-suite.ps1

echo "========================================="
echo "Chef-Public Module - Full Test Suite"
echo "========================================="

# Configuration
$baseUrl = "https://api-staging.chefooz.com"
$chefComplete = "550e8400-e29b-41d4-a716-446655440001"
$chefIncomplete = "550e8400-e29b-41d4-a716-446655440002"

# Test counters
$passed = 0
$failed = 0

# Helper function
function Test-Endpoint {
    param(
        [string]$TestId,
        [string]$Description,
        [string]$Url,
        [int]$ExpectedStatus,
        [scriptblock]$Validation
    )
    
    try {
        echo "`n[$TestId] $Description"
RESPONSE=$(curl -s \
  -X GET \
  "$Url")
        $data = $response.Content | jq .
        
        if ($response.StatusCode -ne $ExpectedStatus) {
            throw "Expected status $ExpectedStatus, got $($response.StatusCode)"
        }
        
        & $Validation -Data $data
        
        echo "✅ PASSED" -ForegroundColor Green
        $script:passed++
    } catch {
        echo "❌ FAILED: $_" -ForegroundColor Red
        $script:failed++
    }
}

# Run tests
echo "`n=== Chef Profile Tests ==="

Test-Endpoint  -TestId "CHEF_PUB_PROFILE_001"  -Description "Get Complete Chef Profile"  -Url "$baseUrl/api/v1/chefs/$chefComplete/public"  -ExpectedStatus 200  -Validation {
        param($Data)
        if ($Data.data.chefId -ne $chefComplete) { throw "Chef ID mismatch" }
        if ($Data.data.totalOrders -lt 0) { throw "Invalid order count" }
    }

Test-Endpoint  -TestId "CHEF_PUB_PROFILE_002"  -Description "Get Minimal Profile"  -Url "$baseUrl/api/v1/chefs/$chefIncomplete/public"  -ExpectedStatus 200  -Validation {
        param($Data)
        if ($Data.data.isOpen -ne $false) { throw "Expected isOpen = false" }
        if ($Data.data.totalOrders -ne 0) { throw "Expected 0 orders" }
    }

echo "`n=== Menu Tests ==="

Test-Endpoint  -TestId "CHEF_PUB_MENU_001"  -Description "Get Grouped Menu"  -Url "$baseUrl/api/v1/chefs/$chefComplete/menu"  -ExpectedStatus 200  -Validation {
        param($Data)
        if ($Data.data.totalItems -lt 0) { throw "Invalid item count" }
        if ($Data.data.categorized -isnot [Array]) { throw "Categorized not array" }
    }

echo "`n=== Reels Tests ==="

Test-Endpoint  -TestId "CHEF_PUB_REELS_001"  -Description "Get Chef Reels"  -Url "$baseUrl/api/v1/chefs/$chefComplete/reels"  -ExpectedStatus 200  -Validation {
        param($Data)
        if ($Data.data.reels -isnot [Array]) { throw "Reels not array" }
        foreach ($reel in $Data.data.reels) {
            if (-not $reel.videoUrl.StartsWith("https://")) {
                throw "Video URL not HTTPS"
            }
        }
    }

# Summary
echo "`n========================================="
echo "Test Summary"
echo "========================================="
echo "Passed: $passed" -ForegroundColor Green
echo "Failed: $failed" -ForegroundColor Red
echo "Total: $($passed + $failed)"

if ($failed -eq 0) {
    echo "`n✅ All tests passed!" -ForegroundColor Green
    exit 0
} else {
    echo "`n❌ Some tests failed" -ForegroundColor Red
    exit 1
}
```

---

## 📊 **Test Results Template**

### **Test Execution Report**

| Category | Test Cases | Passed | Failed | Duration |
|----------|-----------|--------|--------|----------|
| Chef Profile | 7 | 7 | 0 | 3.2s |
| Menu | 6 | 6 | 0 | 4.1s |
| Reels | 7 | 7 | 0 | 2.8s |
| Reorder Preview | 6 | 6 | 0 | 2.5s |
| Integration | 2 | 2 | 0 | 1.8s |
| Performance | 3 | 3 | 0 | 15.0s |
| Security | 2 | 2 | 0 | 1.2s |
| **Total** | **33** | **33** | **0** | **30.6s** |

---

## ✅ **Test Coverage Summary**

### **Coverage by Feature**

- ✅ **Chef Public Profile**: 7 test cases (complete setup, minimal profile, not found, vegType, order count, offline status)
- ✅ **Public Menu**: 6 test cases (grouped menu, price conversion, active items, category names, empty menu, uncategorized)
- ✅ **Chef Reels**: 7 test cases (default limit, S3 conversion, soft-delete filter, custom limit, ordering, no reels, stats)
- ✅ **Reorder Preview**: 6 test cases (authenticated, no orders, no auth, snapshots, last order, delivered only)
- ✅ **Integration**: 2 test cases (end-to-end flow, cross-module consistency)
- ✅ **Performance**: 3 test cases (profile < 500ms, menu < 800ms, reels < 600ms)
- ✅ **Security**: 2 test cases (public access, no sensitive data)

### **Total Test Cases**: 33

### **Critical Test Cases** (Must Pass):
1. ✅ **CHEF_PUB_PROFILE_001**: Complete chef profile
2. ✅ **CHEF_PUB_PROFILE_002**: Minimal profile fallback (graceful degradation)
3. ✅ **CHEF_PUB_MENU_001**: Grouped menu
4. ✅ **CHEF_PUB_MENU_002**: Price conversion (rupees → paise)
5. ✅ **CHEF_PUB_REELS_002**: S3 URI to HTTPS conversion
6. ✅ **CHEF_PUB_REELS_003**: Soft-delete filter (CRITICAL)
7. ✅ **CHEF_PUB_REORDER_004**: Snapshot data (not live menu)
8. ✅ **CHEF_PUB_SEC_002**: No sensitive data exposure

---

**[QA_TEST_CASES_COMPLETE ✅]**

*For feature overview, see `01_FEATURE_OVERVIEW.md`. For technical implementation, see `02_TECHNICAL_GUIDE.md`.*

---

## 🌙 Dark Mode Regression Tests (March 2026)

### TC-CHEF-PUB-DM-001: Chef Public Page — Dark Mode Section Backgrounds

**Type:** Bug Regression  
**Feature area:** `app/chef/[chefId].tsx`  
**Priority:** P1

**Preconditions:**
- Device/simulator in dark mode
- At least one chef with a published menu and reel exists

**Steps:**
1. Enable dark mode
2. Navigate to a chef's public page (via Explore or reel CTA "View Menu")
3. Scroll past the hero section to the body content
4. Observe: "What's Cooking" section background, kitchen info card, trust badges
5. Observe: "Order Again" reorder card
6. Observe: Menu section background, category pills (inactive), menu item count
7. Observe: "Loved by X+ customers" social proof section

**Expected result:**  
- Hero card (`kitchenInfoCard`) uses `colors.surfaceElevated` in dark mode (not near-white `rgba(255,255,255,0.95)`)  
- Section backgrounds (`whatsCookingSection`, `menuSection`, `reorderSection`, `socialProofSection`) use `colors.surface` not `#FFF`  
- Kitchen name, section titles use `colors.textPrimary`; secondary info uses `colors.textSecondary`  
- Trust badges use `colors.surfaceElevated` pill background  
- Category pills use `colors.surfaceElevated` background with `colors.border` outline  
- Reorder card uses `colors.surfaceElevated` background with `colors.border` dashed border  

**Actual result (before fix):** All `backgroundColor: '#FFF'` sections render white in dark mode; `kitchenInfoCard` renders white with black text; trust badges show `#F5F5F5` (light)  
**Fix applied:** Replaced `StyleSheet.create` with `makeStyles(colors, isDark)`; added `useMemo`; all section and component backgrounds now use `colors.surface`, `colors.surfaceElevated`, `colors.border`, `colors.textPrimary`, `colors.textSecondary`, `colors.textMuted` tokens  
**Regression test:** `apps/chefooz-app/src/app/chef/chefId.spec.ts` (visual smoke)  
**Status:** Fixed ✅

---

**Document Version**: 1.1  
**Last Updated**: March 2026  
**Test Environment**: Local Development  
**Next Review**: Q2 2026 (Performance Benchmarking)
