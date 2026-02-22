# 🚚 Delivery Module - Technical Guide

## 📋 **Table of Contents**
- [Architecture Overview](#architecture-overview)
- [Module Structure](#module-structure)
- [API Endpoints](#api-endpoints)
- [Service Layer](#service-layer)
- [Database Schema](#database-schema)
- [DTOs](#dtos)
- [Domain Rules](#domain-rules)
- [Integration Patterns](#integration-patterns)
- [Background Jobs](#background-jobs)
- [Error Handling](#error-handling)
- [Testing Strategy](#testing-strategy)

---

## 🏗️ **Architecture Overview**

### **System Components**

```
┌───────────────────────────────────────────────────────────────────┐
│                  DELIVERY MODULE ECOSYSTEM                         │
└───────────────────────────────────────────────────────────────────┘
                                 │
         ┌───────────────────────┼────────────────────────┐
         │                       │                        │
         ▼                       ▼                        ▼
┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐
│   Delivery       │   │   Delivery       │   │    DTOs          │
│   Controller     │   │   Service        │   │                  │
│                  │   │                  │   │ - UpdateStatus   │
│ PUT /status      │   │ - Status mgmt    │   │ - AcceptReq      │
│ GET /requests    │   │ - Request list   │   │ - RejectReq      │
│ POST /accept     │   │ - Accept/reject  │   │ - UpdateState    │
│ POST /reject     │   │ - State tracking │   │ - Complete       │
│ GET /active      │   │ - Completion     │   │                  │
│ PUT /state       │   │ - Stats          │   │                  │
│ POST /complete   │   │                  │   │                  │
│ GET /stats       │   │                  │   │                  │
│ GET /busy-status │   │                  │   │                  │
└──────────────────┘   └──────────────────┘   └──────────────────┘
         │                       │                        │
         └───────────────────────┼────────────────────────┘
                                 │
         ┌───────────────────────┼────────────────────────┐
         │                       │                        │
         ▼                       ▼                        ▼
┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐
│ DeliveryRequest  │   │ ActiveDelivery   │   │   RiderProfile   │
│   Entity         │   │    Entity        │   │     Entity       │
│ (TypeORM)        │   │  (TypeORM)       │   │   (TypeORM)      │
│                  │   │                  │   │                  │
│ - orderId        │   │ - orderId        │   │ - userId         │
│ - riderId        │   │ - riderId        │   │ - isOnline       │
│ - chefLocation   │   │ - state          │   │ - isBusy         │
│ - customerLoc    │   │ - pickedUpAt     │   │ - activeOrders   │
│ - status         │   │ - deliveredAt    │   │ - maxConcurrent  │
│ - expiresAt      │   │ - payoutPaise    │   │ - city           │
│ - payoutPaise    │   │                  │   │                  │
└──────────────────┘   └──────────────────┘   └──────────────────┘
         │                       │                        │
         └───────────────────────┼────────────────────────┘
                                 │
         ┌───────────────────────┼────────────────────────┐
         │                       │                        │
         ▼                       ▼                        ▼
┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐
│   Assignment     │   │     Timeout      │   │   RiderBusy      │
│    Service       │   │    Processors    │   │    Service       │
│                  │   │                  │   │                  │
│ - FIFO logic     │   │ - Assignment     │   │ - Capacity calc  │
│ - Distance calc  │   │ - Pickup         │   │ - Busy detection │
│ - Retry mgmt     │   │ - Delivery       │   │                  │
│ - Bull queue     │   │ (Cron + Bull)    │   │                  │
└──────────────────┘   └──────────────────┘   └──────────────────┘
         │                       │                        │
         └───────────────────────┼────────────────────────┘
                                 │
         ┌───────────────────────┼────────────────────────┐
         │                       │                        │
         ▼                       ▼                        ▼
┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐
│  Notification    │   │      Order       │   │     Domain       │
│   Dispatcher     │   │     Service      │   │     Rules        │
│                  │   │                  │   │   (@domain)      │
│ - Push notifs    │   │ - Auto-cancel    │   │                  │
│ - Email send     │   │ - Status update  │   │ - canAccept      │
│                  │   │ - Refund logic   │   │ - isEligible     │
└──────────────────┘   └──────────────────┘   └──────────────────┘
```

### **Request Flow: Order Assignment**

```mermaid
sequenceDiagram
    participant Chef
    participant System
    participant Assignment
    participant Bull
    participant Rider
    participant Customer

    Chef->>System: Mark order READY
    System->>Assignment: Trigger rider assignment
    Assignment->>Assignment: Find eligible riders (FIFO + distance)
    Assignment->>Assignment: Lock order row (transaction)
    
    alt Rider Found
        Assignment->>Bull: Schedule timeout check (30s delay)
        Assignment->>Rider: Send delivery request
        Assignment-->>System: Assignment created
        System->>Customer: Notify "Finding delivery partner..."
        
        alt Rider Accepts (within 30s)
            Rider->>System: Accept request
            System->>Assignment: Create active delivery
            System->>Rider: Show navigation
            System->>Customer: Notify "Rider assigned"
            Bull->>Bull: Cancel timeout check (rider accepted)
        else Rider Rejects or Timeout
            Bull->>Assignment: Timeout check triggered
            Assignment->>Assignment: Retry with next rider (max 5 times)
        end
    else No Rider Found
        Assignment->>System: Auto-cancel order
        System->>Customer: Notify cancellation + refund
    end
```

---

## 📁 **Module Structure**

```
apps/chefooz-apis/src/modules/delivery/
├── delivery.controller.ts           # 9 HTTP endpoints (rider-facing)
├── delivery-analytics.controller.ts # Admin analytics endpoints
├── delivery.service.ts              # Core business logic (400 lines)
├── delivery.module.ts               # Module definition + dependencies
├── dto/
│   ├── delivery.dto.ts              # 5 DTOs (status, accept, reject, state, complete)
│   └── delivery-analytics.dto.ts    # Analytics query DTOs
├── services/
│   ├── delivery-assignment.service.ts     # FIFO assignment logic (652 lines)
│   ├── assignment-retry.service.ts        # Retry management
│   ├── assignment-timeout.processor.ts    # Bull queue timeout processor
│   ├── delivery-timeout.cron.ts           # Pickup/delivery timeout monitor
│   ├── rider-availability.service.ts      # Online/offline tracking
│   ├── rider-busy.service.ts              # Capacity management
│   ├── rider-fraud.service.ts             # Fraud detection (Phase 3.6.9)
│   └── delivery-analytics.service.ts      # Admin analytics
└── utils/
    └── delivery-assignment.util.ts        # Helper functions
```

### **Module Definition**

```typescript
// delivery.module.ts
import { Module, forwardRef } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { BullModule } from '@nestjs/bull';

@Module({
  imports: [
    TypeOrmModule.forFeature([
      User,
      Order,
      DeliveryRequest,
      ActiveDelivery,
      RiderProfile,
      RiderLocation,
      TipTransaction,
      ChefKitchen,
    ]),
    BullModule.registerQueue({
      name: 'delivery-assignment',
    }),
    CacheModule,
    AuthModule,
    NotificationModule,
    RiderLocationModule,
    forwardRef(() => OrderModule), // Circular dependency for auto-cancel
  ],
  controllers: [DeliveryController, DeliveryAnalyticsController],
  providers: [
    DeliveryService,
    DeliveryAssignmentService,
    RiderAvailabilityService,
    RiderBusyService,
    DeliveryAnalyticsService,
    AssignmentTimeoutProcessor,
    AssignmentRetryService,
    DeliveryTimeoutCron,
    RiderFraudService,
  ],
  exports: [
    DeliveryService,
    DeliveryAssignmentService,
    RiderBusyService,
    DeliveryAnalyticsService,
    RiderFraudService,
  ],
})
export class DeliveryModule {}
```

**Key Dependencies**:
- **OrderModule**: Auto-cancellation logic (forwardRef to avoid circular dependency)
- **NotificationModule**: Push notifications + email dispatch
- **RiderLocationModule**: GPS tracking + location updates
- **BullModule**: Background job queue (timeout checks, retries)
- **CacheModule**: Request deduplication, rider availability caching

---

## 🔌 **API Endpoints**

**Base URL**: `/v1/delivery`  
**Authentication**: JWT Bearer token (rider user)  
**Content-Type**: `application/json`

---

### **Endpoint 1: Update Delivery Status**

**URL**: `PUT /v1/delivery/status`

**Description**: Toggle rider availability (online/offline/delivering)

**Authentication**: Required (JWT)

**Rate Limit**: 60 requests/minute

**Request Headers**:
```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json
```

**Request Body**:
```json
{
  "status": "online"
}
```

**Request Body Schema**:
```typescript
{
  status: 'offline' | 'online' | 'delivering'; // Required
}
```

**Validation Rules**:
- `status`:
  - Required
  - Must be enum: `offline`, `online`, or `delivering`
  - Cannot set `offline` with active delivery

**Response** (200 OK):
```json
{
  "success": true,
  "message": "Delivery status updated to online",
  "data": {
    "deliveryStatus": "online",
    "currentDeliveryId": null
  }
}
```

**Error Response** (400 - Active Delivery):
```json
{
  "success": false,
  "message": "Cannot go offline with active delivery",
  "errorCode": "ACTIVE_DELIVERY_EXISTS"
}
```

**Business Logic**:
```typescript
// Validate active delivery
if (dto.status === 'offline' && user.currentDeliveryId) {
  throw BadRequestException('Cannot go offline with active delivery');
}

// Update status
user.deliveryStatus = dto.status;
await userRepository.save(user);
```

**Side Effects**:
- User `deliveryStatus` field updated
- If going online → Rider becomes eligible for assignments
- If going offline → Rider excluded from assignment algorithm
- Auto-status changes (accept → 'delivering', complete → 'online')

---

### **Endpoint 2: Get Current Delivery Status**

**URL**: `GET /v1/delivery/status`

**Description**: Retrieve rider's current availability status

**Authentication**: Required (JWT)

**Response** (200 OK):
```json
{
  "success": true,
  "message": "Delivery status retrieved",
  "data": {
    "deliveryStatus": "online",
    "currentDeliveryId": null
  }
}
```

**Performance**: ~50ms response time (simple user lookup)

---

### **Endpoint 3: Get Pending Delivery Requests**

**URL**: `GET /v1/delivery/requests/pending`

**Description**: Fetch delivery requests awaiting rider acceptance

**Authentication**: Required (JWT)

**Rate Limit**: None (real-time dashboard refresh)

**Query Parameters**: None

**Response** (200 OK):
```json
{
  "success": true,
  "message": "Found 3 pending requests",
  "data": [
    {
      "id": "request-uuid-123",
      "orderId": "order-uuid-456",
      "chefId": "chef-uuid-789",
      "chefName": "Chef Ravi",
      "chefLocation": {
        "latitude": 12.9716,
        "longitude": 77.5946,
        "address": "123 MG Road, Bangalore",
        "landmark": "Near City Mall"
      },
      "customerLocation": {
        "latitude": 12.9800,
        "longitude": 77.6000,
        "address": "456 Customer Street",
        "landmark": "Blue building"
      },
      "distanceToPickupKm": 3.2,
      "estimatedPayoutPaise": 8500,
      "estimatedDeliveryMinutes": 25,
      "expiresAt": "2026-02-22T10:35:30.000Z",
      "status": "pending",
      "createdAt": "2026-02-22T10:35:00.000Z"
    }
  ]
}
```

**Response Fields**:
- `distanceToPickupKm`: Calculated using Haversine formula (from rider current location to chef location)
- `estimatedPayoutPaise`: Base payout + distance multiplier + surge pricing (if applicable)
- `expiresAt`: Request expiry time (typically 30 seconds from creation)

**Filtering Logic**:
```typescript
// Only pending requests within expiry window
const requests = await deliveryRequestRepository.find({
  where: {
    status: DeliveryRequestStatus.PENDING,
    expiresAt: MoreThan(new Date()),
  },
  order: { createdAt: 'ASC' }, // FIFO
  take: 5, // Limit to avoid overwhelming rider
});
```

**Business Rules**:
1. **Rider must be online** → Returns empty array if offline
2. **Max 5 requests shown** → Prevents cognitive overload
3. **FIFO ordering** → Oldest requests first
4. **Auto-expire filtering** → Only show valid requests

---

### **Endpoint 4: Accept Delivery Request**

**URL**: `POST /v1/delivery/requests/accept`

**Description**: Accept a delivery request and create active delivery

**Authentication**: Required (JWT)

**Rate Limit**: 60 requests/minute

**Request Body**:
```json
{
  "deliveryRequestId": "request-uuid-123",
  "currentLocation": {
    "latitude": 12.9750,
    "longitude": 77.5980,
    "address": "Rider current location"
  }
}
```

**Request Body Schema**:
```typescript
{
  deliveryRequestId: string; // Required, UUID
  currentLocation?: {        // Optional
    latitude: number;        // -90 to 90
    longitude: number;       // -180 to 180
    address?: string;
    landmark?: string;
  };
}
```

**Response** (200 OK):
```json
{
  "success": true,
  "message": "Delivery accepted",
  "data": {
    "id": "active-delivery-uuid-999",
    "orderId": "order-uuid-456",
    "riderId": "user-uuid-123",
    "chefId": "chef-uuid-789",
    "chefName": "Chef Ravi",
    "chefLocation": { "latitude": 12.9716, "longitude": 77.5946 },
    "customerId": "customer-uuid-111",
    "customerName": "John Doe",
    "customerLocation": { "latitude": 12.9800, "longitude": 77.6000 },
    "orderItems": ["Butter Chicken", "Garlic Naan"],
    "orderValuePaise": 55000,
    "deliveryInstructions": "Ring doorbell, leave at door",
    "state": "navigate_to_chef",
    "estimatedPayoutPaise": 8500,
    "assignedAt": "2026-02-22T10:36:00.000Z"
  }
}
```

**Error Response** (400 - Already Has Active Delivery):
```json
{
  "success": false,
  "message": "Already have an active delivery",
  "errorCode": "ACTIVE_DELIVERY_EXISTS"
}
```

**Error Response** (400 - Request Expired):
```json
{
  "success": false,
  "message": "Request has expired",
  "errorCode": "REQUEST_EXPIRED"
}
```

**Error Response** (400 - Already Processed):
```json
{
  "success": false,
  "message": "Request already processed",
  "errorCode": "REQUEST_ALREADY_PROCESSED"
}
```

**Business Logic**:
```typescript
// 1. Validate rider eligibility
if (user.deliveryStatus !== 'online') {
  throw BadRequestException('Must be online to accept deliveries');
}

if (user.currentDeliveryId) {
  throw BadRequestException('Already have an active delivery');
}

// 2. Validate request
const request = await deliveryRequestRepository.findOne({
  where: { id: dto.deliveryRequestId },
  relations: ['order'],
});

if (!request || request.status !== 'PENDING') {
  throw NotFoundException('Delivery request not found or already processed');
}

if (new Date() > request.expiresAt) {
  request.status = 'EXPIRED';
  await deliveryRequestRepository.save(request);
  throw BadRequestException('Request has expired');
}

// 3. Mark request as accepted
request.status = 'ACCEPTED';
request.riderId = userId;
await deliveryRequestRepository.save(request);

// 4. Create active delivery
const activeDelivery = activeDeliveryRepository.create({
  orderId: request.orderId,
  riderId: userId,
  chefId: request.chefId,
  chefName: request.chefName,
  chefLocation: request.chefLocation,
  customerId: request.order.userId,
  customerName: 'Customer', // TODO: Fetch from user
  customerLocation: request.customerLocation,
  orderItems: extractOrderItems(request.order),
  orderValuePaise: request.order.totalPaise,
  deliveryInstructions: request.order.instructions,
  state: 'navigate_to_chef',
  estimatedPayoutPaise: request.estimatedPayoutPaise,
  assignedAt: new Date(),
});

const savedDelivery = await activeDeliveryRepository.save(activeDelivery);

// 5. Update user status
user.deliveryStatus = 'delivering';
user.currentDeliveryId = savedDelivery.id;
await userRepository.save(user);

// 6. Send notifications
await notificationDispatcher.send(
  request.order.userId,
  'delivery.assigned',
  { orderId, riderName: user.fullName }
);
```

**Side Effects**:
- `DeliveryRequest` status → `ACCEPTED`
- `ActiveDelivery` created
- User status → `'delivering'`
- User `currentDeliveryId` → active delivery ID
- Customer receives "Rider assigned" notification
- Chef receives "Rider coming for pickup" notification

---

### **Endpoint 5: Reject Delivery Request**

**URL**: `POST /v1/delivery/requests/reject`

**Description**: Reject a delivery request with reason

**Authentication**: Required (JWT)

**Rate Limit**: 60 requests/minute

**Request Body**:
```json
{
  "deliveryRequestId": "request-uuid-123",
  "reason": "too_far",
  "note": "Distance exceeds my preference"
}
```

**Request Body Schema**:
```typescript
{
  deliveryRequestId: string;                          // Required, UUID
  reason: 'too_far' | 'busy' | 'low_payout' | 'other'; // Required
  note?: string;                                       // Optional
}
```

**Response** (200 OK):
```json
{
  "success": true,
  "message": "Delivery rejected"
}
```

**Business Logic**:
```typescript
// 1. Validate request exists and is pending
const request = await deliveryRequestRepository.findOne({
  where: { id: dto.deliveryRequestId },
});

if (!request || request.status !== 'PENDING') {
  throw NotFoundException('Delivery request not found or already processed');
}

// 2. Mark as rejected
request.status = 'REJECTED';
request.riderId = userId;
request.rejectionReason = dto.reason;
request.rejectionNote = dto.note;
await deliveryRequestRepository.save(request);

// 3. Trigger re-assignment to next rider
await deliveryAssignmentService.retryAssignment(request.orderId);
```

**Side Effects**:
- Request status → `REJECTED`
- Rejection reason + note logged (for analytics)
- Re-assignment triggered (next rider receives request)
- No penalty if rejection rate <50% in 24 hours
- Reputation penalty if rejection rate >80%

---

### **Endpoint 6: Get Active Delivery**

**URL**: `GET /v1/delivery/active`

**Description**: Retrieve rider's current active delivery

**Authentication**: Required (JWT)

**Response** (200 OK - Active Delivery):
```json
{
  "success": true,
  "message": "Active delivery retrieved",
  "data": {
    "id": "active-delivery-uuid-999",
    "orderId": "order-uuid-456",
    "chefName": "Chef Ravi",
    "chefLocation": { "latitude": 12.9716, "longitude": 77.5946 },
    "customerName": "John Doe",
    "customerLocation": { "latitude": 12.9800, "longitude": 77.6000 },
    "orderItems": ["Butter Chicken", "Garlic Naan"],
    "orderValuePaise": 55000,
    "deliveryInstructions": "Ring doorbell, leave at door",
    "state": "navigate_to_chef",
    "estimatedPayoutPaise": 8500,
    "assignedAt": "2026-02-22T10:36:00.000Z",
    "pickedUpAt": null,
    "deliveredAt": null
  }
}
```

**Response** (200 OK - No Active Delivery):
```json
{
  "success": true,
  "message": "No active delivery",
  "data": null
}
```

**State Cleanup**:
```typescript
// If currentDeliveryId set but delivery not found
if (user.currentDeliveryId && !delivery) {
  // Cleanup inconsistent state
  user.currentDeliveryId = undefined;
  user.deliveryStatus = 'online';
  await userRepository.save(user);
}
```

---

### **Endpoint 7: Update Delivery State**

**URL**: `PUT /v1/delivery/active/state`

**Description**: Update delivery progress (navigate → picked_up → en_route → delivered)

**Authentication**: Required (JWT)

**Rate Limit**: 120 requests/minute

**Request Body**:
```json
{
  "deliveryId": "active-delivery-uuid-999",
  "state": "picked_up",
  "currentLocation": {
    "latitude": 12.9716,
    "longitude": 77.5946
  }
}
```

**Request Body Schema**:
```typescript
{
  deliveryId: string; // Required, UUID
  state: 'navigate_to_chef' | 'picked_up' | 'en_route' | 'delivered'; // Required
  currentLocation?: {
    latitude: number;
    longitude: number;
    address?: string;
  };
}
```

**Response** (200 OK):
```json
{
  "success": true,
  "message": "Delivery state updated to picked_up",
  "data": {
    "id": "active-delivery-uuid-999",
    "state": "picked_up",
    "pickedUpAt": "2026-02-22T10:50:00.000Z"
  }
}
```

**Business Logic**:
```typescript
// 1. Validate delivery belongs to rider
const delivery = await activeDeliveryRepository.findOne({
  where: { id: dto.deliveryId, riderId: userId },
});

if (!delivery) {
  throw NotFoundException('Active delivery not found');
}

// 2. Update state
delivery.state = dto.state;

// 3. Set timestamps based on state
if (dto.state === 'picked_up' && !delivery.pickedUpAt) {
  delivery.pickedUpAt = new Date();
  
  // Update order status
  await orderRepository.update(delivery.orderId, {
    status: 'OUT_FOR_DELIVERY',
    deliveryPickedUpAt: new Date(),
  });
  
  // Notify customer
  await notificationDispatcher.send(
    delivery.customerId,
    'delivery.picked_up',
    { orderId: delivery.orderId }
  );
} else if (dto.state === 'delivered' && !delivery.deliveredAt) {
  delivery.deliveredAt = new Date();
  
  // This transitions to completion flow
  // (typically handled by POST /active/complete endpoint)
}

await activeDeliveryRepository.save(delivery);
```

**State Transition Validation** (Domain Rule):
```typescript
// Valid transitions only
navigate_to_chef → picked_up ✅
navigate_to_chef → en_route ❌ (must pick up first)

picked_up → en_route ✅
picked_up → navigate_to_chef ❌ (cannot go backward)

en_route → delivered ✅
delivered → (any state) ❌ (final state)
```

---

### **Endpoint 8: Complete Delivery**

**URL**: `POST /v1/delivery/active/complete`

**Description**: Mark delivery as completed with optional proof

**Authentication**: Required (JWT)

**Rate Limit**: 60 requests/minute

**Request Body**:
```json
{
  "deliveryId": "active-delivery-uuid-999",
  "deliveryProof": "https://s3.amazonaws.com/proof-photo-url.jpg",
  "finalLocation": {
    "latitude": 12.9800,
    "longitude": 77.6000
  }
}
```

**Request Body Schema**:
```typescript
{
  deliveryId: string;      // Required, UUID
  deliveryProof?: string;  // Optional, photo URL
  finalLocation?: {        // Optional, GPS coordinates at delivery
    latitude: number;
    longitude: number;
    address?: string;
  };
}
```

**Response** (200 OK):
```json
{
  "success": true,
  "message": "Delivery completed",
  "data": {
    "id": "active-delivery-uuid-999",
    "state": "delivered",
    "deliveredAt": "2026-02-22T11:05:00.000Z",
    "deliveryProof": "https://s3.amazonaws.com/proof-photo-url.jpg",
    "actualPayoutPaise": 8500
  }
}
```

**Business Logic**:
```typescript
// 1. Validate delivery belongs to rider
const delivery = await activeDeliveryRepository.findOne({
  where: { id: dto.deliveryId, riderId: userId },
  relations: ['order'],
});

if (!delivery) {
  throw NotFoundException('Active delivery not found');
}

// 2. Mark as delivered
delivery.state = 'delivered';
delivery.deliveredAt = new Date();
delivery.deliveryProof = dto.deliveryProof;
delivery.actualPayoutPaise = delivery.estimatedPayoutPaise; // For now, actual = estimated

await activeDeliveryRepository.save(delivery);

// 3. Update order status
await orderRepository.update(delivery.orderId, {
  status: 'DELIVERED',
  deliveredAt: new Date(),
});

// 4. Update user status back to online
user.deliveryStatus = 'online';
user.currentDeliveryId = undefined;
await userRepository.save(user);

// 5. Notify customer
await notificationDispatcher.send(
  delivery.customerId,
  'delivery.completed',
  { orderId: delivery.orderId }
);

// TODO: Process rider payout (lock for 24 hours for dispute window)
```

**Side Effects**:
- Delivery state → `'delivered'`
- `deliveredAt` timestamp set
- Order status → `'DELIVERED'`
- User status → `'online'` (available for new deliveries)
- User `currentDeliveryId` → `null`
- Customer receives "Order delivered" notification
- Payout scheduled (locked for 24 hours, then released)

---

### **Endpoint 9: Get Delivery Statistics**

**URL**: `GET /v1/delivery/stats`

**Description**: Retrieve rider's delivery performance metrics

**Authentication**: Required (JWT)

**Response** (200 OK):
```json
{
  "success": true,
  "message": "Delivery stats retrieved",
  "data": {
    "totalDeliveries": 127,
    "completedDeliveries": 122,
    "cancelledDeliveries": 5,
    "totalEarningsPaise": 1085000
  }
}
```

**Performance**: ~100ms (aggregation query on active_deliveries table)

---

### **Endpoint 10: Get Busy Status**

**URL**: `GET /v1/delivery/busy-status`

**Description**: Check if rider is at max capacity (for assignment filtering)

**Authentication**: Required (JWT)

**Response** (200 OK - Not Busy):
```json
{
  "success": true,
  "message": "Rider is available for new assignments",
  "data": {
    "isBusy": false,
    "activeDeliveryCount": 1,
    "maxConcurrentDeliveries": 2,
    "reason": null
  }
}
```

**Response** (200 OK - Busy):
```json
{
  "success": true,
  "message": "Rider is busy with active deliveries",
  "data": {
    "isBusy": true,
    "activeDeliveryCount": 2,
    "maxConcurrentDeliveries": 2,
    "reason": "At maximum capacity"
  }
}
```

**Business Logic**:
```typescript
const busyStatus = await riderBusyService.getBusyStatus(userId);

// Calculation
isBusy = activeDeliveryCount >= maxConcurrentDeliveries;
```

**Used By**:
- Assignment service (filter busy riders)
- Rider dashboard (show capacity indicator)

---

## 🔧 **Service Layer**

### **DeliveryService Methods**

#### **Method 1: updateDeliveryStatus**

**Signature**:
```typescript
async updateDeliveryStatus(
  userId: string,
  dto: UpdateDeliveryStatusDto
): Promise<{ success: boolean; message: string; data: any }>
```

**Purpose**: Toggle rider availability (online/offline/delivering)

**Business Rules**:
1. **Cannot go offline with active delivery** → Validation error
2. **Auto-status synchronization** → Status updates based on delivery lifecycle

**Implementation**:
```typescript
// Fetch user
const user = await userRepository.findOne({ where: { id: userId } });
if (!user) throw NotFoundException('User not found');

// Validate offline with active delivery
if (dto.status === 'offline' && user.currentDeliveryId) {
  throw BadRequestException('Cannot go offline with active delivery');
}

// Update status
user.deliveryStatus = dto.status;
await userRepository.save(user);

return {
  success: true,
  message: `Delivery status updated to ${dto.status}`,
  data: {
    deliveryStatus: user.deliveryStatus,
    currentDeliveryId: user.currentDeliveryId,
  },
};
```

---

#### **Method 2: getPendingRequests**

**Signature**:
```typescript
async getPendingRequests(userId: string): Promise<{ success: boolean; message: string; data: DeliveryRequest[] }>
```

**Purpose**: Fetch pending delivery requests for rider

**Query Logic**:
```typescript
// Check rider is online
const user = await userRepository.findOne({ where: { id: userId } });
if (user.deliveryStatus !== 'online') {
  return { success: true, message: 'No requests (rider not online)', data: [] };
}

// Get non-expired pending requests
const requests = await deliveryRequestRepository.find({
  where: {
    status: DeliveryRequestStatus.PENDING,
    expiresAt: MoreThan(new Date()),
  },
  order: { createdAt: 'ASC' }, // FIFO
  take: 5, // Limit to avoid overwhelming rider
});

return { success: true, message: `Found ${requests.length} pending requests`, data: requests };
```

**Performance**: ~80ms (indexed query on `status` + `expiresAt`)

---

#### **Method 3: acceptDeliveryRequest**

**Signature**:
```typescript
async acceptDeliveryRequest(
  userId: string,
  dto: AcceptDeliveryDto
): Promise<{ success: boolean; message: string; data: ActiveDelivery }>
```

**Purpose**: Accept delivery request and create active delivery

**Steps**:
1. **Validate rider eligibility** (online, no active delivery)
2. **Fetch and validate request** (exists, pending, not expired)
3. **Mark request as accepted** (atomic update)
4. **Create active delivery** (with chef/customer details)
5. **Update user status** (to 'delivering')
6. **Send notifications** (customer + chef)

**Transaction Handling**: Uses database transaction for atomic request acceptance + active delivery creation

---

### **DeliveryAssignmentService Methods**

#### **Method 1: findEligibleRider**

**Signature**:
```typescript
async findEligibleRider(order: Order): Promise<RiderProfile | null>
```

**Purpose**: Find best available rider for order using FIFO + distance logic

**Eligibility Criteria**:
1. `isOnline = true` (rider available)
2. `isActive = true` (rider account active)
3. `isBusy = false` (not at max capacity)
4. `city = order.city` (same city as order)
5. `activeOrderCount < maxConcurrentDeliveries` (capacity check)

**Sorting Logic**:
```sql
SELECT rider.*,
  (6371 * acos(
    cos(radians(:lat)) * cos(radians(rider.currentLat)) *
    cos(radians(rider.currentLng) - radians(:lng)) +
    sin(radians(:lat)) * sin(radians(rider.currentLat))
  )) AS distance
FROM rider_profiles rider
WHERE rider.isOnline = true
  AND rider.isActive = true
  AND rider.isBusy = false
  AND LOWER(rider.city) = LOWER(:city)
  AND rider.activeOrderCount < rider.maxConcurrentDeliveries
ORDER BY distance ASC,
         rider.activeOrderCount ASC,
         COALESCE(rider.lastAssignedAt, rider.createdAt) ASC
LIMIT 1;
```

**Priority**:
1. **Distance** (closest rider first)
2. **Active order count** (least busy rider)
3. **Last assigned time** (fair distribution)

---

#### **Method 2: assignRiderToOrder**

**Signature**:
```typescript
async assignRiderToOrder(
  orderId: string,
  riderId: string,
  retryCount: number = 0
): Promise<{ success: boolean; message: string }>
```

**Purpose**: Assign specific rider to order with timeout handling

**Steps**:
1. **Lock order row** (FOR UPDATE transaction)
2. **Create delivery request** (with expiry timestamp)
3. **Schedule timeout check** (Bull queue delayed job)
4. **Send notification to rider**
5. **Return assignment confirmation**

**Timeout Handling**:
```typescript
// Schedule timeout check (30 seconds in production)
await assignmentQueue.add(
  'check-timeout',
  {
    orderId,
    riderId,
    assignedAt: new Date(),
    retryCount,
  },
  {
    delay: envConfig.delivery.assignmentTimeoutSec * 1000,
  }
);
```

---

### **RiderBusyService Methods**

#### **Method: getBusyStatus**

**Signature**:
```typescript
async getBusyStatus(userId: string): Promise<{
  isBusy: boolean;
  activeDeliveryCount: number;
  maxConcurrentDeliveries: number;
  reason: string | null;
}>
```

**Purpose**: Calculate if rider is at max capacity

**Implementation**:
```typescript
const profile = await riderProfileRepository.findOne({
  where: { userId },
});

if (!profile) return null;

const isBusy = profile.activeOrderCount >= profile.maxConcurrentDeliveries;

return {
  isBusy,
  activeDeliveryCount: profile.activeOrderCount,
  maxConcurrentDeliveries: profile.maxConcurrentDeliveries,
  reason: isBusy ? 'At maximum capacity' : null,
};
```

**Used By**:
- Assignment service (filter busy riders)
- GET /delivery/busy-status endpoint

---

## 💾 **Database Schema**

### **DeliveryRequest Entity**

**Table**: `delivery_requests`

**Purpose**: Incoming delivery assignments awaiting rider acceptance

**Columns**:
```sql
CREATE TABLE delivery_requests (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  
  -- References
  order_id UUID NOT NULL REFERENCES orders(id),
  rider_id UUID REFERENCES users(id),
  chef_id UUID NOT NULL,
  
  -- Location data
  chef_name VARCHAR(200) NOT NULL,
  chef_location JSONB NOT NULL, -- { latitude, longitude, address, landmark }
  customer_location JSONB NOT NULL,
  distance_to_pickup_km DECIMAL(10, 2) NOT NULL,
  
  -- Payout & timing
  estimated_payout_paise INT NOT NULL,
  estimated_delivery_minutes INT NOT NULL,
  expires_at TIMESTAMPTZ NOT NULL, -- Request expiry (typically 30 seconds)
  
  -- Status
  status VARCHAR(20) NOT NULL DEFAULT 'pending',
  rejection_reason VARCHAR(20), -- 'too_far', 'busy', 'low_payout', 'other'
  rejection_note TEXT,
  
  -- Timestamps
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  
  -- Indexes
  INDEX idx_delivery_request_status_expires (status, expires_at),
  INDEX idx_delivery_request_order (order_id),
  INDEX idx_delivery_request_rider (rider_id)
);
```

**Enums**:
```typescript
enum DeliveryRequestStatus {
  PENDING = 'pending',
  ACCEPTED = 'accepted',
  REJECTED = 'rejected',
  EXPIRED = 'expired',
  CANCELLED = 'cancelled',
}
```

**Lifecycle**:
1. **Created**: When order marked READY
2. **Accepted**: Rider accepts request
3. **Rejected**: Rider rejects with reason
4. **Expired**: Request not accepted within timeout (30s)
5. **Cancelled**: Order cancelled before acceptance

---

### **ActiveDelivery Entity**

**Table**: `active_deliveries`

**Purpose**: Track ongoing delivery from assignment to completion

**Columns**:
```sql
CREATE TABLE active_deliveries (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  
  -- References (unique constraint on order_id)
  order_id UUID NOT NULL UNIQUE REFERENCES orders(id),
  rider_id UUID NOT NULL REFERENCES users(id),
  chef_id UUID NOT NULL,
  customer_id UUID NOT NULL,
  
  -- Chef details
  chef_name VARCHAR(200) NOT NULL,
  chef_phone VARCHAR(15),
  chef_location JSONB NOT NULL,
  
  -- Customer details
  customer_name VARCHAR(200) NOT NULL,
  customer_phone VARCHAR(15),
  customer_location JSONB NOT NULL,
  
  -- Order info
  order_items JSONB NOT NULL, -- Array of item names ['Butter Chicken', 'Naan']
  order_value_paise INT NOT NULL,
  delivery_instructions TEXT,
  
  -- Delivery state
  state VARCHAR(30) NOT NULL DEFAULT 'navigate_to_chef',
  estimated_payout_paise INT NOT NULL,
  actual_payout_paise INT,
  delivery_proof VARCHAR(500), -- Photo URL
  
  -- Timestamps
  assigned_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  picked_up_at TIMESTAMPTZ,
  delivered_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  
  -- Indexes
  INDEX idx_active_delivery_rider_state (rider_id, state),
  INDEX idx_active_delivery_order (order_id)
);
```

**Enums**:
```typescript
enum ActiveDeliveryState {
  NAVIGATE_TO_CHEF = 'navigate_to_chef',
  PICKED_UP = 'picked_up',
  EN_ROUTE = 'en_route',
  DELIVERED = 'delivered',
}
```

**State Progression**:
```
navigate_to_chef → picked_up → en_route → delivered
```

---

### **RiderProfile Entity (Partial)**

**Table**: `rider_profiles`

**Purpose**: Rider metadata for assignment algorithm

**Relevant Columns**:
```sql
CREATE TABLE rider_profiles (
  id UUID PRIMARY KEY,
  user_id UUID NOT NULL REFERENCES users(id),
  
  -- Availability
  is_online BOOLEAN NOT NULL DEFAULT false,
  is_active BOOLEAN NOT NULL DEFAULT true,
  is_busy BOOLEAN NOT NULL DEFAULT false,
  
  -- Location
  city VARCHAR(100) NOT NULL,
  current_lat DECIMAL(10, 8),
  current_lng DECIMAL(11, 8),
  
  -- Capacity
  active_order_count INT NOT NULL DEFAULT 0,
  max_concurrent_deliveries INT NOT NULL DEFAULT 2,
  
  -- Performance
  last_assigned_at TIMESTAMPTZ,
  reputation_score DECIMAL(3, 2) DEFAULT 5.0,
  
  -- Timestamps
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

**Used By**:
- Assignment service (eligibility filtering)
- Busy service (capacity calculation)

---

## 📦 **DTOs**

### **UpdateDeliveryStatusDto**

```typescript
export class UpdateDeliveryStatusDto {
  @ApiProperty({ 
    enum: ['offline', 'online', 'delivering'],
    description: 'Rider delivery availability status'
  })
  @IsEnum(['offline', 'online', 'delivering'])
  @IsNotEmpty()
  status!: 'offline' | 'online' | 'delivering';
}
```

---

### **AcceptDeliveryDto**

```typescript
export class AcceptDeliveryDto {
  @ApiProperty({ description: 'ID of the delivery request to accept' })
  @IsUUID()
  @IsNotEmpty()
  deliveryRequestId!: string;

  @ApiPropertyOptional({ description: 'Rider current location at acceptance', type: DeliveryLocationDto })
  @IsObject()
  @IsOptional()
  currentLocation?: DeliveryLocationDto;
}

export class DeliveryLocationDto {
  @ApiProperty({ description: 'Latitude coordinate', example: 12.9716 })
  @IsNumber()
  @Min(-90)
  @Max(90)
  latitude!: number;

  @ApiProperty({ description: 'Longitude coordinate', example: 77.5946 })
  @IsNumber()
  @Min(-180)
  @Max(180)
  longitude!: number;

  @ApiPropertyOptional({ description: 'Human-readable address' })
  @IsString()
  @IsOptional()
  address?: string;

  @ApiPropertyOptional({ description: 'Landmark for easier location' })
  @IsString()
  @IsOptional()
  landmark?: string;
}
```

---

### **RejectDeliveryDto**

```typescript
export class RejectDeliveryDto {
  @ApiProperty({ description: 'ID of the delivery request to reject' })
  @IsUUID()
  @IsNotEmpty()
  deliveryRequestId!: string;

  @ApiProperty({ 
    enum: ['too_far', 'busy', 'low_payout', 'other'],
    description: 'Reason for rejection'
  })
  @IsEnum(['too_far', 'busy', 'low_payout', 'other'])
  @IsNotEmpty()
  reason!: 'too_far' | 'busy' | 'low_payout' | 'other';

  @ApiPropertyOptional({ description: 'Optional note explaining rejection' })
  @IsString()
  @IsOptional()
  note?: string;
}
```

---

### **UpdateDeliveryStateDto**

```typescript
export class UpdateDeliveryStateDto {
  @ApiProperty({ description: 'ID of the active delivery' })
  @IsUUID()
  @IsNotEmpty()
  deliveryId!: string;

  @ApiProperty({ 
    enum: ['navigate_to_chef', 'picked_up', 'en_route', 'delivered'],
    description: 'New delivery state'
  })
  @IsEnum(['navigate_to_chef', 'picked_up', 'en_route', 'delivered'])
  @IsNotEmpty()
  state!: 'navigate_to_chef' | 'picked_up' | 'en_route' | 'delivered';

  @ApiPropertyOptional({ description: 'Rider current location', type: DeliveryLocationDto })
  @IsObject()
  @IsOptional()
  currentLocation?: DeliveryLocationDto;
}
```

---

### **CompleteDeliveryDto**

```typescript
export class CompleteDeliveryDto {
  @ApiProperty({ description: 'ID of the delivery to complete' })
  @IsUUID()
  @IsNotEmpty()
  deliveryId!: string;

  @ApiPropertyOptional({ description: 'URL of delivery proof photo (optional)' })
  @IsString()
  @IsOptional()
  deliveryProof?: string;

  @ApiPropertyOptional({ description: 'Final delivery location', type: DeliveryLocationDto })
  @IsObject()
  @IsOptional()
  finalLocation?: DeliveryLocationDto;
}
```

---

## 🔗 **Domain Rules**

**Package**: `@chefooz-app/domain`

**File**: `libs/domain/src/delivery/delivery.rules.ts`

### **Rule 1: canAutoAssignRider**

```typescript
export function canAutoAssignRider(order: DeliveryOrderContext): boolean {
  return order.status === 'PAID' && !order.deliveryPartnerId;
}
```

**Purpose**: Check if order is eligible for automatic rider assignment

---

### **Rule 2: isRiderEligible**

```typescript
export function isRiderEligible(
  rider: RiderContext,
  envConfig: EnvironmentConfig
): { eligible: boolean; reason?: string }
```

**Checks**:
1. Rider status = 'online'
2. No active delivery
3. Reputation score ≥ minimum threshold (from environment config)

---

### **Rule 3: canAcceptDelivery**

```typescript
export function canAcceptDelivery(
  request: DeliveryRequestContext,
  rider: RiderContext,
  envConfig: EnvironmentConfig
): { allowed: boolean; reason?: string }
```

**Checks**:
1. Request not expired
2. Request status = 'PENDING'
3. Rider eligible (using `isRiderEligible`)

---

### **Rule 4: canTransitionActiveDeliveryState**

```typescript
export function canTransitionActiveDeliveryState(
  from: ActiveDeliveryState,
  to: ActiveDeliveryState
): boolean
```

**Valid Transitions**:
```typescript
const validTransitions: Record<ActiveDeliveryState, ActiveDeliveryState[]> = {
  ASSIGNED: ['HEADING_TO_PICKUP', 'PICKED_UP'],
  HEADING_TO_PICKUP: ['ARRIVED_AT_PICKUP', 'PICKED_UP'],
  ARRIVED_AT_PICKUP: ['PICKED_UP'],
  PICKED_UP: ['HEADING_TO_CUSTOMER', 'ARRIVED_AT_CUSTOMER'],
  ARRIVED_AT_CUSTOMER: ['DELIVERED'],
  HEADING_TO_CUSTOMER: ['ARRIVED_AT_CUSTOMER', 'DELIVERED'],
  DELIVERED: [], // Final state
};

return validTransitions[from]?.includes(to) || false;
```

---

## 🔌 **Integration Patterns**

### **Integration 1: Order Module**

**Purpose**: Auto-cancellation logic + order status updates

**Usage**:
```typescript
// Trigger auto-cancel
await orderService.autoCancelOrder({
  orderId,
  reason: 'RIDER_NOT_ACCEPTED',
});
```

**Circular Dependency**: Resolved using `forwardRef(() => OrderModule)`

---

### **Integration 2: Notification Module**

**Purpose**: Push notifications + email dispatch

**Events**:
```typescript
// Rider assigned
await notificationDispatcher.send(
  customerId,
  'delivery.assigned',
  { orderId, riderName }
);

// Rider picked up order
await notificationDispatcher.send(
  customerId,
  'delivery.picked_up',
  { orderId }
);

// Order delivered
await notificationDispatcher.send(
  customerId,
  'delivery.completed',
  { orderId }
);
```

---

### **Integration 3: RiderLocation Module**

**Purpose**: GPS tracking + location updates

**Usage**:
```typescript
// Get rider current location
const location = await riderLocationService.getLatestLocation(riderId);

// Update rider location (every 10 seconds while delivering)
await riderLocationService.updateLocation(riderId, {
  latitude: 12.9716,
  longitude: 77.5946,
  timestamp: new Date(),
});
```

---

## ⚙️ **Background Jobs**

### **Job 1: Assignment Timeout Check**

**Queue**: `delivery-assignment`  
**Job Name**: `check-timeout`  
**Trigger**: Delayed job (30 seconds after assignment in production)  
**Processor**: `AssignmentTimeoutProcessor`

**Purpose**: Check if rider accepted assignment within timeout

**Logic**:
```typescript
@Process('check-timeout')
async handleAssignmentTimeout(job: Job<AssignmentTimeoutJob>) {
  const { orderId, riderId, assignedAt, retryCount } = job.data;
  
  // 1. Validate rider didn't accept
  if (order.deliveryAcceptedAt) {
    logger.log('Rider accepted, skipping timeout');
    return;
  }
  
  // 2. Check if assignment timed out (domain rule)
  const timedOut = hasAssignmentTimedOut(assignedAt, order.deliveryAcceptedAt);
  if (!timedOut) return;
  
  // 3. Check retry eligibility (domain rule)
  const retryDecision = shouldRetryAssignment(retryCount, envConfig);
  
  if (retryDecision.allowed) {
    // Retry with next rider
    await deliveryAssignmentService.retryAssignment(orderId, retryCount + 1);
  } else {
    // Max retries exceeded → Auto-cancel order
    await orderService.autoCancelOrder({
      orderId,
      reason: 'RIDER_NOT_ACCEPTED',
    });
  }
}
```

**Retry Configuration**:

| Environment | Timeout | Max Retries | Total Max Wait |
|-------------|---------|-------------|----------------|
| **Development** | 60s | 10 | 10 minutes |
| **Production** | 30s | 5 | 2.5 minutes |

---

### **Job 2: Pickup Timeout Monitor (Cron)**

**Schedule**: Every minute  
**Service**: `DeliveryTimeoutCron`  
**Method**: `checkPickupTimeouts()`

**Purpose**: Auto-cancel orders where rider accepted but didn't pick up food

**Logic**:
```typescript
@Cron(CronExpression.EVERY_MINUTE)
async checkPickupTimeouts() {
  // Find orders with accepted delivery but not picked up
  const ordersAtRisk = await orderRepository.find({
    where: {
      status: Not(In([OrderStatus.CANCELLED, OrderStatus.DELIVERED])),
      deliveryAcceptedAt: Not(IsNull()),
      pickedUpAt: IsNull(),
    },
  });
  
  // Check each order against policy
  for (const order of ordersAtRisk) {
    const pickupCheck = shouldAutoCancelForPickupDelay({
      acceptedAt: order.deliveryAcceptedAt,
      currentTime: new Date(),
      config: envConfig,
    });
    
    if (!pickupCheck.allowed) {
      // Timeout violation → Auto-cancel
      await orderService.autoCancelOrder({
        orderId: order.id,
        reason: 'PICKUP_TIMEOUT',
      });
    }
  }
}
```

**Timeout Configuration**:

| Environment | Pickup Timeout |
|-------------|----------------|
| **Development** | 30 minutes |
| **Production** | 20 minutes |

---

### **Job 3: Delivery Timeout Monitor (Cron)**

**Schedule**: Every minute  
**Service**: `DeliveryTimeoutCron`  
**Method**: `checkDeliveryTimeouts()`

**Purpose**: Auto-cancel orders where rider picked up but didn't deliver

**Logic**:
```typescript
@Cron(CronExpression.EVERY_MINUTE)
async checkDeliveryTimeouts() {
  // Find orders out for delivery but not yet delivered
  const ordersAtRisk = await orderRepository.find({
    where: {
      status: OrderStatus.OUT_FOR_DELIVERY,
      pickedUpAt: Not(IsNull()),
      deliveredAt: IsNull(),
    },
  });
  
  // Check each order against policy
  for (const order of ordersAtRisk) {
    const etaMinutes = order.estimatedDeliveryMinutes || 30;
    
    const deliveryCheck = shouldAutoCancelForDeliveryDelay({
      pickedUpAt: order.pickedUpAt,
      etaMinutes,
      config: envConfig,
    });
    
    if (!deliveryCheck.allowed) {
      // Timeout violation → Auto-cancel
      await orderService.autoCancelOrder({
        orderId: order.id,
        reason: 'DELIVERY_TIMEOUT',
      });
    }
  }
}
```

**Timeout Configuration**:

| Environment | Delivery Timeout | ETA Buffer |
|-------------|------------------|------------|
| **Development** | 60 minutes | +15 min |
| **Production** | 45 minutes | +15 min |

**Effective Timeout**: `max(ETA + 15 minutes, deliveryTimeoutMin)`

---

## ⚠️ **Error Handling**

### **Error Codes**

| Error Code | HTTP Status | Description |
|------------|-------------|-------------|
| `ACTIVE_DELIVERY_EXISTS` | 400 | Rider already has active delivery |
| `REQUEST_EXPIRED` | 400 | Delivery request expired |
| `REQUEST_ALREADY_PROCESSED` | 400 | Request already accepted/rejected |
| `INVALID_STATE_TRANSITION` | 400 | State transition not allowed |
| `RIDER_NOT_ONLINE` | 400 | Rider must be online to accept |
| `DELIVERY_NOT_FOUND` | 404 | Active delivery not found |
| `REQUEST_NOT_FOUND` | 404 | Delivery request not found |
| `UNAUTHORIZED_RIDER` | 403 | Delivery doesn't belong to rider |

### **Common Error Patterns**

**Cannot Go Offline**:
```typescript
if (dto.status === 'offline' && user.currentDeliveryId) {
  throw new BadRequestException('Cannot go offline with active delivery');
}
```

**Request Expired**:
```typescript
if (new Date() > request.expiresAt) {
  request.status = 'EXPIRED';
  await deliveryRequestRepository.save(request);
  throw new BadRequestException('Request has expired');
}
```

**Invalid State Transition**:
```typescript
if (!canTransitionActiveDeliveryState(currentState, newState)) {
  throw new BadRequestException(
    `Cannot transition from ${currentState} to ${newState}`
  );
}
```

---

## 🧪 **Testing Strategy**

### **Unit Tests**

**File**: `delivery.service.spec.ts`

**Test Cases**:
1. **updateDeliveryStatus**:
   - Updates status successfully
   - Rejects offline with active delivery
   - Handles invalid statuses

2. **acceptDeliveryRequest**:
   - Creates active delivery
   - Updates user status to 'delivering'
   - Validates rider eligibility
   - Handles expired requests

3. **rejectDeliveryRequest**:
   - Marks request as rejected
   - Logs rejection reason
   - Triggers re-assignment

4. **updateDeliveryState**:
   - Updates state successfully
   - Sets timestamps correctly
   - Validates state transitions
   - Sends notifications

5. **completeDelivery**:
   - Marks as delivered
   - Updates user back to online
   - Clears currentDeliveryId
   - Schedules payout

---

### **Integration Tests**

**File**: `delivery.e2e.spec.ts`

**Test Scenarios**:
1. Full assignment flow (order ready → rider assigned → accepted)
2. Timeout retry flow (no acceptance → retry with next rider)
3. Auto-cancel flow (max retries exceeded → order cancelled)
4. Pickup timeout flow (rider doesn't pick up → order cancelled)
5. Delivery completion flow (navigate → pickup → deliver)

---

**[DELIVERY_TECHNICAL_GUIDE_COMPLETE ✅]**

*For business overview, see `01_FEATURE_OVERVIEW.md`. For testing scenarios, see `03_QA_TEST_CASES.md`.*

---

**Document Version**: 1.0  
**Last Updated**: February 22, 2026  
**Module**: Delivery (Week 7 - Chef Fulfillment)  
**Status**: ✅ Complete
