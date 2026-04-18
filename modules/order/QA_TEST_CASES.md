# 🧪 Order Module - QA Test Cases

**Module**: Order Management  
**Version**: 2.0 (Multi-Item Orders + Cart Integration)  
**Last Updated**: 2026-02-15  
**Test Coverage**: Comprehensive (7 test types × 12 functional areas)

---

## 📋 **Table of Contents**

1. [Test Environment Setup](#test-environment-setup)
2. [Functional Tests](#functional-tests)
3. [UX/UI Tests](#uxui-tests)
4. [Edge Case Tests](#edge-case-tests)
5. [Error Handling Tests](#error-handling-tests)
6. [Security Tests](#security-tests)
7. [Performance Tests](#performance-tests)
8. [Regression Tests](#regression-tests)
9. [Platform-Specific Tests](#platform-specific-tests)
10. [Test Data](#test-data)

---

## 🛠️ **Test Environment Setup**

### Prerequisites

**Backend**:
- [ ] PostgreSQL database with test data
- [ ] MongoDB with test reel data
- [ ] Valkey/Redis instance for caching
- [ ] Razorpay test mode credentials
- [ ] Backend API: `https://api-staging.chefooz.com` (staging)

**Frontend**:
- [ ] Expo Go or development build installed
- [ ] iOS simulator (14.0+) and/or Android emulator (API 28+)
- [ ] Test user accounts created

**Test Accounts**:
```
Customer 1:
- Phone: +919876543210
- Trust State: NORMAL

Customer 2 (Restricted):
- Phone: +919876543211
- Trust State: RESTRICTED

Chef:
- Phone: +919876543212
- Kitchen: Active, Accepting Orders

Rider:
- Phone: +919876543213
```

---

## ✅ **Functional Tests**

Tests that verify core business logic works as designed.

---

### Test Case: TC-ORDER-F001 — Create Draft Order from Reel CTA

**Priority**: Critical  
**Type**: Functional  
**Roles**: Customer  
**Platform**: Both (iOS & Android)

**Prerequisites:**
- User logged in via OTP (Phone: +919876543210)
- Chef has active menu items
- Chef kitchen is accepting orders

**Test Data:**
- Menu Item: Margherita Pizza (₹250)
- Quantity: 2
- Reel ID: test-reel-uuid-001

**Steps:**
1. Open Chefooz app
2. Navigate to Feed/Explore
3. Watch a reel with "Order Now" CTA
4. Tap "Order Now" button
5. Select quantity: 2
6. Add customizations (optional)
7. Tap "Add to Cart"
8. Tap "Proceed to Checkout"

**Expected Result:**
- ✅ Order created with status = `created`
- ✅ Order ID returned
- ✅ Subtotal = ₹500 (2 × ₹250)
- ✅ Attribution includes `linkedReelId`
- ✅ Order visible in "My Orders"

**Actual Result:**
[To be filled during testing]

**Status:**
[ ] Not Tested | [ ] Pass | [ ] Fail

---

### Test Case: TC-ORDER-F002 — Checkout Order with Address

**Priority**: Critical  
**Type**: Functional  
**Roles**: Customer  
**Platform**: Both

**Prerequisites:**
- Draft order exists (from TC-ORDER-F001)
- User has saved delivery address

**Test Data:**
- Order ID: [From previous test]
- Address: Home (123, MG Road, Bengaluru)

**Steps:**
1. Navigate to "My Orders"
2. Tap on draft order
3. Tap "Proceed to Checkout"
4. Select delivery address
5. Review pricing breakdown:
   - Subtotal: ₹500
   - Packaging: ₹10
   - Delivery: ₹30 (distance-based)
   - GST: ₹27
   - Total: ₹567
6. Tap "Continue to Payment"

**Expected Result:**
- ✅ Order status remains `created`
- ✅ Payment status = `pending`
- ✅ Address snapshot saved in order
- ✅ Pricing breakdown calculated correctly
- ✅ 15-minute payment lock created
- ✅ Navigate to payment screen

**Actual Result:**
[To be filled during testing]

**Status:**
[ ] Not Tested | [ ] Pass | [ ] Fail

---

### Test Case: TC-ORDER-F003 — UPI Intent Payment Flow

**Priority**: Critical  
**Type**: Functional  
**Roles**: Customer  
**Platform**: Both

**Prerequisites:**
- Order checked out (from TC-ORDER-F002)
- UPI app installed (GPay, PhonePe, or BHIM)
- Internet connection active

**Test Data:**
- Payment Method: UPI_INTENT

**Steps:**
1. On payment screen, select "UPI Intent"
2. Tap "Pay ₹567"
3. UPI app opens automatically (GPay/PhonePe)
4. Verify order details in UPI app
5. Enter UPI PIN and confirm payment
6. Return to Chefooz app

**Expected Result:**
- ✅ UPI Intent URL generated
- ✅ UPI app opens with correct amount
- ✅ Payment intent created with `razorpayOrderId`
- ✅ Order status changes to `paid` after payment
- ✅ Payment status = `paid`
- ✅ Push notification: "Order confirmed"
- ✅ Chef receives notification: "New order from [Customer]"

**Actual Result:**
[To be filled during testing]

**Status:**
[ ] Not Tested | [ ] Pass | [ ] Fail

**Notes:**
- Test with multiple UPI apps (GPay, PhonePe, BHIM)
- Verify Razorpay signature validation

---

### Test Case: TC-ORDER-F004 — Cash on Delivery (COD) Flow

**Priority**: High  
**Type**: Functional  
**Roles**: Customer  
**Platform**: Both

**Prerequisites:**
- Order checked out (from TC-ORDER-F002)
- User trust state = NORMAL (COD allowed)
- COD daily limit not exceeded (<3 orders)

**Test Data:**
- Payment Method: COD

**Steps:**
1. On payment screen, select "Cash on Delivery"
2. Tap "Confirm Order"
3. View order confirmation

**Expected Result:**
- ✅ Order status = `paid` (auto-confirmed)
- ✅ Payment status = `paid`
- ✅ Payment method = `COD`
- ✅ No UPI flow initiated
- ✅ Push notification: "Order confirmed"
- ✅ Chef receives notification immediately
- ✅ Navigate to order details screen

**Actual Result:**
[To be filled during testing]

**Status:**
[ ] Not Tested | [ ] Pass | [ ] Fail

---

### Test Case: TC-ORDER-F005 — Order History Pagination

**Priority**: High  
**Type**: Functional  
**Roles**: Customer  
**Platform**: Both

**Prerequisites:**
- User has >20 orders in history

**Steps:**
1. Navigate to "My Orders"
2. Scroll to bottom of list
3. Observe loading indicator
4. View next page of orders
5. Repeat until no more orders

**Expected Result:**
- ✅ Orders displayed in reverse chronological order
- ✅ 20 orders per page (default)
- ✅ Smooth infinite scroll
- ✅ No duplicate orders
- ✅ No skipped orders
- ✅ "No more orders" message when exhausted

**Actual Result:**
[To be filled during testing]

**Status:**
[ ] Not Tested | [ ] Pass | [ ] Fail

---

### Test Case: TC-ORDER-F006 — Reorder to Cart

**Priority**: High  
**Type**: Functional  
**Roles**: Customer  
**Platform**: Both

**Prerequisites:**
- User has delivered order in history
- Menu items still available

**Test Data:**
- Previous Order: Margherita Pizza (2x) + Garlic Bread (1x)

**Steps:**
1. Navigate to "My Orders"
2. Find previous delivered order
3. Tap "Reorder" button
4. View cart

**Expected Result:**
- ✅ Cart cleared before adding items
- ✅ Available items added to cart
- ✅ Uses **current pricing** (not historical)
- ✅ Toast: "2 items added to cart"
- ✅ If items unavailable: "1 item no longer available"
- ✅ Navigate to cart screen

**Actual Result:**
[To be filled during testing]

**Status:**
[ ] Not Tested | [ ] Pass | [ ] Fail

**Notes:**
- Test with partially unavailable items
- Verify pricing uses current menu prices

---

### Test Case: TC-ORDER-F007 — Live Order Tracking

**Priority**: High  
**Type**: Functional  
**Roles**: Customer  
**Platform**: Both

**Prerequisites:**
- Order in `out_for_delivery` status
- Rider assigned

**Steps:**
1. Navigate to "My Orders"
2. Tap on active order
3. Tap "Track Order"
4. View live status screen
5. Observe status updates every 30 seconds

**Expected Result:**
- ✅ Status badge shows "Out for Delivery" (blue)
- ✅ Delivery ETA displayed (e.g., "Arriving in 10 mins")
- ✅ Rider information card:
  - Name: Suresh Kumar
  - Phone: +919876543213
  - Profile picture
- ✅ Status timeline (vertical):
  - ✅ Order Placed
  - ✅ Payment Confirmed
  - ✅ Chef Preparing
  - ✅ Out for Delivery (current)
  - ⏳ Delivered (pending)
- ✅ "Call Rider" button functional
- ✅ Auto-refresh every 30 seconds

**Actual Result:**
[To be filled during testing]

**Status:**
[ ] Not Tested | [ ] Pass | [ ] Fail

---

### Test Case: TC-ORDER-F008 — Order Delivered & Coin Rewards

**Priority**: Critical  
**Type**: Functional  
**Roles**: Customer, Rider  
**Platform**: Both

**Prerequisites:**
- Order in `out_for_delivery` status
- Rider at delivery location

**Test Data:**
- Order Total: ₹567
- Food Value: ₹500

**Steps:**
1. **Rider**: Mark order as "Delivered" in rider app
2. **Customer**: Receive delivery notification
3. **Customer**: Open order details
4. **Customer**: Check coin balance

**Expected Result:**
- ✅ Order status = `delivered`
- ✅ Delivery status = `DELIVERED`
- ✅ Push notification: "Order delivered! You earned 500 coins"
- ✅ Coin balance increased by 500 (₹500 food value)
- ✅ "Rate Order" button appears
- ✅ Commission calculated for chef (if reel attribution)

**Actual Result:**
[To be filled during testing]

**Status:**
[ ] Not Tested | [ ] Pass | [ ] Fail

**Notes:**
- Verify coin amount = food value only (excludes delivery/tax)
- Check commission calculated correctly (8% of food value)

---

### Test Case: TC-ORDER-F009 — Order Cancellation (Before Pickup)

**Priority**: High  
**Type**: Functional  
**Roles**: Customer  
**Platform**: Both

**Prerequisites:**
- Order in `paid` or `preparing` status
- Order not yet picked up by rider

**Steps:**
1. Navigate to "My Orders"
2. Tap on active order
3. Tap "Cancel Order"
4. Confirm cancellation reason: "Changed my mind"
5. Tap "Confirm Cancellation"

**Expected Result:**
- ✅ Order status = `cancelled`
- ✅ Refund initiated automatically (if paid via UPI)
- ✅ Push notification: "Order cancelled. Refund processed."
- ✅ Chef notified: "Customer cancelled order"
- ✅ Trust state penalty (if NORMAL → WARNING after multiple cancellations)
- ✅ Cancellation logged to `order_status_history`

**Actual Result:**
[To be filled during testing]

**Status:**
[ ] Not Tested | [ ] Pass | [ ] Fail

---

### Test Case: TC-ORDER-F010 — Auto-Cancellation (Payment Timeout)

**Priority**: High  
**Type**: Functional  
**Roles**: System (Automated)  
**Platform**: Backend

**Prerequisites:**
- Order created with payment lock
- Payment not completed within 15 minutes

**Steps:**
1. Create order and checkout
2. Do NOT complete payment
3. Wait 15 minutes
4. Check order status

**Expected Result:**
- ✅ Order status = `cancelled`
- ✅ Reason: "Payment not completed within time limit"
- ✅ Push notification: "Order expired. Please create a new order."
- ✅ Inventory released back to chef
- ✅ No trust state penalty (system action)

**Actual Result:**
[To be filled during testing]

**Status:**
[ ] Not Tested | [ ] Pass | [ ] Fail

---

### Test Case: TC-ORDER-F011 — Auto-Cancellation (Rider Not Accepted)

**Priority**: High  
**Type**: Functional  
**Roles**: System (Automated)  
**Platform**: Backend

**Prerequisites:**
- Order in `ready` status
- No rider accepts delivery within 5 minutes

**Steps:**
1. Chef marks order as "Ready"
2. System assigns delivery requests to riders
3. No rider accepts within 5 minutes
4. Check order status

**Expected Result:**
- ✅ Order status = `cancelled`
- ✅ Reason: "No rider accepted the delivery"
- ✅ Push notification: "Order cancelled. Refund processed."
- ✅ Automatic refund initiated
- ✅ Chef notified: "Delivery unavailable, order cancelled"

**Actual Result:**
[To be filled during testing]

**Status:**
[ ] Not Tested | [ ] Pass | [ ] Fail

---

### Test Case: TC-ORDER-F012 — Commission Attribution (Reel-Linked Order)

**Priority**: Medium  
**Type**: Functional  
**Roles**: Customer, Chef (Creator)  
**Platform**: Both

**Prerequisites:**
- Chef has uploaded reel with linked order
- Customer orders from that reel

**Test Data:**
- Reel ID: test-reel-uuid-001
- Creator Order Value: ₹500

**Steps:**
1. Customer places order via reel CTA
2. Order delivered successfully
3. Check chef's commission balance

**Expected Result:**
- ✅ Order attribution includes:
  - `linkedReelId`: test-reel-uuid-001
  - `creatorUserId`: chef-uuid
  - `creatorOrderValue`: 50000 (₹500 in paise)
- ✅ Commission calculated: ₹40 (8% of ₹500)
- ✅ Commission visible in chef's dashboard
- ✅ Commission status = `pending` (pending payout)

**Actual Result:**
[To be filled during testing]

**Status:**
[ ] Not Tested | [ ] Pass | [ ] Fail

---

## 🎨 **UX/UI Tests**

Tests that verify user experience quality and interface consistency.

---

### Test Case: TC-ORDER-UX001 — Order History Loading States

**Priority**: High  
**Type**: UX  
**Roles**: Customer  
**Platform**: Both

**Steps:**
1. Navigate to "My Orders"
2. Observe initial loading
3. Scroll to trigger pagination

**Expected Result:**
- ✅ Skeleton loaders shown during initial load
- ✅ Shimmer effect on order cards
- ✅ No layout shift when data loads
- ✅ Smooth scroll during pagination
- ✅ "Loading more..." indicator at bottom
- ✅ No blank screen or jarring transitions

**Actual Result:**
[To be filled during testing]

**Status:**
[ ] Not Tested | [ ] Pass | [ ] Fail

---

### Test Case: TC-ORDER-UX002 — Payment Method Selection UI

**Priority**: High  
**Type**: UX  
**Roles**: Customer  
**Platform**: Both

**Steps:**
1. On payment screen, view payment options
2. Tap "UPI Intent" option
3. Tap "Cash on Delivery" option

**Expected Result:**
- ✅ Payment method cards visually distinct
- ✅ Selected option highlighted (border + checkmark)
- ✅ Icons for each method (UPI logo, cash icon)
- ✅ Short descriptions:
  - UPI: "Pay instantly via UPI apps"
  - COD: "Pay cash upon delivery"
- ✅ Disabled state for restricted COD (grayed out)
- ✅ Touch target size ≥44pt (accessibility)

**Actual Result:**
[To be filled during testing]

**Status:**
[ ] Not Tested | [ ] Pass | [ ] Fail

---

### Test Case: TC-ORDER-UX003 — Order Status Badge Color Coding

**Priority**: Medium  
**Type**: UX  
**Roles**: Customer  
**Platform**: Both

**Steps:**
1. View order history
2. Observe status badges for different order states

**Expected Result:**
- ✅ Color coding:
  - 🟡 `preparing` → Yellow
  - 🔵 `out_for_delivery` → Blue
  - 🟢 `delivered` → Green
  - 🔴 `cancelled` → Red
  - ⚪ `payment_pending` → Gray
- ✅ Status text readable on badge background
- ✅ Consistent badge style across screens

**Actual Result:**
[To be filled during testing]

**Status:**
[ ] Not Tested | [ ] Pass | [ ] Fail

---

### Test Case: TC-ORDER-UX004 — Empty State (No Orders)

**Priority**: Medium  
**Type**: UX  
**Roles**: New Customer  
**Platform**: Both

**Steps:**
1. Log in as new user (no orders)
2. Navigate to "My Orders"

**Expected Result:**
- ✅ Empty state illustration (e.g., empty box icon)
- ✅ Message: "No orders yet"
- ✅ Subtext: "Discover chefs and place your first order"
- ✅ "Explore Now" CTA button
- ✅ Button navigates to Explore/Feed

**Actual Result:**
[To be filled during testing]

**Status:**
[ ] Not Tested | [ ] Pass | [ ] Fail

---

### Test Case: TC-ORDER-UX005 — Order Details Pricing Breakdown

**Priority**: Medium  
**Type**: UX  
**Roles**: Customer  
**Platform**: Both

**Steps:**
1. Open order details
2. Scroll to pricing section

**Expected Result:**
- ✅ Clear itemized breakdown:
  ```
  Subtotal (Items)       ₹500.00
  Packaging fee          ₹10.00
  Delivery fee           ₹30.00
  GST (5%)               ₹27.00
  ─────────────────────────────
  Total                  ₹567.00
  ```
- ✅ Each line right-aligned
- ✅ Total row bold and larger font
- ✅ Expandable GST breakdown (CGST + SGST)

**Actual Result:**
[To be filled during testing]

**Status:**
[ ] Not Tested | [ ] Pass | [ ] Fail

---

## 🔬 **Edge Case Tests**

Tests that verify behavior in boundary conditions and unusual scenarios.

---

### Test Case: TC-ORDER-EDGE001 — Order with 20 Items (Maximum)

**Priority**: Medium  
**Type**: Edge Case  
**Roles**: Customer  
**Platform**: Both

**Prerequisites:**
- Chef has ≥20 menu items available

**Steps:**
1. Add 20 different items to cart
2. Attempt to add 21st item

**Expected Result:**
- ✅ 20 items added successfully
- ✅ 21st item blocked with error:
  - Message: "Maximum 20 items per order"
  - Toast notification
- ✅ Order creation successful with 20 items
- ✅ No performance degradation

**Actual Result:**
[To be filled during testing]

**Status:**
[ ] Not Tested | [ ] Pass | [ ] Fail

---

### Test Case: TC-ORDER-EDGE002 — Order Below Minimum Value

**Priority**: Medium  
**Type**: Edge Case  
**Roles**: Customer  
**Platform**: Both

**Prerequisites:**
- Menu item priced at ₹30 (below ₹50 minimum)

**Steps:**
1. Add single ₹30 item to cart
2. Attempt to checkout

**Expected Result:**
- ✅ Checkout blocked
- ✅ Error message: "Minimum order value is ₹50"
- ✅ Suggestion: "Add more items to proceed"
- ✅ Subtotal prominently displayed: ₹30 / ₹50

**Actual Result:**
[To be filled during testing]

**Status:**
[ ] Not Tested | [ ] Pass | [ ] Fail

---

### Test Case: TC-ORDER-EDGE003 — Delivery Address Beyond 15 km

**Priority**: High  
**Type**: Edge Case  
**Roles**: Customer  
**Platform**: Both

**Prerequisites:**
- Chef location: 12.9716°N, 77.5946°E (Bengaluru)
- Customer address: 13.0827°N, 80.2707°E (Chennai) ~290 km

**Steps:**
1. Create order from Chennai chef
2. Select delivery address in Bengaluru
3. Attempt to checkout

**Expected Result:**
- ✅ Checkout blocked
- ✅ Error: "Delivery not available beyond 15 km"
- ✅ Distance displayed: "290 km (max 15 km)"
- ✅ Suggestion: "Choose a closer address"

**Actual Result:**
[To be filled during testing]

**Status:**
[ ] Not Tested | [ ] Pass | [ ] Fail

---

### Test Case: TC-ORDER-EDGE004 — COD Daily Limit Reached

**Priority**: High  
**Type**: Edge Case  
**Roles**: Customer  
**Platform**: Both

**Prerequisites:**
- User has placed 3 COD orders today
- Trust state = NORMAL (limit: 3 COD/day)

**Steps:**
1. Create 4th order of the day
2. Attempt to select "Cash on Delivery"

**Expected Result:**
- ✅ COD option grayed out
- ✅ Message: "Daily COD limit reached (3/3)"
- ✅ Suggestion: "Use UPI or try again tomorrow"
- ✅ UPI option still available

**Actual Result:**
[To be filled during testing]

**Status:**
[ ] Not Tested | [ ] Pass | [ ] Fail

---

### Test Case: TC-ORDER-EDGE005 — Payment Lock Expiry Edge Case

**Priority**: Medium  
**Type**: Edge Case  
**Roles**: Customer  
**Platform**: Both

**Prerequisites:**
- Order checked out with 15-minute lock

**Steps:**
1. Checkout order at 10:00 AM (lock expires 10:15 AM)
2. Initiate payment at 10:14 AM (1 minute before expiry)
3. Complete payment at 10:16 AM (1 minute after expiry)

**Expected Result:**
- ✅ Payment accepted (grace period)
- ✅ Order status = `paid`
- ✅ No double-order created
- ✅ Lock extended after payment initiation

**Actual Result:**
[To be filled during testing]

**Status:**
[ ] Not Tested | [ ] Pass | [ ] Fail

---

### Test Case: TC-ORDER-EDGE006 — Reorder with All Items Unavailable

**Priority**: Medium  
**Type**: Edge Case  
**Roles**: Customer  
**Platform**: Both

**Prerequisites:**
- Previous order had 3 items
- All 3 items now unavailable (chef disabled)

**Steps:**
1. Navigate to "My Orders"
2. Tap "Reorder" on previous order

**Expected Result:**
- ✅ Error: "All items from this order are unavailable"
- ✅ Suggestion: "Browse chef's current menu"
- ✅ "View Menu" button
- ✅ No items added to cart

**Actual Result:**
[To be filled during testing]

**Status:**
[ ] Not Tested | [ ] Pass | [ ] Fail

---

### Test Case: TC-ORDER-EDGE007 — Concurrent Order Creation

**Priority**: High  
**Type**: Edge Case  
**Roles**: Customer (Multiple Devices)  
**Platform**: Both

**Prerequisites:**
- User logged in on 2 devices simultaneously

**Steps:**
1. Device 1: Checkout order and initiate payment
2. Device 2: Simultaneously checkout same order
3. Complete payment on Device 1

**Expected Result:**
- ✅ Device 1: Payment successful
- ✅ Device 2: Error: "Order already being processed"
- ✅ No duplicate payment
- ✅ Distributed lock prevents race condition

**Actual Result:**
[To be filled during testing]

**Status:**
[ ] Not Tested | [ ] Pass | [ ] Fail

---

## ❌ **Error Handling Tests**

Tests that verify graceful handling of failures and error scenarios.

---

### Test Case: TC-ORDER-ERR001 — Network Timeout During Checkout

**Priority**: High  
**Type**: Error Handling  
**Roles**: Customer  
**Platform**: Both

**Network Simulation**: Enable slow 2G or timeout

**Steps:**
1. Enable network throttling (slow 2G)
2. Attempt to checkout order
3. Observe timeout behavior

**Expected Result:**
- ✅ Loading indicator for 30 seconds
- ✅ After timeout: Error message
  - "Request timed out. Please try again."
- ✅ "Retry" button available
- ✅ Order state preserved (not lost)

**Actual Result:**
[To be filled during testing]

**Status:**
[ ] Not Tested | [ ] Pass | [ ] Fail

---

### Test Case: TC-ORDER-ERR002 — Invalid Razorpay Signature

**Priority**: Critical  
**Type**: Error Handling  
**Roles**: System (Security)  
**Platform**: Backend

**Test Data:**
- Valid Razorpay Order ID
- Valid Payment ID
- **Invalid** Signature (tampered)

**Steps:**
1. Create order and initiate payment
2. Simulate webhook with invalid signature
3. Observe backend response

**Expected Result:**
- ✅ Payment rejected
- ✅ Error: "Invalid payment signature"
- ✅ Order status remains `payment_pending`
- ✅ Security event logged
- ✅ Alert sent to admin

**Actual Result:**
[To be filled during testing]

**Status:**
[ ] Not Tested | [ ] Pass | [ ] Fail

---

### Test Case: TC-ORDER-ERR003 — Chef Goes Offline During Order Placement

**Priority**: High  
**Type**: Error Handling  
**Roles**: Customer, Chef  
**Platform**: Both

**Steps:**
1. Customer adds items to cart
2. Chef sets kitchen to "Offline"
3. Customer attempts to checkout

**Expected Result:**
- ✅ Checkout blocked
- ✅ Error: "Chef is no longer accepting orders"
- ✅ Suggestion: "Please try again later"
- ✅ Cart preserved (not cleared)

**Actual Result:**
[To be filled during testing]

**Status:**
[ ] Not Tested | [ ] Pass | [ ] Fail

---

### Test Case: TC-ORDER-ERR004 — Razorpay Service Down

**Priority**: Critical  
**Type**: Error Handling  
**Roles**: Customer  
**Platform**: Both

**Network Simulation**: Block requests to `api.razorpay.com`

**Steps:**
1. Checkout order
2. Select "UPI Intent"
3. Tap "Pay Now"

**Expected Result:**
- ✅ Error: "Payment service unavailable"
- ✅ Suggestion: "Please try COD or retry later"
- ✅ Fallback to COD option highlighted
- ✅ Order not marked as failed
- ✅ Retry button available

**Actual Result:**
[To be filled during testing]

**Status:**
[ ] Not Tested | [ ] Pass | [ ] Fail

---

### Test Case: TC-ORDER-ERR005 — Database Connection Lost During Order Creation

**Priority**: Critical  
**Type**: Error Handling  
**Roles**: System  
**Platform**: Backend

**Simulation**: Kill PostgreSQL connection mid-transaction

**Steps:**
1. Initiate order creation API call
2. Kill database connection during transaction
3. Observe response

**Expected Result:**
- ✅ Transaction rolled back (no partial data)
- ✅ Error: "Service temporarily unavailable"
- ✅ HTTP 503 Service Unavailable
- ✅ No orphaned records in database
- ✅ User can retry without issues

**Actual Result:**
[To be filled during testing]

**Status:**
[ ] Not Tested | [ ] Pass | [ ] Fail

---

## 🔒 **Security Tests**

Tests that verify access control, data protection, and abuse prevention.

---

### Test Case: TC-ORDER-SEC001 — Unauthorized Order Access

**Priority**: Critical  
**Type**: Security  
**Roles**: Customer A, Customer B  
**Platform**: Both

**Steps:**
1. Customer A creates order (Order ID: order-uuid-A)
2. Customer B logs in
3. Customer B attempts to access order-uuid-A

**Expected Result:**
- ✅ Error 403: "Unauthorized access to order"
- ✅ No order details leaked
- ✅ Security event logged
- ✅ Customer B cannot view, cancel, or modify order

**Actual Result:**
[To be filled during testing]

**Status:**
[ ] Not Tested | [ ] Pass | [ ] Fail

---

### Test Case: TC-ORDER-SEC002 — SQL Injection in Order ID

**Priority**: Critical  
**Type**: Security  
**Roles**: Attacker  
**Platform**: Backend

**Test Data:**
- Malicious Order ID: `'; DROP TABLE orders; --`

**Steps:**
1. Send API request with malicious order ID
2. Observe backend behavior

**Expected Result:**
- ✅ Query parameterized (no SQL execution)
- ✅ Error 400: "Invalid order ID format"
- ✅ No database damage
- ✅ Attack logged

**Actual Result:**
[To be filled during testing]

**Status:**
[ ] Not Tested | [ ] Pass | [ ] Fail

---

### Test Case: TC-ORDER-SEC003 — Rate Limit Enforcement

**Priority**: High  
**Type**: Security  
**Roles**: Attacker  
**Platform**: Backend

**Steps:**
1. Make 21 order creation requests within 60 seconds
2. Observe response on 21st request

**Expected Result:**
- ✅ First 20 requests succeed
- ✅ 21st request blocked
- ✅ Error 429: "Rate limit exceeded"
- ✅ Header: `Retry-After: 60`
- ✅ Rate limit resets after 60 seconds

**Actual Result:**
[To be filled during testing]

**Status:**
[ ] Not Tested | [ ] Pass | [ ] Fail

---

### Test Case: TC-ORDER-SEC004 — IDOR (Insecure Direct Object Reference)

**Priority**: Critical  
**Type**: Security  
**Roles**: Attacker  
**Platform**: Backend

**Steps:**
1. Customer A creates order (Order ID: `12345`)
2. Customer B guesses next order ID: `12346`
3. Customer B requests `/orders/12346`

**Expected Result:**
- ✅ Error 403: "Unauthorized access"
- ✅ No order details leaked
- ✅ UUID prevents sequential guessing

**Actual Result:**
[To be filled during testing]

**Status:**
[ ] Not Tested | [ ] Pass | [ ] Fail

---

### Test Case: TC-ORDER-SEC005 — JWT Token Expiry

**Priority**: High  
**Type**: Security  
**Roles**: Customer  
**Platform**: Both

**Steps:**
1. Log in and obtain JWT token
2. Wait 7 days (token expiry)
3. Attempt to create order with expired token

**Expected Result:**
- ✅ Error 401: "Token expired"
- ✅ Redirect to login screen
- ✅ No order created
- ✅ User must re-authenticate

**Actual Result:**
[To be filled during testing]

**Status:**
[ ] Not Tested | [ ] Pass | [ ] Fail

---

### Test Case: TC-ORDER-SEC006 — Abuse Detection (Excessive Cancellations)

**Priority**: High  
**Type**: Security  
**Roles**: Customer  
**Platform**: Both

**Prerequisites:**
- User has cancelled 3 orders in the past 7 days
- Trust state = WARNING

**Steps:**
1. Create 4th order
2. Attempt to cancel

**Expected Result:**
- ✅ Cancellation blocked
- ✅ Error: "You have reached the cancellation limit"
- ✅ Suggestion: "Contact support for assistance"
- ✅ Trust state may escalate to RESTRICTED
- ✅ Abuse event logged

**Actual Result:**
[To be filled during testing]

**Status:**
[ ] Not Tested | [ ] Pass | [ ] Fail

---

### Test Case: TC-ORDER-SEC007 — PII Leakage in Logs

**Priority**: Critical  
**Type**: Security  
**Roles**: DevOps  
**Platform**: Backend

**Steps:**
1. Create order with payment
2. Review backend logs
3. Search for sensitive data

**Expected Result:**
- ✅ No Razorpay secrets in logs
- ✅ No JWT tokens in logs
- ✅ Phone numbers hashed or redacted
- ✅ Payment IDs redacted (last 4 chars only)
- ✅ Address details not logged

**Actual Result:**
[To be filled during testing]

**Status:**
[ ] Not Tested | [ ] Pass | [ ] Fail

---

## ⚡ **Performance Tests**

Tests that verify system performs within acceptable response times and resource usage.

---

### Test Case: TC-ORDER-PERF001 — Order History Load Time

**Priority**: High  
**Type**: Performance  
**Roles**: Customer  
**Platform**: Both

**Prerequisites:**
- User has 100+ orders
- Network: 4G (50 Mbps)

**Steps:**
1. Navigate to "My Orders"
2. Measure time to first render

**Expected Result:**
- ✅ Initial load: <2 seconds
- ✅ Skeleton loaders shown within 100ms
- ✅ First 20 orders rendered within 2 seconds
- ✅ No app freeze or lag

**Actual Result:**
[To be filled during testing]

**Status:**
[ ] Not Tested | [ ] Pass | [ ] Fail

---

### Test Case: TC-ORDER-PERF002 — Order Creation API Response Time

**Priority**: High  
**Type**: Performance  
**Roles**: Backend  
**Platform**: Backend

**Steps:**
1. Send 10 order creation requests
2. Measure average response time

**Expected Result:**
- ✅ Average response time: <200ms
- ✅ P95 response time: <500ms
- ✅ P99 response time: <1000ms
- ✅ No memory leaks

**Actual Result:**
[To be filled during testing]

**Status:**
[ ] Not Tested | [ ] Pass | [ ] Fail

---

### Test Case: TC-ORDER-PERF003 — Concurrent Order Checkouts

**Priority**: High  
**Type**: Performance  
**Roles**: Multiple Customers  
**Platform**: Backend

**Load**: 100 simultaneous checkout requests

**Steps:**
1. Simulate 100 users checking out simultaneously
2. Measure success rate and response times

**Expected Result:**
- ✅ Success rate: >95%
- ✅ Average response time: <500ms
- ✅ No database deadlocks
- ✅ Distributed locks prevent race conditions
- ✅ No order duplication

**Actual Result:**
[To be filled during testing]

**Status:**
[ ] Not Tested | [ ] Pass | [ ] Fail

---

### Test Case: TC-ORDER-PERF004 — Live Order Tracking Polling

**Priority**: Medium  
**Type**: Performance  
**Roles**: Customer  
**Platform**: Both

**Steps:**
1. Open live order tracking screen
2. Let it poll for 5 minutes (10 requests)
3. Measure battery and data usage

**Expected Result:**
- ✅ 10 API requests over 5 minutes (30-second interval)
- ✅ Each request <100 KB data
- ✅ Total data usage: <1 MB
- ✅ Battery drain: <2%
- ✅ No memory leaks

**Actual Result:**
[To be filled during testing]

**Status:**
[ ] Not Tested | [ ] Pass | [ ] Fail

---

### Test Case: TC-ORDER-PERF005 — Database Query Performance

**Priority**: High  
**Type**: Performance  
**Roles**: Backend  
**Platform**: Database

**Prerequisites:**
- 1 million orders in database

**Steps:**
1. Query order history for user with 500 orders
2. Measure query execution time

**Expected Result:**
- ✅ Query execution time: <50ms
- ✅ Uses composite index: `idx_order_user_created`
- ✅ No full table scan
- ✅ EXPLAIN plan shows index usage

**Actual Result:**
[To be filled during testing]

**Status:**
[ ] Not Tested | [ ] Pass | [ ] Fail

---

## 🔄 **Regression Tests**

Tests that verify previously working features haven't broken due to new changes.

---

### Test Case: TC-ORDER-REG001 — Cart-to-Order Integration (Post Cart Module Update)

**Priority**: Critical  
**Type**: Regression  
**Roles**: Customer  
**Platform**: Both

**Context**: Ensure cart checkout still works after any cart module changes

**Steps:**
1. Add 2 items to cart
2. Checkout cart
3. Complete payment
4. Verify order created successfully

**Expected Result:**
- ✅ All TC-ORDER-F001 to TC-ORDER-F003 tests pass
- ✅ No broken API contracts
- ✅ Order created with correct items and pricing

**Actual Result:**
[To be filled during testing]

**Status:**
[ ] Not Tested | [ ] Pass | [ ] Fail

---

### Test Case: TC-ORDER-REG002 — Razorpay Integration (Post Payment Module Update)

**Priority**: Critical  
**Type**: Regression  
**Roles**: Customer  
**Platform**: Both

**Context**: Ensure Razorpay payments work after payment module changes

**Steps:**
1. Create order
2. Initiate UPI Intent payment
3. Complete payment in UPI app
4. Verify order marked as paid

**Expected Result:**
- ✅ UPI Intent URL generated correctly
- ✅ Signature verification works
- ✅ Webhook processing successful
- ✅ Order status updated

**Actual Result:**
[To be filled during testing]

**Status:**
[ ] Not Tested | [ ] Pass | [ ] Fail

---

### Test Case: TC-ORDER-REG003 — Commission Calculation (Post Commission Module Update)

**Priority**: High  
**Type**: Regression  
**Roles**: Chef (Creator), Customer  
**Platform**: Both

**Context**: Ensure commissions calculated correctly after changes

**Steps:**
1. Create reel-linked order
2. Complete order delivery
3. Check chef's commission

**Expected Result:**
- ✅ Commission = 8% of food value
- ✅ Commission status = `pending`
- ✅ Attribution data preserved

**Actual Result:**
[To be filled during testing]

**Status:**
[ ] Not Tested | [ ] Pass | [ ] Fail

---

### Test Case: TC-ORDER-REG004 — Coin Rewards (Post User Module Update)

**Priority**: High  
**Type**: Regression  
**Roles**: Customer  
**Platform**: Both

**Context**: Ensure coin crediting works after user module changes

**Steps:**
1. Place order worth ₹500
2. Complete delivery
3. Check coin balance

**Expected Result:**
- ✅ Coins credited: 500
- ✅ Notification sent
- ✅ Coin transaction logged

**Actual Result:**
[To be filled during testing]

**Status:**
[ ] Not Tested | [ ] Pass | [ ] Fail

---

## 📱 **Platform-Specific Tests**

Tests that verify platform-specific behaviors work correctly.

---

### Test Case: TC-ORDER-PLAT001 — UPI Intent (Android)

**Priority**: Critical  
**Type**: Platform-Specific  
**Roles**: Customer  
**Platform**: Android

**Steps:**
1. Checkout order on Android
2. Select UPI Intent
3. Tap "Pay Now"

**Expected Result:**
- ✅ UPI Intent URL opens in:
  - Google Pay (if installed)
  - PhonePe (if installed)
  - BHIM (if installed)
  - UPI app picker (if multiple apps)
- ✅ Amount pre-filled correctly
- ✅ Return to Chefooz after payment

**Actual Result:**
[To be filled during testing]

**Status:**
[ ] Not Tested | [ ] Pass | [ ] Fail

---

### Test Case: TC-ORDER-PLAT002 — UPI Intent (iOS)

**Priority**: Critical  
**Type**: Platform-Specific  
**Roles**: Customer  
**Platform**: iOS

**Steps:**
1. Checkout order on iOS
2. Select UPI Intent
3. Tap "Pay Now"

**Expected Result:**
- ✅ UPI Intent URL opens in:
  - Google Pay (if installed)
  - PhonePe (if installed)
  - BHIM (if installed)
- ✅ Fallback to Razorpay web checkout if no UPI app
- ✅ Return to Chefooz after payment

**Actual Result:**
[To be filled during testing]

**Status:**
[ ] Not Tested | [ ] Pass | [ ] Fail

---

### Test Case: TC-ORDER-PLAT003 — Push Notifications (iOS)

**Priority**: High  
**Type**: Platform-Specific  
**Roles**: Customer  
**Platform**: iOS

**Steps:**
1. Complete order payment
2. Receive push notification

**Expected Result:**
- ✅ Notification displays on lock screen
- ✅ Tap notification opens order details
- ✅ Notification includes order ID and total
- ✅ Sound and vibration (if enabled)

**Actual Result:**
[To be filled during testing]

**Status:**
[ ] Not Tested | [ ] Pass | [ ] Fail

---

### Test Case: TC-ORDER-PLAT004 — Deep Linking (Order Details)

**Priority**: Medium  
**Type**: Platform-Specific  
**Roles**: Customer  
**Platform**: Both

**Test Data:**
- Deep link: `chefooz://orders/order-uuid-123`

**Steps:**
1. Send deep link via SMS/WhatsApp
2. Tap deep link on device
3. Observe app behavior

**Expected Result:**
- ✅ Chefooz app opens
- ✅ Navigates directly to order details screen
- ✅ Order ID: order-uuid-123
- ✅ Works even if app is closed

**Actual Result:**
[To be filled during testing]

**Status:**
[ ] Not Tested | [ ] Pass | [ ] Fail

---

## 📊 **Test Data**

### Test User Accounts

> ⚠️ **Chefooz uses OTP-only authentication (no passwords)**. Login is phone number + OTP via WhatsApp (primary) or Twilio SMS (fallback). Use `/api/v1/auth/v2/send-otp` → `/api/v1/auth/v2/verify-otp` to obtain JWT tokens.

**OTP Auth Flow (curl — use to obtain JWT for API tests):**
```bash
# Step 1: Request OTP
curl -s -X POST https://api-staging.chefooz.com/api/v1/auth/v2/send-otp \
  -H "Content-Type: application/json" \
  -d '{"phoneNumber": "+919876543210"}' | jq '.data.requestId'

# Step 2: Verify OTP (enter the OTP received via WhatsApp/SMS)
curl -s -X POST https://api-staging.chefooz.com/api/v1/auth/v2/verify-otp \
  -H "Content-Type: application/json" \
  -d '{"requestId": "<REQUEST_ID>", "otp": "<OTP_FROM_WHATSAPP_OR_SMS>"}' | jq '.data.accessToken'

# Save the token
export JWT_TOKEN="<ACCESS_TOKEN_FROM_ABOVE>"
```

```json
{
  "customers": [
    {
      "phone": "+919876543210",
      "authFlow": "POST /api/v1/auth/v2/send-otp → POST /api/v1/auth/v2/verify-otp",
      "note": "trustState: NORMAL"
    },
    {
      "phone": "+919876543211",
      "authFlow": "POST /api/v1/auth/v2/send-otp → POST /api/v1/auth/v2/verify-otp",
      "note": "trustState: RESTRICTED"
    }
  ],
  "chefs": [
    {
      "phone": "+919876543212",
      "authFlow": "POST /api/v1/auth/v2/send-otp → POST /api/v1/auth/v2/verify-otp",
      "note": "kitchenStatus: online, acceptingOrders: true"
    }
  ],
  "riders": [
    {
      "phone": "+919876543213",
      "authFlow": "POST /api/v1/auth/v2/send-otp → POST /api/v1/auth/v2/verify-otp",
      "note": "status: available"
    }
  ]
}
```

---

### Test Menu Items

```json
[
  {
    "id": "menu-item-uuid-001",
    "title": "Margherita Pizza",
    "pricePaise": 25000,
    "availability": { "isAvailable": true },
    "foodType": "veg",
    "prepTimeMinutes": 25
  },
  {
    "id": "menu-item-uuid-002",
    "title": "Garlic Bread",
    "pricePaise": 15000,
    "availability": { "isAvailable": true },
    "foodType": "veg",
    "prepTimeMinutes": 10
  },
  {
    "id": "menu-item-uuid-003",
    "title": "Chicken Tikka",
    "pricePaise": 35000,
    "availability": { "isAvailable": false },
    "foodType": "non-veg",
    "prepTimeMinutes": 30
  }
]
```

---

### Test Addresses

```json
[
  {
    "id": "address-uuid-001",
    "label": "Home",
    "fullName": "Priya Sharma",
    "phone": "+919876543210",
    "line1": "123, MG Road",
    "line2": "Near Central Park",
    "city": "Bengaluru",
    "state": "Karnataka",
    "pincode": "560001",
    "lat": 12.9716,
    "lng": 77.5946
  },
  {
    "id": "address-uuid-002",
    "label": "Office",
    "fullName": "Priya Sharma",
    "phone": "+919876543210",
    "line1": "456, Tech Park",
    "line2": "Whitefield",
    "city": "Bengaluru",
    "state": "Karnataka",
    "pincode": "560066",
    "lat": 12.9698,
    "lng": 77.7500
  }
]
```

---

## ✅ **Test Execution Summary**

### Coverage Matrix

| Test Type | Total Tests | Pass | Fail | Not Tested | Coverage % |
|-----------|-------------|------|------|------------|------------|
| **Functional** | 12 | 0 | 0 | 12 | 0% |
| **UX/UI** | 5 | 0 | 0 | 5 | 0% |
| **Edge Case** | 7 | 0 | 0 | 7 | 0% |
| **Error Handling** | 5 | 0 | 0 | 5 | 0% |
| **Security** | 7 | 0 | 0 | 7 | 0% |
| **Performance** | 5 | 0 | 0 | 5 | 0% |
| **Regression** | 4 | 0 | 0 | 4 | 0% |
| **Platform-Specific** | 4 | 0 | 0 | 4 | 0% |
| **TOTAL** | **49** | **0** | **0** | **49** | **0%** |

**Target**: ≥95% pass rate before production deployment

---

## 🚀 **Pre-Deployment Checklist**

Before deploying Order module to production:

- [ ] All critical (Priority: Critical) tests passed
- [ ] All high-priority (Priority: High) tests passed
- [ ] ≥90% of medium-priority tests passed
- [ ] No security vulnerabilities found
- [ ] Performance benchmarks met
- [ ] Razorpay integration tested in production mode
- [ ] Webhooks tested with real Razorpay events
- [ ] Abuse detection validated with real user patterns
- [ ] Database migrations tested on staging
- [ ] Rollback plan documented and tested

---

## 📚 **Related Documentation**

- **Feature Overview**: [FEATURE_OVERVIEW.md](./FEATURE_OVERVIEW.md)
- **Technical Guide**: [TECHNICAL_GUIDE.md](./TECHNICAL_GUIDE.md)
- **Cart Module QA**: [../cart/QA_TEST_CASES.md](../cart/QA_TEST_CASES.md)
- **Payment Integration**: [../../integrations/PAYMENT_FLOW_COMPLETE.md](../../integrations/PAYMENT_FLOW_COMPLETE.md)

---

## ✅ **Completion Status**

**Test Cases**: 49 total  
**Documentation**: Complete  
**Ready for Testing**: ✅ Yes  
**Last Updated**: 2026-02-15

---

**[MODULE_COMPLETE ✅]**

---

## 🐛 Bug Regression Tests — QA Batch April 2026

### TC-ORDER-BUG-001: Chef marks READY → correct "Order Ready" push sent to customer

**Type:** Bug Regression  
**Feature area:** `chef-orders.service.ts::updateOrderStatus`  
**Priority:** P1

**Preconditions:**
- An order is in PREPARING status
- Customer has push notifications enabled

**Steps:**
1. Chef taps "Mark Ready" in chef order management screen
2. Order status transitions from PREPARING → READY
3. Observe customer device for push notification

**Expected result:** Customer receives — "Order Ready! 🍽️ — Your order #{{orderId}} is ready for pickup or delivery."
**Actual result (before fix):** Customer received "Order Preparing 👨‍🍳 — Chef is now preparing your order" (template `order.cooking`) which is semantically wrong for the READY state.
**Fix applied:** `chef-orders.service.ts` — changed dispatch from `'order.cooking'` to `'order.ready'` in the `ChefOrderStatus.READY` branch. Wrapped in try-catch for resilience.
**Regression test:** Manual — mark order as READY and verify the customer notification says "Order Ready" not "Preparing".
**Status:** Fixed ✅

---

### TC-ORDER-BUG-002: Chef marks PREPARING → customer push sent

**Type:** Bug Regression  
**Feature area:** `chef-orders.service.ts::updateOrderStatus`  
**Priority:** P1

**Preconditions:**
- An order is in ACCEPTED status
- Customer has push notifications enabled

**Steps:**
1. Chef taps "Start Preparing" in chef order management
2. Order transitions ACCEPTED → PREPARING
3. Observe customer device

**Expected result:** Customer receives — "Order Preparing 👨‍🍳 — Chef is now preparing your order #{{orderId}}. It will be ready soon!"
**Actual result (before fix):** No notification was sent when status changed to PREPARING. The PREPARING branch had no dispatch call.
**Fix applied:** `chef-orders.service.ts` — added `notificationDispatcher.send(order.userId, 'order.cooking', { orderId })` in the `ChefOrderStatus.PREPARING` branch.
**Regression test:** Manual — accept and start preparing an order, confirm customer gets "preparing" notification.
**Status:** Fixed ✅

---

### TC-ORDER-BUG-003: Chef marks READY → rider receives pickup notification

**Type:** Bug Regression  
**Feature area:** `chef-orders.service.ts::updateOrderStatus`  
**Priority:** P1

**Preconditions:**
- Order has an assigned rider (`deliveryPartnerId` is set)
- Chef marks order READY

**Steps:**
1. Rider is auto-assigned to an order
2. Chef marks order as READY
3. Observe rider device for push notification

**Expected result:** Rider receives — "Order Ready for Pickup 🏍️ — Order #{{orderId}} is ready at the chef's location. Head over to collect it!"
**Actual result (before fix):** No notification was sent to the rider. Rider had no signal to go pick up the order except by manually polling.
**Fix applied:** `chef-orders.service.ts` — added `notificationDispatcher.send(order.deliveryPartnerId, 'delivery.order_ready_for_pickup', { orderId })` after the customer notification in the READY branch. Added template `delivery.order_ready_for_pickup` to `notification.templates.ts`.
**Regression test:** Manual — assign a rider before marking READY, confirm rider gets the pickup notification.
**Status:** Fixed ✅

---

**Last Updated**: 2026-04-18

### TC-ORD-DM-001: OrderCard Component — White Background in Dark Mode

**Type:** Bug Regression  
**Feature area:** `components/OrderCard.tsx` — used in Profile → Your Orders FlatList  
**Priority:** P1

**Preconditions:**
- Device/simulator dark mode is enabled
- User is logged in and has placed at least one order

**Steps:**
1. Enable dark mode on device
2. Launch app, log in as Customer
3. Navigate to Profile → Your Orders (or any screen that renders `OrderCard`)
4. Observe: card backgrounds, order ID text, date text, attribution text, action divider, "View" button border

**Expected result:** Card uses `colors.surface` (#1C1C1E) background. Text colours follow `colors.textPrimary` / `colors.textSecondary` / `colors.textMuted`. Dividers and borders use `colors.border`.

**Actual result (before fix):** `card.backgroundColor: '#FFFFFF'` caused every OrderCard to render as a white card on dark background, overriding the screen's dark theme.

**Fix applied:** Converted static `StyleSheet.create` to `makeStyles(colors: any)` factory in `components/OrderCard.tsx`; added `const styles = useMemo(() => makeStyles(theme.colors), [theme.colors])` after `const theme = useChefoozTheme()`. Replaced all hardcoded hex values (`#FFFFFF`, `#1A1A1A`, `#999`, `#666`, `#F3F4F6`, `#E5E7EB`, `#374151`) with theme tokens.

**Regression test:** `apps/chefooz-app/src/components/__tests__/OrderCard.dark-mode.spec.ts`  
**Status:** Fixed ✅
---

### TC-ORDER-BUG-004: Admin orders page always showed empty state

**Type:** Bug Regression  
**Feature area:** Admin Dashboard — `/dashboard/orders`  
**Priority:** P0  
**Date fixed:** 2026-03-06

**Preconditions:**
- Logged in as admin user
- Real customer orders exist in the database

**Steps:**
1. Navigate to `/dashboard/orders` in the admin portal
2. Observe the orders table

**Expected result:** All platform orders are listed with status, amount, item count, and date.

**Actual result (before fix):** Table showed "No orders found" even though orders existed in the DB.

**Root cause:** The page used `useOrdersQuery` which calls `GET /v1/orders/history`. That endpoint applies a `WHERE userId = req.user.id` filter. The admin JWT user has no rows in the `orders` table, so the query always returns an empty array.

**Fix applied:**
- Created `GET /api/v1/admin/orders` endpoint (`AdminOrdersController`) with `@Roles('admin')` guard — platform-wide, no user filter.
- Added `adminListOrders()` method on `OrderService` with status/search/date filters and offset pagination.
- Created `adminOrdersClient` and `useAdminOrders` hook in `libs/api-client`.
- Replaced `useOrdersQuery` with `useAdminOrders` in `apps/chefooz-admin/src/app/dashboard/orders/page.tsx`.
- Added page state and Prev/Next pagination controls.

**Regression test:** Manual QA — navigate to `/dashboard/orders` as admin, confirm orders appear.

---

## Live Tracking Map — Production Upgrade (April 2026)

### TC-ORDER-TRACK001: Chef Kitchen Pin on Live Map

**Type:** Manual
**Feature area:** Live Tracking Screen — map
**Priority:** P1

**Preconditions:**
- Order is in `PAID` or later status
- Chef kitchen record has `latitude` and `longitude` stored

**Steps:**
1. Customer places an order from a chef whose kitchen has GPS coordinates
2. Navigate to the order tracking screen
3. Observe the map

**Expected result:** An orange 🍳 marker appears at the chef's kitchen location on the map. `fitToCoordinates` includes the kitchen pin so all three markers are visible.
**Actual result (before fix):** No kitchen marker was shown — API did not return chef coordinates.
**Fix applied:** `getOrderLiveStatus()` now fetches `ChefKitchen.latitude/longitude` via `ChefKitchenService.getKitchen(order.chefId)` and returns `chefLocation: { lat, lng }` in the response. `OrderLiveStatus` type updated with optional `chefLocation` field.
**Status:** Fixed ✅

---

### TC-ORDER-TRACK002: Google Directions Road Route Polyline

**Type:** Manual
**Feature area:** Live Tracking Screen — map polyline
**Priority:** P1

**Preconditions:**
- Order delivery status is `PICKED_UP` or `OUT_FOR_DELIVERY`
- Rider has valid GPS
- Google Maps Directions API is enabled on the project key

**Steps:**
1. Open the live tracking screen for an in-delivery order
2. Wait for the rider location to load (~5s)
3. Observe the green line on the map

**Expected result:** The green line follows actual roads (not a straight line). The Directions API is only called when the rider moves >100 m from the last route fetch origin.
**Actual result (before fix):** A straight geodesic line was drawn between rider and destination regardless of road geometry.
**Fix applied:** Added `decodePolyline()` utility and a `useEffect` that calls `GET maps.googleapis.com/maps/api/directions/json` with an AbortController. `routeCoords` drives the `<Polyline>` component. A dashed fallback line is shown while the route loads.
**Status:** Fixed ✅

---

### TC-ORDER-TRACK003: Custom Marker Rendering (Bike, Home, Chef Hat)

**Type:** Manual
**Feature area:** Live Tracking Screen — custom markers
**Priority:** P2

**Preconditions:**
- Order is in delivery (rider has GPS)

**Steps:**
1. Open live tracking screen
2. Observe all three markers on the map

**Expected result:**
- Rider = green circle with 🏍️ emoji, rotates based on heading
- Destination = red circle with home icon
- Chef kitchen = orange circle with 🍳 emoji (if kitchen has GPS)

**Actual result (before fix):** Plain green/red native pin markers — no context about who/what each pin represents.
**Fix applied:** All three markers use custom `<View>` children with `tracksViewChanges={false}`. Rider marker applies `transform: rotate(heading deg)` for directional context.
**Status:** Fixed ✅

---

### TC-ORDER-TRACK004: Dark Mode Map Style

**Type:** Manual
**Feature area:** Live Tracking Screen — map aesthetics
**Priority:** P3

**Preconditions:**
- Device/app in dark mode

**Steps:**
1. Switch app to dark mode
2. Open live tracking screen

**Expected result:** Map renders with a dark navy palette matching the app theme. POI and transit labels are hidden.
**Actual result (before fix):** Default light Google Maps was shown regardless of app theme.
**Fix applied:** `DARK_MAP_STYLE` constant applied via `customMapStyle` prop when `isDark` is true.
**Status:** Fixed ✅

---

### TC-ORDER-TRACK005: ETA Badge Overlay on Map

**Type:** Manual
**Feature area:** Live Tracking Screen — ETA display
**Priority:** P2

**Preconditions:**
- Order in any trackable status

**Steps:**
1. Open live tracking screen
2. Observe bottom-left of the map tile

**Expected result:** A semi-transparent badge shows the current ETA (e.g. "15-20 min") overlaid on the map.
**Actual result (before fix):** ETA was only shown in the header bar — not visible when the user's focus was on the map.
**Fix applied:** `etaMapBadge` View absolutely positioned at `bottom: 56, left: 12` within the mapSection container.
**Status:** Fixed ✅

---

### TC-ORDER-TRACK006: Rider Accuracy Pulse Circle

**Type:** Manual
**Feature area:** Live Tracking Screen — GPS accuracy visualisation
**Priority:** P3

**Preconditions:**
- Rider has valid GPS coordinates

**Steps:**
1. Open live tracking screen in `PICKED_UP` or `OUT_FOR_DELIVERY` state
2. Observe the area around the rider marker

**Expected result:** A soft green translucent circle (~120 m radius) pulses around the rider marker, indicating GPS accuracy zone.
**Actual result (before fix):** No accuracy visualisation — rider appeared as a bare pin.
**Fix applied:** Added a `<Circle radius={120} fillColor="rgba(34,197,94,0.12)">` co-located with the rider marker.
**Status:** Fixed ✅

---

### TC-ORDER-TRACK007: Bezier Arc Dotted Line (Replaces Google Directions)

**Type:** Manual
**Feature area:** Live Tracking Screen — route visualisation
**Priority:** P2

**Preconditions:**
- Order in `PICKED_UP` or `OUT_FOR_DELIVERY` status
- Rider has valid GPS location

**Steps:**
1. Open live tracking screen with an active delivery
2. Observe the line connecting rider marker to home marker on the map

**Expected result:** A smooth curved dotted line arcs from the rider to the destination. Line transitions from brand green (#10B981) to teal (#0EA5E9) with a subtle glow underlay. No Google Directions API call is made.
**Actual result (before fix):** A straight dashed line or Google Directions road polyline was shown (API cost incurred).
**Fix applied:** Replaced Google Directions `useEffect` with `generateBezierArc()` quadratic bezier computation. Three `<Polyline>` segments render the glow + gradient. `GOOGLE_MAPS_API_KEY` call is commented out.
**Regression test:** Manual — verify no network call to `maps.googleapis.com/maps/api/directions` in network inspector.
**Status:** Fixed ✅

---

### TC-ORDER-TRACK008: Order Item Name Missing (titleSnapshot null)

**Type:** Bug Regression
**Feature area:** Live Tracking Screen — Chef card / order items
**Priority:** P1

**Preconditions:**
- Order created via cart checkout (multi-item flow)
- Order is older (created before snapshot enrichment fix)

**Steps:**
1. Place an order via cart checkout
2. Open the live tracking screen for that order
3. Observe the item name under the chef card

**Expected result:** Each item shows its real dish name (e.g. "Chicken Biryani × 1") with price.
**Actual result (before fix):** Items showed "Item" fallback because `titleSnapshot` was stored as null for some orders.
**Fix applied:** `order.service.ts` `getOrder()` now queries `chef_menu_items` for any item with a null `titleSnapshot` and enriches the response at read time using `menuItemRepository.findBy({ id: In([...]) })`. Does not mutate the stored DB record.
**Regression test:** `apps/chefooz-apis/src/modules/order/order.service.spec.ts`
**Status:** Fixed ✅

---

### TC-ORDER-TRACK009: Rider Stats on Delivery Card (rating, deliveries)

**Type:** Bug Regression
**Feature area:** Live Tracking Screen — Rider card
**Priority:** P1

**Preconditions:**
- Order has a rider assigned
- Rider has entries in `rider_profiles` table

**Steps:**
1. Navigate to live tracking screen with a rider assigned
2. Observe the rider card stats row

**Expected result:** Shows real star rating (e.g. ★ 4.7) and delivery count (e.g. "312 deliveries") from DB. If `rider_profiles` row doesn't exist, shows "Delivery partner" fallback — no hardcoded zeros.
**Actual result (before fix):** Hardcoded "★ 4.8" and "0 delivered" were shown regardless of real data.
**Fix applied:** Backend `getOrderLiveStatus()` now queries `rider_profiles` via `riderProfileRepo.findOne({ userId })` and returns `rating` + `totalDeliveries` in the `courier` object. Frontend renders conditionally only when values are non-null.
**Status:** Fixed ✅

---

### TC-ORDER-TRACK010: Map shows "Unimplemented component: RNMapsMapView" on iOS

**Type:** Bug Regression
**Feature area:** Live Tracking Screen — Map section; Location Picker; Delivery screens
**Priority:** P0

**Preconditions:**
- iOS device (iPhone 16e or any iPhone)
- App launched via `npx expo run:ios` (native build)
- New Architecture enabled (`newArchEnabled: true`)
- Order is in ASSIGNED / PICKED_UP / OUT_FOR_DELIVERY state

**Steps:**
1. Run `npx expo run:ios` on a physical iPhone
2. Navigate to Orders → tap an active order → tap "Track Live"
3. Observe the map section

**Expected result:** Map renders correctly showing chef pin, rider pin, destination pin, and polyline.
**Actual result (before fix):** Pink/salmon overlay area with text "Unimplemented component: RNMapsMapView" — map section completely broken on iOS.

**Root cause (confirmed):**
The local `node_modules/react-native-maps/package.json` was missing `codegenConfig` entirely.
With New Architecture (Fabric), React Native's `generate-artifacts-executor.js` reads `codegenConfig.ios.componentProvider`
from each dependency's `package.json` to populate `ios/build/generated/ios/RCTThirdPartyComponentsProvider.mm`.
Without `codegenConfig`, `RNMapsMapView` is never registered as a Fabric component → "Unimplemented component".

The `react-native-maps` podspec `Generated` subspec explicitly EXCLUDES its own `RCTThirdPartyComponentsProvider.mm`
from compilation (`exclude_files`), intentionally delegating registration to the app's codegen. The npm-published
`react-native-maps@1.26.20` DOES have the complete `codegenConfig` (verified via integrity hash). The local copy
was stripped — likely because the original Android patch was generated against an older build that pre-dated
`codegenConfig` being added to the published tarball.

Android works because it uses CMake/JNI registration independent of `codegenConfig`.
`newArchEnabled` was always correct as `true` — this fix does NOT disable New Architecture.

**Fix applied:**
1. Restored the complete `codegenConfig` to `node_modules/react-native-maps/package.json` matching the
   npm-published `1.26.20` tarball (integrity: `sha512-kWibDz6w...`):
   ```json
   "codegenConfig": {
     "name": "RNMapsSpecs",
     "type": "all",
     "jsSrcsDir": "./src/specs",
     "outputDir": {"ios": "ios/generated", "android": "android/src/main"},
     "includesGeneratedCode": true,
     "android": {"javaPackageName": "com.rnmaps.fabric"},
     "ios": {
       "componentProvider": {
         "RNMapsGoogleMapView": "RNMapsGoogleMapView",
         "RNMapsGooglePolygon": "RNMapsGooglePolygonView",
         "RNMapsMapView": "RNMapsMapView",
         "RNMapsMarker": "RNMapsMarkerView"
       }
     }
   }
   ```
2. Deleted stale `ios/build/generated/ios/RCTThirdPartyComponentsProvider.{mm,h}`
3. Ran `cd ios && pod install` — regenerated `RCTThirdPartyComponentsProvider.mm` now includes all 4 maps entries
4. On next `npx expo run:ios` the Xcode build will compile the correct registry

**Note:** The `patches/react-native-maps+1.26.20.patch` does NOT need to patch `package.json` because a fresh
`npm install` will install the correct npm-published tarball which already has `codegenConfig`. The fix
is only needed if the local `node_modules` is corrupt or was installed from an older cached version.

**How to verify fix:**
```bash
grep "RNMaps" ios/build/generated/ios/RCTThirdPartyComponentsProvider.mm
# Should output 4 lines: RNMapsMapView, RNMapsMarker, RNMapsGoogleMapView, RNMapsGooglePolygon
```

**How to recover if it regresses after npm install:**
```bash
node -e "const p=require('./node_modules/react-native-maps/package.json'); console.log('has codegenConfig:', !!p.codegenConfig)"
# If false, the tarball installed from an old cache. Run: npm install react-native-maps@1.26.20 --force
```

**Regression test:** Manual — run `npx expo run:ios`, navigate to live tracking, confirm map renders.
**Status:** Fixed ✅
**Date:** 2026-04-05

---

### TC-ORDER-TRACK011: Rider pin never appears on customer live tracking (Android rider on delivery/active screen)

**Type:** Bug Regression
**Feature area:** Live Tracking — rider GPS posting; customer tracking map pin
**Priority:** P0

**Preconditions:**
- Rider is on Android physical device
- Order is in `PICKED_UP` or `OUT_FOR_DELIVERY` state
- Rider's active screen is `delivery/active.tsx` (the normal delivery flow)
- Customer opens Orders → active order → Track Live

**Steps:**
1. Rider accepts order and marks it as Picked Up → lands on `delivery/active` screen
2. Customer opens the live tracking screen for the same order
3. Watch the customer map for the rider pin over 30–60 seconds

**Expected result:** Rider pin appears and moves in real time on the customer map.
**Actual result (before fix):** Customer log shows `📍 Live location: {"lat": 0, "lng": 0}`. Rider pin never appears. Backend logs show heartbeat POSTs but no `POST /v1/rider/location`.

**Root cause:**
`delivery/active.tsx` called `Location.watchPositionAsync` to update its own local map display
but **never called `riderLocationService.startTracking()`**. So `POST /v1/rider/location` was
never sent from the delivery screen. Redis key `rider_location:{orderId}` remained empty →
`GET /v1/orders/:id/live-location` returned `{lat:null,lng:null}` → `Number(null) === 0` →
customer received `{lat:0,lng:0}` → `hasLocation` guard suppressed the rider pin.

The feature worked when tested from `rider/orders/[id].tsx` (iOS emulator) because that screen
correctly calls `riderLocationService.startTracking()`. On Android, riders use `delivery/active.tsx`
after PICKED_UP, so the tracking call was never reached.

The heartbeat (`POST /rider-profile/heartbeat`) was correctly posting GPS to `rider_profiles.currentLat/Lng`
(assignment eligibility only), which confused diagnosis — logs showed GPS working but live tracking was empty.

**Fix applied:**
1. `apps/chefooz-app/src/app/delivery/active.tsx`: Added import of `riderLocationService` singleton
   and a `useEffect` that calls `startTracking(activeDelivery.orderId)` when state is `picked_up`
   or `en_route`, and `stopTracking()` on `delivered`. The singleton is idempotent — safe to call
   from both this screen and `rider/orders/[id].tsx`.
2. `libs/types/src/lib/rider-location.types.ts`: Changed `lat/lng: number` → `number | null` to match
   actual backend behaviour (backend returns `null` when no location is stored yet, not `0`).

**How to verify fix:**
- App logs: `🔍 [LocationTrack] useEffect: deliveryStatus= PICKED_UP | willTrack= true` appears as order loads
- App logs: `[🔍 RiderLocation] startTracking called: orderId= ...` appears immediately after
- App logs: `[🔍 RiderLocation] POSTing location: {orderId, lat, lng}` appears within 2s
- Backend logs: `POST /api/v1/rider/location` appears every ~6s in NestJS output
- Redis: `docker exec valkey redis-cli GET "rider_location:{orderId}"` should return real coords
- Customer map: rider pin should appear and move

**Secondary fixes (follow-up 2026-04-04):**
1. Watchdog dead zone: `lastPostAt === 0` guard prevented watchdog from ever triggering when GPS watch silently dies before any post. Fixed by adding `trackingStartedAt` field and using it as baseline in the watchdog when `lastPostAt === 0`.
2. Misleading 400 error: rate-limit and wrong-order-status 400s were both logged as "Rate limit reached". Fixed to show actual backend error message.
3. Diagnostic `🔍` logs added at the top of `startTracking` and `sendLocationUpdate`.

**Regression test:** Manual — rider on Android screen in PICKED_UP state, customer on any platform, confirm rider pin appears and moves.
**Status:** Fixed ✅
**Date:** 2026-04-04

---

### TC-ORDER-TRACK014: requestForegroundPermissionsAsync hangs on every remount — GPS chain permanently frozen

**Type:** Bug Regression
**Feature area:** `rider/orders/[id].tsx`, `rider-location.service.ts`
**Priority:** P0 (GPS tracking completely broken on Android emulator and some real devices)

**Preconditions:**
- Rider is logged in with role=rider
- Order is in OUT_FOR_DELIVERY or PICKED_UP status
- Rider opens the order detail screen

**Steps:**
1. Rider navigates to `rider/orders/[id]`
2. Screen mounts, Effect 1 calls `riderLocationService.startTracking(orderId)`
3. `startTracking` calls `requestPermissions()`
4. `requestPermissions()` calls `requestForegroundPermissionsAsync()` — even though permission is already granted
5. On Android emulator (and some real Android devices), this call triggers the system permission dialog
6. The dialog either hangs indefinitely or is dismissed without action
7. `startTracking` never proceeds past `requestPermissions()` — GPS watch never starts
8. No `POST /v1/rider/location` is ever sent
9. Redis has no location data → customer sees `{lat: 0, lng: 0}`

**Expected result:** Location permission check is instant (silent), GPS watch starts within ~1s, `POST /v1/rider/location` appears in backend logs, customer sees real coordinates.

**Actual result (before fix):** `startTracking` silently hangs at `requestForegroundPermissionsAsync()`. No GPS posts. Customer sees `{lat:0,lng:0}`.

**Root cause:** `requestForegroundPermissionsAsync()` was called unconditionally on every `startTracking()` invocation. On mounts and remounts where permission is already `granted`, calling `request*` again triggers a redundant dialog. On Android the Promise suspends until the user responds (or never resolves).

**Fix applied:**
In `requestPermissions()`, added `getForegroundPermissionsAsync()` check before any dialog call:
```ts
const { status: existingStatus } = await Location.getForegroundPermissionsAsync();
if (existingStatus === 'granted') return true;
// Only show dialog if not yet granted
const { status } = await Location.requestForegroundPermissionsAsync();
```

**Regression test:** Verify `[RiderLocation] Existing permission status: granted` log appears (not `undetermined`) on second and subsequent mounts.

**Status:** Fixed ✅
**Date:** 2026-04-05

---

### TC-ORDER-TRACK013: Entire delivery/ module was dead code — fully removed, rider/ is the single source of truth

**Type:** Architecture Cleanup / Code Hygiene
**Feature area:** `delivery/`, `rider/`, `libs/api-client`
**Priority:** P1 (dead code with crashing hooks + orphaned components)

**Background:**
During debugging of the lat=0,lng=0 issue, investigation revealed the entire `delivery/` module was unreachable dead code. The active rider flow uses the `rider/` module exclusively. The decision was made to delete the delivery module entirely to establish a single source of truth for rider GPS tracking.

**Architecture finding:**
The app contained two separate delivery systems that grew apart:

| System | Path | Fate |
|---|---|---|
| **Rider module** (active) | `rider/orders/index.tsx` → `rider/orders/[id].tsx` | ✅ Kept — the only real rider flow |
| **Delivery module** (legacy) | `delivery/home.tsx` → `delivery/active.tsx` | 🗑️ Deleted entirely |

The rider flow: Profile → Rider Hub → `/rider/home` → `/rider/orders` → `/rider/orders/[id]`.  
The delivery module had no entry point from the rider flow, and `delivery/home.tsx` crashed on mount due to deprecated throwing hooks.

**Files deleted:**
- `apps/chefooz-app/src/app/delivery/active.tsx`
- `apps/chefooz-app/src/app/delivery/home.tsx`
- `apps/chefooz-app/src/app/delivery/wallet.tsx`
- `apps/chefooz-app/src/components/delivery/DeliveryTimeline.tsx` (orphaned — never imported)
- `libs/api-client/src/lib/hooks/useDelivery.ts` (all 10 hooks were deprecated stubs that threw)
- `libs/api-client/src/lib/clients/delivery.client.ts` (unused by any screen)

**Files edited:**
- `libs/api-client/src/index.ts`: removed `export * from './lib/clients/delivery.client'` and the commented-out `useDelivery` block
- `apps/chefooz-app/src/test-exports.ts`: removed stale `useUpdateDeliveryStatus` console.log

**What was NOT deleted (still live):**
- `DeliverySuccessModal` component — used by the customer order tracking screen
- `libs/types` delivery types — shared with backend, kept
- `useDeliveryAnalytics` hook — separate file used by admin module, not part of dead delivery flow

**Additional bug fixed (watchdog loop):**
`restartWatch` never reset `trackingStartedAt` → watchdog fired every 30s indefinitely. Fixed by resetting `trackingStartedAt = Date.now()` at top of `restartWatch`.

**Status:** Fixed ✅
**Date:** 2026-04-05

---

### TC-ORDER-TRACK012: Rider location shows 0,0 after navigating away and back to order detail

**Type:** Bug Regression / Manual
**Feature area:** `rider/orders/[id].tsx`, `rider-location.service.ts`
**Priority:** P0

**Preconditions:**
- Rider is logged in with `role: rider`
- Active order in `PICKED_UP` or `OUT_FOR_DELIVERY` state
- Rider has opened the order detail and tracking is live (customer sees real coordinates)

**Steps:**
1. Rider is on the order detail screen — customer live-tracking map shows real lat/lng
2. Rider navigates away (e.g., switches tab, presses back, opens another screen)
3. Rider returns to the order detail screen
4. Customer observes the live-tracking response

**Expected result:** Rider location continues to update normally with no gap; customer never sees `{lat:0,lng:0}` on return.

**Actual result (before fix):** After navigating away and back:
- Component remounts with `isLoading=true` and `order=undefined`
- The previous single useEffect skipped `startTracking` (no order data yet)
- A 1–3 second gap occurred before the order loaded and tracking restarted
- If the service singleton was also reset (app restart / Fast Refresh), the gap extended to 3–6 seconds
- During the gap the customer's 5-second poll returned `{lat:0,lng:0}` from Redis

**Root causes (multiple):**
1. **Primary**: `startTracking` was gated on `order?.deliveryStatus` — required order API to respond before GPS watch started. Gap = order-load time + GPS fix time (typically 1–5 s).
2. **Secondary**: `gcTime` on `useRiderOrderDetail` defaulted to 5 min. Cache eviction after 5 min navigation caused `isLoading=true` on return, making gap worse.
3. **Tertiary**: `delivery/active.tsx` tracking effect also gated on `activeDelivery.state === 'picked_up'`, creating the same gap pattern.
4. **Debug visibility**: `stopTracking()` had no call-site logging so it was impossible to distinguish intentional vs unexpected stops.
5. **Log noise**: Every 400 response (ASSIGNED status) emitted a warning, making logs unreadable.

**Fixes applied:**
1. **`rider/orders/[id].tsx`**: Replaced the single combined effect with two effects:
   - **Effect 1 (mount-time)**: calls `riderLocationService.startTracking(id)` immediately when `id` and `role=rider` are known, BEFORE order data loads. Backend validates order status; 400 for ASSIGNED is caught gracefully.
   - **Effect 2 (stop-only)**: watches `order?.deliveryStatus` and calls `stopTracking()` only when `DELIVERED`.
2. **`delivery/active.tsx`**: Applied the same two-effect pattern (immediate start + stop-only).
3. **`rider-orders.hooks.ts`**: Added `gcTime: 10 * 60 * 1000` — cache survives 10 min between navigations, preventing `isLoading=true` on normal return visits.
4. **`rider-location.service.ts`**: Added `consecutiveErrors` counter — only logs on first failure and every 5th after (eliminates 400 log spam for ASSIGNED orders). Added `new Error().stack` trace to `stopTracking()` to identify unexpected callers.

**Regression test:** Manual — rider opens PICKED_UP order detail, navigates away 3+ times, confirms customer map never shows 0,0 after return.
**Status:** Fixed ✅
**Date:** 2026-04-05