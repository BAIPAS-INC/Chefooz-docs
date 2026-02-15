# 💳 Payment Module — Technical Guide

**Status**: ✅ COMPLETE  
**Last Updated**: 2025-02-15  
**Module**: Payment (Razorpay Integration)  
**Dependencies**: Order, Notification, Audit-Events  

---

## 📋 Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Database Schema](#database-schema)
3. [API Endpoints](#api-endpoints)
4. [Service Layer](#service-layer)
5. [Frontend Integration](#frontend-integration)
6. [Shared Libraries](#shared-libraries)
7. [Error Handling](#error-handling)
8. [Security Implementation](#security-implementation)
9. [Performance Optimizations](#performance-optimizations)
10. [Testing Guidelines](#testing-guidelines)
11. [Deployment Configuration](#deployment-configuration)

---

## 🏗️ Architecture Overview

### System Context Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                         PAYMENT MODULE                           │
│                                                                   │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐        │
│  │   Payment    │   │   Payment    │   │   Webhook    │        │
│  │  Controller  │───│   Service    │───│   Handler    │        │
│  └──────────────┘   └──────────────┘   └──────────────┘        │
│         │                   │                   │                │
│         │                   │                   │                │
│         ▼                   ▼                   ▼                │
│  ┌──────────────────────────────────────────────────────┐       │
│  │         PaymentIntent Repository (PostgreSQL)         │       │
│  └──────────────────────────────────────────────────────┘       │
└─────────────────────────────────────────────────────────────────┘
           │                   │                   │
           │                   │                   │
           ▼                   ▼                   ▼
    ┌────────────┐     ┌────────────┐     ┌────────────┐
    │ Order      │     │ Razorpay   │     │ Notification│
    │ Service    │     │ API        │     │ Service     │
    └────────────┘     └────────────┘     └────────────┘
```

### Component Architecture

**Backend (NestJS):**
```
apps/chefooz-apis/src/modules/payment/
├── payment.controller.ts       → REST endpoints
├── payment.service.ts          → Business logic
├── payment.module.ts           → DI configuration
├── dto/
│   ├── create-razorpay-order.dto.ts
│   ├── verify-payment.dto.ts
│   └── webhook.dto.ts
└── interfaces/
    └── razorpay.types.ts       → Razorpay API types
```

**Frontend (React Native/Expo):**
```
libs/api-client/src/
├── payment.client.ts           → Axios client
└── usePayment.ts               → React Query hooks

apps/chefooz-app/src/app/(main)/
├── checkout/                   → Checkout screen
├── payment-processing/         → Payment flow
├── payment-success/            → Success screen
└── payment-failed/             → Failure screen
```

**Database:**
```
apps/chefooz-apis/src/database/entities/
└── payment-intent.entity.ts    → Payment tracking
```

### Technology Stack

| Layer | Technology | Purpose |
|-------|------------|---------|
| **Backend Framework** | NestJS 10+ | REST API, dependency injection |
| **Payment Gateway** | Razorpay | Payment processing, UPI Intent |
| **Database** | PostgreSQL 15+ | Payment intent storage |
| **HTTP Client** | Axios | Razorpay API calls |
| **Crypto** | Node.js crypto | HMAC SHA256 signature verification |
| **Frontend State** | React Query | Payment status polling |
| **Frontend SDK** | react-native-razorpay | Native payment UI |

---

## 🗄️ Database Schema

### PaymentIntent Entity

**Table Name:** `payment_intents`

**Purpose:** Track all payment attempts and their lifecycle for idempotency, reconciliation, and audit.

#### Column Definitions

```typescript
@Entity('payment_intents')
@Index(['userId'])
@Index(['razorpayOrderId'])
@Index(['status'])
@Index(['createdAt'])
export class PaymentIntent {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ type: 'uuid' })
  userId: string;

  @Column({ type: 'varchar', length: 100, unique: true })
  razorpayOrderId: string;

  @Column({ type: 'varchar', length: 100, nullable: true })
  razorpayPaymentId: string | null;

  @Column({ type: 'integer' })
  amount: number; // in paise

  @Column({ type: 'varchar', length: 3, default: 'INR' })
  currency: string;

  @Column({
    type: 'enum',
    enum: ['created', 'paid', 'failed', 'refunded'],
    default: 'created',
  })
  status: 'created' | 'paid' | 'failed' | 'refunded';

  @Column({ type: 'uuid', nullable: true })
  orderId: string | null;

  @Column({ type: 'jsonb', default: {} })
  metadata: Record<string, any>;

  @Column({ type: 'jsonb', nullable: true })
  webhookPayload: Record<string, any> | null;

  @Column({ type: 'text', nullable: true })
  failureReason: string | null;

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;
}
```

#### Field Descriptions

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| `id` | UUID | Primary Key | Payment intent unique identifier |
| `userId` | UUID | Foreign Key, NOT NULL, Indexed | User who initiated payment |
| `razorpayOrderId` | VARCHAR(100) | UNIQUE, NOT NULL, Indexed | Razorpay order ID (from API) |
| `razorpayPaymentId` | VARCHAR(100) | Nullable, Indexed | Razorpay payment ID (after payment) |
| `amount` | INTEGER | NOT NULL | Amount in paise (₹1 = 100 paise) |
| `currency` | VARCHAR(3) | NOT NULL, Default 'INR' | ISO currency code |
| `status` | ENUM | NOT NULL, Default 'created', Indexed | Payment lifecycle state |
| `orderId` | UUID | Nullable, Foreign Key | Linked order after payment success |
| `metadata` | JSONB | NOT NULL, Default {} | Cart snapshot, address, promo code |
| `webhookPayload` | JSONB | Nullable | Razorpay webhook event data |
| `failureReason` | TEXT | Nullable | Error message if payment failed |
| `createdAt` | TIMESTAMP | NOT NULL, Indexed | Payment intent creation time |
| `updatedAt` | TIMESTAMP | NOT NULL | Last status update time |

#### Metadata JSONB Structure

```typescript
interface PaymentIntentMetadata {
  cartSnapshot: Array<{
    productId: string;
    quantity: number;
    unitPrice: number;
    notes?: string;
  }>;
  deliveryAddress: {
    addressId?: string;
    line1: string;
    line2?: string;
    city: string;
    state: string;
    postalCode: string;
    latitude: number;
    longitude: number;
  };
  deliveryInstructions?: string;
  promoCode?: string;
  receipt: string; // order_{userId}_{timestamp}
}
```

**Example:**
```json
{
  "cartSnapshot": [
    {
      "productId": "550e8400-e29b-41d4-a716-446655440000",
      "quantity": 2,
      "unitPrice": 25000,
      "notes": "Extra spicy"
    }
  ],
  "deliveryAddress": {
    "line1": "123 MG Road",
    "city": "Mumbai",
    "state": "Maharashtra",
    "postalCode": "400001",
    "latitude": 19.0760,
    "longitude": 72.8777
  },
  "receipt": "order_550e8400_1708012800000"
}
```

#### Indexes

```sql
CREATE INDEX idx_payment_intents_userId ON payment_intents(userId);
CREATE INDEX idx_payment_intents_razorpayOrderId ON payment_intents(razorpayOrderId);
CREATE INDEX idx_payment_intents_status ON payment_intents(status);
CREATE INDEX idx_payment_intents_createdAt ON payment_intents(createdAt);
CREATE UNIQUE INDEX idx_payment_intents_razorpayOrderId_unique ON payment_intents(razorpayOrderId);
```

**Index Purposes:**
- `userId`: User payment history queries
- `razorpayOrderId`: Webhook lookup (unique prevents duplicates)
- `status`: Filter by payment state (admin dashboard)
- `createdAt`: Chronological ordering, date range filters

---

## 🌐 API Endpoints

### 1. Create Razorpay Order

**Endpoint:** `POST /api/v1/payment/create-order`

**Purpose:** Initiates payment by creating a Razorpay order and saving payment intent.

**Authentication:** Required (JWT Bearer token)

**Rate Limiting:** 20 requests/min per user

#### Request

**Headers:**
```
Authorization: Bearer <jwt_token>
Content-Type: application/json
```

**Body:**
```typescript
interface CreateRazorpayOrderRequest {
  items: Array<{
    productId: string;
    quantity: number;
    unitPrice: number;
    notes?: string;
  }>;
  deliveryAddress: {
    addressId?: string;
    line1: string;
    line2?: string;
    city: string;
    state: string;
    postalCode: string;
    latitude: number;
    longitude: number;
  };
  amount: number; // Total in paise
  deliveryInstructions?: string;
  promoCode?: string;
}
```

**Example:**
```bash
curl -X POST https://api.chefooz.com/api/v1/payment/create-order \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." \
  -H "Content-Type: application/json" \
  -d '{
    "items": [
      {
        "productId": "550e8400-e29b-41d4-a716-446655440000",
        "quantity": 2,
        "unitPrice": 25000
      }
    ],
    "deliveryAddress": {
      "line1": "123 MG Road",
      "city": "Mumbai",
      "state": "Maharashtra",
      "postalCode": "400001",
      "latitude": 19.0760,
      "longitude": 72.8777
    },
    "amount": 50000
  }'
```

#### Response

**Success (201):**
```json
{
  "success": true,
  "message": "Razorpay order created successfully",
  "data": {
    "paymentIntentId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
    "orderId": "order_NEHcLbZ5YT4R2a",
    "amount": 50000,
    "currency": "INR",
    "keyId": "rzp_test_1DP5mmOlF5G5ag"
  }
}
```

**Error (400):**
```json
{
  "success": false,
  "message": "Amount mismatch with cart total",
  "errorCode": "AMOUNT_MISMATCH"
}
```

#### Business Logic

1. **Validate amount matches cart total:**
   ```typescript
   const calculatedAmount = dto.items.reduce(
     (sum, item) => sum + item.unitPrice * item.quantity,
     0
   );
   if (calculatedAmount !== dto.amount) {
     throw new BadRequestException('Amount mismatch');
   }
   ```

2. **Generate unique receipt ID:**
   ```typescript
   const receipt = `order_${userId}_${Date.now()}`;
   ```

3. **Create Razorpay order:**
   ```typescript
   const razorpayOrder = await axios.post(
     'https://api.razorpay.com/v1/orders',
     {
       amount: dto.amount,
       currency: 'INR',
       receipt,
       notes: { userId, itemCount: dto.items.length.toString() }
     },
     {
       auth: {
         username: razorpayKeyId,
         password: razorpayKeySecret
       }
     }
   );
   ```

4. **Save payment intent:**
   ```typescript
   const paymentIntent = await paymentIntentRepository.save({
     userId,
     razorpayOrderId: razorpayOrder.data.id,
     amount: dto.amount,
     currency: 'INR',
     status: 'created',
     metadata: {
       cartSnapshot: dto.items,
       deliveryAddress: dto.deliveryAddress,
       receipt
     }
   });
   ```

---

### 2. Verify Payment

**Endpoint:** `POST /api/v1/payment/verify`

**Purpose:** Verifies Razorpay payment signature and creates order after successful payment.

**Authentication:** Required (JWT Bearer token)

**Rate Limiting:** 10 requests/min per user

#### Request

**Headers:**
```
Authorization: Bearer <jwt_token>
Content-Type: application/json
```

**Body:**
```typescript
interface VerifyPaymentRequest {
  razorpay_order_id: string;
  razorpay_payment_id: string;
  razorpay_signature: string;
}
```

**Example:**
```bash
curl -X POST https://api.chefooz.com/api/v1/payment/verify \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." \
  -H "Content-Type: application/json" \
  -d '{
    "razorpay_order_id": "order_NEHcLbZ5YT4R2a",
    "razorpay_payment_id": "pay_NEHdCU9bZ5YT4R2a",
    "razorpay_signature": "a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6q7r8s9t0"
  }'
```

#### Response

**Success (200):**
```json
{
  "success": true,
  "message": "Payment verified successfully",
  "data": {
    "paymentIntentId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
    "orderId": "550e8400-e29b-41d4-a716-446655440000",
    "razorpayPaymentId": "pay_NEHdCU9bZ5YT4R2a"
  }
}
```

**Error (401):**
```json
{
  "success": false,
  "message": "Invalid payment signature",
  "errorCode": "INVALID_SIGNATURE"
}
```

#### Business Logic

1. **Verify signature:**
   ```typescript
   const generatedSignature = crypto
     .createHmac('sha256', razorpayKeySecret)
     .update(`${razorpay_order_id}|${razorpay_payment_id}`)
     .digest('hex');
   
   if (generatedSignature !== razorpay_signature) {
     throw new UnauthorizedException('Invalid signature');
   }
   ```

2. **Check idempotency:**
   ```typescript
   const paymentIntent = await paymentIntentRepository.findOne({
     where: { razorpayOrderId: razorpay_order_id, userId }
   });
   
   if (paymentIntent.status === 'paid') {
     return { success: true, message: 'Already processed' };
   }
   ```

3. **Update payment intent:**
   ```typescript
   paymentIntent.status = 'paid';
   paymentIntent.razorpayPaymentId = razorpay_payment_id;
   await paymentIntentRepository.save(paymentIntent);
   ```

4. **Create order:**
   ```typescript
   const order = await orderService.placePrepaidOrder(paymentIntent);
   paymentIntent.orderId = order.id;
   await paymentIntentRepository.save(paymentIntent);
   ```

5. **Send notification:**
   ```typescript
   await notificationDispatcher.send(userId, 'payment.success', {
     orderId: order.id,
     amount: (paymentIntent.amount / 100).toFixed(2)
   });
   ```

---

### 3. Razorpay Webhook

**Endpoint:** `POST /api/v1/payment/webhook`

**Purpose:** Receives asynchronous payment events from Razorpay.

**Authentication:** None (signature verified instead)

**Rate Limiting:** None (Razorpay controlled)

#### Request

**Headers:**
```
x-razorpay-signature: <hmac_sha256_signature>
Content-Type: application/json
```

**Body:**
```typescript
interface RazorpayWebhookEvent {
  entity: string;
  account_id: string;
  event: 'payment.captured' | 'payment.failed' | 'order.paid';
  contains: string[];
  payload: {
    payment: {
      entity: {
        id: string;
        amount: number;
        currency: string;
        status: string;
        order_id: string;
        method: string;
        error_description?: string;
      };
    };
    order?: {
      entity: {
        id: string;
        amount: number;
        status: string;
      };
    };
  };
  created_at: number;
}
```

**Example:**
```json
{
  "entity": "event",
  "event": "payment.captured",
  "payload": {
    "payment": {
      "entity": {
        "id": "pay_NEHdCU9bZ5YT4R2a",
        "order_id": "order_NEHcLbZ5YT4R2a",
        "amount": 50000,
        "currency": "INR",
        "status": "captured",
        "method": "upi"
      }
    }
  },
  "created_at": 1708012800
}
```

#### Response

**Success (200):**
```json
{
  "success": true,
  "message": "Webhook processed"
}
```

**Error (401):**
```json
{
  "success": false,
  "message": "Invalid webhook signature",
  "errorCode": "INVALID_WEBHOOK_SIGNATURE"
}
```

#### Business Logic

1. **Verify webhook signature:**
   ```typescript
   const generatedSignature = crypto
     .createHmac('sha256', razorpayWebhookSecret)
     .update(JSON.stringify(body))
     .digest('hex');
   
   if (generatedSignature !== signature) {
     throw new UnauthorizedException('Invalid signature');
   }
   ```

2. **Process event by type:**

**payment.captured:**
```typescript
const payment = webhook.payload.payment.entity;
const paymentIntent = await paymentIntentRepository.findOne({
  where: { razorpayOrderId: payment.order_id }
});

if (paymentIntent && paymentIntent.status !== 'paid') {
  paymentIntent.status = 'paid';
  paymentIntent.razorpayPaymentId = payment.id;
  paymentIntent.webhookPayload = webhook;
  await paymentIntentRepository.save(paymentIntent);
  
  // Create order if not already created
  if (!paymentIntent.orderId) {
    const order = await orderService.placePrepaidOrder(paymentIntent);
    paymentIntent.orderId = order.id;
    await paymentIntentRepository.save(paymentIntent);
  }
}
```

**payment.failed:**
```typescript
const payment = webhook.payload.payment.entity;
const paymentIntent = await paymentIntentRepository.findOne({
  where: { razorpayOrderId: payment.order_id }
});

if (paymentIntent) {
  paymentIntent.status = 'failed';
  paymentIntent.failureReason = payment.error_description;
  paymentIntent.webhookPayload = webhook;
  await paymentIntentRepository.save(paymentIntent);
  
  await notificationDispatcher.send(
    paymentIntent.userId,
    'payment.failed',
    { reason: payment.error_description }
  );
}
```

---

### 4. Get Payment Status

**Endpoint:** `GET /api/v1/payment/:paymentIntentId/status`

**Purpose:** Retrieves current status of a payment intent.

**Authentication:** Required (JWT Bearer token)

**Rate Limiting:** 60 requests/min per user (allows polling)

#### Request

**Headers:**
```
Authorization: Bearer <jwt_token>
```

**URL Parameters:**
- `paymentIntentId` (UUID): Payment intent ID

**Example:**
```bash
curl -X GET https://api.chefooz.com/api/v1/payment/7c9e6679-7425-40de-944b-e07fc1f90ae7/status \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

#### Response

**Success (200):**
```json
{
  "success": true,
  "data": {
    "id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
    "status": "paid",
    "amount": 50000,
    "razorpayOrderId": "order_NEHcLbZ5YT4R2a",
    "razorpayPaymentId": "pay_NEHdCU9bZ5YT4R2a",
    "orderId": "550e8400-e29b-41d4-a716-446655440000",
    "createdAt": "2026-02-15T10:30:00Z",
    "updatedAt": "2026-02-15T10:32:15Z"
  }
}
```

**Error (400):**
```json
{
  "success": false,
  "message": "Payment intent not found",
  "errorCode": "PAYMENT_INTENT_NOT_FOUND"
}
```

#### Business Logic

```typescript
const paymentIntent = await paymentIntentRepository.findOne({
  where: { id: paymentIntentId, userId }
});

if (!paymentIntent) {
  throw new BadRequestException('Payment intent not found');
}

return {
  success: true,
  data: {
    id: paymentIntent.id,
    status: paymentIntent.status,
    amount: paymentIntent.amount,
    razorpayOrderId: paymentIntent.razorpayOrderId,
    razorpayPaymentId: paymentIntent.razorpayPaymentId,
    orderId: paymentIntent.orderId,
    createdAt: paymentIntent.createdAt,
    updatedAt: paymentIntent.updatedAt
  }
};
```

---

## 🔧 Service Layer

### PaymentService Methods

**File:** `apps/chefooz-apis/src/modules/payment/payment.service.ts`

#### 1. createRazorpayOrder()

**Purpose:** Creates Razorpay order and saves payment intent.

**Signature:**
```typescript
async createRazorpayOrder(
  userId: string,
  dto: CreateRazorpayOrderDto
): Promise<{
  paymentIntentId: string;
  orderId: string;
  amount: number;
  currency: string;
  keyId: string;
}>
```

**Algorithm:**
1. Validate amount matches cart total
2. Generate unique receipt ID
3. Call Razorpay Orders API
4. Save payment intent with status='created'
5. Return order details for SDK

**Error Handling:**
- Throws `BadRequestException` if amount mismatch
- Throws `BadRequestException` if Razorpay API fails
- Logs all errors with payment intent ID

---

#### 2. verifyRazorpayPayment()

**Purpose:** Verifies payment signature and creates order.

**Signature:**
```typescript
async verifyRazorpayPayment(
  userId: string,
  dto: VerifyPaymentDto
): Promise<{
  success: boolean;
  message: string;
  data: {
    paymentIntentId: string;
    orderId: string;
    razorpayPaymentId: string;
  };
}>
```

**Algorithm:**
1. Verify HMAC signature
2. Find payment intent by razorpayOrderId
3. Check idempotency (already paid?)
4. Update payment intent status to 'paid'
5. Call OrderService.placePrepaidOrder()
6. Link order to payment intent
7. Send success notification

**Error Handling:**
- Throws `UnauthorizedException` if signature invalid
- Throws `BadRequestException` if payment intent not found
- Returns early if already processed (idempotent)

---

#### 3. handleWebhook()

**Purpose:** Processes Razorpay webhook events.

**Signature:**
```typescript
async handleWebhook(
  signature: string,
  body: RazorpayWebhookDto
): Promise<{ success: boolean; message: string }>
```

**Algorithm:**
1. Verify webhook signature
2. Switch on event type:
   - `payment.captured` → handlePaymentCaptured()
   - `payment.failed` → handlePaymentFailed()
   - `order.paid` → handleOrderPaid()
3. Return success response

**Error Handling:**
- Throws `UnauthorizedException` if signature invalid
- Logs unhandled event types
- Catches all errors to prevent webhook retry storms

---

#### 4. getPaymentStatus()

**Purpose:** Retrieves payment intent status.

**Signature:**
```typescript
async getPaymentStatus(
  userId: string,
  paymentIntentId: string
): Promise<{ success: boolean; data: PaymentStatusResponse }>
```

**Algorithm:**
1. Find payment intent by ID and userId
2. Return status and metadata

**Error Handling:**
- Throws `BadRequestException` if not found

---

#### 5. createRazorpayOrderAPI() (Private)

**Purpose:** Calls Razorpay Orders API.

**Signature:**
```typescript
private async createRazorpayOrderAPI(
  request: RazorpayOrderRequest
): Promise<RazorpayOrderResponse>
```

**Implementation:**
```typescript
const response = await axios.post<RazorpayOrderResponse>(
  'https://api.razorpay.com/v1/orders',
  request,
  {
    auth: {
      username: this.razorpayKeyId,
      password: this.razorpayKeySecret
    }
  }
);
return response.data;
```

---

#### 6. verifySignature() (Private)

**Purpose:** Verifies payment signature using HMAC SHA256.

**Signature:**
```typescript
private verifySignature(data: {
  razorpay_order_id: string;
  razorpay_payment_id: string;
  razorpay_signature: string;
}): boolean
```

**Implementation:**
```typescript
const generatedSignature = crypto
  .createHmac('sha256', this.razorpayKeySecret)
  .update(`${data.razorpay_order_id}|${data.razorpay_payment_id}`)
  .digest('hex');

return generatedSignature === data.razorpay_signature;
```

---

#### 7. verifyWebhookSignature() (Private)

**Purpose:** Verifies webhook signature.

**Signature:**
```typescript
private verifyWebhookSignature(signature: string, body: any): boolean
```

**Implementation:**
```typescript
const generatedSignature = crypto
  .createHmac('sha256', this.razorpayWebhookSecret)
  .update(JSON.stringify(body))
  .digest('hex');

return generatedSignature === signature;
```

---

## 📱 Frontend Integration

### React Query Hooks

**File:** `libs/api-client/src/lib/usePayment.ts`

#### 1. useCreatePaymentOrder()

**Purpose:** Mutation hook for creating Razorpay order.

**Usage:**
```typescript
const createOrder = useCreatePaymentOrder();

const handlePay = async () => {
  try {
    const result = await createOrder.mutateAsync({
      items: cartItems,
      deliveryAddress: selectedAddress,
      amount: totalAmount
    });
    
    openRazorpaySDK(result);
  } catch (error) {
    showError('Failed to initiate payment');
  }
};
```

**Returns:**
```typescript
{
  mutateAsync: (request: CreateRazorpayOrderRequest) => Promise<CreateRazorpayOrderResponse>;
  isLoading: boolean;
  error: Error | null;
}
```

---

#### 2. useVerifyPayment()

**Purpose:** Mutation hook for verifying payment signature.

**Usage:**
```typescript
const verifyPayment = useVerifyPayment();

const handlePaymentSuccess = async (response) => {
  try {
    const result = await verifyPayment.mutateAsync({
      razorpay_order_id: response.razorpay_order_id,
      razorpay_payment_id: response.razorpay_payment_id,
      razorpay_signature: response.razorpay_signature
    });
    
    navigateTo('payment-success', { orderId: result.data.orderId });
  } catch (error) {
    navigateTo('payment-failed', { reason: error.message });
  }
};
```

**Side Effects:**
- Invalidates `['orders']` query cache
- Invalidates `['my-orders']` query cache

---

#### 3. usePaymentStatus()

**Purpose:** Query hook for polling payment status.

**Usage:**
```typescript
const { data, isLoading } = usePaymentStatus(paymentIntentId);

useEffect(() => {
  if (data?.status === 'paid') {
    navigateTo('payment-success');
  } else if (data?.status === 'failed') {
    navigateTo('payment-failed');
  }
}, [data]);
```

**Features:**
- Auto-refetch every 5 seconds
- Enabled only when paymentIntentId provided
- Returns null if no data

---

### Razorpay SDK Integration

**File:** `apps/chefooz-app/src/utils/razorpay.ts`

**Implementation:**
```typescript
import RazorpayCheckout from 'react-native-razorpay';

export async function openRazorpay(options: {
  orderId: string;
  amount: number;
  keyId: string;
  userEmail: string;
  userPhone: string;
}): Promise<{
  razorpay_order_id: string;
  razorpay_payment_id: string;
  razorpay_signature: string;
}> {
  return new Promise((resolve, reject) => {
    RazorpayCheckout.open({
      key: options.keyId,
      amount: options.amount,
      currency: 'INR',
      name: 'Chefooz',
      description: 'Food Order Payment',
      order_id: options.orderId,
      prefill: {
        email: options.userEmail,
        contact: options.userPhone,
      },
      theme: {
        color: '#FF6B35',
      },
    })
    .then(resolve)
    .catch(reject);
  });
}
```

---

## 📦 Shared Libraries

### Types

**File:** `libs/types/src/lib/payment.types.ts`

```typescript
export interface CreateRazorpayOrderRequest {
  items: Array<{
    productId: string;
    quantity: number;
    unitPrice: number;
    notes?: string;
  }>;
  deliveryAddress: {
    addressId?: string;
    line1: string;
    line2?: string;
    city: string;
    state: string;
    postalCode: string;
    latitude: number;
    longitude: number;
  };
  amount: number;
  deliveryInstructions?: string;
  promoCode?: string;
}

export interface CreateRazorpayOrderResponse {
  paymentIntentId: string;
  orderId: string;
  amount: number;
  currency: string;
  keyId: string;
}

export interface VerifyPaymentRequest {
  razorpay_order_id: string;
  razorpay_payment_id: string;
  razorpay_signature: string;
}

export interface VerifyPaymentResponse {
  paymentIntentId: string;
  orderId: string;
  razorpayPaymentId: string;
}

export interface PaymentStatusResponse {
  id: string;
  status: 'created' | 'paid' | 'failed' | 'refunded';
  amount: number;
  razorpayOrderId: string;
  razorpayPaymentId: string | null;
  orderId: string | null;
  createdAt: string;
  updatedAt: string;
}
```

---

## ⚠️ Error Handling

### Error Codes

| Error Code | HTTP Status | Scenario | User Message |
|------------|-------------|----------|--------------|
| `AMOUNT_MISMATCH` | 400 | Calculated amount ≠ provided amount | "Payment amount mismatch. Please refresh and try again." |
| `INVALID_SIGNATURE` | 401 | Payment signature verification failed | "Payment verification failed. Your money is safe. Please retry." |
| `PAYMENT_INTENT_NOT_FOUND` | 400 | Payment intent ID invalid or unauthorized | "Payment session expired. Please start a new payment." |
| `RAZORPAY_API_ERROR` | 500 | Razorpay API down or error | "Payment service unavailable. Please try again in a moment." |
| `INVALID_WEBHOOK_SIGNATURE` | 401 | Webhook signature verification failed | N/A (internal error) |
| `ORDER_CREATION_FAILED` | 500 | Order service failed after payment | "Payment successful but order creation failed. Contact support." |

### Error Response Format

```json
{
  "success": false,
  "message": "Human-readable error message",
  "errorCode": "MACHINE_READABLE_CODE",
  "timestamp": "2026-02-15T10:30:00Z",
  "path": "/api/v1/payment/verify"
}
```

---

## 🔒 Security Implementation

### 1. Signature Verification

**Payment Verification:**
```typescript
const signature = crypto
  .createHmac('sha256', SECRET_KEY)
  .update(`${order_id}|${payment_id}`)
  .digest('hex');
```

**Webhook Verification:**
```typescript
const signature = crypto
  .createHmac('sha256', WEBHOOK_SECRET)
  .update(JSON.stringify(body))
  .digest('hex');
```

### 2. Access Control

- All endpoints except webhook require JWT authentication
- Payment intents accessible only by owner (userId check)
- Webhook endpoint signature-protected

### 3. Data Protection

- Razorpay secrets stored in environment variables
- Never expose secret keys in logs or responses
- Payment card data handled entirely by Razorpay (PCI-DSS)

### 4. Rate Limiting

```typescript
@UseGuards(ThrottlerGuard)
@Throttle({ default: { limit: 20, ttl: 60000 } }) // 20/min
```

### 5. Audit Logging

All payment events logged to `audit_events` table:
- Payment intent creation
- Signature verification attempts (success/failure)
- Webhook events
- Order linking

---

## ⚡ Performance Optimizations

### 1. Database Indexes

- `userId`, `razorpayOrderId`, `status`, `createdAt` all indexed
- Unique index on `razorpayOrderId` prevents duplicates

### 2. Webhook Idempotency

```typescript
if (paymentIntent.status === 'paid') {
  return; // Already processed
}
```

### 3. Frontend Polling

- Poll every 5 seconds (not every second)
- Stop polling after payment confirmed
- Timeout after 5 minutes

### 4. Razorpay API Caching

- Order creation response cached in payment intent
- No need to re-fetch from Razorpay

---

## 🧪 Testing Guidelines

### Unit Tests

**Test:** `payment.service.spec.ts`

```typescript
describe('PaymentService', () => {
  it('should verify valid signature', () => {
    const isValid = service['verifySignature']({
      razorpay_order_id: 'order_test',
      razorpay_payment_id: 'pay_test',
      razorpay_signature: 'valid_signature'
    });
    expect(isValid).toBe(true);
  });

  it('should reject invalid signature', () => {
    const isValid = service['verifySignature']({
      razorpay_order_id: 'order_test',
      razorpay_payment_id: 'pay_test',
      razorpay_signature: 'invalid_signature'
    });
    expect(isValid).toBe(false);
  });
});
```

### Integration Tests

**Test:** `payment.e2e.spec.ts`

```typescript
describe('Payment E2E', () => {
  it('should create order, verify payment, and create order', async () => {
    // Create payment intent
    const createResponse = await request(app.getHttpServer())
      .post('/api/v1/payment/create-order')
      .set('Authorization', `Bearer ${token}`)
      .send(createOrderDto)
      .expect(201);

    // Simulate Razorpay callback
    const verifyResponse = await request(app.getHttpServer())
      .post('/api/v1/payment/verify')
      .set('Authorization', `Bearer ${token}`)
      .send({
        razorpay_order_id: createResponse.body.data.orderId,
        razorpay_payment_id: 'pay_test',
        razorpay_signature: generateTestSignature()
      })
      .expect(200);

    expect(verifyResponse.body.data.orderId).toBeDefined();
  });
});
```

---

## 🚀 Deployment Configuration

### Environment Variables

```env
# Razorpay Configuration
RAZORPAY_KEY_ID=rzp_live_xxxxxxxxxxxxxx
RAZORPAY_KEY_SECRET=your_secret_key_here
RAZORPAY_WEBHOOK_SECRET=your_webhook_secret_here

# Database
DATABASE_URL=postgresql://user:pass@localhost:5432/chefooz

# App
NODE_ENV=production
```

### Database Migrations

```bash
# Generate migration
npm run migration:generate -- -n AddPaymentIntent

# Run migration
npm run migration:run
```

### Webhook Configuration (Razorpay Dashboard)

1. Go to Settings → Webhooks
2. Click "Create Webhook"
3. URL: `https://api.chefooz.com/api/v1/payment/webhook`
4. Events: `payment.captured`, `payment.failed`, `order.paid`
5. Secret: Copy and save to `RAZORPAY_WEBHOOK_SECRET`

### Health Check

```bash
curl https://api.chefooz.com/api/v1/health
```

Expected response includes Razorpay API connectivity status.

---

**[PAYMENT_TECHNICAL_GUIDE_COMPLETE ✅]**
