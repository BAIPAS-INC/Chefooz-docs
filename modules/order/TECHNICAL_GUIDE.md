# 🔧 Order Module - Technical Guide

**Module**: Order Management  
**Version**: 2.0 (Multi-Item Orders + Cart Integration)  
**Last Updated**: 2026-02-15  
**Status**: ✅ Production Ready

---

## 📋 **Table of Contents**

1. [Architecture Overview](#architecture-overview)
2. [Database Schema](#database-schema)
3. [API Endpoints](#api-endpoints)
4. [Service Layer](#service-layer)
5. [Frontend Integration](#frontend-integration)
6. [Shared Libraries](#shared-libraries)
7. [Error Handling](#error-handling)
8. [Security](#security)
9. [Performance](#performance)
10. [Testing](#testing)
11. [Deployment](#deployment)

---

## 🏗️ **Architecture Overview**

### System Context Diagram

```
┌────────────────────────────────────────────────────────────┐
│                     Order Module                           │
│                                                            │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐ │
│  │   Customer   │───│  Order API   │───│  OrderService │ │
│  │   (Mobile)   │   │ (Controller) │   │    (Logic)    │ │
│  └──────────────┘   └──────────────┘   └──────────────┘ │
│                            │                    │          │
│                            │                    ├─────────>│ PostgreSQL
│                            │                    │          │ (Orders, History)
│                            │                    │          │
│                            │                    ├─────────>│ Razorpay
│                            │                    │          │ (Payments)
│                            │                    │          │
│                            │                    ├─────────>│ Valkey/Redis
│                            │                    │          │ (Locks, Rate Limiting)
│                            │                    │          │
│                            │                    ├─────────>│ Notification
│                            │                    │          │ (Status Updates)
│                            │                    │          │
│                            │                    └─────────>│ Commission
│                                                            │ (On Delivery)
└────────────────────────────────────────────────────────────┘
```

### Component Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Backend (NestJS)                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  order.controller.ts    ─────>  order.service.ts           │
│  (HTTP Endpoints)                (Business Logic)          │
│       │                                  │                  │
│       │                                  ├─> PostgreSQL    │
│       │                                  ├─> Razorpay      │
│       │                                  ├─> Valkey        │
│       │                                  └─> Notification  │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐  │
│  │  Supporting Services                                │  │
│  ├─────────────────────────────────────────────────────┤  │
│  │  • order-abuse-signals.service.ts                   │  │
│  │  • order-abuse-policy-logger.service.ts             │  │
│  │  • pricing.service.ts (industry-grade)              │  │
│  │  • chef-kitchen.service.ts (availability check)     │  │
│  │  • cart.service.ts (reorder-to-cart)                │  │
│  │  • commission.service.ts (attribution)              │  │
│  │  • delivery-assignment.service.ts (rider)           │  │
│  └─────────────────────────────────────────────────────┘  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                           │
                           │ REST API
                           │
┌─────────────────────────────────────────────────────────────┐
│                    Shared Libraries                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  libs/types/order.types.ts   ────> TypeScript Interfaces   │
│  libs/api-client/order/      ────> Axios Clients           │
│  libs/api-client/order/      ────> React Query Hooks       │
│  libs/domain/order/          ────> Business Rules          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                           │
                           │ Imports
                           │
┌─────────────────────────────────────────────────────────────┐
│                 Frontend (Expo Router)                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  /app/orders.tsx              ────> Order History          │
│  /app/orders/[orderId].tsx    ────> Order Details          │
│  /app/cart/payment.tsx        ────> Payment Screen         │
│                                                             │
│  Hooks:                                                     │
│  • useOrderHistory()          ────> Fetch orders           │
│  • useOrder()                 ────> Fetch order details    │
│  • useCreatePaymentIntent()   ────> UPI/COD payment        │
│  • useReorderToCart()         ────> Reorder functionality  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Technology Stack

| Layer | Technology | Purpose |
|-------|------------|---------|
| **Backend API** | NestJS 10+ | REST API framework |
| **Database (Primary)** | PostgreSQL 15+ | Transactional order data |
| **Database (MongoDB)** | MongoDB 6+ | Reel metadata (for attribution) |
| **Cache & Locks** | Valkey (Redis) | Distributed locks, rate limiting |
| **Payment Gateway** | Razorpay | UPI Intent, signature verification |
| **Frontend** | Expo Router | Mobile navigation |
| **State Management** | React Query | Server state caching |
| **HTTP Client** | Axios | API communication |
| **Validation** | class-validator | DTO validation |

---

## 🗄️ **Database Schema**

### Primary Table: `orders`

```sql
CREATE TABLE orders (
  -- Primary Key
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  
  -- Foreign Keys
  "userId" UUID NOT NULL REFERENCES users(id),
  "chefId" UUID NOT NULL,
  "deliveryPartnerId" UUID REFERENCES users(id),
  
  -- Order Data
  items JSONB NOT NULL,  -- Array of OrderItemSnapshot
  instructions TEXT,
  status VARCHAR(50) NOT NULL DEFAULT 'created',
  
  -- Pricing (in paise)
  "subtotalPaise" INT NOT NULL,
  "deliveryFeePaise" INT,
  "taxFeePaise" INT,
  "totalPaise" INT NOT NULL,
  "pricingBreakdown" JSONB,
  
  -- Address Snapshot
  "addressSnapshot" JSONB,
  
  -- Payment Fields
  "paymentStatus" VARCHAR(50) NOT NULL DEFAULT 'not_initialized',
  "paymentMethod" VARCHAR(20),  -- UPI_INTENT, COD
  "paymentIntentId" VARCHAR(255),  -- Legacy
  "razorpayOrderId" VARCHAR(255),
  "razorpayPaymentId" VARCHAR(255),
  "razorpaySignature" VARCHAR(255),
  "paymentReferenceId" VARCHAR(255),
  "paymentTxnId" VARCHAR(255),
  "paymentMeta" JSONB,
  "lockExpiresAt" TIMESTAMP,  -- 15-minute payment lock
  
  -- Chef Kitchen Management
  "chefStatus" VARCHAR(50) DEFAULT 'NEW',
  "chefNote" TEXT,
  "estimatedPrepMinutes" INT,
  "acceptedAt" TIMESTAMP,
  "rejectedAt" TIMESTAMP,
  "readyAt" TIMESTAMP,
  
  -- Delivery Management
  "deliveryStatus" VARCHAR(50),
  "deliveryDecisionReason" VARCHAR(255),
  "deliveryAssignedAt" TIMESTAMP,
  "deliveryPickedUpAt" TIMESTAMP,
  "deliveryDeliveredAt" TIMESTAMP,
  "deliveryEtaMinutes" INT,
  
  -- Commission Attribution
  attribution JSONB,  -- { linkedReelId, creatorUserId, creatorOrderValue, exploreAttribution }
  
  -- Timestamps
  "createdAt" TIMESTAMP NOT NULL DEFAULT NOW(),
  "updatedAt" TIMESTAMP NOT NULL DEFAULT NOW(),
  "completedAt" TIMESTAMP
);

-- Indexes for Performance
CREATE INDEX idx_order_user_created ON orders("userId", "createdAt");
CREATE INDEX idx_order_chef ON orders("chefId");
CREATE INDEX idx_order_status ON orders(status);
CREATE INDEX idx_chef_status ON orders("chefStatus");
CREATE INDEX idx_razorpay_order_id ON orders("razorpayOrderId");
CREATE INDEX idx_razorpay_payment_id ON orders("razorpayPaymentId");
CREATE INDEX idx_delivery_partner ON orders("deliveryPartnerId");
```

### JSONB Structure: `items` (OrderItemSnapshot[])

```typescript
[
  {
    "id": "item-uuid-1",
    "menuItemId": "menu-item-uuid",
    "quantity": 2,
    "unitPricePaise": 25000,  // ₹250
    "titleSnapshot": "Margherita Pizza",
    "imageSnapshot": "https://cdn.chefooz.com/...",
    "optionsSnapshot": {
      "size": "Large",
      "crust": "Thin"
    },
    "removedIngredients": ["onions"],
    "addedIngredients": [
      {
        "name": "Extra Cheese",
        "pricePaise": 3000  // ₹30
      }
    ],
    "customerCookingPreferences": "Extra crispy",
    "foodType": "veg",
    "prepTimeMinutes": 25,
    "description": "Classic Italian pizza with fresh mozzarella"
  }
]
```

### JSONB Structure: `pricingBreakdown`

```typescript
{
  "items": [
    {
      "id": "item-uuid-1",
      "pricePaise": 25000,
      "quantity": 2,
      "subtotalPaise": 50000,
      "gstExempt": false
    }
  ],
  "subtotal": 45000,
  "packaging": 1000,
  "delivery": {
    "baseFeePaise": 2000,
    "distanceFeePaise": 1000,
    "surgeMultiplier": 1.3,
    "totalDeliveryFeePaise": 3900
  },
  "gst": {
    "cgstPaise": 1200,
    "sgstPaise": 1200,
    "totalGstPaise": 2400
  },
  "platformFee": 500,
  "grandTotal": 51800
}
```

### JSONB Structure: `attribution`

```typescript
{
  // Reel Attribution (for commission)
  "linkedReelId": "reel-uuid",
  "linkedCreatorOrderId": "order-uuid-from-creator",
  "creatorUserId": "chef-uuid",
  "creatorOrderValue": 45000,  // ₹450 in paise (food value only)
  
  // Explore Attribution (for analytics)
  "exploreAttribution": {
    "sourceItemId": "reel-or-chef-uuid",
    "layoutVariant": "FOOD_GRID",
    "timeToOrderMs": 45000,
    "impressionTimestamp": "2026-02-15T10:00:00Z",
    "interactionTimestamp": "2026-02-15T10:00:45Z"
  }
}
```

### JSONB Structure: `addressSnapshot`

```typescript
{
  "label": "Home",
  "fullName": "Priya Sharma",
  "phone": "+919876543210",
  "line1": "123, MG Road",
  "line2": "Near Central Park",
  "city": "Bengaluru",
  "state": "Karnataka",
  "pincode": "560001",
  "country": "India",
  "lat": 12.9716,
  "lng": 77.5946
}
```

---

### Supporting Tables

#### `order_status_history`
```sql
CREATE TABLE order_status_history (
  id UUID PRIMARY KEY,
  "orderId" UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  "previousStatus" VARCHAR(50),
  "newStatus" VARCHAR(50) NOT NULL,
  reason TEXT,
  "triggeredBy" VARCHAR(50),  -- 'user', 'system', 'chef', 'rider'
  "createdAt" TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_order_status_history_order ON order_status_history("orderId");
```

#### `order_events`
```sql
CREATE TABLE order_events (
  id UUID PRIMARY KEY,
  "orderId" UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  "eventType" VARCHAR(100) NOT NULL,  -- 'order.created', 'payment.confirmed', etc.
  "eventData" JSONB,
  "createdAt" TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_order_events_order ON order_events("orderId");
CREATE INDEX idx_order_events_type ON order_events("eventType");
```

---

## 🌐 **API Endpoints**

### Base URL
```
https://api.chefooz.com/api/v1
```

All endpoints require JWT authentication unless specified.

---

### 1. Get Menu Item

**Endpoint**: `GET /menu-items/:id`

**Purpose**: Fetch menu item details for order creation.

**Request**:
```http
GET /api/v1/menu-items/550e8400-e29b-41d4-a716-446655440000
Authorization: Bearer <JWT_TOKEN>
```

**Response** (200):
```json
{
  "success": true,
  "message": "Menu item retrieved successfully",
  "data": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "title": "Margherita Pizza",
    "description": "Classic Italian pizza",
    "pricePaise": 25000,
    "imageUrl": "https://cdn.chefooz.com/...",
    "foodType": "veg",
    "prepTimeMinutes": 25,
    "availability": {
      "isAvailable": true,
      "reason": null
    }
  }
}
```

**Error** (404):
```json
{
  "success": false,
  "message": "Menu item not found",
  "errorCode": "ORDER_MENU_ITEM_NOT_FOUND"
}
```

---

### 2. Create Draft Order (Legacy - Direct from Reel)

**Endpoint**: `POST /orders`

**Purpose**: Create draft order directly from reel CTA (bypasses cart).

**Rate Limit**: 20 requests per minute per user

**Request**:
```http
POST /api/v1/orders
Authorization: Bearer <JWT_TOKEN>
Content-Type: application/json

{
  "chefId": "chef-uuid",
  "items": [
    {
      "menuItemId": "menu-item-uuid",
      "quantity": 2,
      "customizations": {
        "size": "Large",
        "crust": "Thin"
      },
      "removedIngredients": ["onions"],
      "addedIngredients": [
        {
          "name": "Extra Cheese",
          "pricePaise": 3000
        }
      ],
      "customerCookingPreferences": "Extra crispy"
    }
  ],
  "instructions": "Ring doorbell twice",
  "attribution": {
    "linkedReelId": "reel-uuid",
    "linkedCreatorOrderId": "creator-order-uuid",
    "creatorOrderValue": 45000
  }
}
```

**Response** (201):
```json
{
  "success": true,
  "message": "Draft order created successfully",
  "data": {
    "id": "order-uuid",
    "userId": "user-uuid",
    "chefId": "chef-uuid",
    "items": [ /* OrderItemSnapshot[] */ ],
    "status": "created",
    "subtotalPaise": 50000,
    "totalPaise": 50000,
    "createdAt": "2026-02-15T10:00:00Z"
  }
}
```

**Errors**:
- `404`: Menu item not found or unavailable
- `403`: Chef not accepting orders or outside service hours
- `429`: Rate limit exceeded (20 orders/min)

---

### 3. Checkout Order

**Endpoint**: `POST /orders/checkout`

**Purpose**: Attach address, calculate fees, prepare for payment.

**Rate Limit**: 10 requests per minute per user

**Distributed Lock**: 10 seconds on `order:checkout:{orderId}`

**Request**:
```http
POST /api/v1/orders/checkout
Authorization: Bearer <JWT_TOKEN>
Content-Type: application/json

{
  "orderId": "order-uuid",
  "addressId": "address-uuid"
}
```

**Response** (200):
```json
{
  "success": true,
  "message": "Order checked out successfully",
  "data": {
    "paymentIntentId": "stub_intent_12345",
    "totalPaise": 51800
  }
}
```

**Business Logic**:
1. Validate order is in `CREATED` or `PAYMENT_PENDING` status
2. Load address and create snapshot
3. Calculate distance (chef location → delivery address)
4. Call `PricingService.calculatePricing()`:
   - Item subtotal
   - Packaging fee
   - Delivery fee (distance-based + surge)
   - GST (5% on subtotal + packaging + delivery)
   - Platform fee (optional)
5. Update order with pricing breakdown
6. Create 15-minute payment lock
7. Return payment intent stub

**Errors**:
- `404`: Order or address not found
- `403`: Order not in valid status
- `400`: Distance exceeds 15 km limit

---

### 4. Create Payment Intent

**Endpoint**: `POST /orders/payment-intent`

**Purpose**: Initialize UPI Intent or confirm COD payment.

**Rate Limit**: 10 requests per minute per user

**Request** (UPI Intent):
```http
POST /api/v1/orders/payment-intent
Authorization: Bearer <JWT_TOKEN>
Content-Type: application/json

{
  "orderId": "order-uuid",
  "paymentMethod": "UPI_INTENT"
}
```

**Response** (200 - UPI):
```json
{
  "success": true,
  "message": "UPI intent created successfully",
  "data": {
    "orderId": "order-uuid",
    "paymentMethod": "UPI_INTENT",
    "upiIntentUrl": "upi://pay?pa=merchant@paytm&pn=Chefooz&am=518.00&tn=Order%20order-uuid",
    "expiresAt": "2026-02-15T10:15:00Z",
    "isStubbed": false
  }
}
```

**Request** (COD):
```http
POST /api/v1/orders/payment-intent
Authorization: Bearer <JWT_TOKEN>
Content-Type: application/json

{
  "orderId": "order-uuid",
  "paymentMethod": "COD"
}
```

**Response** (200 - COD):
```json
{
  "success": true,
  "message": "COD order confirmed",
  "data": {
    "orderId": "order-uuid",
    "paymentMethod": "COD",
    "isStubbed": true
  }
}
```

**Business Logic**:

**UPI Flow**:
1. Load order and validate status
2. Create Razorpay order via `RazorpayOrderService.createOrder()`
3. Generate UPI Intent deep link
4. Update order:
   - `paymentMethod = 'UPI_INTENT'`
   - `paymentStatus = 'pending'`
   - `razorpayOrderId` stored
5. Return UPI deep link (opens GPay/PhonePe/BHIM)

**COD Flow**:
1. Load order and validate status
2. Check user trust state (RESTRICTED users cannot use COD)
3. Validate COD limit (max 3 per day)
4. Auto-confirm order:
   - `paymentMethod = 'COD'`
   - `paymentStatus = 'paid'`
   - `status = 'paid'`
5. Trigger notifications (chef, customer)
6. Return success (no UPI flow)

**Errors**:
- `404`: Order not found
- `403`: COD disabled for trust state
- `400`: COD daily limit exceeded

---

### 5. Confirm Payment

**Endpoint**: `POST /orders/confirm-payment`

**Purpose**: Verify Razorpay payment signature and mark order as paid.

**Rate Limit**: 10 requests per minute per user

**Distributed Lock**: 10 seconds on `payment:confirm:{orderId}`

**Request**:
```http
POST /api/v1/orders/confirm-payment
Authorization: Bearer <JWT_TOKEN>
Content-Type: application/json

{
  "orderId": "order-uuid",
  "razorpayOrderId": "order_NdGwBw2lA3WI2a",
  "razorpayPaymentId": "pay_NdGwBw2lA3WI2b",
  "razorpaySignature": "9ef4dffbfd84f1318f6739a3ce19f9d85851857ae648f114332d8401e0949a3d"
}
```

**Response** (200):
```json
{
  "success": true,
  "message": "Payment confirmed successfully"
}
```

**Business Logic**:
1. Load order with lock
2. Verify Razorpay signature using `RazorpayCheckoutService.verifyPaymentSignature()`
3. Update order:
   - `paymentStatus = 'paid'`
   - `status = 'paid'`
   - `razorpayPaymentId` and `razorpaySignature` stored
4. Create status history entry
5. Send notifications:
   - Customer: "Payment confirmed"
   - Chef: "New order from [Customer Name]"
6. (Future) Trigger commission calculation

**Errors**:
- `400`: Invalid signature (security violation)
- `404`: Order not found
- `409`: Order already paid (idempotency check)

---

### 6. Get Order History

**Endpoint**: `GET /orders/history`

**Purpose**: Fetch user's order history with cursor pagination.

**Query Parameters**:
- `cursor` (optional): Pagination cursor from previous response
- `limit` (optional, default 20): Number of orders per page (max 50)
- `status` (optional): Filter by status (`delivered`, `cancelled`, `all`)

**Request**:
```http
GET /api/v1/orders/history?cursor=2026-02-15T10:00:00Z&limit=20&status=delivered
Authorization: Bearer <JWT_TOKEN>
```

**Response** (200):
```json
{
  "success": true,
  "message": "Order history retrieved successfully",
  "data": {
    "cursor": "2026-02-10T08:30:00Z",
    "items": [
      {
        "id": "order-uuid-1",
        "createdAt": "2026-02-15T10:00:00Z",
        "status": "delivered",
        "deliveryStatus": "DELIVERED",
        "totalPaise": 51800,
        "chefId": "chef-uuid",
        "chefName": "Masterchef Ravi",
        "itemCount": 2,
        "thumbnailUrl": "https://cdn.chefooz.com/...",
        "attribution": {
          "linkedReelId": "reel-uuid",
          "creatorOrderValue": 45000
        }
      }
    ]
  }
}
```

**Pagination**:
- Uses cursor-based pagination (not offset)
- Cursor = `createdAt` timestamp of last order
- Next request includes `cursor` parameter
- Returns `cursor: null` when no more results

---

### 7. Get Order Details

**Endpoint**: `GET /orders/:id`

**Purpose**: Fetch complete order details including items, pricing, status.

**Request**:
```http
GET /api/v1/orders/550e8400-e29b-41d4-a716-446655440000
Authorization: Bearer <JWT_TOKEN>
```

**Response** (200):
```json
{
  "success": true,
  "message": "Order retrieved successfully",
  "data": {
    "id": "order-uuid",
    "userId": "user-uuid",
    "chefId": "chef-uuid",
    "items": [
      {
        "id": "item-uuid",
        "menuItemId": "menu-item-uuid",
        "quantity": 2,
        "unitPricePaise": 25000,
        "titleSnapshot": "Margherita Pizza",
        "imageSnapshot": "https://cdn.chefooz.com/...",
        "optionsSnapshot": {
          "size": "Large"
        },
        "removedIngredients": ["onions"],
        "addedIngredients": [],
        "foodType": "veg"
      }
    ],
    "instructions": "Ring doorbell twice",
    "status": "out_for_delivery",
    "deliveryStatus": "OUT_FOR_DELIVERY",
    "subtotalPaise": 50000,
    "deliveryFeePaise": 3900,
    "taxFeePaise": 2400,
    "totalPaise": 51800,
    "pricingBreakdown": { /* See schema above */ },
    "addressSnapshot": { /* See schema above */ },
    "paymentStatus": "paid",
    "paymentMethod": "UPI_INTENT",
    "razorpayOrderId": "order_NdGwBw2lA3WI2a",
    "chefStatus": "READY",
    "estimatedPrepMinutes": 25,
    "deliveryEtaMinutes": 10,
    "createdAt": "2026-02-15T10:00:00Z",
    "updatedAt": "2026-02-15T10:30:00Z"
  }
}
```

**Access Control**:
- Only visible to:
  - Customer who placed the order
  - Chef who received the order
  - Assigned delivery rider
  - Admin with proper role

**Errors**:
- `404`: Order not found
- `403`: Unauthorized access

---

### 8. Get Order Live Status

**Endpoint**: `GET /orders/:id/live`

**Purpose**: Real-time tracking data (ETA, rider info, status).

**Request**:
```http
GET /api/v1/orders/550e8400-e29b-41d4-a716-446655440000/live
Authorization: Bearer <JWT_TOKEN>
```

**Response** (200):
```json
{
  "success": true,
  "message": "Live status retrieved successfully",
  "data": {
    "orderId": "order-uuid",
    "status": "out_for_delivery",
    "deliveryStatus": "OUT_FOR_DELIVERY",
    "estimatedPrepMinutes": null,
    "deliveryEtaMinutes": 10,
    "rider": {
      "id": "rider-uuid",
      "name": "Suresh Kumar",
      "phone": "+919876543210",
      "profilePicture": "https://cdn.chefooz.com/...",
      "location": {
        "lat": 12.9716,
        "lng": 77.5946
      }
    },
    "statusHistory": [
      {
        "status": "created",
        "timestamp": "2026-02-15T10:00:00Z"
      },
      {
        "status": "paid",
        "timestamp": "2026-02-15T10:02:00Z"
      },
      {
        "status": "out_for_delivery",
        "timestamp": "2026-02-15T10:30:00Z"
      }
    ]
  }
}
```

**Polling Recommendation**: Frontend should poll this endpoint every 30 seconds while order is active.

---

### 9. Reorder to Cart

**Endpoint**: `POST /orders/:id/reorder-to-cart`

**Purpose**: Clear cart and add all items from previous order with current pricing.

**Request**:
```http
POST /api/v1/orders/550e8400-e29b-41d4-a716-446655440000/reorder-to-cart
Authorization: Bearer <JWT_TOKEN>
```

**Response** (200):
```json
{
  "success": true,
  "message": "2 items added to cart",
  "data": {
    "itemsAdded": 2,
    "itemsUnavailable": 1,
    "chefId": "chef-uuid",
    "unavailableItems": [
      {
        "menuItemId": "menu-item-uuid-2",
        "titleSnapshot": "Garlic Bread",
        "reason": "Item no longer available"
      }
    ]
  }
}
```

**Business Logic**:
1. Load original order
2. Validate order is `DELIVERED` or `CANCELLED`
3. Clear user's cart via `CartService.clearCart()`
4. Iterate through order items:
   - Check if menu item still exists
   - Check if item is available
   - Add to cart with **current pricing** (not historical)
5. Return summary of added vs unavailable items

**Errors**:
- `404`: Order not found
- `403`: Order not in reorderable state
- `400`: All items unavailable

---

### 10. Track Order Telemetry

**Endpoint**: `POST /orders/:id/telemetry`

**Purpose**: Log user engagement with order tracking (for analytics).

**Request**:
```http
POST /api/v1/orders/550e8400-e29b-41d4-a716-446655440000/telemetry
Authorization: Bearer <JWT_TOKEN>
Content-Type: application/json

{
  "eventType": "order_status_viewed",
  "timestamp": "2026-02-15T10:30:00Z"
}
```

**Response** (200):
```json
{
  "success": true,
  "message": "Telemetry tracked"
}
```

**Event Types**:
- `order_status_viewed`: User viewed order details screen
- `order_tracking_opened`: User opened live tracking
- `rider_called`: User called delivery rider

**Usage**: Powers Explore ranking algorithm (engagement signals).

---

### 11. Razorpay Webhook

**Endpoint**: `POST /orders/razorpay/webhook`

**Purpose**: Handle Razorpay payment events (server-to-server).

**Security**: Signature verification required.

**Request**:
```http
POST /api/v1/orders/razorpay/webhook
X-Razorpay-Signature: 9ef4dffbfd84f1318f6739a3ce19f9d85851857ae648f114332d8401e0949a3d
Content-Type: application/json

{
  "event": "payment.captured",
  "payload": {
    "payment": {
      "entity": {
        "id": "pay_NdGwBw2lA3WI2b",
        "order_id": "order_NdGwBw2lA3WI2a",
        "amount": 51800,
        "currency": "INR",
        "status": "captured"
      }
    }
  }
}
```

**Response** (200):
```json
{
  "success": true,
  "message": "Webhook processed"
}
```

**Business Logic**:
1. Verify webhook signature
2. Parse event type
3. Handle event:
   - `payment.captured`: Mark order as paid
   - `payment.failed`: Mark payment failed, cancel order
4. Create audit event
5. Send notifications

---

## 🔧 **Service Layer**

### OrderService Methods

#### Core Order Operations

##### `createDraftOrder(dto: CreateOrderDto, userId: string): Promise<Order>`

**Purpose**: Create draft order directly from reel CTA (legacy flow).

**Algorithm**:
```typescript
1. Load and validate menu items
2. Validate chef availability:
   - Kitchen online?
   - Accepting orders?
   - Within service hours?
3. Calculate item subtotal
4. Validate order creation policy (abuse detection)
5. Build attribution metadata
6. Resolve Explore attribution (if applicable)
7. Create order entity with status = CREATED
8. Save to database
9. Return order
```

**Domain Rules Applied**:
- `canCreateOrder(chefContext, orderConfig)` from `libs/domain`
- `validateOrderCreation(context, config)` for abuse checks

**Rate Limiting**: 20 requests/min via `@RateLimit` decorator

---

##### `createOrderFromCart(data: CreateOrderFromCartDto): Promise<Order>`

**Purpose**: Create order from cart (modern flow, cart checkout).

**Algorithm**:
```typescript
1. Validate address exists
2. Calculate distance (chef → customer)
3. Get time conditions (peak hours?)
4. Call PricingService.calculatePricing():
   - Item subtotal
   - Packaging fee
   - Delivery fee (distance-based + surge)
   - GST breakdown (CGST + SGST)
   - Grand total
5. Validate totals (no NaN, positive)
6. Run abuse detection checks
7. Build attribution (reel + Explore)
8. Create order with address snapshot
9. Save order with pricingBreakdown
10. Return order
```

**Pricing Integration**:
```typescript
const pricingResult = await this.pricingService.calculatePricing({
  items: cartItems.map(item => ({
    id: item.id,
    pricePaise: item.unitPricePaise,
    quantity: item.quantity,
    gstExempt: false,
  })),
  delivery: {
    distanceKm,
    coordinates: { lat: address.lat, lng: address.lng },
    isPeakTime: timeConditions.isPeakTime,
  },
  chefId: cartItems[0].chefId,
});

// Extract breakdown
const { grandTotal, delivery, gst, packaging } = pricingResult;
```

**Error Handling**:
- Cart empty → `404 CART_EMPTY`
- Items unavailable → `400 CART_ITEMS_UNAVAILABLE`
- Chef unavailable → `403 CHEF_NOT_ACCEPTING_ORDERS`
- Distance too far → `400 DELIVERY_DISTANCE_EXCEEDED`

---

##### `checkout(orderId: string, addressId: string, userId: string)`

**Purpose**: Attach address, calculate fees, prepare for payment.

**Algorithm**:
```typescript
1. Load order (must be CREATED or PAYMENT_PENDING)
2. Load address and create snapshot
3. Calculate distance (chef location → delivery address)
4. Get time-based pricing conditions
5. Call PricingService.calculatePricing()
6. Update order:
   - addressSnapshot
   - deliveryFeePaise
   - taxFeePaise
   - totalPaise
   - pricingBreakdown
   - lockExpiresAt = now + 15 minutes
7. Save order
8. Return payment intent stub (for legacy)
```

**Idempotency**: Order can be checked out multiple times if payment hasn't started.

**Lock Mechanism**: 10-second distributed lock via `@WithLock` decorator prevents race conditions.

---

##### `createPaymentIntent(dto: CreatePaymentIntentDto, userId: string)`

**Purpose**: Initialize UPI Intent or confirm COD payment.

**UPI Flow**:
```typescript
1. Load order and validate status
2. Create Razorpay order:
   - amount = order.totalPaise
   - currency = INR
   - receipt = order.id
3. Generate UPI Intent URL:
   upi://pay?pa={merchant_vpa}&pn={merchant_name}&am={amount}&tn={order_id}
4. Update order:
   - paymentMethod = 'UPI_INTENT'
   - paymentStatus = 'pending'
   - razorpayOrderId = razorpayOrder.id
5. Return UPI Intent URL (opens in UPI app)
```

**COD Flow**:
```typescript
1. Load order and validate status
2. Check user trust state:
   - RESTRICTED/BANNED → Reject COD
3. Validate COD daily limit (max 3)
4. Auto-confirm order:
   - paymentMethod = 'COD'
   - paymentStatus = 'paid'
   - status = 'paid'
5. Send notifications (chef, customer)
6. Return success (no UPI flow)
```

**Abuse Detection**:
- COD limit enforced via `validateOrderCreationPolicy()`
- Trust state checked via `bypassesAbuseChecks()`

---

##### `confirmRazorpayPayment(orderId, dto, userId)`

**Purpose**: Verify Razorpay signature and mark order paid.

**Algorithm**:
```typescript
1. Load order with lock
2. Verify Razorpay signature:
   - Signature = HMAC_SHA256(orderId|paymentId, secret)
   - Compare with dto.razorpaySignature
3. If invalid → Throw security error
4. If valid:
   - paymentStatus = 'paid'
   - status = 'paid'
   - razorpayPaymentId = dto.razorpayPaymentId
   - razorpaySignature = dto.razorpaySignature
5. Create status history entry
6. Create order event (audit)
7. Send notifications:
   - Customer: "Payment confirmed"
   - Chef: "New order from [Name]"
8. Return success
```

**Security**:
- 10-second distributed lock prevents double-confirmation
- Signature verification critical (no bypass)
- All payment events logged to `order_events`

---

##### `getOrderHistory(userId, cursor, limit, status?)`

**Purpose**: Fetch user's order history with cursor pagination.

**Algorithm**:
```typescript
1. Build query:
   - WHERE userId = userId
   - AND createdAt < cursor (if cursor provided)
   - AND status = status (if filter provided)
   - ORDER BY createdAt DESC
   - LIMIT limit + 1 (to detect if more results)
2. Execute query with joins:
   - LEFT JOIN users (for chef name)
3. Map results to OrderHistoryItem:
   - Extract first item's image as thumbnail
   - Count total items
   - Include attribution (if exists)
4. Check if more results:
   - If results.length > limit → hasMore = true
   - nextCursor = results[limit - 1].createdAt
5. Return { items, cursor: nextCursor }
```

**Performance**:
- Uses composite index: `idx_order_user_created`
- Cursor-based pagination (no OFFSET, scales to millions)
- Limit capped at 50 to prevent abuse

---

##### `reorderToCart(orderId: string, userId: string)`

**Purpose**: Add all items from previous order to cart with current pricing.

**Algorithm**:
```typescript
1. Load original order
2. Validate order is DELIVERED or CANCELLED
3. Clear user's cart via cartService.clearCart()
4. Iterate through order.items:
   a. Load current menu item by menuItemId
   b. If not found or unavailable:
      - Add to unavailableItems array
      - Continue to next item
   c. If available:
      - Add to cart via cartService.addItem():
        - menuItemId
        - quantity (from order)
        - customizations (from order)
        - Use **current price** (not historical)
5. Invalidate cart caches
6. Return summary:
   - itemsAdded: 2
   - itemsUnavailable: 1
   - unavailableItems: [...]
```

**Key Behavior**:
- ✅ Uses **current pricing**, not historical prices
- ✅ Gracefully handles unavailable items
- ✅ Clears cart before adding (prevents mixed chefs)
- ✅ Preserves customizations from original order

---

##### `getOrderLiveStatus(orderId: string, userId: string)`

**Purpose**: Fetch real-time tracking data (ETA, rider info).

**Algorithm**:
```typescript
1. Load order with relations:
   - user (customer)
   - deliveryPartner (rider)
2. Validate access (user owns order or is chef/rider)
3. Load status history (timeline)
4. If rider assigned:
   - Fetch live location via CourierGateway
   - Calculate ETA based on distance and traffic
5. Build response:
   - Current status
   - Delivery status
   - ETA (prep time or delivery ETA)
   - Rider info (name, phone, profile pic, location)
   - Status history timeline
6. Return live status object
```

**Real-Time Data**:
- Rider location fetched from `courier_gateway` integration
- ETA calculated dynamically (not cached)
- Frontend should poll this endpoint every 30 seconds

---

#### Payment & Lifecycle Methods

##### `markPaymentSuccess(orderId: string)`

**Purpose**: Transition order to PAID status after payment confirmation.

**Algorithm**:
```typescript
1. Load order
2. Update:
   - paymentStatus = 'paid'
   - status = 'paid'
3. Create status history entry
4. Create order event (audit)
5. Send notifications:
   - Chef: "New order received"
   - Customer: "Order confirmed"
6. Save order
```

---

##### `markPaymentFailed(orderId: string, reason: string)`

**Purpose**: Handle payment failures and auto-cancel order.

**Algorithm**:
```typescript
1. Load order
2. Update:
   - paymentStatus = 'failed'
   - status = 'payment_failed'
3. Create status history entry
4. Send notification:
   - Customer: "Payment failed. Please try again."
5. Release order lock
6. Save order
```

---

##### `handleOrderDelivered(orderId: string)`

**Purpose**: Process post-delivery actions (coins, commission).

**Algorithm**:
```typescript
1. Load order with relations
2. Validate order is in DELIVERED status
3. Calculate creator order value (for commission):
   - creatorOrderValue = Sum of item prices (excluding delivery/tax)
4. If attribution exists:
   - Call commissionService.generateCommission()
   - Commission = 8% of creatorOrderValue
5. Credit coins to customer:
   - coinAmount = Sum of item prices (excluding delivery/tax)
   - Call userService.creditCoins(userId, coinAmount)
6. Create order event (audit)
7. Send notification:
   - Customer: "You earned X coins!"
8. Mark commission as generated (prevent double credit)
```

**MVP Beta**: 100% coins on delivery (₹450 food → 450 coins)

---

##### `autoCancelOrder(params: { orderId, reason })`

**Purpose**: System-initiated order cancellations (timeouts).

**Reasons**:
- `RIDER_NOT_ACCEPTED`: No rider accepted within 5 minutes
- `PICKUP_TIMEOUT`: Rider didn't pick up within 30 minutes
- `DELIVERY_TIMEOUT`: Order not delivered within 2 hours

**Algorithm**:
```typescript
1. Load order with user (for trust state)
2. Check if already terminal state → Skip
3. Validate cancellation via domain rules:
   - validateCancellation(context, config)
   - Always allowed for system actions
   - Log for trust state pattern analysis
4. Update order:
   - status = 'cancelled'
   - deliveryDecisionReason = reason
5. Create status history entry
6. Create order event (audit)
7. Send notifications:
   - Customer: "Order cancelled: [reason]"
   - Chef: "Order cancelled"
   - Rider: "Delivery cancelled" (if assigned)
8. Initiate refund (if paid):
   - Call razorpayOrderService.initiateRefund()
```

**Abuse Tracking**:
- System cancellations logged to `order_abuse_policy_logger`
- Patterns analyzed for trust state updates
- No user penalty for system-initiated cancellations

---

#### Supporting Methods

##### `validateChefAvailability(chefId: string)`

**Purpose**: Check if chef can accept new orders.

**Algorithm**:
```typescript
1. Get kitchen details via chefKitchenService
2. Check:
   - isOnline = true?
   - acceptingOrders = true?
3. If schedule exists:
   - Check current time within service hours
4. If any check fails:
   - Throw ForbiddenException('Chef not accepting orders')
```

**Domain Rule**: Uses `canCreateOrder()` from `libs/domain`

---

##### `validateOrderCreationPolicy(userId, totalPaise)`

**Purpose**: Abuse detection for order creation.

**Algorithm**:
```typescript
1. Load user with trust state
2. Fetch abuse signals:
   - recentCancellations (last 7 days)
   - failedPayments (last 24 hours)
   - activeOrders (unpaid)
3. Build context:
   - trustState
   - cancellationCount
   - failedPaymentCount
   - activeOrderCount
4. Call domain rule:
   validateOrderCreation(context, orderConfig)
5. If not allowed:
   - Log policy violation
   - Throw ForbiddenException
```

**Trust State Limits**:
```typescript
NORMAL:     3 active orders, 3 cancellations/week
WARNING:    2 active orders, 2 cancellations/week
RESTRICTED: 1 active order, 0 cancellations
BANNED:     0 orders allowed
```

---

##### `enrichAttribution(attribution)`

**Purpose**: Add creator metadata to attribution object.

**Algorithm**:
```typescript
1. If linkedReelId exists:
   - Load reel from MongoDB
   - Extract creatorUserId
2. If linkedCreatorOrderId exists:
   - Load creator order (chef's self-order)
   - Calculate creatorOrderValue:
     = Sum of item prices (food only, no delivery/tax)
3. Add enriched data to attribution:
   - creatorUserId
   - creatorOrderValue (in paise)
4. Return enriched attribution
```

**Commission Calculation**:
- Commission = 8% of `creatorOrderValue`
- Only applies to orders with reel attribution
- Calculated on delivery, not on payment

---

## 📱 **Frontend Integration**

### React Query Hooks

**Location**: `libs/api-client/src/lib/order/order.hooks.ts`

#### `useOrderHistory(status?, limit?)`

**Purpose**: Fetch order history with infinite scroll.

```typescript
import { useInfiniteQuery } from '@tanstack/react-query';
import { orderClient } from './order.client';

export const useOrderHistory = (status?: string, limit: number = 20) => {
  return useInfiniteQuery({
    queryKey: ['orders', 'history', status],
    queryFn: ({ pageParam }) => 
      orderClient.getOrderHistory({ 
        cursor: pageParam, 
        limit, 
        status 
      }),
    getNextPageParam: (lastPage) => lastPage.data.cursor,
    staleTime: 30000, // 30 seconds
  });
};
```

**Usage**:
```typescript
const { data, fetchNextPage, hasNextPage, isLoading } = useOrderHistory('delivered');

// Render
{data?.pages.map((page) => 
  page.data.items.map((order) => (
    <OrderCard key={order.id} order={order} />
  ))
)}

// Infinite scroll
<Button onPress={fetchNextPage} disabled={!hasNextPage}>
  Load More
</Button>
```

---

#### `useOrder(orderId: string)`

**Purpose**: Fetch single order details.

```typescript
export const useOrder = (orderId: string) => {
  return useQuery({
    queryKey: ['orders', orderId],
    queryFn: () => orderClient.getOrder(orderId),
    enabled: !!orderId,
    staleTime: 60000, // 1 minute
  });
};
```

**Usage**:
```typescript
const { data: order, isLoading, error } = useOrder(orderId);

if (isLoading) return <Spinner />;
if (error) return <ErrorView message={error.message} />;

return <OrderDetailsView order={order.data} />;
```

---

#### `useOrderLiveStatus(orderId: string)`

**Purpose**: Real-time order tracking with auto-refresh.

```typescript
export const useOrderLiveStatus = (orderId: string) => {
  return useQuery({
    queryKey: ['orders', orderId, 'live'],
    queryFn: () => orderClient.getOrderLiveStatus(orderId),
    enabled: !!orderId,
    refetchInterval: 30000, // Poll every 30 seconds
    staleTime: 0, // Always fresh
  });
};
```

**Usage**:
```typescript
const { data: liveStatus } = useOrderLiveStatus(orderId);

return (
  <View>
    <StatusBadge status={liveStatus.status} />
    <EtaCountdown etaMinutes={liveStatus.deliveryEtaMinutes} />
    <RiderInfo rider={liveStatus.rider} />
    <StatusTimeline history={liveStatus.statusHistory} />
  </View>
);
```

---

#### `useCreatePaymentIntent()`

**Purpose**: Initialize UPI Intent or COD payment.

```typescript
export const useCreatePaymentIntent = () => {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: (dto: CreatePaymentIntentDto) => 
      orderClient.createPaymentIntent(dto),
    onSuccess: (data, variables) => {
      // Invalidate order cache
      queryClient.invalidateQueries(['orders', variables.orderId]);
      
      // If UPI Intent, open UPI app
      if (data.data.paymentMethod === 'UPI_INTENT' && data.data.upiIntentUrl) {
        Linking.openURL(data.data.upiIntentUrl);
      }
    },
  });
};
```

**Usage**:
```typescript
const createPaymentIntent = useCreatePaymentIntent();

const handlePayment = async (paymentMethod: 'UPI_INTENT' | 'COD') => {
  try {
    const result = await createPaymentIntent.mutateAsync({
      orderId,
      paymentMethod,
    });
    
    if (paymentMethod === 'COD') {
      // Navigate to order confirmation
      router.push(`/orders/${orderId}`);
    } else {
      // UPI app will open automatically
      // Start polling payment status
      startPolling();
    }
  } catch (error) {
    showToast('Payment failed. Please try again.');
  }
};
```

---

#### `useReorderToCart()`

**Purpose**: Reorder previous order items to cart.

```typescript
export const useReorderToCart = () => {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: (orderId: string) => 
      orderClient.reorderToCart(orderId),
    onSuccess: (data) => {
      // Invalidate cart cache
      queryClient.invalidateQueries(['cart']);
      
      // Show toast
      if (data.data.itemsUnavailable > 0) {
        showToast(`${data.data.itemsAdded} items added. ${data.data.itemsUnavailable} unavailable.`);
      } else {
        showToast(`${data.data.itemsAdded} items added to cart!`);
      }
      
      // Navigate to cart
      router.push('/cart');
    },
  });
};
```

**Usage**:
```typescript
const reorderToCart = useReorderToCart();

<Button onPress={() => reorderToCart.mutate(order.id)}>
  Reorder
</Button>
```

---

### Frontend Screens

#### `/app/orders.tsx` - Order History

**Purpose**: List all user orders with filtering and infinite scroll.

**Key Features**:
- Filter tabs: All, Active, Delivered, Cancelled
- Pull-to-refresh
- Infinite scroll pagination
- Order cards with thumbnail, status, price
- Quick actions: Track Order, Reorder

**Implementation**:
```typescript
import { useOrderHistory } from '@chefooz-app/api-client';

export default function OrdersScreen() {
  const [selectedStatus, setSelectedStatus] = useState('all');
  const { 
    data, 
    fetchNextPage, 
    hasNextPage, 
    isLoading,
    refetch,
  } = useOrderHistory(selectedStatus);

  return (
    <View>
      <FilterTabs
        options={['all', 'delivered', 'cancelled']}
        selected={selectedStatus}
        onSelect={setSelectedStatus}
      />
      
      <FlatList
        data={data?.pages.flatMap(page => page.data.items)}
        renderItem={({ item }) => (
          <OrderCard order={item} />
        )}
        onEndReached={fetchNextPage}
        onEndReachedThreshold={0.5}
        refreshing={isLoading}
        onRefresh={refetch}
      />
    </View>
  );
}
```

---

#### `/app/orders/[orderId].tsx` - Order Details

**Purpose**: View complete order details and live status.

**Key Features**:
- Order status badge
- Items list with quantities and prices
- Pricing breakdown
- Delivery address
- Chef information
- Track Order button (navigates to live tracking)
- Cancel Order button (conditional)
- Help button (contact support)

**Implementation**:
```typescript
import { useOrder } from '@chefooz-app/api-client';

export default function OrderDetailsScreen() {
  const { orderId } = useLocalSearchParams();
  const { data: order, isLoading } = useOrder(orderId);

  if (isLoading) return <LoadingSpinner />;

  return (
    <ScrollView>
      <StatusBadge status={order.data.status} />
      
      <OrderInfoCard order={order.data} />
      
      <ItemsList items={order.data.items} />
      
      <PricingBreakdown breakdown={order.data.pricingBreakdown} />
      
      <DeliveryAddress address={order.data.addressSnapshot} />
      
      {order.data.status !== 'delivered' && (
        <Button onPress={() => router.push(`/orders/${orderId}/live`)}>
          Track Order
        </Button>
      )}
    </ScrollView>
  );
}
```

---

#### `/app/cart/payment.tsx` - Payment Screen

**Purpose**: Choose payment method and complete payment.

**Key Features**:
- Order summary (items, pricing)
- Payment method selection (UPI Intent, COD)
- UPI Intent flow (opens GPay/PhonePe)
- COD confirmation
- Payment status polling

**Implementation**:
```typescript
import { useCreatePaymentIntent } from '@chefooz-app/api-client';

export default function PaymentScreen() {
  const { orderId } = useLocalSearchParams();
  const createPaymentIntent = useCreatePaymentIntent();
  const [selectedMethod, setSelectedMethod] = useState<'UPI_INTENT' | 'COD'>('UPI_INTENT');

  const handlePayment = async () => {
    try {
      await createPaymentIntent.mutateAsync({
        orderId,
        paymentMethod: selectedMethod,
      });
      
      // If UPI: App opens automatically, start polling
      // If COD: Navigate to order confirmation
    } catch (error) {
      showToast('Payment failed. Please try again.');
    }
  };

  return (
    <View>
      <OrderSummary orderId={orderId} />
      
      <PaymentMethodSelector
        options={['UPI_INTENT', 'COD']}
        selected={selectedMethod}
        onSelect={setSelectedMethod}
      />
      
      <Button 
        onPress={handlePayment}
        loading={createPaymentIntent.isLoading}
      >
        {selectedMethod === 'COD' ? 'Confirm Order' : 'Pay Now'}
      </Button>
    </View>
  );
}
```

---

## 📚 **Shared Libraries**

### Types (`libs/types/src/lib/order.types.ts`)

**Key Interfaces**:

```typescript
export interface OrderItemSnapshot {
  id: string;
  menuItemId: string;
  quantity: number;
  unitPricePaise: number;
  titleSnapshot: string;
  imageSnapshot?: string | null;
  optionsSnapshot?: Record<string, any> | null;
  removedIngredients: string[];
  addedIngredients: Ingredient[];
  customerCookingPreferences?: string | null;
  ingredientsList?: string;
  allergenInfo?: string[];
  foodType?: 'veg' | 'non-veg' | 'egg';
  prepTimeMinutes?: number;
  description?: string;
}

export type OrderStatus =
  | 'created'
  | 'payment_pending'
  | 'paid'
  | 'delivered'
  | 'cancelled'
  | 'payment_failed'
  | 'refunded';

export type PaymentMethod = 'UPI_INTENT' | 'COD';
export type PaymentStatus = 'not_initialized' | 'pending' | 'paid' | 'failed' | 'refunded';

export interface OrderAttribution {
  linkedReelId?: string;
  linkedCreatorOrderId?: string;
  creatorOrderValue?: number; // in paise
  exploreAttribution?: {
    sourceItemId: string;
    layoutVariant: 'FOOD_GRID' | 'SOCIAL_FEED' | 'FOOD_FEED';
    timeToOrderMs: number;
    impressionTimestamp: string;
    interactionTimestamp: string;
  };
}

export interface OrderDraft {
  id: string;
  userId: string;
  chefId: string;
  items: OrderItemSnapshot[];
  instructions?: string;
  status: OrderStatus;
  subtotalPaise: number;
  deliveryFeePaise?: number;
  taxFeePaise?: number;
  totalPaise: number;
  attribution?: OrderAttribution;
  createdAt: string;
  updatedAt: string;
}

export interface OrderHistoryItem {
  id: string;
  createdAt: string;
  status: OrderStatus;
  deliveryStatus?: DeliveryStatus;
  totalPaise: number;
  chefId: string;
  chefName: string;
  itemCount: number;
  thumbnailUrl: string;
  attribution?: OrderAttribution;
}

export interface PaymentIntent {
  orderId: string;
  paymentMethod: PaymentMethod;
  upiIntentUrl?: string;
  expiresAt?: string;
  isStubbed?: boolean;
}
```

---

### Domain Rules (`libs/domain/src/lib/order/`)

**File**: `order.rules.ts`

**Key Functions**:

```typescript
export function canCreateOrder(
  chefContext: ChefContext,
  config: OrderConfig
): boolean {
  // Check if chef kitchen is accepting orders
  if (!chefContext.isOnline || !chefContext.acceptingOrders) {
    return false;
  }
  
  // If schedule exists, check service hours
  if (chefContext.schedule) {
    const now = new Date();
    const isWithinServiceHours = checkServiceHours(chefContext.schedule, now);
    if (!isWithinServiceHours) {
      return false;
    }
  }
  
  return true;
}

export function validateOrderCreation(
  context: OrderCreationContext,
  config: OrderConfig
): { allowed: boolean; reason?: string; violations?: OrderAbuseViolation[] } {
  const violations: OrderAbuseViolation[] = [];
  
  // Check trust state
  if (context.trustState === 'BANNED') {
    violations.push({
      type: 'ACCOUNT_BANNED',
      severity: 'CRITICAL',
      message: 'Account is banned from placing orders',
    });
  }
  
  // Check active orders limit
  const maxActiveOrders = getMaxActiveOrders(context.trustState, config);
  if (context.activeOrderCount >= maxActiveOrders) {
    violations.push({
      type: 'TOO_MANY_ACTIVE_ORDERS',
      severity: 'HIGH',
      message: `Maximum ${maxActiveOrders} active orders allowed`,
    });
  }
  
  // Check recent cancellations
  const maxCancellations = getMaxCancellations(context.trustState, config);
  if (context.cancellationCount >= maxCancellations) {
    violations.push({
      type: 'EXCESSIVE_CANCELLATIONS',
      severity: 'HIGH',
      message: `Maximum ${maxCancellations} cancellations per week`,
    });
  }
  
  // Check failed payments
  if (context.failedPaymentCount >= 3) {
    violations.push({
      type: 'MULTIPLE_FAILED_PAYMENTS',
      severity: 'MEDIUM',
      message: 'Too many failed payment attempts',
    });
  }
  
  return {
    allowed: violations.length === 0,
    reason: violations[0]?.message,
    violations,
  };
}

export function canCancelOrder(
  context: CancellationContext,
  config: OrderConfig
): { allowed: boolean; reason?: string } {
  // System actions always allowed
  if (context.isSystemAction) {
    return { allowed: true };
  }
  
  // Check trust state
  if (context.trustState === 'RESTRICTED') {
    return {
      allowed: false,
      reason: 'Account restricted. Please contact support.',
    };
  }
  
  // Cannot cancel after pickup
  if (context.hasPickedUp) {
    return {
      allowed: false,
      reason: 'Cannot cancel after order has been picked up.',
    };
  }
  
  // Check order status
  const allowedStatuses = ['created', 'payment_pending', 'paid', 'accepted', 'preparing'];
  if (!allowedStatuses.includes(context.orderStatus)) {
    return {
      allowed: false,
      reason: 'Order cannot be cancelled in current status.',
    };
  }
  
  return { allowed: true };
}
```

---

## ⚠️ **Error Handling**

### Standard Error Response Format

```json
{
  "success": false,
  "message": "Human-readable error message",
  "errorCode": "ERROR_CODE_CONSTANT",
  "details": {
    "field": "Additional context"
  }
}
```

### Error Codes

| Code | HTTP Status | Description | User Action |
|------|-------------|-------------|-------------|
| `ORDER_NOT_FOUND` | 404 | Order doesn't exist | Check order ID |
| `ORDER_MENU_ITEM_NOT_FOUND` | 404 | Menu item unavailable | Choose another item |
| `ORDER_NOT_READY` | 403 | Order not in valid status | Refresh and retry |
| `CHEF_NOT_ACCEPTING_ORDERS` | 403 | Chef unavailable | Try later or different chef |
| `DELIVERY_DISTANCE_EXCEEDED` | 400 | Address too far (>15 km) | Choose closer address |
| `PAYMENT_SIGNATURE_INVALID` | 400 | Razorpay signature mismatch | Contact support (security) |
| `ORDER_ALREADY_PAID` | 409 | Duplicate payment attempt | Check order status |
| `COD_LIMIT_EXCEEDED` | 403 | Too many COD orders | Use UPI or wait 24 hours |
| `ACCOUNT_RESTRICTED` | 403 | Trust state prevents action | Contact support |
| `RATE_LIMIT_EXCEEDED` | 429 | Too many requests | Wait and retry |

### Error Handling Best Practices

**Backend**:
```typescript
try {
  const order = await this.orderService.createDraftOrder(dto, userId);
  return { success: true, data: order };
} catch (error) {
  if (error instanceof NotFoundException) {
    throw error; // Propagate NestJS exceptions
  }
  
  this.logger.error(`Order creation failed: ${error.message}`, error.stack);
  
  throw new BadRequestException({
    success: false,
    message: 'Failed to create order',
    errorCode: 'ORDER_CREATION_FAILED',
  });
}
```

**Frontend**:
```typescript
const createPaymentIntent = useCreatePaymentIntent();

const handlePayment = async () => {
  try {
    await createPaymentIntent.mutateAsync({ orderId, paymentMethod });
  } catch (error) {
    const errorCode = error.response?.data?.errorCode;
    
    switch (errorCode) {
      case 'COD_LIMIT_EXCEEDED':
        showToast('Daily COD limit reached. Please use UPI.');
        break;
      case 'ACCOUNT_RESTRICTED':
        showAlert('Account Restricted', 'Please contact support.');
        break;
      default:
        showToast('Payment failed. Please try again.');
    }
  }
};
```

---

## 🔒 **Security**

### Authentication & Authorization

**JWT Validation**:
```typescript
@UseGuards(JwtAuthGuard)
@ApiBearerAuth()
export class OrderController {
  // All endpoints require valid JWT
}
```

**Access Control**:
```typescript
async getOrder(orderId: string, userId: string): Promise<Order> {
  const order = await this.orderRepository.findOne({
    where: { id: orderId },
  });
  
  // Verify user owns order, is chef, or is rider
  if (order.userId !== userId && 
      order.chefId !== userId && 
      order.deliveryPartnerId !== userId) {
    throw new ForbiddenException('Unauthorized access to order');
  }
  
  return order;
}
```

---

### Payment Security

**Razorpay Signature Verification**:
```typescript
const generatedSignature = crypto
  .createHmac('sha256', config.razorpay.keySecret)
  .update(`${razorpayOrderId}|${razorpayPaymentId}`)
  .digest('hex');

if (generatedSignature !== razorpaySignature) {
  this.logger.error('⚠️ Invalid Razorpay signature', {
    orderId,
    razorpayOrderId,
    razorpayPaymentId,
  });
  
  throw new BadRequestException({
    success: false,
    message: 'Invalid payment signature',
    errorCode: 'PAYMENT_SIGNATURE_INVALID',
  });
}
```

**Distributed Lock for Payment Confirmation**:
```typescript
@WithLock('payment:confirm:{orderId}', 10000)
async confirmRazorpayPayment(orderId, dto, userId) {
  // Lock prevents concurrent payment confirmations
  // Ensures idempotency (no double charge)
}
```

---

### Data Protection

**PII Handling**:
- ✅ Address snapshots stored (no live references)
- ✅ Payment credentials never logged
- ✅ Phone numbers hashed in logs
- ✅ JWT tokens expire after 7 days

**Database Security**:
- ✅ Row-level security (planned)
- ✅ Encrypted at rest (AWS RDS encryption)
- ✅ SSL/TLS for connections
- ✅ No plain-text passwords

---

### Rate Limiting

**Implementation**:
```typescript
@UseGuards(RateLimitGuard)
@RateLimit(20, 60) // 20 requests per 60 seconds
async createOrder(@Body() dto: CreateOrderDto) {
  // Rate limit enforced via Redis
}
```

**Limits**:
- Order creation: 20/min
- Checkout: 10/min
- Payment confirmation: 10/min
- Payment intent: 10/min

---

### Audit Logging

**Order Events**:
```typescript
await this.orderEventRepository.save({
  id: uuidv4(),
  orderId: order.id,
  eventType: 'payment.confirmed',
  eventData: {
    paymentMethod: order.paymentMethod,
    razorpayPaymentId: order.razorpayPaymentId,
    totalPaise: order.totalPaise,
    triggeredBy: userId,
  },
  createdAt: new Date(),
});
```

**Abuse Policy Logging**:
```typescript
await this.policyLogger.log({
  userId,
  orderId,
  violationType: 'EXCESSIVE_CANCELLATIONS',
  severity: 'HIGH',
  action: 'ORDER_BLOCKED',
  metadata: {
    cancellationCount: 4,
    trustState: 'WARNING',
  },
});
```

---

## ⚡ **Performance**

### Database Optimization

**Indexes**:
```sql
-- Composite index for order history queries
CREATE INDEX idx_order_user_created ON orders("userId", "createdAt");

-- Chef order lookup
CREATE INDEX idx_order_chef ON orders("chefId");

-- Status filtering
CREATE INDEX idx_order_status ON orders(status);

-- Razorpay order lookup (webhook)
CREATE INDEX idx_razorpay_order_id ON orders("razorpayOrderId");
```

**Query Performance**:
- Order history: <100ms (cursor pagination)
- Order details: <50ms (indexed lookup)
- Order creation: <200ms (includes pricing calculation)

---

### Caching Strategy

**Cache Keys**:
```
cart:{userId}                    → TTL: 1 hour
cart:count:{userId}              → TTL: 1 hour
order:lock:{orderId}             → TTL: 15 minutes
payment:confirm:{orderId}        → TTL: 10 seconds
```

**Cache Invalidation**:
```typescript
// On order creation
await this.cacheService.del(`cart:${userId}`);
await this.cacheService.del(`cart:count:${userId}`);

// On payment confirmation
await this.cacheService.del(`order:lock:${orderId}`);
```

---

### Distributed Locks

**Implementation**:
```typescript
@WithLock('order:checkout:{orderId}', 10000)
async checkout(orderId: string, addressId: string, userId: string) {
  // Lock duration: 10 seconds
  // Prevents race conditions on checkout
}
```

**Lock Keys**:
- `order:checkout:{orderId}` → 10 seconds
- `payment:confirm:{orderId}` → 10 seconds
- `order:create:{userId}` → 5 seconds (disabled due to conflicts)

---

### Pagination

**Cursor-Based Pagination**:
```typescript
const query = this.orderRepository
  .createQueryBuilder('order')
  .where('order.userId = :userId', { userId })
  .andWhere('order.createdAt < :cursor', { cursor })
  .orderBy('order.createdAt', 'DESC')
  .limit(limit + 1); // Fetch one extra to check if more results

const results = await query.getMany();
const hasMore = results.length > limit;
const nextCursor = hasMore ? results[limit - 1].createdAt : null;
```

**Benefits**:
- ✅ Scales to millions of orders (no OFFSET)
- ✅ Consistent performance (O(log n) lookup)
- ✅ No skipped or duplicate results

---

## 🧪 **Testing**

### Unit Tests

**File**: `apps/chefooz-apis/src/modules/order/order.service.spec.ts`

**Test Coverage**:
```typescript
describe('OrderService', () => {
  describe('createDraftOrder', () => {
    it('should create draft order with valid data', async () => {
      // Arrange
      const dto = { /* valid order data */ };
      const userId = 'user-uuid';
      
      // Act
      const order = await service.createDraftOrder(dto, userId);
      
      // Assert
      expect(order.status).toBe(OrderStatus.CREATED);
      expect(order.userId).toBe(userId);
      expect(order.items).toHaveLength(2);
    });
    
    it('should reject order if chef unavailable', async () => {
      // Arrange
      jest.spyOn(chefKitchenService, 'isAcceptingOrders')
        .mockResolvedValue(false);
      
      // Act & Assert
      await expect(
        service.createDraftOrder(dto, userId)
      ).rejects.toThrow('Chef not accepting orders');
    });
    
    it('should enforce abuse policy for restricted users', async () => {
      // Arrange
      mockUserWithTrustState('RESTRICTED');
      
      // Act & Assert
      await expect(
        service.createDraftOrder(dto, userId)
      ).rejects.toThrow('Account restricted');
    });
  });
  
  describe('confirmRazorpayPayment', () => {
    it('should verify signature and mark order paid', async () => {
      // Arrange
      const dto = {
        razorpayOrderId: 'order_abc',
        razorpayPaymentId: 'pay_xyz',
        razorpaySignature: 'valid_signature',
      };
      
      jest.spyOn(razorpayCheckoutService, 'verifyPaymentSignature')
        .mockReturnValue(true);
      
      // Act
      await service.confirmRazorpayPayment(orderId, dto, userId);
      
      // Assert
      const order = await orderRepository.findOne({ where: { id: orderId } });
      expect(order.paymentStatus).toBe('paid');
      expect(order.status).toBe('paid');
    });
    
    it('should reject invalid signature', async () => {
      // Arrange
      jest.spyOn(razorpayCheckoutService, 'verifyPaymentSignature')
        .mockReturnValue(false);
      
      // Act & Assert
      await expect(
        service.confirmRazorpayPayment(orderId, dto, userId)
      ).rejects.toThrow('Invalid payment signature');
    });
  });
});
```

---

### Integration Tests

**File**: `apps/chefooz-apis-e2e/src/order/order.e2e-spec.ts`

**Test Scenarios**:
```typescript
describe('Order E2E', () => {
  let app: INestApplication;
  let authToken: string;
  
  beforeAll(async () => {
    app = await createTestApp();
    authToken = await getTestAuthToken();
  });
  
  it('should create order from cart and complete payment', async () => {
    // 1. Add items to cart
    await request(app.getHttpServer())
      .post('/api/v1/cart')
      .set('Authorization', `Bearer ${authToken}`)
      .send({
        menuItemId: 'menu-item-uuid',
        quantity: 2,
      })
      .expect(201);
    
    // 2. Checkout cart
    const checkoutRes = await request(app.getHttpServer())
      .post('/api/v1/cart/checkout')
      .set('Authorization', `Bearer ${authToken}`)
      .send({
        addressId: 'address-uuid',
      })
      .expect(200);
    
    const orderId = checkoutRes.body.data.orderId;
    
    // 3. Create payment intent
    const paymentRes = await request(app.getHttpServer())
      .post('/api/v1/orders/payment-intent')
      .set('Authorization', `Bearer ${authToken}`)
      .send({
        orderId,
        paymentMethod: 'UPI_INTENT',
      })
      .expect(200);
    
    expect(paymentRes.body.data.upiIntentUrl).toBeDefined();
    
    // 4. Simulate Razorpay webhook (payment captured)
    await request(app.getHttpServer())
      .post('/api/v1/orders/razorpay/webhook')
      .set('X-Razorpay-Signature', generateValidSignature())
      .send({
        event: 'payment.captured',
        payload: {
          payment: {
            entity: {
              order_id: paymentRes.body.data.razorpayOrderId,
              id: 'pay_test_123',
              amount: 51800,
              status: 'captured',
            },
          },
        },
      })
      .expect(200);
    
    // 5. Verify order status
    const orderRes = await request(app.getHttpServer())
      .get(`/api/v1/orders/${orderId}`)
      .set('Authorization', `Bearer ${authToken}`)
      .expect(200);
    
    expect(orderRes.body.data.status).toBe('paid');
    expect(orderRes.body.data.paymentStatus).toBe('paid');
  });
});
```

---

### Manual Testing Checklist

**Pre-Deployment**:
- [ ] Create order from cart
- [ ] Create order from reel CTA
- [ ] Checkout with address
- [ ] UPI Intent flow (open GPay/PhonePe)
- [ ] COD flow (auto-confirm)
- [ ] Payment confirmation (verify signature)
- [ ] Order history pagination
- [ ] Reorder to cart
- [ ] Live order tracking
- [ ] Auto-cancellation (timeout scenarios)
- [ ] Abuse detection (test trust states)
- [ ] Commission calculation on delivery
- [ ] Coin crediting on delivery

---

## 🚀 **Deployment**

### Environment Variables

**Required**:
```bash
# Database
DATABASE_URL=postgresql://user:password@host:5432/chefooz
MONGODB_URI=mongodb://host:27017/chefooz

# Razorpay
RAZORPAY_KEY_ID=rzp_live_xxxxx
RAZORPAY_KEY_SECRET=xxxxx
RAZORPAY_WEBHOOK_SECRET=xxxxx

# Cache
REDIS_URL=redis://valkey-host:6379

# Order Configuration
ORDER_PAYMENT_LOCK_MINUTES=15
ORDER_MAX_ITEMS=20
ORDER_MIN_VALUE_PAISE=5000
ORDER_MAX_DISTANCE_KM=15
ORDER_COD_DAILY_LIMIT=3
ORDER_RATE_LIMIT_PER_MIN=20
```

**Optional**:
```bash
# Feature Flags
ENABLE_COD_PAYMENTS=true
ENABLE_ABUSE_DETECTION=true
ENABLE_COIN_REWARDS=true

# Commission
COMMISSION_RATE=0.08  # 8%
```

---

### Database Migrations

**Run Migrations**:
```bash
npm run typeorm:migration:run
```

**Key Migrations**:
1. `1700000001-CreateOrdersTable.ts`
2. `1700000002-AddOrderStatusHistory.ts`
3. `1700000003-AddOrderEvents.ts`
4. `1700000004-AddRazorpayFields.ts`
5. `1700000005-AddOrderAttribution.ts`

---

### Health Checks

**Endpoint**: `GET /api/health`

**Checks**:
- Database connectivity (PostgreSQL)
- Cache connectivity (Valkey)
- Razorpay API reachability

---

### Monitoring

**Key Metrics**:
- Order creation rate (orders/min)
- Payment success rate (%)
- Order cancellation rate (%)
- Average order value (₹)
- API response times (p50, p95, p99)

**Alerts**:
- Payment success rate < 95% → Critical
- Order creation errors > 5% → High
- Database query time > 500ms → Medium
- Razorpay webhook failures → High

---

## 📚 **Related Documentation**

- **Feature Overview**: [FEATURE_OVERVIEW.md](./FEATURE_OVERVIEW.md)
- **QA Test Cases**: [QA_TEST_CASES.md](./QA_TEST_CASES.md)
- **Cart Module**: [../cart/TECHNICAL_GUIDE.md](../cart/TECHNICAL_GUIDE.md)
- **Payment Integration**: [../../integrations/PAYMENT_FLOW_COMPLETE.md](../../integrations/PAYMENT_FLOW_COMPLETE.md)
- **Commission Module**: [../commission/TECHNICAL_GUIDE.md](../commission/TECHNICAL_GUIDE.md)
- **Delivery Module**: [../delivery/TECHNICAL_GUIDE.md](../delivery/TECHNICAL_GUIDE.md)

---

## ✅ **Completion Status**

**Implementation**: 100%  
**Documentation**: Complete  
**Production Status**: ✅ Live  
**Last Audit**: 2026-02-15

---

**[MODULE_COMPLETE ✅]**
