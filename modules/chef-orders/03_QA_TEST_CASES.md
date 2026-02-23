# 🧪 Chef Orders Module - QA Test Cases

## 📋 **Table of Contents**
- [Test Environment Setup](#test-environment-setup)
- [Test Data Preparation](#test-data-preparation)
- [Functional Test Cases](#functional-test-cases)
- [API Test Cases](#api-test-cases)
- [Integration Test Cases](#integration-test-cases)
- [Error Handling Test Cases](#error-handling-test-cases)
- [Performance Test Cases](#performance-test-cases)
- [Security Test Cases](#security-test-cases)
- [Test Automation Scripts](#test-automation-scripts)
- [Test Checklist](#test-checklist)

---

## 🛠️ **Test Environment Setup**

### **Prerequisites**
- Backend API running: `http://https://api-staging.chefooz.com
- PostgreSQL database with test data
- Valid JWT tokens for chef and customer users
- PowerShell 7+ installed (for automation scripts)

### **Test User Accounts**

**Chef Account**:
```json
{
  "userId": "chef-test-user-001",
  "email": "chef@test.chefooz.com",
  "role": "CHEF",
  "kitchenId": "kitchen-test-001",
  "jwtToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." // Required
}
```

**Customer Account**:
```json
{
  "userId": "customer-test-user-001",
  "email": "customer@test.chefooz.com",
  "role": "USER"
}
```

### **Test Configuration**

**PowerShell Variables**:
```powershell
# Test environment configuration
$BASE_URL = "https://api-staging.chefooz.com"
$API_VERSION = "v1"
$CHEF_TOKEN = "Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
$CUSTOMER_TOKEN = "Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."

# Test headers
$CHEF_HEADERS = @{
    "Authorization" = $CHEF_TOKEN
    "Content-Type" = "application/json"
}

$CUSTOMER_HEADERS = @{
    "Authorization" = $CUSTOMER_TOKEN
    "Content-Type" = "application/json"
}
```

---

## 📊 **Test Data Preparation**

### **Sample Order Data**

**Test Order 1 (NEW Status)**:
```json
{
  "id": "order-test-001",
  "orderNumber": "ORD-TEST001",
  "chefId": "chef-test-user-001",
  "userId": "customer-test-user-001",
  "status": "PAID",
  "chefStatus": "NEW",
  "paymentStatus": "PAID",
  "totalPaise": 55000,
  "items": [
    {
      "menuItemId": "menu-item-001",
      "quantity": 2,
      "unitPricePaise": 25000
    }
  ],
  "addressSnapshot": {
    "fullName": "Test Customer",
    "phone": "+919876543210",
    "line1": "123 Test Street",
    "city": "Mumbai",
    "state": "Maharashtra",
    "pincode": "400001"
  }
}
```

**Test Order 2 (ACCEPTED Status)**:
```json
{
  "id": "order-test-002",
  "chefStatus": "ACCEPTED",
  "estimatedPrepMinutes": 30,
  "acceptedAt": "2026-02-22T10:00:00.000Z"
}
```

**Test Order 3 (PREPARING Status)**:
```json
{
  "id": "order-test-003",
  "chefStatus": "PREPARING",
  "estimatedPrepMinutes": 45,
  "acceptedAt": "2026-02-22T09:30:00.000Z"
}
```

### **Kitchen State Setup**

**Kitchen Online & Accepting**:
```sql
UPDATE chef_kitchens
SET is_online = true, accepting_orders = true
WHERE id = 'kitchen-test-001';
```

**Kitchen Offline**:
```sql
UPDATE chef_kitchens
SET is_online = false
WHERE id = 'kitchen-test-001';
```

**Kitchen Paused**:
```sql
UPDATE chef_kitchens
SET is_online = true, accepting_orders = false
WHERE id = 'kitchen-test-001';
```

---

## ✅ **Functional Test Cases**

### **Category 1: View Incoming Orders**

#### **Test Case 1.1: Get All Incoming Orders**

**Test ID**: `CHEF_ORD_001`  
**Priority**: Critical  
**Category**: Dashboard

**Preconditions**:
- Chef is logged in
- At least 3 orders exist (NEW, ACCEPTED, PREPARING)

**Test Steps**:
1. Send GET request to `/v1/chef/orders/incoming`
2. Include JWT token in Authorization header
3. Verify response status 200
4. Verify response contains order array
5. Verify computed fields present (statusGroup, prepRemainingMinutes, isOverdue)

**Expected Result**:
```json
{
  "success": true,
  "message": "Found X incoming orders",
  "data": [
    {
      "id": "order-test-001",
      "orderNumber": "ORD-TEST001",
      "customerName": "Test Customer",
      "items": [...],
      "totalAmount": 550.00,
      "chefStatus": "NEW",
      "statusGroup": "NEW",
      "prepRemainingMinutes": null,
      "isOverdue": false
    }
  ]
}
```

**PowerShell Script**:
```powershell
$response = Invoke-WebRequest `
    -Uri "$BASE_URL/api/$API_VERSION/chef/orders/incoming" `
    -Method GET `
    -Headers $CHEF_HEADERS

Write-Host "Status: $($response.StatusCode)"
Write-Host "Orders found: $(($response.Content | ConvertFrom-Json).data.Count)"
```

---

#### **Test Case 1.2: Verify Order Grouping**

**Test ID**: `CHEF_ORD_002`  
**Priority**: High  
**Category**: Dashboard

**Preconditions**:
- Orders exist in different statuses

**Test Steps**:
1. Get incoming orders
2. Verify statusGroup field for each order:
   - NEW status → statusGroup = "NEW"
   - ACCEPTED status → statusGroup = "ACTIVE"
   - PREPARING status → statusGroup = "ACTIVE"
   - READY status → statusGroup = "READY"

**Expected Result**:
- All orders have correct statusGroup
- UI can filter by statusGroup

**Validation**:
```powershell
$orders = ($response.Content | ConvertFrom-Json).data
foreach ($order in $orders) {
    Write-Host "Order: $($order.orderNumber), Status: $($order.chefStatus), Group: $($order.statusGroup)"
}
```

---

#### **Test Case 1.3: Verify Prep Time Countdown**

**Test ID**: `CHEF_ORD_003`  
**Priority**: High  
**Category**: Dashboard

**Preconditions**:
- At least 1 ACCEPTED order exists with prep time

**Test Steps**:
1. Get incoming orders
2. Find order with ACCEPTED status
3. Verify prepRemainingMinutes is calculated
4. Wait 1 minute and re-fetch
5. Verify prepRemainingMinutes decreased by 1

**Expected Result**:
- prepRemainingMinutes = estimatedPrepMinutes - elapsedMinutes
- Can be negative if overdue
- isOverdue flag set correctly

**Validation**:
```powershell
# First call
$order1 = (Invoke-WebRequest -Uri "$BASE_URL/api/$API_VERSION/chef/orders/incoming" -Method GET -Headers $CHEF_HEADERS | ConvertFrom-Json).data[0]
$remaining1 = $order1.prepRemainingMinutes

Start-Sleep -Seconds 60

# Second call
$order2 = (Invoke-WebRequest -Uri "$BASE_URL/api/$API_VERSION/chef/orders/incoming" -Method GET -Headers $CHEF_HEADERS | ConvertFrom-Json).data[0]
$remaining2 = $order2.prepRemainingMinutes

Write-Host "Remaining time decreased: $($remaining1 - $remaining2) minutes"
```

---

### **Category 2: Accept Orders**

#### **Test Case 2.1: Accept Order Successfully**

**Test ID**: `CHEF_ORD_004`  
**Priority**: Critical  
**Category**: Order Acceptance

**Preconditions**:
- Kitchen is online and accepting orders
- Order status = NEW

**Test Steps**:
1. Send POST request to `/v1/chef/orders/{orderId}/accept`
2. Include valid prep time (30 minutes)
3. Include optional chef note
4. Verify response status 200
5. Verify order status updated to ACCEPTED
6. Verify acceptedAt timestamp set

**Request Body**:
```json
{
  "estimatedPrepMinutes": 30,
  "chefNote": "Will be ready on time"
}
```

**Expected Result**:
```json
{
  "success": true,
  "message": "Order accepted. Will be ready in 30 minutes",
  "data": {
    "id": "order-test-001",
    "status": "ACCEPTED",
    "chefStatus": "ACCEPTED",
    "estimatedPrepMinutes": 30,
    "chefNote": "Will be ready on time",
    "acceptedAt": "2026-02-22T10:35:00.000Z"
  }
}
```

**PowerShell Script**:
```powershell
$body = @{
    estimatedPrepMinutes = 30
    chefNote = "Will be ready on time"
} | ConvertTo-Json

$response = Invoke-WebRequest `
    -Uri "$BASE_URL/api/$API_VERSION/chef/orders/order-test-001/accept" `
    -Method POST `
    -Headers $CHEF_HEADERS `
    -Body $body

Write-Host "Status: $($response.StatusCode)"
Write-Host "Order accepted at: $(($response.Content | ConvertFrom-Json).data.acceptedAt)"
```

---

#### **Test Case 2.2: Accept with Minimum Prep Time (5 minutes)**

**Test ID**: `CHEF_ORD_005`  
**Priority**: High  
**Category**: Order Acceptance

**Test Steps**:
1. Accept order with prep time = 5 minutes
2. Verify acceptance successful

**Request Body**:
```json
{
  "estimatedPrepMinutes": 5
}
```

**Expected Result**:
- Acceptance successful
- No validation error

---

#### **Test Case 2.3: Accept with Maximum Prep Time (180 minutes)**

**Test ID**: `CHEF_ORD_006`  
**Priority**: High  
**Category**: Order Acceptance

**Test Steps**:
1. Accept order with prep time = 180 minutes
2. Verify acceptance successful

**Request Body**:
```json
{
  "estimatedPrepMinutes": 180
}
```

**Expected Result**:
- Acceptance successful
- No validation error

---

#### **Test Case 2.4: Reject - Prep Time Below Minimum**

**Test ID**: `CHEF_ORD_007`  
**Priority**: High  
**Category**: Order Acceptance Validation

**Test Steps**:
1. Attempt to accept order with prep time = 4 minutes
2. Verify response status 400
3. Verify error message about minimum prep time

**Request Body**:
```json
{
  "estimatedPrepMinutes": 4
}
```

**Expected Result**:
```json
{
  "success": false,
  "message": "Validation failed",
  "errors": [
    "estimatedPrepMinutes must not be less than 5"
  ]
}
```

---

#### **Test Case 2.5: Reject - Prep Time Above Maximum**

**Test ID**: `CHEF_ORD_008`  
**Priority**: High  
**Category**: Order Acceptance Validation

**Test Steps**:
1. Attempt to accept order with prep time = 181 minutes
2. Verify response status 400
3. Verify error message about maximum prep time

**Expected Result**:
```json
{
  "success": false,
  "message": "Validation failed",
  "errors": [
    "estimatedPrepMinutes must not be greater than 180"
  ]
}
```

---

#### **Test Case 2.6: Reject - Kitchen Offline**

**Test ID**: `CHEF_ORD_009`  
**Priority**: Critical  
**Category**: Kitchen Availability

**Preconditions**:
- Kitchen status set to offline

**Test Steps**:
1. Set kitchen isOnline = false
2. Attempt to accept order
3. Verify response status 400
4. Verify error message about kitchen offline

**Expected Result**:
```json
{
  "success": false,
  "message": "Cannot accept orders while offline. Please set your kitchen status to 'Online' first.",
  "errorCode": "ORDER_ACCEPT_FAILED"
}
```

**PowerShell Script**:
```powershell
# Set kitchen offline
Invoke-SqlCmd -Query "UPDATE chef_kitchens SET is_online = false WHERE id = 'kitchen-test-001'"

# Attempt to accept order
try {
    Invoke-WebRequest `
        -Uri "$BASE_URL/api/$API_VERSION/chef/orders/order-test-001/accept" `
        -Method POST `
        -Headers $CHEF_HEADERS `
        -Body '{"estimatedPrepMinutes": 30}'
} catch {
    Write-Host "Expected error: $($_.Exception.Response.StatusCode)"
}

# Reset kitchen to online
Invoke-SqlCmd -Query "UPDATE chef_kitchens SET is_online = true WHERE id = 'kitchen-test-001'"
```

---

#### **Test Case 2.7: Reject - Kitchen Paused**

**Test ID**: `CHEF_ORD_010`  
**Priority**: Critical  
**Category**: Kitchen Availability

**Preconditions**:
- Kitchen isOnline = true, acceptingOrders = false

**Test Steps**:
1. Set kitchen acceptingOrders = false
2. Attempt to accept order
3. Verify response status 400
4. Verify error message about kitchen paused

**Expected Result**:
```json
{
  "success": false,
  "message": "Cannot accept orders at this time. Your kitchen is currently paused. Please enable 'Accepting Orders' in your kitchen settings.",
  "errorCode": "ORDER_ACCEPT_FAILED"
}
```

---

#### **Test Case 2.8: Reject - Order Already Accepted**

**Test ID**: `CHEF_ORD_011`  
**Priority**: High  
**Category**: Status Validation

**Preconditions**:
- Order status = ACCEPTED

**Test Steps**:
1. Attempt to accept already-accepted order
2. Verify response status 400
3. Verify error message about invalid status

**Expected Result**:
```json
{
  "success": false,
  "message": "Cannot accept order. Order must be PAID and chef status must be NEW. Current order status: ACCEPTED, chef status: ACCEPTED",
  "errorCode": "ORDER_ACCEPT_FAILED"
}
```

---

#### **Test Case 2.9: Reject - Unauthorized Chef**

**Test ID**: `CHEF_ORD_012`  
**Priority**: Critical  
**Category**: Security

**Preconditions**:
- Order belongs to different chef

**Test Steps**:
1. Use JWT token of different chef
2. Attempt to accept order
3. Verify response status 403
4. Verify error message about unauthorized

**Expected Result**:
```json
{
  "success": false,
  "message": "You are not authorized to manage this order",
  "errorCode": "ORDER_ACCEPT_FAILED"
}
```

---

### **Category 3: Reject Orders**

#### **Test Case 3.1: Reject Order Successfully**

**Test ID**: `CHEF_ORD_013`  
**Priority**: Critical  
**Category**: Order Rejection

**Preconditions**:
- Order status = NEW

**Test Steps**:
1. Send POST request to `/v1/chef/orders/{orderId}/reject`
2. Include rejection reason
3. Verify response status 200
4. Verify order status updated to CANCELLED/REJECTED
5. Verify rejectedAt timestamp set
6. Verify refund initiated

**Request Body**:
```json
{
  "reason": "Out of main ingredient"
}
```

**Expected Result**:
```json
{
  "success": true,
  "message": "Order rejected. Refund will be processed.",
  "data": {
    "id": "order-test-001",
    "status": "CANCELLED",
    "chefStatus": "REJECTED",
    "chefNote": "Out of main ingredient",
    "rejectedAt": "2026-02-22T10:36:00.000Z"
  }
}
```

**PowerShell Script**:
```powershell
$body = @{
    reason = "Out of main ingredient"
} | ConvertTo-Json

$response = Invoke-WebRequest `
    -Uri "$BASE_URL/api/$API_VERSION/chef/orders/order-test-001/reject" `
    -Method POST `
    -Headers $CHEF_HEADERS `
    -Body $body

Write-Host "Status: $($response.StatusCode)"
Write-Host "Order rejected at: $(($response.Content | ConvertFrom-Json).data.rejectedAt)"
```

---

#### **Test Case 3.2: Verify Automatic Refund**

**Test ID**: `CHEF_ORD_014`  
**Priority**: Critical  
**Category**: Integration

**Preconditions**:
- Order has valid payment record

**Test Steps**:
1. Reject order
2. Query payment records
3. Verify refund initiated
4. Verify payment status = REFUNDED

**Validation Query**:
```sql
SELECT 
    o.id, 
    o.status, 
    p.status as payment_status,
    p.refund_initiated_at,
    p.refund_amount
FROM orders o
JOIN payments p ON o.id = p.order_id
WHERE o.id = 'order-test-001';
```

**Expected Result**:
- payment_status = 'REFUNDED'
- refund_amount = original payment amount
- refund_initiated_at IS NOT NULL

---

#### **Test Case 3.3: Reject - Missing Reason**

**Test ID**: `CHEF_ORD_015`  
**Priority**: High  
**Category**: Validation

**Test Steps**:
1. Attempt to reject order without reason
2. Verify response status 400
3. Verify error message about required field

**Request Body**:
```json
{
  "reason": ""
}
```

**Expected Result**:
```json
{
  "success": false,
  "message": "Validation failed",
  "errors": [
    "reason should not be empty"
  ]
}
```

---

#### **Test Case 3.4: Reject - Already Accepted Order**

**Test ID**: `CHEF_ORD_016`  
**Priority**: High  
**Category**: Status Validation

**Preconditions**:
- Order status = ACCEPTED

**Test Steps**:
1. Attempt to reject ACCEPTED order
2. Verify response status 400
3. Verify error message about invalid status

**Expected Result**:
```json
{
  "success": false,
  "message": "Cannot reject order in ACCEPTED status. Only NEW orders can be rejected.",
  "errorCode": "ORDER_REJECT_FAILED"
}
```

---

### **Category 4: Update Prep Status**

#### **Test Case 4.1: Mark as PREPARING**

**Test ID**: `CHEF_ORD_017`  
**Priority**: Critical  
**Category**: Status Update

**Preconditions**:
- Order status = ACCEPTED

**Test Steps**:
1. Send POST request to `/v1/chef/orders/{orderId}/status`
2. Set status to PREPARING
3. Verify response status 200
4. Verify order status updated

**Request Body**:
```json
{
  "status": "PREPARING"
}
```

**Expected Result**:
```json
{
  "success": true,
  "message": "Order status updated to PREPARING",
  "data": {
    "id": "order-test-002",
    "status": "PREPARING",
    "chefStatus": "PREPARING"
  }
}
```

**PowerShell Script**:
```powershell
$body = @{
    status = "PREPARING"
} | ConvertTo-Json

$response = Invoke-WebRequest `
    -Uri "$BASE_URL/api/$API_VERSION/chef/orders/order-test-002/status" `
    -Method POST `
    -Headers $CHEF_HEADERS `
    -Body $body

Write-Host "Status: $($response.StatusCode)"
Write-Host "New status: $(($response.Content | ConvertFrom-Json).data.status)"
```

---

#### **Test Case 4.2: Mark as READY**

**Test ID**: `CHEF_ORD_018`  
**Priority**: Critical  
**Category**: Status Update

**Preconditions**:
- Order status = PREPARING

**Test Steps**:
1. Send POST request to `/v1/chef/orders/{orderId}/status`
2. Set status to READY
3. Verify response status 200
4. Verify order status updated
5. Verify readyAt timestamp set
6. Verify customer notified

**Request Body**:
```json
{
  "status": "READY"
}
```

**Expected Result**:
```json
{
  "success": true,
  "message": "Order status updated to READY",
  "data": {
    "id": "order-test-003",
    "status": "READY",
    "chefStatus": "READY",
    "readyAt": "2026-02-22T11:05:00.000Z"
  }
}
```

---

#### **Test Case 4.3: Reject - Skip PREPARING**

**Test ID**: `CHEF_ORD_019`  
**Priority**: High  
**Category**: Status Validation

**Preconditions**:
- Order status = ACCEPTED

**Test Steps**:
1. Attempt to mark as READY directly from ACCEPTED
2. Verify response status 400
3. Verify error message about invalid transition

**Expected Result**:
```json
{
  "success": false,
  "message": "Can only mark as READY after PREPARING status",
  "errorCode": "ORDER_STATUS_UPDATE_FAILED"
}
```

---

#### **Test Case 4.4: Reject - Backward Transition**

**Test ID**: `CHEF_ORD_020`  
**Priority**: High  
**Category**: Status Validation

**Preconditions**:
- Order status = PREPARING

**Test Steps**:
1. Attempt to mark as ACCEPTED (backward)
2. Verify response status 400
3. Verify error message about invalid transition

**Expected Result**:
- Error response
- Status unchanged

---

### **Category 5: Update Prep Time**

#### **Test Case 5.1: Update Prep Time (Small Change)**

**Test ID**: `CHEF_ORD_021`  
**Priority**: High  
**Category**: Prep Time Update

**Preconditions**:
- Order status = ACCEPTED
- Current prep time = 30 minutes

**Test Steps**:
1. Send POST request to `/v1/chef/orders/{orderId}/prep-time`
2. Set new prep time = 35 minutes (5 min increase)
3. Verify response status 200
4. Verify prep time updated
5. Verify customer NOT notified (change < 10 min)

**Request Body**:
```json
{
  "estimatedPrepMinutes": 35
}
```

**Expected Result**:
```json
{
  "success": true,
  "message": "Prep time updated to 35 minutes",
  "data": {
    "id": "order-test-002",
    "estimatedPrepMinutes": 35
  }
}
```

**Notification Check**:
```powershell
# Query notification logs (should be empty for <10 min change)
# Verify no 'order.prep_time_updated' event sent
```

---

#### **Test Case 5.2: Update Prep Time (Large Change)**

**Test ID**: `CHEF_ORD_022`  
**Priority**: Critical  
**Category**: Prep Time Update

**Preconditions**:
- Order status = ACCEPTED
- Current prep time = 30 minutes

**Test Steps**:
1. Update prep time to 45 minutes (15 min increase)
2. Verify response status 200
3. Verify prep time updated
4. Verify customer notified (change ≥ 10 min)

**Request Body**:
```json
{
  "estimatedPrepMinutes": 45
}
```

**Expected Result**:
- Prep time updated
- Customer receives push notification
- Notification includes old and new prep times

**Notification Verification**:
```powershell
# Check notification logs
# Event: 'order.prep_time_updated'
# Payload: { oldPrepTime: 30, newPrepTime: 45 }
```

---

#### **Test Case 5.3: Update Prep Time (Decrease)**

**Test ID**: `CHEF_ORD_023`  
**Priority**: High  
**Category**: Prep Time Update

**Preconditions**:
- Current prep time = 45 minutes

**Test Steps**:
1. Update prep time to 30 minutes (15 min decrease)
2. Verify notification sent (good news for customer!)

**Expected Result**:
- Prep time updated
- Customer notified (faster preparation)

---

#### **Test Case 5.4: Reject - Invalid Prep Time Range**

**Test ID**: `CHEF_ORD_024`  
**Priority**: High  
**Category**: Validation

**Test Steps**:
1. Attempt to update prep time to 200 minutes (above max)
2. Verify response status 400

**Expected Result**:
```json
{
  "success": false,
  "message": "Estimated prep time must be between 5 and 180 minutes",
  "errorCode": "PREP_TIME_UPDATE_FAILED"
}
```

---

#### **Test Case 5.5: Reject - Update for Completed Order**

**Test ID**: `CHEF_ORD_025`  
**Priority**: High  
**Category**: Status Validation

**Preconditions**:
- Order status = READY

**Test Steps**:
1. Attempt to update prep time for READY order
2. Verify response status 400
3. Verify error message about invalid order state

**Expected Result**:
```json
{
  "success": false,
  "message": "Can only update prep time for accepted or preparing orders",
  "errorCode": "PREP_TIME_UPDATE_FAILED"
}
```

---

## 🔌 **Integration Test Cases**

### **Category 6: End-to-End Flows**

#### **Test Case 6.1: Full Order Acceptance Flow**

**Test ID**: `CHEF_ORD_026`  
**Priority**: Critical  
**Category**: E2E

**Test Steps**:
1. Create NEW order (via Order module)
2. Chef views incoming orders
3. Chef accepts order with 30 min prep
4. Verify customer receives notification
5. Chef marks as PREPARING
6. Chef marks as READY
7. Verify customer receives "ready" notification
8. Verify delivery system notified

**PowerShell E2E Script**:
```powershell
# Step 1: Create order (Order module API)
$newOrder = Invoke-WebRequest -Uri "$BASE_URL/api/$API_VERSION/orders" -Method POST -Headers $CUSTOMER_HEADERS -Body $orderData

# Step 2: Chef views incoming
$incoming = Invoke-WebRequest -Uri "$BASE_URL/api/$API_VERSION/chef/orders/incoming" -Method GET -Headers $CHEF_HEADERS
$orderId = (($incoming.Content | ConvertFrom-Json).data[0]).id

# Step 3: Accept order
$acceptBody = '{"estimatedPrepMinutes": 30}'
$accepted = Invoke-WebRequest -Uri "$BASE_URL/api/$API_VERSION/chef/orders/$orderId/accept" -Method POST -Headers $CHEF_HEADERS -Body $acceptBody

# Step 4: Mark as PREPARING
$preparingBody = '{"status": "PREPARING"}'
Invoke-WebRequest -Uri "$BASE_URL/api/$API_VERSION/chef/orders/$orderId/status" -Method POST -Headers $CHEF_HEADERS -Body $preparingBody

# Step 5: Mark as READY
$readyBody = '{"status": "READY"}'
Invoke-WebRequest -Uri "$BASE_URL/api/$API_VERSION/chef/orders/$orderId/status" -Method POST -Headers $CHEF_HEADERS -Body $readyBody

Write-Host "✅ Full flow complete"
```

---

#### **Test Case 6.2: Full Order Rejection Flow**

**Test ID**: `CHEF_ORD_027`  
**Priority**: Critical  
**Category**: E2E

**Test Steps**:
1. Create NEW order with payment
2. Chef views incoming orders
3. Chef rejects order with reason
4. Verify order status = CANCELLED
5. Verify refund initiated
6. Verify customer receives cancellation email
7. Verify chef receives confirmation email

**Verification**:
```powershell
# After rejection, verify:
# 1. Order status
$order = Invoke-WebRequest -Uri "$BASE_URL/api/$API_VERSION/orders/$orderId" -Method GET -Headers $CUSTOMER_HEADERS
$orderStatus = (($order.Content | ConvertFrom-Json).data).status
Write-Host "Order status: $orderStatus" # Should be CANCELLED

# 2. Payment status
# Query payment table to verify refund initiated
```

---

#### **Test Case 6.3: Prep Time Adjustment Mid-Cooking**

**Test ID**: `CHEF_ORD_028`  
**Priority**: High  
**Category**: E2E

**Test Steps**:
1. Accept order with 30 min prep
2. Wait 10 minutes (or simulate elapsed time)
3. Update prep time to 50 minutes
4. Verify customer notified of delay
5. Verify dashboard countdown recalculated

---

## 🛡️ **Security Test Cases**

### **Category 7: Authentication & Authorization**

#### **Test Case 7.1: Access Without JWT Token**

**Test ID**: `CHEF_ORD_029`  
**Priority**: Critical  
**Category**: Security

**Test Steps**:
1. Send request without Authorization header
2. Verify response status 401
3. Verify error message about missing token

**Expected Result**:
```json
{
  "statusCode": 401,
  "message": "Unauthorized"
}
```

---

#### **Test Case 7.2: Access with Invalid JWT Token**

**Test ID**: `CHEF_ORD_030`  
**Priority**: Critical  
**Category**: Security

**Test Steps**:
1. Send request with invalid/expired token
2. Verify response status 401

---

#### **Test Case 7.3: Access Other Chef's Orders**

**Test ID**: `CHEF_ORD_031`  
**Priority**: Critical  
**Category**: Authorization

**Test Steps**:
1. Chef A attempts to accept Chef B's order
2. Verify response status 403
3. Verify error message about unauthorized access

---

#### **Test Case 7.4: Customer Role Attempting Chef Actions**

**Test ID**: `CHEF_ORD_032`  
**Priority**: Critical  
**Category**: Authorization

**Test Steps**:
1. Use customer JWT token
2. Attempt to access chef endpoints
3. Verify response status 403

---

## ⚡ **Performance Test Cases**

### **Category 8: Performance & Load**

#### **Test Case 8.1: Response Time - Get Incoming Orders**

**Test ID**: `CHEF_ORD_033`  
**Priority**: High  
**Category**: Performance

**Test Steps**:
1. Measure response time for incoming orders API
2. Test with 10, 50, 100 orders in database
3. Verify p95 response time < 200ms

**PowerShell Performance Test**:
```powershell
$iterations = 10
$times = @()

for ($i = 1; $i -le $iterations; $i++) {
    $start = Get-Date
    Invoke-WebRequest -Uri "$BASE_URL/api/$API_VERSION/chef/orders/incoming" -Method GET -Headers $CHEF_HEADERS
    $end = Get-Date
    $duration = ($end - $start).TotalMilliseconds
    $times += $duration
    Write-Host "Request $i: $duration ms"
}

$avg = ($times | Measure-Object -Average).Average
$p95 = ($times | Sort-Object)[[Math]::Floor($iterations * 0.95)]
Write-Host "Average: $avg ms"
Write-Host "P95: $p95 ms"
```

**Expected Result**:
- Average < 150ms
- P95 < 200ms

---

#### **Test Case 8.2: Concurrent Order Acceptances**

**Test ID**: `CHEF_ORD_034`  
**Priority**: Medium  
**Category**: Concurrency

**Test Steps**:
1. Create 5 NEW orders
2. Attempt to accept all 5 simultaneously (parallel requests)
3. Verify all 5 accepted successfully
4. Verify no race conditions or database locks

---

#### **Test Case 8.3: Database Query Optimization**

**Test ID**: `CHEF_ORD_035`  
**Priority**: Medium  
**Category**: Performance

**Test Steps**:
1. Enable PostgreSQL query logging
2. Fetch incoming orders
3. Verify query count ≤ 2 (orders + menu items)
4. Verify no N+1 query issues

---

## 📝 **Test Automation Scripts**

### **Full Test Suite (PowerShell)**

```powershell
# ============================================
# Chef Orders Module - Full Test Suite
# ============================================

# Configuration
$BASE_URL = "https://api-staging.chefooz.com"
$API_VERSION = "v1"
$CHEF_TOKEN = "Bearer YOUR_CHEF_TOKEN_HERE"
$CHEF_HEADERS = @{
    "Authorization" = $CHEF_TOKEN
    "Content-Type" = "application/json"
}

# Test counters
$totalTests = 0
$passedTests = 0
$failedTests = 0

# Helper function: Run test
function Run-Test {
    param (
        [string]$TestName,
        [scriptblock]$TestScript
    )
    
    $script:totalTests++
    Write-Host "`n========================================" -ForegroundColor Cyan
    Write-Host "Running: $TestName" -ForegroundColor Cyan
    Write-Host "========================================" -ForegroundColor Cyan
    
    try {
        & $TestScript
        Write-Host "✅ PASSED" -ForegroundColor Green
        $script:passedTests++
    } catch {
        Write-Host "❌ FAILED: $($_.Exception.Message)" -ForegroundColor Red
        $script:failedTests++
    }
}

# ============================================
# Test 1: Get Incoming Orders
# ============================================
Run-Test "Get Incoming Orders" {
    $response = Invoke-WebRequest `
        -Uri "$BASE_URL/api/$API_VERSION/chef/orders/incoming" `
        -Method GET `
        -Headers $CHEF_HEADERS
    
    if ($response.StatusCode -ne 200) {
        throw "Expected 200, got $($response.StatusCode)"
    }
    
    $data = ($response.Content | ConvertFrom-Json).data
    Write-Host "Found $($data.Count) orders"
}

# ============================================
# Test 2: Accept Order
# ============================================
Run-Test "Accept Order" {
    # First, get an order
    $orders = (Invoke-WebRequest -Uri "$BASE_URL/api/$API_VERSION/chef/orders/incoming" -Method GET -Headers $CHEF_HEADERS | ConvertFrom-Json).data
    $newOrder = $orders | Where-Object { $_.chefStatus -eq "NEW" } | Select-Object -First 1
    
    if (-not $newOrder) {
        throw "No NEW orders available for testing"
    }
    
    $body = @{
        estimatedPrepMinutes = 30
        chefNote = "Test acceptance"
    } | ConvertTo-Json
    
    $response = Invoke-WebRequest `
        -Uri "$BASE_URL/api/$API_VERSION/chef/orders/$($newOrder.id)/accept" `
        -Method POST `
        -Headers $CHEF_HEADERS `
        -Body $body
    
    if ($response.StatusCode -ne 200) {
        throw "Expected 200, got $($response.StatusCode)"
    }
    
    $result = ($response.Content | ConvertFrom-Json).data
    if ($result.status -ne "ACCEPTED") {
        throw "Order status not updated to ACCEPTED"
    }
    
    Write-Host "Order accepted: $($newOrder.id)"
}

# ============================================
# Test 3: Reject Order
# ============================================
Run-Test "Reject Order" {
    # Get a NEW order
    $orders = (Invoke-WebRequest -Uri "$BASE_URL/api/$API_VERSION/chef/orders/incoming" -Method GET -Headers $CHEF_HEADERS | ConvertFrom-Json).data
    $newOrder = $orders | Where-Object { $_.chefStatus -eq "NEW" } | Select-Object -First 1
    
    if (-not $newOrder) {
        throw "No NEW orders available for testing"
    }
    
    $body = @{
        reason = "Test rejection - out of ingredients"
    } | ConvertTo-Json
    
    $response = Invoke-WebRequest `
        -Uri "$BASE_URL/api/$API_VERSION/chef/orders/$($newOrder.id)/reject" `
        -Method POST `
        -Headers $CHEF_HEADERS `
        -Body $body
    
    if ($response.StatusCode -ne 200) {
        throw "Expected 200, got $($response.StatusCode)"
    }
    
    $result = ($response.Content | ConvertFrom-Json).data
    if ($result.chefStatus -ne "REJECTED") {
        throw "Order status not updated to REJECTED"
    }
    
    Write-Host "Order rejected: $($newOrder.id)"
}

# ============================================
# Test 4: Update Status to PREPARING
# ============================================
Run-Test "Update Status to PREPARING" {
    # Get an ACCEPTED order
    $orders = (Invoke-WebRequest -Uri "$BASE_URL/api/$API_VERSION/chef/orders/incoming" -Method GET -Headers $CHEF_HEADERS | ConvertFrom-Json).data
    $acceptedOrder = $orders | Where-Object { $_.chefStatus -eq "ACCEPTED" } | Select-Object -First 1
    
    if (-not $acceptedOrder) {
        throw "No ACCEPTED orders available for testing"
    }
    
    $body = @{
        status = "PREPARING"
    } | ConvertTo-Json
    
    $response = Invoke-WebRequest `
        -Uri "$BASE_URL/api/$API_VERSION/chef/orders/$($acceptedOrder.id)/status" `
        -Method POST `
        -Headers $CHEF_HEADERS `
        -Body $body
    
    if ($response.StatusCode -ne 200) {
        throw "Expected 200, got $($response.StatusCode)"
    }
    
    Write-Host "Order status updated to PREPARING"
}

# ============================================
# Test 5: Update Prep Time
# ============================================
Run-Test "Update Prep Time" {
    # Get an ACCEPTED or PREPARING order
    $orders = (Invoke-WebRequest -Uri "$BASE_URL/api/$API_VERSION/chef/orders/incoming" -Method GET -Headers $CHEF_HEADERS | ConvertFrom-Json).data
    $order = $orders | Where-Object { $_.chefStatus -in @("ACCEPTED", "PREPARING") } | Select-Object -First 1
    
    if (-not $order) {
        throw "No active orders available for testing"
    }
    
    $body = @{
        estimatedPrepMinutes = 45
    } | ConvertTo-Json
    
    $response = Invoke-WebRequest `
        -Uri "$BASE_URL/api/$API_VERSION/chef/orders/$($order.id)/prep-time" `
        -Method POST `
        -Headers $CHEF_HEADERS `
        -Body $body
    
    if ($response.StatusCode -ne 200) {
        throw "Expected 200, got $($response.StatusCode)"
    }
    
    Write-Host "Prep time updated to 45 minutes"
}

# ============================================
# Test Summary
# ============================================
Write-Host "`n========================================" -ForegroundColor Cyan
Write-Host "TEST SUMMARY" -ForegroundColor Cyan
Write-Host "========================================" -ForegroundColor Cyan
Write-Host "Total Tests: $totalTests"
Write-Host "Passed: $passedTests" -ForegroundColor Green
Write-Host "Failed: $failedTests" -ForegroundColor Red
Write-Host "Success Rate: $([Math]::Round(($passedTests / $totalTests) * 100, 2))%"

if ($failedTests -eq 0) {
    Write-Host "`n✅ ALL TESTS PASSED!" -ForegroundColor Green
} else {
    Write-Host "`n⚠️  SOME TESTS FAILED" -ForegroundColor Yellow
}
```

---

## ✅ **Test Checklist**

### **Pre-QA Setup**
- [ ] Backend API running locally or on staging
- [ ] Database seeded with test data
- [ ] Valid JWT tokens for chef and customer
- [ ] PowerShell 7+ installed
- [ ] Test configuration variables set

### **Functional Tests**
- [ ] Get incoming orders (all statuses)
- [ ] Verify order grouping by statusGroup
- [ ] Verify prep time countdown
- [ ] Accept order successfully
- [ ] Accept with min prep time (5 min)
- [ ] Accept with max prep time (180 min)
- [ ] Reject order successfully
- [ ] Verify automatic refund triggered
- [ ] Update status to PREPARING
- [ ] Update status to READY
- [ ] Update prep time (small change, no notification)
- [ ] Update prep time (large change, with notification)

### **Validation Tests**
- [ ] Reject - Prep time below minimum
- [ ] Reject - Prep time above maximum
- [ ] Reject - Kitchen offline
- [ ] Reject - Kitchen paused
- [ ] Reject - Order already accepted
- [ ] Reject - Missing rejection reason
- [ ] Reject - Invalid status transition (ACCEPTED → reject)
- [ ] Reject - Skip PREPARING (ACCEPTED → READY)
- [ ] Reject - Backward transition (PREPARING → ACCEPTED)

### **Security Tests**
- [ ] Access without JWT token
- [ ] Access with invalid JWT token
- [ ] Access other chef's orders
- [ ] Customer role attempting chef actions

### **Integration Tests**
- [ ] Full order acceptance flow (E2E)
- [ ] Full order rejection flow (E2E)
- [ ] Prep time adjustment mid-cooking

### **Performance Tests**
- [ ] Response time < 200ms (p95)
- [ ] Concurrent order acceptances
- [ ] Database query optimization (≤2 queries)

### **Post-QA Verification**
- [ ] All test cases documented
- [ ] All edge cases covered
- [ ] Automation scripts tested
- [ ] Bug reports filed (if any)
- [ ] QA sign-off obtained

---

**[CHEF_ORDERS_QA_TEST_CASES_COMPLETE ✅]**

*For business overview, see `01_FEATURE_OVERVIEW.md`. For technical details, see `02_TECHNICAL_GUIDE.md`.*

---

**Document Version**: 1.0  
**Last Updated**: February 22, 2026  
**Module**: Chef-Orders (Week 7 - Chef Fulfillment)  
**Status**: ✅ Complete  
**Total Test Cases**: 35
