# 🧪 **Checkout Module – QA Test Cases**

**Module:** Checkout (Integrated Cross-Module Feature)  
**Version:** 1.0  
**Last Updated:** January 2025  
**Test Environment:** Staging + Production

---

## 📋 **Table of Contents**

1. [Test Overview](#1-test-overview)
2. [Functional Tests](#2-functional-tests)
3. [Integration Tests](#3-integration-tests)
4. [Edge Cases & Boundary Tests](#4-edge-cases--boundary-tests)
5. [Validation Tests](#5-validation-tests)
6. [Performance Tests](#6-performance-tests)
7. [Security Tests](#7-security-tests)
8. [Platform-Specific Tests](#8-platform-specific-tests)
9. [Pricing Calculation Tests](#9-pricing-calculation-tests)
10. [Distance Calculation Tests](#10-distance-calculation-tests)
11. [Payment Integration Tests](#11-payment-integration-tests)
12. [Regression Tests](#12-regression-tests)

---

## 1. Test Overview

### 1.1 Test Environment Setup

**Prerequisites:**
- ✅ Staging database with test users, chefs, menu items, addresses
- ✅ Razorpay test mode credentials configured
- ✅ Redis server running (rate limiting, locks)
- ✅ Mobile app builds: iOS (TestFlight) + Android (Internal Testing)
- ✅ Test payment cards (Razorpay test mode)

**Test Data:**

> ⚠️ **Chefooz uses OTP-only authentication (no passwords)**. Login via mobile phone number + OTP sent via WhatsApp (primary) or Twilio SMS (fallback). Use the `/api/v1/auth/v2/send-otp` → `/api/v1/auth/v2/verify-otp` flow to obtain a JWT token before running tests.

**OTP Auth Flow (curl):**
```bash
# Step 1: Request OTP
curl -s -X POST https://api-staging.chefooz.com/api/v1/auth/v2/send-otp \
  -H "Content-Type: application/json" \
  -d '{"phoneNumber": "+919876543210"}' | jq '.data.requestId'

# Step 2: Verify OTP
curl -s -X POST https://api-staging.chefooz.com/api/v1/auth/v2/verify-otp \
  -H "Content-Type: application/json" \
  -d '{"requestId": "<REQUEST_ID>", "otp": "<OTP_FROM_WHATSAPP_OR_SMS>"}' | jq '.data.accessToken'

export JWT_TOKEN="<ACCESS_TOKEN_FROM_ABOVE>"
```

| Entity | Test Data |
|--------|-----------|
| **Test User 1** | Phone: `+919876543210` (2 saved addresses) |
| **Test User 2** | Phone: `+919876543211` (no addresses) |
| **Test Chef** | Phone: `+919876543212` (5 active menu items) |
| **Test Address 1** | 3 km from chef (within base fee range) |
| **Test Address 2** | 8 km from chef (distance surcharge applies) |
| **Test Address 3** | 16 km from chef (exceeds 15 km limit) |

### 1.2 Testing Tools

| Tool | Purpose |
|------|---------|
| **Postman / curl** | API endpoint testing |
| **Detox** | React Native E2E testing |
| **Jest** | Unit tests |
| **Artillery** | Load testing |
| **Charles Proxy** | Network inspection |

### 1.3 Test Coverage Goals

| Category | Target Coverage |
|----------|----------------|
| **Backend APIs** | >90% code coverage |
| **Service Methods** | >95% (critical path) |
| **Frontend Screens** | >80% user flows |
| **Edge Cases** | 100% documented scenarios |

---

## 2. Functional Tests

### Test Suite: F-CHECKOUT

---

#### **F-CHECKOUT-001: Happy Path - Complete Checkout Flow**

**Priority**: Critical  
**Type**: End-to-End

**Preconditions:**
- User logged in
- Cart has 2 items (₹400 subtotal)
- User has 1 saved address (5 km from chef)
- Current time: Off-peak (4 PM)

**Steps:**
1. Navigate to cart screen
2. Tap "Proceed to Checkout"
3. Verify address selection screen loads
4. Select existing address
5. Tap "Proceed to Payment"
6. Verify payment screen displays correct breakdown
7. Select "UPI Intent"
8. Tap "Place Order"
9. Verify Razorpay SDK opens
10. Complete payment (test card)
11. Verify order confirmation screen

**Expected Results:**
- ✅ Address selection shows 1 saved address
- ✅ Payment screen displays:
  - Subtotal: ₹400
  - Packaging: ₹10
  - Delivery: ₹30 (base ₹20 + ₹10 distance)
  - GST: ₹22
  - Grand Total: ₹462
- ✅ Razorpay SDK initializes successfully
- ✅ Order status changes: `CREATED` → `PAID`
- ✅ Confirmation screen shows order ID + ETA

**Acceptance Criteria:**
- Complete flow <3 minutes
- No errors or crashes
- Cart clears after successful checkout

---

#### **F-CHECKOUT-002: Checkout with Default Address Auto-Selected**

**Priority**: High  
**Type**: UI Validation

**Preconditions:**
- User has 3 saved addresses
- Address 2 marked as default

**Steps:**
1. Navigate to cart
2. Tap "Proceed to Checkout"
3. Observe address selection screen

**Expected Results:**
- ✅ Address 2 (default) is pre-selected (radio button checked)
- ✅ "DEFAULT" badge visible on Address 2
- ✅ User can change selection before proceeding

---

#### **F-CHECKOUT-003: Add New Address During Checkout**

**Priority**: High  
**Type**: User Flow

**Preconditions:**
- User has 1 saved address
- User wants to deliver to new location

**Steps:**
1. On address selection screen, tap "Add New Address"
2. Fill address form:
   - Label: "Work"
   - Full name: "John Doe"
   - Phone: "+91 9876543210"
   - Line 1: "123 MG Road"
   - City: "Bangalore"
   - Pincode: "560001"
3. Tap "Get Current Location" (GPS coordinates captured)
4. Tap "Save Address"
5. Verify new address appears in list
6. Select new address
7. Proceed to payment

**Expected Results:**
- ✅ Address saves successfully
- ✅ New address appears in selection list
- ✅ Coordinates auto-populate from GPS
- ✅ Distance recalculated for new address
- ✅ Delivery fee updates dynamically

---

#### **F-CHECKOUT-004: COD Payment Method**

**Priority**: High  
**Type**: Payment Flow

**Preconditions:**
- Cart ready for checkout
- Address selected

**Steps:**
1. On payment screen, select "Cash on Delivery"
2. Tap "Place Order"
3. Wait for order confirmation

**Expected Results:**
- ✅ No Razorpay SDK opens
- ✅ Order status: `CREATED` → `COD_PENDING`
- ✅ Confirmation message: "Order placed! Pay cash on delivery."
- ✅ Order appears in "Active Orders" section

**Notes:**
- COD skips payment gateway entirely
- Chef sees order marked as "COD"

---

#### **F-CHECKOUT-005: UPI Intent Flow**

**Priority**: Critical  
**Type**: Payment Integration

**Preconditions:**
- Cart ready for checkout
- PhonePe app installed on device

**Steps:**
1. Select "UPI Intent"
2. Tap "Place Order"
3. Verify PhonePe app opens (deep link)
4. Enter UPI PIN in PhonePe
5. Complete payment
6. Return to Chefooz app

**Expected Results:**
- ✅ Deep link opens PhonePe (or GPay/Paytm based on user choice)
- ✅ Payment amount matches checkout total
- ✅ After payment, Chefooz app shows success screen
- ✅ Order status: `PAID`

**Failure Scenario:**
- User cancels payment in UPI app
- Expected: Return to payment screen with error: "Payment cancelled"
- Order status remains: `CREATED`

---

#### **F-CHECKOUT-006: Checkout with Surge Pricing**

**Priority**: High  
**Type**: Dynamic Pricing

**Preconditions:**
- Cart ready
- Current time: Peak hour (1 PM, lunch)
- Address: 3 km from chef

**Steps:**
1. Navigate to payment screen
2. Observe delivery fee breakdown

**Expected Results:**
- ✅ Base delivery fee: ₹20
- ✅ Surge applied: ₹4 (20% of ₹20)
- ✅ Total delivery fee: ₹24
- ✅ Surge indicator visible: "Peak time delivery" badge

**Verification:**
- Test at 12:00 PM, 1:00 PM, 7:00 PM, 9:00 PM (peak hours)
- Compare with 4:00 PM (off-peak) — should be ₹20 (no surge)

---

#### **F-CHECKOUT-007: View Price Breakdown Modal**

**Priority**: Medium  
**Type**: UI Feature

**Preconditions:**
- On payment screen
- Condensed price summary visible

**Steps:**
1. Tap "View breakup" link below grand total
2. Observe modal

**Expected Results:**
- ✅ Modal displays:
  - Item total: ₹400
  - Packaging charges: ₹10
  - Delivery fee breakdown:
    - Base fee: ₹20
    - Distance surcharge: ₹10 (for 5 km)
    - Surge: ₹6 (if peak time)
  - GST breakdown:
    - CGST (2.5%): ₹11
    - SGST (2.5%): ₹11
  - Grand Total: ₹468
- ✅ Modal has "Close" button
- ✅ All amounts formatted as ₹XX (no decimal unless .50)

---

#### **F-CHECKOUT-008: Change Address from Payment Screen**

**Priority**: High  
**Type**: Navigation

**Preconditions:**
- On payment screen
- Address already selected

**Steps:**
1. Tap "Change" button next to delivery address
2. Verify navigation to address selection screen
3. Select different address (8 km from chef)
4. Return to payment screen

**Expected Results:**
- ✅ Address updates immediately
- ✅ Delivery fee recalculates: ₹45 (base ₹20 + ₹25 distance)
- ✅ Grand total updates
- ✅ No data loss (cart items intact)

---

#### **F-CHECKOUT-009: Order Confirmation Screen Elements**

**Priority**: High  
**Type**: UI Validation

**Preconditions:**
- Order successfully placed

**Steps:**
1. Observe confirmation screen after payment

**Expected Results:**
- ✅ Success icon (checkmark animation)
- ✅ Order ID displayed
- ✅ Estimated delivery time: "45-60 minutes"
- ✅ Chef name + avatar
- ✅ "Track Order" CTA button
- ✅ "Back to Home" button
- ✅ No option to edit order (immutable)

---

#### **F-CHECKOUT-010: Empty Cart Redirect**

**Priority**: Medium  
**Type**: Edge Case

**Preconditions:**
- User navigates to checkout URL directly
- Cart is empty

**Steps:**
1. Navigate to `/cart/payment`
2. Observe behavior

**Expected Results:**
- ✅ Alert: "Your cart is empty"
- ✅ Redirect to home screen (explore feed)
- ✅ No crash or blank screen

---

## 3. Integration Tests

### Test Suite: I-CHECKOUT

---

#### **I-CHECKOUT-001: Cart to Order Integration**

**Priority**: Critical  
**Type**: Module Integration

**Preconditions:**
- Cart has 3 items from same chef
- Cart total: ₹650

**Steps:**
1. Call `POST /api/v1/cart/checkout` (cart checkout endpoint)
2. Verify order created
3. Call `POST /api/v1/orders/checkout` with order ID + address ID
4. Verify payment intent generated

**Expected Results:**
- ✅ Order created with status: `CREATED`
- ✅ Order items match cart items (snapshots saved)
- ✅ Cart clears after checkout
- ✅ Payment intent ID generated (UUID format)
- ✅ Order total matches cart total + delivery + GST

**API Response Validation:**

```json
// Step 1 Response
{
  "success": true,
  "message": "Cart checked out successfully",
  "data": {
    "orderId": "abc-123",
    "orderStatus": "CREATED",
    "totalAmountPaise": 65000,
    "itemCount": 3
  }
}

// Step 3 Response
{
  "success": true,
  "message": "Order checked out successfully",
  "data": {
    "paymentIntentId": "pi_xyz789",
    "totalPaise": 71500
  }
}
```

---

#### **I-CHECKOUT-002: Address to Distance Calculation**

**Priority**: Critical  
**Type**: Service Integration

**Test Data:**
- Chef location: `28.6139, 77.2090` (Delhi)
- Customer address: `28.7041, 77.1025` (Delhi NCR)
- Expected distance: ~12.5 km

**Steps:**
1. Create order with items
2. Call checkout with address ID
3. Verify distance calculation
4. Verify delivery fee

**Expected Results:**
- ✅ Distance calculated: ~12.5 km (±0.5 km tolerance)
- ✅ Delivery fee: ₹67.50
  - Base: ₹20
  - Distance: (12.5 - 3) × ₹5 = ₹47.50
- ✅ `pricingBreakdown.distanceKm` stored in order

**Database Verification:**

```sql
SELECT 
  id,
  delivery_fee_paise,
  pricing_breakdown->'distanceKm' AS distance
FROM orders
WHERE id = 'abc-123';

-- Expected:
-- delivery_fee_paise: 6750
-- distance: 12.5
```

---

#### **I-CHECKOUT-003: PricingService Integration**

**Priority**: Critical  
**Type**: Service Integration

**Preconditions:**
- Order with 2 items (₹400 subtotal)
- Distance: 5 km
- Time: Peak hour (1 PM)

**Steps:**
1. Call `PricingService.calculatePricing()` with:
   - items: [{ pricePaise: 20000, quantity: 2 }]
   - delivery: { distanceKm: 5, isPeakTime: true }
2. Verify breakdown

**Expected Results:**

```typescript
{
  breakdown: {
    itemTotal: 40000, // ₹400
    packagingCharges: 1000, // ₹10
    totalDeliveryFee: 3600, // ₹36 (base ₹20 + distance ₹10, + surge 20%)
    deliveryBreakdown: {
      base: 2000,
      distanceSurcharge: 1000,
      surgeSurcharge: 600,
    },
    gst: {
      cgst: 1115,
      sgst: 1115,
      total: 2230, // ₹22.30
    },
    grandTotal: 46830, // ₹468.30
  }
}
```

---

#### **I-CHECKOUT-004: Razorpay Payment Intent Creation**

**Priority**: Critical  
**Type**: Payment Gateway Integration

**Preconditions:**
- Order checked out (address attached, fees calculated)
- Razorpay test mode enabled

**Steps:**
1. Call `POST /api/v1/orders/:orderId/payment-intent`
2. Verify Razorpay order created
3. Verify response payload

**Expected Results:**
- ✅ Razorpay API called successfully
- ✅ Response contains:
  - `razorpayOrderId`: "order_Mkl1234567890"
  - `amountPaise`: 46830
  - `currency`: "INR"
  - `keyId`: "rzp_test_xxxxx"
- ✅ Frontend can initialize Razorpay SDK with payload

**Razorpay API Validation:**

```bash
# Verify order created on Razorpay dashboard
curl https://api.razorpay.com/v1/orders/order_Mkl1234567890 \
  -u rzp_test_xxxxx:secret

# Expected response:
{
  "id": "order_Mkl1234567890",
  "entity": "order",
  "amount": 46830,
  "currency": "INR",
  "status": "created"
}
```

---

#### **I-CHECKOUT-005: Order Status History Tracking**

**Priority**: High  
**Type**: Audit Trail

**Preconditions:**
- Order created

**Steps:**
1. Call `POST /api/v1/orders/checkout` (status: `CREATED`)
2. Complete payment (status: `PAID`)
3. Query `order_status_history` table

**Expected Results:**
- ✅ 2 history entries created:

| ID | Order ID | Status | Note | Created At |
|----|----------|--------|------|------------|
| 1 | abc-123 | CREATED | Order submitted for payment | 2025-01-15 10:30:00 |
| 2 | abc-123 | PAID | Payment confirmed | 2025-01-15 10:31:23 |

---

#### **I-CHECKOUT-006: Address Snapshot Immutability**

**Priority**: High  
**Type**: Data Integrity

**Preconditions:**
- Order checked out with address ID: `addr-1`
- Address snapshot saved in order

**Steps:**
1. Verify order has `address_snapshot` JSON field populated
2. Update address in addresses table:
   ```sql
   UPDATE addresses
   SET line1 = 'New Street 456', pincode = '560002'
   WHERE id = 'addr-1';
   ```
3. Query order's `address_snapshot`

**Expected Results:**
- ✅ Order's `address_snapshot` remains unchanged:
  ```json
  {
    "label": "Home",
    "line1": "Old Street 123",
    "pincode": "560001",
    "lat": 28.6139,
    "lng": 77.2090
  }
  ```
- ✅ Address updates don't affect past orders

**Business Rule:**
- Addresses are snapshot at checkout time
- Delivery always goes to original address, even if user edits address later

---

## 4. Edge Cases & Boundary Tests

### Test Suite: E-CHECKOUT

---

#### **E-CHECKOUT-001: Distance Exactly 3 km**

**Priority**: Medium  
**Type**: Boundary Condition

**Test Data:**
- Chef location: `28.6139, 77.2090`
- Customer address: Exactly 3.00 km away

**Steps:**
1. Checkout order
2. Verify delivery fee

**Expected Results:**
- ✅ Distance: 3.00 km
- ✅ Delivery fee: ₹20 (base only, no distance surcharge)
- ✅ `deliveryBreakdown.distanceSurcharge`: 0

---

#### **E-CHECKOUT-002: Distance 3.01 km (Just Above Threshold)**

**Priority**: Medium  
**Type**: Boundary Condition

**Test Data:**
- Distance: 3.01 km

**Expected Results:**
- ✅ Distance surcharge applies: 0.01 × ₹5 = ₹0.05
- ✅ Delivery fee: ₹20.05 (rounded to ₹20 in paise: 2005)

---

#### **E-CHECKOUT-003: Distance Exceeds 15 km Limit**

**Priority**: High  
**Type**: Validation

**Test Data:**
- Distance: 16 km

**Steps:**
1. Attempt checkout with distant address

**Expected Results:**
- ✅ Error response:
  ```json
  {
    "success": false,
    "message": "Delivery distance exceeds 15km limit",
    "errorCode": "DISTANCE_EXCEEDED",
    "statusCode": 400
  }
  ```
- ✅ Order status remains `CREATED` (not progressed)
- ✅ No payment intent generated

---

#### **E-CHECKOUT-004: Distance Calculation Failure (Invalid Coordinates)**

**Priority**: High  
**Type**: Fallback Mechanism

**Test Data:**
- Chef location: `null, null` (missing coordinates)

**Steps:**
1. Attempt checkout
2. Observe fallback behavior

**Expected Results:**
- ✅ Warning logged: "Failed to calculate distance, using default: 3km"
- ✅ Delivery fee: ₹20 (base fee for 3 km default)
- ✅ Order proceeds successfully (not blocked)

---

#### **E-CHECKOUT-005: No Saved Addresses**

**Priority**: High  
**Type**: User Flow

**Preconditions:**
- New user with no addresses

**Steps:**
1. Add items to cart
2. Proceed to checkout
3. Observe address selection screen

**Expected Results:**
- ✅ Empty state UI displays:
  - Icon: Map marker crossed out
  - Text: "No Addresses Saved"
  - Subtext: "Add a delivery address to continue"
- ✅ "Add New Address" button prominent
- ✅ "Proceed to Payment" button disabled

---

#### **E-CHECKOUT-006: Single Item in Cart**

**Priority**: Medium  
**Type**: Edge Case

**Test Data:**
- Cart has 1 item (₹50)

**Steps:**
1. Checkout cart

**Expected Results:**
- ✅ Order created successfully
- ✅ Packaging: ₹10
- ✅ Delivery: ₹20
- ✅ GST: ₹4
- ✅ Grand Total: ₹84

---

#### **E-CHECKOUT-007: Cart with 100 Items (Large Order)**

**Priority**: Medium  
**Type**: Performance

**Test Data:**
- Cart has 100 items (various quantities)

**Steps:**
1. Checkout cart
2. Measure response time

**Expected Results:**
- ✅ Order creation: <2 seconds
- ✅ All 100 items saved as snapshots
- ✅ Pricing calculation accurate
- ✅ No timeout errors

---

#### **E-CHECKOUT-008: Surge Pricing at Exact Peak Hour Boundary**

**Priority**: Medium  
**Type**: Timing Edge Case

**Test Data:**
- Test at 12:00:00 PM, 12:00:30 PM, 12:59:59 PM, 1:00:00 PM, 2:00:00 PM, 2:00:01 PM

**Steps:**
1. At each timestamp, call checkout
2. Verify surge pricing status

**Expected Results:**
- ✅ 12:00:00 PM - 1:59:59 PM: Surge applies (20%)
- ✅ 2:00:00 PM onwards: No surge (off-peak)

---

#### **E-CHECKOUT-009: Zero-Price Item (Free Sample)**

**Priority**: Low  
**Type**: Edge Case

**Test Data:**
- Cart has 1 paid item (₹100) + 1 free item (₹0)

**Steps:**
1. Checkout cart

**Expected Results:**
- ✅ Subtotal: ₹100 (free item not counted)
- ✅ Delivery: ₹20
- ✅ GST: ₹6
- ✅ Total: ₹126
- ✅ Both items appear in order (free item quantity recorded)

---

#### **E-CHECKOUT-010: Address Without Coordinates (Manual Entry)**

**Priority**: Medium  
**Type**: Validation

**Test Data:**
- Address saved with lat/lng = null (user typed address without GPS)

**Steps:**
1. Attempt checkout with this address

**Expected Results:**
- ✅ Error: "Address missing coordinates. Please re-enter address with location enabled."
- ✅ User redirected to edit address screen
- ✅ Order not created

---

## 5. Validation Tests

### Test Suite: V-CHECKOUT

---

#### **V-CHECKOUT-001: Order ID Validation**

**Priority**: High  
**Type**: Input Validation

**Test Cases:**

| Input | Expected Result |
|-------|----------------|
| Valid UUID | ✅ Accepted |
| Invalid UUID format ("abc123") | ❌ 400 Bad Request: "Invalid order ID format" |
| Non-existent UUID | ❌ 404 Not Found: "Order not found" |
| Another user's order ID | ❌ 404 Not Found (ownership validation) |

**API Test:**

```bash
# Invalid format
curl -s -X POST https://api-staging.chefooz.com/api/v1/orders/checkout \
  -H "Authorization: Bearer $JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"orderId": "invalid", "addressId": "addr-1"}'

# Expected: 400 Bad Request
```

---

#### **V-CHECKOUT-002: Address ID Validation**

**Priority**: High  
**Type**: Input Validation

**Test Cases:**

| Input | Expected Result |
|-------|----------------|
| Valid address ID | ✅ Accepted |
| Invalid UUID | ❌ 400 Bad Request |
| Non-existent address | ❌ 404 Not Found: "Address not found" |
| Another user's address | ❌ 404 Not Found |

---

#### **V-CHECKOUT-003: Order Status Validation**

**Priority**: Critical  
**Type**: State Validation

**Test Data:**

| Order Status | Checkout Allowed? | Expected Result |
|--------------|------------------|----------------|
| CREATED | ✅ Yes | Proceeds normally |
| PAYMENT_PENDING | ✅ Yes | Proceeds (retrying checkout) |
| PAID | ❌ No | 403 Forbidden: "Order already paid" |
| CANCELLED | ❌ No | 403 Forbidden: "Order cancelled" |
| COMPLETED | ❌ No | 403 Forbidden: "Order already completed" |

**API Test:**

```bash
# Attempt to checkout already-paid order
curl -s -X POST https://api-staging.chefooz.com/api/v1/orders/checkout \
  -H "Authorization: Bearer $JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"orderId": "paid-order-id", "addressId": "addr-1"}'

# Expected: 403 Forbidden
```

---

#### **V-CHECKOUT-004: Cart Validation Before Checkout**

**Priority**: Critical  
**Type**: Business Rule

**Test Cases:**

| Scenario | Expected Result |
|----------|----------------|
| Cart empty | ❌ Error: "Cart is empty" |
| Menu item no longer available | ❌ Error: "Cart has validation errors" |
| Menu item price changed | ❌ Error: "Cart has validation errors" |
| Chef account deactivated | ❌ Error: "Chef not available" |

**Steps:**
1. Add items to cart
2. Admin deactivates chef
3. Attempt checkout

**Expected Results:**
- ✅ Cart validation fails
- ✅ Error message: "Chef not available for orders"
- ✅ User prompted to clear cart

---

#### **V-CHECKOUT-005: Coordinate Validation**

**Priority**: High  
**Type**: Data Validation

**Test Cases:**

| Latitude | Longitude | Valid? |
|----------|-----------|--------|
| 28.6139 | 77.2090 | ✅ Yes |
| 91.0 | 77.2090 | ❌ No (lat > 90) |
| -91.0 | 77.2090 | ❌ No (lat < -90) |
| 28.6139 | 181.0 | ❌ No (lng > 180) |
| null | 77.2090 | ❌ No (lat missing) |

**Expected Behavior:**
- Invalid coordinates trigger fallback distance (3 km)
- Warning logged: "Invalid coordinates detected"

---

## 6. Performance Tests

### Test Suite: P-CHECKOUT

---

#### **P-CHECKOUT-001: Checkout API Response Time**

**Priority**: High  
**Type**: Performance Benchmark

**Test Setup:**
- Artillery load testing tool
- 100 concurrent users
- 10 requests per user

**Performance Targets:**

| Metric | Target | Max Acceptable |
|--------|--------|----------------|
| Avg response time | <500ms | <800ms |
| P95 response time | <800ms | <1200ms |
| P99 response time | <1200ms | <2000ms |
| Error rate | <1% | <5% |

**Artillery Config:**

```yaml
config:
  target: "https://api.chefooz.com"
  phases:
    - duration: 60
      arrivalRate: 100
  processor: "./checkout-flow.js"

scenarios:
  - name: "Checkout Flow"
    flow:
      - post:
          url: "/v1/orders/checkout"
          headers:
            Authorization: "Bearer {{ token }}"
          json:
            orderId: "{{ orderId }}"
            addressId: "{{ addressId }}"
```

**Success Criteria:**
- ✅ 95% of requests complete within 800ms
- ✅ Zero 500 errors
- ✅ Database connection pool stable

---

#### **P-CHECKOUT-002: Distance Calculation Performance**

**Priority**: Medium  
**Type**: Algorithm Performance

**Test:**
- Measure time to calculate distance
- 10,000 iterations

**Code:**

```typescript
const iterations = 10000;
const start = Date.now();

for (let i = 0; i < iterations; i++) {
  calculateDistance(28.6139, 77.2090, 28.7041, 77.1025);
}

const duration = Date.now() - start;
const avgTime = duration / iterations;

console.log(`Avg time per calculation: ${avgTime}ms`);
```

**Expected Results:**
- ✅ Avg time: <1ms per calculation
- ✅ Total time for 10k calculations: <5 seconds

---

#### **P-CHECKOUT-003: PricingService Calculation Performance**

**Priority**: Medium  
**Type**: Service Performance

**Test:**
- Measure `PricingService.calculatePricing()` execution time
- 1,000 iterations with varying inputs

**Expected Results:**
- ✅ Avg time: <50ms
- ✅ P95: <100ms
- ✅ No memory leaks

---

#### **P-CHECKOUT-004: Checkout Flow End-to-End Time**

**Priority**: High  
**Type**: User Experience

**Test:**
- Measure time from "Proceed to Checkout" tap to order confirmation
- Mobile device: iPhone 12 Pro
- Network: 4G (100 Mbps)

**Steps:**
1. Start timer
2. Tap "Proceed to Checkout"
3. Select address (1 second)
4. Tap "Proceed to Payment"
5. Review order (5 seconds)
6. Tap "Place Order"
7. Complete UPI payment (20 seconds)
8. Stop timer at confirmation screen

**Performance Targets:**

| Scenario | Target | Max Acceptable |
|----------|--------|----------------|
| **First-time user** | <2 minutes | <3 minutes |
| **Repeat user (saved address)** | <45 seconds | <60 seconds |
| **COD order** | <30 seconds | <45 seconds |

---

## 7. Security Tests

### Test Suite: S-CHECKOUT

---

#### **S-CHECKOUT-001: Authentication Required**

**Priority**: Critical  
**Type**: Security

**Test:**
1. Call checkout API without `Authorization` header

**Expected Results:**
- ✅ 401 Unauthorized
- ✅ Response: "User not authenticated"

---

#### **S-CHECKOUT-002: JWT Token Validation**

**Priority**: Critical  
**Type**: Security

**Test Cases:**

| Token | Expected Result |
|-------|----------------|
| Valid JWT | ✅ 200 OK |
| Expired JWT | ❌ 401 Unauthorized: "Token expired" |
| Invalid signature | ❌ 401 Unauthorized: "Invalid token" |
| Malformed JWT | ❌ 401 Unauthorized |

---

#### **S-CHECKOUT-003: Rate Limiting**

**Priority**: High  
**Type**: Security

**Test:**
1. Make 11 consecutive checkout requests within 60 seconds

**Expected Results:**
- ✅ First 10 requests: 200 OK
- ✅ 11th request: 429 Too Many Requests
- ✅ Response: "Rate limit exceeded. Try again in 60 seconds."

---

#### **S-CHECKOUT-004: Distributed Lock (Prevent Concurrent Checkouts)**

**Priority**: High  
**Type**: Concurrency Control

**Test:**
1. Initiate checkout for order ID `abc-123`
2. Before first request completes, initiate second checkout for same order ID

**Expected Results:**
- ✅ First request: Acquires lock, proceeds normally
- ✅ Second request: 409 Conflict: "Operation already in progress. Please wait."
- ✅ After first request completes, lock released

---

#### **S-CHECKOUT-005: Ownership Validation**

**Priority**: Critical  
**Type**: Authorization

**Test:**
1. User A creates order (ID: `order-a`)
2. User B attempts to checkout order `order-a` (using User B's JWT)

**Expected Results:**
- ✅ 404 Not Found (order doesn't exist for User B)
- ✅ Database queried with `WHERE order_id = ? AND user_id = ?`

---

#### **S-CHECKOUT-006: SQL Injection Prevention**

**Priority**: Critical  
**Type**: Security

**Test:**
1. Inject SQL in order ID:
   ```json
   {
     "orderId": "abc-123' OR '1'='1",
     "addressId": "addr-1"
   }
   ```

**Expected Results:**
- ✅ 400 Bad Request: "Invalid order ID format"
- ✅ No SQL executed (parameterized queries used)

---

## 8. Platform-Specific Tests

### Test Suite: PL-CHECKOUT

---

#### **PL-CHECKOUT-001: iOS Address Picker**

**Priority**: High  
**Type**: Platform

**Device**: iPhone 13 Pro (iOS 17)

**Steps:**
1. On address selection screen, tap address card
2. Observe visual feedback

**Expected Results:**
- ✅ Radio button animates smoothly
- ✅ Selected card has orange border
- ✅ Haptic feedback on selection (iOS)

---

#### **PL-CHECKOUT-002: Android Back Button Behavior**

**Priority**: High  
**Type**: Platform

**Device**: Samsung Galaxy S23 (Android 14)

**Steps:**
1. On payment screen, press hardware back button

**Expected Results:**
- ✅ Navigate back to address selection screen
- ✅ Cart data intact
- ✅ No data loss

---

#### **PL-CHECKOUT-003: iOS UPI Deep Link**

**Priority**: Critical  
**Type**: Payment Integration

**Device**: iPhone (PhonePe installed)

**Steps:**
1. Select "UPI Intent"
2. Tap "Place Order"
3. Observe app switching

**Expected Results:**
- ✅ PhonePe app opens via deep link
- ✅ Payment amount pre-filled
- ✅ After payment, Chefooz app resumes (not killed)

---

#### **PL-CHECKOUT-004: Android UPI Deep Link**

**Priority**: Critical  
**Type**: Payment Integration

**Device**: Android (GPay installed)

**Steps:**
1. Select "UPI Intent"
2. Tap "Place Order"
3. Observe app switching

**Expected Results:**
- ✅ GPay app opens
- ✅ Payment UPI ID visible
- ✅ After payment, Chefooz returns to foreground

---

#### **PL-CHECKOUT-005: iOS Keyboard Handling (Add Address)**

**Priority**: Medium  
**Type**: UI

**Steps:**
1. Tap "Add New Address"
2. Focus on "Line 1" input field
3. Observe keyboard

**Expected Results:**
- ✅ Keyboard pushes form up (not covering inputs)
- ✅ "Next" button on keyboard navigates to next field
- ✅ "Done" dismisses keyboard

---

#### **PL-CHECKOUT-006: Android Keyboard Handling**

**Priority**: Medium  
**Type**: UI

**Steps:**
1. On address form, type address
2. Press back button (hardware)

**Expected Results:**
- ✅ Keyboard dismisses (form remains visible)
- ✅ Second back press navigates away

---

## 9. Pricing Calculation Tests

### Test Suite: PR-CHECKOUT

---

#### **PR-CHECKOUT-001: Base Delivery Fee (0-3 km)**

**Test Data:**

| Distance | Expected Fee |
|----------|-------------|
| 0 km | ₹20 |
| 1 km | ₹20 |
| 2 km | ₹20 |
| 3 km | ₹20 |

---

#### **PR-CHECKOUT-002: Distance Surcharge (3+ km)**

**Test Data:**

| Distance | Calculation | Expected Fee |
|----------|-------------|-------------|
| 4 km | ₹20 + (1 × ₹5) | ₹25 |
| 5 km | ₹20 + (2 × ₹5) | ₹30 |
| 8 km | ₹20 + (5 × ₹5) | ₹45 |
| 10 km | ₹20 + (7 × ₹5) | ₹55 |
| 15 km | ₹20 + (12 × ₹5) | ₹80 |

---

#### **PR-CHECKOUT-003: Surge Pricing (20% Peak Hours)**

**Test Data:**

| Base Fee | Surge? | Expected Fee |
|----------|--------|-------------|
| ₹20 | No | ₹20 |
| ₹20 | Yes | ₹24 |
| ₹45 | Yes | ₹54 |
| ₹80 | Yes | ₹96 |

**Peak Hours**: 7-10 AM, 12-2 PM, 7-10 PM

---

#### **PR-CHECKOUT-004: Max Delivery Fee Cap (₹150)**

**Test Data:**

| Distance | Calculated Fee | Capped Fee |
|----------|---------------|-----------|
| 20 km | ₹20 + (17 × ₹5) = ₹105 | ₹105 |
| 25 km | ₹20 + (22 × ₹5) = ₹130 | ₹130 |
| 30 km | ₹20 + (27 × ₹5) = ₹155 | ₹150 (capped) |
| 40 km | ₹20 + (37 × ₹5) = ₹205 | ₹150 (capped) |

---

#### **PR-CHECKOUT-005: GST Calculation (5%)**

**Test Data:**

| Subtotal | GST (5%) | Total |
|----------|---------|-------|
| ₹100 | ₹5 | ₹105 |
| ₹400 | ₹20 | ₹420 |
| ₹430 (incl. packaging + delivery) | ₹21.50 | ₹451.50 |

**GST Split:**
- CGST: 2.5%
- SGST: 2.5%

---

#### **PR-CHECKOUT-006: Packaging Charges**

**Test Data:**

| Order | Packaging Fee |
|-------|--------------|
| 1 item | ₹10 |
| 5 items | ₹10 |
| 100 items | ₹10 |

**Rule**: Flat ₹10 per order (not per item)

---

#### **PR-CHECKOUT-007: Complete Pricing Breakdown Example**

**Scenario:**
- Items: 3 × ₹150 = ₹450
- Distance: 8 km
- Time: 1 PM (peak)

**Calculation:**

```
Item Total:       ₹450.00
Packaging:        ₹ 10.00
Delivery:
  Base:           ₹ 20.00
  Distance:       ₹ 25.00 (5 km × ₹5)
  Surge (20%):    ₹  9.00
  Total Delivery: ₹ 54.00
---
Subtotal:         ₹514.00
GST (5%):         ₹ 25.70
---
GRAND TOTAL:      ₹539.70
```

**Expected Response:**

```json
{
  "breakdown": {
    "itemTotal": 45000,
    "packagingCharges": 1000,
    "totalDeliveryFee": 5400,
    "deliveryBreakdown": {
      "base": 2000,
      "distanceSurcharge": 2500,
      "surgeSurcharge": 900
    },
    "gst": {
      "cgst": 1285,
      "sgst": 1285,
      "total": 2570
    },
    "grandTotal": 53970
  }
}
```

---

## 10. Distance Calculation Tests

### Test Suite: D-CHECKOUT

---

#### **D-CHECKOUT-001: Same Location (0 km)**

**Test Data:**
- Point A: `28.6139, 77.2090`
- Point B: `28.6139, 77.2090`

**Expected:**
- Distance: 0 km

---

#### **D-CHECKOUT-002: Known Distances**

**Test Data:**

| From | To | Expected Distance |
|------|-----|------------------|
| Delhi (28.6139, 77.2090) | Gurgaon (28.4595, 77.0266) | ~23 km |
| Mumbai (19.0760, 72.8777) | Pune (18.5204, 73.8567) | ~148 km |
| Bangalore (12.9716, 77.5946) | Mysore (12.2958, 76.6394) | ~143 km |

**Tolerance:** ±2 km (Haversine vs actual road distance)

---

#### **D-CHECKOUT-003: Coordinate Validation**

**Test Cases:**

| Lat | Lng | Valid? | Error |
|-----|-----|--------|-------|
| 28.6139 | 77.2090 | ✅ Yes | - |
| 91.0 | 77.2090 | ❌ No | "Latitude out of range" |
| 28.6139 | 181.0 | ❌ No | "Longitude out of range" |
| null | 77.2090 | ❌ No | "Missing coordinates" |

---

#### **D-CHECKOUT-004: Fallback Distance (3 km Default)**

**Test:**
1. Chef location missing coordinates
2. Attempt checkout

**Expected:**
- ✅ Warning logged: "Failed to calculate distance, using default: 3km"
- ✅ Delivery fee: ₹20 (base fee for 3 km)

---

## 11. Payment Integration Tests

### Test Suite: PAY-CHECKOUT

---

#### **PAY-CHECKOUT-001: Razorpay Test Card - Success**

**Priority**: Critical  
**Type**: Payment Gateway

**Test Data:**
- Card: 4111 1111 1111 1111 (Visa test card)
- CVV: 123
- Expiry: 12/25

**Steps:**
1. Checkout order
2. Select "Cards/Wallets"
3. Enter test card details in Razorpay SDK
4. Complete payment

**Expected Results:**
- ✅ Payment succeeds
- ✅ Order status: `PAID`
- ✅ Razorpay webhook fired: `payment.captured`

---

#### **PAY-CHECKOUT-002: Razorpay Test Card - Failure**

**Priority**: High  
**Type**: Payment Gateway

**Test Data:**
- Card: 4000 0000 0000 0002 (Decline test card)

**Steps:**
1. Checkout order
2. Enter failing test card
3. Attempt payment

**Expected Results:**
- ✅ Payment fails
- ✅ Error: "Payment declined"
- ✅ Order status remains: `CREATED`
- ✅ User can retry payment

---

#### **PAY-CHECKOUT-003: UPI Intent - User Cancels**

**Priority**: High  
**Type**: UPI Flow

**Steps:**
1. Select "UPI Intent"
2. PhonePe app opens
3. User presses "Cancel" in PhonePe

**Expected Results:**
- ✅ Return to Chefooz app
- ✅ Error message: "Payment cancelled. Please try again."
- ✅ Order status: `CREATED` (not changed)

---

#### **PAY-CHECKOUT-004: COD Order Confirmation**

**Priority**: High  
**Type**: Payment Method

**Steps:**
1. Select "Cash on Delivery"
2. Tap "Place Order"

**Expected Results:**
- ✅ No Razorpay SDK
- ✅ Order status: `COD_PENDING`
- ✅ Confirmation: "Order placed! Pay ₹XXX on delivery."

---

## 12. Regression Tests

### Test Suite: R-CHECKOUT

---

#### **R-CHECKOUT-001: Cart Clears After Checkout**

**Bug History**: Cart items persisted after successful checkout

**Test:**
1. Add items to cart
2. Complete checkout
3. Navigate back to cart screen

**Expected:**
- ✅ Cart empty
- ✅ Empty state UI displayed

---

#### **R-CHECKOUT-002: Address Snapshot Preserved**

**Bug History**: Edited addresses affected past orders

**Test:**
1. Checkout order with address `addr-1`
2. Edit `addr-1` to new pincode
3. View order details

**Expected:**
- ✅ Order shows original address (snapshot)
- ✅ New orders use updated address

---

#### **R-CHECKOUT-003: Distance Fallback When Chef Location Missing**

**Bug History**: Checkout failed when chef location null

**Test:**
1. Create chef with no location
2. Attempt checkout

**Expected:**
- ✅ Fallback distance: 3 km
- ✅ Order proceeds normally

---

#### **R-CHECKOUT-004: Surge Pricing Only During Peak Hours**

**Bug History**: Surge applied 24/7

**Test:**
1. Checkout at 4 PM (off-peak)
2. Verify no surge

**Expected:**
- ✅ Delivery fee: ₹20 (no surge)

---

#### **R-CHECKOUT-005: Rate Limiting Per User**

**Bug History**: Rate limit was global, not per-user

**Test:**
1. User A makes 10 checkout requests (rate limit reached)
2. User B makes 1 checkout request

**Expected:**
- ✅ User B's request succeeds (separate rate limit)

---

## 📊 **Test Summary Table**

| Test Suite | Total Cases | Priority | Status |
|------------|-------------|----------|--------|
| **Functional** | 10 | Critical | ⏳ Pending |
| **Integration** | 6 | Critical | ⏳ Pending |
| **Edge Cases** | 10 | High | ⏳ Pending |
| **Validation** | 5 | High | ⏳ Pending |
| **Performance** | 4 | High | ⏳ Pending |
| **Security** | 6 | Critical | ⏳ Pending |
| **Platform** | 6 | High | ⏳ Pending |
| **Pricing** | 7 | Medium | ⏳ Pending |
| **Distance** | 4 | Medium | ⏳ Pending |
| **Payment** | 4 | Critical | ⏳ Pending |
| **Regression** | 5 | High | ⏳ Pending |
| **TOTAL** | **67** | - | - |

---

## 🎯 **Test Execution Checklist**

### Pre-Launch (Staging)

- [ ] All functional tests pass (F-CHECKOUT-001 to 010)
- [ ] Integration tests validated (I-CHECKOUT-001 to 006)
- [ ] Edge cases covered (E-CHECKOUT-001 to 010)
- [ ] Performance benchmarks met (P-CHECKOUT-001 to 004)
- [ ] Security tests pass (S-CHECKOUT-001 to 006)
- [ ] Platform-specific tests (iOS + Android)
- [ ] Payment integration tests (Razorpay test mode)

### Pre-Production

- [ ] Regression suite clean (no regressions)
- [ ] Load testing: 100 concurrent users (no failures)
- [ ] Razorpay production credentials configured
- [ ] Rate limiting tuned (10 req/min validated)
- [ ] Monitoring dashboards configured
- [ ] Rollback plan documented

---

## ✅ **[QA_TEST_CASES_COMPLETE]**

**Document Status**: Complete  
**Total Test Cases**: 67  
**Coverage Areas**: Functional, Integration, Edge Cases, Validation, Performance, Security, Platform, Pricing, Distance, Payment, Regression  
**Next Step**: Create MODULE_COMPLETE.md summary document

---

## Dark Mode Regression Tests (Added March 2026)

### TC-CHECKOUT-DM-001: Checkout Prepare Screen in Dark Mode

**Type:** Bug Regression / Manual  
**Feature area:** Checkout (`app/checkout/prepare.tsx`)  
**Priority:** P0

**Preconditions:**
- Device set to dark appearance
- Items in cart

**Steps:**
1. Add items to cart and proceed to checkout in dark mode
2. Observe: container background, header, order summary cards, change-card buttons, warning banners, info boxes, back button, footer

**Expected result:** Background `#0A0A0A`, cards `#1C1C1E`, warning banners tinted (`colors.warning + '30'`), info boxes tinted (`colors.info + '20'`), all text readable  
**Actual result (before fix):** White screen with white cards, banners, and footer  
**Fix applied:** Converted `StyleSheet.create` to `makeStyles(colors)` factory; all 20+ hardcoded hex values replaced  
**Regression test:** `apps/chefooz-app/src/app/checkout/prepare.tsx`  
**Status:** Fixed ✅

---

**Last Updated**: March 2026
