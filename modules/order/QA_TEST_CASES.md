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
- Email: test-customer@chefooz.com
- Phone: +919876543210
- Trust State: NORMAL

Customer 2 (Restricted):
- Email: test-restricted@chefooz.com
- Phone: +919876543211
- Trust State: RESTRICTED

Chef:
- Email: test-chef@chefooz.com
- Phone: +919876543212
- Kitchen: Active, Accepting Orders

Rider:
- Email: test-rider@chefooz.com
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
- User logged in as test-customer@chefooz.com
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

```json
{
  "customers": [
    {
      "phone": "+919876543210",
      "note": "Use OTP auth to obtain JWT — trustState: NORMAL"
    },
    {
      "phone": "+919876543211",
      "note": "Use OTP auth to obtain JWT — trustState: RESTRICTED"
    }
  ],
  "chefs": [
    {
      "phone": "+919876543212",
      "note": "Use OTP auth to obtain JWT — kitchenStatus: online, acceptingOrders: true"
    }
  ],
  "riders": [
    {
      "phone": "+919876543213",
      "note": "Use OTP auth to obtain JWT — status: available"
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
