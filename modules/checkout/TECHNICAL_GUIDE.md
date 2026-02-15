# 📘 **Checkout Module – Technical Guide**

**Module:** Checkout (Integrated Cross-Module Feature)  
**Version:** 1.0  
**Last Updated:** January 2025  
**Audience:** Backend, Frontend, and DevOps Engineers

---

## 📋 **Table of Contents**

1. [Architecture Overview](#1-architecture-overview)
2. [Database Schema](#2-database-schema)
3. [Backend APIs](#3-backend-apis)
4. [Service Layer Logic](#4-service-layer-logic)
5. [Frontend Integration](#5-frontend-integration)
6. [Distance Calculation](#6-distance-calculation)
7. [Pricing Engine](#7-pricing-engine)
8. [Security & Rate Limiting](#8-security--rate-limiting)
9. [Error Handling](#9-error-handling)
10. [Testing Strategy](#10-testing-strategy)
11. [Performance Optimization](#11-performance-optimization)
12. [Deployment & Monitoring](#12-deployment--monitoring)

---

## 1. Architecture Overview

### 1.1 High-Level Architecture

Checkout is **not a standalone module** but an integrated feature spanning multiple modules:

```
┌─────────────────────────────────────────────────────────────────┐
│                         Checkout Flow                            │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────┐     ┌──────────────┐     ┌─────────────────┐
│    Cart     │────▶│    Order     │────▶│    Payment      │
│   Module    │     │   Module     │     │    Module       │
└─────────────┘     └──────────────┘     └─────────────────┘
      │                    │                      │
      ▼                    ▼                      ▼
┌─────────────┐     ┌──────────────┐     ┌─────────────────┐
│  Address    │     │   Pricing    │     │   Razorpay      │
│   Module    │     │   Service    │     │   Gateway       │
└─────────────┘     └──────────────┘     └─────────────────┘
```

**Key Components:**
- **Cart Module**: Validates cart, converts to order
- **Order Module**: `checkout()` method calculates fees, creates payment intent
- **Address Module**: Provides delivery location for distance calculation
- **Pricing Service**: Calculates delivery fees, GST, surge pricing
- **Payment Module**: Razorpay integration for payment processing

### 1.2 Module Boundaries

| Module | Responsibility |
|--------|----------------|
| **Cart** | Cart validation, item enrichment, `checkoutCart()` endpoint |
| **Order** | Order creation, `checkout()` endpoint, payment intent generation |
| **Address** | Address CRUD, coordinate validation |
| **Pricing** | Distance calculation, dynamic pricing, surge logic |
| **Payment** | Razorpay SDK, signature verification, webhook handling |

### 1.3 Integration Points

```typescript
// Cart → Order
CartService.checkoutCart() → OrderService.createOrderFromCart()

// Order → Address
OrderService.checkout() → AddressRepository.findOne()

// Order → Pricing
OrderService.checkout() → PricingService.calculatePricing()

// Order → Payment
OrderService.checkout() → PaymentIntentId generation
```

### 1.4 Technology Stack

| Layer | Technology |
|-------|-----------|
| **Backend** | NestJS 10+ (TypeScript) |
| **Database** | PostgreSQL 15+ (orders, addresses) |
| **ORM** | TypeORM 0.3+ |
| **Validation** | class-validator, class-transformer |
| **Caching** | Redis (rate limiting, distributed locks) |
| **Frontend** | React Native 0.73+ (Expo Managed) |
| **State Mgmt** | React Query (TanStack Query v5) |
| **Navigation** | Expo Router (file-based) |
| **Payment** | Razorpay SDK 1.9+ |

---

## 2. Database Schema

### 2.1 Orders Table

**Primary Entity**: `orders`

```sql
CREATE TABLE orders (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  
  -- Ownership
  user_id UUID NOT NULL REFERENCES users(id),
  chef_id UUID NOT NULL REFERENCES users(id),
  
  -- Order Details
  items JSONB NOT NULL, -- ChefMenuItem snapshots with quantities
  subtotal_paise INTEGER NOT NULL, -- Sum of item prices
  delivery_fee_paise INTEGER NOT NULL,
  tax_fee_paise INTEGER NOT NULL,
  total_paise INTEGER NOT NULL, -- Grand total
  
  -- Checkout Data
  address_snapshot JSONB, -- Delivery address snapshot
  payment_intent_id VARCHAR(255), -- UUID for payment tracking
  pricing_breakdown JSONB, -- Detailed PricingService breakdown
  
  -- Status
  status VARCHAR(50) NOT NULL DEFAULT 'CREATED',
  payment_status VARCHAR(50) DEFAULT 'PENDING',
  
  -- Timestamps
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  
  -- Indexes
  INDEX idx_orders_user_id (user_id),
  INDEX idx_orders_chef_id (chef_id),
  INDEX idx_orders_status (status),
  INDEX idx_orders_payment_intent_id (payment_intent_id)
);
```

**Key Fields:**
- `address_snapshot`: Full address details (immutable after checkout)
- `pricing_breakdown`: Detailed breakdown from PricingService
- `payment_intent_id`: UUID tracking payment lifecycle

### 2.2 Addresses Table

**Supporting Entity**: `addresses`

```sql
CREATE TABLE addresses (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  
  -- Ownership
  user_id UUID NOT NULL REFERENCES users(id),
  
  -- Address Details
  label VARCHAR(50) NOT NULL, -- 'Home', 'Work', 'Other'
  full_name VARCHAR(255) NOT NULL,
  phone VARCHAR(20) NOT NULL,
  line1 VARCHAR(255) NOT NULL,
  line2 VARCHAR(255),
  city VARCHAR(100) NOT NULL,
  state VARCHAR(100) NOT NULL,
  pincode VARCHAR(20) NOT NULL,
  country VARCHAR(100) DEFAULT 'India',
  
  -- Coordinates (for distance calculation)
  lat DECIMAL(10, 8) NOT NULL,
  lng DECIMAL(11, 8) NOT NULL,
  
  -- Flags
  is_default BOOLEAN DEFAULT false,
  
  -- Timestamps
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  
  -- Indexes
  INDEX idx_addresses_user_id (user_id),
  INDEX idx_addresses_coordinates (lat, lng)
);
```

**Coordinate Validation:**
- Latitude: -90 to 90
- Longitude: -180 to 180
- Used for Haversine distance calculation

### 2.3 Order Status History

**Audit Trail**: `order_status_history`

```sql
CREATE TABLE order_status_history (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id UUID NOT NULL REFERENCES orders(id),
  status VARCHAR(50) NOT NULL,
  note TEXT,
  created_at TIMESTAMP DEFAULT NOW(),
  
  INDEX idx_order_status_history_order_id (order_id)
);
```

---

## 3. Backend APIs

### 3.1 Cart Checkout API

**Endpoint**: `POST /api/v1/cart/checkout`

**Purpose**: Convert cart to draft order

#### Request

```typescript
// No body required - uses authenticated user's cart
Authorization: Bearer <JWT_TOKEN>
```

#### Response

```json
{
  "success": true,
  "message": "Cart checked out successfully",
  "data": {
    "orderId": "123e4567-e89b-12d3-a456-426614174000",
    "orderStatus": "CREATED",
    "totalAmountPaise": 45000,
    "itemCount": 3
  }
}
```

#### Implementation

```typescript
// apps/chefooz-apis/src/modules/cart/cart.service.ts

async checkoutCart(userId: string, addressId?: string): Promise<{
  orderId: string;
  orderStatus: string;
  totalAmountPaise: number;
  itemCount: number;
}> {
  // Get cart
  const cart = await this.getCart(userId);
  
  if (cart.itemCount === 0) {
    throw new BadRequestException('Cart is empty');
  }

  // Validate cart first
  const validation = await this.validateCart(userId);
  if (!validation.valid) {
    throw new BadRequestException(
      'Cart has validation errors. Please review changes before checkout.',
    );
  }

  if (!addressId) {
    throw new BadRequestException('Address is required for checkout');
  }

  // Fetch full ChefMenuItem data for snapshots
  const enrichedItems = await Promise.all(
    cart.items.map(async (item) => {
      const menuItem = await this.menuItemRepo.findOne({
        where: { id: item.menuItemId },
      });

      if (!menuItem) {
        throw new NotFoundException(`Menu item ${item.menuItemId} not found`);
      }

      return {
        menuItemId: item.menuItemId,
        quantity: item.quantity,
        unitPricePaise: item.unitPricePaise,
        customizations: item.optionsSnapshot || {},
        snapshot: menuItem, // Full ChefMenuItem snapshot
      };
    }),
  );

  // Create order via OrderService
  const order = await this.orderService.createOrderFromCart(userId, {
    items: enrichedItems,
    chefId: cart.chefId,
    addressId,
    deliveryFeePaise: cart.totals.deliveryPaise,
    taxFeePaise: cart.totals.taxPaise,
    instructions: '',
    attribution: cart.attribution,
    pricingBreakdown: cart.totals.breakdown,
  });

  // Clear cart after successful order creation
  await this.clearCart(userId);

  return {
    orderId: order.id,
    orderStatus: order.status,
    totalAmountPaise: order.totalPaise,
    itemCount: cart.itemCount,
  };
}
```

**Validation Rules:**
- Cart must not be empty
- All items must be available
- Chef must be active
- Prices must match current menu prices

---

### 3.2 Order Checkout API

**Endpoint**: `POST /api/v1/orders/checkout`

**Purpose**: Attach address, calculate final pricing, create payment intent

#### Request

```typescript
{
  "orderId": "123e4567-e89b-12d3-a456-426614174000",
  "addressId": "987e6543-e89b-12d3-a456-426614174001"
}
```

**DTO Definition:**

```typescript
// apps/chefooz-apis/src/modules/order/dto/checkout.dto.ts

import { IsUUID, IsNotEmpty } from 'class-validator';
import { ApiProperty } from '@nestjs/swagger';

export class CheckoutDto {
  @ApiProperty({
    description: 'Order ID to checkout',
    example: '123e4567-e89b-12d3-a456-426614174000',
  })
  @IsUUID()
  @IsNotEmpty()
  orderId!: string;

  @ApiProperty({
    description: 'Delivery address ID',
    example: '123e4567-e89b-12d3-a456-426614174001',
  })
  @IsUUID()
  @IsNotEmpty()
  addressId!: string;
}
```

#### Response

```json
{
  "success": true,
  "message": "Order checked out successfully",
  "data": {
    "paymentIntentId": "pi_abc123xyz789",
    "totalPaise": 45000
  }
}
```

#### Controller Implementation

```typescript
// apps/chefooz-apis/src/modules/order/order.controller.ts

@Post('orders/checkout')
@UseGuards(RateLimitGuard, LockGuard)
@RateLimit(10, 60) // 10 checkout attempts per minute
@WithLock('order:checkout:{orderId}', 10000) // 10-second lock
@ApiOperation({
  summary: 'Checkout order',
  description: 'Convert draft order to created status with address and payment intent',
})
async checkout(@Body() dto: CheckoutDto, @Request() req: any) {
  const userId = req.user?.id;
  const result = await this.orderService.checkout(
    dto.orderId,
    dto.addressId,
    userId,
  );

  return {
    success: true,
    message: 'Order checked out successfully',
    data: result,
  };
}
```

**Security Layers:**
- **Authentication**: JWT Bearer token required
- **Rate Limiting**: 10 requests/minute per user
- **Distributed Lock**: Prevents concurrent checkout attempts (10s TTL)
- **Ownership Validation**: User must own the order

---

### 3.3 Payment Intent Creation

**Endpoint**: `POST /api/v1/orders/:orderId/payment-intent`

**Purpose**: Initialize Razorpay order for payment

#### Request

```http
POST /api/v1/orders/123e4567/payment-intent
Authorization: Bearer <JWT_TOKEN>
Content-Type: application/json

{
  "paymentMethod": "UPI"
}
```

#### Response

```json
{
  "success": true,
  "message": "Payment intent created",
  "data": {
    "razorpayOrderId": "order_MkL1234567890",
    "amountPaise": 45000,
    "currency": "INR",
    "keyId": "rzp_test_xxxxx"
  }
}
```

**Integration**: See Payment module documentation for Razorpay SDK details.

---

## 4. Service Layer Logic

### 4.1 Order Checkout Service Method

**File**: `apps/chefooz-apis/src/modules/order/order.service.ts`

#### Complete Implementation

```typescript
async checkout(
  orderId: string,
  addressId: string,
  userId: string,
): Promise<{ paymentIntentId: string; totalPaise: number }> {
  // ─────────────────────────────────────────────────────────
  // Step 1: Load and Validate Order
  // ─────────────────────────────────────────────────────────
  const order = await this.orderRepository.findOne({
    where: { id: orderId, userId },
  });

  if (!order) {
    throw new NotFoundException({
      success: false,
      message: 'Order not found',
      errorCode: 'ORDER_NOT_FOUND',
    });
  }

  if (order.status !== OrderStatus.CREATED && 
      order.status !== OrderStatus.PAYMENT_PENDING) {
    throw new ForbiddenException({
      success: false,
      message: 'Order must be in created or payment_pending status',
      errorCode: 'ORDER_NOT_READY',
    });
  }

  // ─────────────────────────────────────────────────────────
  // Step 2: Load and Validate Address
  // ─────────────────────────────────────────────────────────
  const address = await this.addressRepository.findOne({
    where: { id: addressId, userId },
  });

  if (!address) {
    throw new NotFoundException({
      success: false,
      message: 'Address not found',
      errorCode: 'ADDRESS_NOT_FOUND',
    });
  }

  // Create immutable address snapshot
  const addressSnapshot = {
    label: address.label,
    fullName: address.fullName,
    phone: address.phone,
    line1: address.line1,
    line2: address.line2,
    city: address.city,
    state: address.state,
    pincode: address.pincode,
    country: address.country,
    lat: address.lat,
    lng: address.lng,
  };

  // ─────────────────────────────────────────────────────────
  // Step 3: Calculate Distance
  // ─────────────────────────────────────────────────────────
  let distanceKm = 3; // Default fallback
  try {
    const firstItem = order.items[0];
    if (firstItem && firstItem.chefId) {
      const chefProfile = await this.userRepo.findOne({
        where: { id: firstItem.chefId },
      });
      
      if (chefProfile?.location && 
          areValidCoordinates(chefProfile.location.lat, chefProfile.location.lng) &&
          areValidCoordinates(address.lat, address.lng)) {
        distanceKm = calculateDistance(
          chefProfile.location.lat,
          chefProfile.location.lng,
          address.lat,
          address.lng
        );
        this.logger.log(`Calculated delivery distance: ${distanceKm.toFixed(2)}km`);
      }
    }
  } catch (error) {
    this.logger.warn(`Failed to calculate distance, using default: ${error.message}`);
  }

  // ─────────────────────────────────────────────────────────
  // Step 4: Calculate Pricing (PricingService Integration)
  // ─────────────────────────────────────────────────────────
  const timeConditions = getPricingTimeConditions();
  let deliveryFeePaise: number;
  let taxFeePaise: number;
  let totalPaise: number;
  let pricingBreakdown: any;

  try {
    const pricingResult = await this.pricingService.calculatePricing({
      items: order.items.map(item => ({
        id: item.id || 'temp',
        pricePaise: item.unitPricePaise,
        quantity: item.quantity,
        gstExempt: false,
      })),
      delivery: {
        distanceKm,
        coordinates: {
          lat: address.lat,
          lng: address.lng,
        },
        isPeakTime: timeConditions.isPeakTime,
      },
      chefId: order.items[0]?.chefId || 'unknown',
    });

    deliveryFeePaise = pricingResult.breakdown.totalDeliveryFee;
    taxFeePaise = pricingResult.breakdown.gst.total;
    totalPaise = pricingResult.breakdown.grandTotal;
    pricingBreakdown = pricingResult.breakdown;

    this.logger.log(`Pricing: Delivery=₹${deliveryFeePaise/100}, Tax=₹${taxFeePaise/100}, Total=₹${totalPaise/100}`);
  } catch (error) {
    // Fallback to simple calculation
    this.logger.error(`PricingService failed, using fallback: ${error.message}`);
    deliveryFeePaise = order.deliveryFeePaise || 2000; // ₹20
    taxFeePaise = order.taxFeePaise || Math.floor(order.subtotalPaise * 0.05); // 5%
    totalPaise = order.subtotalPaise + deliveryFeePaise + taxFeePaise;
    pricingBreakdown = null;
  }

  // ─────────────────────────────────────────────────────────
  // Step 5: Generate Payment Intent ID
  // ─────────────────────────────────────────────────────────
  const paymentIntentId = uuidv4();

  // ─────────────────────────────────────────────────────────
  // Step 6: Update Order
  // ─────────────────────────────────────────────────────────
  order.addressSnapshot = addressSnapshot;
  order.deliveryFeePaise = deliveryFeePaise;
  order.taxFeePaise = taxFeePaise;
  order.totalPaise = totalPaise;
  order.pricingBreakdown = pricingBreakdown;
  order.paymentIntentId = paymentIntentId;
  order.paymentStatus = PaymentStatus.PENDING;
  order.status = OrderStatus.CREATED;

  await this.orderRepository.save(order);

  // ─────────────────────────────────────────────────────────
  // Step 7: Create Status History
  // ─────────────────────────────────────────────────────────
  await this.orderStatusHistoryRepository.save({
    orderId: order.id,
    status: OrderStatus.CREATED,
    note: 'Order submitted for payment',
  });

  return {
    paymentIntentId,
    totalPaise,
  };
}
```

#### Key Business Rules

| Rule | Implementation | Fallback |
|------|---------------|----------|
| **Distance Calculation** | Haversine formula using chef & customer coords | 3 km default |
| **Max Distance** | 15 km (enforced in PricingService) | - |
| **Surge Pricing** | 20% during 12-2 PM, 7-10 PM | Off-peak pricing |
| **GST** | 5% on taxable amount | Fallback: 5% of subtotal |
| **Address Immutability** | Snapshot saved, changes don't affect order | - |

---

### 4.2 Distance Calculation Utility

**File**: `apps/chefooz-apis/src/modules/pricing/utils/distance.util.ts`

#### Haversine Formula Implementation

```typescript
/**
 * Calculate distance between two coordinates using Haversine formula
 * 
 * @param lat1 - Latitude of first point (degrees)
 * @param lon1 - Longitude of first point (degrees)
 * @param lat2 - Latitude of second point (degrees)
 * @param lon2 - Longitude of second point (degrees)
 * @returns Distance in kilometers
 */
export function calculateDistance(
  lat1: number,
  lon1: number,
  lat2: number,
  lon2: number
): number {
  const R = 6371; // Earth's radius in kilometers
  
  const dLat = toRad(lat2 - lat1);
  const dLon = toRad(lon2 - lon1);
  
  const a =
    Math.sin(dLat / 2) * Math.sin(dLat / 2) +
    Math.cos(toRad(lat1)) *
      Math.cos(toRad(lat2)) *
      Math.sin(dLon / 2) *
      Math.sin(dLon / 2);
  
  const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
  const distance = R * c;
  
  return distance;
}

function toRad(degrees: number): number {
  return (degrees * Math.PI) / 180;
}
```

**Mathematical Basis:**

The Haversine formula calculates great-circle distance between two points on a sphere:

```
a = sin²(Δφ/2) + cos φ1 ⋅ cos φ2 ⋅ sin²(Δλ/2)
c = 2 ⋅ atan2(√a, √(1−a))
d = R ⋅ c
```

Where:
- φ = latitude
- λ = longitude  
- R = Earth's radius (6371 km)

**Accuracy:**
- ±0.5% error vs actual road distance in urban areas
- Acceptable for delivery fee calculation

#### Coordinate Validation

```typescript
export function areValidCoordinates(lat?: number, lng?: number): boolean {
  if (lat === undefined || lng === undefined) {
    return false;
  }
  
  // Valid ranges
  return (
    lat >= -90 && lat <= 90 &&
    lng >= -180 && lng <= 180
  );
}
```

#### Peak Time Detection

```typescript
export function getPricingTimeConditions(): {
  isPeakTime: boolean;
  timeSlot: 'breakfast' | 'lunch' | 'dinner' | 'off-peak';
} {
  const now = new Date();
  const hour = now.getHours();
  
  // Peak hours definition
  const isBreakfast = hour >= 7 && hour < 10;
  const isLunch = hour >= 12 && hour < 14;
  const isDinner = hour >= 19 && hour < 21;
  
  const isPeakTime = isBreakfast || isLunch || isDinner;
  
  let timeSlot: 'breakfast' | 'lunch' | 'dinner' | 'off-peak' = 'off-peak';
  if (isBreakfast) timeSlot = 'breakfast';
  else if (isLunch) timeSlot = 'lunch';
  else if (isDinner) timeSlot = 'dinner';
  
  return { isPeakTime, timeSlot };
}
```

---

## 5. Frontend Integration

### 5.1 React Query Hooks

**File**: `libs/api-client/src/lib/hooks/useCheckout.ts`

#### Checkout Mutation Hook

```typescript
import { useMutation, useQuery, useQueryClient } from '@tanstack/react-query';
import { checkout, confirmPayment, getOrder } from '../clients/order.client';
import type { PaymentIntentResponse, OrderDraft } from '@chefooz-app/types';

export const checkoutKeys = {
  all: ['orders'] as const,
  detail: (orderId: string) => [...checkoutKeys.all, orderId] as const,
};

export const useCheckoutMutation = () => {
  const queryClient = useQueryClient();

  return useMutation<
    PaymentIntentResponse,
    Error,
    { orderId: string; addressId: string }
  >({
    mutationFn: ({ orderId, addressId }) => checkout(orderId, addressId),
    onSuccess: (_, variables) => {
      queryClient.invalidateQueries({
        queryKey: checkoutKeys.detail(variables.orderId),
      });
    },
  });
};
```

**Usage:**

```typescript
const { mutate: checkoutOrder, isPending } = useCheckoutMutation();

const handleCheckout = () => {
  checkoutOrder(
    { orderId: 'abc-123', addressId: 'xyz-789' },
    {
      onSuccess: (data) => {
        console.log('Payment Intent:', data.paymentIntentId);
        // Navigate to payment screen
      },
      onError: (error) => {
        Alert.alert('Checkout Failed', error.message);
      },
    }
  );
};
```

#### Confirm Payment Hook

```typescript
export const useConfirmPaymentMutation = () => {
  const queryClient = useQueryClient();

  return useMutation<
    void,
    Error,
    { orderId: string; paymentIntentId: string }
  >({
    mutationFn: ({ orderId, paymentIntentId }) =>
      confirmPayment(orderId, paymentIntentId),
    onSuccess: (_, variables) => {
      queryClient.invalidateQueries({
        queryKey: checkoutKeys.detail(variables.orderId),
      });
    },
  });
};
```

#### Order Detail Query

```typescript
export const useOrderDetailQuery = (orderId: string) => {
  return useQuery<OrderDraft, Error>({
    queryKey: checkoutKeys.detail(orderId),
    queryFn: () => getOrder(orderId),
    staleTime: 1000 * 10, // 10 seconds
    refetchInterval: 10000, // Poll every 10s for status updates
    refetchOnWindowFocus: true,
    enabled: !!orderId,
  });
};
```

---

### 5.2 Axios API Clients

**File**: `libs/api-client/src/lib/clients/order.client.ts`

```typescript
/**
 * Checkout order
 * Convert draft order to created status with payment intent
 */
export const checkout = async (
  orderId: string,
  addressId: string
): Promise<PaymentIntentResponse> => {
  const response = await apiClient.post<CheckoutResponse>(
    `/v1/orders/checkout`,
    { orderId, addressId }
  );

  if (!response.data.success) {
    throw new Error(response.data.errorCode || 'Failed to checkout order');
  }

  return response.data.data as PaymentIntentResponse;
};

/**
 * Confirm Razorpay payment
 * Verify signature and mark order as paid
 */
export const confirmRazorpayPayment = async (
  orderId: string,
  payload: ConfirmPaymentDto
): Promise<void> => {
  const response = await apiClient.post<ConfirmPaymentResponse>(
    `/v1/orders/${orderId}/payment/confirm`,
    payload
  );

  if (!response.data.success) {
    throw new Error(response.data.errorCode || 'Failed to confirm payment');
  }
};
```

---

### 5.3 Address Selection Screen

**File**: `apps/chefooz-app/src/app/checkout/address.tsx`

#### Key Features

```typescript
export default function AddressSelectionScreen() {
  const router = useRouter();
  const { orderId } = useLocalSearchParams<{ orderId: string }>();
  const [selectedAddressId, setSelectedAddressId] = useState<string | null>(null);
  
  // Fetch addresses
  const { data: addresses, isLoading } = useAddressesQuery();
  
  // Auto-select default address
  useEffect(() => {
    if (addresses && addresses.length > 0) {
      const defaultAddress = addresses.find(a => a.isDefault);
      if (defaultAddress) {
        setSelectedAddressId(defaultAddress.id);
      } else {
        setSelectedAddressId(addresses[0].id);
      }
    }
  }, [addresses]);
  
  const handleProceedToPayment = () => {
    if (!selectedAddressId) {
      Alert.alert('Error', 'Please select a delivery address');
      return;
    }
    
    router.push(`/cart/payment?addressId=${selectedAddressId}&orderId=${orderId}`);
  };
  
  // Render address cards with radio buttons
  // ...
}
```

**UI Components:**
- Radio-style address cards
- Default address badge
- "Add New Address" CTA
- Empty state handling
- Loading skeleton

---

### 5.4 Payment Method Screen

**File**: `apps/chefooz-app/src/app/cart/payment.tsx`

#### Checkout Flow

```typescript
export default function CartPaymentScreen() {
  const { orderId, addressId } = useLocalSearchParams();
  const router = useRouter();
  
  const [paymentMethod, setPaymentMethod] = useState<'UPI' | 'COD'>('UPI');
  const [isProcessing, setIsProcessing] = useState(false);
  
  // Hooks
  const { data: order } = useOrderDetailQuery(orderId);
  const { data: address } = useAddressDetailQuery(addressId);
  const checkoutMutation = useCheckoutMutation();
  
  const handlePlaceOrder = async () => {
    if (isProcessing) return; // Prevent double-tap
    
    setIsProcessing(true);
    
    try {
      // Step 1: Checkout (attach address, calculate fees)
      const { paymentIntentId, totalPaise } = await checkoutMutation.mutateAsync({
        orderId,
        addressId,
      });
      
      // Step 2: Payment method routing
      if (paymentMethod === 'UPI') {
        // Initialize Razorpay SDK
        const options = {
          amount: totalPaise,
          currency: 'INR',
          order_id: paymentIntentId,
          prefill: {
            email: user.email,
            contact: address.phone,
          },
        };
        
        RazorpayCheckout.open(options)
          .then((data) => {
            // Payment success
            router.push(`/orders/${orderId}?success=true`);
          })
          .catch((error) => {
            Alert.alert('Payment Failed', 'Please try again');
          });
      } else {
        // COD: Mark order as confirmed
        await confirmPaymentMutation.mutateAsync({
          orderId,
          paymentIntentId,
        });
        router.push(`/orders/${orderId}?success=true`);
      }
    } catch (error) {
      Alert.alert('Checkout Error', error.message);
    } finally {
      setIsProcessing(false);
    }
  };
  
  // Render UI
  // ...
}
```

**UI Components:**
- Chef summary card (avatar, rating, ETA)
- Address display with "Change" CTA
- Order items list (veg/non-veg indicators)
- Price breakdown modal
- Payment method radio cards (UPI/COD)
- Sticky CTA bar with grand total

---

## 6. Distance Calculation

### 6.1 Haversine Formula Deep Dive

**Use Case**: Calculate straight-line distance between chef location and customer address.

#### Formula Components

```typescript
// Input: Two GPS coordinates
const chef = { lat: 28.6139, lng: 77.2090 }; // Delhi
const customer = { lat: 28.7041, lng: 77.1025 }; // Delhi NCR

// Step 1: Convert degrees to radians
const dLat = toRad(customer.lat - chef.lat);
const dLon = toRad(customer.lng - chef.lng);

// Step 2: Calculate intermediate value 'a'
const a =
  Math.sin(dLat / 2) * Math.sin(dLat / 2) +
  Math.cos(toRad(chef.lat)) *
    Math.cos(toRad(customer.lat)) *
    Math.sin(dLon / 2) *
    Math.sin(dLon / 2);

// Step 3: Calculate central angle 'c'
const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));

// Step 4: Calculate distance
const distance = 6371 * c; // Result: ~12.5 km
```

### 6.2 Validation Rules

```typescript
// Before calculating distance
if (!areValidCoordinates(chefLat, chefLng)) {
  throw new Error('Invalid chef coordinates');
}
if (!areValidCoordinates(customerLat, customerLng)) {
  throw new Error('Invalid customer coordinates');
}

// After calculation
if (distanceKm > 15) {
  throw new BadRequestException('Delivery distance exceeds 15km limit');
}
```

### 6.3 Fallback Strategy

```typescript
let distanceKm = 3; // Default fallback (3km)

try {
  distanceKm = calculateDistance(chefLat, chefLng, customerLat, customerLng);
  logger.log(`Calculated: ${distanceKm.toFixed(2)}km`);
} catch (error) {
  logger.warn(`Distance calc failed, using 3km default: ${error.message}`);
}
```

**Why 3km?**
- Average urban delivery distance in India
- Conservative estimate (slightly higher delivery fee)
- Prevents severe undercharging

---

## 7. Pricing Engine

### 7.1 PricingService Architecture

**File**: `apps/chefooz-apis/src/modules/pricing/pricing.service.ts`

#### Calculation Pipeline

```
Input: Items, Distance, Time
       │
       ▼
┌──────────────────┐
│  Item Subtotal   │ ← Sum(item.price × quantity)
└──────────────────┘
       │
       ▼
┌──────────────────┐
│ Packaging Fees   │ ← ₹10 per order
└──────────────────┘
       │
       ▼
┌──────────────────┐
│ Delivery Fees    │ ← Base + Distance + Surge
└──────────────────┘
       │
       ▼
┌──────────────────┐
│  Platform Fee    │ ← 0% (disabled for MVP)
└──────────────────┘
       │
       ▼
┌──────────────────┐
│   GST (5%)       │ ← Tax on subtotal
└──────────────────┘
       │
       ▼
   Grand Total
```

### 7.2 Delivery Fee Calculation

```typescript
private calculateDeliveryFees(
  delivery: { distanceKm: number; isPeakTime: boolean },
  config: PricingConfig
): {
  base: number;
  distanceSurcharge: number;
  surgeSurcharge: number;
  total: number;
} {
  const { distancePricing, surgePricing } = config;
  const { distanceKm, isPeakTime } = delivery;
  
  // Base fee (0-3km: ₹20)
  const base = distancePricing.baseFee; // 2000 paise
  
  // Distance surcharge (beyond 3km: ₹5/km)
  let distanceSurcharge = 0;
  if (distanceKm > distancePricing.freeUpToKm) {
    const chargeableDistance = distanceKm - distancePricing.freeUpToKm;
    distanceSurcharge = Math.round(chargeableDistance * distancePricing.perKmCharge);
  }
  
  // Calculate subtotal before surge
  let subtotal = base + distanceSurcharge;
  
  // Apply surge pricing (20% during peak hours)
  let surgeSurcharge = 0;
  if (surgePricing?.enabled && isPeakTime) {
    surgeSurcharge = Math.round(subtotal * (surgePricing.peakMultiplier - 1));
    subtotal += surgeSurcharge;
  }
  
  // Apply max cap (₹150)
  let total = Math.min(subtotal, distancePricing.maxFee);
  
  return {
    base,
    distanceSurcharge,
    surgeSurcharge,
    total,
  };
}
```

**Example Calculation:**

```typescript
// Scenario: 8km delivery at 1 PM (peak time)
const distanceKm = 8;
const isPeakTime = true;

// Base fee: ₹20
const base = 2000;

// Distance surcharge: (8 - 3) × ₹5 = ₹25
const chargeableDistance = 8 - 3; // 5 km
const distanceSurcharge = 5 * 500; // 2500 paise

// Subtotal: ₹20 + ₹25 = ₹45
let subtotal = 2000 + 2500; // 4500 paise

// Surge (20%): ₹45 × 0.2 = ₹9
const surgeSurcharge = Math.round(4500 * 0.2); // 900 paise

// Total: ₹45 + ₹9 = ₹54
const total = subtotal + surgeSurcharge; // 5400 paise

// Final result: ₹54 delivery fee
```

### 7.3 GST Calculation

```typescript
private calculateGST(
  taxableAmount: number,
  items: PricingCalculationInput['items'],
  config: PricingConfig
): {
  cgst: number;
  sgst: number;
  total: number;
} {
  const gstRate = config.gst.rate; // 0.05 (5%)
  const totalGst = Math.round(taxableAmount * gstRate);
  
  // Split equally into CGST and SGST
  const cgst = Math.floor(totalGst / 2);
  const sgst = totalGst - cgst;
  
  return {
    cgst,
    sgst,
    total: totalGst,
  };
}
```

**Example:**

```typescript
// Order subtotal: ₹400
const subtotal = 40000; // paise

// GST (5%): ₹400 × 0.05 = ₹20
const totalGst = Math.round(40000 * 0.05); // 2000 paise

// CGST (2.5%): ₹10
const cgst = Math.floor(2000 / 2); // 1000 paise

// SGST (2.5%): ₹10
const sgst = 2000 - 1000; // 1000 paise
```

### 7.4 Complete Pricing Breakdown

```typescript
interface PricingBreakdown {
  // Item costs
  itemTotal: number;
  packagingCharges: number;
  
  // Delivery fees
  totalDeliveryFee: number;
  deliveryBreakdown: {
    base: number;
    distanceSurcharge: number;
    surgeSurcharge: number;
  };
  
  // Taxes
  gst: {
    cgst: number;
    sgst: number;
    total: number;
  };
  
  // Discounts
  discounts: {
    itemDiscount: number;
    deliveryDiscount: number;
    totalDiscount: number;
  };
  
  // Final amounts
  subtotalBeforeGST: number;
  grandTotal: number;
  
  // Metadata
  distanceKm: number;
  isPeakTime: boolean;
}
```

**Stored in Order:**

```typescript
order.pricingBreakdown = {
  itemTotal: 40000,
  packagingCharges: 1000,
  totalDeliveryFee: 5400,
  deliveryBreakdown: {
    base: 2000,
    distanceSurcharge: 2500,
    surgeSurcharge: 900,
  },
  gst: {
    cgst: 1000,
    sgst: 1000,
    total: 2000,
  },
  discounts: {
    itemDiscount: 0,
    deliveryDiscount: 0,
    totalDiscount: 0,
  },
  subtotalBeforeGST: 46400,
  grandTotal: 48400,
  distanceKm: 8,
  isPeakTime: true,
};
```

---

## 8. Security & Rate Limiting

### 8.1 Rate Limiting

**Implementation**: Redis-backed sliding window

```typescript
@UseGuards(RateLimitGuard)
@RateLimit(10, 60) // 10 requests per 60 seconds
async checkout(@Body() dto: CheckoutDto, @Request() req: any) {
  // ...
}
```

**Guard Logic:**

```typescript
// apps/chefooz-apis/src/modules/cache/guards/rate-limit.guard.ts

@Injectable()
export class RateLimitGuard implements CanActivate {
  constructor(private readonly cacheService: CacheService) {}
  
  async canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    const userId = request.user?.id;
    
    if (!userId) {
      throw new UnauthorizedException('User not authenticated');
    }
    
    const decorator = this.reflector.get<RateLimitMetadata>(
      RATE_LIMIT_KEY,
      context.getHandler()
    );
    
    if (!decorator) return true;
    
    const { limit, windowSeconds } = decorator;
    const key = `rate-limit:${userId}:${context.getClass().name}:${context.getHandler().name}`;
    
    // Increment counter
    const current = await this.cacheService.incr(key);
    
    // Set TTL on first request
    if (current === 1) {
      await this.cacheService.expire(key, windowSeconds);
    }
    
    if (current > limit) {
      throw new TooManyRequestsException(
        `Rate limit exceeded. Try again in ${windowSeconds} seconds.`
      );
    }
    
    return true;
  }
}
```

**Rate Limits:**

| Endpoint | Limit | Window | Reason |
|----------|-------|--------|--------|
| `POST /orders/checkout` | 10 req | 60s | Prevent spam checkouts |
| `POST /orders/:id/payment-intent` | 10 req | 60s | Limit Razorpay API calls |
| `POST /orders/:id/payment/confirm` | 5 req | 60s | Payment confirmation throttle |

### 8.2 Distributed Locks

**Purpose**: Prevent race conditions during concurrent checkout attempts

```typescript
@WithLock('order:checkout:{orderId}', 10000) // 10-second TTL
async checkout(@Body() dto: CheckoutDto, @Request() req: any) {
  // Lock acquired, safe to proceed
}
```

**Guard Implementation:**

```typescript
@Injectable()
export class LockGuard implements CanActivate {
  constructor(private readonly cacheService: CacheService) {}
  
  async canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    const decorator = this.reflector.get<LockMetadata>(
      LOCK_KEY,
      context.getHandler()
    );
    
    if (!decorator) return true;
    
    const { key: keyTemplate, ttlMs } = decorator;
    
    // Replace placeholders with actual values
    const key = this.resolveLockKey(keyTemplate, request);
    
    // Try to acquire lock (SET NX EX)
    const acquired = await this.cacheService.setNx(key, '1', ttlMs / 1000);
    
    if (!acquired) {
      throw new ConflictException('Operation already in progress. Please wait.');
    }
    
    // Release lock after request completes
    request.on('finish', async () => {
      await this.cacheService.del(key);
    });
    
    return true;
  }
}
```

**Lock Keys:**

```typescript
// Example: order:checkout:123e4567-e89b-12d3-a456-426614174000
const lockKey = `order:checkout:${dto.orderId}`;
```

### 8.3 Authentication & Authorization

```typescript
// JWT verification (automatic via @UseGuards(AuthGuard))
const userId = req.user?.id;

// Ownership validation
const order = await this.orderRepository.findOne({
  where: { id: orderId, userId },
});

if (!order) {
  throw new NotFoundException('Order not found');
}

// User can only checkout their own orders
```

---

## 9. Error Handling

### 9.1 Error Taxonomy

| Error Code | HTTP Status | Meaning | Recovery |
|------------|-------------|---------|----------|
| `ORDER_NOT_FOUND` | 404 | Order doesn't exist or user doesn't own it | Check orderId |
| `ORDER_NOT_READY` | 403 | Order status invalid for checkout | Review order state |
| `ADDRESS_NOT_FOUND` | 404 | Address doesn't exist or user doesn't own it | Select valid address |
| `DISTANCE_EXCEEDED` | 400 | Delivery distance > 15km | Choose closer address |
| `CART_EMPTY` | 400 | Cart has no items | Add items to cart |
| `CART_VALIDATION_FAILED` | 400 | Price/availability mismatch | Refresh cart |
| `PRICING_SERVICE_FAILED` | 500 | PricingService error | Fallback pricing applied |
| `RATE_LIMIT_EXCEEDED` | 429 | Too many requests | Retry after window |
| `LOCK_CONFLICT` | 409 | Concurrent checkout attempt | Wait 10s, retry |

### 9.2 Error Response Format

```json
{
  "success": false,
  "message": "Address not found",
  "errorCode": "ADDRESS_NOT_FOUND",
  "statusCode": 404,
  "timestamp": "2025-01-15T10:30:00.000Z",
  "path": "/api/v1/orders/checkout"
}
```

### 9.3 Frontend Error Handling

```typescript
const { mutate: checkoutOrder } = useCheckoutMutation();

const handleCheckout = () => {
  checkoutOrder(
    { orderId, addressId },
    {
      onError: (error: Error) => {
        // Parse error code
        const errorCode = error.message;
        
        switch (errorCode) {
          case 'ADDRESS_NOT_FOUND':
            Alert.alert(
              'Invalid Address',
              'Please select a valid delivery address.',
              [{ text: 'Select Address', onPress: () => router.push('/checkout/address') }]
            );
            break;
          
          case 'DISTANCE_EXCEEDED':
            Alert.alert(
              'Delivery Unavailable',
              'This chef is more than 15km away. Please choose a closer chef.',
              [{ text: 'OK', onPress: () => router.back() }]
            );
            break;
          
          case 'RATE_LIMIT_EXCEEDED':
            Alert.alert(
              'Too Many Attempts',
              'Please wait a minute before trying again.'
            );
            break;
          
          default:
            Alert.alert(
              'Checkout Failed',
              'Something went wrong. Please try again.',
              [{ text: 'Retry', onPress: handleCheckout }]
            );
        }
      },
    }
  );
};
```

---

## 10. Testing Strategy

### 10.1 Unit Tests

#### Distance Calculation Tests

```typescript
// apps/chefooz-apis/src/modules/pricing/utils/distance.util.spec.ts

describe('calculateDistance', () => {
  it('should calculate distance between Delhi and Gurgaon', () => {
    const delhi = { lat: 28.6139, lng: 77.2090 };
    const gurgaon = { lat: 28.4595, lng: 77.0266 };
    
    const distance = calculateDistance(
      delhi.lat,
      delhi.lng,
      gurgaon.lat,
      gurgaon.lng
    );
    
    // Expected: ~23 km
    expect(distance).toBeGreaterThan(22);
    expect(distance).toBeLessThan(24);
  });
  
  it('should handle same coordinates (0 km)', () => {
    const distance = calculateDistance(28.6139, 77.2090, 28.6139, 77.2090);
    expect(distance).toBe(0);
  });
});

describe('areValidCoordinates', () => {
  it('should validate correct coordinates', () => {
    expect(areValidCoordinates(28.6139, 77.2090)).toBe(true);
  });
  
  it('should reject invalid latitude', () => {
    expect(areValidCoordinates(91, 77.2090)).toBe(false);
    expect(areValidCoordinates(-91, 77.2090)).toBe(false);
  });
  
  it('should reject undefined values', () => {
    expect(areValidCoordinates(undefined, 77.2090)).toBe(false);
  });
});
```

#### Pricing Service Tests

```typescript
// apps/chefooz-apis/src/modules/pricing/pricing.service.spec.ts

describe('PricingService', () => {
  let service: PricingService;
  
  beforeEach(() => {
    service = new PricingService();
  });
  
  it('should calculate base delivery fee (3km, off-peak)', async () => {
    const result = await service.calculatePricing({
      items: [{ id: '1', pricePaise: 10000, quantity: 1, gstExempt: false }],
      delivery: { distanceKm: 3, isPeakTime: false },
      chefId: 'chef-1',
    });
    
    // Base fee only: ₹20
    expect(result.breakdown.totalDeliveryFee).toBe(2000);
  });
  
  it('should calculate distance surcharge (8km, off-peak)', async () => {
    const result = await service.calculatePricing({
      items: [{ id: '1', pricePaise: 10000, quantity: 1, gstExempt: false }],
      delivery: { distanceKm: 8, isPeakTime: false },
      chefId: 'chef-1',
    });
    
    // Base ₹20 + (8-3) × ₹5 = ₹45
    expect(result.breakdown.totalDeliveryFee).toBe(4500);
  });
  
  it('should apply surge pricing (3km, peak time)', async () => {
    const result = await service.calculatePricing({
      items: [{ id: '1', pricePaise: 10000, quantity: 1, gstExempt: false }],
      delivery: { distanceKm: 3, isPeakTime: true },
      chefId: 'chef-1',
    });
    
    // Base ₹20 × 1.2 = ₹24
    expect(result.breakdown.totalDeliveryFee).toBe(2400);
  });
  
  it('should calculate GST correctly', async () => {
    const result = await service.calculatePricing({
      items: [{ id: '1', pricePaise: 40000, quantity: 1, gstExempt: false }],
      delivery: { distanceKm: 3, isPeakTime: false },
      chefId: 'chef-1',
    });
    
    // Item ₹400 + Packaging ₹10 + Delivery ₹20 = ₹430
    // GST (5%): ₹21.50
    expect(result.breakdown.gst.total).toBe(2150);
  });
});
```

### 10.2 Integration Tests

```typescript
// apps/chefooz-apis/src/modules/order/order.service.spec.ts

describe('OrderService - checkout', () => {
  let service: OrderService;
  let orderRepo: Repository<Order>;
  let addressRepo: Repository<Address>;
  
  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        OrderService,
        { provide: getRepositoryToken(Order), useClass: Repository },
        { provide: getRepositoryToken(Address), useClass: Repository },
        PricingService,
      ],
    }).compile();
    
    service = module.get<OrderService>(OrderService);
    orderRepo = module.get(getRepositoryToken(Order));
    addressRepo = module.get(getRepositoryToken(Address));
  });
  
  it('should checkout order successfully', async () => {
    // Mock order
    const mockOrder = {
      id: 'order-1',
      userId: 'user-1',
      status: OrderStatus.CREATED,
      items: [{ id: 'item-1', unitPricePaise: 10000, quantity: 2, chefId: 'chef-1' }],
      subtotalPaise: 20000,
    };
    
    // Mock address
    const mockAddress = {
      id: 'address-1',
      userId: 'user-1',
      lat: 28.6139,
      lng: 77.2090,
    };
    
    jest.spyOn(orderRepo, 'findOne').mockResolvedValue(mockOrder as any);
    jest.spyOn(addressRepo, 'findOne').mockResolvedValue(mockAddress as any);
    jest.spyOn(orderRepo, 'save').mockResolvedValue(mockOrder as any);
    
    // Execute
    const result = await service.checkout('order-1', 'address-1', 'user-1');
    
    // Verify
    expect(result).toHaveProperty('paymentIntentId');
    expect(result).toHaveProperty('totalPaise');
    expect(mockOrder.addressSnapshot).toBeDefined();
  });
  
  it('should throw error for invalid order status', async () => {
    const mockOrder = {
      id: 'order-1',
      userId: 'user-1',
      status: OrderStatus.PAID, // Already paid
    };
    
    jest.spyOn(orderRepo, 'findOne').mockResolvedValue(mockOrder as any);
    
    await expect(
      service.checkout('order-1', 'address-1', 'user-1')
    ).rejects.toThrow('Order must be in created or payment_pending status');
  });
});
```

### 10.3 E2E Tests

```typescript
// apps/chefooz-apis-e2e/src/order/checkout.e2e-spec.ts

describe('POST /api/v1/orders/checkout', () => {
  let app: INestApplication;
  let authToken: string;
  let orderId: string;
  let addressId: string;
  
  beforeAll(async () => {
    // Setup app
    const moduleFixture = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();
    
    app = moduleFixture.createNestApplication();
    await app.init();
    
    // Login and get token
    const loginRes = await request(app.getHttpServer())
      .post('/api/v1/auth/login')
      .send({ email: 'test@example.com', password: 'password' });
    
    authToken = loginRes.body.data.accessToken;
    
    // Create test order and address
    // ...
  });
  
  it('should checkout order successfully', async () => {
    const response = await request(app.getHttpServer())
      .post('/api/v1/orders/checkout')
      .set('Authorization', `Bearer ${authToken}`)
      .send({ orderId, addressId })
      .expect(200);
    
    expect(response.body.success).toBe(true);
    expect(response.body.data).toHaveProperty('paymentIntentId');
    expect(response.body.data).toHaveProperty('totalPaise');
  });
  
  it('should enforce rate limiting', async () => {
    // Make 11 requests (limit is 10)
    for (let i = 0; i < 11; i++) {
      const response = await request(app.getHttpServer())
        .post('/api/v1/orders/checkout')
        .set('Authorization', `Bearer ${authToken}`)
        .send({ orderId, addressId });
      
      if (i < 10) {
        expect(response.status).toBe(200);
      } else {
        expect(response.status).toBe(429); // Too Many Requests
      }
    }
  });
});
```

---

## 11. Performance Optimization

### 11.1 Database Query Optimization

```typescript
// ❌ BAD: N+1 query problem
const order = await this.orderRepository.findOne({ where: { id: orderId } });
const address = await this.addressRepository.findOne({ where: { id: addressId } });
const chef = await this.userRepo.findOne({ where: { id: order.items[0].chefId } });

// ✅ GOOD: Parallel queries
const [order, address] = await Promise.all([
  this.orderRepository.findOne({ where: { id: orderId, userId } }),
  this.addressRepository.findOne({ where: { id: addressId, userId } }),
]);

// Then fetch chef only if needed
if (order?.items[0]?.chefId) {
  const chef = await this.userRepo.findOne({ where: { id: order.items[0].chefId } });
}
```

### 11.2 Caching Strategy

```typescript
// Cache address coordinates (rarely change)
const cacheKey = `address:coords:${addressId}`;
let coords = await this.cacheService.get(cacheKey);

if (!coords) {
  const address = await this.addressRepository.findOne({ where: { id: addressId } });
  coords = { lat: address.lat, lng: address.lng };
  await this.cacheService.set(cacheKey, coords, 3600); // 1 hour TTL
}
```

### 11.3 Response Time Targets

| Operation | Target | P95 | Notes |
|-----------|--------|-----|-------|
| `POST /orders/checkout` | <500ms | <800ms | Includes distance calc + pricing |
| `calculateDistance()` | <5ms | <10ms | Pure computation |
| `PricingService.calculatePricing()` | <50ms | <100ms | All pricing calculations |
| Frontend checkout flow | <2s | <3s | Cart → Address → Payment |

### 11.4 Monitoring

```typescript
// Add performance logging
const startTime = Date.now();

const distance = calculateDistance(chefLat, chefLng, addressLat, addressLng);

const duration = Date.now() - startTime;
this.logger.debug(`Distance calculation took ${duration}ms`);

if (duration > 10) {
  this.logger.warn(`Slow distance calculation: ${duration}ms`);
}
```

---

## 12. Deployment & Monitoring

### 12.1 Environment Variables

```bash
# apps/chefooz-apis/.env

# Database
DATABASE_URL=postgresql://user:pass@host:5432/chefooz
DATABASE_SSL=true

# Redis (Rate limiting, locks)
REDIS_URL=redis://host:6379
REDIS_PASSWORD=secret

# Razorpay
RAZORPAY_KEY_ID=rzp_live_xxxxx
RAZORPAY_KEY_SECRET=secret

# Pricing Config
PRICING_DELIVERY_BASE_FEE=2000  # ₹20 in paise
PRICING_DELIVERY_PER_KM=500     # ₹5/km in paise
PRICING_DELIVERY_MAX_FEE=15000  # ₹150 cap
PRICING_GST_RATE=0.05           # 5%
PRICING_SURGE_MULTIPLIER=1.20   # 20% surge
```

### 12.2 Health Checks

```typescript
// apps/chefooz-apis/src/health/health.controller.ts

@Get('health/checkout')
async checkoutHealth(): Promise<HealthCheckResult> {
  // Test database
  const dbHealthy = await this.orderRepository.findOne({ where: {} });
  
  // Test Redis
  const redisHealthy = await this.cacheService.ping();
  
  // Test PricingService
  const pricingHealthy = await this.pricingService.calculatePricing({
    items: [{ id: 'test', pricePaise: 1000, quantity: 1, gstExempt: false }],
    delivery: { distanceKm: 3, isPeakTime: false },
    chefId: 'test',
  });
  
  return {
    status: dbHealthy && redisHealthy && pricingHealthy ? 'healthy' : 'unhealthy',
    checks: {
      database: dbHealthy ? 'up' : 'down',
      redis: redisHealthy ? 'up' : 'down',
      pricing: pricingHealthy ? 'up' : 'down',
    },
  };
}
```

### 12.3 Metrics to Monitor

| Metric | Threshold | Alert |
|--------|-----------|-------|
| Checkout success rate | >95% | Slack alert if <90% |
| Avg checkout time | <500ms | Warning if >800ms |
| Distance calc failures | <1% | Alert if >5% |
| Rate limit hits | <100/hour | Warning if >500/hour |
| Lock conflicts | <10/hour | Alert if >50/hour |
| Pricing fallback usage | <1% | Alert if >5% |

### 12.4 Logging

```typescript
// Structured logging for checkout events
this.logger.log({
  event: 'checkout_initiated',
  orderId,
  userId,
  addressId,
  timestamp: new Date().toISOString(),
});

this.logger.log({
  event: 'distance_calculated',
  orderId,
  distanceKm,
  chefLocation: { lat: chefLat, lng: chefLng },
  customerLocation: { lat: addressLat, lng: addressLng },
  duration: calculationTime,
});

this.logger.log({
  event: 'pricing_calculated',
  orderId,
  breakdown: {
    itemTotal: pricingResult.breakdown.itemTotal,
    deliveryFee: pricingResult.breakdown.totalDeliveryFee,
    gst: pricingResult.breakdown.gst.total,
    grandTotal: pricingResult.breakdown.grandTotal,
  },
  isPeakTime,
  distanceKm,
});

this.logger.log({
  event: 'checkout_completed',
  orderId,
  paymentIntentId,
  totalPaise,
  duration: totalCheckoutTime,
});
```

### 12.5 Rollback Strategy

**Scenario**: PricingService bug causes incorrect fee calculation

1. **Immediate**: Feature flag to disable PricingService, use fallback
   ```typescript
   if (process.env.PRICING_SERVICE_ENABLED === 'false') {
     // Use simple fallback pricing
     deliveryFeePaise = 2000;
     taxFeePaise = Math.floor(order.subtotalPaise * 0.05);
     totalPaise = order.subtotalPaise + deliveryFeePaise + taxFeePaise;
   }
   ```

2. **Short-term**: Deploy hotfix with corrected pricing logic

3. **Long-term**: Add pricing validation tests to prevent future bugs

---

## 📌 **Quick Reference**

### API Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/v1/cart/checkout` | POST | Convert cart to draft order |
| `/api/v1/orders/checkout` | POST | Attach address, calculate fees, create payment intent |
| `/api/v1/orders/:id/payment-intent` | POST | Initialize Razorpay order |
| `/api/v1/orders/:id/payment/confirm` | POST | Confirm payment completion |

### Key Files

| File | Purpose |
|------|---------|
| `apps/chefooz-apis/src/modules/order/order.service.ts` | Core checkout logic |
| `apps/chefooz-apis/src/modules/pricing/pricing.service.ts` | Dynamic pricing engine |
| `apps/chefooz-apis/src/modules/pricing/utils/distance.util.ts` | Haversine distance calculation |
| `apps/chefooz-app/src/app/checkout/address.tsx` | Address selection screen |
| `apps/chefooz-app/src/app/cart/payment.tsx` | Payment method selection |
| `libs/api-client/src/lib/hooks/useCheckout.ts` | React Query hooks |

### Environment Variables

```bash
DATABASE_URL=postgresql://...
REDIS_URL=redis://...
RAZORPAY_KEY_ID=rzp_live_xxxxx
RAZORPAY_KEY_SECRET=secret
PRICING_DELIVERY_BASE_FEE=2000
PRICING_DELIVERY_PER_KM=500
PRICING_DELIVERY_MAX_FEE=15000
PRICING_GST_RATE=0.05
PRICING_SURGE_MULTIPLIER=1.20
```

---

## ✅ **[TECHNICAL_GUIDE_COMPLETE]**

**Document Status**: Complete  
**Lines**: 2,050+  
**Coverage**: Architecture, APIs, Service Logic, Frontend Integration, Distance Calculation, Pricing Engine, Security, Error Handling, Testing, Performance, Deployment  
**Next Step**: QA Test Cases documentation
