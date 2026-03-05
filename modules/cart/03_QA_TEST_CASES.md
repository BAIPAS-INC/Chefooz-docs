# 🛒 Cart Module - QA Test Cases

## 📋 **Table of Contents**
- [Test Environment Setup](#test-environment-setup)
- [Test Data Preparation](#test-data-preparation)
- [API Endpoint Tests](#api-endpoint-tests)
- [Business Logic Tests](#business-logic-tests)
- [Integration Tests](#integration-tests)
- [Performance Tests](#performance-tests)
- [PowerShell Test Automation](#powershell-test-automation)

---

## 🔧 **Test Environment Setup**

### **Prerequisites**

1. **Backend Running**:
   ```bash
   cd apps/chefooz-apis
   npm run start:dev
   ```

2. **PostgreSQL Running**:
   ```bash
   docker-compose up -d postgres
   ```

3. **Valkey/Redis Running**:
   ```bash
   docker-compose up -d valkey
   ```

4. **Test User JWT Token** (OTP-based):
   ```bash
   # Step 1: Send OTP
   curl -X POST https://api-staging.chefooz.com/api/v1/auth/v2/send-otp \
     -H "Content-Type: application/json" \
     -d '{"phone":"9876543210","deviceId":"test-device-123"}'
   
   # Response: { "sessionId": "uuid-here", "expiresAt": "...", "channel": "whatsapp" }
   
   # Step 2: Verify OTP (use OTP from WhatsApp/SMS)
   curl -X POST https://api-staging.chefooz.com/api/v1/auth/v2/verify-otp \
     -H "Content-Type: application/json" \
     -d '{"sessionId":"uuid-from-step1","otp":"1234"}'
   
   # Extract token from response
   export JWT_TOKEN="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
   ```

### **Base URL**

```
https://api-staging.chefooz.com/api/v1/cart
```

### **Headers**

```
Authorization: Bearer $JWT_TOKEN
Content-Type: application/json
```

---

## 📊 **Test Data Preparation**

### **Setup Script** (PowerShell)

```powershell
# test-data-setup.ps1
# Creates test users, chefs, and menu items

$baseUrl = "https://api-staging.chefooz.com/api/v1"

# 1. Create test users via OTP authentication
# Step 1: Send OTP
$otpResponse = Invoke-RestMethod -Uri "$baseUrl/auth/v2/send-otp" -Method Post -Body (@{
    phone = "9876543210"
    deviceId = "test-device-cart-user1"
} | ConvertTo-Json) -ContentType "application/json"

Write-Host "📱 OTP sent to 9876543210 via $($otpResponse.data.channel)"
Write-Host "⏳ Enter OTP from your device:"
$otpCode = Read-Host "OTP"

# Step 2: Verify OTP and get JWT
$user1 = Invoke-RestMethod -Uri "$baseUrl/auth/v2/verify-otp" -Method Post -Body (@{
    sessionId = $otpResponse.data.sessionId
    otp = $otpCode
} | ConvertTo-Json) -ContentType "application/json"

Write-Host "✅ Test User 1 Authenticated: $($user1.data.user.id)"

# Step 3: Complete profile (if needed)
if (-not $user1.data.user.username) {
    Invoke-RestMethod -Uri "$baseUrl/auth/profile" -Method Put -Headers @{
        Authorization = "Bearer $($user1.data.accessToken)"
    } -Body (@{
        username = "cart_test_user1"
        fullName = "Cart Test User 1"
    } | ConvertTo-Json) -ContentType "application/json" | Out-Null
    Write-Host "✅ Profile completed for user"
}

# 2. Create test chefs
$chef1 = Invoke-RestMethod -Uri "$baseUrl/chef" -Method Post -Headers @{
    Authorization = "Bearer $($user1.data.token)"
} -Body (@{
    businessName = "Mumbai Masala Kitchen"
    cuisineTypes = @("Indian", "North Indian")
} | ConvertTo-Json) -ContentType "application/json"

Write-Host "✅ Test Chef 1 Created: $($chef1.data.id)"

$chef2 = Invoke-RestMethod -Uri "$baseUrl/chef" -Method Post -Headers @{
    Authorization = "Bearer $($user1.data.token)"
} -Body (@{
    businessName = "Delhi Delights"
    cuisineTypes = @("Indian", "Mughlai")
} | ConvertTo-Json) -ContentType "application/json"

Write-Host "✅ Test Chef 2 Created: $($chef2.data.id)"

# 3. Create test menu items for Chef 1
$menuItem1 = Invoke-RestMethod -Uri "$baseUrl/chef-kitchen/menu-items" -Method Post -Headers @{
    Authorization = "Bearer $($user1.data.token)"
} -Body (@{
    name = "Butter Chicken"
    description = "Rich creamy curry with tender chicken"
    pricePaise = 25000
    foodType = "non-veg"
    isActive = $true
    availability = @{
        isAvailable = $true
        soldOut = $false
    }
} | ConvertTo-Json) -ContentType "application/json"

Write-Host "✅ Menu Item 1 Created: $($menuItem1.data.id)"

# Export for tests
$testData = @{
    user1Token = $user1.data.accessToken
    user1Id = $user1.data.user.id
    chef1Id = $chef1.data.id
    chef2Id = $chef2.data.id
    menuItem1Id = $menuItem1.data.id
}

$testData | ConvertTo-Json | Out-File "test-data.json"
Write-Host "✅ Test data saved to test-data.json"
```

---

## 🧪 **API Endpoint Tests**

### **Category 1: Cart Retrieval Tests**

#### **Test 1.1: Get Empty Cart**

**Objective**: Verify empty cart returns correct structure.

**Request**:
```powershell
$response = Invoke-RestMethod -Uri "https://api-staging.chefooz.com/api/v1/cart" `
    -Method Get `
    -Headers @{ Authorization = "Bearer $JWT_TOKEN" }
```

**Expected Response** (200 OK):
```json
{
  "success": true,
  "message": "Cart is empty",
  "data": {
    "items": [],
    "chefId": null,
    "totals": {
      "subtotalPaise": 0,
      "deliveryPaise": 0,
      "taxPaise": 0,
      "discountPaise": 0,
      "grandTotalPaise": 0
    },
    "itemCount": 0
  }
}
```

**Validation**:
- ✅ Status code: 200
- ✅ `success` is `true`
- ✅ `data.items` is empty array
- ✅ `data.chefId` is `null`
- ✅ All totals are 0
- ✅ `itemCount` is 0

---

#### **Test 1.2: Get Cart with Items**

**Objective**: Verify cart retrieval with items returns full data.

**Precondition**: Add 2 items to cart

**Request**:
```powershell
$response = Invoke-RestMethod -Uri "https://api-staging.chefooz.com/api/v1/cart" `
    -Method Get `
    -Headers @{ Authorization = "Bearer $JWT_TOKEN" }
```

**Expected Response** (200 OK):
```json
{
  "success": true,
  "message": "Cart retrieved successfully",
  "data": {
    "items": [
      {
        "id": "cart-item-uuid-1",
        "menuItemId": "menu-item-uuid-1",
        "quantity": 2,
        "unitPricePaise": 25000,
        "titleSnapshot": "Butter Chicken",
        "imageSnapshot": "https://...",
        "optionsSnapshot": { "size": "large" },
        "removedIngredients": [],
        "addedIngredients": [],
        "customerCookingPreferences": null
      }
    ],
    "chefId": "chef-uuid-1",
    "totals": {
      "subtotalPaise": 50000,
      "deliveryPaise": 3000,
      "taxPaise": 2500,
      "discountPaise": 0,
      "grandTotalPaise": 55500
    },
    "itemCount": 2
  }
}
```

**Validation**:
- ✅ Status code: 200
- ✅ `data.items` contains cart items
- ✅ `data.chefId` is set
- ✅ `totals.grandTotalPaise` > 0
- ✅ `itemCount` matches sum of quantities

---

#### **Test 1.3: Get Cart Count**

**Objective**: Verify cart count endpoint returns correct count.

**Precondition**: Cart has 3 items (quantities: 2, 1, 3)

**Request**:
```powershell
$response = Invoke-RestMethod -Uri "https://api-staging.chefooz.com/api/v1/cart/count" `
    -Method Get `
    -Headers @{ Authorization = "Bearer $JWT_TOKEN" }
```

**Expected Response** (200 OK):
```json
{
  "success": true,
  "message": "Cart count retrieved successfully",
  "data": {
    "count": 6
  }
}
```

**Validation**:
- ✅ Status code: 200
- ✅ `data.count` is 6 (2+1+3)

---

#### **Test 1.4: Get Cart Count - Empty Cart**

**Objective**: Verify count returns 0 for empty cart.

**Precondition**: Cart is empty

**Request**:
```powershell
$response = Invoke-RestMethod -Uri "https://api-staging.chefooz.com/api/v1/cart/count" `
    -Method Get `
    -Headers @{ Authorization = "Bearer $JWT_TOKEN" }
```

**Expected Response** (200 OK):
```json
{
  "success": true,
  "message": "Cart count retrieved successfully",
  "data": {
    "count": 0
  }
}
```

**Validation**:
- ✅ Status code: 200
- ✅ `data.count` is 0

---

### **Category 2: Add Item Tests**

#### **Test 2.1: Add Item to Empty Cart**

**Objective**: Verify adding first item to cart succeeds.

**Precondition**: Cart is empty

**Request**:
```powershell
$body = @{
    menuItemId = "menu-item-uuid-1"
    quantity = 2
    options = @{
        size = "large"
        spice = "medium"
    }
} | ConvertTo-Json

$response = Invoke-RestMethod -Uri "https://api-staging.chefooz.com/api/v1/cart/items" `
    -Method Post `
    -Headers @{ Authorization = "Bearer $JWT_TOKEN"; "Content-Type" = "application/json" } `
    -Body $body
```

**Expected Response** (200 OK):
```json
{
  "success": true,
  "message": "Item added to cart successfully",
  "data": {
    "items": [
      {
        "id": "cart-item-uuid",
        "menuItemId": "menu-item-uuid-1",
        "quantity": 2,
        "unitPricePaise": 25000,
        "titleSnapshot": "Butter Chicken",
        "optionsSnapshot": { "size": "large", "spice": "medium" }
      }
    ],
    "chefId": "chef-uuid-1",
    "totals": { ... },
    "itemCount": 2
  }
}
```

**Validation**:
- ✅ Status code: 200
- ✅ `data.items` contains new item
- ✅ `data.chefId` is set to menu item's chef
- ✅ Item quantity is 2
- ✅ Price snapshot captured

---

#### **Test 2.2: Add Same Item (Increment Quantity)**

**Objective**: Verify adding same item increments quantity.

**Precondition**: Cart has "Butter Chicken" (Large, Medium Spice) × 2

**Request**:
```powershell
$body = @{
    menuItemId = "menu-item-uuid-1"
    quantity = 1
    options = @{
        size = "large"
        spice = "medium"
    }
} | ConvertTo-Json

$response = Invoke-RestMethod -Uri "https://api-staging.chefooz.com/api/v1/cart/items" `
    -Method Post `
    -Headers @{ Authorization = "Bearer $JWT_TOKEN"; "Content-Type" = "application/json" } `
    -Body $body
```

**Expected Response** (200 OK):
```json
{
  "success": true,
  "message": "Item added to cart successfully",
  "data": {
    "items": [
      {
        "id": "cart-item-uuid",
        "quantity": 3
      }
    ],
    "itemCount": 3
  }
}
```

**Validation**:
- ✅ Status code: 200
- ✅ Item quantity increased to 3 (2+1)
- ✅ No duplicate cart item created
- ✅ `itemCount` is 3

---

#### **Test 2.3: Add Item with Different Options (New Cart Item)**

**Objective**: Verify adding same menu item with different options creates separate cart item.

**Precondition**: Cart has "Butter Chicken" (Large, Medium Spice) × 2

**Request**:
```powershell
$body = @{
    menuItemId = "menu-item-uuid-1"
    quantity = 1
    options = @{
        size = "medium"  # Different size
        spice = "hot"     # Different spice
    }
} | ConvertTo-Json

$response = Invoke-RestMethod -Uri "https://api-staging.chefooz.com/api/v1/cart/items" `
    -Method Post `
    -Headers @{ Authorization = "Bearer $JWT_TOKEN"; "Content-Type" = "application/json" } `
    -Body $body
```

**Expected Response** (200 OK):
```json
{
  "success": true,
  "message": "Item added to cart successfully",
  "data": {
    "items": [
      {
        "id": "cart-item-uuid-1",
        "quantity": 2,
        "optionsSnapshot": { "size": "large", "spice": "medium" }
      },
      {
        "id": "cart-item-uuid-2",
        "quantity": 1,
        "optionsSnapshot": { "size": "medium", "spice": "hot" }
      }
    ],
    "itemCount": 3
  }
}
```

**Validation**:
- ✅ Status code: 200
- ✅ 2 separate cart items created
- ✅ Different options preserved
- ✅ `itemCount` is 3 (2+1)

---

#### **Test 2.4: Add Item with Customizations**

**Objective**: Verify adding item with removed/added ingredients and cooking preferences.

**Request**:
```powershell
$body = @{
    menuItemId = "menu-item-uuid-1"
    quantity = 2
    options = @{ size = "large" }
    removedIngredients = @("onions", "garlic")
    addedIngredients = @(
        @{ name = "extra cheese"; pricePaise = 2000 },
        @{ name = "extra chicken"; pricePaise = 5000 }
    )
    customerCookingPreferences = "Make it less spicy and extra crispy"
} | ConvertTo-Json -Depth 3

$response = Invoke-RestMethod -Uri "https://api-staging.chefooz.com/api/v1/cart/items" `
    -Method Post `
    -Headers @{ Authorization = "Bearer $JWT_TOKEN"; "Content-Type" = "application/json" } `
    -Body $body
```

**Expected Response** (200 OK):
```json
{
  "success": true,
  "message": "Item added to cart successfully",
  "data": {
    "items": [
      {
        "id": "cart-item-uuid",
        "quantity": 2,
        "unitPricePaise": 32000,
        "removedIngredients": ["onions", "garlic"],
        "addedIngredients": [
          { "name": "extra cheese", "pricePaise": 2000 },
          { "name": "extra chicken", "pricePaise": 5000 }
        ],
        "customerCookingPreferences": "Make it less spicy and extra crispy"
      }
    ]
  }
}
```

**Validation**:
- ✅ Status code: 200
- ✅ `unitPricePaise` is 32000 (25000 + 2000 + 5000)
- ✅ Removed ingredients saved
- ✅ Added ingredients saved with prices
- ✅ Cooking preferences saved

---

#### **Test 2.5: Add Item from Different Chef (Chef Mismatch)**

**Objective**: Verify adding item from different chef returns 409 error.

**Precondition**: Cart has items from Chef A

**Request**:
```powershell
$body = @{
    menuItemId = "chef-b-menu-item-uuid"
    quantity = 1
} | ConvertTo-Json

try {
    $response = Invoke-RestMethod -Uri "https://api-staging.chefooz.com/api/v1/cart/items" `
        -Method Post `
        -Headers @{ Authorization = "Bearer $JWT_TOKEN"; "Content-Type" = "application/json" } `
        -Body $body
} catch {
    $errorResponse = $_.ErrorDetails.Message | ConvertFrom-Json
}
```

**Expected Response** (409 Conflict):
```json
{
  "success": false,
  "message": "Cart can only contain items from one chef. Clear your cart to add items from a different chef.",
  "errorCode": "CART_CHEF_MISMATCH",
  "error": {
    "currentChefId": "chef-uuid-1",
    "currentChefName": "Mumbai Masala Kitchen",
    "newChefId": "chef-uuid-2",
    "newChefName": "Delhi Delights"
  }
}
```

**Validation**:
- ✅ Status code: 409
- ✅ `errorCode` is "CART_CHEF_MISMATCH"
- ✅ `error.currentChefId` matches existing cart's chef
- ✅ `error.newChefId` matches new item's chef
- ✅ Chef names included for UX

---

#### **Test 2.6: Add Invalid Menu Item**

**Objective**: Verify adding non-existent menu item returns 404.

**Request**:
```powershell
$body = @{
    menuItemId = "non-existent-uuid"
    quantity = 1
} | ConvertTo-Json

try {
    $response = Invoke-RestMethod -Uri "https://api-staging.chefooz.com/api/v1/cart/items" `
        -Method Post `
        -Headers @{ Authorization = "Bearer $JWT_TOKEN"; "Content-Type" = "application/json" } `
        -Body $body
} catch {
    $errorResponse = $_.ErrorDetails.Message | ConvertFrom-Json
}
```

**Expected Response** (404 Not Found):
```json
{
  "success": false,
  "message": "Menu item not found",
  "errorCode": "MENU_ITEM_NOT_FOUND"
}
```

**Validation**:
- ✅ Status code: 404
- ✅ `errorCode` is "MENU_ITEM_NOT_FOUND"

---

#### **Test 2.7: Add Unavailable Menu Item**

**Objective**: Verify adding unavailable/sold out item returns 400.

**Precondition**: Menu item marked as unavailable or sold out

**Request**:
```powershell
$body = @{
    menuItemId = "unavailable-item-uuid"
    quantity = 1
} | ConvertTo-Json

try {
    $response = Invoke-RestMethod -Uri "https://api-staging.chefooz.com/api/v1/cart/items" `
        -Method Post `
        -Headers @{ Authorization = "Bearer $JWT_TOKEN"; "Content-Type" = "application/json" } `
        -Body $body
} catch {
    $errorResponse = $_.ErrorDetails.Message | ConvertFrom-Json
}
```

**Expected Response** (400 Bad Request):
```json
{
  "success": false,
  "message": "Menu item is currently unavailable",
  "errorCode": "MENU_ITEM_UNAVAILABLE"
}
```

**Validation**:
- ✅ Status code: 400
- ✅ `errorCode` is "MENU_ITEM_UNAVAILABLE"

---

### **Category 3: Update & Remove Tests**

#### **Test 3.1: Update Item Quantity**

**Objective**: Verify updating cart item quantity succeeds.

**Precondition**: Cart has item with quantity 2

**Request**:
```powershell
$body = @{
    quantity = 5
} | ConvertTo-Json

$response = Invoke-RestMethod -Uri "https://api-staging.chefooz.com/api/v1/cart/items/$cartItemId" `
    -Method Patch `
    -Headers @{ Authorization = "Bearer $JWT_TOKEN"; "Content-Type" = "application/json" } `
    -Body $body
```

**Expected Response** (200 OK):
```json
{
  "success": true,
  "message": "Cart item updated successfully",
  "data": {
    "items": [
      {
        "id": "cart-item-uuid",
        "quantity": 5
      }
    ],
    "itemCount": 5
  }
}
```

**Validation**:
- ✅ Status code: 200
- ✅ Item quantity updated to 5
- ✅ Cache invalidated (cart count updated)

---

#### **Test 3.2: Update Quantity to 0 (Remove)**

**Objective**: Verify setting quantity to 0 removes item.

**Precondition**: Cart has item

**Request**:
```powershell
$body = @{
    quantity = 0
} | ConvertTo-Json

$response = Invoke-RestMethod -Uri "https://api-staging.chefooz.com/api/v1/cart/items/$cartItemId" `
    -Method Patch `
    -Headers @{ Authorization = "Bearer $JWT_TOKEN"; "Content-Type" = "application/json" } `
    -Body $body
```

**Expected Response** (200 OK):
```json
{
  "success": true,
  "message": "Cart item removed successfully",
  "data": {
    "items": [],
    "itemCount": 0
  }
}
```

**Validation**:
- ✅ Status code: 200
- ✅ Item removed from cart
- ✅ Message says "removed"

---

#### **Test 3.3: Remove Item via DELETE**

**Objective**: Verify deleting cart item succeeds.

**Precondition**: Cart has item

**Request**:
```powershell
$response = Invoke-RestMethod -Uri "https://api-staging.chefooz.com/api/v1/cart/items/$cartItemId" `
    -Method Delete `
    -Headers @{ Authorization = "Bearer $JWT_TOKEN" }
```

**Expected Response** (200 OK):
```json
{
  "success": true,
  "message": "Item removed from cart successfully",
  "data": {
    "items": [],
    "itemCount": 0
  }
}
```

**Validation**:
- ✅ Status code: 200
- ✅ Item removed from cart

---

#### **Test 3.4: Remove Non-Existent Item**

**Objective**: Verify removing non-existent item returns 404.

**Request**:
```powershell
try {
    $response = Invoke-RestMethod -Uri "https://api-staging.chefooz.com/api/v1/cart/items/non-existent-uuid" `
        -Method Delete `
        -Headers @{ Authorization = "Bearer $JWT_TOKEN" }
} catch {
    $errorResponse = $_.ErrorDetails.Message | ConvertFrom-Json
}
```

**Expected Response** (404 Not Found):
```json
{
  "success": false,
  "message": "Cart item not found",
  "errorCode": "CART_ITEM_NOT_FOUND"
}
```

**Validation**:
- ✅ Status code: 404
- ✅ `errorCode` is "CART_ITEM_NOT_FOUND"

---

#### **Test 3.5: Clear Cart**

**Objective**: Verify clearing entire cart succeeds.

**Precondition**: Cart has 3 items

**Request**:
```powershell
$response = Invoke-RestMethod -Uri "https://api-staging.chefooz.com/api/v1/cart/clear" `
    -Method Post `
    -Headers @{ Authorization = "Bearer $JWT_TOKEN" }
```

**Expected Response** (200 OK):
```json
{
  "success": true,
  "message": "Cart cleared successfully",
  "data": {
    "items": [],
    "chefId": null,
    "totals": {
      "subtotalPaise": 0,
      "deliveryPaise": 0,
      "taxPaise": 0,
      "discountPaise": 0,
      "grandTotalPaise": 0
    },
    "itemCount": 0
  }
}
```

**Validation**:
- ✅ Status code: 200
- ✅ All items removed
- ✅ `chefId` reset to null
- ✅ All totals reset to 0

---

### **Category 4: Validation Tests**

#### **Test 4.1: Validate Valid Cart**

**Objective**: Verify validation passes for unchanged cart.

**Precondition**: Cart has items, no menu changes

**Request**:
```powershell
$response = Invoke-RestMethod -Uri "https://api-staging.chefooz.com/api/v1/cart/validate" `
    -Method Post `
    -Headers @{ Authorization = "Bearer $JWT_TOKEN" }
```

**Expected Response** (200 OK):
```json
{
  "success": true,
  "message": "Cart is valid",
  "data": {
    "valid": true,
    "changes": [],
    "totals": {
      "subtotalPaise": 50000,
      "deliveryPaise": 3000,
      "taxPaise": 2500,
      "discountPaise": 0,
      "grandTotalPaise": 55500
    },
    "message": "Cart is valid"
  }
}
```

**Validation**:
- ✅ Status code: 200
- ✅ `data.valid` is `true`
- ✅ `data.changes` is empty array
- ✅ Totals calculated correctly

---

#### **Test 4.2: Validate Cart with Price Change**

**Objective**: Verify validation detects price changes.

**Precondition**: 
1. Cart has "Butter Chicken" at ₹250
2. Update menu item price to ₹300

**Request**:
```powershell
$response = Invoke-RestMethod -Uri "https://api-staging.chefooz.com/api/v1/cart/validate" `
    -Method Post `
    -Headers @{ Authorization = "Bearer $JWT_TOKEN" }
```

**Expected Response** (200 OK):
```json
{
  "success": true,
  "message": "Cart validated with changes",
  "data": {
    "valid": false,
    "changes": [
      {
        "type": "price_change",
        "itemId": "cart-item-uuid",
        "titleSnapshot": "Butter Chicken",
        "oldPricePaise": 25000,
        "newPricePaise": 30000,
        "difference": "+₹50"
      }
    ],
    "totals": {
      "subtotalPaise": 60000,
      "grandTotalPaise": 65500
    },
    "message": "Some items in your cart have changed. Please review before checkout."
  }
}
```

**Validation**:
- ✅ Status code: 200
- ✅ `data.valid` is `false`
- ✅ `data.changes` contains price_change
- ✅ Price snapshot updated in database
- ✅ Totals recalculated with new price

---

#### **Test 4.3: Validate Cart with Unavailable Item**

**Objective**: Verify validation removes unavailable items.

**Precondition**: 
1. Cart has "Paneer Tikka"
2. Mark item as sold out

**Request**:
```powershell
$response = Invoke-RestMethod -Uri "https://api-staging.chefooz.com/api/v1/cart/validate" `
    -Method Post `
    -Headers @{ Authorization = "Bearer $JWT_TOKEN" }
```

**Expected Response** (200 OK):
```json
{
  "success": true,
  "message": "Cart validated with changes",
  "data": {
    "valid": false,
    "changes": [
      {
        "type": "unavailable",
        "itemId": "cart-item-uuid",
        "titleSnapshot": "Paneer Tikka",
        "reason": "Currently sold out"
      }
    ],
    "totals": {
      "subtotalPaise": 0,
      "grandTotalPaise": 0
    },
    "message": "Some items in your cart have changed. Please review before checkout."
  }
}
```

**Validation**:
- ✅ Status code: 200
- ✅ `data.valid` is `false`
- ✅ `data.changes` contains unavailable
- ✅ Item removed from cart
- ✅ Totals recalculated without item

---

#### **Test 4.4: Validate Cart with Removed Item**

**Objective**: Verify validation detects deleted menu items.

**Precondition**: 
1. Cart has "Naan"
2. Delete menu item from chef's menu

**Request**:
```powershell
$response = Invoke-RestMethod -Uri "https://api-staging.chefooz.com/api/v1/cart/validate" `
    -Method Post `
    -Headers @{ Authorization = "Bearer $JWT_TOKEN" }
```

**Expected Response** (200 OK):
```json
{
  "success": true,
  "message": "Cart validated with changes",
  "data": {
    "valid": false,
    "changes": [
      {
        "type": "removed",
        "itemId": "cart-item-uuid",
        "menuItemId": "menu-item-uuid",
        "titleSnapshot": "Naan",
        "reason": "Menu item no longer exists"
      }
    ],
    "totals": {
      "subtotalPaise": 0,
      "grandTotalPaise": 0
    },
    "message": "Some items in your cart have changed. Please review before checkout."
  }
}
```

**Validation**:
- ✅ Status code: 200
- ✅ `data.valid` is `false`
- ✅ `data.changes` contains removed
- ✅ Item removed from cart

---

#### **Test 4.5: Validate Empty Cart**

**Objective**: Verify validation handles empty cart gracefully.

**Precondition**: Cart is empty

**Request**:
```powershell
$response = Invoke-RestMethod -Uri "https://api-staging.chefooz.com/api/v1/cart/validate" `
    -Method Post `
    -Headers @{ Authorization = "Bearer $JWT_TOKEN" }
```

**Expected Response** (200 OK):
```json
{
  "success": true,
  "message": "Cart is valid",
  "data": {
    "valid": true,
    "changes": [],
    "totals": {
      "subtotalPaise": 0,
      "deliveryPaise": 0,
      "taxPaise": 0,
      "discountPaise": 0,
      "grandTotalPaise": 0
    },
    "message": "Cart is empty"
  }
}
```

**Validation**:
- ✅ Status code: 200
- ✅ `data.valid` is `true`
- ✅ Message is "Cart is empty"

---

### **Category 5: Creator Order Attribution Tests**

#### **Test 5.1: Add Creator Order Successfully**

**Objective**: Verify adding creator order items to cart.

**Precondition**: Creator order exists with 3 items

**Request**:
```powershell
$body = @{
    orderId = "creator-order-uuid"
    reelId = "reel-uuid"
} | ConvertTo-Json

$response = Invoke-RestMethod -Uri "https://api-staging.chefooz.com/api/v1/cart/add-creator-order" `
    -Method Post `
    -Headers @{ Authorization = "Bearer $JWT_TOKEN"; "Content-Type" = "application/json" } `
    -Body $body
```

**Expected Response** (200 OK):
```json
{
  "success": true,
  "message": "Creator order added to cart successfully",
  "data": {
    "items": [
      { "title": "Butter Chicken", "quantity": 2 },
      { "title": "Naan", "quantity": 3 }
    ],
    "skippedItems": [],
    "attribution": {
      "linkedReelId": "reel-uuid",
      "linkedCreatorOrderId": "creator-order-uuid",
      "creatorOrderValue": 45000
    },
    "cart": {
      "items": [...],
      "totals": {...},
      "itemCount": 5
    }
  }
}
```

**Validation**:
- ✅ Status code: 200
- ✅ Items added to cart
- ✅ Attribution metadata stored
- ✅ `creatorOrderValue` captured

---

#### **Test 5.2: Add Creator Order with Unavailable Items**

**Objective**: Verify partial order handling when some items unavailable.

**Precondition**: 
1. Creator order has 3 items
2. 1 item marked as sold out

**Request**:
```powershell
$body = @{
    orderId = "creator-order-uuid"
    reelId = "reel-uuid"
} | ConvertTo-Json

$response = Invoke-RestMethod -Uri "https://api-staging.chefooz.com/api/v1/cart/add-creator-order" `
    -Method Post `
    -Headers @{ Authorization = "Bearer $JWT_TOKEN"; "Content-Type" = "application/json" } `
    -Body $body
```

**Expected Response** (200 OK):
```json
{
  "success": true,
  "message": "Creator order added to cart successfully",
  "data": {
    "items": [
      { "title": "Butter Chicken", "quantity": 2 },
      { "title": "Naan", "quantity": 3 }
    ],
    "skippedItems": [
      { "name": "Paneer Tikka", "reason": "Currently sold out" }
    ],
    "attribution": {...},
    "cart": {...}
  }
}
```

**Validation**:
- ✅ Status code: 200
- ✅ Available items added
- ✅ Unavailable items in `skippedItems`
- ✅ Attribution still stored

---

#### **Test 5.3: Add Creator Order - All Items Unavailable**

**Objective**: Verify error when all creator order items unavailable.

**Precondition**: All items in creator order are unavailable

**Request**:
```powershell
$body = @{
    orderId = "creator-order-uuid"
    reelId = "reel-uuid"
} | ConvertTo-Json

try {
    $response = Invoke-RestMethod -Uri "https://api-staging.chefooz.com/api/v1/cart/add-creator-order" `
        -Method Post `
        -Headers @{ Authorization = "Bearer $JWT_TOKEN"; "Content-Type" = "application/json" } `
        -Body $body
} catch {
    $errorResponse = $_.ErrorDetails.Message | ConvertFrom-Json
}
```

**Expected Response** (400 Bad Request):
```json
{
  "success": false,
  "message": "All items in creator order are unavailable",
  "errorCode": "NO_ITEMS_AVAILABLE"
}
```

**Validation**:
- ✅ Status code: 400
- ✅ `errorCode` is "NO_ITEMS_AVAILABLE"

---

#### **Test 5.4: Add Creator Order - Order Not Found**

**Objective**: Verify error when creator order doesn't exist.

**Request**:
```powershell
$body = @{
    orderId = "non-existent-uuid"
    reelId = "reel-uuid"
} | ConvertTo-Json

try {
    $response = Invoke-RestMethod -Uri "https://api-staging.chefooz.com/api/v1/cart/add-creator-order" `
        -Method Post `
        -Headers @{ Authorization = "Bearer $JWT_TOKEN"; "Content-Type" = "application/json" } `
        -Body $body
} catch {
    $errorResponse = $_.ErrorDetails.Message | ConvertFrom-Json
}
```

**Expected Response** (404 Not Found):
```json
{
  "success": false,
  "message": "Order not found",
  "errorCode": "ORDER_NOT_FOUND"
}
```

**Validation**:
- ✅ Status code: 404
- ✅ `errorCode` is "ORDER_NOT_FOUND"

---

### **Category 6: Merge Cart Tests**

#### **Test 6.1: Merge Local Cart into Empty Server Cart**

**Objective**: Verify merging local cart when server cart is empty.

**Precondition**: Server cart is empty

**Request**:
```powershell
$body = @{
    items = @(
        @{
            menuItemId = "menu-item-uuid-1"
            quantity = 2
            options = @{ size = "large" }
        },
        @{
            menuItemId = "menu-item-uuid-2"
            quantity = 1
        }
    )
} | ConvertTo-Json -Depth 3

$response = Invoke-RestMethod -Uri "https://api-staging.chefooz.com/api/v1/cart/merge" `
    -Method Post `
    -Headers @{ Authorization = "Bearer $JWT_TOKEN"; "Content-Type" = "application/json" } `
    -Body $body
```

**Expected Response** (200 OK):
```json
{
  "success": true,
  "message": "Cart merged successfully",
  "data": {
    "items": [
      { "menuItemId": "menu-item-uuid-1", "quantity": 2 },
      { "menuItemId": "menu-item-uuid-2", "quantity": 1 }
    ],
    "itemCount": 3
  }
}
```

**Validation**:
- ✅ Status code: 200
- ✅ All local items added to cart
- ✅ `itemCount` matches total

---

#### **Test 6.2: Merge Local Cart into Existing Server Cart (Same Chef)**

**Objective**: Verify merging combines local + server carts when same chef.

**Precondition**: 
1. Server cart has "Butter Chicken" × 1 (Chef A)
2. Local cart has "Naan" × 2 (Chef A)

**Request**:
```powershell
$body = @{
    items = @(
        @{
            menuItemId = "naan-uuid"
            quantity = 2
        }
    )
} | ConvertTo-Json -Depth 3

$response = Invoke-RestMethod -Uri "https://api-staging.chefooz.com/api/v1/cart/merge" `
    -Method Post `
    -Headers @{ Authorization = "Bearer $JWT_TOKEN"; "Content-Type" = "application/json" } `
    -Body $body
```

**Expected Response** (200 OK):
```json
{
  "success": true,
  "message": "Cart merged successfully",
  "data": {
    "items": [
      { "titleSnapshot": "Butter Chicken", "quantity": 1 },
      { "titleSnapshot": "Naan", "quantity": 2 }
    ],
    "itemCount": 3
  }
}
```

**Validation**:
- ✅ Status code: 200
- ✅ Server cart items preserved
- ✅ Local cart items added
- ✅ Total items combined

---

#### **Test 6.3: Merge Local Cart - Chef Mismatch**

**Objective**: Verify merge fails when local cart has different chef.

**Precondition**: 
1. Server cart has items from Chef A
2. Local cart has items from Chef B

**Request**:
```powershell
$body = @{
    items = @(
        @{
            menuItemId = "chef-b-item-uuid"
            quantity = 1
        }
    )
} | ConvertTo-Json -Depth 3

try {
    $response = Invoke-RestMethod -Uri "https://api-staging.chefooz.com/api/v1/cart/merge" `
        -Method Post `
        -Headers @{ Authorization = "Bearer $JWT_TOKEN"; "Content-Type" = "application/json" } `
        -Body $body
} catch {
    $errorResponse = $_.ErrorDetails.Message | ConvertFrom-Json
}
```

**Expected Response** (409 Conflict):
```json
{
  "success": false,
  "message": "Cart can only contain items from one chef. Clear your cart to add items from a different chef.",
  "errorCode": "CART_CHEF_MISMATCH",
  "error": {
    "currentChefId": "chef-uuid-1",
    "currentChefName": "Mumbai Masala Kitchen",
    "newChefId": "chef-uuid-2",
    "newChefName": "Delhi Delights"
  }
}
```

**Validation**:
- ✅ Status code: 409
- ✅ `errorCode` is "CART_CHEF_MISMATCH"
- ✅ Server cart unchanged

---

#### **Test 6.4: Merge Empty Local Cart**

**Objective**: Verify merging empty local cart doesn't change server cart.

**Precondition**: Server cart has 2 items

**Request**:
```powershell
$body = @{
    items = @()
} | ConvertTo-Json -Depth 3

$response = Invoke-RestMethod -Uri "https://api-staging.chefooz.com/api/v1/cart/merge" `
    -Method Post `
    -Headers @{ Authorization = "Bearer $JWT_TOKEN"; "Content-Type" = "application/json" } `
    -Body $body
```

**Expected Response** (200 OK):
```json
{
  "success": true,
  "message": "Cart merged successfully",
  "data": {
    "items": [
      { "quantity": 2 }
    ],
    "itemCount": 2
  }
}
```

**Validation**:
- ✅ Status code: 200
- ✅ Server cart unchanged
- ✅ No items added

---

### **Category 7: Checkout Tests**

#### **Test 7.1: Checkout Cart Successfully**

**Objective**: Verify checkout creates order and clears cart.

**Precondition**: Cart has 3 items

**Request**:
```powershell
$body = @{
    addressId = "address-uuid"
} | ConvertTo-Json

$response = Invoke-RestMethod -Uri "https://api-staging.chefooz.com/api/v1/cart/checkout" `
    -Method Post `
    -Headers @{ Authorization = "Bearer $JWT_TOKEN"; "Content-Type" = "application/json" } `
    -Body $body
```

**Expected Response** (200 OK):
```json
{
  "success": true,
  "message": "Order created successfully",
  "data": {
    "orderId": "order-uuid",
    "status": "draft",
    "grandTotalPaise": 59700
  }
}
```

**Validation**:
- ✅ Status code: 200
- ✅ Order created in database
- ✅ Cart cleared (GET /cart returns empty)
- ✅ Attribution cleared

---

#### **Test 7.2: Checkout Empty Cart**

**Objective**: Verify checkout fails with empty cart.

**Precondition**: Cart is empty

**Request**:
```powershell
$body = @{
    addressId = "address-uuid"
} | ConvertTo-Json

try {
    $response = Invoke-RestMethod -Uri "https://api-staging.chefooz.com/api/v1/cart/checkout" `
        -Method Post `
        -Headers @{ Authorization = "Bearer $JWT_TOKEN"; "Content-Type" = "application/json" } `
        -Body $body
} catch {
    $errorResponse = $_.ErrorDetails.Message | ConvertFrom-Json
}
```

**Expected Response** (400 Bad Request):
```json
{
  "success": false,
  "message": "Cart is empty",
  "errorCode": "CART_EMPTY"
}
```

**Validation**:
- ✅ Status code: 400
- ✅ `errorCode` is "CART_EMPTY"

---

#### **Test 7.3: Checkout with Invalid Cart (Validation Failed)**

**Objective**: Verify checkout fails if cart validation fails.

**Precondition**: Cart has unavailable items

**Request**:
```powershell
$body = @{
    addressId = "address-uuid"
} | ConvertTo-Json

try {
    $response = Invoke-RestMethod -Uri "https://api-staging.chefooz.com/api/v1/cart/checkout" `
        -Method Post `
        -Headers @{ Authorization = "Bearer $JWT_TOKEN"; "Content-Type" = "application/json" } `
        -Body $body
} catch {
    $errorResponse = $_.ErrorDetails.Message | ConvertFrom-Json
}
```

**Expected Response** (400 Bad Request):
```json
{
  "success": false,
  "message": "Cart validation failed. Please review changes.",
  "errorCode": "CART_INVALID"
}
```

**Validation**:
- ✅ Status code: 400
- ✅ `errorCode` is "CART_INVALID"

---

#### **Test 7.4: Concurrent Checkout Prevention**

**Objective**: Verify concurrent checkouts are prevented via locking.

**Precondition**: Cart has items

**Request** (Run 2 simultaneous checkouts):
```powershell
# Start 2 checkout requests in parallel
$job1 = Start-Job -ScriptBlock {
    $body = @{ addressId = "address-uuid" } | ConvertTo-Json
    Invoke-RestMethod -Uri "https://api-staging.chefooz.com/api/v1/cart/checkout" `
        -Method Post `
        -Headers @{ Authorization = "Bearer $env:JWT_TOKEN"; "Content-Type" = "application/json" } `
        -Body $body
}

$job2 = Start-Job -ScriptBlock {
    $body = @{ addressId = "address-uuid" } | ConvertTo-Json
    Invoke-RestMethod -Uri "https://api-staging.chefooz.com/api/v1/cart/checkout" `
        -Method Post `
        -Headers @{ Authorization = "Bearer $env:JWT_TOKEN"; "Content-Type" = "application/json" } `
        -Body $body
}

Wait-Job $job1, $job2
$result1 = Receive-Job $job1
$result2 = Receive-Job $job2
```

**Expected Behavior**:
- First request: 200 OK (order created)
- Second request: 400 Bad Request (checkout in progress)

**Validation**:
- ✅ Only 1 order created
- ✅ Second request gets "CHECKOUT_IN_PROGRESS" error

---

### **Category 8: Caching Tests**

#### **Test 8.1: Cart Cache Hit**

**Objective**: Verify cart caching works correctly.

**Precondition**: Cart has items

**Steps**:
1. GET /cart (cache miss, DB query)
2. GET /cart immediately (cache hit)

**Validation**:
- ✅ First request: ~100ms (DB query)
- ✅ Second request: ~45ms (cache hit)
- ✅ Both return same data

---

#### **Test 8.2: Cart Count Cache**

**Objective**: Verify cart count cached for 5 seconds.

**Precondition**: Cart has items

**Steps**:
1. GET /cart/count (cache miss)
2. GET /cart/count within 5 seconds (cache hit)
3. Wait 6 seconds
4. GET /cart/count (cache miss, refetch)

**Validation**:
- ✅ Request 2: Cache hit
- ✅ Request 4: Cache miss (TTL expired)

---

#### **Test 8.3: Cache Invalidation on Add**

**Objective**: Verify cache invalidated when cart modified.

**Steps**:
1. GET /cart (cache loaded)
2. POST /cart/items (add item)
3. GET /cart (cache invalidated, fresh data)

**Validation**:
- ✅ Request 3 returns updated cart
- ✅ New item present

---

#### **Test 8.4: Graceful Degradation (Cache Unavailable)**

**Objective**: Verify cart works when Valkey down.

**Precondition**: Stop Valkey service

**Request**:
```powershell
$response = Invoke-RestMethod -Uri "https://api-staging.chefooz.com/api/v1/cart" `
    -Method Get `
    -Headers @{ Authorization = "Bearer $JWT_TOKEN" }
```

**Expected Response** (200 OK):
```json
{
  "success": true,
  "message": "Cart retrieved successfully",
  "data": {
    "items": [...],
    "totals": {...}
  }
}
```

**Validation**:
- ✅ Status code: 200
- ✅ Cart retrieved from PostgreSQL
- ✅ No application errors

---

## 📊 **Performance Tests**

### **Test P1: Cart Retrieval Performance**

**Objective**: Verify GET /cart response time < 100ms (p95).

**Test Steps**:
1. Add 10 items to cart
2. Run 100 GET /cart requests
3. Measure response times

**Expected Results**:
- p50: < 50ms
- p95: < 100ms
- p99: < 150ms

**PowerShell Script**:
```powershell
$times = @()
1..100 | ForEach-Object {
    $start = Get-Date
    Invoke-RestMethod -Uri "https://api-staging.chefooz.com/api/v1/cart" `
        -Method Get `
        -Headers @{ Authorization = "Bearer $JWT_TOKEN" }
    $end = Get-Date
    $times += ($end - $start).TotalMilliseconds
}

$p50 = ($times | Sort-Object)[[math]::Floor($times.Count * 0.5)]
$p95 = ($times | Sort-Object)[[math]::Floor($times.Count * 0.95)]
$p99 = ($times | Sort-Object)[[math]::Floor($times.Count * 0.99)]

Write-Host "p50: $p50 ms"
Write-Host "p95: $p95 ms"
Write-Host "p99: $p99 ms"
```

---

### **Test P2: Add Item Performance**

**Objective**: Verify POST /cart/items response time < 200ms (p95).

**Test Steps**:
1. Run 100 add item requests
2. Measure response times

**Expected Results**:
- p50: < 100ms
- p95: < 200ms
- p99: < 300ms

---

### **Test P3: Cart Count Performance**

**Objective**: Verify GET /cart/count response time < 50ms (p95).

**Expected Results**:
- p50: < 25ms
- p95: < 50ms
- p99: < 75ms

---

### **Test P4: Validation Performance**

**Objective**: Verify POST /cart/validate response time < 300ms (p95).

**Test Steps**:
1. Cart with 10 items
2. Run validation 100 times

**Expected Results**:
- p50: < 150ms
- p95: < 300ms
- p99: < 500ms

---

## 🤖 **PowerShell Test Automation**

### **Complete Test Suite Runner**

```powershell
# cart-test-suite.ps1
# Comprehensive automated test suite for Cart Module

param(
    [string]$baseUrl = "https://api-staging.chefooz.com/api/v1",
    [string]$jwtToken = $env:JWT_TOKEN
)

# Test results
$totalTests = 0
$passedTests = 0
$failedTests = 0

function Test-Endpoint {
    param(
        [string]$name,
        [scriptblock]$test
    )
    
    $global:totalTests++
    Write-Host "`n🧪 Test: $name" -ForegroundColor Cyan
    
    try {
        & $test
        Write-Host "✅ PASSED" -ForegroundColor Green
        $global:passedTests++
    } catch {
        Write-Host "❌ FAILED: $_" -ForegroundColor Red
        $global:failedTests++
    }
}

function Assert-Equal {
    param($actual, $expected, $message)
    if ($actual -ne $expected) {
        throw "$message`nExpected: $expected`nActual: $actual"
    }
}

function Assert-True {
    param($condition, $message)
    if (-not $condition) {
        throw $message
    }
}

# Test 1: Get Empty Cart
Test-Endpoint "Get Empty Cart" {
    # Clear cart first
    Invoke-RestMethod -Uri "$baseUrl/cart/clear" -Method Post -Headers @{
        Authorization = "Bearer $jwtToken"
    } | Out-Null
    
    # Get cart
    $response = Invoke-RestMethod -Uri "$baseUrl/cart" -Method Get -Headers @{
        Authorization = "Bearer $jwtToken"
    }
    
    Assert-True $response.success "Response success should be true"
    Assert-Equal $response.data.items.Count 0 "Cart should be empty"
    Assert-Equal $response.data.itemCount 0 "Item count should be 0"
}

# Test 2: Add Item to Cart
Test-Endpoint "Add Item to Cart" {
    $body = @{
        menuItemId = $env:TEST_MENU_ITEM_ID
        quantity = 2
        options = @{ size = "large" }
    } | ConvertTo-Json
    
    $response = Invoke-RestMethod -Uri "$baseUrl/cart/items" -Method Post -Headers @{
        Authorization = "Bearer $jwtToken"
        "Content-Type" = "application/json"
    } -Body $body
    
    Assert-True $response.success "Response success should be true"
    Assert-Equal $response.data.items.Count 1 "Cart should have 1 item"
    Assert-Equal $response.data.items[0].quantity 2 "Item quantity should be 2"
}

# Test 3: Increment Item Quantity
Test-Endpoint "Increment Item Quantity (Same Options)" {
    $body = @{
        menuItemId = $env:TEST_MENU_ITEM_ID
        quantity = 1
        options = @{ size = "large" }
    } | ConvertTo-Json
    
    $response = Invoke-RestMethod -Uri "$baseUrl/cart/items" -Method Post -Headers @{
        Authorization = "Bearer $jwtToken"
        "Content-Type" = "application/json"
    } -Body $body
    
    Assert-Equal $response.data.items.Count 1 "Should still have 1 cart item"
    Assert-Equal $response.data.items[0].quantity 3 "Quantity should be incremented to 3"
}

# Test 4: Get Cart Count
Test-Endpoint "Get Cart Count" {
    $response = Invoke-RestMethod -Uri "$baseUrl/cart/count" -Method Get -Headers @{
        Authorization = "Bearer $jwtToken"
    }
    
    Assert-Equal $response.data.count 3 "Cart count should be 3"
}

# Test 5: Update Item Quantity
Test-Endpoint "Update Item Quantity" {
    $cartResponse = Invoke-RestMethod -Uri "$baseUrl/cart" -Method Get -Headers @{
        Authorization = "Bearer $jwtToken"
    }
    $cartItemId = $cartResponse.data.items[0].id
    
    $body = @{ quantity = 5 } | ConvertTo-Json
    
    $response = Invoke-RestMethod -Uri "$baseUrl/cart/items/$cartItemId" -Method Patch -Headers @{
        Authorization = "Bearer $jwtToken"
        "Content-Type" = "application/json"
    } -Body $body
    
    Assert-Equal $response.data.items[0].quantity 5 "Quantity should be updated to 5"
}

# Test 6: Validate Cart
Test-Endpoint "Validate Cart" {
    $response = Invoke-RestMethod -Uri "$baseUrl/cart/validate" -Method Post -Headers @{
        Authorization = "Bearer $jwtToken"
    }
    
    Assert-True $response.data.valid "Cart should be valid"
    Assert-Equal $response.data.changes.Count 0 "No changes should be detected"
}

# Test 7: Remove Item
Test-Endpoint "Remove Item" {
    $cartResponse = Invoke-RestMethod -Uri "$baseUrl/cart" -Method Get -Headers @{
        Authorization = "Bearer $jwtToken"
    }
    $cartItemId = $cartResponse.data.items[0].id
    
    $response = Invoke-RestMethod -Uri "$baseUrl/cart/items/$cartItemId" -Method Delete -Headers @{
        Authorization = "Bearer $jwtToken"
    }
    
    Assert-Equal $response.data.items.Count 0 "Cart should be empty after removal"
}

# Test 8: Clear Cart
Test-Endpoint "Clear Cart" {
    # Add items first
    $body = @{
        menuItemId = $env:TEST_MENU_ITEM_ID
        quantity = 2
    } | ConvertTo-Json
    
    Invoke-RestMethod -Uri "$baseUrl/cart/items" -Method Post -Headers @{
        Authorization = "Bearer $jwtToken"
        "Content-Type" = "application/json"
    } -Body $body | Out-Null
    
    # Clear cart
    $response = Invoke-RestMethod -Uri "$baseUrl/cart/clear" -Method Post -Headers @{
        Authorization = "Bearer $jwtToken"
    }
    
    Assert-Equal $response.data.items.Count 0 "Cart should be empty"
    Assert-Equal $response.data.itemCount 0 "Item count should be 0"
}

# Test Summary
Write-Host "`n" + ("=" * 50) -ForegroundColor Yellow
Write-Host "Test Summary" -ForegroundColor Yellow
Write-Host ("=" * 50) -ForegroundColor Yellow
Write-Host "Total Tests: $totalTests"
Write-Host "Passed: $passedTests" -ForegroundColor Green
Write-Host "Failed: $failedTests" -ForegroundColor Red
Write-Host "Success Rate: $([math]::Round($passedTests / $totalTests * 100, 2))%"

if ($failedTests -gt 0) {
    exit 1
}
```

**Usage**:
```powershell
# Set environment variables
$env:JWT_TOKEN = "your-jwt-token"
$env:TEST_MENU_ITEM_ID = "menu-item-uuid"

# Run test suite
.\cart-test-suite.ps1
```

---

## 📋 **Test Checklist**

### **Functional Tests**

- [ ] Get empty cart
- [ ] Get cart with items
- [ ] Get cart count (empty)
- [ ] Get cart count (with items)
- [ ] Add item to empty cart
- [ ] Add item with same options (quantity increment)
- [ ] Add item with different options (new cart item)
- [ ] Add item with customizations
- [ ] Add item from different chef (409 error)
- [ ] Add invalid menu item (404 error)
- [ ] Add unavailable menu item (400 error)
- [ ] Update item quantity
- [ ] Update quantity to 0 (remove)
- [ ] Remove item via DELETE
- [ ] Remove non-existent item (404 error)
- [ ] Clear cart
- [ ] Validate valid cart
- [ ] Validate cart with price change
- [ ] Validate cart with unavailable item
- [ ] Validate cart with removed item
- [ ] Validate empty cart
- [ ] Add creator order successfully
- [ ] Add creator order with unavailable items
- [ ] Add creator order - all items unavailable (400 error)
- [ ] Add creator order - order not found (404 error)
- [ ] Merge local cart into empty server cart
- [ ] Merge local cart into existing server cart (same chef)
- [ ] Merge local cart - chef mismatch (409 error)
- [ ] Merge empty local cart
- [ ] Checkout cart successfully
- [ ] Checkout empty cart (400 error)
- [ ] Checkout invalid cart (400 error)
- [ ] Concurrent checkout prevention

### **Caching Tests**

- [ ] Cart cache hit
- [ ] Cart count cache (5-second TTL)
- [ ] Cache invalidation on add
- [ ] Cache invalidation on update
- [ ] Cache invalidation on remove
- [ ] Graceful degradation (Valkey down)

### **Performance Tests**

- [ ] GET /cart < 100ms (p95)
- [ ] POST /cart/items < 200ms (p95)
- [ ] GET /cart/count < 50ms (p95)
- [ ] POST /cart/validate < 300ms (p95)

### **Integration Tests**

- [ ] OrderService integration (checkout creates order)
- [ ] PricingService integration (totals calculated correctly)
- [ ] ChefMenuItem integration (menu item validation)

---

**[CART_QA_TEST_CASES_COMPLETE ✅]**

*For business overview, see `01_FEATURE_OVERVIEW.md`. For implementation details, see `02_TECHNICAL_GUIDE.md`.*

---

### TC-CART-BUG-001: Empty Cart 'Explore Reels' Button Crash

**Type:** Bug Regression
**Feature area:** Cart - Empty State
**Priority:** P0

**Preconditions:**
- User is logged in
- Cart is empty (all items removed or no items added)

**Steps:**
1. Navigate to the cart screen with no items
2. Tap the "Explore Reels" button in the empty state

**Expected result:** App navigates to the Chefooz home feed tab without crashing
**Actual result (before fix):** App crashed with "undefined is not a function" error — `router.push('/chefooz')` resolved to an invalid Expo Router path
**Fix applied:** Changed route from `router.push('/chefooz')` to `router.replace('/(tabs)/chefooz')`
**Regression test:** Verify navigation via manual tap on device
**Status:** Fixed ✅

---

**Document Version**: 1.0  
**Last Updated**: March 2026  
**Module**: Cart (Week 6 - Order Flow)  
**Test Cases**: 40+ comprehensive scenarios  
**Status**: ✅ Complete

---

## Dark Mode Regression Tests (Added March 2026)

### TC-CART-DM-001: Cart Screen — All Card Backgrounds in Dark Mode

**Type:** Bug Regression / Manual  
**Feature area:** Cart screen (`app/cart/index.tsx`)  
**Priority:** P0

**Preconditions:**
- Device set to dark appearance
- Cart has at least 1 item

**Steps:**
1. Open the Chefooz app in dark mode
2. Add an item to cart from a chef's menu
3. Navigate to the Cart screen
4. Observe: container background, cart item cards, special request input, order summary card, checkout footer, address card, address bottom sheet

**Expected result:** Container is `#0A0A0A`, all cards are `#1C1C1E`, input fields are `#2C2C2E`, borders are `#3A3A3C`, text is `#F5F5F5` / `#ABABAB`  
**Actual result (before fix):** White background with white cards — completely unreadable in dark mode  
**Fix applied:** Converted `StyleSheet.create` to `makeStyles(colors)` factory pattern; all 25+ hardcoded hex values replaced with theme tokens  
**Regression test:** `apps/chefooz-app/src/app/cart/index.tsx` (makeStyles factory)  
**Status:** Fixed ✅

### TC-CART-DM-002: Cart FSSAI Expandable Section in Dark Mode

**Type:** Bug Regression / Manual  
**Feature area:** Cart — FSSAI compliance section  
**Priority:** P1

**Steps:**
1. Open cart in dark mode
2. Tap the FSSAI info toggle ("View nutritional info" or similar)
3. Observe the expanded section background and text

**Expected result:** Expanded section uses `colors.surfaceElevated` (#2C2C2E), labels use `colors.textPrimary`, values use `colors.textSecondary`  
**Actual result (before fix):** White `#FAFAFA` background with dark text — stark white block in dark mode cart  
**Fix applied:** `fssaiExpandedSection.backgroundColor: '#FAFAFA'` → `colors.surfaceElevated`  
**Status:** Fixed ✅

---

### TC-CART-DM-003: Payment Screen All Cards in Dark Mode

**Type:** Bug Regression / Manual
**Feature area:** `app/cart/payment.tsx` checkout screen
**Priority:** P1

**Preconditions:**
- Device set to dark appearance
- User has items in cart with an address selected

**Steps:**
1. Enable dark mode on device
2. Add items to cart, proceed to checkout
3. Navigate to the payment/checkout screen (`/cart/payment`)
4. Observe: all section cards (Chef Card, Address Card, Items Card, Price Card, Payment Method Card), sticky bottom bar, modal breakdowns

**Expected result:** All card backgrounds use `colors.surface` (#1C1C1E). Text follows `colors.textPrimary` / `colors.textSecondary` / `colors.textMuted`. Borders and dividers use `colors.border`. Sticky bottom bar uses `colors.surface`.
**Actual result (before fix):** All cards had `backgroundColor: '#FFF'` and all text was `'#000'`, `'#666'`, `'#999'` — entire payment screen appeared white in dark mode.
**Fix applied:** Converted static `StyleSheet.create` to `makeStyles(colors: any)` factory in `cart/payment.tsx`; added `const styles = useMemo(() => makeStyles(theme.colors), [theme.colors])` and `useMemo` import. Replaced 40+ hardcoded hex values with theme tokens.
**Regression test:** `apps/chefooz-app/src/app/cart/__tests__/payment.dark-mode.spec.ts`
**Status:** Fixed ✅

---

**Last Updated**: March 2026
