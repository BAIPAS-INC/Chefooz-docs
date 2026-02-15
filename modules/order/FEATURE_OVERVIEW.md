# 📦 Order Module - Feature Overview

**Module**: Order Management  
**Version**: 2.0 (Multi-Item Orders + Cart Integration)  
**Last Updated**: 2026-02-15  
**Status**: ✅ Production Ready

---

## 🎯 **Business Purpose**

The Order module is the revenue engine of Chefooz, enabling customers to purchase food from chefs with a seamless checkout experience. It handles the complete order lifecycle from creation to delivery, including payment processing, order tracking, commission attribution, and post-order operations.

### Key Value Propositions

**For Customers:**
- 🛒 Multi-item orders from a single chef (like modern food apps)
- 💳 Flexible payment options (UPI Intent, Cash on Delivery)
- 📍 Address-based delivery with pricing transparency
- 📱 Real-time order tracking with live status updates
- 🔄 Quick reorder functionality for favorite meals
- 🎁 Coin rewards on order delivery (100% of dish price)

**For Chefs:**
- 💰 Automated order management and payment processing
- 🎨 Commission-eligible orders from reel conversions
- 📊 Order history and analytics
- ⏱️ Kitchen schedule integration for availability control
- 🚫 Fraud protection with abuse detection

**For the Platform:**
- 💵 Revenue through delivery fees, packaging charges, and GST
- 🎬 Commission system tied to reel attribution (creator economy)
- 🔒 Payment security with Razorpay integration
- 📈 Order analytics for Explore ranking algorithm
- 🛡️ Fraud detection and cancellation abuse prevention

---

## 🧑‍🤝‍🧑 **User Personas & Use Cases**

### Persona 1: Hungry Customer (Primary User)

**Demographics:**
- Age: 18-45
- Tech-savvy mobile users
- Values convenience, quality, and authenticity
- Budget: ₹100-₹1000 per order

**Use Cases:**

#### UC-01: Browse and Order
1. Discover chef through Explore/Feed/Search
2. View chef's menu items
3. Add items to cart with customizations
4. Proceed to checkout
5. Select delivery address
6. Choose payment method (UPI/COD)
7. Complete payment
8. Track order status in real-time

#### UC-02: Quick Reorder
1. View order history
2. Find previous favorite order
3. Tap "Reorder" button
4. Items automatically added to cart with current pricing
5. Complete checkout flow

#### UC-03: Order Tracking
1. Receive push notification: "Order accepted by chef"
2. Open order details screen
3. View real-time status updates:
   - ✅ Payment confirmed
   - 👨‍🍳 Chef preparing (25 mins remaining)
   - 🚗 Out for delivery (Arriving in 10 mins)
   - ✅ Delivered
4. Receive coin rewards notification
5. Rate order and leave review

---

### Persona 2: Chef (Order Fulfillment)

**Demographics:**
- Home chefs or small food business owners
- Age: 25-55
- Wants to monetize cooking skills
- Needs simple order management

**Use Cases:**

#### UC-04: Receive and Accept Orders
1. Receive push notification: "New order from Priya"
2. Review order details (items, quantity, address)
3. Accept order and set preparation time
4. Update order status as cooking progresses
5. Mark as ready for pickup when done

#### UC-05: Manage Kitchen Availability
1. Set daily working hours in schedule
2. Toggle "Accepting Orders" when capacity is full
3. System automatically prevents new orders when unavailable
4. Reopen when ready to accept more orders

---

### Persona 3: Delivery Rider (Logistics)

**Use Cases:**

#### UC-06: Deliver Orders
1. Receive delivery assignment notification
2. View pickup location (chef's address)
3. Navigate to chef's location
4. Pick up order and mark as "Picked Up"
5. Navigate to customer's address
6. Deliver order and mark as "Delivered"
7. Receive delivery fee payment

---

## 🎨 **Key Features**

### 1. **Multi-Item Order Creation**

**Overview**: Customers can add multiple items from a single chef to their cart and checkout together.

**How It Works:**
- User adds items to cart from chef's menu
- Cart enforces single-chef rule (cannot mix items from different chefs)
- Checkout converts cart into a draft order
- Order includes all item snapshots (title, price, image, customizations)

**Business Rules:**
- ✅ Orders can have 1-20 items (configurable limit)
- ✅ All items must be from the same chef
- ✅ Items snapshot at time of order (price freezing)
- ✅ Chef must be accepting orders and within service hours
- ✅ Minimum order value: ₹50 (configurable)

---

### 2. **Flexible Payment System**

**Overview**: Support for both online (UPI Intent) and offline (Cash on Delivery) payments.

**Payment Methods:**

#### A. UPI Intent (Recommended)
- Creates Razorpay order
- Generates UPI deep link
- Opens user's preferred UPI app (GPay, PhonePe, BHIM)
- Webhook confirms payment
- Order auto-transitions to "paid" status

**Advantages:**
- ✅ Instant payment confirmation
- ✅ No cash handling
- ✅ Automatic refunds possible
- ✅ Lower fraud risk

#### B. Cash on Delivery (COD)
- Customer pays rider in cash upon delivery
- No online payment required
- Order marked as "paid" upon delivery confirmation
- Chef receives payout after delivery

**Advantages:**
- ✅ Accessible to all users (no UPI app needed)
- ✅ Trust-building for first-time users
- ✅ No payment gateway dependencies

**Abuse Protection:**
- 🛡️ Users with poor trust scores may have COD disabled
- 🛡️ Maximum 3 COD orders per day per user (configurable)
- 🛡️ COD cancellation patterns tracked for fraud detection

---

### 3. **Order Lifecycle Management**

**Status Flow:**

```
DRAFT → CREATED → PAYMENT_PENDING → PAID → ACCEPTED → 
PREPARING → READY → OUT_FOR_DELIVERY → DELIVERED
```

**Alternative Flows:**
```
PAYMENT_PENDING → PAYMENT_FAILED
PAID → CANCELLED (if chef rejects or rider timeout)
DELIVERED → REFUNDED (if dispute approved)
```

**Status Definitions:**

| Status | Description | Customer View | Chef View |
|--------|-------------|---------------|-----------|
| **DRAFT** | Cart converted to order | Hidden (internal) | Hidden |
| **CREATED** | Order created with address | "Order placed" | "New order" |
| **PAYMENT_PENDING** | Awaiting payment | "Complete payment" | Hidden |
| **PAID** | Payment confirmed | "Awaiting acceptance" | "Accept or reject" |
| **ACCEPTED** | Chef accepted order | "Chef is preparing" | "Start cooking" |
| **PREPARING** | Chef is cooking | "Being prepared" | "Mark as ready" |
| **READY** | Food ready for pickup | "Ready for pickup" | "Waiting for rider" |
| **OUT_FOR_DELIVERY** | Rider picked up | "On the way!" | "Out for delivery" |
| **DELIVERED** | Order delivered | "Delivered ✅" | "Completed ✅" |
| **CANCELLED** | Order cancelled | "Cancelled" | "Cancelled" |
| **PAYMENT_FAILED** | Payment unsuccessful | "Payment failed" | Hidden |

**Automatic Transitions:**
- ⏱️ Payment lock expires after 15 minutes → AUTO-CANCEL
- ⏱️ Rider doesn't accept within 5 minutes → AUTO-CANCEL
- ⏱️ Rider doesn't pick up within 30 minutes → AUTO-CANCEL
- ⏱️ Order not delivered within 2 hours → AUTO-CANCEL

---

### 4. **Address & Delivery Management**

**Address Snapshot:**
- Order captures address snapshot at checkout time
- Snapshot includes: street, landmark, city, pincode, coordinates
- Prevents issues if user edits/deletes address after order
- Used for delivery routing and ETA calculation

**Delivery Fee Calculation:**
- Base distance: 0-3 km → ₹20
- Additional: ₹5 per km beyond 3 km
- Maximum distance: 15 km
- Surge pricing during peak hours (6-9 PM): +30%

**Distance Calculation:**
- Haversine formula for accurate distance
- Chef's kitchen location → Customer's delivery address
- Displayed transparently before payment

---

### 5. **Pricing & Commission System**

**Order Pricing Breakdown:**

```
Subtotal (items)       ₹450.00
Packaging fee          ₹10.00
Delivery fee           ₹30.00
----------------------------
Subtotal               ₹490.00
GST (5%)               ₹24.50
----------------------------
Total                  ₹514.50
```

**Commission Attribution:**
- Orders placed from reel CTA are commission-eligible
- Commission = 8% of `creatorOrderValue` (food value only)
- Attribution data stored in `order.attribution`:
  ```json
  {
    "linkedReelId": "reel-uuid",
    "linkedCreatorOrderId": "order-uuid",
    "creatorUserId": "chef-uuid",
    "creatorOrderValue": 45000  // ₹450 in paise
  }
  ```
- Commission calculated on delivery (not on payment)
- Chef receives 92% of food value, platform keeps 8%

---

### 6. **Order History & Reordering**

**Order History:**
- Cursor-based pagination (infinite scroll)
- Filter by status: All, Delivered, Cancelled
- Displays: Order ID, Date, Chef name, Total, Status
- Thumbnail: First item's image
- Quick actions: Track, Reorder, Rate

**Reorder Functionality:**
- Tap "Reorder" button
- All items from previous order added to cart
- Pricing uses **current menu prices** (not historical)
- Handles unavailable items gracefully:
  - Shows warning: "2 items no longer available"
  - Only available items added to cart
- User can modify quantities before checkout

---

### 7. **Real-Time Order Tracking**

**Live Status Screen:**
- Order status badge with color coding:
  - 🟡 Preparing → Yellow
  - 🔵 Out for Delivery → Blue
  - 🟢 Delivered → Green
  - 🔴 Cancelled → Red
- Estimated preparation time (e.g., "25 mins remaining")
- Delivery ETA (e.g., "Arriving in 10 mins")
- Rider information (name, phone, profile picture)
- Live location tracking (map with rider's position)
- Status history timeline

**Telemetry Events:**
- Customer views order status screen → Tracked
- Customer taps "Track Order" → Tracked
- Data used for Explore ranking (engagement signals)

---

### 8. **Coin Rewards System (MVP Beta)**

**How It Works:**
- Customer receives 100% of dish price back as coins on delivery
- Example: Order ₹450 worth of food → Earn 450 coins
- Coins credited automatically when order status = DELIVERED
- Coins can be redeemed for future orders

**Business Rationale:**
- Encourages first-time purchases (perceived as "free")
- Builds habit loop (users return to spend coins)
- Increases customer lifetime value
- Time-limited beta feature to drive adoption

---

### 9. **Fraud & Abuse Detection**

**Order Abuse Signals:**
- Excessive order cancellations (>3 per week)
- COD orders with no-delivery patterns
- Multiple failed payment attempts
- Rapid order creation (rate limiting)
- Geographic abuse (ordering from too far repeatedly)

**Trust State System:**
```
NORMAL → WARNING → RESTRICTED → BANNED
```

**Restrictions by Trust State:**

| Trust State | Max COD Orders/Day | Can Cancel After Payment? | Rate Limit |
|-------------|--------------------|-----------------------------|------------|
| NORMAL      | 3                  | Yes (with penalty)          | 20/min     |
| WARNING     | 1                  | Yes (with penalty)          | 10/min     |
| RESTRICTED  | 0                  | No (must contact support)   | 5/min      |
| BANNED      | 0                  | No                          | 0          |

**Abuse Policy Logger:**
- All abuse-related actions logged to `audit_events`
- Includes: user ID, order ID, violation type, action taken
- Used for pattern analysis and appeals

---

### 10. **Auto-Cancellation System**

**Scenarios:**

#### A. Payment Timeout
- Lock duration: 15 minutes
- Reason: "Payment not completed within time limit"
- Inventory released back to chef

#### B. Rider Assignment Failure
- Timeout: 5 minutes after order marked "ready"
- Reason: "No rider accepted the delivery"
- Customer refunded automatically

#### C. Pickup Timeout
- Timeout: 30 minutes after rider assignment
- Reason: "Rider did not pick up order"
- Customer refunded, rider penalized

#### D. Delivery Timeout
- Timeout: 2 hours after rider pickup
- Reason: "Order not delivered within expected time"
- Customer refunded, escalated to support

**Notifications:**
- Customer notified via push + in-app
- Chef notified if order was accepted
- Rider notified if assigned
- Refund initiated automatically for paid orders

---

## 🔗 **Integration Points**

### Internal Modules

| Module | Integration Purpose | Direction |
|--------|---------------------|-----------|
| **Cart** | Order creation from cart | Cart → Order |
| **Payment** | Razorpay order creation | Order → Payment |
| **Chef-Kitchen** | Menu item retrieval, availability check | Order ↔ Chef-Kitchen |
| **Delivery** | Rider assignment, tracking | Order ↔ Delivery |
| **Commission** | Commission calculation on delivery | Order → Commission |
| **Notification** | Order status updates | Order → Notification |
| **User** | Coin crediting on delivery | Order → User |
| **Analytics** | Order conversion tracking | Order → Analytics |
| **Review** | Post-delivery rating trigger | Order → Review |
| **Address** | Address snapshot at checkout | Order → Address |

### External Services

| Service | Purpose | Criticality |
|---------|---------|-------------|
| **Razorpay** | UPI Intent payment processing | Critical |
| **AWS MediaConvert** | Order receipt generation (future) | Optional |
| **Courier APIs** | Delivery partner integration (future) | Optional |

---

## 📊 **Key Metrics**

### Business Metrics
- **Order Conversion Rate**: (Orders Paid / Orders Created) × 100
- **Average Order Value (AOV)**: Total Revenue / Orders Delivered
- **Reorder Rate**: (Customers with >1 order / Total customers) × 100
- **Order Cancellation Rate**: (Cancelled Orders / Total Orders) × 100
- **COD Success Rate**: (COD Delivered / COD Created) × 100

### Operational Metrics
- **Order Fulfillment Time**: Time from PAID → DELIVERED
- **Chef Acceptance Rate**: (Accepted Orders / Paid Orders) × 100
- **Delivery Success Rate**: (Delivered Orders / Out for Delivery) × 100
- **Payment Success Rate**: (Paid Orders / Payment Pending) × 100

### User Experience Metrics
- **Time to Checkout**: Duration from cart view → payment initiated
- **Order Tracking Views**: Number of times customer checks status
- **Reorder Rate**: Percentage of users who reorder within 7 days

---

## 🚨 **Business Rules & Constraints**

### Order Creation
- ✅ Customer must be authenticated
- ✅ Cart must have at least 1 item
- ✅ All items must be from the same chef
- ✅ Chef must be accepting orders (kitchen online)
- ✅ Chef must be within service hours (if schedule set)
- ✅ Minimum order value: ₹50
- ✅ Maximum items per order: 20
- ✅ Rate limit: 20 order creations per minute per user

### Checkout
- ✅ Order must be in DRAFT or CREATED status
- ✅ Customer must have a valid delivery address
- ✅ Delivery distance ≤ 15 km
- ✅ Address must have valid coordinates
- ✅ Rate limit: 10 checkouts per minute per user
- ✅ Idempotency: Same order can't be checked out twice

### Payment
- ✅ Payment lock expires after 15 minutes
- ✅ UPI Intent URL valid for 15 minutes
- ✅ Only one payment intent per order
- ✅ Razorpay signature must be verified
- ✅ COD orders auto-confirmed without UPI flow

### Cancellation
- ✅ NORMAL users: Can cancel before OUT_FOR_DELIVERY
- ✅ WARNING users: Can cancel with penalty
- ✅ RESTRICTED users: Cannot cancel, must contact support
- ✅ System can auto-cancel for timeouts (no restrictions)
- ✅ Cancellation after pickup incurs trust penalty

### Reordering
- ✅ Only DELIVERED or CANCELLED orders can be reordered
- ✅ Uses current pricing, not historical prices
- ✅ Unavailable items excluded from reorder
- ✅ Cart cleared before reorder items added

---

## 🛡️ **Security & Compliance**

### Data Protection
- 🔒 Order details only visible to:
  - Customer who placed the order
  - Chef who received the order
  - Assigned delivery rider
  - Admin with proper role
- 🔒 Payment credentials never stored (tokenized by Razorpay)
- 🔒 Address snapshots protect customer privacy (no references to live address)

### Payment Security
- ✅ Razorpay PCI-DSS compliant
- ✅ Signature verification for all payments
- ✅ Webhook IP whitelisting (future enhancement)
- ✅ No plain-text payment data in logs

### Fraud Prevention
- ✅ Rate limiting on all endpoints
- ✅ Distributed locks prevent double-payment
- ✅ Trust state system limits abuse
- ✅ Abuse signals logged for pattern analysis
- ✅ System-level overrides for auto-cancellations

### FSSAI Compliance
- ✅ Chef's FSSAI license linked to orders
- ✅ Order receipts include FSSAI number (future)
- ✅ Food type (veg/non-veg) clearly labeled

---

## 🔄 **User Workflows**

### Workflow 1: Standard Order Placement

```
1. Customer browses chef's menu
2. Adds 3 items to cart (₹450 subtotal)
3. Taps "Proceed to Checkout"
4. Selects delivery address
5. Views pricing breakdown:
   - Subtotal: ₹450
   - Delivery: ₹30
   - GST: ₹24
   - Total: ₹504
6. Chooses payment method: UPI Intent
7. Order created with 15-minute payment lock
8. Razorpay order generated
9. UPI app opens (GPay)
10. Customer confirms payment in GPay
11. Razorpay webhook confirms payment
12. Order status → PAID
13. Chef receives notification
14. Chef accepts order (estimated 25 mins)
15. Order status → ACCEPTED → PREPARING
16. Chef marks as READY
17. Rider assigned automatically
18. Rider picks up order → OUT_FOR_DELIVERY
19. Customer tracks live location
20. Rider delivers → DELIVERED
21. Customer receives 450 coins
22. Rating prompt appears
```

**Duration**: ~45 minutes (typical)

---

### Workflow 2: Reorder Flow

```
1. Customer opens "My Orders"
2. Finds previous order "Margherita Pizza + Garlic Bread"
3. Taps "Reorder" button
4. System checks item availability:
   - Margherita Pizza: ✅ Available (₹250 → ₹270 updated)
   - Garlic Bread: ❌ Unavailable
5. Cart cleared and Margherita Pizza added
6. Toast: "1 item no longer available"
7. Customer adjusts quantity or proceeds
8. Checkout flow (same as Workflow 1)
```

**Duration**: ~2 minutes to cart

---

### Workflow 3: COD Order Flow

```
1-6. (Same as Workflow 1, steps 1-6)
7. Chooses payment method: Cash on Delivery
8. Order auto-confirmed (no UPI flow)
9. Order status → PAID (trust-based)
10-19. (Same as Workflow 1, steps 13-19)
20. Rider collects cash from customer
21. Rider marks as DELIVERED
22. Chef receives payout (minus platform fee)
23. Customer receives 450 coins
```

**Benefits**: No payment gateway dependency, accessible to all users

---

## 📱 **Frontend Screens**

### Screen 1: Order History (`/orders`)
- **Purpose**: View all past and active orders
- **Components**:
  - Filter tabs: All, Active, Delivered, Cancelled
  - Order cards with thumbnail, chef name, date, total, status
  - "Track Order" button for active orders
  - "Reorder" button for delivered orders
  - Pull-to-refresh
  - Infinite scroll pagination

### Screen 2: Order Details (`/orders/[orderId]`)
- **Purpose**: View order details and track status
- **Components**:
  - Status badge with color coding
  - Order ID and date
  - Items list with quantities and prices
  - Delivery address snapshot
  - Pricing breakdown
  - Chef info card
  - Rider info card (when assigned)
  - "Cancel Order" button (conditional)
  - "Rate Order" button (after delivery)
  - "Help" button (contact support)

### Screen 3: Order Tracking (`/orders/[orderId]` - Live Mode)
- **Purpose**: Real-time order tracking
- **Components**:
  - Live status timeline
  - Estimated preparation time
  - Delivery ETA with countdown
  - Rider information (name, phone, profile pic)
  - Live map with rider's location (future)
  - "Call Rider" button
  - Auto-refresh every 30 seconds

### Screen 4: Rate Order (`/orders/[orderId]/rate-order`)
- **Purpose**: Post-delivery rating and review
- **Components**:
  - 5-star rating for food quality
  - 5-star rating for delivery experience
  - Text review (optional)
  - Upload photos (optional)
  - Tips for rider (optional)
  - Submit button

---

## 🚀 **Future Enhancements**

### Phase 1 (Q2 2026)
- [ ] Group ordering (multiple users contributing to one order)
- [ ] Scheduled orders (order for later delivery)
- [ ] Order splitting (share bill with friends)

### Phase 2 (Q3 2026)
- [ ] Subscription orders (weekly meal plans)
- [ ] Bulk orders for events
- [ ] Corporate ordering (office lunch)

### Phase 3 (Q4 2026)
- [ ] Order insurance (refund guarantee)
- [ ] Live video feed from kitchen (transparency)
- [ ] Drone delivery integration

---

## 📚 **Related Documentation**

- **Technical Guide**: [TECHNICAL_GUIDE.md](./TECHNICAL_GUIDE.md)
- **QA Test Cases**: [QA_TEST_CASES.md](./QA_TEST_CASES.md)
- **Cart Module**: [../cart/FEATURE_OVERVIEW.md](../cart/FEATURE_OVERVIEW.md)
- **Payment Integration**: [../../integrations/PAYMENT_FLOW_COMPLETE.md](../../integrations/PAYMENT_FLOW_COMPLETE.md)
- **Commission System**: [../commission/FEATURE_OVERVIEW.md](../commission/FEATURE_OVERVIEW.md)
- **Delivery Module**: [../delivery/FEATURE_OVERVIEW.md](../delivery/FEATURE_OVERVIEW.md)

---

## ✅ **Completion Status**

**Feature Coverage**: 100%  
**Documentation**: Complete  
**Production Status**: ✅ Live  
**Last Audit**: 2026-02-15

---

**[MODULE_COMPLETE ✅]**
