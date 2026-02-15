# 🛒 Checkout Module — Feature Overview

**Status**: ✅ COMPLETE  
**Date**: 2025-02-15  
**Module**: Checkout Flow (Cart → Address → Payment → Order)  
**Related Docs**: [Cart Module](../cart/), [Order Module](../order/), [Payment Module](../payment/)

---

## 📋 Overview

The Checkout module orchestrates the complete order placement flow from cart review through address selection to payment confirmation. It acts as the critical bridge between browsing/shopping and order fulfillment, ensuring all necessary information is collected and validated before payment processing.

**Key Points:**
- ✅ Multi-step checkout flow (Cart → Address → Payment)
- ✅ Address selection and management integration
- ✅ Delivery fee calculation based on distance
- ✅ Dynamic pricing with surge and GST
- ✅ Payment method selection (UPI Intent, COD, Cards)
- ✅ Order validation and confirmation

---

## 🎯 Business Purpose

### Why This Module Exists

**Problem Solved:**
- Streamline order placement process
- Reduce cart abandonment through clear steps
- Ensure all required information collected before payment
- Calculate accurate delivery fees and taxes
- Validate order feasibility (distance, availability)
- Provide payment flexibility (online/COD)

**Value Propositions:**

**For Customers:**
- Clear, step-by-step checkout process
- No surprises - all costs shown upfront
- Multiple payment options (UPI, cards, COD)
- Address management without leaving checkout
- Save time with default address selection
- Order validation before payment commitment

**For Chefs:**
- Confirmed delivery address before order acceptance
- Accurate distance-based delivery fees
- Payment guaranteed before preparation
- Clear order details with customer preferences
- Reduced cancellations due to address issues

**For Platform:**
- Higher conversion rates with streamlined flow
- Reduced support tickets (clear pricing breakdown)
- Better delivery logistics (accurate addresses)
- Lower fraud risk (address validation)
- Revenue optimization (surge pricing, delivery fees)

---

## 👥 User Personas

### 1. **First-Time Customer**
**Characteristics:**
- New to the platform
- Needs guidance through checkout
- Concerned about payment security
- Wants to understand all charges

**Goals:**
- Complete first order successfully
- Understand delivery fees and timing
- Feel confident about payment security
- Know exactly when food will arrive

**Pain Points:**
- Confusing multi-step processes
- Hidden charges appearing at payment
- Unclear delivery radius limitations
- Complex payment flows

### 2. **Repeat Customer**
**Characteristics:**
- Has saved addresses
- Familiar with payment methods
- Expects fast checkout
- May reorder favorites

**Goals:**
- Complete checkout in under 30 seconds
- Use saved default address
- Quick payment with preferred method
- Track order immediately

**Pain Points:**
- Too many confirmation steps
- Having to re-enter address every time
- Payment method not remembered
- Slow checkout process

### 3. **Cautious Shopper**
**Characteristics:**
- Reviews cart multiple times
- Checks all charges carefully
- May abandon cart to compare prices
- Prefers COD over online payment

**Goals:**
- Understand every charge line item
- Verify chef's delivery radius
- See exact delivery time estimate
- Option to pay cash on delivery

**Pain Points:**
- Unclear pricing breakdown
- Surprise delivery fees
- No COD option
- Forced to pay before seeing total cost

---

## 🔑 Key Features

### 1. **Multi-Step Checkout Flow**

**What It Does:**
Breaks checkout into manageable steps: Cart Review → Address Selection → Payment Method → Confirmation.

**Business Rules:**
- Cart must have at least 1 item
- All items must be from same chef (single-chef orders only)
- Address must be within chef's delivery radius (15km max)
- Minimum order value: ₹50 (enforced in cart)
- Each step validates before proceeding to next

**User Experience:**
1. **Step 1: Cart Review** (`/cart`)
   - View all items with quantities
   - See FSSAI compliance info
   - View initial price breakdown
   - Edit quantities or remove items
   - Click "Proceed to Checkout"

2. **Step 2: Address Selection** (`/checkout/address`)
   - See all saved addresses
   - Default address auto-selected
   - Radio button selection for others
   - Add new address option
   - Click "Proceed to Payment"

3. **Step 3: Payment Method** (`/cart/payment`)
   - View final price breakdown
   - Delivery fee based on distance
   - GST (5%) and packaging (₹10) shown
   - Select payment method (UPI/COD/Cards)
   - Click "Place Order" or "Pay Now"

4. **Step 4: Confirmation** (`/orders/[orderId]`)
   - Order confirmed screen
   - Estimated delivery time
   - Real-time tracking link
   - Receipt with order details

**Technical Flow:**
```
Cart (/cart) 
  → Check cart not empty
  → Route to /checkout/address

Address Selection (/checkout/address)
  → Load saved addresses
  → Auto-select default
  → Pass addressId to payment

Payment (/cart/payment?addressId=xyz)
  → Validate addressId
  → Calculate delivery fee from distance
  → Call /v1/cart/checkout API
  → Create order (CREATED status)
  → Open payment gateway (if online)
  → Confirm payment
  → Update order (PAID status)
  → Route to /orders/{orderId}
```

---

### 2. **Address Management Integration**

**What It Does:**
Seamlessly integrates address selection, creation, and editing within the checkout flow.

**Business Rules:**
- User must have at least one address to proceed
- Address must be complete (line1, city, state, pincode, coordinates)
- Phone number validated (10 digits)
- Pincode validated (6 digits)
- Address label: Home, Work, or Other
- One address marked as default

**User Experience:**

**Selecting Existing Address:**
- Radio button selection
- Default address pre-selected
- Visual distinction for default (badge)
- Shows label icon (home/work/other)
- Full address text displayed

**Adding New Address:**
- Click "Add New Address"
- Opens location picker with map
- Drag marker to exact location
- Reverse geocoding fills address
- Confirm and save
- Returns to checkout with new address selected

**Empty State:**
- No addresses saved
- Shows friendly prompt: "No addresses saved"
- "Add New Address" button prominent
- Cannot proceed without address

**Integration Points:**
- `/address-list`: Full CRUD operations
- `/location-picker`: Map-based address entry
- `/location-search`: Search by locality/landmark
- Address component reused across flows

---

### 3. **Dynamic Pricing Calculation**

**What It Does:**
Calculates final order price including delivery fees, surge pricing, GST, and packaging based on multiple factors.

**Business Rules:**

**Pricing Components:**
1. **Subtotal**: Sum of (item price × quantity) for all items
2. **Packaging Fee**: Flat ₹10 per order
3. **Delivery Fee**: Distance-based calculation:
   - 0-3km: ₹20 base fee
   - Beyond 3km: +₹5 per km
   - Peak time surge: +20% (12-2 PM, 7-10 PM)
   - Maximum distance: 15km
4. **GST (5%)**: Applied on food value only
   - CGST: 2.5%
   - SGST: 2.5%
5. **Grand Total**: Subtotal + Packaging + Delivery + GST

**Example Calculation:**
```
Subtotal:      ₹400 (food items)
Packaging:     ₹10
Delivery:      ₹35 (7km × ₹5)
Surge (20%):   ₹7 (peak time)
GST (5%):      ₹20 (5% of ₹400)
----------------------------
Grand Total:   ₹472
```

**User Experience:**
- Pricing breakdown shown in cart
- Updated in real-time during address selection
- Final breakdown before payment
- Each charge explained with tooltip
- No hidden charges
- Total matches payment amount exactly

**Technical Implementation:**
- PricingService calculates all fees
- Distance calculated from chef location to delivery address
- Time-based conditions checked for surge
- Fallback calculation if service fails
- Breakdown stored in order for audit

---

### 4. **Order Validation**

**What It Does:**
Validates order feasibility before payment processing to prevent failures.

**Validation Rules:**

**Cart Validation:**
- ✅ Cart not empty
- ✅ All items from same chef
- ✅ All items still available (not sold out)
- ✅ Prices unchanged since cart creation
- ✅ Quantities within stock limits

**Address Validation:**
- ✅ Address exists and belongs to user
- ✅ Address has valid coordinates
- ✅ Within 15km delivery radius
- ✅ Chef delivers to that area
- ✅ Complete address information

**Chef Validation:**
- ✅ Chef account active
- ✅ Kitchen open (operational hours)
- ✅ Chef accepting orders
- ✅ Not blocked/suspended
- ✅ FSSAI license valid

**User Trust Validation:**
- ✅ User trust state: NORMAL or WARNING (not RESTRICTED/BANNED)
- ✅ COD limit not exceeded (for COD orders)
- ✅ No pending unpaid COD orders
- ✅ Daily order limit not reached

**Error Handling:**
If any validation fails:
- Clear error message shown
- User redirected to appropriate step
- Cart items updated if needed
- Suggestions provided (e.g., "Chef is closed, try after 5 PM")

---

### 5. **Payment Method Selection**

**What It Does:**
Allows customers to choose payment method and handles method-specific flows.

**Payment Methods:**

**1. UPI Intent (Primary):**
- Opens UPI apps (Google Pay, PhonePe, Paytm, BHIM)
- Deep linking to user's preferred UPI app
- Payment within UPI app (no web/SDK overlay)
- Returns to Chefooz after payment
- Instant order confirmation

**2. Cash on Delivery:**
- Available for NORMAL trust users only
- Daily limit: 3 COD orders
- No payment processing required
- Order created immediately
- Payment collected by delivery person

**3. Cards/Wallets (via Razorpay SDK):**
- Credit/Debit cards
- Paytm wallet, PhonePe wallet
- Net banking (all major banks)
- Razorpay SDK overlay
- Secure payment processing

**Business Rules:**
- COD disabled for RESTRICTED users
- UPI Intent requires UPI app installed
- Minimum order value: ₹50 (all methods)
- Payment timeout: 15 minutes
- Failed payment: order auto-cancelled

**User Experience:**
- Visual icons for each method
- Default method: UPI Intent (most popular)
- Saved payment preferences (future)
- Clear CTAs: "Pay Now" vs "Place Order" (COD)
- Payment status tracking
- Retry option on failure

---

### 6. **Order Confirmation**

**What It Does:**
Provides immediate feedback after order placement and guides user to tracking.

**Business Rules:**
- Order ID generated (UUID)
- Initial status: CREATED or PAID (depending on payment)
- Chef notified immediately
- Customer receipt generated
- Estimated delivery time calculated
- Order tracking enabled

**User Experience:**

**Success Screen:**
- Green checkmark animation
- "Order Placed Successfully!" message
- Order ID displayed prominently
- Estimated delivery time: "45-60 mins"
- Chef name and profile photo
- "Track Order" button (primary CTA)
- "Go to Home" button (secondary)
- Push notification sent

**Order Details:**
- Items ordered with quantities
- Delivery address snapshot
- Payment method used
- Total amount paid
- Order timestamp
- Chef contact info (if needed)

**Next Actions:**
- Navigate to order tracking (`/orders/[orderId]`)
- Real-time status updates (8-second polling)
- Live delivery tracking (when out for delivery)
- Reorder option after delivery

---

### 7. **Distance Calculation**

**What It Does:**
Calculates straight-line distance between chef and customer for delivery fee computation.

**Algorithm:**
Haversine formula for great-circle distance:
```typescript
function calculateDistance(
  lat1: number, lng1: number,
  lat2: number, lng2: number
): number {
  const R = 6371; // Earth's radius in km
  const dLat = toRad(lat2 - lat1);
  const dLng = toRad(lng2 - lng1);
  
  const a = Math.sin(dLat/2) * Math.sin(dLat/2) +
    Math.cos(toRad(lat1)) * Math.cos(toRad(lat2)) *
    Math.sin(dLng/2) * Math.sin(dLng/2);
  
  const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a));
  return R * c;
}
```

**Business Rules:**
- Maximum delivery distance: 15km
- Orders beyond 15km: rejected with message
- Distance stored in order for audit
- Used for delivery fee calculation
- Fallback: 3km if calculation fails

**User Experience:**
- Distance not shown to user (internal)
- Delivery fee reflects distance
- "Too far" message if beyond radius
- Suggestion: "Chef delivers within 15km only"

---

### 8. **Surge Pricing**

**What It Does:**
Applies dynamic pricing during peak demand hours to optimize delivery capacity.

**Peak Times:**
- Lunch: 12:00 PM - 2:00 PM
- Dinner: 7:00 PM - 10:00 PM
- Weekends: All day (future enhancement)

**Surge Calculation:**
- Base delivery fee calculated first
- Peak time: +20% surge
- Applied to delivery fee only (not food price)
- Clearly labeled in breakdown

**Example:**
```
Delivery Fee (7km):     ₹35
Peak Time Surge (20%):  ₹7
Total Delivery:         ₹42
```

**User Experience:**
- Orange badge: "Peak Time" shown during surge
- Tooltip explains: "Higher demand, slightly higher delivery fee"
- Total delivery fee prominently displayed
- Comparison: "Usually ₹35, now ₹42"
- Transparent pricing (no hidden surge)

---

## 🔗 Integration Points

### Internal Module Dependencies

| Module | Relationship | Purpose |
|--------|--------------|---------|
| **Cart** | Upstream Dependency | Source of order items and chef info |
| **Address** | Upstream Dependency | Delivery location for fee calculation |
| **Order** | Downstream Consumer | Creates order after checkout |
| **Payment** | Downstream Consumer | Processes payment for online orders |
| **Pricing** | Shared Service | Calculates delivery fees, GST, surge |
| **User** | Shared Service | User authentication and trust state |
| **Chef** | Validation | Chef availability and delivery radius |
| **Notification** | Downstream Consumer | Order confirmation notifications |
| **Analytics** | Downstream Consumer | Checkout funnel metrics |

### External Service Dependencies

| Service | Purpose | Failure Handling |
|---------|---------|------------------|
| **Google Maps Geocoding** | Address validation and reverse geocoding | Use last known coordinates |
| **Razorpay SDK** | Payment processing (cards, wallets) | Show retry option |
| **UPI Apps** | UPI Intent payments | Fallback to Razorpay SDK |

---

## 📊 Key Business Metrics

### Conversion Metrics
- **Checkout Initiation Rate**: (Checkouts Started / Cart Views) × 100
  - Target: 60%+
- **Checkout Completion Rate**: (Orders Placed / Checkouts Started) × 100
  - Target: 70%+
- **Payment Success Rate**: (Paid Orders / Payment Attempts) × 100
  - Target: 90%+

### Abandonment Metrics
- **Cart Abandonment**: Users who add items but don't checkout
- **Address Abandonment**: Users who start checkout but don't select address
- **Payment Abandonment**: Users who reach payment but don't complete
- **Breakdown by Step**:
  - Cart → Address: Target < 15% drop
  - Address → Payment: Target < 10% drop
  - Payment → Confirmation: Target < 5% drop

### Revenue Metrics
- **Average Order Value (AOV)**: Target ₹450+
- **Delivery Fee Revenue**: Average ₹35 per order
- **Surge Pricing Revenue**: 10-15% of delivery fee revenue
- **COD vs Online Split**: Target 30% COD, 70% online

### User Experience Metrics
- **Time to Checkout**: From cart to payment
  - Target: < 60 seconds
- **Address Selection Time**: Average time on address screen
  - Target: < 15 seconds
- **Payment Completion Time**: From payment screen to confirmation
  - Target: < 45 seconds

---

## 🛡️ Security & Compliance

### Data Security

**PII Protection:**
- Delivery addresses encrypted at rest
- Phone numbers masked in logs
- Payment details never stored (handled by Razorpay)
- Address coordinates stored for distance calculation only

**Access Control:**
- User can only checkout own cart
- Address selection limited to user's addresses
- Order creation requires JWT authentication
- Rate limiting on checkout API (10 requests/min)

### Compliance

**Food Safety:**
- FSSAI license info shown in cart
- Chef compliance verified before checkout
- Food safety warnings displayed if applicable

**Consumer Protection:**
- All charges disclosed before payment
- No hidden fees
- Clear refund/cancellation policy
- Order confirmation sent immediately

### Fraud Prevention

**Trust State Enforcement:**
- RESTRICTED users: No COD allowed
- BANNED users: Cannot place orders
- WARNING users: COD limit reduced to 1/day

**Abuse Detection:**
- Daily order limits enforced
- Concurrent checkout attempts blocked (distributed locks)
- Failed payment attempts tracked
- Address validation prevents fake locations

---

## 📱 User Workflows

### Workflow 1: First Order (New Customer)

**Scenario:** Customer placing first order with new address

**Steps:**

1. **Review Cart**
   - User: Views cart with 3 items (₹450)
   - App: Shows subtotal, initial price estimate
   - User: Clicks "Proceed to Checkout"
   - App: Routes to `/checkout/address`

2. **Add First Address**
   - App: Shows empty state: "No addresses saved"
   - User: Clicks "Add New Address"
   - App: Opens location picker with map
   - User: Searches "MG Road, Mumbai" or uses current location
   - User: Drags marker to exact building
   - App: Reverse geocodes to fill address form
   - User: Confirms address details (line1, line2, phone)
   - User: Marks as "Home" and "Default"
   - App: Saves address, returns to checkout
   - App: Auto-selects new address

3. **Proceed to Payment**
   - User: Sees address selected
   - User: Clicks "Proceed to Payment"
   - App: Routes to `/cart/payment?addressId=abc123`
   - App: Calls `/v1/cart/checkout` API
   - Backend: Validates cart, address, chef availability
   - Backend: Calculates distance: 5km
   - Backend: Calculates delivery fee: ₹25
   - Backend: Applies GST: ₹22.50 (5%)
   - Backend: Grand total: ₹507.50
   - App: Shows final price breakdown

4. **Select Payment & Pay**
   - User: Selects "UPI (Google Pay)"
   - User: Clicks "Pay ₹507.50"
   - App: Creates order (status: CREATED)
   - App: Initiates Razorpay payment
   - App: Opens Google Pay via Intent URL
   - User: Approves payment in Google Pay
   - Google Pay: Confirms payment
   - App: Receives payment confirmation
   - App: Calls `/v1/orders/:orderId/confirm-payment`
   - Backend: Updates order status: PAID
   - Backend: Notifies chef via push notification
   - App: Routes to `/orders/{orderId}`

5. **Order Confirmation**
   - App: Shows success animation
   - App: Displays order ID, estimated delivery: 50 mins
   - User: Sees "Track Order" button
   - User: Clicks "Track Order"
   - App: Shows real-time tracking with status updates

**Duration:** 2-3 minutes (first-time address entry adds time)

---

### Workflow 2: Repeat Order (Existing Customer)

**Scenario:** Regular customer with saved addresses

**Steps:**

1. **Review Cart**
   - User: Views cart with 2 items (₹350)
   - User: Clicks "Proceed to Checkout" (1 click)

2. **Address Auto-Selected**
   - App: Shows address screen with default selected
   - User: Verifies default address is correct
   - User: Clicks "Proceed to Payment" (1 click)

3. **Quick Payment**
   - App: Shows final breakdown (₹387)
   - User: Confirms UPI payment method
   - User: Clicks "Pay Now" (1 click)
   - App: Opens Google Pay (instant)
   - User: Confirms in Google Pay (1 tap)

4. **Instant Confirmation**
   - App: Order confirmed immediately
   - App: Shows tracking screen

**Duration:** 30-45 seconds

---

### Workflow 3: COD Order

**Scenario:** Customer prefers cash payment

**Steps:**

1. **Standard Checkout**
   - Cart → Address selection (same as above)

2. **Select COD**
   - User: On payment screen
   - User: Selects "Cash on Delivery"
   - App: Shows message: "Pay when order arrives"
   - User: Clicks "Place Order"

3. **Immediate Confirmation**
   - App: Creates order instantly (no payment processing)
   - Backend: Order status: PAID (payment_method: COD)
   - App: Shows confirmation screen
   - App: Routes to tracking

**Duration:** 30 seconds (no payment processing wait)

**Note:** COD subject to trust state validation

---

## 🖥️ Frontend Screens

### 1. Cart Screen (Pre-Checkout)
**File:** `apps/chefooz-app/src/app/cart/index.tsx`

**Components:**
- Chef header (profile photo, name, badge)
- FSSAI info banner
- Cart items list (editable quantities)
- Cooking preferences input
- Price breakdown summary
- "Proceed to Checkout" button (sticky)

**State Management:**
- Cart data (React Query: useCart)
- Item quantity updates (useUpdateCartItem)
- Remove item (useRemoveCartItem)

**User Actions:**
- Edit quantities (+/- buttons)
- Remove items (swipe or X button)
- Add cooking preferences
- Navigate to checkout

---

### 2. Address Selection Screen
**File:** `apps/chefooz-app/src/app/checkout/address.tsx`

**Components:**
- Header with back button
- "Select Delivery Address" title
- Address cards (radio button selection)
- Default badge for default address
- Address label icons (Home/Work/Other)
- Full address text
- "Add New Address" button
- "Proceed to Payment" button (sticky)

**State Management:**
- Addresses list (React Query: useAddressesQuery)
- Selected address ID (local state)
- Cart validation (useCart)

**User Actions:**
- Select address (radio button)
- Add new address (opens location picker)
- Proceed to payment

**Empty State:**
- "No Addresses Saved" message
- Map marker icon
- "Add New Address" button prominent

---

### 3. Payment Method Screen
**File:** `apps/chefooz-app/src/app/cart/payment.tsx`

**Components:**
- Order summary card
- Chef info (name, profile)
- Delivery address display
- Price breakdown (subtotal, delivery, GST, packaging, total)
- Payment method selector (radio buttons)
- Payment method icons (UPI, COD, Cards)
- "Pay Now" / "Place Order" button (method-dependent)

**State Management:**
- Address ID (route params)
- Cart data (useCart)
- Selected payment method (local state)
- Payment processing (local state)

**User Actions:**
- Review final order
- Select payment method
- Initiate payment
- Handle payment success/failure

**Validation:**
- Address ID present (redirects if missing)
- Cart not empty
- Chef available

---

### 4. Order Confirmation Screen
**File:** `apps/chefooz-app/src/app/orders/[orderId].tsx`

**Components:**
- Success animation (checkmark)
- "Order Placed!" header
- Order ID display
- Estimated delivery time
- Chef info card
- Order summary (items, address, payment)
- "Track Order" button (primary)
- "Go to Home" button (secondary)

**State Management:**
- Order details (React Query: useOrder)
- Live status (React Query: useOrderLiveStatus with 8s polling)

**User Actions:**
- View order details
- Track order in real-time
- Navigate home
- Reorder (after delivery)

---

## 🧪 Testing Considerations

### Test Scenarios

**Happy Path:**
1. ✅ Complete checkout with default address
2. ✅ Add new address during checkout
3. ✅ Pay with UPI Intent successfully
4. ✅ Place COD order successfully
5. ✅ Track order after placement

**Error Handling:**
1. ⚠️ Empty cart at checkout
2. ⚠️ No addresses saved
3. ⚠️ Address beyond delivery radius (>15km)
4. ⚠️ Chef closed during checkout
5. ⚠️ Items sold out after cart creation
6. ⚠️ Payment failure (insufficient balance)
7. ⚠️ Payment timeout (15 minutes)

**Edge Cases:**
1. 🔍 Price change between cart and payment
2. 🔍 Concurrent checkout attempts
3. 🔍 Address selection cancelled mid-flow
4. 🔍 Payment completed but app crashes
5. 🔍 Distance calculation failure
6. 🔍 Surge pricing applied during checkout
7. 🔍 COD limit exceeded

### Test Data

**Test Users:**
```
Normal User: test1@chefooz.com (can use COD)
Restricted User: test2@chefooz.com (no COD)
New User: test3@chefooz.com (no addresses)
```

**Test Addresses:**
```
Within Range (5km): MG Road, Mumbai (19.0760, 72.8777)
Far (16km): Thane (19.2183, 72.9781) - Should fail
Peak Time Test: Run between 7-10 PM for surge
```

**Test Cart:**
```
2x Butter Chicken @ ₹350 = ₹700
Packaging: ₹10
Delivery (5km): ₹25
GST (5%): ₹35
Total: ₹770
```

---

## 🚀 Future Enhancements

### Phase 1 (Current Release)
- ✅ Multi-step checkout flow
- ✅ Address selection and management
- ✅ Distance-based delivery fees
- ✅ Dynamic surge pricing
- ✅ UPI Intent and COD support

### Phase 2 (Q2 2026)
- 📋 Scheduled orders (order for later)
- 📋 Delivery time slot selection
- 📋 Multiple payment methods saved
- 📋 Split payment (partial COD + online)
- 📋 Group orders (order with friends)

### Phase 3 (Q3 2026)
- 📋 Express checkout (skip address for repeat customers)
- 📋 One-tap reorder from order history
- 📋 Subscription orders (weekly/monthly)
- 📋 Referral discounts at checkout
- 📋 Loyalty points redemption

### Phase 4 (Q4 2026)
- 📋 Multi-chef orders (combine items from different chefs)
- 📋 Delivery bundling (reduce fees for nearby deliveries)
- 📋 Green delivery options (bike/walk for nearby)
- 📋 Gift orders (send to someone else's address)

---

## 📚 Related Documentation

**Technical Documentation:**
- [Checkout Technical Guide](./TECHNICAL_GUIDE.md)
- [Checkout QA Test Cases](./QA_TEST_CASES.md)

**Integration Modules:**
- [Cart Module](../cart/)
- [Order Module](../order/)
- [Payment Module](../payment/)
- [Address Module](../address/)

**Historical Guides:**
- `application-guides/CHECKOUT_FLOW_COMPLETE.md`
- `application-guides/CHECKOUT_UI_POLISH_SLICE_COMPLETE.md`
- `application-guides/UPI_INTENT_CHECKOUT_COMPLETE.md`

---

**[CHECKOUT_FEATURE_OVERVIEW_COMPLETE ✅]**
