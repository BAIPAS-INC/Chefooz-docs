# 🛍️ Customer Order Journey — End-to-End Documentation

**Journey Type**: P0 Critical — Core Transaction Flow  
**Status**: ✅ COMPLETE  
**Last Updated**: 2026-02-15  
**Depends On**: Modules documented in Weeks 1-6 (All Complete)  
**Business Value**: Primary revenue generation flow — food discovery to order delivery

---

## 📋 Table of Contents

1. [Journey Overview](#journey-overview)
2. [Prerequisites & Entry Points](#prerequisites--entry-points)
3. [Step-by-Step User Flow](#step-by-step-user-flow)
4. [Complete Flow Diagram](#complete-flow-diagram)
5. [Module Integration Map](#module-integration-map)
6. [API Call Sequences](#api-call-sequences)
7. [State Transitions](#state-transitions)
8. [Error Scenarios & Recovery](#error-scenarios--recovery)
9. [Analytics & Tracking](#analytics--tracking)
10. [Performance SLAs](#performance-slas)
11. [Testing Strategy](#testing-strategy)
12. [Related Documentation](#related-documentation)

---

## Journey Overview

### 🎯 Purpose

The **Customer Order Journey** is Chefooz's primary revenue-generating flow that enables users to discover dishes, add them to their cart, complete checkout, make payment, and track their order until delivery. This journey spans 8 core modules and represents the complete transaction lifecycle from content discovery to order fulfillment.

### 📊 Business Objectives

| Objective | Metric | Target | Priority |
|-----------|--------|--------|----------|
| **Order Conversion Rate** | (Orders / Menu Views) × 100 | 8-12% | P0 |
| **Cart Abandonment Rate** | (Abandoned Carts / Total Carts) × 100 | <35% | P0 |
| **Checkout Success Rate** | (Completed Checkouts / Started) × 100 | >85% | P0 |
| **Payment Success Rate** | (Successful Payments / Attempts) × 100 | >92% | P0 |
| **End-to-End Time** | Discovery → Order Placed | <5 min | P1 |
| **Average Order Value** | Total Order Value / Orders | ₹450+ | P1 |
| **Repeat Order Rate** | Orders Using Reorder Feature | 25%+ | P2 |

### 👤 Primary User Personas

#### **Persona 1: Hungry Customer (Primary)**
**Demographics**:
- Age: 18-45 years
- Tech-savvy, mobile-first
- Budget: ₹100-1000 per order
- Location: Urban/suburban (within chef delivery radius)

**Behaviors**:
- Browses food reels during free time
- Makes ordering decisions within 2-3 minutes
- Prefers UPI Intent payments (fast, no OTP)
- Tracks order status actively post-placement
- Saves favorite dishes for future reordering

**Goals**:
- Discover delicious food quickly
- Complete checkout without friction
- Pay securely with preferred method
- Track order in real-time
- Reorder favorite dishes easily

**Pain Points**:
- Too many steps in checkout flow
- Unexpected price changes at checkout
- Payment failures without clear errors
- Unclear delivery timelines
- Forgetting what I ordered previously

#### **Persona 2: Repeat Customer**
**Demographics**:
- Existing Chefooz user (3+ orders)
- Has saved addresses and payment methods
- Follows favorite chefs

**Behaviors**:
- Uses "Reorder" feature frequently
- Expects saved address selection
- Completes checkout in <30 seconds
- Rarely browses — direct menu access
- Checks order history for coin rewards

**Goals**:
- Reorder previous meals quickly
- Skip repetitive data entry
- Fast checkout (<30s)
- Earn and redeem coins

#### **Persona 3: First-Time Customer**
**Demographics**:
- New to Chefooz platform
- May not have address saved
- Concerned about payment security

**Behaviors**:
- Explores multiple chef menus before deciding
- Reads reviews and ratings carefully
- Prefers COD for first order
- Needs guidance through checkout

**Goals**:
- Build trust with platform
- Understand delivery process
- Complete first order successfully
- Save preferences for future

### 🌟 Key Value Propositions

**For Customers**:
- 🎥 **Discover food visually** through short-form video reels
- 🛒 **Flexible cart** — add items, review, modify before checkout
- 📍 **Distance-based pricing** — transparent delivery fee calculation
- 💳 **Multiple payment options** — UPI Intent, COD, cards, wallets
- 📦 **Real-time tracking** — live order status with push notifications
- 🔄 **Quick reorder** — one-tap reorder from order history
- 🪙 **Coin rewards** — earn 100% of dish price as redeemable coins

**For Chefs**:
- ✅ **Confirmed orders** — payment guaranteed before acceptance
- 📍 **Validated addresses** — accurate delivery locations
- 💰 **Fair pricing** — distance-based delivery fees
- 📊 **Order insights** — understand customer preferences
- 🔔 **Real-time notifications** — instant order alerts

**For Platform**:
- 💸 **Revenue certainty** — commission calculated on confirmed payments
- 📉 **Reduced support** — validation catches issues before payment
- 📈 **Higher conversion** — optimized checkout reduces abandonment
- 🔐 **Payment security** — Razorpay integration with fraud detection
- 📊 **Data insights** — track conversion funnels and drop-off points

---

## Prerequisites & Entry Points

### 🔑 Prerequisites

#### **Required for Cart Addition**:
- ✅ User must be logged in (JWT authentication)
- ✅ Chef must be online (`isOnline = true`)
- ✅ Menu item must be available (`isAvailable = true`)
- ✅ User must be within chef's delivery radius

#### **Required for Checkout**:
- ✅ Cart must contain at least one item
- ✅ Cart must belong to single chef (no multi-chef orders)
- ✅ User must have at least one saved address

#### **Required for Payment**:
- ✅ Checkout must be validated successfully
- ✅ Delivery address must be within chef's radius
- ✅ Final order total must be calculated
- ✅ Payment method must be selected (UPI Intent, COD, Cards)

#### **Optional Enhancements**:
- ⭕ Saved addresses (speeds up checkout)
- ⭕ Previous orders (enables reorder feature)
- ⭕ Earned coins (enables coin redemption)
- ⭕ Following chefs (enables following-only feed filter)

### 🚪 Entry Points (How Users Enter This Journey)

| Entry Point | Module | User Action | Frequency |
|-------------|--------|-------------|-----------|
| **Feed Reel** | Feed | Tap reel → Tap "Order Now" CTA → View chef menu | 40% |
| **Explore Discovery** | Explore | Browse trending dishes → Tap dish card → View menu | 25% |
| **Chef Profile** | Chef-Public | Search chef → Open profile → Browse menu | 20% |
| **Reorder** | Order | Order history → Tap "Reorder" button | 10% |
| **Direct Link** | External | Push notification, SMS, email link | 5% |

### 📱 Device & Platform Support

| Platform | Minimum Version | Support Level |
|----------|----------------|---------------|
| **iOS** | iOS 13+ | Full support |
| **Android** | Android 8+ | Full support |
| **Web (Admin)** | Modern browsers | View-only (no ordering) |

### 🌐 Network Requirements

- **Minimum Connection**: 3G/4G (2 Mbps)
- **Offline Behavior**: Cart cached locally, syncs on reconnect
- **Low Bandwidth**: Images compressed, lazy loading enabled
- **No Connection**: Show cached cart, queue API calls for retry

---

## Step-by-Step User Flow

### **Flow Summary (8 Steps)**

```
Browse Content → View Chef Menu → Add to Cart → Review Cart → 
Select Address → Confirm Checkout → Make Payment → Track Order
```

---

### **Step 1: Browse Content (Feed/Explore)**

**Objective**: Discover food content through visual reels or curated sections.

#### **User Actions**:
1. Open Chefooz app
2. Land on **Feed Tab** (default) or tap **Explore Tab**
3. Scroll through short-form video reels (Feed) or curated sections (Explore)
4. Watch reel to see dish preparation, plating, chef personality

#### **System Behavior**:
- **Feed Module**: Fetches ranked feed with cursor-based pagination
  - Applies ranking algorithm: engagement × recency × CRS reputation boost
  - First page cached for 5 minutes
  - Includes following-only filter if enabled
- **Explore Module**: Returns 5 curated sections (Trending, Recommended, New Dishes, Exclusive, Promoted)
  - Each section limited to 5 items for consolidated view
  - "See All" expands to full paginated list
- **Reel Data**: Each reel includes:
  - Video URL, thumbnail, title, description
  - Chef info: name, avatar, rating, delivery ETA
  - Menu item overlay: price, availability, "Order Now" CTA
  - Engagement metrics: likes, views, saves

#### **Key APIs**:
```
GET /api/v1/feed?limit=20&cursor=<token>
GET /api/v1/explore/consolidated
GET /api/v1/explore/sections/<sectionName>?limit=20&cursor=<token>
```

#### **Validation Rules**:
- ✅ Only show reels with processed video (`videoUrl` not null)
- ✅ Exclude `MENU_SHOWCASE` reels if feature flag disabled
- ✅ Hide reels from blocked/deactivated users
- ✅ Hide reels from chefs outside delivery radius (if location known)

#### **User Decision Point**:
- **Interested in Dish?** → Tap reel → See **"Order Now"** button → Proceed to Step 2
- **Not Interested?** → Swipe to next reel → Continue browsing

#### **Analytics Events**:
```json
{
  "event": "reel_viewed",
  "properties": {
    "reel_id": "uuid",
    "chef_id": "uuid",
    "source": "feed",
    "duration_seconds": 5
  }
}
```

#### **Performance SLA**:
- Feed fetch: <200ms (cached), <800ms (uncached)
- Video playback start: <1s (CDN delivery)

---

### **Step 2: View Chef Menu (Chef-Public)**

**Objective**: Browse chef's full menu organized by categories to select dishes for ordering.

#### **User Actions**:
1. Tap **"Order Now"** CTA from reel overlay
2. Navigate to chef's public menu page
3. Browse menu items grouped by platform categories (Starters, Mains, Desserts, etc.)
4. View item details: photo, price, description, dietary info, allergens
5. Check chef status: rating, open/closed, delivery radius, ETA

#### **System Behavior**:
- **Chef-Public Module**: Fetches chef profile + menu items
  - Groups menu items by `PlatformCategory`
  - Filters out unavailable items (`isAvailable = false`)
  - Calculates chef metrics:
    - Average rating (from Review module)
    - Total orders (count of DELIVERED orders)
    - `isOpen` status (from `ChefKitchen.isOnline`)
    - Delivery ETA (estimated minutes)
  - Includes reorder suggestions if user has previous orders with this chef
- **Menu Item Data**:
  - Name, image, price (snapshot at current time)
  - Description, dietary type (veg/non-veg/egg)
  - Allergen warnings, nutrition info
  - Availability status, out-of-stock reasons

#### **Key APIs**:
```
GET /api/v1/chefs/:chefId/public
  → Returns chef profile + menu items

GET /api/v1/chefs/:chefId/menu?category=<categoryId>
  → Returns filtered menu items by category

GET /api/v1/chefs/:chefId/reorder-suggestions?userId=<userId>
  → Returns previous orders for quick reorder
```

#### **Validation Rules**:
- ✅ Chef must be online (`isOnline = true`) to accept orders
- ✅ Menu items must be available (`isAvailable = true`)
- ✅ User must be within chef's delivery radius (checked at checkout)
- ✅ If chef offline: show "Currently Closed" banner with next opening time

#### **User Decision Points**:
- **Want to Order?** → Tap item → See "Add to Cart" button → Proceed to Step 3
- **Not Interested?** → Back to feed/explore → Continue browsing
- **Reorder Previous?** → Tap "Reorder" button → Auto-add to cart → Skip to Step 4

#### **Analytics Events**:
```json
{
  "event": "chef_menu_viewed",
  "properties": {
    "chef_id": "uuid",
    "source": "reel_cta",
    "menu_items_count": 15,
    "is_online": true
  }
}
```

#### **Performance SLA**:
- Chef profile fetch: <300ms
- Menu items fetch: <500ms
- Reorder suggestions: <200ms (cached)

---

### **Step 3: Add to Cart (Cart)**

**Objective**: Add selected menu items to persistent server-side cart for later checkout.

#### **User Actions**:
1. Tap **"Add to Cart"** button on menu item
2. (Optional) Adjust quantity using +/- buttons
3. (Optional) Add special instructions (e.g., "Extra spicy", "No onions")
4. Confirm addition
5. See cart badge update with item count
6. Continue browsing or proceed to cart review

#### **System Behavior**:
- **Cart Module**: Server-side persistent cart management
  - **Single-Chef Enforcement**: Cart scoped to one chef only
    - If cart empty → Add item, associate cart with chef
    - If cart has items from same chef → Add item successfully
    - If cart has items from different chef → Return `409 Conflict`
      - Frontend shows modal: "Replace cart with items from [New Chef]?"
      - User chooses: "Keep Current" or "Start Fresh"
  - **Price Snapshot**: Capture current price, title, image at add time
    - Protects user from unexpected price increases
    - Validation at checkout compares snapshot vs. current price
  - **Cross-Device Sync**: Cart stored in PostgreSQL `cart_items` table
    - Valkey cache (15-minute TTL) for fast reads
    - Syncs across mobile, web, any device with same user JWT
  - **Quantity Limits**: Max 10 of same item per cart (configurable)

#### **Key APIs**:
```
POST /api/v1/cart/items
Request Body:
{
  "chefMenuItemId": "uuid",
  "quantity": 2,
  "specialInstructions": "Extra spicy"
}

Response (200 OK):
{
  "success": true,
  "message": "Item added to cart",
  "data": {
    "cartId": "uuid",
    "chefId": "uuid",
    "items": [
      {
        "cartItemId": "uuid",
        "menuItemId": "uuid",
        "title": "Chicken Biryani",
        "price": 350,
        "quantity": 2,
        "priceSnapshot": 350,
        "titleSnapshot": "Chicken Biryani",
        "imageSnapshot": "https://cdn.chefooz.com/..."
      }
    ],
    "itemCount": 2,
    "subtotal": 700
  }
}

Response (409 Conflict - Different Chef):
{
  "success": false,
  "message": "Cart contains items from different chef",
  "errorCode": "CART_DIFFERENT_CHEF",
  "data": {
    "currentChef": {
      "chefId": "uuid",
      "kitchenName": "Golden Spoon"
    },
    "attemptedChef": {
      "chefId": "uuid",
      "kitchenName": "Tasty Bites"
    }
  }
}
```

#### **Validation Rules**:
- ✅ User must be authenticated (JWT required)
- ✅ Menu item must exist and be available
- ✅ Chef must be online (`isOnline = true`)
- ✅ Quantity must be 1-10
- ✅ Single-chef cart enforcement
- ✅ Price snapshot captured at add time

#### **User Decision Points**:
- **Cart Conflict (Different Chef)?**
  - **Keep Current Cart** → Cancel new addition, stay on current chef menu
  - **Start Fresh Cart** → Clear cart, add new item, switch to new chef
- **Continue Shopping?** → Browse more items → Repeat Step 3
- **Ready to Checkout?** → Tap cart icon → Proceed to Step 4

#### **Analytics Events**:
```json
{
  "event": "item_added_to_cart",
  "properties": {
    "menu_item_id": "uuid",
    "chef_id": "uuid",
    "quantity": 2,
    "price_snapshot": 350,
    "cart_item_count": 2,
    "subtotal": 700
  }
}

{
  "event": "cart_conflict_occurred",
  "properties": {
    "current_chef_id": "uuid",
    "attempted_chef_id": "uuid",
    "user_action": "kept_current" // or "started_fresh"
  }
}
```

#### **Performance SLA**:
- Add to cart API: <200ms
- Cart badge update: <100ms (cached count)

---

### **Step 4: Review Cart (Cart)**

**Objective**: Review cart items, adjust quantities, see subtotal, and proceed to checkout.

#### **User Actions**:
1. Tap **cart icon** from top navigation (shows badge count)
2. View all cart items with:
   - Item thumbnails, titles, prices
   - Quantity controls (+/-)
   - Special instructions
   - Remove item button (trash icon)
3. Review cart summary:
   - Subtotal (sum of all item prices × quantities)
   - Chef info: kitchen name, avatar, delivery ETA
4. Adjust quantities or remove items as needed
5. Tap **"Proceed to Checkout"** button

#### **System Behavior**:
- **Cart Module**: Retrieves full cart with enriched data
  - Fetches cart from Valkey cache (15-min TTL) or PostgreSQL
  - Enriches with current menu item data:
    - Current price vs. snapshot price (highlight if changed)
    - Current availability (warn if item now unavailable)
    - Current chef status (warn if chef now offline)
  - Calculates subtotal: `Σ(quantity × priceSnapshot)`
  - Returns chef info for context (kitchen name, ETA, avatar)
- **Quantity Update**: Optimistic UI update, then API call
- **Item Removal**: Soft delete (mark as removed), recalculate subtotal

#### **Key APIs**:
```
GET /api/v1/cart
Response (200 OK):
{
  "success": true,
  "data": {
    "cartId": "uuid",
    "chefId": "uuid",
    "chef": {
      "kitchenName": "Golden Spoon",
      "avatar": "https://cdn.chefooz.com/...",
      "etaMinutes": 30,
      "isOnline": true
    },
    "items": [
      {
        "cartItemId": "uuid",
        "menuItemId": "uuid",
        "title": "Chicken Biryani",
        "image": "https://cdn.chefooz.com/...",
        "priceSnapshot": 350,
        "currentPrice": 350,
        "quantity": 2,
        "specialInstructions": "Extra spicy",
        "isAvailable": true,
        "priceChanged": false
      }
    ],
    "itemCount": 2,
    "subtotal": 700
  }
}

PATCH /api/v1/cart/items/:cartItemId
Request Body:
{
  "quantity": 3  // Update quantity
}

DELETE /api/v1/cart/items/:cartItemId
  → Removes item from cart
```

#### **Validation Rules**:
- ✅ Cart must contain at least 1 item to proceed to checkout
- ✅ All items must still be available (warn if unavailable)
- ✅ Chef must still be online (warn if offline)
- ✅ Price changes highlighted but don't block checkout (user informed)

#### **User Decision Points**:
- **Price Changed?**
  - Show alert: "Price changed from ₹350 to ₹380. Updated cart total."
  - User can accept (proceed) or remove item
- **Item Unavailable?**
  - Show alert: "Chicken Biryani is no longer available."
  - Auto-remove from cart (with undo option)
- **Chef Offline?**
  - Show banner: "Golden Spoon is currently closed. You can save cart for later."
  - Disable checkout button until chef comes online
- **Ready to Checkout?** → Tap "Proceed to Checkout" → Proceed to Step 5
- **Continue Shopping?** → Back to chef menu → Repeat Step 3

#### **Analytics Events**:
```json
{
  "event": "cart_viewed",
  "properties": {
    "cart_id": "uuid",
    "chef_id": "uuid",
    "item_count": 2,
    "subtotal": 700
  }
}

{
  "event": "cart_item_quantity_changed",
  "properties": {
    "cart_item_id": "uuid",
    "old_quantity": 2,
    "new_quantity": 3
  }
}

{
  "event": "cart_item_removed",
  "properties": {
    "cart_item_id": "uuid",
    "menu_item_id": "uuid"
  }
}
```

#### **Performance SLA**:
- Cart fetch: <200ms (cached), <500ms (uncached)
- Quantity update: <150ms
- Item removal: <150ms

---

### **Step 5: Select Address (Checkout)**

**Objective**: Choose delivery address and calculate distance-based delivery fee.

#### **User Actions**:
1. Tap **"Proceed to Checkout"** from cart screen
2. Navigate to **Address Selection** screen
3. See list of saved addresses (if any) with:
   - Address label (Home, Work, etc.)
   - Full address text
   - Distance from chef (calculated)
   - "Deliver Here" button
4. (Optional) Tap **"Add New Address"** to enter new address:
   - Address line 1 (required)
   - Address line 2 (optional)
   - Landmark (optional)
   - City, state, zip code
   - Save address label (Home/Work/Other)
5. Select address by tapping **"Deliver Here"**
6. System calculates delivery fee based on distance

#### **System Behavior**:
- **Checkout Module**: Address selection + distance calculation
  - Fetches user's saved addresses from `user_addresses` table
  - Calculates distance from chef's kitchen to each address:
    - Uses Haversine formula for lat/long distance
    - Rounds to 2 decimal places (e.g., 3.45 km)
  - Validates address is within chef's delivery radius:
    - If distance > `chef.deliveryRadiusKm` → Show error, block selection
    - If distance <= radius → Allow selection
  - Calculates delivery fee:
    - Base fee: ₹20 (0-3 km)
    - Distance fee: ₹10 per km beyond 3 km
    - Surge multiplier (if enabled): 1.2x-1.5x during peak hours
    - GST: 5% on delivery fee
  - Returns order preview:
    - Subtotal (from cart)
    - Delivery fee (calculated)
    - Platform fee (₹5 flat)
    - GST (5% on delivery + platform fees)
    - Grand total

#### **Key APIs**:
```
GET /api/v1/users/addresses
Response (200 OK):
{
  "success": true,
  "data": {
    "addresses": [
      {
        "addressId": "uuid",
        "label": "Home",
        "addressLine1": "123 Main St",
        "addressLine2": "Apt 4B",
        "landmark": "Near Central Park",
        "city": "Mumbai",
        "state": "Maharashtra",
        "zipCode": "400001",
        "latitude": 19.0760,
        "longitude": 72.8777,
        "isDefault": true
      }
    ]
  }
}

POST /api/v1/checkout/calculate-delivery
Request Body:
{
  "chefId": "uuid",
  "addressId": "uuid"
}

Response (200 OK):
{
  "success": true,
  "data": {
    "distanceKm": 4.5,
    "deliveryFee": 35,  // ₹20 base + ₹15 (1.5 km × ₹10)
    "surgeMultiplier": 1.0,
    "platformFee": 5,
    "gst": 2.0,  // 5% on (35 + 5)
    "subtotal": 700,
    "grandTotal": 742
  }
}

Response (400 Bad Request - Out of Radius):
{
  "success": false,
  "message": "Address is outside chef's delivery radius",
  "errorCode": "ADDRESS_OUT_OF_RADIUS",
  "data": {
    "distanceKm": 8.2,
    "maxRadiusKm": 5.0
  }
}
```

#### **Validation Rules**:
- ✅ Address must be within chef's delivery radius
- ✅ Address must have valid latitude/longitude (for distance calculation)
- ✅ Delivery fee calculated dynamically (not cached)
- ✅ Surge multiplier applied during peak hours (12pm-2pm, 7pm-10pm)
- ✅ GST calculated as 5% on (delivery fee + platform fee)

#### **User Decision Points**:
- **Address Out of Radius?**
  - Show error: "This address is 8.2 km away. Chef delivers within 5 km."
  - Options: "Add Different Address" or "Find Nearby Chefs"
- **No Saved Addresses?**
  - Show prompt: "Add your first delivery address"
  - Redirect to address form
- **Address Selected?** → See order preview with delivery fee → Proceed to Step 6

#### **Analytics Events**:
```json
{
  "event": "address_selected",
  "properties": {
    "address_id": "uuid",
    "distance_km": 4.5,
    "delivery_fee": 35
  }
}

{
  "event": "address_out_of_radius",
  "properties": {
    "distance_km": 8.2,
    "max_radius_km": 5.0,
    "chef_id": "uuid"
  }
}

{
  "event": "checkout_preview_viewed",
  "properties": {
    "subtotal": 700,
    "delivery_fee": 35,
    "platform_fee": 5,
    "gst": 2.0,
    "grand_total": 742
  }
}
```

#### **Performance SLA**:
- Address fetch: <200ms
- Distance calculation: <100ms (Haversine formula)
- Delivery fee calculation: <50ms

---

### **Step 6: Confirm Checkout (Checkout)**

**Objective**: Finalize order details, validate cart and address, prepare for payment.

#### **User Actions**:
1. Review **Checkout Summary** screen with:
   - Cart items (titles, quantities, prices)
   - Delivery address (full text + distance)
   - Price breakdown:
     - Subtotal
     - Delivery fee
     - Platform fee
     - GST
     - Grand total
2. (Optional) Add order notes (e.g., "Ring doorbell twice")
3. Select payment method:
   - UPI Intent (recommended, fast)
   - Cash on Delivery (COD)
   - Credit/Debit Card
   - Wallet (Paytm, PhonePe)
4. Tap **"Place Order"** button

#### **System Behavior**:
- **Checkout Module**: Pre-payment validation + order preparation
  - **Validate Cart**:
    - All items still available? (check `isAvailable`)
    - Chef still online? (check `isOnline`)
    - Prices still match snapshots? (compare current vs. snapshot)
    - Cart not empty?
  - **Validate Address**:
    - Still within delivery radius? (recalculate distance)
    - Valid coordinates? (latitude/longitude)
  - **Calculate Final Total**:
    - Recalculate delivery fee (distance may have changed if address updated)
    - Apply surge multiplier (if peak hour)
    - Add platform fee + GST
  - **Create Razorpay Order** (if online payment):
    - Call Razorpay Orders API with amount in paise (₹742 → 74200 paise)
    - Store Razorpay order ID in `payment_intents` table
    - Return order ID to frontend for payment UI
  - **Create Order Record** (status: `PENDING_PAYMENT`):
    - Insert into `orders` table with:
      - User ID, chef ID, cart items
      - Delivery address, distance, fees
      - Grand total, payment method
      - Status: `PENDING_PAYMENT`
    - Clear cart items (soft delete, keep for audit)

#### **Key APIs**:
```
POST /api/v1/checkout/validate
Request Body:
{
  "cartId": "uuid",
  "addressId": "uuid",
  "paymentMethod": "UPI_INTENT"
}

Response (200 OK):
{
  "success": true,
  "data": {
    "isValid": true,
    "subtotal": 700,
    "deliveryFee": 35,
    "platformFee": 5,
    "gst": 2.0,
    "grandTotal": 742,
    "razorpayOrderId": "order_abc123",  // Only for online payment
    "orderPreview": {
      "items": [...],
      "deliveryAddress": {...},
      "distanceKm": 4.5
    }
  }
}

Response (400 Bad Request - Validation Failed):
{
  "success": false,
  "message": "Checkout validation failed",
  "errorCode": "CHECKOUT_VALIDATION_FAILED",
  "data": {
    "errors": [
      {
        "field": "cart.items[0]",
        "message": "Chicken Biryani is no longer available"
      },
      {
        "field": "chef",
        "message": "Chef is currently offline"
      }
    ]
  }
}

POST /api/v1/orders/create
Request Body:
{
  "cartId": "uuid",
  "addressId": "uuid",
  "paymentMethod": "UPI_INTENT",
  "razorpayOrderId": "order_abc123",
  "orderNotes": "Ring doorbell twice"
}

Response (201 Created):
{
  "success": true,
  "message": "Order created successfully",
  "data": {
    "orderId": "uuid",
    "orderNumber": "CHF-20260215-0123",
    "status": "PENDING_PAYMENT",
    "grandTotal": 742,
    "razorpayOrderId": "order_abc123"
  }
}
```

#### **Validation Rules**:
- ✅ Cart must not be empty
- ✅ All cart items must be available
- ✅ Chef must be online
- ✅ Address must be within delivery radius
- ✅ Prices must match (or user notified of changes)
- ✅ Payment method must be valid
- ✅ Razorpay order created successfully (online payments only)

#### **User Decision Points**:
- **Validation Failed?**
  - Show errors: "Chicken Biryani is no longer available. Please review cart."
  - Redirect back to cart → User removes unavailable item → Retry
- **Chef Offline?**
  - Show error: "Chef is currently offline. Please try again later."
  - Options: "Save Cart" or "Find Similar Chefs"
- **Validation Passed?** → Redirect to payment UI → Proceed to Step 7

#### **Analytics Events**:
```json
{
  "event": "checkout_initiated",
  "properties": {
    "cart_id": "uuid",
    "chef_id": "uuid",
    "subtotal": 700,
    "grand_total": 742,
    "payment_method": "UPI_INTENT"
  }
}

{
  "event": "checkout_validation_failed",
  "properties": {
    "errors": [
      {"field": "cart.items[0]", "message": "Item unavailable"}
    ]
  }
}

{
  "event": "order_created",
  "properties": {
    "order_id": "uuid",
    "order_number": "CHF-20260215-0123",
    "status": "PENDING_PAYMENT",
    "grand_total": 742
  }
}
```

#### **Performance SLA**:
- Checkout validation: <500ms
- Razorpay order creation: <1s
- Order record creation: <300ms

---

### **Step 7: Make Payment (Payment)**

**Objective**: Complete payment securely and verify transaction.

#### **User Actions**:
1. **Redirected to Razorpay Payment UI** (for online payment):
   - See payment methods: UPI Intent, Cards, Wallets, Net Banking
   - (Recommended) Select **UPI Intent**:
     - See UPI apps installed: Google Pay, PhonePe, Paytm, etc.
     - Tap preferred app → App opens with payment request
     - Enter UPI PIN → Confirm payment
     - Return to Chefooz app
2. **COD Selected** (Cash on Delivery):
   - Skip payment UI
   - Order confirmed immediately (status: `CONFIRMED`)
   - Pay delivery person in cash on arrival
3. **Wait for Payment Confirmation**:
   - See loading spinner: "Verifying payment..."
   - Backend verifies payment with Razorpay
   - Success: Redirect to **Order Tracking** screen

#### **System Behavior**:
- **Payment Module**: Razorpay integration + verification
  - **Online Payment Flow**:
    1. Frontend calls Razorpay SDK with `razorpayOrderId`
    2. User completes payment in Razorpay UI
    3. Razorpay returns `razorpayPaymentId` + `razorpaySignature` to frontend
    4. Frontend sends to backend for verification:
       - Backend verifies HMAC SHA256 signature:
         ```
         expectedSignature = HMAC_SHA256(
           razorpayOrderId + "|" + razorpayPaymentId, 
           razorpayKeySecret
         )
         if (expectedSignature === razorpaySignature) → Payment valid
         ```
    5. If valid → Update order status to `CONFIRMED`
    6. If invalid → Return error, order remains `PENDING_PAYMENT`
  - **COD Flow**:
    1. No payment gateway interaction
    2. Order status immediately set to `CONFIRMED`
    3. Chef notified to prepare order
    4. Payment collected by delivery person on arrival
  - **Webhook Handling** (Asynchronous):
    - Razorpay sends webhook on payment success: `payment.captured`
    - Backend verifies webhook signature
    - Updates order status if not already updated (idempotency)
    - Stores payment details in `payment_intents` table

#### **Key APIs**:
```
POST /api/v1/payments/verify
Request Body:
{
  "orderId": "uuid",
  "razorpayOrderId": "order_abc123",
  "razorpayPaymentId": "pay_xyz789",
  "razorpaySignature": "abcdef123456..."
}

Response (200 OK):
{
  "success": true,
  "message": "Payment verified successfully",
  "data": {
    "orderId": "uuid",
    "status": "CONFIRMED",
    "paymentId": "pay_xyz789",
    "paidAmount": 742
  }
}

Response (400 Bad Request - Invalid Signature):
{
  "success": false,
  "message": "Payment verification failed",
  "errorCode": "PAYMENT_VERIFICATION_FAILED"
}

POST /api/v1/payments/webhook (Razorpay Webhook)
Request Headers:
{
  "x-razorpay-signature": "webhook_signature"
}
Request Body:
{
  "event": "payment.captured",
  "payload": {
    "payment": {
      "entity": {
        "id": "pay_xyz789",
        "order_id": "order_abc123",
        "status": "captured",
        "amount": 74200  // in paise
      }
    }
  }
}

Response (200 OK):
{
  "success": true
}
```

#### **Validation Rules**:
- ✅ Razorpay signature must match expected signature (HMAC SHA256)
- ✅ Payment amount must match order grand total (in paise)
- ✅ Razorpay payment status must be `captured` or `authorized`
- ✅ Order must be in `PENDING_PAYMENT` status (idempotency check)
- ✅ Webhook signature must be valid (prevents spoofing)

#### **User Decision Points**:
- **Payment Success?**
  - Redirect to Order Tracking screen → Proceed to Step 8
  - Show success message: "Order confirmed! Chef is preparing your food."
- **Payment Failed?**
  - Show error: "Payment failed. Please retry or choose different method."
  - Options: "Retry Payment" or "Change Payment Method"
  - Order remains in `PENDING_PAYMENT` status
- **Payment Pending?**
  - Show loading: "Payment processing... Please wait."
  - Auto-refresh status every 3 seconds (max 30 seconds)
  - If timeout → Show "Payment pending, check order status in a moment"

#### **Analytics Events**:
```json
{
  "event": "payment_initiated",
  "properties": {
    "order_id": "uuid",
    "razorpay_order_id": "order_abc123",
    "payment_method": "UPI_INTENT",
    "amount": 742
  }
}

{
  "event": "payment_success",
  "properties": {
    "order_id": "uuid",
    "razorpay_payment_id": "pay_xyz789",
    "amount": 742,
    "duration_seconds": 15
  }
}

{
  "event": "payment_failed",
  "properties": {
    "order_id": "uuid",
    "error_code": "PAYMENT_VERIFICATION_FAILED",
    "error_message": "Invalid signature"
  }
}
```

#### **Performance SLA**:
- Payment verification: <500ms
- Webhook processing: <200ms (async)
- Order status update: <300ms

---

### **Step 8: Track Order (Order)**

**Objective**: Monitor order status in real-time from confirmation to delivery.

#### **User Actions**:
1. **Redirected to Order Tracking Screen** after payment success
2. View real-time order status:
   - **CONFIRMED**: "Chef is preparing your food"
   - **PREPARING**: "Your order is being prepared"
   - **READY_FOR_PICKUP**: "Order ready! Delivery partner on the way"
   - **OUT_FOR_DELIVERY**: "Delivery partner picked up your order"
   - **DELIVERED**: "Order delivered! Enjoy your meal!"
3. See order details:
   - Order number (e.g., CHF-20260215-0123)
   - Chef info: name, avatar, contact
   - Estimated delivery time (countdown timer)
   - Delivery address
   - Order items with quantities
   - Total amount paid
4. Receive push notifications on status changes
5. (Optional) Contact chef or support via in-app chat
6. **After Delivery**: Rate order, earn coins (100% of dish price)

#### **System Behavior**:
- **Order Module**: Real-time status tracking + notifications
  - **Status Lifecycle**:
    ```
    PENDING_PAYMENT → CONFIRMED → PREPARING → READY_FOR_PICKUP → 
    OUT_FOR_DELIVERY → DELIVERED
    ```
  - **Chef Updates Status**: Chef manually updates status via chef app
  - **Push Notifications**: Sent on each status change via FCM (Firebase Cloud Messaging)
  - **Coin Rewards**: Automatically granted when status → `DELIVERED`
    - Coins = 100% of each dish price (not total order)
    - Example: ₹350 Biryani × 2 = 700 coins (redeemable as ₹7)
  - **Order History**: Stored in `orders` table with full audit trail
  - **Reorder Feature**: Order history shows "Reorder" button
    - Taps "Reorder" → Auto-adds all items to cart with current prices
    - Single-chef validation still applies

#### **Key APIs**:
```
GET /api/v1/orders/:orderId
Response (200 OK):
{
  "success": true,
  "data": {
    "orderId": "uuid",
    "orderNumber": "CHF-20260215-0123",
    "status": "PREPARING",
    "createdAt": "2026-02-15T10:30:00Z",
    "estimatedDeliveryTime": "2026-02-15T11:00:00Z",
    "chef": {
      "chefId": "uuid",
      "kitchenName": "Golden Spoon",
      "avatar": "https://cdn.chefooz.com/...",
      "phone": "+91-XXXXXXXXXX"
    },
    "deliveryAddress": {
      "addressLine1": "123 Main St",
      "city": "Mumbai",
      "distanceKm": 4.5
    },
    "items": [
      {
        "title": "Chicken Biryani",
        "quantity": 2,
        "price": 350
      }
    ],
    "priceBreakdown": {
      "subtotal": 700,
      "deliveryFee": 35,
      "platformFee": 5,
      "gst": 2.0,
      "grandTotal": 742
    },
    "coinsEarned": 700  // Only after DELIVERED status
  }
}

PATCH /api/v1/orders/:orderId/status
Request Body:
{
  "status": "PREPARING"
}

Response (200 OK):
{
  "success": true,
  "message": "Order status updated",
  "data": {
    "orderId": "uuid",
    "status": "PREPARING",
    "updatedAt": "2026-02-15T10:35:00Z"
  }
}

GET /api/v1/orders/history?userId=<userId>&limit=20
  → Returns user's past orders for reorder feature
```

#### **Validation Rules**:
- ✅ Only chef can update order status (authorization check)
- ✅ Status transitions must follow lifecycle order (no skipping)
- ✅ Coins granted only once (idempotency check on status → DELIVERED)
- ✅ Push notifications sent only on status change (not on polling)

#### **User Decision Points**:
- **Order Delivered?**
  - Show rating prompt: "How was your food?"
  - User rates 1-5 stars + optional review
  - Coins automatically credited to wallet
- **Want to Reorder?**
  - Tap "Reorder" from order history
  - All items added to cart with current prices
  - Single-chef validation applies (may need to clear cart if different chef)
- **Issue with Order?**
  - Tap "Help" → Contact support via in-app chat
  - Report issue: Late delivery, wrong items, quality issues

#### **Analytics Events**:
```json
{
  "event": "order_status_changed",
  "properties": {
    "order_id": "uuid",
    "old_status": "CONFIRMED",
    "new_status": "PREPARING",
    "chef_id": "uuid"
  }
}

{
  "event": "order_delivered",
  "properties": {
    "order_id": "uuid",
    "delivery_time_minutes": 28,
    "coins_earned": 700
  }
}

{
  "event": "order_rated",
  "properties": {
    "order_id": "uuid",
    "rating": 5,
    "has_review": true
  }
}

{
  "event": "order_reordered",
  "properties": {
    "original_order_id": "uuid",
    "new_cart_id": "uuid",
    "items_count": 2
  }
}
```

#### **Performance SLA**:
- Order fetch: <300ms
- Status update: <200ms
- Push notification delivery: <5s
- Coin credit: <500ms (async)

---

## Complete Flow Diagram

### **End-to-End User Journey (Mermaid Sequence Diagram)**

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant Feed as Feed Module
    participant Explore as Explore Module
    participant ChefPublic as Chef-Public Module
    participant Cart as Cart Module
    participant Checkout as Checkout Module
    participant Payment as Payment Module
    participant Order as Order Module
    participant Razorpay as Razorpay Gateway
    participant FCM as Push Notifications

    Note over User,FCM: STEP 1: BROWSE CONTENT
    User->>Feed: Open app → Land on Feed tab
    Feed->>Feed: Apply ranking algorithm<br/>(engagement × recency × CRS)
    Feed-->>User: Return ranked reels (paginated)
    User->>Feed: Watch reel, see "Order Now" CTA

    Note over User,FCM: STEP 2: VIEW CHEF MENU
    User->>ChefPublic: Tap "Order Now" → View chef menu
    ChefPublic->>ChefPublic: Fetch chef profile + menu items<br/>Group by platform categories
    ChefPublic-->>User: Return menu (grouped by category)
    User->>ChefPublic: Browse menu, view item details

    Note over User,FCM: STEP 3: ADD TO CART
    User->>Cart: Tap "Add to Cart" on menu item
    Cart->>Cart: Validate single-chef enforcement
    alt Cart has items from different chef
        Cart-->>User: 409 Conflict: "Replace cart?"
        User->>Cart: Choose: Keep Current OR Start Fresh
    end
    Cart->>Cart: Capture price snapshot
    Cart->>Cart: Save to PostgreSQL + Valkey cache
    Cart-->>User: 200 OK: Item added, cart badge updated
    User->>User: Continue shopping or proceed to cart

    Note over User,FCM: STEP 4: REVIEW CART
    User->>Cart: Tap cart icon → View cart
    Cart->>Cart: Fetch cart from cache/DB<br/>Enrich with current menu data
    Cart-->>User: Return cart items + subtotal
    User->>Cart: Adjust quantities or remove items
    Cart->>Cart: Update cart, recalculate subtotal
    Cart-->>User: Updated cart
    User->>Checkout: Tap "Proceed to Checkout"

    Note over User,FCM: STEP 5: SELECT ADDRESS
    User->>Checkout: Navigate to address selection
    Checkout->>Checkout: Fetch user's saved addresses
    Checkout-->>User: Return addresses
    User->>Checkout: Select address (or add new)
    Checkout->>Checkout: Calculate distance (Haversine)<br/>Validate within radius
    alt Address out of radius
        Checkout-->>User: 400 Error: "Out of delivery radius"
    end
    Checkout->>Checkout: Calculate delivery fee<br/>(base + distance + surge + GST)
    Checkout-->>User: Order preview with grand total

    Note over User,FCM: STEP 6: CONFIRM CHECKOUT
    User->>Checkout: Review order preview
    User->>Checkout: Select payment method (UPI/COD/Card)
    User->>Checkout: Tap "Place Order"
    Checkout->>Checkout: Validate cart, address, chef status
    alt Validation failed
        Checkout-->>User: 400 Error: "Validation failed"
    end
    Checkout->>Razorpay: Create Razorpay order
    Razorpay-->>Checkout: Return razorpayOrderId
    Checkout->>Order: Create order (status: PENDING_PAYMENT)
    Order-->>Checkout: Return orderId
    Checkout-->>User: Redirect to payment UI

    Note over User,FCM: STEP 7: MAKE PAYMENT
    alt Online Payment (UPI Intent)
        User->>Razorpay: Complete payment in Razorpay UI
        Razorpay-->>User: Return razorpayPaymentId + signature
        User->>Payment: Send payment details to backend
        Payment->>Payment: Verify HMAC SHA256 signature
        alt Signature invalid
            Payment-->>User: 400 Error: "Payment verification failed"
        end
        Payment->>Order: Update order status → CONFIRMED
        Razorpay->>Payment: Send webhook (async): payment.captured
        Payment->>Payment: Verify webhook signature<br/>Update payment_intents table
    else COD Payment
        Checkout->>Order: Update order status → CONFIRMED
    end
    Payment-->>User: Payment success, redirect to tracking

    Note over User,FCM: STEP 8: TRACK ORDER
    User->>Order: Navigate to order tracking screen
    Order->>Order: Fetch order details
    Order-->>User: Return order + status
    User->>Order: Poll for status updates (every 10s)
    
    Note over User,FCM: Chef Updates Status (External)
    Order->>FCM: Status changed → Send push notification
    FCM-->>User: Receive notification: "Order is preparing"
    Order->>Order: Status → DELIVERED
    Order->>Order: Credit coins (100% of dish price)
    Order->>FCM: Send notification: "Order delivered!"
    FCM-->>User: Receive delivery notification
    
    User->>Order: Rate order (1-5 stars)
    Order-->>User: Thank you, coins credited!

    Note over User,FCM: JOURNEY COMPLETE ✅
```

---

## Module Integration Map

### **Modules Involved in Customer Order Journey**

| # | Module | Purpose | Key APIs | Data Stored | Integration Points |
|---|--------|---------|----------|-------------|-------------------|
| **1** | **Feed** | Discover content via ranked feed | `GET /api/v1/feed` | MongoDB: Reels | → Chef-Public (reel CTA) |
| **2** | **Explore** | Discover content via curated sections | `GET /api/v1/explore/consolidated` | MongoDB: Reels | → Chef-Public (dish card) |
| **3** | **Chef-Public** | Browse chef menu for ordering | `GET /api/v1/chefs/:id/public`<br/>`GET /api/v1/chefs/:id/menu` | PostgreSQL: ChefKitchen, ChefMenuItem | → Cart (add to cart) |
| **4** | **Cart** | Persistent server-side cart | `POST /api/v1/cart/items`<br/>`GET /api/v1/cart`<br/>`PATCH /api/v1/cart/items/:id`<br/>`DELETE /api/v1/cart/items/:id` | PostgreSQL: cart_items<br/>Valkey: cart cache | → Checkout (proceed) |
| **5** | **Checkout** | Address selection + delivery fee calculation | `GET /api/v1/users/addresses`<br/>`POST /api/v1/checkout/calculate-delivery`<br/>`POST /api/v1/checkout/validate` | PostgreSQL: user_addresses | → Payment (create order) |
| **6** | **Payment** | Razorpay payment integration | `POST /api/v1/payments/verify`<br/>`POST /api/v1/payments/webhook` | PostgreSQL: payment_intents | → Order (confirm payment) |
| **7** | **Order** | Order lifecycle management | `GET /api/v1/orders/:id`<br/>`PATCH /api/v1/orders/:id/status`<br/>`GET /api/v1/orders/history` | PostgreSQL: orders, order_items | → Coins (reward on delivery) |
| **8** | **Auth** | JWT authentication | `POST /api/v1/auth/login` | PostgreSQL: users, tokens | All authenticated endpoints |

### **Data Flow Across Modules**

```mermaid
graph LR
    Feed[Feed Module<br/>Reels Discovery] -->|Reel CTA| ChefPublic[Chef-Public Module<br/>Menu Browsing]
    Explore[Explore Module<br/>Curated Discovery] -->|Dish Card| ChefPublic
    ChefPublic -->|Add to Cart| Cart[Cart Module<br/>Persistent Cart]
    Cart -->|Proceed to Checkout| Checkout[Checkout Module<br/>Address + Delivery Fee]
    Checkout -->|Create Order| Payment[Payment Module<br/>Razorpay Integration]
    Payment -->|Verify Payment| Order[Order Module<br/>Tracking + Fulfillment]
    Order -->|Reorder| Cart
    Order -->|Rate| Reviews[Reviews Module<br/>Future]
    
    Auth[Auth Module<br/>JWT] -.->|Authenticate| Cart
    Auth -.->|Authenticate| Checkout
    Auth -.->|Authenticate| Payment
    Auth -.->|Authenticate| Order

    style Feed fill:#e1f5ff
    style Explore fill:#e1f5ff
    style ChefPublic fill:#fff4e1
    style Cart fill:#ffe1e1
    style Checkout fill:#ffe1e1
    style Payment fill:#e1ffe1
    style Order fill:#e1ffe1
    style Auth fill:#f0f0f0
```

### **Cross-Module Dependencies**

| Source Module | Target Module | Dependency Type | What's Shared |
|---------------|---------------|----------------|---------------|
| **Feed** | **Chef-Public** | Navigation | Reel → Chef menu link |
| **Explore** | **Chef-Public** | Navigation | Dish card → Chef menu link |
| **Chef-Public** | **Cart** | Data | Menu item ID, chef ID, price snapshot |
| **Cart** | **Checkout** | Data | Cart items, subtotal, chef ID |
| **Checkout** | **Payment** | Data | Order total, Razorpay order ID |
| **Payment** | **Order** | Status Update | Payment success → Order confirmed |
| **Order** | **Cart** | Reorder | Order items → Auto-add to cart |
| **Auth** | **All Modules** | Security | JWT token for user authentication |

---

## API Call Sequences

### **Sequence 1: Browse Feed → Add to Cart**

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant App as Mobile App
    participant API as Backend API
    participant Feed as Feed Service
    participant ChefPublic as Chef-Public Service
    participant Cart as Cart Service
    participant DB as PostgreSQL
    participant Cache as Valkey

    User->>App: Open app
    App->>API: GET /api/v1/feed?limit=20
    API->>Feed: Fetch ranked feed
    Feed->>Cache: Check feed cache (key: feed:all:page:0)
    alt Cache hit
        Cache-->>Feed: Return cached feed
    else Cache miss
        Feed->>DB: Query reels with ranking algorithm
        DB-->>Feed: Return ranked reels
        Feed->>Cache: Store feed in cache (TTL: 5min)
    end
    Feed-->>API: Return feed
    API-->>App: 200 OK: Feed reels
    App-->>User: Display feed

    User->>App: Tap reel → Tap "Order Now"
    App->>API: GET /api/v1/chefs/:chefId/public
    API->>ChefPublic: Fetch chef profile + menu
    ChefPublic->>DB: Query chef + menu items
    DB-->>ChefPublic: Return chef data
    ChefPublic-->>API: Return chef profile + menu
    API-->>App: 200 OK: Chef menu
    App-->>User: Display menu

    User->>App: Tap "Add to Cart" on menu item
    App->>API: POST /api/v1/cart/items<br/>{menuItemId, quantity}
    API->>Cart: Add item to cart
    Cart->>DB: Check existing cart for user
    alt Cart exists with different chef
        DB-->>Cart: Return existing cart
        Cart-->>API: 409 Conflict: Different chef
        API-->>App: 409 Conflict
        App-->>User: Show "Replace cart?" modal
        User->>App: Choose "Start Fresh"
        App->>API: DELETE /api/v1/cart
        API->>Cart: Clear cart
        Cart->>DB: Delete cart items
        App->>API: POST /api/v1/cart/items (retry)
    end
    Cart->>DB: INSERT cart_item with price snapshot
    DB-->>Cart: Success
    Cart->>Cache: Invalidate cart cache
    Cart-->>API: 200 OK: Item added
    API-->>App: 200 OK: Cart updated
    App-->>User: Show cart badge (item count)
```

---

### **Sequence 2: Checkout → Payment → Order Confirmation**

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant App as Mobile App
    participant API as Backend API
    participant Checkout as Checkout Service
    participant Payment as Payment Service
    participant Order as Order Service
    participant Razorpay as Razorpay API
    participant DB as PostgreSQL
    participant FCM as Firebase Cloud Messaging

    User->>App: Tap "Proceed to Checkout"
    App->>API: GET /api/v1/users/addresses
    API->>Checkout: Fetch addresses
    Checkout->>DB: Query user_addresses
    DB-->>Checkout: Return addresses
    Checkout-->>API: Return addresses
    API-->>App: 200 OK: Addresses
    App-->>User: Display address selection

    User->>App: Select address
    App->>API: POST /api/v1/checkout/calculate-delivery<br/>{chefId, addressId}
    API->>Checkout: Calculate delivery fee
    Checkout->>DB: Fetch chef location + address coordinates
    DB-->>Checkout: Return locations
    Checkout->>Checkout: Calculate distance (Haversine)<br/>Calculate delivery fee
    Checkout-->>API: Return delivery fee + grand total
    API-->>App: 200 OK: Order preview
    App-->>User: Display order preview

    User->>App: Tap "Place Order"
    App->>API: POST /api/v1/checkout/validate<br/>{cartId, addressId, paymentMethod}
    API->>Checkout: Validate checkout
    Checkout->>DB: Validate cart items, chef status, address
    alt Validation failed
        DB-->>Checkout: Return errors
        Checkout-->>API: 400 Bad Request: Validation errors
        API-->>App: 400 Bad Request
        App-->>User: Show error messages
    end
    
    Checkout->>Razorpay: POST /v1/orders<br/>{amount, currency}
    Razorpay-->>Checkout: Return razorpayOrderId
    Checkout->>DB: INSERT payment_intent
    DB-->>Checkout: Success
    
    Checkout->>Order: Create order (status: PENDING_PAYMENT)
    Order->>DB: INSERT order + order_items
    DB-->>Order: Return orderId
    Order-->>Checkout: Return orderId
    Checkout-->>API: Return razorpayOrderId + orderId
    API-->>App: 200 OK: Payment ready
    App-->>User: Show Razorpay payment UI

    User->>App: Complete payment in Razorpay
    App->>App: Razorpay SDK returns<br/>razorpayPaymentId + signature
    App->>API: POST /api/v1/payments/verify<br/>{orderId, razorpayOrderId,<br/>razorpayPaymentId, razorpaySignature}
    API->>Payment: Verify payment
    Payment->>Payment: Verify HMAC SHA256 signature
    alt Signature invalid
        Payment-->>API: 400 Bad Request: Verification failed
        API-->>App: 400 Bad Request
        App-->>User: Show "Payment failed" error
    end
    
    Payment->>DB: UPDATE payment_intent (status: CAPTURED)
    Payment->>Order: Update order status → CONFIRMED
    Order->>DB: UPDATE order (status: CONFIRMED)
    DB-->>Order: Success
    Order->>FCM: Send push notification<br/>"Order confirmed!"
    FCM-->>User: Receive notification
    Order-->>Payment: Success
    Payment-->>API: 200 OK: Payment verified
    API-->>App: 200 OK: Order confirmed
    App-->>User: Redirect to order tracking

    Note over User,FCM: Async Webhook (Backup Verification)
    Razorpay->>API: POST /api/v1/payments/webhook<br/>{event: payment.captured}
    API->>Payment: Process webhook
    Payment->>Payment: Verify webhook signature
    Payment->>DB: UPDATE payment_intent (idempotency check)
    Payment-->>API: 200 OK
```

---

### **Sequence 3: Order Tracking + Status Updates**

```mermaid
sequenceDiagram
    autonumber
    actor User
    actor Chef
    participant App as Mobile App
    participant ChefApp as Chef App
    participant API as Backend API
    participant Order as Order Service
    participant DB as PostgreSQL
    participant FCM as Firebase Cloud Messaging

    User->>App: Navigate to order tracking
    App->>API: GET /api/v1/orders/:orderId
    API->>Order: Fetch order details
    Order->>DB: Query order + order_items
    DB-->>Order: Return order data
    Order-->>API: Return order details
    API-->>App: 200 OK: Order details
    App-->>User: Display order tracking UI

    loop Poll every 10 seconds
        App->>API: GET /api/v1/orders/:orderId
        API->>Order: Fetch order
        Order->>DB: Query order
        DB-->>Order: Return order
        Order-->>API: Return order
        API-->>App: 200 OK: Order
        App->>App: Update UI if status changed
    end

    Note over Chef,FCM: Chef Updates Status
    Chef->>ChefApp: Update order status → PREPARING
    ChefApp->>API: PATCH /api/v1/orders/:orderId/status<br/>{status: PREPARING}
    API->>Order: Update status
    Order->>DB: UPDATE order (status: PREPARING)
    DB-->>Order: Success
    Order->>FCM: Send push notification<br/>"Chef is preparing your food"
    FCM-->>User: Receive notification
    Order-->>API: 200 OK: Status updated
    API-->>ChefApp: 200 OK
    ChefApp-->>Chef: Show success message

    Chef->>ChefApp: Update status → READY_FOR_PICKUP
    ChefApp->>API: PATCH /api/v1/orders/:orderId/status<br/>{status: READY_FOR_PICKUP}
    API->>Order: Update status
    Order->>DB: UPDATE order
    Order->>FCM: Send notification<br/>"Order ready! Delivery on the way"
    FCM-->>User: Receive notification
    Order-->>API: 200 OK
    API-->>ChefApp: 200 OK

    Chef->>ChefApp: Update status → OUT_FOR_DELIVERY
    ChefApp->>API: PATCH /api/v1/orders/:orderId/status
    API->>Order: Update status
    Order->>DB: UPDATE order
    Order->>FCM: Send notification<br/>"Delivery partner picked up order"
    FCM-->>User: Receive notification
    Order-->>API: 200 OK

    Chef->>ChefApp: Update status → DELIVERED
    ChefApp->>API: PATCH /api/v1/orders/:orderId/status<br/>{status: DELIVERED}
    API->>Order: Update status
    Order->>DB: UPDATE order (status: DELIVERED)
    Order->>Order: Calculate coins<br/>(100% of dish price)
    Order->>DB: INSERT coins transaction
    Order->>FCM: Send notification<br/>"Order delivered! 700 coins earned"
    FCM-->>User: Receive notification
    Order-->>API: 200 OK: Status + coins
    API-->>ChefApp: 200 OK
    ChefApp-->>Chef: Show success

    User->>App: Tap "Rate Order"
    App->>API: POST /api/v1/orders/:orderId/rate<br/>{rating: 5, review: "Delicious!"}
    API->>Order: Save rating
    Order->>DB: INSERT order_rating
    Order-->>API: 200 OK
    API-->>App: 200 OK: Thank you!
    App-->>User: Show thank you message
```

---

## State Transitions

### **Order Status State Machine**

```mermaid
stateDiagram-v2
    [*] --> PENDING_PAYMENT: Checkout validated,<br/>order created

    PENDING_PAYMENT --> CONFIRMED: Payment verified<br/>(online) OR<br/>COD selected
    PENDING_PAYMENT --> PAYMENT_FAILED: Payment verification<br/>failed
    PENDING_PAYMENT --> CANCELLED: User cancels<br/>before payment

    PAYMENT_FAILED --> PENDING_PAYMENT: User retries<br/>payment
    PAYMENT_FAILED --> CANCELLED: User abandons<br/>order

    CONFIRMED --> PREPARING: Chef accepts<br/>order
    CONFIRMED --> REJECTED: Chef rejects<br/>(unavailable items)

    PREPARING --> READY_FOR_PICKUP: Chef marks<br/>ready
    PREPARING --> CANCELLED: User/Chef<br/>cancels

    READY_FOR_PICKUP --> OUT_FOR_DELIVERY: Delivery partner<br/>picks up
    READY_FOR_PICKUP --> CANCELLED: User/Chef<br/>cancels

    OUT_FOR_DELIVERY --> DELIVERED: Delivery partner<br/>confirms delivery
    OUT_FOR_DELIVERY --> CANCELLED: Delivery failed

    DELIVERED --> [*]: Coins earned,<br/>order complete

    REJECTED --> REFUND_INITIATED: Auto-refund<br/>triggered
    CANCELLED --> REFUND_INITIATED: Refund<br/>triggered
    REFUND_INITIATED --> REFUNDED: Refund<br/>processed
    REFUNDED --> [*]

    note right of CONFIRMED
        Push notification sent:<br/>"Order confirmed! Chef is preparing."
    end note

    note right of PREPARING
        Push notification sent:<br/>"Chef is preparing your food."
    end note

    note right of READY_FOR_PICKUP
        Push notification sent:<br/>"Order ready! Delivery on the way."
    end note

    note right of OUT_FOR_DELIVERY
        Push notification sent:<br/>"Delivery partner picked up order."
    end note

    note right of DELIVERED
        Push notification sent:<br/>"Order delivered! Enjoy your meal."<br/><br/>Coins auto-credited:<br/>100% of dish price
    end note
```

### **Cart State Transitions**

```mermaid
stateDiagram-v2
    [*] --> EMPTY: User has no cart

    EMPTY --> ACTIVE: First item added

    ACTIVE --> ACTIVE: Add/update/remove items<br/>(same chef)
    ACTIVE --> CHEF_CONFLICT: Attempt to add item<br/>from different chef
    ACTIVE --> EMPTY: Remove all items

    CHEF_CONFLICT --> ACTIVE: User keeps<br/>current cart
    CHEF_CONFLICT --> REPLACED: User replaces<br/>with new chef

    REPLACED --> ACTIVE: New item added

    ACTIVE --> CHECKOUT_INITIATED: User proceeds<br/>to checkout

    CHECKOUT_INITIATED --> VALIDATED: Cart validation<br/>passed
    CHECKOUT_INITIATED --> ACTIVE: Validation failed,<br/>user returns

    VALIDATED --> ORDER_PLACED: Payment successful

    ORDER_PLACED --> ARCHIVED: Order confirmed,<br/>cart cleared

    ARCHIVED --> [*]

    note right of ACTIVE
        Cart synced across devices<br/>via Valkey cache (15min TTL)
    end note

    note right of CHEF_CONFLICT
        Frontend shows modal:<br/>"Replace cart with items<br/>from [New Chef]?"
    end note

    note right of VALIDATED
        Cart items snapshot<br/>captured for order
    end note

    note right of ARCHIVED
        Cart items soft-deleted,<br/>kept for audit trail
    end note
```

### **Payment State Transitions**

```mermaid
stateDiagram-v2
    [*] --> PAYMENT_INITIATED: User taps<br/>"Place Order"

    PAYMENT_INITIATED --> RAZORPAY_ORDER_CREATED: Razorpay order<br/>created successfully
    PAYMENT_INITIATED --> PAYMENT_FAILED: Razorpay order<br/>creation failed

    RAZORPAY_ORDER_CREATED --> AWAITING_PAYMENT: Razorpay UI<br/>shown to user

    AWAITING_PAYMENT --> PAYMENT_SUCCESS: User completes<br/>payment in Razorpay
    AWAITING_PAYMENT --> PAYMENT_FAILED: User cancels or<br/>payment fails
    AWAITING_PAYMENT --> PAYMENT_TIMEOUT: 30 seconds elapsed,<br/>no response

    PAYMENT_SUCCESS --> VERIFYING_SIGNATURE: Backend receives<br/>payment details

    VERIFYING_SIGNATURE --> SIGNATURE_VALID: HMAC SHA256<br/>signature matches
    VERIFYING_SIGNATURE --> SIGNATURE_INVALID: Signature<br/>verification failed

    SIGNATURE_VALID --> ORDER_CONFIRMED: Order status →<br/>CONFIRMED
    SIGNATURE_INVALID --> PAYMENT_FAILED: Treat as<br/>failed payment

    ORDER_CONFIRMED --> WEBHOOK_RECEIVED: Razorpay webhook<br/>(async backup)
    WEBHOOK_RECEIVED --> WEBHOOK_VERIFIED: Webhook signature<br/>valid
    WEBHOOK_VERIFIED --> [*]: Payment complete

    PAYMENT_FAILED --> RETRY_AVAILABLE: User can retry<br/>payment
    RETRY_AVAILABLE --> PAYMENT_INITIATED: User clicks<br/>"Retry Payment"

    PAYMENT_TIMEOUT --> RETRY_AVAILABLE: User can check<br/>status or retry

    note right of RAZORPAY_ORDER_CREATED
        payment_intent record created:<br/>status = PENDING
    end note

    note right of SIGNATURE_VALID
        payment_intent updated:<br/>status = CAPTURED
    end note

    note right of WEBHOOK_VERIFIED
        Idempotency check:<br/>Skip if already processed
    end note

    state COD_FLOW {
        [*] --> COD_SELECTED: User selects<br/>Cash on Delivery
        COD_SELECTED --> ORDER_CONFIRMED_COD: Order confirmed<br/>immediately
        ORDER_CONFIRMED_COD --> [*]: No payment<br/>verification needed
    }
```

---

## Error Scenarios & Recovery

### **Error Catalog (Customer Order Journey)**

| # | Error Scenario | Module | Error Code | HTTP Status | User Impact | Recovery Path |
|---|----------------|--------|------------|-------------|-------------|---------------|
| **E1** | **Cart Conflict (Different Chef)** | Cart | `CART_DIFFERENT_CHEF` | 409 Conflict | Cannot add item from new chef | Show modal: "Keep Current" or "Start Fresh" |
| **E2** | **Cart Empty at Checkout** | Cart | `CART_EMPTY` | 400 Bad Request | Cannot proceed to checkout | Redirect to chef menu, add items |
| **E3** | **Menu Item Unavailable** | Cart | `ITEM_UNAVAILABLE` | 400 Bad Request | Item no longer available | Show alert, auto-remove from cart |
| **E4** | **Chef Offline** | Cart/Checkout | `CHEF_OFFLINE` | 400 Bad Request | Cannot place order | Show banner: "Chef closed. Save cart for later" |
| **E5** | **Address Out of Radius** | Checkout | `ADDRESS_OUT_OF_RADIUS` | 400 Bad Request | Cannot deliver to address | Show error, suggest "Add Different Address" |
| **E6** | **Price Changed** | Checkout | `PRICE_CHANGED` | 200 OK (Warning) | Cart total updated | Show alert: "Price changed from ₹X to ₹Y" |
| **E7** | **Checkout Validation Failed** | Checkout | `CHECKOUT_VALIDATION_FAILED` | 400 Bad Request | Cannot proceed to payment | Show errors, redirect to cart |
| **E8** | **Razorpay Order Creation Failed** | Payment | `RAZORPAY_ORDER_FAILED` | 500 Internal Error | Cannot initiate payment | Show error, retry after 5s |
| **E9** | **Payment Verification Failed** | Payment | `PAYMENT_VERIFICATION_FAILED` | 400 Bad Request | Payment not confirmed | Show "Payment failed. Retry?" |
| **E10** | **Payment Timeout** | Payment | `PAYMENT_TIMEOUT` | 408 Request Timeout | Payment status unknown | Show "Checking status..." + retry option |
| **E11** | **Order Not Found** | Order | `ORDER_NOT_FOUND` | 404 Not Found | Cannot view order details | Redirect to order history |
| **E12** | **Unauthorized Status Update** | Order | `UNAUTHORIZED` | 403 Forbidden | Cannot update order status | Only chef can update (auth check) |

---

### **Error Flow Diagram**

```mermaid
flowchart TD
    Start([User Action]) --> Action{Which Action?}
    
    Action -->|Add to Cart| CheckCart{Cart has items<br/>from different chef?}
    CheckCart -->|Yes| E1[E1: CART_DIFFERENT_CHEF<br/>409 Conflict]
    E1 --> ShowModal[Show modal:<br/>'Keep Current' OR 'Start Fresh']
    ShowModal --> UserChoice{User Choice?}
    UserChoice -->|Keep Current| CancelAdd[Cancel add, stay on menu]
    UserChoice -->|Start Fresh| ClearCart[Clear cart, add new item]
    CheckCart -->|No| ValidateItem{Item available<br/>& Chef online?}
    ValidateItem -->|No| E3[E3: ITEM_UNAVAILABLE<br/>400 Bad Request]
    E3 --> ShowAlert1[Show alert: 'Item unavailable']
    ValidateItem -->|Yes| AddSuccess[Item added to cart ✅]

    Action -->|Proceed to Checkout| CheckCartEmpty{Cart empty?}
    CheckCartEmpty -->|Yes| E2[E2: CART_EMPTY<br/>400 Bad Request]
    E2 --> RedirectMenu[Redirect to chef menu]
    CheckCartEmpty -->|No| CheckChef{Chef online?}
    CheckChef -->|No| E4[E4: CHEF_OFFLINE<br/>400 Bad Request]
    E4 --> ShowBanner[Show banner: 'Chef closed']
    CheckChef -->|Yes| ProceedAddress[Proceed to address selection]

    ProceedAddress --> SelectAddress{Select Address}
    SelectAddress --> CalcDistance[Calculate distance]
    CalcDistance --> CheckRadius{Within delivery<br/>radius?}
    CheckRadius -->|No| E5[E5: ADDRESS_OUT_OF_RADIUS<br/>400 Bad Request]
    E5 --> ShowError1[Show error + suggest<br/>'Add Different Address']
    CheckRadius -->|Yes| ValidateCheckout[Validate checkout]

    ValidateCheckout --> CheckPrice{Price changed?}
    CheckPrice -->|Yes| E6[E6: PRICE_CHANGED<br/>200 OK Warning]
    E6 --> ShowAlert2[Show alert: 'Price updated']
    CheckPrice -->|No| CheckValidation{All validations<br/>passed?}
    CheckValidation -->|No| E7[E7: CHECKOUT_VALIDATION_FAILED<br/>400 Bad Request]
    E7 --> ShowErrors[Show validation errors]
    ShowErrors --> RedirectCart[Redirect to cart]
    CheckValidation -->|Yes| CreateRazorpay[Create Razorpay order]

    CreateRazorpay --> RazorpaySuccess{Razorpay order<br/>created?}
    RazorpaySuccess -->|No| E8[E8: RAZORPAY_ORDER_FAILED<br/>500 Internal Error]
    E8 --> RetryRazorpay[Retry after 5 seconds]
    RazorpaySuccess -->|Yes| ShowPaymentUI[Show payment UI]

    ShowPaymentUI --> UserPays{User completes<br/>payment?}
    UserPays -->|Timeout| E10[E10: PAYMENT_TIMEOUT<br/>408 Timeout]
    E10 --> CheckStatus[Show 'Checking status...'<br/>+ Retry option]
    UserPays -->|Success| VerifySignature[Verify HMAC signature]
    VerifySignature --> SignatureValid{Signature valid?}
    SignatureValid -->|No| E9[E9: PAYMENT_VERIFICATION_FAILED<br/>400 Bad Request]
    E9 --> ShowRetry[Show 'Payment failed. Retry?']
    SignatureValid -->|Yes| ConfirmOrder[Order confirmed ✅]

    Action -->|Track Order| FetchOrder{Order exists?}
    FetchOrder -->|No| E11[E11: ORDER_NOT_FOUND<br/>404 Not Found]
    E11 --> RedirectHistory[Redirect to order history]
    FetchOrder -->|Yes| ShowTracking[Show order tracking ✅]

    Action -->|Update Status| CheckAuth{User is chef?}
    CheckAuth -->|No| E12[E12: UNAUTHORIZED<br/>403 Forbidden]
    E12 --> ShowAuthError[Show 'Unauthorized' error]
    CheckAuth -->|Yes| UpdateStatus[Update order status ✅]

    style E1 fill:#ffcccc
    style E2 fill:#ffcccc
    style E3 fill:#ffcccc
    style E4 fill:#ffcccc
    style E5 fill:#ffcccc
    style E6 fill:#fff4cc
    style E7 fill:#ffcccc
    style E8 fill:#ffcccc
    style E9 fill:#ffcccc
    style E10 fill:#ffcccc
    style E11 fill:#ffcccc
    style E12 fill:#ffcccc
    style AddSuccess fill:#ccffcc
    style ConfirmOrder fill:#ccffcc
    style ShowTracking fill:#ccffcc
    style UpdateStatus fill:#ccffcc
```

---

### **Detailed Error Recovery Paths**

#### **E1: Cart Conflict (Different Chef)** — 409 Conflict

**Scenario**: User has items from Chef A in cart, tries to add item from Chef B.

**Error Response**:
```json
{
  "success": false,
  "message": "Cart contains items from different chef",
  "errorCode": "CART_DIFFERENT_CHEF",
  "data": {
    "currentChef": {
      "chefId": "uuid-A",
      "kitchenName": "Golden Spoon"
    },
    "attemptedChef": {
      "chefId": "uuid-B",
      "kitchenName": "Tasty Bites"
    }
  }
}
```

**Recovery Steps**:
1. Frontend shows modal:
   - Title: "Replace Cart Items?"
   - Message: "Your cart contains items from **Golden Spoon**. Do you want to replace with items from **Tasty Bites**?"
   - Buttons: **"Keep Current"** (dismiss) | **"Start Fresh"** (clear cart)
2. User chooses:
   - **Keep Current**: Cancel new item addition, stay on current chef menu
   - **Start Fresh**: Call `DELETE /api/v1/cart`, then retry `POST /api/v1/cart/items`
3. Analytics event: `cart_conflict_occurred` with user action

**Prevention**:
- Show banner on chef menu: "You have items from another chef. Adding will replace cart."
- Gray out "Add to Cart" button with tooltip on hover

---

#### **E5: Address Out of Radius** — 400 Bad Request

**Scenario**: User selects address 8 km away, chef delivers within 5 km only.

**Error Response**:
```json
{
  "success": false,
  "message": "Address is outside chef's delivery radius",
  "errorCode": "ADDRESS_OUT_OF_RADIUS",
  "data": {
    "distanceKm": 8.2,
    "maxRadiusKm": 5.0,
    "suggestedAction": "add_different_address"
  }
}
```

**Recovery Steps**:
1. Show error banner:
   - Message: "This address is **8.2 km** away. **Golden Spoon** delivers within **5 km**."
   - Icon: ⚠️ (warning)
2. Provide options:
   - **"Add Different Address"**: Open address form
   - **"Find Nearby Chefs"**: Navigate to Explore with location filter
3. Disable "Proceed to Payment" button until valid address selected
4. Analytics event: `address_out_of_radius`

**Prevention**:
- Show delivery radius on chef profile page
- Highlight addresses within radius with green checkmark
- Filter chefs in Feed/Explore based on user's current location (if available)

---

#### **E6: Price Changed** — 200 OK (Warning)

**Scenario**: Cart has item snapshot at ₹350, current price is ₹380.

**Success Response with Warning**:
```json
{
  "success": true,
  "message": "Cart validated with warnings",
  "data": {
    "warnings": [
      {
        "type": "PRICE_CHANGED",
        "cartItemId": "uuid",
        "menuItemTitle": "Chicken Biryani",
        "oldPrice": 350,
        "newPrice": 380,
        "priceDifference": 30
      }
    ],
    "subtotal": 760,  // Updated subtotal
    "grandTotal": 797  // Updated grand total
  }
}
```

**Recovery Steps**:
1. Show alert dialog:
   - Title: "Price Updated"
   - Message: "**Chicken Biryani** price changed from **₹350** to **₹380**. Updated cart total: **₹797**."
   - Buttons: **"OK, Proceed"** | **"Remove Item"**
2. User chooses:
   - **OK, Proceed**: Accept new price, continue to payment
   - **Remove Item**: Call `DELETE /api/v1/cart/items/:id`, return to cart
3. Update cart UI to highlight changed prices (yellow background)
4. Analytics event: `price_changed_at_checkout`

**Prevention**:
- Show price snapshot capture time in cart: "Price locked 5 minutes ago"
- Refresh cart prices on cart screen open (validate before checkout)

---

#### **E9: Payment Verification Failed** — 400 Bad Request

**Scenario**: User completed payment in Razorpay, but signature verification failed.

**Error Response**:
```json
{
  "success": false,
  "message": "Payment verification failed",
  "errorCode": "PAYMENT_VERIFICATION_FAILED",
  "data": {
    "orderId": "uuid",
    "razorpayPaymentId": "pay_xyz789",
    "reason": "Invalid signature"
  }
}
```

**Recovery Steps**:
1. Show error screen:
   - Title: "Payment Verification Failed"
   - Message: "We couldn't verify your payment. Please retry or contact support."
   - Icon: ❌ (error)
2. Provide options:
   - **"Retry Payment"**: Redirect back to Razorpay UI with same `razorpayOrderId`
   - **"Check Payment Status"**: Call `GET /api/v1/orders/:orderId` to check status
   - **"Contact Support"**: Open in-app chat with payment ID pre-filled
3. Order remains in `PENDING_PAYMENT` status (can retry)
4. Analytics event: `payment_verification_failed`

**Prevention**:
- Razorpay webhook acts as backup verification (async)
- If webhook confirms payment → Auto-update order status
- Show "Payment processing..." for 30 seconds before showing retry option

---

#### **E10: Payment Timeout** — 408 Request Timeout

**Scenario**: User completed payment, but no response received within 30 seconds.

**Error Response**:
```json
{
  "success": false,
  "message": "Payment status check timed out",
  "errorCode": "PAYMENT_TIMEOUT",
  "data": {
    "orderId": "uuid",
    "razorpayOrderId": "order_abc123",
    "suggestedAction": "check_status"
  }
}
```

**Recovery Steps**:
1. Show timeout screen:
   - Title: "Payment Pending"
   - Message: "Your payment is being processed. Check order status in a moment."
   - Icon: ⏱️ (loading)
2. Provide options:
   - **"Check Status"**: Call `GET /api/v1/orders/:orderId` to fetch latest status
   - **"Retry Payment"**: Redirect back to Razorpay UI (if order still `PENDING_PAYMENT`)
   - **"View Orders"**: Navigate to order history
3. Auto-refresh status every 5 seconds for 1 minute
4. Analytics event: `payment_timeout`

**Prevention**:
- Set Razorpay timeout to 3 minutes (default)
- Show progress bar during payment verification
- Implement webhook for async confirmation (backup)

---

## Analytics & Tracking

### **Analytics Events Across Journey**

| # | Event Name | Module | Trigger | Properties | Priority |
|---|------------|--------|---------|------------|----------|
| **1** | `reel_viewed` | Feed | User views reel >2s | `reel_id`, `chef_id`, `source`, `duration_seconds` | P1 |
| **2** | `reel_cta_clicked` | Feed | Tap "Order Now" CTA | `reel_id`, `chef_id`, `source` | P0 |
| **3** | `chef_menu_viewed` | Chef-Public | Open chef menu page | `chef_id`, `source`, `menu_items_count`, `is_online` | P0 |
| **4** | `item_added_to_cart` | Cart | Add item to cart | `menu_item_id`, `chef_id`, `quantity`, `price_snapshot`, `cart_item_count`, `subtotal` | P0 |
| **5** | `cart_conflict_occurred` | Cart | Try to add different chef | `current_chef_id`, `attempted_chef_id`, `user_action` | P1 |
| **6** | `cart_viewed` | Cart | Open cart screen | `cart_id`, `chef_id`, `item_count`, `subtotal` | P1 |
| **7** | `cart_item_quantity_changed` | Cart | Update quantity | `cart_item_id`, `old_quantity`, `new_quantity` | P2 |
| **8** | `cart_item_removed` | Cart | Remove item from cart | `cart_item_id`, `menu_item_id` | P2 |
| **9** | `checkout_initiated` | Checkout | Tap "Proceed to Checkout" | `cart_id`, `chef_id`, `subtotal`, `grand_total`, `payment_method` | P0 |
| **10** | `address_selected` | Checkout | Select delivery address | `address_id`, `distance_km`, `delivery_fee` | P0 |
| **11** | `address_out_of_radius` | Checkout | Address validation failed | `distance_km`, `max_radius_km`, `chef_id` | P1 |
| **12** | `checkout_preview_viewed` | Checkout | View order preview | `subtotal`, `delivery_fee`, `platform_fee`, `gst`, `grand_total` | P1 |
| **13** | `checkout_validation_failed` | Checkout | Validation errors | `errors` (array) | P1 |
| **14** | `order_created` | Order | Order record created | `order_id`, `order_number`, `status`, `grand_total` | P0 |
| **15** | `payment_initiated` | Payment | Tap "Place Order" | `order_id`, `razorpay_order_id`, `payment_method`, `amount` | P0 |
| **16** | `payment_success` | Payment | Payment verified | `order_id`, `razorpay_payment_id`, `amount`, `duration_seconds` | P0 |
| **17** | `payment_failed` | Payment | Payment verification failed | `order_id`, `error_code`, `error_message` | P0 |
| **18** | `order_status_changed` | Order | Status updated by chef | `order_id`, `old_status`, `new_status`, `chef_id` | P0 |
| **19** | `order_delivered` | Order | Status → DELIVERED | `order_id`, `delivery_time_minutes`, `coins_earned` | P0 |
| **20** | `order_rated` | Order | User rates order | `order_id`, `rating`, `has_review` | P1 |
| **21** | `order_reordered` | Order | Tap "Reorder" button | `original_order_id`, `new_cart_id`, `items_count` | P1 |

---

### **Conversion Funnel Tracking**

```mermaid
graph TD
    A[Step 1: Reel Viewed<br/>100% Baseline] --> B[Step 2: CTA Clicked<br/>Target: 15%]
    B --> C[Step 3: Menu Viewed<br/>Target: 90% of CTA clicks]
    C --> D[Step 4: Item Added to Cart<br/>Target: 40% of menu views]
    D --> E[Step 5: Cart Viewed<br/>Target: 70% of cart additions]
    E --> F[Step 6: Checkout Initiated<br/>Target: 60% of cart views]
    F --> G[Step 7: Address Selected<br/>Target: 95% of checkouts]
    G --> H[Step 8: Order Created<br/>Target: 90% of address selections]
    H --> I[Step 9: Payment Success<br/>Target: 92% of orders]
    I --> J[Step 10: Order Delivered<br/>Target: 98% of paid orders]
    J --> K[Step 11: Order Rated<br/>Target: 50% of deliveries]

    style A fill:#e1f5ff
    style B fill:#e1f5ff
    style C fill:#fff4e1
    style D fill:#ffe1e1
    style E fill:#ffe1e1
    style F fill:#ffe1e1
    style G fill:#ffe1e1
    style H fill:#e1ffe1
    style I fill:#e1ffe1
    style J fill:#e1ffe1
    style K fill:#f0f0f0
```

### **Drop-off Analysis (Where Users Exit Journey)**

| Step Transition | Expected Conversion | Drop-off Risk | Mitigation Strategy |
|-----------------|---------------------|---------------|---------------------|
| **Reel → CTA Click** | 15% | HIGH | Improve CTA visibility, clearer copy |
| **CTA → Menu View** | 90% | LOW | Fast loading, cache chef menu |
| **Menu → Add to Cart** | 40% | HIGH | Better item photos, competitive pricing |
| **Cart → Checkout** | 60% | MEDIUM | Reduce friction, show delivery ETA |
| **Checkout → Address** | 95% | LOW | Pre-select default address |
| **Address → Order** | 90% | MEDIUM | Validate address early, clear pricing |
| **Order → Payment** | 92% | HIGH | Multiple payment options, retry UX |
| **Payment → Delivery** | 98% | LOW | Clear status updates, push notifications |
| **Delivery → Rating** | 50% | LOW (Non-blocking) | Incentivize with bonus coins |

---

### **Event Payload Examples**

#### **Event: `item_added_to_cart`**
```json
{
  "event": "item_added_to_cart",
  "timestamp": "2026-02-15T10:30:00Z",
  "user_id": "uuid",
  "session_id": "session-uuid",
  "properties": {
    "menu_item_id": "uuid",
    "menu_item_title": "Chicken Biryani",
    "chef_id": "uuid",
    "chef_kitchen_name": "Golden Spoon",
    "quantity": 2,
    "price_snapshot": 350,
    "cart_item_count": 2,
    "subtotal": 700,
    "source": "chef_menu",
    "device": "iOS",
    "app_version": "1.2.0"
  }
}
```

#### **Event: `checkout_validation_failed`**
```json
{
  "event": "checkout_validation_failed",
  "timestamp": "2026-02-15T10:35:00Z",
  "user_id": "uuid",
  "session_id": "session-uuid",
  "properties": {
    "cart_id": "uuid",
    "chef_id": "uuid",
    "errors": [
      {
        "field": "cart.items[0]",
        "message": "Chicken Biryani is no longer available",
        "error_code": "ITEM_UNAVAILABLE"
      },
      {
        "field": "chef",
        "message": "Chef is currently offline",
        "error_code": "CHEF_OFFLINE"
      }
    ],
    "subtotal": 700,
    "device": "iOS"
  }
}
```

#### **Event: `payment_success`**
```json
{
  "event": "payment_success",
  "timestamp": "2026-02-15T10:40:00Z",
  "user_id": "uuid",
  "session_id": "session-uuid",
  "properties": {
    "order_id": "uuid",
    "order_number": "CHF-20260215-0123",
    "razorpay_order_id": "order_abc123",
    "razorpay_payment_id": "pay_xyz789",
    "payment_method": "UPI_INTENT",
    "amount": 742,
    "duration_seconds": 15,
    "chef_id": "uuid",
    "device": "iOS"
  }
}
```

---

## Performance SLAs

### **End-to-End Journey SLAs**

| Journey Segment | SLA | Target | Critical Threshold | Measurement |
|-----------------|-----|--------|-------------------|-------------|
| **Browse → Add to Cart** | <5 seconds | <3 seconds | >10 seconds | Client-side timer |
| **Cart → Checkout** | <3 seconds | <2 seconds | >7 seconds | Client-side timer |
| **Checkout → Payment UI** | <5 seconds | <3 seconds | >10 seconds | API response time + Razorpay load |
| **Payment → Confirmation** | <10 seconds | <5 seconds | >30 seconds | Razorpay verification time |
| **End-to-End (Reel → Order Placed)** | <5 minutes | <3 minutes | >10 minutes | Total journey time |

---

### **API Response Time SLAs**

| API Endpoint | Method | SLA (p95) | Target (p50) | Cached | Priority |
|--------------|--------|-----------|--------------|--------|----------|
| `GET /api/v1/feed` | GET | <800ms | <200ms | Yes (5min) | P0 |
| `GET /api/v1/explore/consolidated` | GET | <500ms | <200ms | Yes (15min) | P1 |
| `GET /api/v1/chefs/:id/public` | GET | <500ms | <300ms | No | P0 |
| `GET /api/v1/chefs/:id/menu` | GET | <600ms | <400ms | Yes (10min) | P0 |
| `POST /api/v1/cart/items` | POST | <300ms | <200ms | No | P0 |
| `GET /api/v1/cart` | GET | <500ms | <200ms | Yes (15min) | P0 |
| `PATCH /api/v1/cart/items/:id` | PATCH | <200ms | <150ms | No | P1 |
| `DELETE /api/v1/cart/items/:id` | DELETE | <200ms | <150ms | No | P1 |
| `GET /api/v1/users/addresses` | GET | <300ms | <200ms | Yes (30min) | P0 |
| `POST /api/v1/checkout/calculate-delivery` | POST | <200ms | <100ms | No | P0 |
| `POST /api/v1/checkout/validate` | POST | <800ms | <500ms | No | P0 |
| `POST /api/v1/orders/create` | POST | <500ms | <300ms | No | P0 |
| `POST /api/v1/payments/verify` | POST | <1000ms | <500ms | No | P0 |
| `GET /api/v1/orders/:id` | GET | <400ms | <300ms | No | P0 |
| `PATCH /api/v1/orders/:id/status` | PATCH | <300ms | <200ms | No | P0 |

---

### **Database Query Performance**

| Query Type | Target Execution Time | Index Required | Priority |
|------------|----------------------|----------------|----------|
| **Fetch user cart** | <50ms | `cart_items(user_id, deleted_at)` | P0 |
| **Fetch chef menu** | <100ms | `chef_menu_items(chef_id, is_available)` | P0 |
| **Calculate delivery fee** | <20ms | `chef_kitchen(chef_id)`<br/>`user_addresses(address_id)` | P0 |
| **Validate checkout** | <200ms | Composite (cart + chef + menu) | P0 |
| **Create order** | <150ms | `orders(user_id, created_at)` | P0 |
| **Fetch order details** | <100ms | `orders(order_id)` | P0 |
| **Update order status** | <50ms | `orders(order_id)` | P0 |

---

### **Caching Strategy**

| Data Type | Cache Location | TTL | Invalidation Trigger | Priority |
|-----------|----------------|-----|---------------------|----------|
| **Feed (first page)** | Valkey | 5 minutes | Manual refresh | P0 |
| **Explore sections** | Valkey | 15 minutes | Manual refresh | P1 |
| **Chef profile** | Valkey | 10 minutes | Chef updates profile | P1 |
| **Chef menu** | Valkey | 10 minutes | Menu item added/updated | P0 |
| **User cart** | Valkey | 15 minutes | Cart item added/updated/removed | P0 |
| **User addresses** | Valkey | 30 minutes | Address added/updated | P1 |
| **Order details** | None | N/A | Real-time updates required | P0 |

---

## Testing Strategy

### **Testing Pyramid (Customer Order Journey)**

```mermaid
graph TD
    A[Unit Tests<br/>70% Coverage] --> B[Integration Tests<br/>20% Coverage]
    B --> C[E2E Tests<br/>10% Coverage]
    
    A1[Cart Module<br/>Service Tests] --> A
    A2[Checkout Module<br/>Service Tests] --> A
    A3[Payment Module<br/>Service Tests] --> A
    A4[Order Module<br/>Service Tests] --> A
    
    B1[Cart → Checkout<br/>API Integration] --> B
    B2[Checkout → Payment<br/>Razorpay Mock] --> B
    B3[Payment → Order<br/>Webhook Tests] --> B
    
    C1[Full Journey<br/>Cypress E2E] --> C
    C2[Payment Flows<br/>Razorpay Sandbox] --> C
    C3[Error Scenarios<br/>Failure Injection] --> C

    style A fill:#e1f5ff
    style B fill:#fff4e1
    style C fill:#ffe1e1
```

---

### **Test Coverage by Type**

#### **1. Unit Tests (70% Coverage)**

| Module | Test Category | Test Count | Examples |
|--------|--------------|------------|----------|
| **Cart** | Service Logic | 25 | Single-chef enforcement, quantity validation, price snapshot |
| **Cart** | Repository | 10 | CRUD operations, soft delete |
| **Checkout** | Service Logic | 20 | Distance calculation (Haversine), delivery fee calculation, validation rules |
| **Checkout** | Repository | 8 | Address queries, order preview |
| **Payment** | Service Logic | 15 | HMAC signature verification, webhook validation, idempotency |
| **Payment** | Repository | 6 | Payment intent CRUD |
| **Order** | Service Logic | 18 | Status transitions, coin calculation, order history |
| **Order** | Repository | 10 | Order CRUD, status updates |

**Example Unit Test (Cart Service)**:
```typescript
describe('CartService', () => {
  describe('addItemToCart', () => {
    it('should enforce single-chef cart rule', async () => {
      // Arrange
      const existingCart = { chefId: 'chef-A' };
      const newItem = { chefId: 'chef-B', menuItemId: 'item-1', quantity: 2 };

      // Act & Assert
      await expect(cartService.addItemToCart(userId, newItem))
        .rejects.toThrow(CartConflictException);
    });

    it('should capture price snapshot on add', async () => {
      // Arrange
      const menuItem = { id: 'item-1', price: 350, title: 'Biryani' };
      const cartItem = { menuItemId: 'item-1', quantity: 2 };

      // Act
      const result = await cartService.addItemToCart(userId, cartItem);

      // Assert
      expect(result.priceSnapshot).toBe(350);
      expect(result.titleSnapshot).toBe('Biryani');
    });
  });
});
```

---

#### **2. Integration Tests (20% Coverage)**

| Integration | Test Category | Test Count | Examples |
|-------------|--------------|------------|----------|
| **Cart → Checkout** | API Flow | 12 | Cart to checkout transition, validation errors |
| **Checkout → Payment** | Razorpay Mock | 10 | Razorpay order creation, payment verification |
| **Payment → Order** | Webhook | 8 | Webhook signature, status update idempotency |
| **Order → Notification** | FCM Mock | 6 | Push notification on status change |

**Example Integration Test (Checkout → Payment)**:
```typescript
describe('Checkout to Payment Flow (Integration)', () => {
  it('should create Razorpay order on checkout validation', async () => {
    // Arrange
    const cart = await createTestCart();
    const address = await createTestAddress();
    const razorpayMock = jest.spyOn(razorpayService, 'createOrder')
      .mockResolvedValue({ id: 'order_abc123', amount: 74200 });

    // Act
    const response = await request(app.getHttpServer())
      .post('/api/v1/checkout/validate')
      .set('Authorization', `Bearer ${jwtToken}`)
      .send({ cartId: cart.id, addressId: address.id, paymentMethod: 'UPI_INTENT' });

    // Assert
    expect(response.status).toBe(200);
    expect(response.body.data.razorpayOrderId).toBe('order_abc123');
    expect(razorpayMock).toHaveBeenCalledWith({ amount: 74200, currency: 'INR' });
  });
});
```

---

#### **3. E2E Tests (10% Coverage)**

| Journey Scenario | Test Tool | Test Count | Examples |
|------------------|-----------|------------|----------|
| **Happy Path** | Cypress | 5 | Full journey: Feed → Order Placed |
| **Error Scenarios** | Cypress | 8 | Cart conflict, address out of radius, payment failure |
| **Payment Flows** | Razorpay Sandbox | 4 | UPI Intent, COD, Card, Payment timeout |
| **Performance** | k6 Load Testing | 3 | Concurrent users, API response times |

**Example E2E Test (Cypress)**:
```javascript
describe('Customer Order Journey (E2E)', () => {
  it('should complete full order flow from feed to order placed', () => {
    // Step 1: Login
    cy.login('customer@test.com', 'password');

    // Step 2: Browse Feed
    cy.visit('/feed');
    cy.get('[data-testid="reel-card"]').first().click();
    cy.get('[data-testid="order-now-cta"]').click();

    // Step 3: View Chef Menu
    cy.url().should('include', '/chef/');
    cy.get('[data-testid="menu-item"]').first().within(() => {
      cy.get('[data-testid="add-to-cart"]').click();
    });

    // Step 4: Review Cart
    cy.get('[data-testid="cart-icon"]').click();
    cy.get('[data-testid="cart-item"]').should('have.length', 1);
    cy.get('[data-testid="proceed-checkout"]').click();

    // Step 5: Select Address
    cy.get('[data-testid="address-card"]').first().within(() => {
      cy.get('[data-testid="deliver-here"]').click();
    });

    // Step 6: Confirm Checkout
    cy.get('[data-testid="place-order"]').click();

    // Step 7: Mock Payment (Razorpay Sandbox)
    cy.mockRazorpayPayment({ success: true });

    // Step 8: Verify Order Placed
    cy.url().should('include', '/orders/');
    cy.get('[data-testid="order-status"]').should('contain', 'CONFIRMED');
  });
});
```

---

### **Testing Checklist**

#### **Functional Tests**

- [ ] **Feed Module**:
  - [ ] Feed loads with ranked reels
  - [ ] CTA click navigates to chef menu
  - [ ] Following-only filter works correctly

- [ ] **Cart Module**:
  - [ ] Add item to cart (new cart)
  - [ ] Add item to cart (existing cart, same chef)
  - [ ] Cart conflict (different chef) shows modal
  - [ ] Update cart item quantity
  - [ ] Remove cart item
  - [ ] Cart syncs across devices (JWT)

- [ ] **Checkout Module**:
  - [ ] Fetch user addresses
  - [ ] Select address within radius
  - [ ] Address out of radius shows error
  - [ ] Delivery fee calculated correctly (distance-based)
  - [ ] Checkout validation passes (happy path)
  - [ ] Checkout validation fails (item unavailable, chef offline)

- [ ] **Payment Module**:
  - [ ] Razorpay order created successfully
  - [ ] Payment signature verified (HMAC SHA256)
  - [ ] Invalid signature rejected
  - [ ] Webhook processed correctly
  - [ ] Idempotency check (duplicate webhook)
  - [ ] COD bypasses payment gateway

- [ ] **Order Module**:
  - [ ] Order created with status `PENDING_PAYMENT`
  - [ ] Order status updated to `CONFIRMED` on payment
  - [ ] Status transitions follow lifecycle
  - [ ] Push notifications sent on status change
  - [ ] Coins credited on delivery
  - [ ] Reorder feature adds items to cart

#### **Integration Tests**

- [ ] **Cart → Checkout**:
  - [ ] Cart items passed to checkout
  - [ ] Subtotal calculated correctly

- [ ] **Checkout → Payment**:
  - [ ] Razorpay order ID returned
  - [ ] Order record created

- [ ] **Payment → Order**:
  - [ ] Payment success updates order status
  - [ ] Payment failure leaves order in `PENDING_PAYMENT`

- [ ] **Order → Notification**:
  - [ ] Status change triggers push notification
  - [ ] Notification payload correct

#### **E2E Tests**

- [ ] **Full Journey**:
  - [ ] Browse feed → Add to cart → Checkout → Pay → Track order
  - [ ] Reorder from order history

- [ ] **Error Scenarios**:
  - [ ] Cart conflict (different chef)
  - [ ] Address out of radius
  - [ ] Payment verification failed
  - [ ] Chef offline at checkout

#### **Performance Tests**

- [ ] **Load Testing**:
  - [ ] 100 concurrent users browsing feed
  - [ ] 50 concurrent users adding to cart
  - [ ] 20 concurrent users completing checkout

- [ ] **Response Time**:
  - [ ] Feed API: <800ms (p95)
  - [ ] Cart API: <300ms (p95)
  - [ ] Checkout API: <500ms (p95)
  - [ ] Payment verification: <1000ms (p95)

---

## Related Documentation

### **Module Documentation (Week 1-6)**

| Module | Feature Overview | Technical Guide | QA Test Cases | Module Complete |
|--------|------------------|-----------------|---------------|----------------|
| **Feed** | [FEATURE_OVERVIEW.md](../modules/feed/FEATURE_OVERVIEW.md) | [TECHNICAL_GUIDE.md](../modules/feed/TECHNICAL_GUIDE.md) | [QA_TEST_CASES.md](../modules/feed/QA_TEST_CASES.md) | [MODULE_COMPLETE.md](../modules/feed/MODULE_COMPLETE.md) |
| **Explore** | [FEATURE_OVERVIEW.md](../modules/explore/FEATURE_OVERVIEW.md) | [TECHNICAL_GUIDE.md](../modules/explore/TECHNICAL_GUIDE.md) | [QA_TEST_CASES.md](../modules/explore/QA_TEST_CASES.md) | [MODULE_COMPLETE.md](../modules/explore/MODULE_COMPLETE.md) |
| **Chef-Public** | [01_FEATURE_OVERVIEW.md](../modules/chef-public/01_FEATURE_OVERVIEW.md) | [02_TECHNICAL_GUIDE.md](../modules/chef-public/02_TECHNICAL_GUIDE.md) | [03_QA_TEST_CASES.md](../modules/chef-public/03_QA_TEST_CASES.md) | [MODULE_COMPLETE.md](../modules/chef-public/MODULE_COMPLETE.md) |
| **Cart** | [01_FEATURE_OVERVIEW.md](../modules/cart/01_FEATURE_OVERVIEW.md) | [02_TECHNICAL_GUIDE.md](../modules/cart/02_TECHNICAL_GUIDE.md) | [03_QA_TEST_CASES.md](../modules/cart/03_QA_TEST_CASES.md) | [MODULE_COMPLETE.md](../modules/cart/MODULE_COMPLETE.md) |
| **Checkout** | [FEATURE_OVERVIEW.md](../modules/checkout/FEATURE_OVERVIEW.md) | [TECHNICAL_GUIDE.md](../modules/checkout/TECHNICAL_GUIDE.md) | [QA_TEST_CASES.md](../modules/checkout/QA_TEST_CASES.md) | [MODULE_COMPLETE.md](../modules/checkout/MODULE_COMPLETE.md) |
| **Payment** | [FEATURE_OVERVIEW.md](../modules/payment/FEATURE_OVERVIEW.md) | [TECHNICAL_GUIDE.md](../modules/payment/TECHNICAL_GUIDE.md) | [QA_TEST_CASES.md](../modules/payment/QA_TEST_CASES.md) | [MODULE_COMPLETE.md](../modules/payment/MODULE_COMPLETE.md) |
| **Order** | [FEATURE_OVERVIEW.md](../modules/order/FEATURE_OVERVIEW.md) | [TECHNICAL_GUIDE.md](../modules/order/TECHNICAL_GUIDE.md) | [QA_TEST_CASES.md](../modules/order/QA_TEST_CASES.md) | [MODULE_COMPLETE.md](../modules/order/MODULE_COMPLETE.md) |
| **Auth** | [01_FEATURE_OVERVIEW.md](../modules/auth/01_FEATURE_OVERVIEW.md) | [TECHNICAL_GUIDE.md](../modules/auth/TECHNICAL_GUIDE.md) | [QA_TEST_CASES.md](../modules/auth/QA_TEST_CASES.md) | [AUTH_MODULE_COMPLETE.md](../modules/auth/AUTH_MODULE_COMPLETE.md) |

---

### **Journey Tracker**

- [JOURNEYS_STATUS.md](./JOURNEYS_STATUS.md) — Track progress across all 7 journey documents

---

### **Architecture & Technical Specs**

- [Monorepo Architecture](../../README.md) — Nx monorepo structure
- [Global Development Rules](../../.github/copilot-instructions.md) — Coding standards
- [API Versioning](../architecture/API_VERSIONING.md) — API v1, v2 strategy
- [Database Schema](../architecture/DATABASE_SCHEMA.md) — PostgreSQL + MongoDB models
- [Caching Strategy](../architecture/CACHING.md) — Valkey/Redis caching patterns
- [Push Notifications](../architecture/PUSH_NOTIFICATIONS.md) — FCM integration

---

### **Business & Product Docs**

- [Business Model Canvas](../business/BUSINESS_MODEL.md) — Revenue streams, cost structure
- [User Personas](../business/USER_PERSONAS.md) — Customer, Chef, Admin personas
- [Competitive Analysis](../business/COMPETITIVE_ANALYSIS.md) — Zomato, Swiggy comparison
- [Feature Roadmap](../business/ROADMAP.md) — Q1-Q4 2026 feature priorities

---

### **Operational Runbooks**

- [Deployment Guide](../ops/DEPLOYMENT.md) — Staging, production deployment steps
- [Monitoring & Alerts](../ops/MONITORING.md) — Datadog dashboards, alert thresholds
- [Incident Response](../ops/INCIDENT_RESPONSE.md) — Severity levels, escalation paths
- [Database Backup](../ops/BACKUP_RESTORE.md) — Backup schedule, restore procedures

---

## 🎉 Journey Complete Marker

**Customer Order Journey Documentation Complete!**

**Summary**:
- ✅ 11 sections documented with comprehensive detail
- ✅ 7 Mermaid diagrams created (sequence, flowchart, stateDiagram)
- ✅ 8 modules integrated (Feed, Explore, Chef-Public, Cart, Checkout, Payment, Order, Auth)
- ✅ 21 analytics events defined with payload examples
- ✅ 12 error scenarios with recovery paths
- ✅ Performance SLAs documented (API, DB, cache)
- ✅ Testing strategy: Unit (70%) + Integration (20%) + E2E (10%)
- ✅ 3,500+ lines of detailed documentation

**Total Lines**: 3,502 lines

**Completion Date**: 2026-02-15

**Next Journey**: [CHEF_ONBOARDING_JOURNEY.md](./CHEF_ONBOARDING_JOURNEY.md)

**Resume Command**: "Continue with CHEF_ONBOARDING_JOURNEY creation, then update JOURNEYS_STATUS.md"

---

**[SLICE_COMPLETE ✅]**
**[JOURNEY: CUSTOMER_ORDER_JOURNEY — COMPLETE]**
