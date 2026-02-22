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
# or for local: $baseUrl = "http://localhost:3000/v1"
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

## 📊 **Test Execution Summary**

```powershell
# Run all tests and generate report
$totalTests = 35
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

**Document Version**: 1.0  
**Last Updated**: February 22, 2026  
**Module**: Delivery (Week 7 - Chef Fulfillment)  
**Test Cases**: 35 (10 categories)  
**Status**: ✅ Complete
