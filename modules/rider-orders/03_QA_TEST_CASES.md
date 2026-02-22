# 🧪 Rider Orders Module - QA Test Cases

## 📋 **Table of Contents**
- [Test Environment Setup](#test-environment-setup)
- [Test Data Prerequisites](#test-data-prerequisites)
- [Category 1: Order List & Filtering](#category-1-order-list--filtering)
- [Category 2: Order Detail](#category-2-order-detail)
- [Category 3: Order Acceptance](#category-3-order-acceptance)
- [Category 4: Order Rejection](#category-4-order-rejection)
- [Category 5: Delivery Status Updates](#category-5-delivery-status-updates)
- [Category 6: GPS Fraud Detection](#category-6-gps-fraud-detection)
- [Category 7: Earnings Automation](#category-7-earnings-automation)
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
# Rider token
$riderHeaders = @{
    "Authorization" = "Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
    "Content-Type" = "application/json"
}

# Different rider token (for negative tests)
$rider2Headers = @{
    "Authorization" = "Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
    "Content-Type" = "application/json"
}

# Admin token (for test data setup)
$adminHeaders = @{
    "Authorization" = "Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
    "Content-Type" = "application/json"
}
```

### **Test Users**

| Role | Email | User ID |
|------|-------|---------|
| **Rider 1** | rider1@test.com | `uuid-rider-001` |
| **Rider 2** | rider2@test.com | `uuid-rider-002` |
| **Customer** | customer1@test.com | `uuid-customer-001` |
| **Chef** | chef1@test.com | `uuid-chef-001` |

### **Test Data Cleanup**
```sql
-- Run before each test suite
UPDATE orders SET 
  delivery_partner_id = NULL,
  delivery_status = 'PENDING',
  assigned_at = NULL,
  delivery_accepted_at = NULL,
  delivery_rejected_at = NULL,
  picked_up_at = NULL,
  out_for_delivery_at = NULL,
  delivered_at = NULL
WHERE id IN ('test-order-001', 'test-order-002', 'test-order-003');

DELETE FROM rider_earnings WHERE rider_id = 'uuid-rider-001';
```

---

## 📊 **Test Data Prerequisites**

### **1. Create Test Order**
```powershell
# Use existing order or create via cart/checkout flow
$orderId = "test-order-001"

# Assign order to rider (admin endpoint)
$assignBody = @{
    riderId = "uuid-rider-001"
} | ConvertTo-Json

Invoke-RestMethod -Uri "$baseUrl/internal/delivery/assign/$orderId" -Method Post -Headers $adminHeaders -Body $assignBody
```

### **2. Set Chef Kitchen Location**
```sql
-- Ensure chef kitchen has GPS coordinates for fraud detection
UPDATE chef_kitchens 
SET latitude = 12.9716, longitude = 77.5946 
WHERE chef_id = 'uuid-chef-001';
```

### **3. Set Customer Delivery Address**
```sql
-- Ensure order has delivery address with GPS
UPDATE orders 
SET address_snapshot = '{
  "label": "Home",
  "address": "123 MG Road, Bangalore",
  "latitude": 12.9800,
  "longitude": 77.6000,
  "landmark": "Near Indiranagar Metro"
}'::jsonb
WHERE id = 'test-order-001';
```

---

## 📝 **Category 1: Order List & Filtering**

### **TC-OL-001: Get All Rider Orders**

**Objective**: Verify rider can retrieve all assigned orders

**Preconditions**:
- Rider authenticated
- 3 orders assigned to rider with different statuses

**Test Steps**:
```powershell
$response = Invoke-RestMethod -Uri "$baseUrl/rider/orders" -Method Get -Headers $riderHeaders

Write-Host "Total Orders: $($response.data.Count)"
$response.data | ForEach-Object {
    Write-Host "Order: $($_.id), Status: $($_.deliveryStatus)"
}
```

**Expected Result**:
```json
{
  "success": true,
  "message": "Rider orders retrieved successfully",
  "data": [
    {
      "id": "test-order-001",
      "deliveryStatus": "ASSIGNED",
      "customerName": "John Doe",
      "totalPaise": 45000,
      "itemCount": 3
    },
    {
      "id": "test-order-002",
      "deliveryStatus": "PICKED_UP",
      "customerName": "Jane Smith",
      "totalPaise": 32000,
      "itemCount": 2
    }
  ]
}
```

**Pass Criteria**:
- HTTP 200
- Returns only orders assigned to authenticated rider
- Sorted by `assignedAt` DESC (most recent first)

---

### **TC-OL-002: Filter Orders by Status**

**Objective**: Verify rider can filter orders by delivery status

**Test Steps**:
```powershell
# Filter by PICKED_UP status
$response = Invoke-RestMethod -Uri "$baseUrl/rider/orders?status=PICKED_UP" -Method Get -Headers $riderHeaders

Write-Host "Filtered Orders: $($response.data.Count)"
$response.data | ForEach-Object {
    Write-Host "Order: $($_.id), Status: $($_.deliveryStatus)"
}

# Verify all returned orders have PICKED_UP status
$allPickedUp = $response.data | Where-Object { $_.deliveryStatus -ne "PICKED_UP" } | Measure-Object | Select-Object -ExpandProperty Count
Write-Host "Non-PICKED_UP orders: $allPickedUp (should be 0)"
```

**Pass Criteria**:
- HTTP 200
- All returned orders have `deliveryStatus = "PICKED_UP"`
- Orders with other statuses excluded

---

### **TC-OL-003: Empty Order List**

**Objective**: Verify response when rider has no orders

**Preconditions**:
- Rider with no assigned orders

**Test Steps**:
```powershell
$response = Invoke-RestMethod -Uri "$baseUrl/rider/orders" -Method Get -Headers $rider2Headers

Write-Host "Total Orders: $($response.data.Count)"
```

**Expected Result**:
```json
{
  "success": true,
  "message": "Rider orders retrieved successfully",
  "data": []
}
```

**Pass Criteria**:
- HTTP 200
- Empty array returned

---

### **TC-OL-004: Order List Contains Customer Info**

**Objective**: Verify order list includes customer details for calling

**Test Steps**:
```powershell
$response = Invoke-RestMethod -Uri "$baseUrl/rider/orders" -Method Get -Headers $riderHeaders

$order = $response.data[0]
Write-Host "Customer Name: $($order.customerName)"
Write-Host "Customer Phone: $($order.customerPhone)"
Write-Host "Delivery Address: $($order.addressSnapshot.address)"
```

**Pass Criteria**:
- Each order includes `customerName`, `customerPhone`, `addressSnapshot`
- Phone number in format: `+919876543210`

---

## 📝 **Category 2: Order Detail**

### **TC-OD-001: Get Order Detail**

**Objective**: Verify rider can retrieve detailed order information

**Preconditions**:
- Order assigned to rider

**Test Steps**:
```powershell
$orderId = "test-order-001"
$response = Invoke-RestMethod -Uri "$baseUrl/rider/orders/$orderId" -Method Get -Headers $riderHeaders

Write-Host "Order ID: $($response.data.id)"
Write-Host "Items: $($response.data.items.Count)"
Write-Host "Customer: $($response.data.customerDetails.name)"
Write-Host "Phone: $($response.data.customerDetails.phone)"
Write-Host "Delivery Address: $($response.data.addressSnapshot.address)"
```

**Expected Result**:
```json
{
  "success": true,
  "message": "Order detail retrieved successfully",
  "data": {
    "id": "test-order-001",
    "items": [
      {
        "name": "Butter Chicken",
        "quantity": 2,
        "pricePaise": 25000
      }
    ],
    "customerDetails": {
      "id": "uuid-customer-001",
      "name": "John Doe",
      "phone": "+919876543210",
      "avatarUrl": "https://cdn.chefooz.com/avatars/user-001.jpg"
    },
    "addressSnapshot": {
      "label": "Home",
      "address": "123 MG Road, Bangalore",
      "latitude": 12.9800,
      "longitude": 77.6000
    },
    "deliveryStatus": "ASSIGNED",
    "totalPaise": 45000,
    "deliveryFeePaise": 5000
  }
}
```

**Pass Criteria**:
- HTTP 200
- Includes all order items
- Includes customer contact details
- Includes delivery address with GPS coordinates

---

### **TC-OD-002: Get Order Detail Not Assigned ❌**

**Objective**: Verify rider cannot view orders not assigned to them

**Test Steps**:
```powershell
# Order assigned to rider1, accessed by rider2
$orderId = "test-order-001"

try {
    $response = Invoke-RestMethod -Uri "$baseUrl/rider/orders/$orderId" -Method Get -Headers $rider2Headers
} catch {
    $errorResponse = $_.ErrorDetails.Message | ConvertFrom-Json
    Write-Host "Error: $($errorResponse.message)"
    Write-Host "Error Code: $($errorResponse.errorCode)"
}
```

**Expected Result**:
```json
{
  "success": false,
  "message": "Order not found or not assigned to you",
  "errorCode": "ORDER_NOT_FOUND"
}
```

**Pass Criteria**:
- HTTP 404
- Error code: `ORDER_NOT_FOUND`

---

### **TC-OD-003: Get Order Detail Invalid ID ❌**

**Objective**: Verify error handling for invalid order ID

**Test Steps**:
```powershell
$invalidOrderId = "00000000-0000-0000-0000-000000000000"

try {
    $response = Invoke-RestMethod -Uri "$baseUrl/rider/orders/$invalidOrderId" -Method Get -Headers $riderHeaders
} catch {
    $errorResponse = $_.ErrorDetails.Message | ConvertFrom-Json
    Write-Host "Error: $($errorResponse.message)"
}
```

**Pass Criteria**:
- HTTP 404
- Error message: "Order not found or not assigned to you"

---

## 📝 **Category 3: Order Acceptance**

### **TC-OA-001: Accept Assigned Order**

**Objective**: Verify rider can accept order within 30-second window

**Preconditions**:
- Order assigned to rider (status: ASSIGNED)
- Within 30-second acceptance window

**Test Steps**:
```powershell
$orderId = "test-order-001"
$response = Invoke-RestMethod -Uri "$baseUrl/rider/orders/$orderId/accept" -Method Patch -Headers $riderHeaders

Write-Host "Success: $($response.success)"
Write-Host "Message: $($response.message)"
Write-Host "Accepted At: $($response.data.deliveryAcceptedAt)"
```

**Expected Result**:
```json
{
  "success": true,
  "message": "Order accepted successfully",
  "data": {
    "id": "test-order-001",
    "deliveryStatus": "ASSIGNED",
    "deliveryAcceptedAt": "2026-02-22T10:30:15.000Z"
  }
}
```

**Pass Criteria**:
- HTTP 200
- `deliveryAcceptedAt` timestamp set
- 30-second timeout cancelled

---

### **TC-OA-002: Accept Already Accepted Order (Idempotent)**

**Objective**: Verify accepting already accepted order returns success

**Preconditions**:
- Order already accepted by rider

**Test Steps**:
```powershell
$orderId = "test-order-001"

# Accept twice
Invoke-RestMethod -Uri "$baseUrl/rider/orders/$orderId/accept" -Method Patch -Headers $riderHeaders
$response2 = Invoke-RestMethod -Uri "$baseUrl/rider/orders/$orderId/accept" -Method Patch -Headers $riderHeaders

Write-Host "Second acceptance success: $($response2.success)"
```

**Pass Criteria**:
- HTTP 200
- Returns existing acceptance (no error)

---

### **TC-OA-003: Accept Order Not Assigned ❌**

**Objective**: Verify rider cannot accept order assigned to another rider

**Test Steps**:
```powershell
# Order assigned to rider1, acceptance attempted by rider2
$orderId = "test-order-001"

try {
    $response = Invoke-RestMethod -Uri "$baseUrl/rider/orders/$orderId/accept" -Method Patch -Headers $rider2Headers
} catch {
    $errorResponse = $_.ErrorDetails.Message | ConvertFrom-Json
    Write-Host "Error: $($errorResponse.message)"
    Write-Host "Error Code: $($errorResponse.errorCode)"
}
```

**Expected Result**:
```json
{
  "success": false,
  "message": "This order is not assigned to you",
  "errorCode": "NOT_AUTHORIZED"
}
```

**Pass Criteria**:
- HTTP 403 Forbidden
- Error code: `NOT_AUTHORIZED`

---

### **TC-OA-004: Accept Order Already Picked Up ❌**

**Objective**: Verify rider cannot accept order already picked up

**Preconditions**:
- Order status: PICKED_UP

**Test Steps**:
```powershell
$orderId = "test-order-002"

try {
    $response = Invoke-RestMethod -Uri "$baseUrl/rider/orders/$orderId/accept" -Method Patch -Headers $riderHeaders
} catch {
    $errorResponse = $_.ErrorDetails.Message | ConvertFrom-Json
    Write-Host "Error: $($errorResponse.message)"
}
```

**Expected Result**:
```json
{
  "success": false,
  "message": "Cannot accept order in PICKED_UP status. Order must be ASSIGNED.",
  "errorCode": "INVALID_ACCEPTANCE"
}
```

**Pass Criteria**:
- HTTP 400 Bad Request
- Error indicates invalid status for acceptance

---

## 📝 **Category 4: Order Rejection**

### **TC-OR-001: Reject Assigned Order**

**Objective**: Verify rider can reject order with reason

**Preconditions**:
- Order assigned to rider (status: ASSIGNED)

**Test Steps**:
```powershell
$orderId = "test-order-001"
$rejectBody = @{
    reason = "Too far from current location"
} | ConvertTo-Json

$response = Invoke-RestMethod -Uri "$baseUrl/rider/orders/$orderId/reject" -Method Patch -Headers $riderHeaders -Body $rejectBody

Write-Host "Success: $($response.success)"
Write-Host "Message: $($response.message)"

# Wait 5 seconds, check order reassigned
Start-Sleep -Seconds 5
$orderCheck = Invoke-RestMethod -Uri "$baseUrl/internal/orders/$orderId" -Method Get -Headers $adminHeaders
Write-Host "New Rider: $($orderCheck.data.deliveryPartnerId)"
```

**Expected Result**:
```json
{
  "success": true,
  "message": "Order rejected successfully, reassignment in progress"
}
```

**Pass Criteria**:
- HTTP 200
- Order unassigned from rider (`deliveryPartnerId = null`)
- Automatic reassignment triggered (new rider assigned within 30s)
- Reason logged in database

---

### **TC-OR-002: Reject Order Without Reason**

**Objective**: Verify rejection works without reason (optional field)

**Test Steps**:
```powershell
$orderId = "test-order-001"
$rejectBody = @{} | ConvertTo-Json

$response = Invoke-RestMethod -Uri "$baseUrl/rider/orders/$orderId/reject" -Method Patch -Headers $riderHeaders -Body $rejectBody

Write-Host "Success: $($response.success)"
```

**Pass Criteria**:
- HTTP 200
- Rejection succeeds without reason

---

### **TC-OR-003: Reject Order Already Picked Up ❌**

**Objective**: Verify rider cannot reject order already picked up

**Preconditions**:
- Order status: PICKED_UP

**Test Steps**:
```powershell
$orderId = "test-order-002"
$rejectBody = @{
    reason = "Cannot deliver"
} | ConvertTo-Json

try {
    $response = Invoke-RestMethod -Uri "$baseUrl/rider/orders/$orderId/reject" -Method Patch -Headers $riderHeaders -Body $rejectBody
} catch {
    $errorResponse = $_.ErrorDetails.Message | ConvertFrom-Json
    Write-Host "Error: $($errorResponse.message)"
}
```

**Expected Result**:
```json
{
  "success": false,
  "message": "Cannot reject order in PICKED_UP status. Order must be ASSIGNED.",
  "errorCode": "INVALID_REJECTION"
}
```

**Pass Criteria**:
- HTTP 400 Bad Request
- Error indicates invalid status for rejection

---

### **TC-OR-004: Reject Order Already Rejected ❌**

**Objective**: Verify rider cannot reject order twice

**Preconditions**:
- Order already rejected

**Test Steps**:
```powershell
$orderId = "test-order-001"
$rejectBody = @{
    reason = "Too far"
} | ConvertTo-Json

# Reject twice
Invoke-RestMethod -Uri "$baseUrl/rider/orders/$orderId/reject" -Method Patch -Headers $riderHeaders -Body $rejectBody

try {
    $response2 = Invoke-RestMethod -Uri "$baseUrl/rider/orders/$orderId/reject" -Method Patch -Headers $riderHeaders -Body $rejectBody
} catch {
    $errorResponse = $_.ErrorDetails.Message | ConvertFrom-Json
    Write-Host "Error: $($errorResponse.message)"
}
```

**Expected Result**:
```json
{
  "success": false,
  "message": "Order already rejected",
  "errorCode": "ALREADY_REJECTED"
}
```

**Pass Criteria**:
- HTTP 400 Bad Request
- Error code: `ALREADY_REJECTED`

---

## 📝 **Category 5: Delivery Status Updates**

### **TC-DS-001: Update Status to PICKED_UP**

**Objective**: Verify rider can mark order as picked up with GPS

**Preconditions**:
- Order status: ASSIGNED
- Order accepted by rider
- Rider at chef kitchen location (within 100m)

**Test Steps**:
```powershell
$orderId = "test-order-001"
$statusBody = @{
    status = "PICKED_UP"
    lat = 12.9716
    lng = 77.5946
} | ConvertTo-Json

$response = Invoke-RestMethod -Uri "$baseUrl/rider/orders/$orderId/status" -Method Patch -Headers $riderHeaders -Body $statusBody

Write-Host "Success: $($response.success)"
Write-Host "Message: $($response.message)"
Write-Host "Picked Up At: $($response.data.pickedUpAt)"
```

**Expected Result**:
```json
{
  "success": true,
  "message": "Order status updated to PICKED_UP",
  "data": {
    "id": "test-order-001",
    "deliveryStatus": "PICKED_UP",
    "pickedUpAt": "2026-02-22T11:00:00.000Z"
  }
}
```

**Side Effects**:
- `pickedUpAt` timestamp set
- "On the way" notification sent to customer
- ETA calculation triggered

**Pass Criteria**:
- HTTP 200
- Status updated to PICKED_UP
- Timestamp set

---

### **TC-DS-002: Update Status to OUT_FOR_DELIVERY**

**Objective**: Verify rider can mark order out for delivery

**Preconditions**:
- Order status: PICKED_UP

**Test Steps**:
```powershell
$orderId = "test-order-001"
$statusBody = @{
    status = "OUT_FOR_DELIVERY"
} | ConvertTo-Json

$response = Invoke-RestMethod -Uri "$baseUrl/rider/orders/$orderId/status" -Method Patch -Headers $riderHeaders -Body $statusBody

Write-Host "Success: $($response.success)"
Write-Host "Out For Delivery At: $($response.data.outForDeliveryAt)"
```

**Expected Result**:
- HTTP 200
- Status updated to OUT_FOR_DELIVERY
- `outForDeliveryAt` timestamp set

---

### **TC-DS-003: Update Status to DELIVERED**

**Objective**: Verify rider can mark order as delivered with GPS

**Preconditions**:
- Order status: OUT_FOR_DELIVERY
- Rider at customer location (within 100m)

**Test Steps**:
```powershell
$orderId = "test-order-001"
$statusBody = @{
    status = "DELIVERED"
    lat = 12.9800
    lng = 77.6000
} | ConvertTo-Json

$response = Invoke-RestMethod -Uri "$baseUrl/rider/orders/$orderId/status" -Method Patch -Headers $riderHeaders -Body $statusBody

Write-Host "Success: $($response.success)"
Write-Host "Delivered At: $($response.data.deliveredAt)"

# Check earnings created
Start-Sleep -Seconds 2
$earnings = Invoke-RestMethod -Uri "$baseUrl/rider/earnings" -Method Get -Headers $riderHeaders
Write-Host "Earnings Count: $($earnings.data.Count)"
```

**Expected Result**:
```json
{
  "success": true,
  "message": "Order status updated to DELIVERED",
  "data": {
    "id": "test-order-001",
    "deliveryStatus": "DELIVERED",
    "deliveredAt": "2026-02-22T11:30:00.000Z"
  }
}
```

**Side Effects**:
- `deliveredAt` timestamp set
- "Delivered" notification sent to customer
- Rider location cleared
- Active deliveries count decremented
- Earning record created (₹50)

**Pass Criteria**:
- HTTP 200
- Status updated to DELIVERED
- Earning created with correct amount

---

### **TC-DS-004: Invalid Status Transition ❌**

**Objective**: Verify invalid status transitions rejected

**Test Steps**:
```powershell
# Try to go from DELIVERED back to OUT_FOR_DELIVERY
$orderId = "test-order-001"
$statusBody = @{
    status = "OUT_FOR_DELIVERY"
} | ConvertTo-Json

try {
    $response = Invoke-RestMethod -Uri "$baseUrl/rider/orders/$orderId/status" -Method Patch -Headers $riderHeaders -Body $statusBody
} catch {
    $errorResponse = $_.ErrorDetails.Message | ConvertFrom-Json
    Write-Host "Error: $($errorResponse.message)"
    Write-Host "Error Code: $($errorResponse.errorCode)"
}
```

**Expected Result**:
```json
{
  "success": false,
  "message": "Invalid delivery status transition from DELIVERED to OUT_FOR_DELIVERY",
  "errorCode": "INVALID_STATUS_TRANSITION"
}
```

**Pass Criteria**:
- HTTP 400 Bad Request
- Error code: `INVALID_STATUS_TRANSITION`

---

### **TC-DS-005: Skip Status Transition ❌**

**Objective**: Verify cannot skip status (ASSIGNED → DELIVERED directly)

**Test Steps**:
```powershell
$orderId = "test-order-001"
$statusBody = @{
    status = "DELIVERED"
    lat = 12.9800
    lng = 77.6000
} | ConvertTo-Json

try {
    $response = Invoke-RestMethod -Uri "$baseUrl/rider/orders/$orderId/status" -Method Patch -Headers $riderHeaders -Body $statusBody
} catch {
    $errorResponse = $_.ErrorDetails.Message | ConvertFrom-Json
    Write-Host "Error: $($errorResponse.message)"
}
```

**Pass Criteria**:
- HTTP 400 Bad Request
- Error indicates invalid transition

---

## 📝 **Category 6: GPS Fraud Detection**

### **TC-GF-001: Fake Pickup Detection**

**Objective**: Verify fake pickup blocked when GPS too far from restaurant

**Preconditions**:
- Order status: ASSIGNED
- Rider at location far from chef kitchen (>100m)

**Test Steps**:
```powershell
$orderId = "test-order-001"

# Chef kitchen at (12.9716, 77.5946)
# Rider location 10 km away (12.9000, 77.5000)
$statusBody = @{
    status = "PICKED_UP"
    lat = 12.9000
    lng = 77.5000
} | ConvertTo-Json

try {
    $response = Invoke-RestMethod -Uri "$baseUrl/rider/orders/$orderId/status" -Method Patch -Headers $riderHeaders -Body $statusBody
} catch {
    $errorResponse = $_.ErrorDetails.Message | ConvertFrom-Json
    Write-Host "Error: $($errorResponse.message)"
    Write-Host "Error Code: $($errorResponse.errorCode)"
}
```

**Expected Result**:
```json
{
  "success": false,
  "message": "Pickup confirmation failed validation",
  "errorCode": "FAKE_PICKUP_DETECTED"
}
```

**Pass Criteria**:
- HTTP 400 Bad Request
- Error code: `FAKE_PICKUP_DETECTED`
- Order remains in ASSIGNED status

---

### **TC-GF-002: Legitimate Pickup**

**Objective**: Verify legitimate pickup succeeds when GPS within 100m

**Test Steps**:
```powershell
$orderId = "test-order-001"

# Chef kitchen at (12.9716, 77.5946)
# Rider location 50m away (12.9720, 77.5950)
$statusBody = @{
    status = "PICKED_UP"
    lat = 12.9720
    lng = 77.5950
} | ConvertTo-Json

$response = Invoke-RestMethod -Uri "$baseUrl/rider/orders/$orderId/status" -Method Patch -Headers $riderHeaders -Body $statusBody

Write-Host "Success: $($response.success)"
Write-Host "Status: $($response.data.deliveryStatus)"
```

**Pass Criteria**:
- HTTP 200
- Status updated to PICKED_UP

---

### **TC-GF-003: Fake Delivery Detection**

**Objective**: Verify fake delivery blocked when GPS too far from customer

**Preconditions**:
- Order status: OUT_FOR_DELIVERY
- Rider at location far from customer address (>100m)

**Test Steps**:
```powershell
$orderId = "test-order-001"

# Customer address at (12.9800, 77.6000)
# Rider location 9 km away (12.9000, 77.5000)
$statusBody = @{
    status = "DELIVERED"
    lat = 12.9000
    lng = 77.5000
} | ConvertTo-Json

try {
    $response = Invoke-RestMethod -Uri "$baseUrl/rider/orders/$orderId/status" -Method Patch -Headers $riderHeaders -Body $statusBody
} catch {
    $errorResponse = $_.ErrorDetails.Message | ConvertFrom-Json
    Write-Host "Error: $($errorResponse.message)"
    Write-Host "Error Code: $($errorResponse.errorCode)"
}
```

**Expected Result**:
```json
{
  "success": false,
  "message": "Delivery confirmation failed validation",
  "errorCode": "FAKE_DELIVERY_DETECTED"
}
```

**Pass Criteria**:
- HTTP 400 Bad Request
- Error code: `FAKE_DELIVERY_DETECTED`
- Order remains in OUT_FOR_DELIVERY status

---

### **TC-GF-004: Legitimate Delivery**

**Objective**: Verify legitimate delivery succeeds when GPS within 100m

**Test Steps**:
```powershell
$orderId = "test-order-001"

# Customer address at (12.9800, 77.6000)
# Rider location 80m away (12.9805, 77.6005)
$statusBody = @{
    status = "DELIVERED"
    lat = 12.9805
    lng = 77.6005
} | ConvertTo-Json

$response = Invoke-RestMethod -Uri "$baseUrl/rider/orders/$orderId/status" -Method Patch -Headers $riderHeaders -Body $statusBody

Write-Host "Success: $($response.success)"
Write-Host "Status: $($response.data.deliveryStatus)"
```

**Pass Criteria**:
- HTTP 200
- Status updated to DELIVERED
- Earning created

---

### **TC-GF-005: Pickup Without GPS (Phase 3.6)**

**Objective**: Verify pickup succeeds without GPS during Phase 3.6 rollout

**Test Steps**:
```powershell
$orderId = "test-order-001"

# No GPS coordinates provided
$statusBody = @{
    status = "PICKED_UP"
} | ConvertTo-Json

$response = Invoke-RestMethod -Uri "$baseUrl/rider/orders/$orderId/status" -Method Patch -Headers $riderHeaders -Body $statusBody

Write-Host "Success: $($response.success)"
Write-Host "Status: $($response.data.deliveryStatus)"
```

**Pass Criteria**:
- HTTP 200 (allowed during Phase 3.6)
- Status updated to PICKED_UP
- Warning logged in backend (not enforced yet)

---

## 📝 **Category 7: Earnings Automation**

### **TC-EA-001: Earning Created on Delivery**

**Objective**: Verify earning record automatically created when order delivered

**Preconditions**:
- Order status: OUT_FOR_DELIVERY
- Delivery fee: ₹50 (5000 paise)

**Test Steps**:
```powershell
$orderId = "test-order-001"

# Mark as delivered
$statusBody = @{
    status = "DELIVERED"
    lat = 12.9805
    lng = 77.6005
} | ConvertTo-Json

Invoke-RestMethod -Uri "$baseUrl/rider/orders/$orderId/status" -Method Patch -Headers $riderHeaders -Body $statusBody

# Wait for earning creation
Start-Sleep -Seconds 2

# Check earnings
$earnings = Invoke-RestMethod -Uri "$baseUrl/rider/earnings" -Method Get -Headers $riderHeaders

Write-Host "Total Earnings: $($earnings.data.Count)"
$todayEarning = $earnings.data | Where-Object { $_.orderId -eq $orderId }
Write-Host "Order Earning: ₹$($todayEarning.amountPaise / 100)"
```

**Expected Earning Record**:
```json
{
  "id": "uuid-earning-001",
  "riderId": "uuid-rider-001",
  "orderId": "test-order-001",
  "amountPaise": 5000,
  "type": "DELIVERY_FEE",
  "status": "PENDING_PAYOUT",
  "completedAt": "2026-02-22T11:30:00.000Z"
}
```

**Pass Criteria**:
- Earning created with correct amount (₹50)
- Type: `DELIVERY_FEE`
- Status: `PENDING_PAYOUT`

---

### **TC-EA-002: No Earning for Non-Delivered Orders**

**Objective**: Verify earning not created until order delivered

**Test Steps**:
```powershell
$orderId = "test-order-002"

# Mark as PICKED_UP (not delivered)
$statusBody = @{
    status = "PICKED_UP"
    lat = 12.9720
    lng = 77.5950
} | ConvertTo-Json

Invoke-RestMethod -Uri "$baseUrl/rider/orders/$orderId/status" -Method Patch -Headers $riderHeaders -Body $statusBody

# Check earnings (should not include this order)
$earnings = Invoke-RestMethod -Uri "$baseUrl/rider/earnings" -Method Get -Headers $riderHeaders
$pickedUpEarning = $earnings.data | Where-Object { $_.orderId -eq $orderId }

Write-Host "Earning found for PICKED_UP order: $($pickedUpEarning -ne $null)"
```

**Pass Criteria**:
- No earning created for PICKED_UP order
- Earning only created on DELIVERED status

---

### **TC-EA-003: Multiple Order Earnings Accumulate**

**Objective**: Verify earnings from multiple orders accumulate correctly

**Test Steps**:
```powershell
# Deliver 3 orders with different fees
$orders = @(
    @{ id = "test-order-001"; fee = 5000 },
    @{ id = "test-order-002"; fee = 6000 },
    @{ id = "test-order-003"; fee = 4500 }
)

foreach ($order in $orders) {
    $statusBody = @{
        status = "DELIVERED"
        lat = 12.9805
        lng = 77.6005
    } | ConvertTo-Json
    
    Invoke-RestMethod -Uri "$baseUrl/rider/orders/$($order.id)/status" -Method Patch -Headers $riderHeaders -Body $statusBody
    Start-Sleep -Seconds 1
}

# Check total earnings
$earnings = Invoke-RestMethod -Uri "$baseUrl/rider/earnings" -Method Get -Headers $riderHeaders
$total = ($earnings.data | Measure-Object -Property amountPaise -Sum).Sum

Write-Host "Total Earnings: ₹$($total / 100)"
Write-Host "Expected: ₹155 (50+60+45)"
```

**Pass Criteria**:
- 3 earning records created
- Total: ₹155 (50+60+45)

---

## ⚡ **Performance Tests**

### **PT-001: Order List Response Time**

**Objective**: Verify order list retrieval under 150ms

**Test Steps**:
```powershell
$startTime = Get-Date
$response = Invoke-RestMethod -Uri "$baseUrl/rider/orders" -Method Get -Headers $riderHeaders
$endTime = Get-Date
$duration = ($endTime - $startTime).TotalMilliseconds

Write-Host "Response Time: $duration ms"
```

**Pass Criteria**:
- Response time < 150ms

---

### **PT-002: Order Detail Response Time**

**Objective**: Verify order detail retrieval under 120ms

**Test Steps**:
```powershell
$orderId = "test-order-001"
$startTime = Get-Date
$response = Invoke-RestMethod -Uri "$baseUrl/rider/orders/$orderId" -Method Get -Headers $riderHeaders
$endTime = Get-Date
$duration = ($endTime - $startTime).TotalMilliseconds

Write-Host "Response Time: $duration ms"
```

**Pass Criteria**:
- Response time < 120ms

---

### **PT-003: Status Update Response Time**

**Objective**: Verify status update completes within timeout

**Test Steps**:
```powershell
$orderId = "test-order-001"
$statusBody = @{
    status = "DELIVERED"
    lat = 12.9805
    lng = 77.6005
} | ConvertTo-Json

$startTime = Get-Date
$response = Invoke-RestMethod -Uri "$baseUrl/rider/orders/$orderId/status" -Method Patch -Headers $riderHeaders -Body $statusBody
$endTime = Get-Date
$duration = ($endTime - $startTime).TotalMilliseconds

Write-Host "Response Time: $duration ms"
```

**Pass Criteria**:
- Response time < 500ms (includes GPS check + earning creation)

---

## 🧩 **Edge Case Tests**

### **EC-001: Concurrent Status Updates**

**Objective**: Verify thread safety for concurrent status updates

**Test Steps**:
```powershell
$orderId = "test-order-001"
$statusBody = @{
    status = "PICKED_UP"
    lat = 12.9720
    lng = 77.5950
} | ConvertTo-Json

# Launch 3 concurrent status updates
$jobs = 1..3 | ForEach-Object {
    Start-Job -ScriptBlock {
        Invoke-RestMethod -Uri "$using:baseUrl/rider/orders/$using:orderId/status" -Method Patch -Headers $using:riderHeaders -Body $using:statusBody
    }
}

Wait-Job $jobs
$results = Receive-Job $jobs

Write-Host "Successful updates: $($results | Where-Object { $_.success -eq $true } | Measure-Object | Select-Object -ExpandProperty Count)"
```

**Pass Criteria**:
- Only 1 update succeeds
- Others return error (already PICKED_UP) or success (idempotent)
- Database consistent (no duplicate timestamps)

---

### **EC-002: Order With Missing Customer Info**

**Objective**: Verify graceful handling when customer info unavailable

**Preconditions**:
- Order with null user relation

**Test Steps**:
```powershell
$orderId = "test-order-missing-user"
$response = Invoke-RestMethod -Uri "$baseUrl/rider/orders/$orderId" -Method Get -Headers $riderHeaders

Write-Host "Customer Name: $($response.data.customerDetails.name)"
```

**Pass Criteria**:
- HTTP 200
- Customer name defaults to "Customer"
- Phone null (not error)

---

### **EC-003: Rider Accepts Multiple Orders Simultaneously**

**Objective**: Verify rider can accept multiple orders at once

**Test Steps**:
```powershell
$orders = @("test-order-001", "test-order-002", "test-order-003")

foreach ($orderId in $orders) {
    $response = Invoke-RestMethod -Uri "$baseUrl/rider/orders/$orderId/accept" -Method Patch -Headers $riderHeaders
    Write-Host "Order $orderId accepted: $($response.success)"
}

# Check active deliveries count
$orderList = Invoke-RestMethod -Uri "$baseUrl/rider/orders?status=ASSIGNED" -Method Get -Headers $riderHeaders
Write-Host "Active assigned orders: $($orderList.data.Count)"
```

**Pass Criteria**:
- All 3 orders accepted successfully
- Rider can manage multiple concurrent orders

---

## ✅ **Test Checklist**

### **Order List & Filtering (4 tests)**
- [ ] TC-OL-001: Get all rider orders
- [ ] TC-OL-002: Filter orders by status
- [ ] TC-OL-003: Empty order list
- [ ] TC-OL-004: Order list contains customer info

### **Order Detail (3 tests)**
- [ ] TC-OD-001: Get order detail
- [ ] TC-OD-002: Get order detail not assigned (403)
- [ ] TC-OD-003: Get order detail invalid ID (404)

### **Order Acceptance (4 tests)**
- [ ] TC-OA-001: Accept assigned order
- [ ] TC-OA-002: Accept already accepted order (idempotent)
- [ ] TC-OA-003: Accept order not assigned (403)
- [ ] TC-OA-004: Accept order already picked up (400)

### **Order Rejection (4 tests)**
- [ ] TC-OR-001: Reject assigned order
- [ ] TC-OR-002: Reject order without reason
- [ ] TC-OR-003: Reject order already picked up (400)
- [ ] TC-OR-004: Reject order already rejected (400)

### **Delivery Status Updates (5 tests)**
- [ ] TC-DS-001: Update status to PICKED_UP
- [ ] TC-DS-002: Update status to OUT_FOR_DELIVERY
- [ ] TC-DS-003: Update status to DELIVERED
- [ ] TC-DS-004: Invalid status transition (400)
- [ ] TC-DS-005: Skip status transition (400)

### **GPS Fraud Detection (5 tests)**
- [ ] TC-GF-001: Fake pickup detection (400)
- [ ] TC-GF-002: Legitimate pickup
- [ ] TC-GF-003: Fake delivery detection (400)
- [ ] TC-GF-004: Legitimate delivery
- [ ] TC-GF-005: Pickup without GPS (Phase 3.6)

### **Earnings Automation (3 tests)**
- [ ] TC-EA-001: Earning created on delivery
- [ ] TC-EA-002: No earning for non-delivered orders
- [ ] TC-EA-003: Multiple order earnings accumulate

### **Performance (3 tests)**
- [ ] PT-001: Order list response time < 150ms
- [ ] PT-002: Order detail response time < 120ms
- [ ] PT-003: Status update response time < 500ms

### **Edge Cases (3 tests)**
- [ ] EC-001: Concurrent status updates
- [ ] EC-002: Order with missing customer info
- [ ] EC-003: Rider accepts multiple orders simultaneously

**Total**: 34 test cases

---

## 📊 **Test Execution Summary**

```powershell
# Run all tests and generate report
$totalTests = 34
$passedTests = 0
$failedTests = 0

# Execute each category...
# (Test execution script here)

Write-Host "===== RIDER-ORDERS MODULE TEST REPORT ====="
Write-Host "Total Tests: $totalTests"
Write-Host "Passed: $passedTests ($([math]::Round($passedTests/$totalTests*100, 2))%)"
Write-Host "Failed: $failedTests ($([math]::Round($failedTests/$totalTests*100, 2))%)"
```

---

**[RIDER-ORDERS_QA_TEST_CASES_COMPLETE ✅]**

*For feature overview, see `01_FEATURE_OVERVIEW.md`. For technical details, see `02_TECHNICAL_GUIDE.md`.*

---

**Document Version**: 1.0  
**Last Updated**: February 22, 2026  
**Module**: Rider-Orders (Week 7 - Chef Fulfillment)  
**Test Cases**: 34 (9 categories)  
**Status**: ✅ Complete
