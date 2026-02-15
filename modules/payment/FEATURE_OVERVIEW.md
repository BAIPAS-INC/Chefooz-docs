# 💳 Payment Module — Feature Overview

**Status**: ✅ COMPLETE  
**Date**: 2025-02-15  
**Module**: Payment Management (Razorpay Integration)  
**Related Docs**: [Order Module](../order/), [Checkout Module](../checkout/)

---

## 📋 Overview

The Payment module provides a secure, reliable integration with Razorpay payment gateway for processing online payments in the Chefooz platform. It handles the complete payment lifecycle from order creation through payment verification, including webhook handling for asynchronous payment confirmations.

**Key Points:**
- ✅ Razorpay payment gateway integration with Orders API
- ✅ UPI Intent payments with deep linking support
- ✅ Secure HMAC SHA256 signature verification
- ✅ Payment intent tracking for idempotency
- ✅ Webhook handling for asynchronous payment notifications
- ✅ COD (Cash on Delivery) bypass (handled in Order module)

---

## 🎯 Business Purpose

### Why This Module Exists

**Problem Solved:**
- Enable secure online payments for food orders
- Support multiple payment methods (UPI, cards, wallets, net banking)
- Handle payment failures and retries gracefully
- Provide payment reconciliation data for finance team
- Prevent duplicate charges through idempotent processing

**Value Propositions:**

**For Customers:**
- Multiple payment options through Razorpay
- Fast UPI Intent checkout (no OTP delays)
- Secure payment processing with industry-standard encryption
- Clear payment status tracking
- Automatic refund processing for cancellations

**For Chefs:**
- Guaranteed payment before order acceptance
- No cash handling required for online orders
- Faster settlement through Razorpay
- Protection from payment frauds

**For Platform:**
- Lower operational costs (no cash collection)
- Better conversion rates with multiple payment methods
- Automated reconciliation through Razorpay dashboard
- Fraud detection via Razorpay's risk engine
- Commission calculation on confirmed payments only

---

## 👥 User Personas

### 1. **Online Customer (Primary User)**
**Characteristics:**
- Prefers digital payments over cash
- Uses UPI apps (Google Pay, PhonePe, Paytm)
- Expects fast, frictionless checkout
- Concerned about payment security

**Goals:**
- Complete payment quickly (under 30 seconds)
- Use preferred payment method
- Get instant payment confirmation
- Track payment status in real-time

**Pain Points:**
- Long checkout processes
- Payment failures without clear error messages
- Duplicate charges
- Unclear refund timelines

### 2. **Chef (Order Recipient)**
**Characteristics:**
- Relies on confirmed payments before preparing food
- Monitors real-time order notifications
- Concerned about payment settlement timelines

**Goals:**
- Receive orders only after payment confirmation
- Get payment settlement within 2-3 business days
- Minimize order cancellations due to payment issues

**Pain Points:**
- False payment confirmations
- Delayed payment settlements
- Chargebacks and disputes

### 3. **Finance Team (Admin User)**
**Characteristics:**
- Manages payment reconciliation
- Handles refund requests
- Monitors payment gateway health

**Goals:**
- Reconcile payments daily
- Process refunds within 24 hours
- Track payment success rates
- Identify payment gateway issues

**Pain Points:**
- Manual reconciliation effort
- Unclear payment failure reasons
- Missing webhook events
- Duplicate payment entries

---

## 🔑 Key Features

### 1. **Razorpay Order Creation**

**What It Does:**
Creates a Razorpay order before initiating payment, establishing payment intent and enabling signature verification.

**Business Rules:**
- Amount must match cart total (validated server-side)
- Minimum order value: ₹1 (100 paise)
- Currency: INR only
- Order validity: 15 minutes (auto-cancels after timeout)
- Generates unique receipt ID: `order_{userId}_{timestamp}`

**User Experience:**
1. User clicks "Pay Now" on checkout screen
2. App calls backend to create Razorpay order
3. Backend validates cart and creates Razorpay order
4. App receives order details (orderId, amount, keyId)
5. App opens Razorpay SDK with order details

**Technical Flow:**
```
Customer App → Backend API → Razorpay API → Backend DB → Customer App
                (Validate)    (Create Order)  (Save Intent)  (SDK Ready)
```

---

### 2. **Payment Intent Tracking**

**What It Does:**
Stores payment attempt metadata in database for idempotency, reconciliation, and debugging.

**Business Rules:**
- One payment intent per Razorpay order ID (unique constraint)
- Stores cart snapshot for post-payment order creation
- Tracks payment status: created → paid/failed/refunded
- Links to order after successful payment
- Retains webhook payload for audit trail

**User Experience:**
- Invisible to user (backend tracking)
- Enables retry logic without duplicate charges
- Provides payment history in user dashboard

**Data Stored:**
```
PaymentIntent:
  - id (UUID)
  - userId
  - razorpayOrderId
  - razorpayPaymentId (after payment)
  - amount (paise)
  - status (created/paid/failed/refunded)
  - orderId (linked after payment)
  - metadata (cart snapshot, address)
  - webhookPayload (for reconciliation)
  - failureReason (if failed)
```

---

### 3. **Payment Signature Verification**

**What It Does:**
Verifies Razorpay payment authenticity using HMAC SHA256 signature, preventing payment tampering and fraud.

**Business Rules:**
- Signature = HMAC_SHA256(order_id|payment_id, secret_key)
- Signature must match Razorpay-provided signature
- Verification happens twice: frontend callback + webhook
- Failed verification = payment rejected + no order created

**User Experience:**
1. User completes payment in Razorpay SDK
2. Razorpay returns: order_id, payment_id, signature
3. App sends all three to backend
4. Backend verifies signature
5. If valid: order created, user sees success screen
6. If invalid: error shown, payment refunded

**Security Measures:**
- Secret key stored in backend environment variables only
- Signature never exposed to frontend
- Webhook signature verified separately
- All failed verifications logged to audit events

---

### 4. **Webhook Handling**

**What It Does:**
Receives asynchronous payment notifications from Razorpay for payment state changes, ensuring eventual consistency.

**Business Rules:**
- Webhook signature verified before processing
- Events processed: payment.captured, payment.failed, order.paid
- Idempotent processing (duplicate events ignored)
- Notifications sent to user on payment status change

**Supported Events:**
1. **payment.captured**: Payment successfully collected
   - Update payment intent to 'paid'
   - Create order if not already created
   - Send success notification

2. **payment.failed**: Payment declined by bank/UPI app
   - Update payment intent to 'failed'
   - Store failure reason
   - Send failure notification with retry option

3. **order.paid**: Order fully paid (all payments captured)
   - Finalize order creation
   - Update payment intent with order ID
   - Trigger chef notification

**User Experience:**
- User may receive payment confirmation via webhook even if app crashes during payment
- Payment status updated in background
- Push notification sent when payment confirmed

**Webhook URL:**
```
POST https://api.chefooz.com/api/v1/payment/webhook
Header: x-razorpay-signature
```

---

### 5. **Payment Status Tracking**

**What It Does:**
Provides real-time payment status lookup for users and system components.

**Business Rules:**
- Status updated in real-time as payment progresses
- Polling interval: 5 seconds (frontend)
- States: created, paid, failed, refunded
- Accessible only by payment intent owner (security)

**User Experience:**
1. After initiating payment, user sees loading screen
2. App polls payment status every 5 seconds
3. When status changes to 'paid', show success screen
4. If status changes to 'failed', show retry option
5. Timeout after 5 minutes, assume payment failed

**Use Cases:**
- Payment confirmation screen polling
- Order history payment badge colors
- Admin reconciliation dashboard
- Customer support debugging

---

### 6. **Refund Processing** (Future Enhancement)

**What It Does:**
Initiates refunds for cancelled orders or failed deliveries.

**Business Rules (Planned):**
- Refunds processed within 24 hours
- Refund amount = original payment amount
- Razorpay fees not refunded
- Refund status: initiated → processing → completed
- Notification sent on refund completion

**Current Status:**
- Manual refunds via Razorpay dashboard
- Automated refunds planned for Phase 2

---

## 🔗 Integration Points

### Internal Module Dependencies

| Module | Relationship | Purpose |
|--------|--------------|---------|
| **Order** | Upstream Dependency | Creates order after payment verification |
| **Cart** | Data Source | Cart items included in payment intent metadata |
| **User** | Ownership | Payment intents linked to user ID |
| **Notification** | Downstream Consumer | Sends payment success/failure notifications |
| **Audit-Events** | Downstream Consumer | Logs all payment verification attempts |
| **Commission** | Indirect | Commission calculated only for paid orders |
| **Analytics** | Downstream Consumer | Tracks payment success rates, failure reasons |

### External Service Dependencies

| Service | Purpose | Failure Handling |
|---------|---------|------------------|
| **Razorpay API** | Order creation, payment processing | Show error to user, retry after 30 seconds |
| **Razorpay Webhooks** | Asynchronous payment updates | Retry up to 10 times with exponential backoff (Razorpay-side) |

---

## 📊 Key Business Metrics

### Success Metrics
- **Payment Success Rate**: Target 95%+
  - Formula: (Paid Orders / Total Payment Attempts) × 100
- **Average Payment Time**: Target < 30 seconds
  - From "Pay Now" click to payment confirmation
- **Webhook Delivery Rate**: Target 99%+
  - Percentage of webhook events successfully processed

### Failure Metrics
- **Payment Failure Rate**: Track < 5%
  - Breakdown by reason: insufficient funds, bank decline, technical error
- **Signature Verification Failures**: Track < 0.1%
  - Indicates potential fraud attempts or integration bugs
- **Payment Gateway Downtime**: Track < 0.01%
  - Razorpay API unavailability

### Financial Metrics
- **Gross Merchandise Value (GMV)**: Total payment volume
- **Average Order Value (AOV)**: Average payment amount
- **Refund Rate**: Target < 2%
  - (Refunded Orders / Paid Orders) × 100

---

## 🛡️ Security & Compliance

### Payment Security

**Data Protection:**
- Payment card details never stored in Chefooz database
- Razorpay handles PCI-DSS compliance
- Only payment IDs and order IDs stored
- Webhook payload stored for audit (no sensitive data)

**Signature Verification:**
- HMAC SHA256 with secret key
- Prevents payment amount tampering
- Prevents payment ID spoofing
- Webhook signature verified separately

**Access Control:**
- Payment intents accessible only by owner
- JWT authentication required on all endpoints
- Webhook endpoint signature-protected (no JWT)

### Compliance

**PCI-DSS:**
- Handled entirely by Razorpay
- Chefooz is not in PCI-DSS scope

**RBI Guidelines:**
- UPI transactions comply with NPCI guidelines
- Refunds processed per RBI timelines
- Payment failures clearly communicated

**Data Privacy:**
- Payment intent metadata encrypted at rest
- Personal data (address, items) stored in metadata
- Data retention: 7 years (compliance requirement)

### Fraud Prevention

**Measures:**
- Razorpay's fraud detection engine
- Rate limiting on payment creation (20/min per user)
- Distributed locks prevent concurrent payments
- Failed signature verifications logged for analysis

---

## 📱 User Workflows

### Workflow 1: Successful UPI Payment

**Scenario:** Customer completes payment using Google Pay

**Steps:**

1. **Initiate Checkout**
   - User: Clicks "Proceed to Pay" on checkout screen
   - App: Validates cart items
   - App: Calls `POST /api/v1/payment/create-order`
   - Backend: Creates Razorpay order
   - Backend: Saves payment intent with status='created'
   - Backend: Returns orderId, amount, keyId

2. **Open Razorpay SDK**
   - App: Opens Razorpay checkout SDK
   - SDK: Shows payment options (UPI, cards, wallets)
   - User: Selects "UPI" → "Google Pay"
   - SDK: Opens Google Pay via Intent URL

3. **Complete Payment**
   - User: Approves payment in Google Pay
   - Google Pay: Completes UPI transaction
   - SDK: Receives payment confirmation
   - SDK: Returns razorpay_payment_id, razorpay_signature

4. **Verify Payment**
   - App: Calls `POST /api/v1/payment/verify`
   - Backend: Verifies HMAC signature
   - Backend: Updates payment intent to 'paid'
   - Backend: Calls OrderService.placePrepaidOrder()
   - Backend: Returns orderId

5. **Show Success**
   - App: Navigates to order confirmation screen
   - App: Shows order ID, estimated delivery time
   - Backend: Sends push notification to chef
   - Backend: Sends push notification to user

**Duration:** 20-45 seconds

---

### Workflow 2: Payment Failure

**Scenario:** Customer's payment fails due to insufficient balance

**Steps:**

1. **Initiate Checkout** (same as success flow)

2. **Open Razorpay SDK** (same as success flow)

3. **Payment Fails**
   - User: Approves payment in UPI app
   - Bank: Declines due to insufficient funds
   - UPI App: Shows "Payment Failed"
   - SDK: Receives payment failure event
   - SDK: Calls onFailure callback

4. **Handle Failure**
   - App: Shows error message: "Payment failed. Please try again."
   - Backend: Receives webhook: payment.failed
   - Backend: Updates payment intent to 'failed'
   - Backend: Stores failure reason: "Insufficient funds"

5. **Retry or Exit**
   - User Option 1: Clicks "Retry Payment" → restart flow
   - User Option 2: Clicks "Cancel" → return to cart
   - System: Auto-cancels order after 15 minutes

**Duration:** 15-30 seconds

---

### Workflow 3: Webhook-Based Confirmation

**Scenario:** User closes app during payment, webhook confirms payment later

**Steps:**

1. **Initiate Checkout** (same as success flow)

2. **User Closes App**
   - User: Approves payment in UPI app
   - User: Closes app before SDK callback
   - Payment: Completes successfully on Razorpay side

3. **Webhook Arrives**
   - Razorpay: Sends payment.captured webhook
   - Backend: Receives webhook at `/payment/webhook`
   - Backend: Verifies webhook signature
   - Backend: Finds payment intent by razorpayOrderId

4. **Complete Order**
   - Backend: Updates payment intent to 'paid'
   - Backend: Creates order via OrderService
   - Backend: Links order to payment intent
   - Backend: Sends push notification: "Payment confirmed! Order placed."

5. **User Returns**
   - User: Opens app
   - App: Fetches orders
   - App: Shows newly created order in "Active Orders"

**Duration:** 1-10 minutes (depends on user return)

---

## 🖥️ Frontend Screens

### 1. Checkout Screen (Pre-Payment)
**File:** `apps/chefooz-app/src/app/(main)/checkout/index.tsx`

**Components:**
- Cart items list (read-only)
- Delivery address card (editable)
- Pricing breakdown (subtotal, delivery, GST, total)
- "Proceed to Pay" button (primary CTA)

**State Management:**
- Cart items (Zustand: cartStore)
- Selected address (Zustand: addressStore)
- Payment loading state (local state)

**User Actions:**
- Edit delivery address
- Apply promo code (if available)
- Click "Proceed to Pay" → trigger payment flow

---

### 2. Payment Processing Screen
**File:** `apps/chefooz-app/src/app/(main)/payment-processing/index.tsx`

**Components:**
- Loading spinner
- "Processing Payment..." text
- Razorpay SDK modal (overlay)
- Cancel button (bottom)

**State Management:**
- Payment intent ID (route params)
- Payment status polling (React Query)

**User Actions:**
- Complete payment in Razorpay SDK
- Click "Cancel" to abort payment

**Razorpay SDK Options:**
```typescript
const options = {
  key: razorpayKeyId,
  amount: amount, // paise
  currency: 'INR',
  order_id: orderId,
  name: 'Chefooz',
  description: 'Food Order Payment',
  prefill: {
    email: user.email,
    contact: user.phone,
  },
  theme: {
    color: '#FF6B35', // Chefooz brand color
  },
};
```

---

### 3. Payment Success Screen
**File:** `apps/chefooz-app/src/app/(main)/payment-success/index.tsx`

**Components:**
- Success icon (green checkmark animation)
- "Payment Successful!" header
- Order ID display
- Estimated delivery time
- "View Order" button (primary CTA)
- "Go to Home" button (secondary)

**State Management:**
- Order ID (route params)
- Order details (React Query: useOrder)

**User Actions:**
- Click "View Order" → navigate to order tracking screen
- Click "Go to Home" → navigate to home feed

---

### 4. Payment Failure Screen
**File:** `apps/chefooz-app/src/app/(main)/payment-failed/index.tsx`

**Components:**
- Error icon (red X animation)
- "Payment Failed" header
- Failure reason text
- "Retry Payment" button (primary CTA)
- "Back to Cart" button (secondary)

**State Management:**
- Payment intent ID (route params)
- Failure reason (from payment status API)

**User Actions:**
- Click "Retry Payment" → recreate payment intent, reopen SDK
- Click "Back to Cart" → navigate to cart screen

---

## 🧪 Testing Considerations

### Test Scenarios

**Happy Path:**
1. ✅ Create payment intent successfully
2. ✅ Verify valid signature and create order
3. ✅ Receive and process payment.captured webhook
4. ✅ Track payment status correctly

**Error Handling:**
1. ⚠️ Razorpay API down during order creation
2. ⚠️ Invalid signature verification (fraud attempt)
3. ⚠️ Webhook signature verification failure
4. ⚠️ Duplicate webhook events (idempotency check)
5. ⚠️ Payment intent not found during verification

**Edge Cases:**
1. 🔍 Amount mismatch between cart and payment intent
2. 🔍 User closes app during payment (webhook recovers)
3. 🔍 Payment timeout (15-minute expiry)
4. 🔍 Concurrent payment attempts (distributed locks)
5. 🔍 Webhook arrives before SDK callback

### Test Data

**Test Cards (Razorpay Test Mode):**
- Success: 4111 1111 1111 1111, CVV: 123, Expiry: Any future date
- Failure: 4000 0000 0000 0002

**Test UPI IDs:**
- Success: success@razorpay
- Failure: failure@razorpay

---

## 🚀 Future Enhancements

### Phase 1 (Current Release)
- ✅ Razorpay order creation
- ✅ Payment signature verification
- ✅ Webhook handling
- ✅ Payment status tracking

### Phase 2 (Q2 2026)
- 📋 Automated refund processing
- 📋 Partial refunds (delivery fee retained)
- 📋 Payment method preferences storage
- 📋 Saved cards / UPI IDs (tokenization)

### Phase 3 (Q3 2026)
- 📋 Alternative payment gateways (Cashfree, Stripe)
- 📋 International payments (Stripe Integration)
- 📋 EMI options for high-value orders
- 📋 Subscription payments (recurring orders)

### Phase 4 (Q4 2026)
- 📋 Split payments (multiple contributors)
- 📋 Wallet balance (Chefooz wallet)
- 📋 Loyalty points redemption
- 📋 Gift card payments

---

## 📚 Related Documentation

**Technical Documentation:**
- [Payment Technical Guide](./TECHNICAL_GUIDE.md)
- [Payment QA Test Cases](./QA_TEST_CASES.md)

**Integration Guides:**
- [Order Module](../order/)
- [Checkout Module](../checkout/)

**Setup Guides:**
- `application-guides/RAZORPAY_PAYMENT_SETUP.md`
- `application-guides/PAYMENT_INTENT_IDEMPOTENCY_FIX.md`

---

**[PAYMENT_FEATURE_OVERVIEW_COMPLETE ✅]**
