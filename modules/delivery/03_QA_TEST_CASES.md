# 🧪 Delivery Module - QA Test Cases

## 📋 **Table of Contents**
- [Test Environment Setup](#test-environment-setup)
- [Test Data Prerequisites](#test-data-prerequisites)
- [Category 1: Status Management](#category-1-status-management)
- [Category 2: Pending Requests](#category-2-pending-requests)
- [Category 3: Request Acceptance](#category-3-request-acceptance)
- [Category 4: Request Rejection](#category-4-request-rejection)
- [Category 5: State Updates](#category-5-state-updates)
- [Category 6: Delivery Completion](#category-6-delivery-completion)
- [Category 7: Busy Status](#category-7-busy-status)
- [Category 8: Assignment Timeout](#category-8-assignment-timeout)
- [Category 9: Pickup Timeout](#category-9-pickup-timeout)
- [Category 10: Delivery Timeout](#category-10-delivery-timeout)
- [Performance Tests](#performance-tests)
- [Edge Case Tests](#edge-case-tests)
- [Test Checklist](#test-checklist)

---

## 🔧 **Test Environment Setup**

### **API Base URL**
```powershell
$baseUrl = "https://api-dev.chefooz.com/v1"
# or for local: $baseUrl = "https://api-staging.chefooz.com/v1"
```

### **Authentication Headers**
```powershell
# Rider token (replace with actual JWT)
$riderHeaders = @{
    "Authorization" = "Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
    "Content-Type" = "application/json"
}
```

### **Test Users**

| Role | Email | User ID |
|------|-------|---------|
| **Rider 1** | rider1@test.com | `uuid-rider-001` |
| **Rider 2** | rider2@test.com | `uuid-rider-002` |
| **Chef** | chef1@test.com | `uuid-chef-001` |
| **Customer** | customer1@test.com | `uuid-customer-001` |

### **Test Data Cleanup**
```sql
-- Run before each test suite
TRUNCATE TABLE delivery_requests CASCADE;
TRUNCATE TABLE active_deliveries CASCADE;
UPDATE users SET delivery_status = 'offline', current_delivery_id = NULL WHERE role = 'rider';
UPDATE rider_profiles SET active_order_count = 0, is_busy = false;
```

---

## 📊 **Test Data Prerequisites**

### **1. Create Test Order**
```powershell
# Create a test order (status = PAID)
$orderBody = @{
    userId = "uuid-customer-001"
    chefId = "uuid-chef-001"
    items = @(
        @{ menuItemId = "menu-item-1"; quantity = 2 }
    )
    totalPaise = 55000
    deliveryAddress = @{
        latitude = 12.9800
        longitude = 77.6000
        address = "123 Test Street"
    }
} | ConvertTo-Json

$order = Invoke-RestMethod -Uri "$baseUrl/orders" -Method Post -Headers $riderHeaders -Body $orderBody
$orderId = $order.data.id
```

### **2. Mark Order PAID**
```powershell
Invoke-RestMethod -Uri "$baseUrl/orders/$orderId/status" -Method Put -Headers $riderHeaders -Body (@{
    status = "PAID"
} | ConvertTo-Json)
```

### **3. Mark Order READY (triggers rider assignment)**
```powershell
Invoke-RestMethod -Uri "$baseUrl/chef/orders/$orderId/ready" -Method Post -Headers $chefHeaders
```

---

## 📝 **Category 1: Status Management**

### **TC-DS-001: Update Status to Online**

**Objective**: Verify rider can go online

**Preconditions**:
- Rider registered
- Rider currently offline

**Test Steps**:
```powershell
# Go online
$body = @{
    status = "online"
} | ConvertTo-Json

$response = Invoke-RestMethod -Uri "$baseUrl/delivery/status" -Method Put -Headers $riderHeaders -Body $body

# Verify response
Write-Host "Status: $($response.data.deliveryStatus)"
```

**Expected Result**:
```json
{
  "success": true,
  "message": "Delivery status updated to online",
  "data": {
    "deliveryStatus": "online",
    "currentDeliveryId": null
  }
}
```

**Pass Criteria**:
- HTTP 200
- `deliveryStatus` = "online"
- `currentDeliveryId` = null

---

### **TC-DS-002: Update Status to Offline (Without Active Delivery)**

**Objective**: Verify rider can go offline when not delivering

**Preconditions**:
- Rider online
- No active delivery

**Test Steps**:
```powershell
$body = @{
    status = "offline"
} | ConvertTo-Json

$response = Invoke-RestMethod -Uri "$baseUrl/delivery/status" -Method Put -Headers $riderHeaders -Body $body
```

**Expected Result**:
```json
{
  "success": true,
  "message": "Delivery status updated to offline",
  "data": {
    "deliveryStatus": "offline",
    "currentDeliveryId": null
  }
}
```

**Pass Criteria**:
- HTTP 200
- `deliveryStatus` = "offline"

---

### **TC-DS-003: Cannot Go Offline with Active Delivery ❌**

**Objective**: Verify rider CANNOT go offline while delivering

**Preconditions**:
- Rider has active delivery

**Test Steps**:
```powershell
# Try to go offline
$body = @{
    status = "offline"
} | ConvertTo-Json

try {
    $response = Invoke-RestMethod -Uri "$baseUrl/delivery/status" -Method Put -Headers $riderHeaders -Body $body
} catch {
    $errorResponse = $_.ErrorDetails.Message | ConvertFrom-Json
    Write-Host "Error: $($errorResponse.message)"
}
```

**Expected Result**:
```json
{
  "success": false,
  "message": "Cannot go offline with active delivery",
  "errorCode": "ACTIVE_DELIVERY_EXISTS"
}
```

**Pass Criteria**:
- HTTP 400
- Error code: `ACTIVE_DELIVERY_EXISTS`

---

### **TC-DS-004: Get Current Status**

**Objective**: Verify rider can retrieve current status

**Test Steps**:
```powershell
$response = Invoke-RestMethod -Uri "$baseUrl/delivery/status" -Method Get -Headers $riderHeaders
Write-Host "Current Status: $($response.data.deliveryStatus)"
```

**Expected Result**:
```json
{
  "success": true,
  "message": "Delivery status retrieved",
  "data": {
    "deliveryStatus": "online",
    "currentDeliveryId": null
  }
}
```

**Pass Criteria**:
- HTTP 200
- Returns current status accurately

---

## 📝 **Category 2: Pending Requests**

### **TC-PR-001: Get Pending Requests (Rider Online)**

**Objective**: Verify online rider sees pending requests

**Preconditions**:
- Rider online
- At least 1 pending delivery request exists
- Request not expired

**Test Steps**:
```powershell
$response = Invoke-RestMethod -Uri "$baseUrl/delivery/requests/pending" -Method Get -Headers $riderHeaders

Write-Host "Found $($response.data.Count) pending requests"
foreach ($request in $response.data) {
    Write-Host "Request: $($request.id), Payout: ₹$($request.estimatedPayoutPaise / 100), Distance: $($request.distanceToPickupKm) km"
}
```

**Expected Result**:
```json
{
  "success": true,
  "message": "Found 2 pending requests",
  "data": [
    {
      "id": "request-uuid-123",
      "orderId": "order-uuid-456",
      "chefName": "Chef Ravi",
      "chefLocation": { "latitude": 12.9716, "longitude": 77.5946 },
      "customerLocation": { "latitude": 12.9800, "longitude": 77.6000 },
      "distanceToPickupKm": 3.2,
      "estimatedPayoutPaise": 8500,
      "expiresAt": "2026-02-22T10:35:30.000Z",
      "status": "pending"
    }
  ]
}
```

**Pass Criteria**:
- HTTP 200
- Array of pending requests
- Each request has required fields
- `expiresAt` > current time

---

### **TC-PR-002: No Pending Requests (Rider Offline)**

**Objective**: Verify offline rider doesn't see pending requests

**Preconditions**:
- Rider offline

**Test Steps**:
```powershell
# Ensure rider is offline
Invoke-RestMethod -Uri "$baseUrl/delivery/status" -Method Put -Headers $riderHeaders -Body (@{
    status = "offline"
} | ConvertTo-Json)

# Get pending requests
$response = Invoke-RestMethod -Uri "$baseUrl/delivery/requests/pending" -Method Get -Headers $riderHeaders
```

**Expected Result**:
```json
{
  "success": true,
  "message": "No requests (rider not online)",
  "data": []
}
```

**Pass Criteria**:
- HTTP 200
- Empty array `data: []`

---

### **TC-PR-003: Max 5 Pending Requests Shown**

**Objective**: Verify system limits pending requests to 5

**Preconditions**:
- More than 5 pending requests exist

**Test Steps**:
```powershell
$response = Invoke-RestMethod -Uri "$baseUrl/delivery/requests/pending" -Method Get -Headers $riderHeaders
Write-Host "Request count: $($response.data.Count)"
```

**Expected Result**:
- `data.length` ≤ 5

**Pass Criteria**:
- HTTP 200
- Max 5 requests returned

---

### **TC-PR-004: Expired Requests Not Shown**

**Objective**: Verify expired requests filtered out

**Preconditions**:
- Request with `expiresAt` < current time exists

**Test Steps**:
```powershell
$response = Invoke-RestMethod -Uri "$baseUrl/delivery/requests/pending" -Method Get -Headers $riderHeaders

# Check no expired requests
foreach ($request in $response.data) {
    $expiry = [DateTime]::Parse($request.expiresAt)
    if ($expiry -lt (Get-Date)) {
        Write-Host "FAIL: Expired request shown"
        exit 1
    }
}
```

**Pass Criteria**:
- All returned requests have `expiresAt` > current time

---

## 📝 **Category 3: Request Acceptance**

### **TC-RA-001: Accept Delivery Request**

**Objective**: Verify rider can accept delivery request

**Preconditions**:
- Rider online
- No active delivery
- Valid pending request exists

**Test Steps**:
```powershell
# Get pending requests
$pendingResponse = Invoke-RestMethod -Uri "$baseUrl/delivery/requests/pending" -Method Get -Headers $riderHeaders
$requestId = $pendingResponse.data[0].id

# Accept request
$body = @{
    deliveryRequestId = $requestId
    currentLocation = @{
        latitude = 12.9750
        longitude = 77.5980
        address = "Rider location"
    }
} | ConvertTo-Json

$response = Invoke-RestMethod -Uri "$baseUrl/delivery/requests/accept" -Method Post -Headers $riderHeaders -Body $body

Write-Host "Active Delivery ID: $($response.data.id)"
Write-Host "Order ID: $($response.data.orderId)"
Write-Host "State: $($response.data.state)"
```

**Expected Result**:
```json
{
  "success": true,
  "message": "Delivery accepted",
  "data": {
    "id": "active-delivery-uuid-999",
    "orderId": "order-uuid-456",
    "riderId": "uuid-rider-001",
    "chefName": "Chef Ravi",
    "chefLocation": { "latitude": 12.9716, "longitude": 77.5946 },
    "customerName": "John Doe",
    "customerLocation": { "latitude": 12.9800, "longitude": 77.6000 },
    "orderItems": ["Butter Chicken", "Garlic Naan"],
    "orderValuePaise": 55000,
    "state": "navigate_to_chef",
    "estimatedPayoutPaise": 8500,
    "assignedAt": "2026-02-22T10:36:00.000Z"
  }
}
```

**Pass Criteria**:
- HTTP 200
- `ActiveDelivery` created
- `state` = "navigate_to_chef"
- Rider status auto-updated to "delivering"
- `currentDeliveryId` set

---

### **TC-RA-002: Cannot Accept When Already Delivering ❌**

**Objective**: Verify rider cannot accept multiple deliveries

**Preconditions**:
- Rider has active delivery

**Test Steps**:
```powershell
# Try to accept another request
$body = @{
    deliveryRequestId = "another-request-uuid"
} | ConvertTo-Json

try {
    $response = Invoke-RestMethod -Uri "$baseUrl/delivery/requests/accept" -Method Post -Headers $riderHeaders -Body $body
} catch {
    $errorResponse = $_.ErrorDetails.Message | ConvertFrom-Json
    Write-Host "Error: $($errorResponse.message)"
}
```

**Expected Result**:
```json
{
  "success": false,
  "message": "Already have an active delivery",
  "errorCode": "ACTIVE_DELIVERY_EXISTS"
}
```

**Pass Criteria**:
- HTTP 400
- Error code: `ACTIVE_DELIVERY_EXISTS`

---

### **TC-RA-003: Cannot Accept Expired Request ❌**

**Objective**: Verify expired requests cannot be accepted

**Preconditions**:
- Request with `expiresAt` < current time

**Test Steps**:
```powershell
# Wait for request to expire (30 seconds)
Start-Sleep -Seconds 35

# Try to accept expired request
$body = @{
    deliveryRequestId = $expiredRequestId
} | ConvertTo-Json

try {
    $response = Invoke-RestMethod -Uri "$baseUrl/delivery/requests/accept" -Method Post -Headers $riderHeaders -Body $body
} catch {
    $errorResponse = $_.ErrorDetails.Message | ConvertFrom-Json
    Write-Host "Error: $($errorResponse.errorCode)"
}
```

**Expected Result**:
```json
{
  "success": false,
  "message": "Request has expired",
  "errorCode": "REQUEST_EXPIRED"
}
```

**Pass Criteria**:
- HTTP 400
- Error code: `REQUEST_EXPIRED`
- Request status updated to "EXPIRED"

---

### **TC-RA-004: Cannot Accept Already Processed Request ❌**

**Objective**: Verify request can only be accepted once

**Preconditions**:
- Request already accepted by another rider

**Test Steps**:
```powershell
# Rider 2 tries to accept same request
$body = @{
    deliveryRequestId = $alreadyAcceptedRequestId
} | ConvertTo-Json

try {
    $response = Invoke-RestMethod -Uri "$baseUrl/delivery/requests/accept" -Method Post -Headers $rider2Headers -Body $body
} catch {
    $errorResponse = $_.ErrorDetails.Message | ConvertFrom-Json
    Write-Host "Error: $($errorResponse.message)"
}
```

**Expected Result**:
```json
{
  "success": false,
  "message": "Request already processed",
  "errorCode": "REQUEST_ALREADY_PROCESSED"
}
```

**Pass Criteria**:
- HTTP 400
- Error code: `REQUEST_ALREADY_PROCESSED`

---

### **TC-RA-005: Rider Status Auto-Updated on Accept**

**Objective**: Verify rider status changes to "delivering" on accept

**Test Steps**:
```powershell
# Accept delivery
$acceptBody = @{
    deliveryRequestId = $requestId
} | ConvertTo-Json
Invoke-RestMethod -Uri "$baseUrl/delivery/requests/accept" -Method Post -Headers $riderHeaders -Body $acceptBody

# Get status
$statusResponse = Invoke-RestMethod -Uri "$baseUrl/delivery/status" -Method Get -Headers $riderHeaders
Write-Host "Status after accept: $($statusResponse.data.deliveryStatus)"
```

**Expected Result**:
- `deliveryStatus` = "delivering"
- `currentDeliveryId` set to active delivery ID

**Pass Criteria**:
- Auto-status sync working

---

## 📝 **Category 4: Request Rejection**

### **TC-RJ-001: Reject Delivery Request (Reason: too_far)**

**Objective**: Verify rider can reject request with reason

**Preconditions**:
- Valid pending request

**Test Steps**:
```powershell
$body = @{
    deliveryRequestId = $requestId
    reason = "too_far"
    note = "Distance exceeds my preference"
} | ConvertTo-Json

$response = Invoke-RestMethod -Uri "$baseUrl/delivery/requests/reject" -Method Post -Headers $riderHeaders -Body $body
```

**Expected Result**:
```json
{
  "success": true,
  "message": "Delivery rejected"
}
```

**Pass Criteria**:
- HTTP 200
- Request status → "REJECTED"
- Rejection reason logged
- Re-assignment triggered

---

### **TC-RJ-002: Rejection Triggers Re-Assignment**

**Objective**: Verify rejected request reassigned to next rider

**Test Steps**:
```powershell
# Rider 1 rejects
$body = @{
    deliveryRequestId = $requestId
    reason = "busy"
} | ConvertTo-Json
Invoke-RestMethod -Uri "$baseUrl/delivery/requests/reject" -Method Post -Headers $rider1Headers -Body $body

# Wait 5 seconds for re-assignment
Start-Sleep -Seconds 5

# Rider 2 checks pending requests
$rider2Requests = Invoke-RestMethod -Uri "$baseUrl/delivery/requests/pending" -Method Get -Headers $rider2Headers
Write-Host "Rider 2 received re-assigned request: $($rider2Requests.data.Count -gt 0)"
```

**Pass Criteria**:
- Rider 2 receives new request for same order

---

### **TC-RJ-003: Rejection Reason Enum Validation ❌**

**Objective**: Verify only valid rejection reasons accepted

**Test Steps**:
```powershell
$body = @{
    deliveryRequestId = $requestId
    reason = "invalid_reason"
} | ConvertTo-Json

try {
    $response = Invoke-RestMethod -Uri "$baseUrl/delivery/requests/reject" -Method Post -Headers $riderHeaders -Body $body
} catch {
    $errorResponse = $_.ErrorDetails.Message | ConvertFrom-Json
    Write-Host "Validation Error: $($errorResponse.message)"
}
```

**Expected Result**:
- HTTP 400
- Validation error for invalid enum

**Pass Criteria**:
- Only accepts: `too_far`, `busy`, `low_payout`, `other`

---

## 📝 **Category 5: State Updates**

### **TC-SU-001: Update State to picked_up**

**Objective**: Verify rider can mark food picked up

**Preconditions**:
- Active delivery exists
- Current state = "navigate_to_chef"

**Test Steps**:
```powershell
$body = @{
    deliveryId = $activeDeliveryId
    state = "picked_up"
    currentLocation = @{
        latitude = 12.9716
        longitude = 77.5946
    }
} | ConvertTo-Json

$response = Invoke-RestMethod -Uri "$baseUrl/delivery/active/state" -Method Put -Headers $riderHeaders -Body $body
Write-Host "New State: $($response.data.state)"
Write-Host "Picked Up At: $($response.data.pickedUpAt)"
```

**Expected Result**:
```json
{
  "success": true,
  "message": "Delivery state updated to picked_up",
  "data": {
    "id": "active-delivery-uuid-999",
    "state": "picked_up",
    "pickedUpAt": "2026-02-22T10:50:00.000Z"
  }
}
```

**Pass Criteria**:
- HTTP 200
- `state` = "picked_up"
- `pickedUpAt` timestamp set
- Order status → "OUT_FOR_DELIVERY"

---

### **TC-SU-002: Update State to en_route**

**Objective**: Verify rider can mark en route to customer

**Preconditions**:
- Active delivery with state = "picked_up"

**Test Steps**:
```powershell
$body = @{
    deliveryId = $activeDeliveryId
    state = "en_route"
} | ConvertTo-Json

$response = Invoke-RestMethod -Uri "$baseUrl/delivery/active/state" -Method Put -Headers $riderHeaders -Body $body
```

**Expected Result**:
- `state` = "en_route"

---

### **TC-SU-003: Invalid State Transition ❌**

**Objective**: Verify backward state transitions rejected

**Preconditions**:
- Active delivery with state = "picked_up"

**Test Steps**:
```powershell
# Try to go back to navigate_to_chef
$body = @{
    deliveryId = $activeDeliveryId
    state = "navigate_to_chef"
} | ConvertTo-Json

try {
    $response = Invoke-RestMethod -Uri "$baseUrl/delivery/active/state" -Method Put -Headers $riderHeaders -Body $body
} catch {
    $errorResponse = $_.ErrorDetails.Message | ConvertFrom-Json
    Write-Host "Error: $($errorResponse.message)"
}
```

**Expected Result**:
```json
{
  "success": false,
  "message": "Cannot transition from picked_up to navigate_to_chef",
  "errorCode": "INVALID_STATE_TRANSITION"
}
```

**Pass Criteria**:
- HTTP 400
- Error code: `INVALID_STATE_TRANSITION`

---

### **TC-SU-004: State Transition Sends Customer Notification**

**Objective**: Verify customer notified on state changes

**Test Steps**:
```powershell
# Update state to picked_up
$body = @{
    deliveryId = $activeDeliveryId
    state = "picked_up"
} | ConvertTo-Json
Invoke-RestMethod -Uri "$baseUrl/delivery/active/state" -Method Put -Headers $riderHeaders -Body $body

# Check customer notifications (admin endpoint)
$notifications = Invoke-RestMethod -Uri "$baseUrl/notifications/user/$customerId" -Method Get -Headers $adminHeaders
Write-Host "Latest notification: $($notifications.data[0].type)"
```

**Expected Result**:
- Notification type: `delivery.picked_up`
- Sent to customer

---

## 📝 **Category 6: Delivery Completion**

### **TC-DC-001: Complete Delivery**

**Objective**: Verify rider can complete delivery

**Preconditions**:
- Active delivery exists
- State = "en_route" or "delivered"

**Test Steps**:
```powershell
$body = @{
    deliveryId = $activeDeliveryId
    deliveryProof = "https://s3.amazonaws.com/proof-photo.jpg"
    finalLocation = @{
        latitude = 12.9800
        longitude = 77.6000
    }
} | ConvertTo-Json

$response = Invoke-RestMethod -Uri "$baseUrl/delivery/active/complete" -Method Post -Headers $riderHeaders -Body $body

Write-Host "Delivered At: $($response.data.deliveredAt)"
Write-Host "Payout: ₹$($response.data.actualPayoutPaise / 100)"
```

**Expected Result**:
```json
{
  "success": true,
  "message": "Delivery completed",
  "data": {
    "id": "active-delivery-uuid-999",
    "state": "delivered",
    "deliveredAt": "2026-02-22T11:05:00.000Z",
    "deliveryProof": "https://s3.amazonaws.com/proof-photo.jpg",
    "actualPayoutPaise": 8500
  }
}
```

**Pass Criteria**:
- HTTP 200
- `state` = "delivered"
- `deliveredAt` timestamp set
- Order status → "DELIVERED"

---

### **TC-DC-002: Rider Status Reset on Completion**

**Objective**: Verify rider status back to "online" after completion

**Test Steps**:
```powershell
# Complete delivery
$completeBody = @{
    deliveryId = $activeDeliveryId
} | ConvertTo-Json
Invoke-RestMethod -Uri "$baseUrl/delivery/active/complete" -Method Post -Headers $riderHeaders -Body $completeBody

# Get status
$statusResponse = Invoke-RestMethod -Uri "$baseUrl/delivery/status" -Method Get -Headers $riderHeaders
Write-Host "Status after complete: $($statusResponse.data.deliveryStatus)"
Write-Host "Current Delivery ID: $($statusResponse.data.currentDeliveryId)"
```

**Expected Result**:
- `deliveryStatus` = "online"
- `currentDeliveryId` = null

**Pass Criteria**:
- Rider available for new assignments

---

### **TC-DC-003: Customer Notified on Completion**

**Objective**: Verify customer receives delivery confirmation

**Test Steps**:
```powershell
# Complete delivery
Invoke-RestMethod -Uri "$baseUrl/delivery/active/complete" -Method Post -Headers $riderHeaders -Body $completeBody

# Check customer notifications
$notifications = Invoke-RestMethod -Uri "$baseUrl/notifications/user/$customerId" -Method Get -Headers $adminHeaders
$latestNotif = $notifications.data[0]
Write-Host "Notification Type: $($latestNotif.type)"
```

**Expected Result**:
- Notification type: `delivery.completed`

---

## 📝 **Category 7: Busy Status**

### **TC-BS-001: Busy Status Check (Not Busy)**

**Objective**: Verify rider not busy with 1 delivery (max = 2)

**Preconditions**:
- Rider has 1 active delivery
- `maxConcurrentDeliveries` = 2

**Test Steps**:
```powershell
$response = Invoke-RestMethod -Uri "$baseUrl/delivery/busy-status" -Method Get -Headers $riderHeaders
Write-Host "Is Busy: $($response.data.isBusy)"
Write-Host "Active Count: $($response.data.activeDeliveryCount)"
```

**Expected Result**:
```json
{
  "success": true,
  "message": "Rider is available for new assignments",
  "data": {
    "isBusy": false,
    "activeDeliveryCount": 1,
    "maxConcurrentDeliveries": 2,
    "reason": null
  }
}
```

**Pass Criteria**:
- `isBusy` = false

---

### **TC-BS-002: Busy Status Check (Busy)**

**Objective**: Verify rider busy at max capacity

**Preconditions**:
- Rider has 2 active deliveries
- `maxConcurrentDeliveries` = 2

**Test Steps**:
```powershell
$response = Invoke-RestMethod -Uri "$baseUrl/delivery/busy-status" -Method Get -Headers $riderHeaders
```

**Expected Result**:
```json
{
  "success": true,
  "message": "Rider is busy with active deliveries",
  "data": {
    "isBusy": true,
    "activeDeliveryCount": 2,
    "maxConcurrentDeliveries": 2,
    "reason": "At maximum capacity"
  }
}
```

**Pass Criteria**:
- `isBusy` = true

---

## 📝 **Category 8: Assignment Timeout**

### **TC-AT-001: Assignment Timeout Retry (Rider Doesn't Accept)**

**Objective**: Verify system retries with next rider if no acceptance

**Preconditions**:
- Rider 1 assigned
- Rider 1 does not accept within 30 seconds

**Test Steps**:
```powershell
# Mark order READY (triggers assignment)
Invoke-RestMethod -Uri "$baseUrl/chef/orders/$orderId/ready" -Method Post -Headers $chefHeaders

# Wait for assignment timeout (30 seconds)
Start-Sleep -Seconds 35

# Check if Rider 2 received re-assigned request
$rider2Requests = Invoke-RestMethod -Uri "$baseUrl/delivery/requests/pending" -Method Get -Headers $rider2Headers
Write-Host "Rider 2 has request after retry: $($rider2Requests.data.Count -gt 0)"
```

**Expected Result**:
- Rider 2 receives request (retry triggered)

**Pass Criteria**:
- Retry count incremented
- New rider assigned

---

### **TC-AT-002: Max Retries Exceeded → Auto-Cancel**

**Objective**: Verify order auto-cancelled after 5 failed assignments

**Preconditions**:
- 5 riders assigned, none accept

**Test Steps**:
```powershell
# Wait for 5 assignment cycles (5 × 30s = 150s)
Start-Sleep -Seconds 160

# Check order status
$order = Invoke-RestMethod -Uri "$baseUrl/orders/$orderId" -Method Get -Headers $customerHeaders
Write-Host "Order Status: $($order.data.status)"
```

**Expected Result**:
- Order status = "CANCELLED"
- Cancellation reason = "RIDER_NOT_ACCEPTED"
- Refund initiated

**Pass Criteria**:
- Auto-cancel after max retries

---

## 📝 **Category 9: Pickup Timeout**

### **TC-PT-001: Pickup Timeout → Auto-Cancel**

**Objective**: Verify order auto-cancelled if rider doesn't pick up within 20 minutes

**Preconditions**:
- Rider accepted delivery
- 20 minutes passed without pickup

**Test Steps**:
```powershell
# Accept delivery
$acceptBody = @{
    deliveryRequestId = $requestId
} | ConvertTo-Json
Invoke-RestMethod -Uri "$baseUrl/delivery/requests/accept" -Method Post -Headers $riderHeaders -Body $acceptBody

# Wait 21 minutes
Start-Sleep -Seconds 1260

# Check order status
$order = Invoke-RestMethod -Uri "$baseUrl/orders/$orderId" -Method Get -Headers $customerHeaders
Write-Host "Order Status: $($order.data.status)"
```

**Expected Result**:
- Order status = "CANCELLED"
- Cancellation reason = "PICKUP_TIMEOUT"
- Refund initiated
- Rider flagged

**Pass Criteria**:
- Auto-cancel after pickup timeout

---

### **TC-PT-002: No Timeout if Picked Up**

**Objective**: Verify no timeout if rider picks up food

**Test Steps**:
```powershell
# Accept delivery
Invoke-RestMethod -Uri "$baseUrl/delivery/requests/accept" -Method Post -Headers $riderHeaders -Body $acceptBody

# Update state to picked_up (within 20 min)
Start-Sleep -Seconds 300 # 5 minutes
$pickupBody = @{
    deliveryId = $activeDeliveryId
    state = "picked_up"
} | ConvertTo-Json
Invoke-RestMethod -Uri "$baseUrl/delivery/active/state" -Method Put -Headers $riderHeaders -Body $pickupBody

# Wait another 20 minutes
Start-Sleep -Seconds 1200

# Check order not cancelled
$order = Invoke-RestMethod -Uri "$baseUrl/orders/$orderId" -Method Get -Headers $customerHeaders
Write-Host "Order Status: $($order.data.status)"
```

**Expected Result**:
- Order status = "OUT_FOR_DELIVERY"
- No cancellation

---

## 📝 **Category 10: Delivery Timeout**

### **TC-DT-001: Delivery Timeout → Auto-Cancel**

**Objective**: Verify order auto-cancelled if rider doesn't deliver within timeout

**Preconditions**:
- Rider picked up food
- 45 minutes passed without delivery (or ETA + 15 min)

**Test Steps**:
```powershell
# Accept and pickup
Invoke-RestMethod -Uri "$baseUrl/delivery/requests/accept" -Method Post -Headers $riderHeaders -Body $acceptBody
Invoke-RestMethod -Uri "$baseUrl/delivery/active/state" -Method Put -Headers $riderHeaders -Body $pickupBody

# Wait 50 minutes
Start-Sleep -Seconds 3000

# Check order status
$order = Invoke-RestMethod -Uri "$baseUrl/orders/$orderId" -Method Get -Headers $customerHeaders
Write-Host "Order Status: $($order.data.status)"
```

**Expected Result**:
- Order status = "CANCELLED"
- Cancellation reason = "DELIVERY_TIMEOUT"
- Refund initiated
- Rider flagged for suspension

**Pass Criteria**:
- Auto-cancel after delivery timeout

---

### **TC-DT-002: Timeout Based on ETA**

**Objective**: Verify timeout uses ETA + 15 min if greater than 45 min

**Preconditions**:
- Order ETA = 60 minutes
- Timeout should be 60 + 15 = 75 minutes

**Test Steps**:
```powershell
# Wait 50 minutes (should NOT cancel)
Start-Sleep -Seconds 3000
$order = Invoke-RestMethod -Uri "$baseUrl/orders/$orderId" -Method Get -Headers $customerHeaders
Write-Host "Status at 50 min: $($order.data.status)"

# Wait 80 minutes total (should cancel)
Start-Sleep -Seconds 1800
$order = Invoke-RestMethod -Uri "$baseUrl/orders/$orderId" -Method Get -Headers $customerHeaders
Write-Host "Status at 80 min: $($order.data.status)"
```

**Expected Result**:
- No cancel at 50 min
- Cancel at 80 min (ETA + 15)

---

## ⚡ **Performance Tests**

### **PT-001: Pending Requests Query Speed**

**Objective**: Verify pending requests query under 100ms

**Test Steps**:
```powershell
$startTime = Get-Date
$response = Invoke-RestMethod -Uri "$baseUrl/delivery/requests/pending" -Method Get -Headers $riderHeaders
$endTime = Get-Date
$duration = ($endTime - $startTime).TotalMilliseconds
Write-Host "Query Duration: $duration ms"
```

**Pass Criteria**:
- Response time < 100ms

---

### **PT-002: Assignment Speed**

**Objective**: Verify rider assigned within 3 seconds

**Test Steps**:
```powershell
$startTime = Get-Date
Invoke-RestMethod -Uri "$baseUrl/chef/orders/$orderId/ready" -Method Post -Headers $chefHeaders

# Poll for assignment
do {
    Start-Sleep -Milliseconds 500
    $order = Invoke-RestMethod -Uri "$baseUrl/orders/$orderId" -Method Get -Headers $customerHeaders
} until ($order.data.deliveryPartnerId -ne $null -or ((Get-Date) - $startTime).TotalSeconds -gt 10)

$endTime = Get-Date
$duration = ($endTime - $startTime).TotalSeconds
Write-Host "Assignment Duration: $duration seconds"
```

**Pass Criteria**:
- Assignment < 3 seconds

---

## 🧩 **Edge Case Tests**

### **EC-001: Delivery Cleanup (Inconsistent State)**

**Objective**: Verify system cleans up inconsistent state

**Preconditions**:
- User has `currentDeliveryId` set
- But active delivery doesn't exist

**Test Steps**:
```powershell
# Manually set inconsistent state (admin endpoint)
Invoke-RestMethod -Uri "$baseUrl/admin/users/$riderId/set-current-delivery" -Method Put -Headers $adminHeaders -Body (@{
    currentDeliveryId = "non-existent-uuid"
} | ConvertTo-Json)

# Get active delivery (should trigger cleanup)
$response = Invoke-RestMethod -Uri "$baseUrl/delivery/active" -Method Get -Headers $riderHeaders

# Get status (should be cleaned)
$status = Invoke-RestMethod -Uri "$baseUrl/delivery/status" -Method Get -Headers $riderHeaders
Write-Host "Status: $($status.data.deliveryStatus)"
Write-Host "Current Delivery ID: $($status.data.currentDeliveryId)"
```

**Expected Result**:
- `currentDeliveryId` = null
- `deliveryStatus` = "online"

**Pass Criteria**:
- Inconsistent state cleaned up

---

### **EC-002: Concurrent Accept Attempts (Race Condition)**

**Objective**: Verify only one rider can accept request

**Test Steps**:
```powershell
# Start two accept requests simultaneously
$job1 = Start-Job -ScriptBlock {
    Invoke-RestMethod -Uri "$using:baseUrl/delivery/requests/accept" -Method Post -Headers $using:rider1Headers -Body $using:acceptBody
}

$job2 = Start-Job -ScriptBlock {
    Invoke-RestMethod -Uri "$using:baseUrl/delivery/requests/accept" -Method Post -Headers $using:rider2Headers -Body $using:acceptBody
}

# Wait for both to complete
Wait-Job $job1, $job2
$result1 = Receive-Job $job1
$result2 = Receive-Job $job2

Write-Host "Rider 1 Success: $($result1.success)"
Write-Host "Rider 2 Success: $($result2.success)"
```

**Expected Result**:
- Only one rider succeeds
- Other gets `REQUEST_ALREADY_PROCESSED`

---

## ✅ **Test Checklist**

### **Status Management (4 tests)**
- [ ] TC-DS-001: Go online
- [ ] TC-DS-002: Go offline (no delivery)
- [ ] TC-DS-003: Cannot go offline with delivery
- [ ] TC-DS-004: Get current status

### **Pending Requests (4 tests)**
- [ ] TC-PR-001: Get pending requests (online)
- [ ] TC-PR-002: No requests (offline)
- [ ] TC-PR-003: Max 5 requests
- [ ] TC-PR-004: Expired filtered out

### **Request Acceptance (5 tests)**
- [ ] TC-RA-001: Accept request
- [ ] TC-RA-002: Cannot accept multiple
- [ ] TC-RA-003: Cannot accept expired
- [ ] TC-RA-004: Cannot accept processed
- [ ] TC-RA-005: Status auto-updated

### **Request Rejection (3 tests)**
- [ ] TC-RJ-001: Reject with reason
- [ ] TC-RJ-002: Triggers re-assignment
- [ ] TC-RJ-003: Reason enum validation

### **State Updates (4 tests)**
- [ ] TC-SU-001: Update to picked_up
- [ ] TC-SU-002: Update to en_route
- [ ] TC-SU-003: Invalid transition
- [ ] TC-SU-004: Customer notification

### **Delivery Completion (3 tests)**
- [ ] TC-DC-001: Complete delivery
- [ ] TC-DC-002: Status reset
- [ ] TC-DC-003: Customer notified

### **Busy Status (2 tests)**
- [ ] TC-BS-001: Not busy
- [ ] TC-BS-002: Busy

### **Assignment Timeout (2 tests)**
- [ ] TC-AT-001: Timeout retry
- [ ] TC-AT-002: Max retries → cancel

### **Pickup Timeout (2 tests)**
- [ ] TC-PT-001: Pickup timeout → cancel
- [ ] TC-PT-002: No timeout if picked up

### **Delivery Timeout (2 tests)**
- [ ] TC-DT-001: Delivery timeout → cancel
- [ ] TC-DT-002: ETA-based timeout

### **Performance (2 tests)**
- [ ] PT-001: Pending requests < 100ms
- [ ] PT-002: Assignment < 3s

### **Edge Cases (2 tests)**
- [ ] EC-001: State cleanup
- [ ] EC-002: Race condition

**Total**: 35 test cases

---

## � **Category 11: COD Assignment & Retry-Until-Pickup (March 2026 Bug Fix)**

### TC-DEL-36: COD order triggers immediate rider assignment on confirmation

**Type:** Bug Regression  
**Feature area:** `order.service.ts` → `createPaymentIntent()` COD branch  
**Priority:** P0

**Preconditions:**
- At least one rider is online and active in the same city as the order

**Steps:**
1. Place an order using Cash on Delivery (COD)
2. Call `POST /v1/orders/payment-intent` with `{ paymentMethod: "COD" }`
3. Observe order status and delivery partner assignment immediately after response

**Expected result:** Within ~2 seconds of the COD confirmation response, `order.deliveryPartnerId` is set (or a retry cycle picks it up within 30s if no riders are available).  
**Actual result (before fix):** `autoAssignRider()` was never called in the COD branch. The order waited up to 30s for the retry cron to fire.  
**Fix applied:** Added `this.deliveryAssignmentService.autoAssignRider(order.id)` non-blocking call immediately after saving the COD order in `createPaymentIntent()`.  
**Regression test:** `apps/chefooz-apis/src/modules/order/order.service.spec.ts` — COD auto-assignment test  
**Status:** Fixed ✅

---

### TC-DEL-37: Retry cron continues after maxAssignmentRetries (no hard stop)

**Type:** Bug Regression  
**Feature area:** `assignment-retry.service.ts`  
**Priority:** P1

**Preconditions:**
- Order is paid (COD or UPI), no riders available in city
- `maxAssignmentRetries` is reached

**Steps:**
1. Create a paid order with no riders available
2. Wait for `maxAssignmentRetries` cron cycles (~2.5 minutes for 5 retries × 30s)
3. Bring a rider online
4. Wait for next cron cycle (up to 30s)

**Expected result:** After a rider comes online, the order is assigned regardless of `assignmentRetryCount`.  
**Actual result (before fix):** After `maxAssignmentRetries`, the cron excluded the order with `WHERE assignmentRetryCount < :maxRetries`. The order was silently abandoned.  
**Fix applied:** Removed the upper-bound `assignmentRetryCount` filter from the cron query. After threshold, order is flagged `needsManualAssignment = true` for ops visibility but continues to be retried.  
**Status:** Fixed ✅

---

### TC-DEL-38: needsManualAssignment flag is set and visible in admin dashboard

**Type:** Manual  
**Feature area:** `assignment-retry.service.ts`, `admin-orders.controller.ts`, admin orders page  
**Priority:** P1

**Preconditions:**
- Order has `assignmentRetryCount >= maxAssignmentRetries` and no rider assigned

**Steps:**
1. Navigate to Admin → Orders → "Unassigned" tab
2. Check for orders flagged with warning icon

**Expected result:** Orders with `needsManualAssignment = true` appear with a warning icon and are sorted to the top. The "Assign Rider" button is visible.  
**Status:** Implemented ✅

---

### TC-DEL-39: Admin manual rider assignment — successful flow

**Type:** Manual  
**Feature area:** `GET /v1/admin/orders/:id/eligible-riders`, `POST /v1/admin/orders/:id/assign-rider`  
**Priority:** P1

**Preconditions:**
- Unassigned paid order exists
- Admin is logged in

**Steps:**
1. Navigate to Admin → Orders → Unassigned tab
2. Click "Assign Rider" on any unassigned order
3. Select a rider from the dialog (including offline/busy ones)
4. Click "Confirm Assignment"

**Expected result:** 
- Rider receives assignment notification
- `order.deliveryPartnerId` is set
- `order.needsManualAssignment` is cleared to `false`
- Unassigned orders list refreshes (order disappears)  
**Status:** Implemented ✅

---

### TC-DEL-40: Admin manual assignment — bypass busy check

**Type:** Manual  
**Feature area:** `DeliveryAssignmentService.adminManualAssignRider()`  
**Priority:** P2

**Preconditions:**
- A rider exists with `isBusy = true` (at capacity)

**Steps:**
1. Open the Assign Rider dialog for an unassigned order
2. Select the busy rider
3. Confirm

**Expected result:** Assignment succeeds (admin override bypasses `isBusy` check).  
**Expected result (auto-assign):** The same rider would be skipped by `autoAssignRider` since it respects `isBusy`.  
**Status:** Implemented ✅

---

### TC-DEL-41: Auto-assignment initial failure leaves order in retry queue

**Type:** Bug Regression  
**Feature area:** Auto-assignment / AssignmentRetryService  
**Priority:** P0

**Preconditions:**
- No online riders available at the moment of order creation
- A rider comes online within 30 seconds of the order being placed

**Steps:**
1. Place an order (stubbed or real payment) with no online riders
2. Observe backend logs — `autoAssignRider` fires and returns "No eligible rider found"
3. Wait 30 seconds — rider comes online and sends heartbeat
4. Wait for the `AssignmentRetryService` cron to fire (every 30s)

**Expected result:** At step 4, the retry cron picks up the order (`needsManualAssignment = false`, `assignmentRetryCount < max`), calls `autoAssignRider` again, and successfully assigns the now-online rider.  
**Actual result (before fix):** `autoAssignRider` called `markNeedsManualAttention()` on the first failed attempt, setting `needsManualAssignment = true` immediately. The retry cron's query filters `needsManualAssignment = false`, so the order was permanently excluded from auto-retry after just 1 attempt. The rider who came online 17 seconds later was never tried.  
**Fix applied:** Removed `markNeedsManualAttention()` call from `autoAssignRider()`. The method now logs a warning and returns, leaving the order for the `AssignmentRetryService` to retry. `markNeedsManualAttention` is only called from `retryAssignment()` (when `retryCount >= maxAssignmentRetries`) and from the cron's bulk-UPDATE step.  
**Regression test:** `apps/chefooz-apis/src/modules/delivery/services/delivery-assignment.service.spec.ts`  
**Status:** Fixed ✅

---

### TC-DEL-42: Customer coins credited after real rider delivery

**Type:** Bug Regression  
**Feature area:** Rider delivery → coin reward  
**Priority:** P0

**Preconditions:**
- Order placed, accepted, and progressed to READY by chef
- Rider assigned and picks up order
- `RAZORPAY_STUB_PAYMENTS=false` (real or stub UPI — does not matter, bug existed in both)

**Steps:**
1. Rider taps "Mark Delivered" in the rider app
2. Check user's `coins` column in the DB immediately after

**Expected result:** User's `coins` are incremented by `floor(dishTotal_paise / 100) × reputationMultiplier`. If the order was linked to a `USER_REVIEW` reel, a pending commission ledger entry is also created for the reel creator.  
**Actual result (before fix):** `handleOrderDelivered()` in `OrderService` was only called from the payment simulator. Real rider delivery (`rider-orders.service.ts DELIVERED case`) never called it → 0 coins credited, 0 commission created.  
**Fix applied:** Injected `OrderService` into `RiderOrdersService`; added non-blocking `this.orderService.handleOrderDelivered(orderId)` call after rider earning record is created in the `DELIVERED` block.  
**Regression test:** `apps/chefooz-apis/src/modules/rider-orders/rider-orders.service.spec.ts`  
**Status:** Fixed ✅

---

### TC-DEL-43: Reel creator commission created after real delivery of attributed order

**Type:** Bug Regression  
**Feature area:** Rider delivery → reel commission  
**Priority:** P1

**Preconditions:**
- Order was placed after watching a `USER_REVIEW` reel (order has `attribution.linkedReelId`)
- Rider completes delivery

**Steps:**
1. Rider marks order DELIVERED  
2. Check `commission_ledger` table for a `PENDING` entry with this `orderId`

**Expected result:** A `PENDING` commission entry exists. On next `processPendingCommissionsJob` run (or admin trigger `POST /admin/jobs/process-commissions`), the reel creator's `coins` are incremented.  
**Actual result (before fix):** Same root cause as TC-DEL-42 — `handleOrderDelivered` never called on real delivery.  
**Fix applied:** Same fix — `handleOrderDelivered` now wired into real delivery path.  
**Regression test:** `apps/chefooz-apis/src/modules/rider-orders/rider-orders.service.spec.ts`  
**Status:** Fixed ✅

---

### TC-DEL-44: Assignment retry cron runs only once across multiple pods (distributed lock)

**Type:** Bug Regression  
**Feature area:** AssignmentRetryService / multi-pod deployment  
**Priority:** P0

**Preconditions:**
- Backend running with 2+ pod/worker instances (observable via PID 18 and PID 60 in logs)
- At least 1 unassigned paid order exists

**Steps:**
1. Observe backend logs during a 30s retry cycle with 2+ workers
2. Before fix: both workers log "🚴 Retrying assignment for order X (attempt 1/15)" simultaneously for the same order
3. After fix: only one worker logs the retry; all others log "Distributed lock held by another instance, skipping..."

**Expected result:** Exactly one pod acquires the Redis lock per cron tick and processes the retry cycle. All other pods skip. The lock auto-expires in 25 s (< 30 s cron interval), so no pod is permanently blocked.  
**Actual result (before fix):** `isRunning` was an in-memory boolean — each pod had its own `isRunning = false`, so all pods started the cycle simultaneously. Every unassigned order received duplicate `autoAssignRider` calls. If a rider was available, the pessimistic DB lock inside `assignRiderToOrder` prevented double-assignment, but `assignmentRetryCount` was double-incremented — burning through the retry budget twice as fast.  
**Fix applied:** Replaced `isRunning` with `cacheService.acquireLock('lock:assignment-retry:cron', 25_000)` (Redis `SET NX PX`). Lock is released in `finally` block via `releaseLock` (Lua atomic check-and-delete). TTL of 25 s ensures auto-expiry if the pod crashes mid-cycle.  
**Regression test:** `apps/chefooz-apis/src/modules/delivery/services/assignment-retry.service.spec.ts`  
**Status:** Fixed ✅

---

### TC-DEL-45: `handleOrderDelivered` early-return skipped notification and ratingPending for non-attributed orders

**Type:** Bug Regression  
**Feature area:** Order delivery → post-delivery side effects  
**Priority:** P0

**Preconditions:**
- Order placed directly (not via a reel) — `order.attribution.linkedReelId` is null/absent
- Rider delivers order

**Steps:**
1. Rider marks order DELIVERED
2. Check that `order.delivered` push notification was sent to buyer
3. Check that `order.ratingPending = true` was set in DB

**Expected result:** Notification sent and `ratingPending = true`. Log shows `✅ Order X marked for rider rating`. No coin credit to buyer (coins are NOT earned for placing orders — only reel creators earn commission coins when their reel drives orders).  
**Actual result (before fix):** `handleOrderDelivered()` checked `order.attribution?.linkedReelId` and returned early when absent — "has no attribution, skipping commission" — skipping the `order.delivered` notification and `ratingPending` update for ALL non-reel orders.  
**Fix applied (TC-DEL-45):** Restructured `handleOrderDelivered()` so notification (step 1) and ratingPending (step 2) run unconditionally. The attribution check only gates step 3 (commission block).  
**Business model clarification (TC-DEL-45 followup):** An earlier version of this code erroneously included a "buyer loyalty coins" step that credited coins to the buyer on every delivery. This was REMOVED — it does not match the Chefooz business model. Coins are earned ONLY by reel creators via commission when their USER_REVIEW reel drives another user's order. Buyers receive no coins for placing orders.  
**Files changed:**  
- `apps/chefooz-apis/src/modules/order/order.service.ts` — removed buyer loyalty coin credit (step 2), removed `UserService` injection  
- `apps/chefooz-apis/src/modules/order/order.module.ts` — removed `UserModule` import  
**Regression test:** `apps/chefooz-apis/src/modules/order/order.service.spec.ts`  
**Status:** Fixed ✅

---

### TC-DEL-46: Commission created for reel-attributed order placed via "View Menu" CTA

**Type:** Bug Regression  
**Feature area:** Attribution chain — reel CTA → cart → checkout → commission  
**Priority:** P0

**Preconditions:**
- User A has posted a USER_REVIEW reel with a linkedMenu (MENU_SHOWCASE) or any shoppable reel
- User B views that reel and taps the "View Menu →" CTA

**Steps:**
1. User B taps "View Menu" on User A's reel in the feed  
2. User B is navigated to the chef page; cart contains the first item from the reel
3. User B proceeds to checkout and places the order
4. Rider delivers the order
5. Check `order.attribution.linkedReelId` in DB — should be the reel's ID
6. Check commission ledger — should have a PENDING entry for User A

**Expected result:** `order.attribution.linkedReelId` = User A's reel ID. After `processPendingCommissionsJob` runs, User A receives coins per the V2 commission formula.  
**Actual result (before fix):** `addToCart.mutate({ menuItemId, quantity: 1 })` was called without `reelId`. `cart.service.addItem()` never wrote attribution to Redis. `checkoutCart()` read null attribution. `order.attribution` = null in DB. Commission check returned "has no attribution, skipping commission".  
**Fix applied (5-file chain):**  
1. `libs/types/src/lib/cart.types.ts` — Added `reelId?: string` to `AddCartItemPayload`  
2. `apps/chefooz-apis/src/modules/cart/dto/add-item.dto.ts` — Added `@IsOptional() @IsString() reelId?: string`  
3. `apps/chefooz-apis/src/modules/cart/cart.service.ts` — `addItem()` now writes `cart:{userId}:attribution` to Redis when `dto.reelId` present (same format as `addCreatorOrderToCart`)  
4. `apps/chefooz-app/src/components/ReelCard.tsx` — `addToCart.mutate(...)` now passes `reelId: item.id`  
5. `apps/chefooz-app/src/components/home-feed/OrderablePostCard.tsx` — Same fix, passes `reelId: post.id`  
**Regression test:** `apps/chefooz-apis/src/modules/cart/cart.service.spec.ts`  
**Status:** Fixed ✅

---

## 📊 **Test Execution Summary**

```powershell
# Run all tests and generate report
$totalTests = 46
$passedTests = 0
$failedTests = 0

# Execute each category...
# (Test execution script here)

Write-Host "===== DELIVERY MODULE TEST REPORT ====="
Write-Host "Total Tests: $totalTests"
Write-Host "Passed: $passedTests ($([math]::Round($passedTests/$totalTests*100, 2))%)"
Write-Host "Failed: $failedTests ($([math]::Round($failedTests/$totalTests*100, 2))%)"
```

---

**[DELIVERY_QA_TEST_CASES_COMPLETE ✅]**

*For feature overview, see `01_FEATURE_OVERVIEW.md`. For technical details, see `02_TECHNICAL_GUIDE.md`.*

---

**Document Version**: 1.2  
**Last Updated**: March 2026  
**Module**: Delivery  
**Test Cases**: 50 (12 categories)  
**Status**: ✅ Complete

---

## Category 12: Live Tracking Bugs (Regression Suite)

### TC-DELIVERY-47: Rider location updates not stored — `req.user.userId` undefined

**Type:** Bug Regression  
**Feature area:** `POST /api/v1/rider/location` — `RiderLocationController`  
**Priority:** P0

**Preconditions:**
- Rider is authenticated (JWT issued)
- Order is in `ASSIGNED` status with `deliveryPartnerId` set to the rider's UUID

**Steps:**
1. Rider app calls `POST /api/v1/rider/location` with `{ orderId, latitude, longitude }`
2. Server receives request

**Expected result:** `200 OK`, coordinates stored in Redis key `rider_location:{orderId}`

**Actual result (before fix):** `404 Not Found — Order not found or not assigned to this rider`  
Root cause: `RiderLocationController` extracted `riderId = req.user.userId`. Because `JwtStrategy.validate()` returns the full `User` entity, `req.user.userId` is `undefined`. The service query `WHERE deliveryPartnerId = undefined` never matches any row.

**Fix applied:** Changed `req.user.userId` → `req.user.id` in `rider-location.controller.ts:40`

**Regression test:** `apps/chefooz-apis/src/modules/rider-location/rider-location.controller.spec.ts`  
**Status:** Fixed ✅

---

### TC-DELIVERY-48: Live/Static button hidden on `ASSIGNED` status

**Type:** Bug Regression / UI  
**Feature area:** `apps/chefooz-app/src/app/orders/[id]/track.tsx`  
**Priority:** P1

**Preconditions:**
- Customer has a paid order with a rider manually assigned by admin
- Order is in `ASSIGNED` delivery status (rider not yet picked up)

**Steps:**
1. Customer opens the order tracking screen
2. Customer expects to see the Live/Static map toggle button

**Expected result:** Button is visible but disabled, with label "Live" and 50% opacity, indicating tracking will be available after pickup.

**Actual result (before fix):** Button is completely hidden because the condition only showed it for `PICKED_UP` or `OUT_FOR_DELIVERY`. Customer has no feedback that tracking exists.

**Fix applied:** Expanded condition to include `ASSIGNED` status, rendered button with `disabled={true}` and reduced opacity.

**Regression test:** Manual UI check on 375px device  
**Status:** Fixed ✅

---

### TC-DELIVERY-49: Static map shows Delhi for orders outside Delhi

**Type:** Bug Regression / UI  
**Feature area:** `apps/chefooz-app/src/app/orders/[id]/track.tsx`  
**Priority:** P1

**Preconditions:**
- Order placed in Bengaluru or any non-Delhi city
- Chef or customer address coordinates absent from API response

**Steps:**
1. Customer opens tracking screen when `chefInfo.latitude` / `addressSnapshot.latitude` are missing from the response

**Expected result:** Map defaults to a generic India centre view; no polyline drawn.

**Actual result (before fix):** Map centred at `28.6100, 77.2000` (Delhi) — a hardcoded fallback. Polyline drawn between two Delhi points.

**Fix applied:** Removed all hardcoded Delhi fallbacks. Added `hasChefCoords` / `hasDestCoords` guards. `staticMapRegion` falls back to `{ latitude: 20.5937, longitude: 78.9629 }` (India centre). Polylines only rendered when both endpoints have real coordinates.

**Regression test:** Manual UI check with mock response missing coordinates  
**Status:** Fixed ✅

---

### TC-DELIVERY-50: No UI feedback between rider assignment and pickup (ASSIGNED state)

---

### TC-DELIVERY-51: Active delivery screen crashes on render (deprecated hook stubs)

**Type:** Bug Regression
**Feature area:** `apps/chefooz-app/src/app/delivery/active.tsx`
**Priority:** P0

**Preconditions:**
- Rider is assigned to an order
- Rider navigates to `/delivery/active`

**Steps:**
1. Rider accepts a delivery request and is routed to the active delivery screen
2. Screen attempts to render

**Expected result:** Active delivery screen loads with map and delivery details.
**Actual result (before fix):** Screen crashed immediately — `useActiveDelivery`, `useUpdateDeliveryState`, and `useCompleteDelivery` were deprecated stubs in `@chefooz-app/api-client` that threw `Error: deprecated` on every render.
**Fix applied:** Replaced all three deprecated hook calls with inline `useQuery` / `useMutation` calls using `apiClient` directly (interceptor handles auth). Added `refetchInterval: 15000` to the active delivery query.
**Regression test:** Manual — open `/delivery/active` as a rider; screen should load without crash.
**Status:** Fixed ✅

---

### TC-DELIVERY-52: Rider stuck on active delivery screen after order auto-cancel

**Type:** Bug Regression
**Feature area:** `apps/chefooz-app/src/app/delivery/active.tsx`, `apps/chefooz-apis/src/modules/delivery/delivery.service.ts`
**Priority:** P1

**Preconditions:**
- Rider has accepted an order and is on the active delivery screen
- Order is still in `out_for_delivery` or pre-pickup state
- Pickup timeout cron fires (`PICKUP_TIMEOUT` or `DELIVERY_TIMEOUT`)

**Steps:**
1. Rider accepts delivery and stays on `/delivery/active`
2. Timeout cron fires, marking `order.status = CANCELLED`
3. Within 15 seconds, the active delivery query refetches. Backend `getActiveDelivery` detects `order.status === CANCELLED`, clears `user.currentDeliveryId`, returns `null`.
4. Frontend `useEffect` detects `activeDelivery === null` with `hadDeliveryRef.current === true`
5. Alert shown: "This order was cancelled. You have been returned to the delivery dashboard."
6. Rider taps OK → routed to `/delivery/home`

**Expected result:** Rider sees alert explaining cancellation and is unblocked within 15 seconds.
**Actual result (before fix):** Rider was permanently stuck — `autoCancelOrder` did NOT clear `user.currentDeliveryId`, so subsequent `getActiveDelivery` calls kept returning the cancelled delivery. Screen had no polling and no cancellation check.
**Fix applied:**
- `autoCancelOrder` in `order.service.ts` now clears `user.currentDeliveryId = undefined` and `user.deliveryStatus = 'online'` via `userRepo.update()` when `order.deliveryPartnerId` is set.
- `getActiveDelivery` in `delivery.service.ts` also has a defensive guard: if the loaded delivery's order is CANCELLED/REFUNDED, it performs the same cleanup and returns `null`.
- `delivery/active.tsx` added `refetchInterval: 15000` and a `hadDeliveryRef` to detect mid-session cancellation and show an alert.
**Regression test:** Dev test: assign rider to order → trigger auto-cancel → verify rider is redirected within 15s with alert.
**Status:** Fixed ✅

**Type:** Bug Regression / UX  
**Feature area:** `apps/chefooz-app/src/app/orders/[id]/track.tsx`  
**Priority:** P2

**Preconditions:**
- Order is in `ASSIGNED` status, rider has not yet picked up

**Steps:**
1. Customer opens tracking screen
2. Customer sees Rider Card (name, phone)
3. Customer looks for status guidance

**Expected result:** A contextual card reads "Rider is on the way to restaurant — Live tracking starts once your rider picks up the order."

**Actual result (before fix):** No such card existed. Transition between "Finding rider..." and active tracking was silent.

**Fix applied:** Added ASSIGNED status banner card above Rider Card, visible only when `deliveryStatus === DeliveryStatus.ASSIGNED && riderInfo` is truthy.

**Status:** Fixed ✅

---

### TC-DELIVERY-53: Pickup timeout fires while chef is still preparing (false auto-cancel)

**Type:** Bug Regression / Automated
**Feature area:** Delivery timeout cron / Pickup policy
**Priority:** P0

**Preconditions:**
- `pickupTimeoutMin = 30` (dev config)
- Order placed at 6:00 PM
- Rider assigned and accepts at 6:01 PM
- Chef prep time = 45 minutes (food ready at 6:46 PM)

**Steps:**
1. Rider accepts delivery request at 6:01 PM
2. Chef starts preparing — order status moves to `PREPARING`
3. Cron runs at 6:32 PM (31 min since rider accepted)
4. Food is not yet ready (`order.readyAt` is null, status is `PREPARING`)

**Expected result:** Cron does NOT cancel the order. Rider correctly waits for food to be ready.

**Actual result (before fix):** Cron measured time from `deliveryAcceptedAt` (6:01 PM). At 6:32 PM, `minutesSinceAcceptance = 31 > 30` → order auto-cancelled while chef was still cooking.

**Fix applied:**
- `shouldAutoCancelForPickupDelay` now accepts `readyAt: Date | null` instead of `acceptedAt: Date`
- If `readyAt` is null → food not ready → `allowed: true` (no cancellation)
- Clock starts only after chef marks food READY
- `checkPickupTimeouts` cron now queries `status = READY` orders and passes `order.readyAt`

**Regression test:** `libs/domain/src/policies/delivery.policy.spec.ts` — `shouldAutoCancelForPickupDelay` describe block
**Status:** Fixed ✅

---

### TC-DELIVERY-54: Rider home page shows stale "1 active order" / busy banner after order auto-cancel

**Type:** Bug Regression / Manual
**Feature area:** Rider home page (`delivery/home.tsx`), `autoCancelOrder`, `RiderBusyBanner`
**Priority:** P0

**Preconditions:**
- Rider has an active order in `ASSIGNED` delivery status
- Order is auto-cancelled by timeout cron (`autoCancelOrder`)

**Steps:**
1. Auto-cancel fires (pickup or delivery timeout)
2. Rider opens home page

**Expected result:**
- Banner shows "Available for new deliveries" (green)
- "My Deliveries" Assigned section is empty
- Active order count = 0

**Actual result (before fix):**
- Banner shows "You're currently delivering an order — New assignment paused (1/2 active)" (yellow)
- "My Deliveries" Assigned section shows the cancelled order with "This order is cancelled" label
- Backend log: `Found 1 orders for rider ... status: ASSIGNED`

**Root cause:** `autoCancelOrder` set `order.status = CANCELLED` and cleared `user.currentDeliveryId`, but never set `order.deliveryStatus = 'CANCELLED'`. The rider orders query filters by `order.deliveryStatus = ASSIGNED`, and the `ACTIVE_DELIVERY_STATUSES` busy-state count includes `ASSIGNED` — both still matched the cancelled order.

**Fix applied:**
- In `autoCancelOrder` (`order.service.ts`): set `order.deliveryStatus = 'CANCELLED'` before saving — immediately removes order from `deliveryStatus = ASSIGNED` queries
- Inject `RiderBusyService` into `OrderService` and call `recalculateBusyState(riderId)` after clearing user state — immediately corrects `riderProfile.isBusy` and `riderProfile.activeOrderCount`

**Regression test:** n/a (integration-level; covered by manual QA)
**Status:** Fixed ✅

---

### TC-DEL-55: Online rider with `isBusy=true` (stale flag) is not assigned new orders

**Type:** Bug Regression / Manual
**Feature area:** `RiderAvailabilityService.updateHeartbeat`, `DeliveryAssignmentService.findEligibleRider`
**Priority:** P0

**Preconditions:**
- Rider is verified (`isActive=true`)
- Rider has no active orders (`activeOrderCount=0`, `maxConcurrentDeliveries=5`)
- `isBusy=true` stuck in DB — set by `recalculateBusyState` while heartbeat was stale or rider was offline
- Rider is actively sending heartbeats (`isOnline=true`, fresh `lastHeartbeatAt` in logs)

**Steps:**
1. Rider comes back online — heartbeats arrive every 30s
2. Customer places an order in the same city (Bengaluru)
3. Chef accepts the order
4. `AssignmentRetryService` cron fires → calls `findEligibleRider`
5. Query: `WHERE rider.isOnline = true AND rider.isActive = true AND rider.isBusy = false AND LOWER(rider.city) IN (...)`

**Expected result:** Rider matched and assigned. Log: `✅ Rider assigned to order`.

**Actual result (before fix):** `No eligible riders found for order <id> in Bengaluru`. Rider excluded because `isBusy = true` in DB even though `activeOrderCount = 0`.

**Root cause:**
`shouldRiderBeBusy` returns `isBusy: true` when called while heartbeat is stale (rule 2) or rider is offline (rule 3). `recalculateBusyState` persists this. When the rider comes back and sends heartbeats, `updateHeartbeat` restored `isOnline=true` and a fresh `lastHeartbeatAt` — but **never cleared `isBusy`**. The flag stayed permanently `true`.

**Fix applied:**
In `RiderAvailabilityService.updateHeartbeat` (`rider-availability.service.ts`): after setting `isOnline=true`, if `rider.isBusy && rider.activeOrderCount < rider.maxConcurrentDeliveries`, set `rider.isBusy = false`. Self-heals on every fresh heartbeat with no extra queries.

**Regression test:** n/a (integration-level)
**Status:** Fixed ✅
**Date:** 2026-03-29

---

### TC-DEL-56: Customer tracking screen shows static San Francisco map after payment

**Type:** Bug Regression / Manual
**Feature area:** `payment-status.tsx`, `orders/[orderId]/track.tsx`, `orders/[id]/track.tsx`
**Priority:** P0

**Preconditions:**
- Customer completes payment via UPI/Razorpay
- Payment status screen polls and receives `paymentStatus: 'paid'`

**Steps:**
1. Customer places order and completes payment
2. Payment status screen detects `paid` and navigates to the tracking screen
3. Customer is on the tracking screen, order transitions to `OUT_FOR_DELIVERY`

**Expected result:**
- Customer sees tracking screen with correct city map
- Rider marker appears and follows the rider's GPS in real time (5s polling)

**Actual result (before fix):**
- Customer is navigated to `orders/[orderId]/track.tsx` (old simulation screen)
- Map was centred on San Francisco (lat: 37.78825, lng: -122.4324) — hardcoded
- A fake WebSocket simulation randomly moved a marker around SF
- No real rider GPS data was ever shown regardless of actual delivery location

**Root cause:**
`payment-status.tsx` used `pathname: '/orders/[orderId]/track'` which explicitly
routed to the old `[orderId]/track.tsx` file (WebSocket simulation screen).
The real tracking screen `[id]/track.tsx` uses REST polling (`GET /v1/orders/:id/live-location`)
against Redis-backed real GPS data but was unreachable from the payment flow.

**Fix applied:**
1. `payment-status.tsx`: changed route to `pathname: '/orders/[id]/track'`, `params: { id: orderId }`
2. `orders/[orderId]/track.tsx`: replaced 630-line simulation with a redirect component
   that bounces to `[id]/track.tsx` for any remaining legacy navigations
3. `orders/[id]/track.tsx`: removed controlled `region` prop during live tracking
   (was fighting `animateToRegion` and locking map). Enabled `scrollEnabled` and
   `zoomEnabled` when live tracking is active (Zomato/Swiggy style interaction).
   `animateToRegion` now also fires when `showLiveTracking` first activates.

**Regression test:** Manual — place order, complete payment, verify map shows Bengaluru
**Status:** Fixed ✅
**Date:** 2026-03-29

---

### TC-DEL-57: Rider location not posted when state transitions to en_route after screen mount

**Type:** Bug Regression / Manual
**Feature area:** `apps/chefooz-app/src/app/delivery/active.tsx`
**Priority:** P0

**Preconditions:**
- Rider opens `delivery/active.tsx` while delivery state is `navigate_to_chef`
- Rider presses button to mark food as collected → state becomes `picked_up`
- Rider presses button again → state becomes `en_route`

**Steps:**
1. Rider mounts `active.tsx` at state `navigate_to_chef`
2. Location `useEffect` runs, creates `setInterval`; captures `state = 'navigate_to_chef'` from closure
3. Rider advances to `en_route` (button press)
4. `activeDelivery.state = 'en_route'` in React Query, BUT `activeDelivery?.id` unchanged
5. `useEffect` does NOT re-run; interval still has stale `state = 'navigate_to_chef'`
6. Interval condition `state !== 'picked_up' && state !== 'en_route'` → true → returns early
7. No GPS is ever posted to backend → customer tracking screen always shows `{lat:0, lng:0}`

**Expected result:** Rider location posted every 5s once state reaches `picked_up` or `en_route`

**Actual result (before fix):** Location never posted if state advanced while `active.tsx` was open

**Fix applied:**
- Added `activeDeliveryRef = useRef<ActiveDelivery | null>(null)` that stays synced via
  the existing `useEffect([activeDelivery])`.
- `setInterval` now reads `activeDeliveryRef.current` (not the stale closure capture).
- Also added `heading` and `speed` fields to location posts (from `expo-location` coords).

**Regression test:** n/a (integration-level)
**Status:** Fixed ✅
**Date:** 2026-03-29

---

### TC-DEL-58: Rider GPS tracking silently blocked by background location permission denial

**Type:** Bug Regression / Manual
**Feature area:** `apps/chefooz-app/src/services/rider-location.service.ts`, `rider/orders/[id].tsx`
**Priority:** P0

**Preconditions:**
- Rider is logged in and has been assigned an order
- Rider's device has location set to "While Using App" (foreground only) — the default iOS choice
- Rider opens `rider/orders/[id].tsx` and marks the order as `PICKED_UP`

**Steps:**
1. Rider opens order detail screen (`rider/orders/[id].tsx`)
2. Delivery status is `PICKED_UP` or `OUT_FOR_DELIVERY` → `useEffect` calls `riderLocationService.startTracking(orderId)`
3. `startTracking()` calls `requestPermissions()`
4. Foreground permission: granted ✅
5. Background permission: denied (user chose "While Using App") ⛔
6. `requestPermissions()` returns `false`
7. `startTracking()` logs `Location permissions not granted` and returns `false`
8. No GPS is EVER posted to Redis → `GET /v1/orders/:id/live-location` returns null
9. Customer tracking screen shows fixed map, rider marker never moves

**Expected result:** Rider location is tracked and posted every 6s while the order is active

**Actual result (before fix):**
Logs on rider device:
```
WARN  Background location permission denied
ERROR  Location permissions not granted
```
Customer tracking silently fails — map is static, no error shown to user.

**Root cause:**
`RiderLocationService.requestPermissions()` required *both* foreground AND background
permissions and returned `false` if either was denied. `watchPositionAsync()` only
requires foreground permission ("While Using App"). Requiring background = "Always Allow"
to even start tracking was unnecessarily strict and breaks on the default iOS permission choice.

**Fix applied:**
`rider-location.service.ts` — background permission is now requested opportunistically.
A warning is logged if denied, but `requestPermissions()` still returns `true` so
tracking proceeds with foreground-only GPS (works while app is open).

**Regression test:** n/a (integration-level)
**Status:** Fixed ✅
**Date:** 2026-03-30
