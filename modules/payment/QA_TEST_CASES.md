# 💳 Payment Module — QA Test Cases

**Status**: ✅ COMPLETE  
**Last Updated**: 2025-02-15  
**Module**: Payment (Razorpay Integration)  
**Test Environment**: Staging + Razorpay Test Mode  

---

## 📋 Table of Contents

1. [Test Environment Setup](#test-environment-setup)
2. [Functional Tests](#functional-tests)
3. [UX/UI Tests](#uxui-tests)
4. [Edge Case Tests](#edge-case-tests)
5. [Error Handling Tests](#error-handling-tests)
6. [Security Tests](#security-tests)
7. [Performance Tests](#performance-tests)
8. [Regression Tests](#regression-tests)
9. [Platform-Specific Tests](#platform-specific-tests)
10. [Test Execution Summary](#test-execution-summary)

---

## 🛠️ Test Environment Setup

### Prerequisites

**Backend:**
- NestJS API running on staging (`https://staging-api.chefooz.com`)
- PostgreSQL database accessible
- Razorpay Test Mode enabled
- Environment variables configured:
  ```env
  RAZORPAY_KEY_ID=rzp_test_xxxxxxxxxxxxxx
  RAZORPAY_KEY_SECRET=test_secret_key
  RAZORPAY_WEBHOOK_SECRET=test_webhook_secret
  ```

**Mobile App:**
- Expo app built with staging API endpoint
- react-native-razorpay installed and configured
- Test user accounts created

**Tools:**
- Postman/Insomnia for API testing
- Razorpay Dashboard (Test Mode) for webhook monitoring
- Database client (pgAdmin/DBeaver) for data verification

---

### Test Accounts

**Test User 1 (Customer - Normal Trust):**
```
Email: test.customer1@chefooz.com
Phone: +919876543210
User ID: 550e8400-e29b-41d4-a716-446655440001
Trust State: NORMAL
```

**Test User 2 (Customer - Restricted Trust):**
```
Email: test.customer2@chefooz.com
Phone: +919876543211
User ID: 550e8400-e29b-41d4-a716-446655440002
Trust State: RESTRICTED
```

**Test Chef:**
```
Email: chef.test@chefooz.com
Phone: +919876543220
User ID: 550e8400-e29b-41d4-a716-446655440010
```

---

### Test Payment Methods

**Razorpay Test Cards:**
```
Success Card:
  Number: 4111 1111 1111 1111
  CVV: 123
  Expiry: 12/30

Failure Card (Insufficient Funds):
  Number: 4000 0000 0000 0002
  CVV: 123
  Expiry: 12/30
```

**Razorpay Test UPI IDs:**
```
Success: success@razorpay
Failure: failure@razorpay
```

---

### Test Data

**Test Menu Item:**
```json
{
  "id": "menu-item-001",
  "name": "Butter Chicken",
  "price": 35000,
  "chefId": "550e8400-e29b-41d4-a716-446655440010"
}
```

**Test Cart:**
```json
{
  "items": [
    {
      "productId": "menu-item-001",
      "quantity": 2,
      "unitPrice": 35000
    }
  ],
  "totalAmount": 70000
}
```

**Test Address:**
```json
{
  "line1": "Test Address 123, MG Road",
  "city": "Mumbai",
  "state": "Maharashtra",
  "postalCode": "400001",
  "latitude": 19.0760,
  "longitude": 72.8777
}
```

---

## ✅ Functional Tests

### TC-PAY-F-001: Create Razorpay Order Successfully

**Priority:** Critical  
**Type:** Functional  
**Preconditions:**
- User logged in
- Cart has 1+ items
- Delivery address selected

**Test Steps:**
1. Navigate to checkout screen
2. Verify cart items displayed
3. Click "Proceed to Pay" button
4. Observe API request to `/api/v1/payment/create-order`

**Expected Result:**
- API returns 201 status
- Response contains:
  ```json
  {
    "success": true,
    "data": {
      "paymentIntentId": "<uuid>",
      "orderId": "order_<razorpay_id>",
      "amount": 70000,
      "currency": "INR",
      "keyId": "rzp_test_xxxxx"
    }
  }
  ```
- Database: PaymentIntent created with status='created'
- Razorpay SDK opens with payment options

**Test Data:**
- User: test.customer1@chefooz.com
- Cart: 2x Butter Chicken (₹700)

**Actual Result:** _______________  
**Status:** ⬜ Pass ⬜ Fail  
**Notes:** _______________

---

### TC-PAY-F-002: Complete UPI Payment Successfully

**Priority:** Critical  
**Type:** Functional  
**Preconditions:**
- Razorpay order created (TC-PAY-F-001 passed)
- Razorpay SDK open

**Test Steps:**
1. In Razorpay SDK, select "UPI"
2. Enter test UPI ID: `success@razorpay`
3. Click "Pay"
4. Observe SDK callback with payment details
5. App calls `/api/v1/payment/verify`
6. Observe API response

**Expected Result:**
- SDK returns success with:
  - `razorpay_order_id`
  - `razorpay_payment_id`
  - `razorpay_signature`
- API returns 200 status:
  ```json
  {
    "success": true,
    "message": "Payment verified successfully",
    "data": {
      "paymentIntentId": "<uuid>",
      "orderId": "<uuid>",
      "razorpayPaymentId": "pay_xxxxx"
    }
  }
  ```
- Database: PaymentIntent status updated to 'paid'
- Database: Order created with status='PAID'
- Push notification sent to chef
- App navigates to payment success screen

**Test Data:**
- UPI ID: success@razorpay
- Amount: ₹700

**Actual Result:** _______________  
**Status:** ⬜ Pass ⬜ Fail  
**Notes:** _______________

---

### TC-PAY-F-003: Complete Card Payment Successfully

**Priority:** High  
**Type:** Functional  
**Preconditions:**
- Razorpay order created

**Test Steps:**
1. In Razorpay SDK, select "Cards"
2. Enter test card: 4111 1111 1111 1111
3. CVV: 123, Expiry: 12/30
4. Click "Pay"
5. Observe payment flow

**Expected Result:**
- Payment succeeds
- Order created successfully
- User sees payment success screen

**Test Data:**
- Card: 4111 1111 1111 1111

**Actual Result:** _______________  
**Status:** ⬜ Pass ⬜ Fail  
**Notes:** _______________

---

### TC-PAY-F-004: Handle Payment Failure

**Priority:** Critical  
**Type:** Functional  
**Preconditions:**
- Razorpay order created

**Test Steps:**
1. In Razorpay SDK, select "UPI"
2. Enter test UPI ID: `failure@razorpay`
3. Click "Pay"
4. Observe SDK callback

**Expected Result:**
- SDK returns failure callback
- App shows error message: "Payment failed. Please try again."
- Database: PaymentIntent status remains 'created' (or marked 'failed' via webhook)
- User can retry payment
- No order created

**Test Data:**
- UPI ID: failure@razorpay

**Actual Result:** _______________  
**Status:** ⬜ Pass ⬜ Fail  
**Notes:** _______________

---

### TC-PAY-F-005: Verify Payment Status Polling

**Priority:** High  
**Type:** Functional  
**Preconditions:**
- Payment initiated
- User on payment processing screen

**Test Steps:**
1. Complete payment but close app before callback
2. Open app after 10 seconds
3. Observe payment status polling

**Expected Result:**
- App calls `/api/v1/payment/:paymentIntentId/status` every 5 seconds
- When status changes to 'paid', app navigates to success screen
- Polling stops after success

**Test Data:**
- Polling interval: 5 seconds
- Timeout: 5 minutes

**Actual Result:** _______________  
**Status:** ⬜ Pass ⬜ Fail  
**Notes:** _______________

---

### TC-PAY-F-006: Process Webhook - payment.captured

**Priority:** Critical  
**Type:** Functional  
**Preconditions:**
- Razorpay order created
- Webhook URL configured in Razorpay dashboard

**Test Steps:**
1. Manually trigger `payment.captured` webhook from Razorpay dashboard
2. Observe backend logs
3. Check database for payment intent updates

**Expected Result:**
- Webhook received at `/api/v1/payment/webhook`
- Signature verified successfully
- PaymentIntent status updated to 'paid'
- If order not created, order created now
- Webhook response: 200 OK

**Test Data:**
- Webhook event: payment.captured
- Signature: Valid HMAC SHA256

**API Test (cURL):**
```bash
curl -X POST https://staging-api.chefooz.com/api/v1/payment/webhook \
  -H "Content-Type: application/json" \
  -H "x-razorpay-signature: <calculated_signature>" \
  -d '{
    "entity": "event",
    "event": "payment.captured",
    "payload": {
      "payment": {
        "entity": {
          "id": "pay_test12345",
          "order_id": "order_test12345",
          "amount": 70000,
          "currency": "INR",
          "status": "captured",
          "method": "upi"
        }
      }
    }
  }'
```

**Actual Result:** _______________  
**Status:** ⬜ Pass ⬜ Fail  
**Notes:** _______________

---

### TC-PAY-F-007: Process Webhook - payment.failed

**Priority:** High  
**Type:** Functional  
**Preconditions:**
- Razorpay order created

**Test Steps:**
1. Trigger `payment.failed` webhook
2. Observe database updates
3. Check notification sent

**Expected Result:**
- PaymentIntent status updated to 'failed'
- Failure reason stored: "Insufficient funds"
- Push notification sent: "Payment failed. Please retry."
- No order created

**Test Data:**
- Webhook event: payment.failed

**Actual Result:** _______________  
**Status:** ⬜ Pass ⬜ Fail  
**Notes:** _______________

---

### TC-PAY-F-008: Handle Amount Mismatch

**Priority:** Critical  
**Type:** Functional  
**Preconditions:**
- User logged in

**Test Steps:**
1. Call `/api/v1/payment/create-order` with:
   ```json
   {
     "items": [{"productId": "x", "quantity": 2, "unitPrice": 35000}],
     "amount": 50000
   }
   ```
   (Calculated: 70000, Provided: 50000)
2. Observe API response

**Expected Result:**
- API returns 400 Bad Request
- Response:
  ```json
  {
    "success": false,
    "message": "Amount mismatch with cart total",
    "errorCode": "AMOUNT_MISMATCH"
  }
  ```
- No payment intent created
- User sees error: "Price mismatch. Please refresh cart."

**Test Data:**
- Calculated: ₹700
- Provided: ₹500

**Actual Result:** _______________  
**Status:** ⬜ Pass ⬜ Fail  
**Notes:** _______________

---

### TC-PAY-F-009: Verify Idempotent Payment Verification

**Priority:** High  
**Type:** Functional  
**Preconditions:**
- Payment already verified once

**Test Steps:**
1. Call `/api/v1/payment/verify` with same payment details
2. Observe response

**Expected Result:**
- API returns 200 OK
- Response:
  ```json
  {
    "success": true,
    "message": "Payment already verified",
    "data": {
      "paymentIntentId": "<uuid>",
      "orderId": "<existing_order_id>"
    }
  }
  ```
- No duplicate order created
- No additional notifications sent

**Test Data:**
- Payment already processed

**Actual Result:** _______________  
**Status:** ⬜ Pass ⬜ Fail  
**Notes:** _______________

---

### TC-PAY-F-010: Cancel Payment Before Completion

**Priority:** Medium  
**Type:** Functional  
**Preconditions:**
- Razorpay SDK open

**Test Steps:**
1. Click "Cancel" or back button in Razorpay SDK
2. Observe app behavior

**Expected Result:**
- SDK returns cancel callback
- App shows: "Payment cancelled"
- PaymentIntent status remains 'created'
- User returned to checkout screen
- Can retry payment

**Test Data:**
- User action: Cancel

**Actual Result:** _______________  
**Status:** ⬜ Pass ⬜ Fail  
**Notes:** _______________

---

## 🎨 UX/UI Tests

### TC-PAY-UI-001: Payment Processing Loading State

**Priority:** Medium  
**Type:** UX/UI  
**Preconditions:**
- Razorpay order creation in progress

**Test Steps:**
1. Click "Proceed to Pay"
2. Observe UI during API call

**Expected Result:**
- Loading spinner displayed
- "Processing..." text shown
- Button disabled (no double-click)
- Loading takes < 2 seconds

**Test Data:**
- Network: Normal speed

**Actual Result:** _______________  
**Status:** ⬜ Pass ⬜ Fail  
**Notes:** _______________

---

### TC-PAY-UI-002: Razorpay SDK Appearance

**Priority:** Medium  
**Type:** UX/UI  
**Preconditions:**
- Payment order created

**Test Steps:**
1. Open Razorpay SDK
2. Verify branding and theme

**Expected Result:**
- Theme color: #FF6B35 (Chefooz brand)
- Title: "Chefooz"
- Description: "Food Order Payment"
- User email and phone prefilled
- Payment methods displayed: UPI, Cards, Wallets, Net Banking

**Test Data:**
- Theme color: #FF6B35

**Actual Result:** _______________  
**Status:** ⬜ Pass ⬜ Fail  
**Notes:** _______________

---

### TC-PAY-UI-003: Payment Success Screen

**Priority:** High  
**Type:** UX/UI  
**Preconditions:**
- Payment completed successfully

**Test Steps:**
1. Navigate to payment success screen
2. Verify UI elements

**Expected Result:**
- Green checkmark animation plays
- "Payment Successful!" header displayed
- Order ID shown
- "Estimated Delivery: 45 mins" displayed
- "View Order" button (primary CTA)
- "Go to Home" button (secondary CTA)

**Test Data:**
- Order ID: Actual order ID

**Actual Result:** _______________  
**Status:** ⬜ Pass ⬜ Fail  
**Notes:** _______________

---

### TC-PAY-UI-004: Payment Failure Screen

**Priority:** High  
**Type:** UX/UI  
**Preconditions:**
- Payment failed

**Test Steps:**
1. Navigate to payment failure screen
2. Verify UI elements

**Expected Result:**
- Red X animation plays
- "Payment Failed" header displayed
- Failure reason: "Insufficient balance" (or actual reason)
- "Retry Payment" button (primary CTA)
- "Back to Cart" button (secondary CTA)

**Test Data:**
- Failure reason: From API

**Actual Result:** _______________  
**Status:** ⬜ Pass ⬜ Fail  
**Notes:** _______________

---

### TC-PAY-UI-005: Polling Status Indicator

**Priority:** Low  
**Type:** UX/UI  
**Preconditions:**
- Payment status polling active

**Test Steps:**
1. Observe payment processing screen during polling
2. Verify loading indicator

**Expected Result:**
- Subtle loading indicator (spinner or pulse)
- "Confirming payment..." text
- No full-screen overlay (non-intrusive)

**Test Data:**
- Polling interval: 5 seconds

**Actual Result:** _______________  
**Status:** ⬜ Pass ⬜ Fail  
**Notes:** _______________

---

## 🔍 Edge Case Tests

### TC-PAY-E-001: Minimum Payment Amount

**Priority:** Medium  
**Type:** Edge Case  
**Preconditions:**
- User has cart with very low value

**Test Steps:**
1. Add item worth ₹0.50 (50 paise)
2. Attempt to create payment order

**Expected Result:**
- API rejects with 400 Bad Request
- Message: "Minimum order value is ₹1"
- User cannot proceed to payment

**Test Data:**
- Amount: 50 paise (below minimum)

**Actual Result:** _______________  
**Status:** ⬜ Pass ⬜ Fail  
**Notes:** _______________

---

### TC-PAY-E-002: Maximum Payment Amount

**Priority:** Low  
**Type:** Edge Case  
**Preconditions:**
- User has cart with very high value

**Test Steps:**
1. Add items worth ₹100,000
2. Attempt to create payment order

**Expected Result:**
- Payment order created successfully
- No amount limit on payment module
- (Order module may have its own limits)

**Test Data:**
- Amount: ₹100,000

**Actual Result:** _______________  
**Status:** ⬜ Pass ⬜ Fail  
**Notes:** _______________

---

### TC-PAY-E-003: Payment Timeout (15 Minutes)

**Priority:** Medium  
**Type:** Edge Case  
**Preconditions:**
- Payment order created

**Test Steps:**
1. Create payment order
2. Wait 16 minutes without completing payment
3. Attempt to verify payment

**Expected Result:**
- Razorpay order expires after 15 minutes
- Payment verification may fail
- User must create new payment order
- Old payment intent remains in 'created' state

**Test Data:**
- Wait time: 16 minutes

**Actual Result:** _______________  
**Status:** ⬜ Pass ⬜ Fail  
**Notes:** _______________

---

### TC-PAY-E-004: Concurrent Payment Attempts

**Priority:** High  
**Type:** Edge Case  
**Preconditions:**
- User on checkout screen

**Test Steps:**
1. Click "Proceed to Pay" rapidly 5 times
2. Observe API calls and responses

**Expected Result:**
- Only 1 payment order created
- Subsequent clicks disabled or ignored
- No duplicate payment intents

**Test Data:**
- Clicks: 5 rapid clicks

**Actual Result:** _______________  
**Status:** ⬜ Pass ⬜ Fail  
**Notes:** _______________

---

### TC-PAY-E-005: Webhook Before SDK Callback

**Priority:** High  
**Type:** Edge Case  
**Preconditions:**
- Payment completed

**Test Steps:**
1. Complete payment
2. Simulate webhook arriving before SDK callback
3. Verify both are processed correctly

**Expected Result:**
- Webhook processes first, updates payment intent to 'paid'
- SDK callback arrives, detects already processed (idempotent)
- Order created only once
- No errors or duplicate orders

**Test Data:**
- Webhook arrives first

**Actual Result:** _______________  
**Status:** ⬜ Pass ⬜ Fail  
**Notes:** _______________

---

### TC-PAY-E-006: Invalid Payment Intent ID

**Priority:** Medium  
**Type:** Edge Case  
**Preconditions:**
- User authenticated

**Test Steps:**
1. Call `/api/v1/payment/<invalid_uuid>/status`
2. Observe response

**Expected Result:**
- API returns 400 Bad Request
- Response:
  ```json
  {
    "success": false,
    "message": "Payment intent not found",
    "errorCode": "PAYMENT_INTENT_NOT_FOUND"
  }
  ```

**Test Data:**
- Payment Intent ID: invalid-uuid-12345

**Actual Result:** _______________  
**Status:** ⬜ Pass ⬜ Fail  
**Notes:** _______________

---

### TC-PAY-E-007: User Closes App During Payment

**Priority:** High  
**Type:** Edge Case  
**Preconditions:**
- Payment in progress

**Test Steps:**
1. Open Razorpay SDK
2. Complete payment in UPI app
3. Force-close Chefooz app before SDK callback
4. Wait for webhook to arrive (1-2 minutes)
5. Reopen app
6. Check order status

**Expected Result:**
- Webhook processes payment in background
- Order created via webhook
- When user reopens app, order visible in "Active Orders"
- Push notification received: "Payment confirmed!"

**Test Data:**
- App closed before callback

**Actual Result:** _______________  
**Status:** ⬜ Pass ⬜ Fail  
**Notes:** _______________

---

## ⚠️ Error Handling Tests

### TC-PAY-ERR-001: Razorpay API Down During Order Creation

**Priority:** Critical  
**Type:** Error Handling  
**Preconditions:**
- Razorpay API unavailable (simulate with invalid credentials)

**Test Steps:**
1. Temporarily set `RAZORPAY_KEY_ID=invalid`
2. Attempt to create payment order
3. Observe error handling

**Expected Result:**
- API returns 500 Internal Server Error (or 503 Service Unavailable)
- Response:
  ```json
  {
    "success": false,
    "message": "Payment service unavailable. Please try again in a moment.",
    "errorCode": "RAZORPAY_API_ERROR"
  }
  ```
- User sees: "Payment service unavailable. Please try again."
- No payment intent created

**Test Data:**
- Razorpay API: Down

**Actual Result:** _______________  
**Status:** ⬜ Pass ⬜ Fail  
**Notes:** _______________

---

### TC-PAY-ERR-002: Invalid Signature Verification

**Priority:** Critical  
**Type:** Error Handling  
**Preconditions:**
- Payment completed

**Test Steps:**
1. Call `/api/v1/payment/verify` with tampered signature:
   ```json
   {
     "razorpay_order_id": "order_test",
     "razorpay_payment_id": "pay_test",
     "razorpay_signature": "invalid_signature_12345"
   }
   ```
2. Observe response

**Expected Result:**
- API returns 401 Unauthorized
- Response:
  ```json
  {
    "success": false,
    "message": "Invalid payment signature",
    "errorCode": "INVALID_SIGNATURE"
  }
  ```
- No order created
- Audit event logged with signature mismatch
- User sees: "Payment verification failed. Contact support."

**Test Data:**
- Signature: Tampered

**Actual Result:** _______________  
**Status:** ⬜ Pass ⬜ Fail  
**Notes:** _______________

---

### TC-PAY-ERR-003: Webhook Signature Verification Failure

**Priority:** High  
**Type:** Error Handling  
**Preconditions:**
- Webhook configured

**Test Steps:**
1. Send webhook with invalid signature
2. Observe backend logs

**Expected Result:**
- Webhook rejected with 401 Unauthorized
- Response:
  ```json
  {
    "success": false,
    "message": "Invalid webhook signature",
    "errorCode": "INVALID_WEBHOOK_SIGNATURE"
  }
  ```
- No payment intent updated
- Security alert logged

**Test Data:**
- Signature: Invalid

**Actual Result:** _______________  
**Status:** ⬜ Pass ⬜ Fail  
**Notes:** _______________

---

### TC-PAY-ERR-004: Database Connection Lost During Verification

**Priority:** Medium  
**Type:** Error Handling  
**Preconditions:**
- Payment completed

**Test Steps:**
1. Stop PostgreSQL database
2. Attempt to verify payment
3. Observe error handling

**Expected Result:**
- API returns 500 Internal Server Error
- Response:
  ```json
  {
    "success": false,
    "message": "Service temporarily unavailable",
    "errorCode": "DATABASE_ERROR"
  }
  ```
- User sees: "Service unavailable. Please try again."
- Payment can be retried after database restored

**Test Data:**
- Database: Offline

**Actual Result:** _______________  
**Status:** ⬜ Pass ⬜ Fail  
**Notes:** _______________

---

### TC-PAY-ERR-005: Order Service Failure After Payment

**Priority:** Critical  
**Type:** Error Handling  
**Preconditions:**
- Payment verified successfully
- Order service unavailable

**Test Steps:**
1. Mock OrderService.placePrepaidOrder() to throw error
2. Verify payment
3. Observe error handling

**Expected Result:**
- API returns 500 Internal Server Error
- Response:
  ```json
  {
    "success": false,
    "message": "Payment successful but order creation failed. Contact support.",
    "errorCode": "ORDER_CREATION_FAILED"
  }
  ```
- PaymentIntent marked 'paid' (payment successful)
- Order NOT created
- Manual reconciliation required
- User sees: "Payment successful but order failed. Contact support with payment ID: pay_xxxxx"

**Test Data:**
- Order service: Failing

**Actual Result:** _______________  
**Status:** ⬜ Pass ⬜ Fail  
**Notes:** _______________

---

## 🔒 Security Tests

### TC-PAY-SEC-001: Unauthorized Payment Status Access

**Priority:** Critical  
**Type:** Security  
**Preconditions:**
- Two users: User A, User B
- User A created payment intent

**Test Steps:**
1. User B attempts to access User A's payment status:
   ```bash
   GET /api/v1/payment/<userA_payment_intent_id>/status
   Authorization: Bearer <userB_token>
   ```
2. Observe response

**Expected Result:**
- API returns 400 Bad Request
- Response:
  ```json
  {
    "success": false,
    "message": "Payment intent not found",
    "errorCode": "PAYMENT_INTENT_NOT_FOUND"
  }
  ```
- No data leaked

**Test Data:**
- User A payment intent accessed by User B

**Actual Result:** _______________  
**Status:** ⬜ Pass ⬜ Fail  
**Notes:** _______________

---

### TC-PAY-SEC-002: SQL Injection in Payment Status

**Priority:** High  
**Type:** Security  
**Preconditions:**
- Attacker attempts SQL injection

**Test Steps:**
1. Call payment status with SQL injection payload:
   ```bash
   GET /api/v1/payment/'; DROP TABLE payment_intents; --/status
   ```
2. Observe response and database integrity

**Expected Result:**
- API returns 400 Bad Request (invalid UUID format)
- No SQL executed
- Database tables intact
- Security alert logged

**Test Data:**
- Payload: SQL injection

**Actual Result:** _______________  
**Status:** ⬜ Pass ⬜ Fail  
**Notes:** _______________

---

### TC-PAY-SEC-003: Rate Limiting on Payment Creation

**Priority:** High  
**Type:** Security  
**Preconditions:**
- User authenticated

**Test Steps:**
1. Call `/api/v1/payment/create-order` 25 times in 1 minute
2. Observe rate limiting

**Expected Result:**
- First 20 requests succeed
- Requests 21-25 return 429 Too Many Requests
- Response:
  ```json
  {
    "statusCode": 429,
    "message": "Too Many Requests",
    "error": "ThrottlerException"
  }
  ```
- User must wait 1 minute before retrying

**Test Data:**
- Requests: 25 in 1 minute

**Actual Result:** _______________  
**Status:** ⬜ Pass ⬜ Fail  
**Notes:** _______________

---

### TC-PAY-SEC-004: IDOR - Access Other User's Payment Intent

**Priority:** Critical  
**Type:** Security  
**Preconditions:**
- Two users with payment intents

**Test Steps:**
1. User A calls:
   ```bash
   POST /api/v1/payment/verify
   Authorization: Bearer <userA_token>
   Body: {
     "razorpay_order_id": "<userB_order_id>",
     ...
   }
   ```
2. Observe response

**Expected Result:**
- API returns 400 Bad Request
- Response: "Payment intent not found"
- No order created for User B's payment
- Security audit logged

**Test Data:**
- User A tries to verify User B's payment

**Actual Result:** _______________  
**Status:** ⬜ Pass ⬜ Fail  
**Notes:** _______________

---

### TC-PAY-SEC-005: JWT Token Expiry During Payment

**Priority:** Medium  
**Type:** Security  
**Preconditions:**
- JWT token about to expire

**Test Steps:**
1. Start payment with token expiring in 10 seconds
2. Complete payment after token expires
3. Attempt to verify payment

**Expected Result:**
- Verification fails with 401 Unauthorized
- User must re-authenticate
- Payment can be verified after login (via webhook or retry)

**Test Data:**
- Token: Expired

**Actual Result:** _______________  
**Status:** ⬜ Pass ⬜ Fail  
**Notes:** _______________

---

### TC-PAY-SEC-006: PII Leakage in Logs

**Priority:** High  
**Type:** Security  
**Preconditions:**
- Payment created with user data

**Test Steps:**
1. Create payment order
2. Check application logs

**Expected Result:**
- Logs contain payment intent ID, order ID
- Logs do NOT contain:
  - Razorpay secret key
  - Payment card details (handled by Razorpay)
  - User's full address (only city/state logged)
  - Payment signatures

**Test Data:**
- Logs: Application logs

**Actual Result:** _______________  
**Status:** ⬜ Pass ⬜ Fail  
**Notes:** _______________

---

### TC-PAY-SEC-007: Webhook Replay Attack

**Priority:** High  
**Type:** Security  
**Preconditions:**
- Webhook already processed

**Test Steps:**
1. Capture a valid webhook payload
2. Replay the same webhook 5 minutes later
3. Observe backend behavior

**Expected Result:**
- Webhook accepted (signature valid)
- Idempotency check detects duplicate
- PaymentIntent not updated again
- No duplicate order created
- Response: 200 OK (but no action taken)

**Test Data:**
- Webhook: Replayed

**Actual Result:** _______________  
**Status:** ⬜ Pass ⬜ Fail  
**Notes:** _______________

---

## ⚡ Performance Tests

### TC-PAY-PERF-001: Payment Order Creation Time

**Priority:** High  
**Type:** Performance  
**Preconditions:**
- Normal network conditions

**Test Steps:**
1. Click "Proceed to Pay"
2. Measure time until Razorpay SDK opens

**Expected Result:**
- Time: < 2 seconds
- API response time: < 1 second
- Total: < 2 seconds (including SDK initialization)

**Test Data:**
- Iterations: 10 tests
- Average time: _________

**Actual Result:** _______________  
**Status:** ⬜ Pass ⬜ Fail  
**Notes:** _______________

---

### TC-PAY-PERF-002: Payment Verification Time

**Priority:** High  
**Type:** Performance  
**Preconditions:**
- Payment completed

**Test Steps:**
1. SDK returns payment details
2. Measure time until order created

**Expected Result:**
- Time: < 3 seconds
- API response time: < 2 seconds
- Includes: signature verification + order creation

**Test Data:**
- Iterations: 10 tests
- Average time: _________

**Actual Result:** _______________  
**Status:** ⬜ Pass ⬜ Fail  
**Notes:** _______________

---

### TC-PAY-PERF-003: Webhook Processing Time

**Priority:** Medium  
**Type:** Performance  
**Preconditions:**
- Webhook event sent

**Test Steps:**
1. Send webhook to backend
2. Measure processing time

**Expected Result:**
- Time: < 500ms
- Includes: signature verification + database update
- Razorpay expects response within 5 seconds

**Test Data:**
- Iterations: 10 webhooks
- Average time: _________

**Actual Result:** _______________  
**Status:** ⬜ Pass ⬜ Fail  
**Notes:** _______________

---

### TC-PAY-PERF-004: Concurrent Payment Creations

**Priority:** Medium  
**Type:** Performance  
**Preconditions:**
- 10 users attempting payment simultaneously

**Test Steps:**
1. Simulate 10 concurrent payment order creations
2. Observe API performance

**Expected Result:**
- All 10 requests succeed
- No rate limiting triggered (10 < 20/min limit)
- Response time: < 2 seconds for all
- No database deadlocks

**Test Data:**
- Concurrent users: 10

**Actual Result:** _______________  
**Status:** ⬜ Pass ⬜ Fail  
**Notes:** _______________

---

### TC-PAY-PERF-005: Payment Status Polling Efficiency

**Priority:** Medium  
**Type:** Performance  
**Preconditions:**
- Payment status polling active

**Test Steps:**
1. Monitor network traffic during polling
2. Measure data transferred and API calls

**Expected Result:**
- API called every 5 seconds (not every second)
- Data transferred: ~500 bytes per request
- Polling stops after 5 minutes (timeout)
- No memory leaks in polling logic

**Test Data:**
- Polling duration: 5 minutes
- Total API calls: ~60 calls

**Actual Result:** _______________  
**Status:** ⬜ Pass ⬜ Fail  
**Notes:** _______________

---

## 🔄 Regression Tests

### TC-PAY-REG-001: Order Creation After Payment

**Priority:** Critical  
**Type:** Regression  
**Preconditions:**
- Payment verified successfully

**Test Steps:**
1. Complete payment flow
2. Verify order created with correct data

**Expected Result:**
- Order status: PAID
- Order amount matches payment amount
- Order items match cart snapshot
- Order address matches payment intent metadata
- Commission calculation triggered

**Test Data:**
- Payment: ₹700
- Order: Same amount

**Actual Result:** _______________  
**Status:** ⬜ Pass ⬜ Fail  
**Notes:** _______________

---

### TC-PAY-REG-002: Notification After Payment

**Priority:** High  
**Type:** Regression  
**Preconditions:**
- Payment verified successfully

**Test Steps:**
1. Complete payment
2. Check notifications sent

**Expected Result:**
- Customer notification: "Payment successful! Order placed."
- Chef notification: "New order received: Order #12345"
- Push notification payload contains orderId

**Test Data:**
- Notifications: 2 sent

**Actual Result:** _______________  
**Status:** ⬜ Pass ⬜ Fail  
**Notes:** _______________

---

### TC-PAY-REG-003: Cart Cleared After Order

**Priority:** Medium  
**Type:** Regression  
**Preconditions:**
- Payment verified, order created

**Test Steps:**
1. Complete payment
2. Check cart state

**Expected Result:**
- Cart cleared (0 items)
- User redirected to order tracking
- Cannot reorder same cart (already processed)

**Test Data:**
- Cart: Cleared

**Actual Result:** _______________  
**Status:** ⬜ Pass ⬜ Fail  
**Notes:** _______________

---

### TC-PAY-REG-004: Razorpay Dashboard Reconciliation

**Priority:** High  
**Type:** Regression  
**Preconditions:**
- 5 payments completed

**Test Steps:**
1. Check Razorpay dashboard
2. Compare with database payment_intents table

**Expected Result:**
- All payments visible in Razorpay dashboard
- Status matches database status
- Amount matches exactly
- Order IDs match

**Test Data:**
- Payments: 5 orders

**Actual Result:** _______________  
**Status:** ⬜ Pass ⬜ Fail  
**Notes:** _______________

---

## 📱 Platform-Specific Tests

### TC-PAY-PLAT-001: UPI Intent on Android

**Priority:** Critical  
**Type:** Platform-Specific  
**Preconditions:**
- Android device
- Google Pay/PhonePe installed

**Test Steps:**
1. Select UPI payment
2. Choose Google Pay
3. Complete payment

**Expected Result:**
- Google Pay opens via Intent
- Payment completes successfully
- App returns to Chefooz after payment
- Order created

**Test Data:**
- Platform: Android
- UPI App: Google Pay

**Actual Result:** _______________  
**Status:** ⬜ Pass ⬜ Fail  
**Notes:** _______________

---

### TC-PAY-PLAT-002: UPI Intent on iOS

**Priority:** Critical  
**Type:** Platform-Specific  
**Preconditions:**
- iOS device
- PhonePe/Google Pay installed

**Test Steps:**
1. Select UPI payment
2. Choose PhonePe
3. Complete payment

**Expected Result:**
- PhonePe opens via Universal Link
- Payment completes successfully
- App returns to Chefooz after payment
- Order created

**Test Data:**
- Platform: iOS
- UPI App: PhonePe

**Actual Result:** _______________  
**Status:** ⬜ Pass ⬜ Fail  
**Notes:** _______________

---

### TC-PAY-PLAT-003: Push Notification on iOS

**Priority:** High  
**Type:** Platform-Specific  
**Preconditions:**
- iOS device
- Push notifications enabled

**Test Steps:**
1. Complete payment
2. Close app
3. Wait for webhook to process
4. Observe notification

**Expected Result:**
- Push notification received
- Notification title: "Payment Confirmed!"
- Notification body: "Your order #12345 is confirmed."
- Tapping notification opens order tracking screen

**Test Data:**
- Platform: iOS

**Actual Result:** _______________  
**Status:** ⬜ Pass ⬜ Fail  
**Notes:** _______________

---

### TC-PAY-PLAT-004: Deep Link to Order After Payment

**Priority:** Medium  
**Type:** Platform-Specific  
**Preconditions:**
- Payment completed

**Test Steps:**
1. Receive push notification
2. Tap notification
3. App opens to specific screen

**Expected Result:**
- App opens (or comes to foreground)
- Navigates directly to order tracking screen
- Shows order details immediately

**Test Data:**
- Deep link: chefooz://order/<orderId>

**Actual Result:** _______________  
**Status:** ⬜ Pass ⬜ Fail  
**Notes:** _______________

---

## 📊 Test Execution Summary

### Coverage Matrix

| Category | Test Cases | Pass | Fail | Blocked | Pass Rate |
|----------|-----------|------|------|---------|-----------|
| Functional | 10 | ___ | ___ | ___ | ___% |
| UX/UI | 5 | ___ | ___ | ___ | ___% |
| Edge Cases | 7 | ___ | ___ | ___ | ___% |
| Error Handling | 5 | ___ | ___ | ___ | ___% |
| Security | 7 | ___ | ___ | ___ | ___% |
| Performance | 5 | ___ | ___ | ___ | ___% |
| Regression | 4 | ___ | ___ | ___ | ___% |
| Platform-Specific | 4 | ___ | ___ | ___ | ___% |
| **TOTAL** | **47** | ___ | ___ | ___ | ___% |

---

### Pre-Deployment Checklist

**Environment:**
- [ ] Razorpay production keys configured
- [ ] Webhook URL updated to production
- [ ] Database migrations applied
- [ ] Environment variables verified

**Functional:**
- [ ] All critical test cases passed
- [ ] Payment flow tested end-to-end
- [ ] Webhook processing verified
- [ ] Signature verification working

**Security:**
- [ ] Rate limiting enabled
- [ ] JWT authentication enforced
- [ ] PII protection verified
- [ ] Audit logging enabled

**Performance:**
- [ ] Payment creation < 2 seconds
- [ ] Webhook processing < 500ms
- [ ] No memory leaks in polling

**Integration:**
- [ ] Order creation after payment tested
- [ ] Notification dispatch verified
- [ ] Cart clearing confirmed

**Monitoring:**
- [ ] Error tracking configured (Sentry)
- [ ] Payment metrics dashboard setup
- [ ] Razorpay dashboard access verified

---

### Sign-Off

**QA Lead:** _______________  
**Date:** _______________  
**Approval:** ⬜ Approved ⬜ Rejected  
**Comments:** _______________

---

**[PAYMENT_QA_TEST_CASES_COMPLETE ✅]**
