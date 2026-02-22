# 👨‍🍳 Chef Orders Module - Technical Guide

## 📋 **Table of Contents**
- [Architecture Overview](#architecture-overview)
- [Module Structure](#module-structure)
- [API Endpoints](#api-endpoints)
- [Service Layer](#service-layer)
- [Database Schema](#database-schema)
- [DTOs](#dtos)
- [Shared Types](#shared-types)
- [Integration Patterns](#integration-patterns)
- [Error Handling](#error-handling)
- [Testing Strategy](#testing-strategy)

---

## 🏗️ **Architecture Overview**

### **System Components**

```
┌──────────────────────────────────────────────────────────────┐
│                  CHEF ORDERS ECOSYSTEM                        │
└──────────────────────────────────────────────────────────────┘
                               │
         ┌─────────────────────┼─────────────────────┐
         │                     │                     │
         ▼                     ▼                     ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│  ChefOrders     │  │  ChefOrders     │  │     DTOs        │
│  Controller     │  │   Service       │  │                 │
│                 │  │                 │  │ - Accept        │
│ GET /incoming   │  │ - Business      │  │ - Reject        │
│ POST /accept    │  │   Logic         │  │ - UpdateStatus  │
│ POST /reject    │  │ - Validation    │  │ - UpdateTime    │
│ POST /status    │  │ - Transform     │  │                 │
│ POST /prep-time │  │ - Notify        │  │                 │
└─────────────────┘  └─────────────────┘  └─────────────────┘
         │                     │                     │
         └─────────────────────┼─────────────────────┘
                               │
         ┌─────────────────────┼─────────────────────┐
         │                     │                     │
         ▼                     ▼                     ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│  Order Entity   │  │  ChefKitchen    │  │  ChefMenuItem   │
│  (TypeORM)      │  │   Entity        │  │    Entity       │
│                 │  │                 │  │                 │
│ - status        │  │ - isOnline      │  │ - name          │
│ - chefStatus    │  │ - accepting     │  │ - pricePaise    │
│ - timeline      │  │   Orders        │  │ - isActive      │
│ - items         │  │                 │  │                 │
└─────────────────┘  └─────────────────┘  └─────────────────┘
         │                     │                     │
         └─────────────────────┼─────────────────────┘
                               │
         ┌─────────────────────┼─────────────────────┐
         │                     │                     │
         ▼                     ▼                     ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│  Notification   │  │     Order       │  │     Domain      │
│  Dispatcher     │  │    Service      │  │     Rules       │
│                 │  │                 │  │   (@domain)     │
│ - Push Notif    │  │ - Refund Logic  │  │                 │
│ - Email Send    │  │ - Order CRUD    │  │ - canAccept     │
│                 │  │                 │  │ - canReject     │
└─────────────────┘  └─────────────────┘  └─────────────────┘
```

### **Request Flow**

```mermaid
sequenceDiagram
    participant C as Chef (Frontend)
    participant Ctrl as ChefOrdersController
    participant Svc as ChefOrdersService
    participant DB as PostgreSQL
    participant N as NotificationDispatcher
    participant O as OrderService

    C->>Ctrl: POST /v1/chef/orders/:id/accept
    Ctrl->>Svc: acceptOrder(orderId, chefId, dto)
    
    Svc->>DB: Fetch ChefKitchen (validate online & accepting)
    DB-->>Svc: ChefKitchen entity
    
    Svc->>DB: Fetch Order (validate exists & belongs to chef)
    DB-->>Svc: Order entity
    
    Svc->>Svc: Validate domain rules (canChefAcceptOrder)
    Svc->>Svc: Validate prep time (5-180 minutes)
    
    Svc->>DB: Update Order (status = ACCEPTED, acceptedAt, prep time)
    DB-->>Svc: Updated Order
    
    Svc->>N: Send notification to customer
    N-->>Svc: Notification sent
    
    Svc-->>Ctrl: Return updated Order
    Ctrl-->>C: 200 OK with order data
```

---

## 📁 **Module Structure**

```
apps/chefooz-apis/src/modules/chef-orders/
├── chef-orders.controller.ts    # HTTP endpoints (5 routes)
├── chef-orders.service.ts       # Business logic (591 lines)
├── chef-orders.module.ts        # Module definition
└── dto/
    ├── accept-order.dto.ts      # Accept order input validation
    ├── reject-order.dto.ts      # Reject order input validation
    ├── update-prep-status.dto.ts # Status update input validation
    └── update-prep-time.dto.ts  # Prep time update input validation
```

### **Module Definition**

```typescript
// chef-orders.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { Order } from '../../database/entities/order.entity';
import { ChefMenuItem } from '../chef-kitchen/entities/chef-menu-item.entity';
import { ChefKitchen } from '../chef-kitchen/entities/chef-kitchen.entity';
import { ChefOrdersController } from './chef-orders.controller';
import { ChefOrdersService } from './chef-orders.service';
import { NotificationModule } from '../notification/notification.module';
import { OrderModule } from '../order/order.module';

/**
 * Chef Orders Module
 * Kitchen management for chefs
 * 
 * Provides:
 * - Incoming order viewing
 * - Order acceptance/rejection
 * - Preparation status updates
 * 
 * Dependencies:
 * - OrderModule: For refund logic on rejections
 * - NotificationModule: For customer notifications
 */
@Module({
  imports: [
    TypeOrmModule.forFeature([Order, ChefMenuItem, ChefKitchen]),
    NotificationModule,
    OrderModule,
  ],
  controllers: [ChefOrdersController],
  providers: [ChefOrdersService],
  exports: [ChefOrdersService],
})
export class ChefOrdersModule {}
```

**Dependencies**:
- **TypeORM Entities**: Order, ChefMenuItem, ChefKitchen
- **NotificationModule**: Push notifications + email dispatch
- **OrderModule**: Refund processing on rejection

---

## 🔌 **API Endpoints**

**Base URL**: `/v1/chef/orders`  
**Authentication**: JWT Bearer token (chef user)  
**Content-Type**: `application/json`

---

### **Endpoint 1: Get Incoming Orders**

**URL**: `GET /v1/chef/orders/incoming`

**Description**: Fetch all orders awaiting chef action or currently being prepared

**Authentication**: Required (JWT)

**Rate Limit**: None (real-time dashboard refresh)

**Request Headers**:
```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Query Parameters**: None

**Response** (200 OK):
```json
{
  "success": true,
  "message": "Found 5 incoming orders",
  "data": [
    {
      "id": "order-uuid-123",
      "orderNumber": "ORD-ABCD1234",
      "userId": "user-uuid-456",
      "customerName": "Rajesh Kumar",
      "customerPhone": "+919876543210",
      "items": [
        {
          "menuItemId": "menu-item-uuid-789",
          "menuItemName": "Butter Chicken",
          "quantity": 2,
          "price": 250.00,
          "specialInstructions": "Extra spicy"
        }
      ],
      "totalAmount": 550.00,
      "deliveryAddress": {
        "line1": "123 MG Road",
        "line2": "Near City Mall",
        "city": "Mumbai",
        "state": "Maharashtra",
        "postalCode": "400001",
        "country": "India"
      },
      "chefStatus": "NEW",
      "estimatedPrepMinutes": null,
      "chefNote": null,
      "specialInstructions": "Extra spicy",
      "createdAt": "2026-02-22T10:30:00.000Z",
      "acceptedAt": null,
      "rejectedAt": null,
      "readyAt": null,
      "statusGroup": "NEW",
      "prepRemainingMinutes": null,
      "isOverdue": false,
      "reelSource": {
        "reelId": "reel-uuid-999",
        "creatorName": "Reel Creator"
      }
    }
  ]
}
```

**Response Fields**:
- `orderNumber`: Display-friendly order ID (ORD-XXXXXXXX)
- `items[].price`: In rupees (converted from paise)
- `totalAmount`: In rupees (converted from paise)
- `statusGroup`: UI grouping (NEW | ACTIVE | READY | COMPLETED)
- `prepRemainingMinutes`: Countdown timer (negative if overdue)
- `isOverdue`: Boolean flag for red badge display
- `reelSource`: Optional (only if order from creator reel)

**Error Response** (500):
```json
{
  "success": false,
  "message": "Failed to fetch incoming orders",
  "errorCode": "INCOMING_ORDERS_FETCH_FAILED"
}
```

**Business Logic**:
```typescript
// Filters orders by:
// 1. Chef ownership (chefId = JWT user ID)
// 2. Payment status = PAID
// 3. Order status IN (PAID, ACCEPTED, PREPARING, READY, OUT_FOR_DELIVERY)
// 4. Excludes COMPLETED, CANCELLED, REFUNDED

// Joins:
// - User table (for customer name if not in address snapshot)
// - ChefMenuItem table (for menu item names)

// Sorting: createdAt DESC (newest first)
```

**Performance**:
- Response time: ~120ms (p95)
- Database queries: 2 (orders + menu items)
- No caching (real-time data required)

---

### **Endpoint 2: Accept Order**

**URL**: `POST /v1/chef/orders/:orderId/accept`

**Description**: Accept order and set estimated preparation time

**Authentication**: Required (JWT)

**Rate Limit**: 60 requests/minute

**URL Parameters**:
- `orderId` (string, required): Order UUID

**Request Headers**:
```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json
```

**Request Body**:
```json
{
  "estimatedPrepMinutes": 30,
  "chefNote": "Will use cottage cheese instead of paneer"
}
```

**Request Body Schema**:
```typescript
{
  estimatedPrepMinutes: number; // Required, 5-180
  chefNote?: string;            // Optional, max 500 chars
}
```

**Validation Rules**:
- `estimatedPrepMinutes`:
  - Required
  - Must be integer
  - Min: 5 (minimum prep time)
  - Max: 180 (3 hours maximum)
- `chefNote`:
  - Optional
  - Max length: 500 characters
  - Use for ingredient substitutions or delays

**Response** (200 OK):
```json
{
  "success": true,
  "message": "Order accepted. Will be ready in 30 minutes",
  "data": {
    "id": "order-uuid-123",
    "status": "ACCEPTED",
    "chefStatus": "ACCEPTED",
    "estimatedPrepMinutes": 30,
    "chefNote": "Will use cottage cheese instead of paneer",
    "acceptedAt": "2026-02-22T10:35:00.000Z"
  }
}
```

**Error Response** (400 - Kitchen Offline):
```json
{
  "success": false,
  "message": "Cannot accept orders while offline. Please set your kitchen status to 'Online' first.",
  "errorCode": "ORDER_ACCEPT_FAILED"
}
```

**Error Response** (400 - Kitchen Paused):
```json
{
  "success": false,
  "message": "Cannot accept orders at this time. Your kitchen is currently paused. Please enable 'Accepting Orders' in your kitchen settings.",
  "errorCode": "ORDER_ACCEPT_FAILED"
}
```

**Error Response** (400 - Invalid Status):
```json
{
  "success": false,
  "message": "Cannot accept order. Order must be PAID and chef status must be NEW. Current order status: ACCEPTED, chef status: ACCEPTED",
  "errorCode": "ORDER_ACCEPT_FAILED"
}
```

**Error Response** (403 - Unauthorized):
```json
{
  "success": false,
  "message": "You are not authorized to manage this order",
  "errorCode": "ORDER_ACCEPT_FAILED"
}
```

**Error Response** (404 - Not Found):
```json
{
  "success": false,
  "message": "Order not found",
  "errorCode": "ORDER_ACCEPT_FAILED"
}
```

**Business Logic**:
```typescript
// 1. Validate Kitchen Availability
if (!chefKitchen.isOnline) {
  throw BadRequestException('Kitchen offline');
}
if (!chefKitchen.acceptingOrders) {
  throw BadRequestException('Kitchen paused');
}

// 2. Validate Order Ownership
if (order.chefId !== chefId) {
  throw ForbiddenException('Not authorized');
}

// 3. Validate Status Transition (Domain Rule)
if (!canChefAcceptOrder(orderDomain, order.chefStatus)) {
  throw BadRequestException('Invalid status transition');
}

// 4. Validate Prep Time Range
if (prepMinutes < 5 || prepMinutes > 180) {
  throw BadRequestException('Prep time must be 5-180 minutes');
}

// 5. Update Order
order.status = OrderStatus.ACCEPTED;        // Main status
order.chefStatus = ChefOrderStatus.ACCEPTED; // Legacy
order.estimatedPrepMinutes = dto.estimatedPrepMinutes;
order.chefNote = dto.chefNote;
order.acceptedAt = new Date();

// 6. Send Customer Notification
await notificationDispatcher.send(
  order.userId,
  'order.accepted',
  { orderId, estimatedPrepMinutes }
);
```

**Side Effects**:
- Order status updated in database
- Customer receives push notification
- Order timeline updated with `acceptedAt` timestamp

---

### **Endpoint 3: Reject Order**

**URL**: `POST /v1/chef/orders/:orderId/reject`

**Description**: Reject order and trigger automatic refund

**Authentication**: Required (JWT)

**Rate Limit**: 30 requests/minute

**URL Parameters**:
- `orderId` (string, required): Order UUID

**Request Headers**:
```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json
```

**Request Body**:
```json
{
  "reason": "Out of main ingredient"
}
```

**Request Body Schema**:
```typescript
{
  reason: string; // Required, max 500 chars
}
```

**Validation Rules**:
- `reason`:
  - Required (cannot be empty)
  - Max length: 500 characters
  - Will be sent to customer in notification

**Response** (200 OK):
```json
{
  "success": true,
  "message": "Order rejected. Refund will be processed.",
  "data": {
    "id": "order-uuid-123",
    "status": "CANCELLED",
    "chefStatus": "REJECTED",
    "chefNote": "Out of main ingredient",
    "rejectedAt": "2026-02-22T10:36:00.000Z"
  }
}
```

**Error Response** (400 - Invalid Status):
```json
{
  "success": false,
  "message": "Cannot reject order in ACCEPTED status. Only NEW orders can be rejected.",
  "errorCode": "ORDER_REJECT_FAILED"
}
```

**Error Response** (403 - Unauthorized):
```json
{
  "success": false,
  "message": "You are not authorized to manage this order",
  "errorCode": "ORDER_REJECT_FAILED"
}
```

**Error Response** (404 - Not Found):
```json
{
  "success": false,
  "message": "Order not found",
  "errorCode": "ORDER_REJECT_FAILED"
}
```

**Business Logic**:
```typescript
// 1. Validate Order Ownership
if (order.chefId !== chefId) {
  throw ForbiddenException('Not authorized');
}

// 2. Validate Status (Domain Rule)
if (!canChefRejectOrder(order.chefStatus)) {
  throw BadRequestException('Only NEW orders can be rejected');
}

// 3. Abuse Detection (Pattern Tracking)
const abuseCheck = validateCancellationAbuse(
  {
    orderStatus: order.status,
    trustState: order.user.trustState,
    hasPickedUp: !!order.deliveryPartnerId,
    isSystemAction: false,
  },
  orderConfig
);
console.log('[ABUSE-CHECK]', abuseCheck); // Log for pattern analysis

// 4. Update Order
order.chefStatus = ChefOrderStatus.REJECTED;
order.status = OrderStatus.CANCELLED;
order.chefNote = dto.reason;
order.rejectedAt = new Date();

// 5. Trigger Automatic Refund
await orderService.handleOrderRefunded(orderId);

// 6. Send Notifications
await notificationDispatcher.send(
  order.userId,
  'order.cancelled',
  { orderId, reason: dto.reason }
);

// 7. Send Cancellation Emails
await emailEventDispatcher.sendOrderCancelledEmails(
  orderId,
  dto.reason,
  true // refundExpected
);
```

**Side Effects**:
- Order status updated to CANCELLED/REJECTED
- **Automatic refund initiated** via Razorpay (full amount)
- Customer receives push notification + email
- Chef receives confirmation email
- Rejection logged for abuse pattern tracking

**Refund Details**:
- Amount: Full order total (100%)
- Method: Original payment method (Razorpay)
- Timeline: 3-5 business days
- Status: Tracked in `payment_status` field

---

### **Endpoint 4: Update Prep Status**

**URL**: `POST /v1/chef/orders/:orderId/status`

**Description**: Update order preparation status (PREPARING or READY)

**Authentication**: Required (JWT)

**Rate Limit**: 120 requests/minute

**URL Parameters**:
- `orderId` (string, required): Order UUID

**Request Headers**:
```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json
```

**Request Body** (Mark as Preparing):
```json
{
  "status": "PREPARING"
}
```

**Request Body** (Mark as Ready):
```json
{
  "status": "READY"
}
```

**Request Body Schema**:
```typescript
{
  status: 'PREPARING' | 'READY'; // Required
}
```

**Validation Rules**:
- `status`:
  - Required
  - Must be enum: `PREPARING` or `READY`
  - Other statuses handled by system (not chef-facing)

**Response** (200 OK - PREPARING):
```json
{
  "success": true,
  "message": "Order status updated to PREPARING",
  "data": {
    "id": "order-uuid-123",
    "status": "PREPARING",
    "chefStatus": "PREPARING"
  }
}
```

**Response** (200 OK - READY):
```json
{
  "success": true,
  "message": "Order status updated to READY",
  "data": {
    "id": "order-uuid-123",
    "status": "READY",
    "chefStatus": "READY",
    "readyAt": "2026-02-22T11:05:00.000Z"
  }
}
```

**Error Response** (400 - Invalid Transition):
```json
{
  "success": false,
  "message": "Can only mark as PREPARING after ACCEPTED status",
  "errorCode": "ORDER_STATUS_UPDATE_FAILED"
}
```

**Error Response** (400 - Invalid Transition):
```json
{
  "success": false,
  "message": "Can only mark as READY after PREPARING status",
  "errorCode": "ORDER_STATUS_UPDATE_FAILED"
}
```

**Business Logic**:
```typescript
// 1. Validate Order Ownership
if (order.chefId !== chefId) {
  throw ForbiddenException('Not authorized');
}

// 2. Validate Status Transitions (Domain Rules)
if (newStatus === ChefOrderStatus.PREPARING) {
  if (!canChefStartPreparing(orderDomain, currentStatus)) {
    throw BadRequestException('Can only mark as PREPARING after ACCEPTED');
  }
  order.status = OrderStatus.PREPARING; // Update main status
}

if (newStatus === ChefOrderStatus.READY) {
  if (!canChefMarkReady(orderDomain, currentStatus)) {
    throw BadRequestException('Can only mark as READY after PREPARING');
  }
  order.status = OrderStatus.READY;
  order.readyAt = new Date();
  
  // Send notification when ready
  await notificationDispatcher.send(
    order.userId,
    'order.cooking', // Legacy event name
    { orderId }
  );
}

// 3. Update chef status (legacy field)
order.chefStatus = newStatus;
```

**Side Effects**:
- Order status updated (both main + chef status)
- Customer receives notification when marked READY
- Delivery partner system notified when READY
- Timeline updated with `readyAt` timestamp

**Status Transition Rules**:
```
ACCEPTED → PREPARING ✅
ACCEPTED → READY ❌ (must prepare first)

PREPARING → READY ✅
PREPARING → ACCEPTED ❌ (no backward transitions)

READY → Any ❌ (order locked for delivery)
```

---

### **Endpoint 5: Update Prep Time**

**URL**: `POST /v1/chef/orders/:orderId/prep-time`

**Description**: Adjust estimated preparation time mid-cooking

**Authentication**: Required (JWT)

**Rate Limit**: 60 requests/minute

**URL Parameters**:
- `orderId` (string, required): Order UUID

**Request Headers**:
```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json
```

**Request Body**:
```json
{
  "estimatedPrepMinutes": 45
}
```

**Request Body Schema**:
```typescript
{
  estimatedPrepMinutes: number; // Required, 5-180
}
```

**Validation Rules**:
- `estimatedPrepMinutes`:
  - Required
  - Must be integer
  - Min: 5 (minimum prep time)
  - Max: 180 (3 hours maximum)

**Response** (200 OK):
```json
{
  "success": true,
  "message": "Prep time updated to 45 minutes",
  "data": {
    "id": "order-uuid-123",
    "estimatedPrepMinutes": 45
  }
}
```

**Error Response** (400 - Invalid State):
```json
{
  "success": false,
  "message": "Can only update prep time for accepted or preparing orders",
  "errorCode": "PREP_TIME_UPDATE_FAILED"
}
```

**Error Response** (400 - Invalid Range):
```json
{
  "success": false,
  "message": "Estimated prep time must be between 5 and 180 minutes",
  "errorCode": "PREP_TIME_UPDATE_FAILED"
}
```

**Business Logic**:
```typescript
// 1. Validate Order Ownership
if (order.chefId !== chefId) {
  throw ForbiddenException('Not authorized');
}

// 2. Validate Prep Time Range
if (prepMinutes < 5 || prepMinutes > 180) {
  throw BadRequestException('Prep time must be 5-180 minutes');
}

// 3. Validate Order State
if (order.chefStatus !== ChefOrderStatus.ACCEPTED &&
    order.chefStatus !== ChefOrderStatus.PREPARING) {
  throw BadRequestException('Can only update prep time for active orders');
}

// 4. Calculate Time Difference
const oldPrepTime = order.estimatedPrepMinutes;
const timeDifference = Math.abs(prepMinutes - oldPrepTime);

// 5. Update Prep Time
order.estimatedPrepMinutes = prepMinutes;

// 6. Send Notification if Significant Change (≥10 minutes)
if (oldPrepTime && timeDifference >= 10) {
  await notificationDispatcher.send(
    order.userId,
    'order.prep_time_updated',
    {
      orderId,
      oldPrepTime,
      newPrepTime: prepMinutes
    }
  );
  console.log(`📱 Notification sent (change: ${timeDifference} min)`);
} else {
  console.log(`ℹ️ Skipping notification (change < 10 minutes)`);
}
```

**Side Effects**:
- Prep time updated in database
- Customer notified **only if change ≥10 minutes**
- Dashboard countdown timer recalculated
- Overdue calculation adjusted

**Notification Logic**:
```typescript
// Send notification if change >= 10 minutes
if (Math.abs(newPrepTime - oldPrepTime) >= 10) {
  sendNotification();
}

// Examples:
// 30 → 45 minutes = 15 min change → ✅ Notify
// 30 → 35 minutes = 5 min change  → ⏭️ Skip
// 45 → 30 minutes = 15 min change → ✅ Notify (faster!)
```

---

## 🔧 **Service Layer**

### **ChefOrdersService Methods**

#### **Method 1: getIncomingOrders**

**Signature**:
```typescript
async getIncomingOrders(chefId: string): Promise<ChefOrderResponse[]>
```

**Purpose**: Fetch all orders for chef's kitchen dashboard

**Query Logic**:
```typescript
const orders = await this.orderRepository
  .createQueryBuilder('order')
  .leftJoinAndSelect('order.user', 'user')
  .where('order.chefId = :chefId', { chefId })
  .andWhere('order.status IN (:...statuses)', { 
    statuses: [
      OrderStatus.PAID,              // Awaiting chef acceptance
      OrderStatus.ACCEPTED,          // Chef accepted
      OrderStatus.PREPARING,         // Chef cooking
      OrderStatus.READY,             // Ready for pickup
      OrderStatus.OUT_FOR_DELIVERY,  // Courier picked up
    ] 
  })
  .andWhere('order.paymentStatus = :paymentStatus', { 
    paymentStatus: PaymentStatus.PAID 
  })
  .orderBy('order.createdAt', 'DESC')
  .getMany();
```

**Transformation**:
```typescript
// For each order:
// 1. Fetch menu item names (join ChefMenuItem table)
// 2. Convert prices from paise to rupees (divide by 100)
// 3. Extract address from addressSnapshot JSONB
// 4. Extract reel source from attribution JSONB (if exists)
// 5. Calculate statusGroup (NEW | ACTIVE | READY | COMPLETED)
// 6. Calculate prepRemainingMinutes (can be negative if overdue)
// 7. Calculate isOverdue flag (remaining < 0)

return {
  id: order.id,
  orderNumber: `ORD-${order.id.substring(0, 8).toUpperCase()}`,
  userId: order.userId,
  customerName: address.fullName || user?.fullName || 'Unknown',
  customerPhone: address.phone || user?.phone || 'N/A',
  items: itemsWithNames,
  totalAmount: order.totalPaise / 100,
  deliveryAddress: {
    line1: address.line1 || '',
    line2: address.line2,
    city: address.city || '',
    state: address.state || '',
    postalCode: address.pincode || '',
    country: address.country,
  },
  chefStatus: order.chefStatus,
  estimatedPrepMinutes: order.estimatedPrepMinutes,
  createdAt: order.createdAt.toISOString(),
  acceptedAt: order.acceptedAt?.toISOString(),
  readyAt: order.readyAt?.toISOString(),
  chefNote: order.chefNote,
  // Computed fields
  statusGroup: this.getStatusGroup(order.chefStatus),
  prepRemainingMinutes: this.getPrepRemainingMinutes(
    order.acceptedAt, 
    order.estimatedPrepMinutes
  ),
  isOverdue: this.isOrderOverdue(
    order.acceptedAt,
    order.estimatedPrepMinutes,
    order.chefStatus
  ),
  reelSource: attribution?.linkedReelId ? {
    reelId: attribution.linkedReelId,
    creatorName: 'Reel Creator', // TODO: Join with user/reel data
  } : undefined,
};
```

**Helper: getStatusGroup**:
```typescript
private getStatusGroup(status: ChefOrderStatus): OrderStatusGroup {
  switch (status) {
    case ChefOrderStatus.NEW:
      return OrderStatusGroup.NEW;
    case ChefOrderStatus.ACCEPTED:
    case ChefOrderStatus.PREPARING:
      return OrderStatusGroup.ACTIVE;
    case ChefOrderStatus.READY:
    case ChefOrderStatus.OUT_FOR_DELIVERY:
      return OrderStatusGroup.READY;
    case ChefOrderStatus.COMPLETED:
    case ChefOrderStatus.REJECTED:
      return OrderStatusGroup.COMPLETED;
    default:
      return OrderStatusGroup.NEW;
  }
}
```

**Helper: getPrepRemainingMinutes**:
```typescript
private getPrepRemainingMinutes(
  acceptedAt: Date | undefined,
  estimatedPrepMinutes: number | undefined,
): number | undefined {
  if (!acceptedAt || !estimatedPrepMinutes) return undefined;
  
  const now = new Date();
  const elapsedMs = now.getTime() - acceptedAt.getTime();
  const elapsedMinutes = Math.floor(elapsedMs / 60000);
  
  // Can return negative if overdue
  return estimatedPrepMinutes - elapsedMinutes;
}
```

**Helper: isOrderOverdue**:
```typescript
private isOrderOverdue(
  acceptedAt: Date | undefined,
  estimatedPrepMinutes: number | undefined,
  chefStatus: ChefOrderStatus,
): boolean {
  if (!acceptedAt || !estimatedPrepMinutes) return false;
  
  // Not overdue if already READY or COMPLETED
  if (chefStatus === ChefOrderStatus.READY || 
      chefStatus === ChefOrderStatus.COMPLETED) {
    return false;
  }
  
  const remaining = this.getPrepRemainingMinutes(acceptedAt, estimatedPrepMinutes);
  return remaining !== undefined && remaining < 0;
}
```

---

#### **Method 2: acceptOrder**

**Signature**:
```typescript
async acceptOrder(
  orderId: string,
  chefId: string,
  dto: AcceptOrderDto,
): Promise<Order>
```

**Steps**:
1. **Fetch ChefKitchen**: Validate kitchen exists and is accepting orders
2. **Validate Kitchen State**:
   - `isOnline` must be `true`
   - `acceptingOrders` must be `true`
3. **Fetch Order**: Validate exists and belongs to chef
4. **Validate Ownership**: `order.chefId === chefId`
5. **Validate Status**: Use domain rule `canChefAcceptOrder()`
6. **Validate Prep Time**: Range 5-180 minutes
7. **Update Order**:
   - `status = ACCEPTED` (main status)
   - `chefStatus = ACCEPTED` (legacy)
   - `estimatedPrepMinutes = dto.estimatedPrepMinutes`
   - `chefNote = dto.chefNote`
   - `acceptedAt = new Date()`
8. **Send Notification**: Push to customer with prep estimate
9. **Return**: Updated order entity

---

#### **Method 3: rejectOrder**

**Signature**:
```typescript
async rejectOrder(
  orderId: string,
  chefId: string,
  dto: RejectOrderDto,
): Promise<Order>
```

**Steps**:
1. **Fetch Order**: With user relation (needed for abuse check)
2. **Validate Ownership**: `order.chefId === chefId`
3. **Validate Status**: Use domain rule `canChefRejectOrder()`
4. **Abuse Detection**: Validate cancellation policy (log for pattern tracking)
5. **Update Order**:
   - `chefStatus = REJECTED`
   - `status = CANCELLED`
   - `chefNote = dto.reason`
   - `rejectedAt = new Date()`
6. **Trigger Refund**: Call `orderService.handleOrderRefunded(orderId)`
7. **Send Notifications**: Push + email to customer and chef
8. **Return**: Updated order entity

**Abuse Check Logic**:
```typescript
const abuseCheckResult = validateCancellationAbuse(
  {
    orderStatus: order.status,
    trustState: order.user.trustState || 'NORMAL',
    hasPickedUp: !!order.deliveryPartnerId,
    isSystemAction: false, // Chef-initiated
  },
  orderConfig,
);

// Log for pattern analysis (even if allowed)
console.log(`[ABUSE-CHECK] Chef rejection for order ${orderId}:`, {
  userId: order.userId,
  trustState: order.user.trustState,
  hasPickedUp,
  allowed: abuseCheckResult.allowed,
  reason: abuseCheckResult.reason,
});
```

---

#### **Method 4: updateOrderStatus**

**Signature**:
```typescript
async updateOrderStatus(
  orderId: string,
  chefId: string,
  dto: UpdatePrepStatusDto,
): Promise<Order>
```

**Steps**:
1. **Fetch Order**: Validate exists
2. **Validate Ownership**: `order.chefId === chefId`
3. **Build Domain Model**: For domain rule validation
4. **Validate Transition**:
   - If `PREPARING`: Use `canChefStartPreparing()`
   - If `READY`: Use `canChefMarkReady()`
5. **Update Order**:
   - Update `status` (main field)
   - Update `chefStatus` (legacy field)
   - Set `readyAt` if marking READY
6. **Send Notification**: If marking READY, notify customer
7. **Return**: Updated order entity

---

#### **Method 5: updatePrepTime**

**Signature**:
```typescript
async updatePrepTime(
  orderId: string,
  chefId: string,
  estimatedPrepMinutes: number,
): Promise<Order>
```

**Steps**:
1. **Fetch Order**: Validate exists
2. **Validate Ownership**: `order.chefId === chefId`
3. **Validate Prep Time**: Range 5-180 minutes
4. **Validate Order State**: Must be ACCEPTED or PREPARING
5. **Calculate Time Difference**: `Math.abs(new - old)`
6. **Update Order**: `estimatedPrepMinutes = new value`
7. **Conditional Notification**:
   - If `difference >= 10 minutes`: Send notification
   - If `difference < 10 minutes`: Skip notification
8. **Return**: Updated order entity

---

## 💾 **Database Schema**

### **Order Entity** (Partial - Chef Orders Relevant Fields)

**Table**: `orders`

**Relevant Columns**:
```sql
CREATE TABLE orders (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  
  -- Ownership
  user_id UUID NOT NULL REFERENCES users(id),
  chef_id UUID NOT NULL REFERENCES users(id),
  
  -- Status Fields
  status VARCHAR(50) NOT NULL DEFAULT 'DRAFT',
  chef_status VARCHAR(50) NOT NULL DEFAULT 'NEW',
  payment_status VARCHAR(50) NOT NULL DEFAULT 'PENDING',
  
  -- Pricing (in paise)
  total_paise INTEGER NOT NULL,
  
  -- Items & Address (JSONB)
  items JSONB NOT NULL,
  address_snapshot JSONB NOT NULL,
  attribution JSONB,
  
  -- Chef Management
  estimated_prep_minutes INTEGER,
  chef_note TEXT,
  instructions TEXT,
  
  -- Timeline
  created_at TIMESTAMP NOT NULL DEFAULT NOW(),
  accepted_at TIMESTAMP,
  rejected_at TIMESTAMP,
  ready_at TIMESTAMP,
  
  -- Indexes
  INDEX idx_orders_chef_status (chef_id, status, payment_status),
  INDEX idx_orders_timeline (chef_id, created_at DESC)
);
```

**Status Enums**:
```typescript
// Main Order Status (Industry Standard)
enum OrderStatus {
  DRAFT = 'DRAFT',           // Cart not yet checked out
  PAID = 'PAID',             // Payment successful, awaiting chef acceptance
  ACCEPTED = 'ACCEPTED',     // Chef accepted, preparing
  PREPARING = 'PREPARING',   // Chef cooking
  READY = 'READY',           // Ready for pickup/delivery
  OUT_FOR_DELIVERY = 'OUT_FOR_DELIVERY', // Courier picked up
  DELIVERED = 'DELIVERED',   // Customer received
  COMPLETED = 'COMPLETED',   // Transaction complete
  CANCELLED = 'CANCELLED',   // Cancelled/Rejected
  REFUNDED = 'REFUNDED',     // Refund processed
}

// Chef Order Status (Legacy - Backward Compatibility)
enum ChefOrderStatus {
  NEW = 'NEW',
  ACCEPTED = 'ACCEPTED',
  REJECTED = 'REJECTED',
  PREPARING = 'PREPARING',
  READY = 'READY',
  OUT_FOR_DELIVERY = 'OUT_FOR_DELIVERY',
  COMPLETED = 'COMPLETED',
}
```

**JSONB Schema**:

**items**:
```json
[
  {
    "menuItemId": "uuid",
    "quantity": 2,
    "unitPricePaise": 25000,
    "titleSnapshot": "Butter Chicken",
    "imageSnapshot": "https://...",
    "optionsSnapshot": {"size": "large"},
    "removedIngredients": ["onions"],
    "addedIngredients": [{"name": "extra cheese", "pricePaise": 2000}],
    "customerCookingPreferences": "Extra spicy"
  }
]
```

**addressSnapshot**:
```json
{
  "fullName": "Rajesh Kumar",
  "phone": "+919876543210",
  "line1": "123 MG Road",
  "line2": "Near City Mall",
  "city": "Mumbai",
  "state": "Maharashtra",
  "pincode": "400001",
  "country": "India",
  "coordinates": {"lat": 19.0760, "lng": 72.8777}
}
```

**attribution** (optional):
```json
{
  "linkedReelId": "reel-uuid",
  "linkedCreatorOrderId": "creator-order-uuid",
  "creatorOrderValue": 45000
}
```

---

### **Related Entities**

**ChefKitchen Entity** (Partial):
```typescript
@Entity('chef_kitchens')
export class ChefKitchen {
  @PrimaryGeneratedColumn('uuid')
  id!: string;

  @Column({ name: 'chef_id', type: 'uuid' })
  chefId!: string;

  @Column({ name: 'is_online', default: false })
  isOnline!: boolean;

  @Column({ name: 'accepting_orders', default: true })
  acceptingOrders!: boolean;
}
```

**ChefMenuItem Entity** (Partial):
```typescript
@Entity('chef_menu_items')
export class ChefMenuItem {
  @PrimaryGeneratedColumn('uuid')
  id!: string;

  @Column({ name: 'chef_kitchen_id', type: 'uuid' })
  chefKitchenId!: string;

  @Column({ type: 'varchar', length: 255 })
  name!: string;

  @Column({ name: 'price_paise', type: 'integer' })
  pricePaise!: number;

  @Column({ name: 'is_active', default: true })
  isActive!: boolean;
}
```

---

## 📦 **DTOs**

### **AcceptOrderDto**

```typescript
import { IsInt, IsOptional, IsString, Max, MaxLength, Min } from 'class-validator';
import { ApiProperty } from '@nestjs/swagger';

export class AcceptOrderDto {
  @ApiProperty({
    description: 'Estimated preparation time in minutes',
    minimum: 5,
    maximum: 180,
    example: 30,
  })
  @IsInt()
  @Min(5, { message: 'Minimum preparation time is 5 minutes' })
  @Max(180, { message: 'Maximum preparation time is 180 minutes (3 hours)' })
  estimatedPrepMinutes!: number;

  @ApiProperty({
    description: 'Optional note from chef (e.g., substitutions, delays)',
    required: false,
    maxLength: 500,
    example: 'Will use cottage cheese instead of paneer',
  })
  @IsOptional()
  @IsString()
  @MaxLength(500)
  chefNote?: string;
}
```

---

### **RejectOrderDto**

```typescript
import { IsNotEmpty, IsString, MaxLength } from 'class-validator';
import { ApiProperty } from '@nestjs/swagger';

export class RejectOrderDto {
  @ApiProperty({
    description: 'Reason for rejecting the order (required)',
    maxLength: 500,
    example: 'Out of stock for main ingredient',
  })
  @IsNotEmpty({ message: 'Rejection reason is required' })
  @IsString()
  @MaxLength(500)
  reason!: string;
}
```

---

### **UpdatePrepStatusDto**

```typescript
import { IsEnum } from 'class-validator';
import { ApiProperty } from '@nestjs/swagger';

export enum ChefPrepStatus {
  PREPARING = 'PREPARING',
  READY = 'READY',
}

export class UpdatePrepStatusDto {
  @ApiProperty({
    description: 'New preparation status',
    enum: ChefPrepStatus,
    example: 'PREPARING',
  })
  @IsEnum(ChefPrepStatus, { message: 'Status must be PREPARING or READY' })
  status!: ChefPrepStatus;
}
```

---

### **UpdatePrepTimeDto**

```typescript
import { IsInt, Min, Max } from 'class-validator';
import { ApiProperty } from '@nestjs/swagger';

export class UpdatePrepTimeDto {
  @ApiProperty({
    description: 'New estimated preparation time in minutes',
    minimum: 5,
    maximum: 180,
    example: 45,
  })
  @IsInt()
  @Min(5, { message: 'Minimum preparation time is 5 minutes' })
  @Max(180, { message: 'Maximum preparation time is 180 minutes (3 hours)' })
  estimatedPrepMinutes!: number;
}
```

---

## 🔗 **Shared Types**

**Package**: `@chefooz-app/types`

**File**: `libs/types/src/lib/chef-orders.types.ts`

### **ChefOrder Interface**

```typescript
export interface ChefOrder {
  id: string;
  orderNumber: string;
  userId: string;
  customerName: string;
  customerPhone: string;
  items: ChefOrderItem[];
  totalAmount: number; // In rupees
  deliveryAddress: DeliveryAddress;
  chefStatus: ChefOrderStatus;
  estimatedPrepMinutes?: number;
  chefNote?: string;
  specialInstructions?: string;
  createdAt: string; // ISO string
  acceptedAt?: string;
  rejectedAt?: string;
  readyAt?: string;
  // Computed fields
  statusGroup: OrderStatusGroup;
  prepRemainingMinutes?: number; // Negative if overdue
  isOverdue: boolean;
  reelSource?: {
    reelId: string;
    creatorName: string;
  };
}
```

### **ChefOrderStatus Enum**

```typescript
export enum ChefOrderStatus {
  NEW = 'NEW',
  ACCEPTED = 'ACCEPTED',
  REJECTED = 'REJECTED',
  PREPARING = 'PREPARING',
  READY = 'READY',
  OUT_FOR_DELIVERY = 'OUT_FOR_DELIVERY',
  COMPLETED = 'COMPLETED',
}
```

### **OrderStatusGroup Enum**

```typescript
export enum OrderStatusGroup {
  NEW = 'NEW',           // NEW status
  ACTIVE = 'ACTIVE',     // ACCEPTED, PREPARING
  READY = 'READY',       // READY, OUT_FOR_DELIVERY
  COMPLETED = 'COMPLETED', // COMPLETED, REJECTED
}
```

---

## 🔌 **Integration Patterns**

### **Integration 1: Notification Module**

**Purpose**: Send push notifications and emails to customers

**Usage**:
```typescript
// Push Notification
await this.notificationDispatcher.send(
  userId,
  'order.accepted',
  { orderId, estimatedPrepMinutes }
);

// Email Notification
await this.emailEventDispatcher.sendOrderCancelledEmails(
  orderId,
  reason,
  refundExpected
);
```

**Event Types**:
- `order.accepted`: Chef accepted order
- `order.cancelled`: Order cancelled/rejected
- `order.cooking`: Order ready for delivery (legacy event name)
- `order.prep_time_updated`: Prep time adjusted

---

### **Integration 2: Order Module**

**Purpose**: Refund processing on order rejection

**Usage**:
```typescript
// Trigger automatic refund
await this.orderService.handleOrderRefunded(orderId);
```

**Refund Flow**:
1. OrderService fetches order and payment record
2. Calls Razorpay refund API (full amount)
3. Updates payment status to REFUNDED
4. Updates order status to REFUNDED
5. Logs refund transaction

---

### **Integration 3: Domain Package**

**Purpose**: Centralized business rule validation

**Functions**:
```typescript
import {
  canChefAcceptOrder,
  canChefRejectOrder,
  canChefStartPreparing,
  canChefMarkReady,
} from '@chefooz-app/domain';

// Validate acceptance
if (!canChefAcceptOrder(orderDomain, order.chefStatus)) {
  throw BadRequestException('Invalid status transition');
}

// Validate rejection
if (!canChefRejectOrder(order.chefStatus)) {
  throw BadRequestException('Only NEW orders can be rejected');
}
```

**Domain Model**:
```typescript
interface OrderDomainModel {
  id: string;
  userId: string;
  chefId: string;
  status: string;
  paymentStatus: string;
  totalAmountPaise: number;
  createdAt: Date;
}
```

---

### **Integration 4: Chef-Kitchen Module**

**Purpose**: Kitchen availability validation

**Usage**:
```typescript
const chefKitchen = await this.chefKitchenRepository.findOne({
  where: { chefId },
});

if (!chefKitchen.isOnline) {
  throw BadRequestException('Kitchen offline');
}

if (!chefKitchen.acceptingOrders) {
  throw BadRequestException('Kitchen paused');
}
```

**Kitchen States**:
- `isOnline = true`: Chef is actively working
- `acceptingOrders = true`: Kitchen has capacity for new orders

---

## ⚠️ **Error Handling**

### **Error Codes**

| Error Code | HTTP Status | Description |
|------------|-------------|-------------|
| `INCOMING_ORDERS_FETCH_FAILED` | 500 | Database query error |
| `ORDER_ACCEPT_FAILED` | 400/403/404 | Accept validation failed |
| `ORDER_REJECT_FAILED` | 400/403/404 | Reject validation failed |
| `ORDER_STATUS_UPDATE_FAILED` | 400/403/404 | Status update failed |
| `PREP_TIME_UPDATE_FAILED` | 400/403/404 | Prep time update failed |

### **Common Error Patterns**

**Kitchen Offline**:
```typescript
throw new BadRequestException(
  'Cannot accept orders while offline. Please set your kitchen status to "Online" first.'
);
```

**Kitchen Paused**:
```typescript
throw new BadRequestException(
  'Cannot accept orders at this time. Your kitchen is currently paused. ' +
  'Please enable "Accepting Orders" in your kitchen settings.'
);
```

**Unauthorized**:
```typescript
if (order.chefId !== chefId) {
  throw new ForbiddenException(
    'You are not authorized to manage this order'
  );
}
```

**Invalid Status**:
```typescript
if (!canChefAcceptOrder(orderDomain, order.chefStatus)) {
  throw new BadRequestException(
    `Cannot accept order. Order must be PAID and chef status must be NEW. ` +
    `Current order status: ${order.status}, chef status: ${order.chefStatus}`
  );
}
```

---

## 🧪 **Testing Strategy**

### **Unit Tests**

**File**: `chef-orders.service.spec.ts`

**Test Cases**:
1. **getIncomingOrders**:
   - Returns orders for chef
   - Filters by payment status
   - Transforms to ChefOrderResponse
   - Calculates computed fields correctly
   - Handles empty results

2. **acceptOrder**:
   - Validates kitchen availability
   - Validates order ownership
   - Validates status transition
   - Updates order correctly
   - Sends notification
   - Handles errors gracefully

3. **rejectOrder**:
   - Validates order ownership
   - Validates rejection allowed
   - Triggers refund
   - Sends notifications
   - Logs abuse check

4. **updateOrderStatus**:
   - Validates status transitions
   - Updates main + chef status
   - Sets readyAt timestamp
   - Sends notification on READY

5. **updatePrepTime**:
   - Validates prep time range
   - Validates order state
   - Sends notification if ≥10 min change
   - Skips notification if <10 min

### **Integration Tests**

**File**: `chef-orders.e2e.spec.ts`

**Test Scenarios**:
1. Full order acceptance flow
2. Order rejection with refund
3. Status progression (ACCEPTED → PREPARING → READY)
4. Prep time update with notification
5. Kitchen offline prevention
6. Unauthorized access prevention

---

**[CHEF_ORDERS_TECHNICAL_GUIDE_COMPLETE ✅]**

*For business overview, see `01_FEATURE_OVERVIEW.md`. For testing scenarios, see `03_QA_TEST_CASES.md`.*

---

**Document Version**: 1.0  
**Last Updated**: February 22, 2026  
**Module**: Chef-Orders (Week 7 - Chef Fulfillment)  
**Status**: ✅ Complete
