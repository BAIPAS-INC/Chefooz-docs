# 🧪 Delivery-ETA Module - QA Test Cases

## 📋 **Table of Contents**
- [Test Environment Setup](#test-environment-setup)
- [Test Data Prerequisites](#test-data-prerequisites)
- [Category 1: ETA Calculation](#category-1-eta-calculation)
- [Category 2: ETA Retrieval](#category-2-eta-retrieval)
- [Category 3: ETA Recalculation](#category-3-eta-recalculation)
- [Category 4: Fallback Logic](#category-4-fallback-logic)
- [Category 5: ETA Smoothing](#category-5-eta-smoothing)
- [Category 6: Lifecycle Hooks](#category-6-lifecycle-hooks)
- [Category 7: Rate Limiting](#category-7-rate-limiting)
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
# Customer token
$customerHeaders = @{
    "Authorization" = "Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
    "Content-Type" = "application/json"
}

# Admin token (for recalculation endpoint)
$adminHeaders = @{
    "Authorization" = "Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
    "Content-Type" = "application/json"
}
```

### **Test Users**

| Role | Email | User ID |
|------|-------|---------|
| **Customer** | customer1@test.com | `uuid-customer-001` |
| **Rider** | rider1@test.com | `uuid-rider-001` |
| **Admin** | admin@chefooz.com | `uuid-admin-001` |

### **Test Data Cleanup**
```sql
-- Run before each test suite
UPDATE orders SET estimated_delivery_minutes = NULL, eta_last_calculated_at = NULL, eta_source = 'none';
```

---

## 📊 **Test Data Prerequisites**

### **1. Create Test Order with Delivery**
```powershell
# Simplified: Use existing order or create via cart/checkout flow
$orderId = "existing-order-uuid-123"

# Ensure order status is PICKED_UP
$statusBody = @{
    status = "PICKED_UP"
} | ConvertTo-Json

Invoke-RestMethod -Uri "$baseUrl/orders/$orderId/status" -Method Put -Headers $adminHeaders -Body $statusBody
```

### **2. Set Rider Location (Simulate GPS)**
```powershell
# Mock rider location (via admin endpoint)
$locationBody = @{
    riderId = "uuid-rider-001"
    orderId = $orderId
    latitude = 12.9716
    longitude = 77.5946
} | ConvertTo-Json

Invoke-RestMethod -Uri "$baseUrl/internal/rider-location/update" -Method Post -Headers $adminHeaders -Body $locationBody
```

### **3. Configure Google Maps API Key**
```bash
# .env file
GOOGLE_MAPS_API_KEY=AIzaSy...
```

---

## 📝 **Category 1: ETA Calculation**

### **TC-EC-001: Calculate ETA Using Google Maps**

**Objective**: Verify ETA calculated when rider picks up order

**Preconditions**:
- Order exists with status PAID
- Rider assigned
- Rider location available
- Google Maps API key configured

**Test Steps**:
```powershell
# 1. Update order status to PICKED_UP (triggers ETA calculation)
$statusBody = @{
    status = "PICKED_UP"
} | ConvertTo-Json

Invoke-RestMethod -Uri "$baseUrl/orders/$orderId/status" -Method Put -Headers $adminHeaders -Body $statusBody

# 2. Wait 2 seconds for ETA calculation
Start-Sleep -Seconds 2

# 3. Get ETA
$response = Invoke-RestMethod -Uri "$baseUrl/orders/$orderId/eta" -Method Get -Headers $customerHeaders

Write-Host "ETA Minutes: $($response.data.etaMinutes)"
Write-Host "Distance: $($response.data.distanceKm) km"
Write-Host "Source: $($response.data.source)"
```

**Expected Result**:
```json
{
  "success": true,
  "message": "ETA retrieved successfully",
  "data": {
    "etaMinutes": 15,
    "distanceKm": 3.2,
    "lastUpdatedAt": "2026-02-22T11:30:00.000Z",
    "source": "maps"
  }
}
```

**Pass Criteria**:
- HTTP 200
- `etaMinutes` is a positive integer
- `source` = "maps"
- `lastUpdatedAt` is recent (within last 10 seconds)
- `distanceKm` matches approximate distance

---

### **TC-EC-002: ETA Includes 2-Minute Buffer**

**Objective**: Verify Google Maps ETA has 2-minute buffer added

**Test Steps**:
```powershell
# Compare Google Maps raw ETA vs. returned ETA
# (Requires access to Google Maps API response or logs)

# Check logs for:
# "Google Maps ETA: X minutes" (raw)
# vs. database value (raw + 2)
```

**Expected Behavior**:
- Database ETA = Google Maps duration ÷ 60 + 2 minutes

**Pass Criteria**:
- Buffer consistently added

---

### **TC-EC-003: ETA Not Calculated for Non-Picked-Up Orders**

**Objective**: Verify ETA only calculated after pickup

**Preconditions**:
- Order status: PREPARING (not PICKED_UP)

**Test Steps**:
```powershell
# Get ETA for order not yet picked up
$response = Invoke-RestMethod -Uri "$baseUrl/orders/$orderId/eta" -Method Get -Headers $customerHeaders

Write-Host "ETA: $($response.data.etaMinutes)"
Write-Host "Source: $($response.data.source)"
```

**Expected Result**:
```json
{
  "success": true,
  "message": "ETA retrieved successfully",
  "data": {
    "etaMinutes": null,
    "distanceKm": null,
    "lastUpdatedAt": null,
    "source": "none"
  }
}
```

**Pass Criteria**:
- `etaMinutes` = null
- `source` = "none"

---

### **TC-EC-004: ETA Calculation with Traffic Data**

**Objective**: Verify ETA uses `duration_in_traffic` when available

**Test Steps**:
```powershell
# Trigger ETA calculation during peak traffic hours (8-10 AM, 6-8 PM)
# Compare ETA with off-peak hours

# Peak hour ETA
$peakEta = (Invoke-RestMethod -Uri "$baseUrl/orders/$orderId/eta" -Method Get -Headers $customerHeaders).data.etaMinutes

# Off-peak hour ETA (same route)
$offPeakEta = ...

Write-Host "Peak ETA: $peakEta min"
Write-Host "Off-peak ETA: $offPeakEta min"
```

**Expected Behavior**:
- Peak hour ETA > Off-peak ETA (typically +20-40% longer)

**Pass Criteria**:
- Traffic awareness confirmed

---

## 📝 **Category 2: ETA Retrieval**

### **TC-ER-001: Get ETA for Order**

**Objective**: Verify customer can retrieve current ETA

**Preconditions**:
- Order has ETA calculated

**Test Steps**:
```powershell
$response = Invoke-RestMethod -Uri "$baseUrl/orders/$orderId/eta" -Method Get -Headers $customerHeaders

Write-Host "Success: $($response.success)"
Write-Host "ETA: $($response.data.etaMinutes) minutes"
```

**Expected Result**:
- HTTP 200
- Valid ETA data returned

**Pass Criteria**:
- Response time < 200ms

---

### **TC-ER-002: Get ETA Returns Distance**

**Objective**: Verify distance to customer calculated correctly

**Test Steps**:
```powershell
$response = Invoke-RestMethod -Uri "$baseUrl/orders/$orderId/eta" -Method Get -Headers $customerHeaders
$distance = $response.data.distanceKm

Write-Host "Distance: $distance km"
```

**Expected Behavior**:
- Distance calculated using Haversine formula
- Matches approximate distance (±10% tolerance)

**Pass Criteria**:
- `distanceKm` is non-null positive number

---

### **TC-ER-003: Get ETA for Order Not Found ❌**

**Objective**: Verify error handling for invalid order ID

**Test Steps**:
```powershell
$invalidOrderId = "00000000-0000-0000-0000-000000000000"

try {
    $response = Invoke-RestMethod -Uri "$baseUrl/orders/$invalidOrderId/eta" -Method Get -Headers $customerHeaders
} catch {
    $errorResponse = $_.ErrorDetails.Message | ConvertFrom-Json
    Write-Host "Error: $($errorResponse.message)"
}
```

**Expected Result**:
```json
{
  "success": false,
  "message": "Order not found",
  "errorCode": "ORDER_NOT_FOUND"
}
```

**Pass Criteria**:
- HTTP 404
- Error code: `ORDER_NOT_FOUND`

---

### **TC-ER-004: Get ETA Returns Last Updated Timestamp**

**Objective**: Verify `lastUpdatedAt` timestamp accurate

**Test Steps**:
```powershell
$response = Invoke-RestMethod -Uri "$baseUrl/orders/$orderId/eta" -Method Get -Headers $customerHeaders
$lastUpdated = [DateTime]::Parse($response.data.lastUpdatedAt)
$now = Get-Date

$ageSeconds = ($now - $lastUpdated).TotalSeconds
Write-Host "ETA Age: $ageSeconds seconds"
```

**Pass Criteria**:
- `lastUpdatedAt` within last 5 minutes

---

## 📝 **Category 3: ETA Recalculation**

### **TC-RC-001: Admin Force Recalculate ETA**

**Objective**: Verify admin can force ETA recalculation

**Preconditions**:
- Admin or system role token
- Order with existing ETA

**Test Steps**:
```powershell
$response = Invoke-RestMethod -Uri "$baseUrl/internal/orders/$orderId/recalculate-eta" -Method Post -Headers $adminHeaders

Write-Host "New ETA: $($response.data.newEtaMinutes) minutes"
Write-Host "Calculated At: $($response.data.calculatedAt)"
```

**Expected Result**:
```json
{
  "success": true,
  "message": "ETA recalculated successfully",
  "data": {
    "newEtaMinutes": 12,
    "calculatedAt": "2026-02-22T11:35:00.000Z"
  }
}
```

**Pass Criteria**:
- HTTP 200
- New ETA returned
- Database updated

---

### **TC-RC-002: Non-Admin Cannot Recalculate ETA ❌**

**Objective**: Verify recalculation endpoint restricted to admins

**Test Steps**:
```powershell
try {
    $response = Invoke-RestMethod -Uri "$baseUrl/internal/orders/$orderId/recalculate-eta" -Method Post -Headers $customerHeaders
} catch {
    $errorResponse = $_.ErrorDetails.Message | ConvertFrom-Json
    Write-Host "Error: $($errorResponse.message)"
}
```

**Expected Result**:
- HTTP 403 Forbidden

**Pass Criteria**:
- Access denied for non-admin users

---

### **TC-RC-003: Recalculation Bypasses Rate Limit**

**Objective**: Verify forced recalculation ignores 60-second interval

**Test Steps**:
```powershell
# Recalculate twice in quick succession
Invoke-RestMethod -Uri "$baseUrl/internal/orders/$orderId/recalculate-eta" -Method Post -Headers $adminHeaders
Start-Sleep -Seconds 5
$response2 = Invoke-RestMethod -Uri "$baseUrl/internal/orders/$orderId/recalculate-eta" -Method Post -Headers $adminHeaders

Write-Host "Second recalculation succeeded: $($response2.success)"
```

**Pass Criteria**:
- Both recalculations succeed
- No rate limit error

---

## 📝 **Category 4: Fallback Logic**

### **TC-FL-001: Fallback ETA When Google Maps Fails**

**Objective**: Verify distance-based fallback when API fails

**Test Steps**:
```powershell
# Temporarily disable Google Maps API (set invalid key)
# Or mock API timeout

$response = Invoke-RestMethod -Uri "$baseUrl/orders/$orderId/eta" -Method Get -Headers $customerHeaders

Write-Host "ETA: $($response.data.etaMinutes) min"
Write-Host "Source: $($response.data.source)"
```

**Expected Result**:
- `etaMinutes` calculated using distance ÷ 20 km/h + 3 min
- `source` = "fallback"

**Pass Criteria**:
- ETA still provided despite API failure
- Accuracy within ±5 minutes of actual

---

### **TC-FL-002: Fallback ETA Calculation Accuracy**

**Objective**: Verify fallback ETA formula accuracy

**Test Data**:
| Distance | Expected ETA | Actual ETA | Deviation |
|----------|--------------|------------|-----------|
| 2 km | 9 min | ? | ? |
| 5 km | 18 min | ? | ? |
| 10 km | 33 min | ? | ? |

**Formula**:
```
etaMinutes = ⌈(distance_km / 20) × 60⌉ + 3

Examples:
2 km → ⌈6⌉ + 3 = 9 min
5 km → ⌈15⌉ + 3 = 18 min
10 km → ⌈30⌉ + 3 = 33 min
```

**Pass Criteria**:
- Fallback ETA matches formula

---

### **TC-FL-003: Default ETA When No Rider Location**

**Objective**: Verify default ETA used when rider location unavailable

**Preconditions**:
- Order status: PICKED_UP
- Rider location not available (GPS disabled)

**Test Steps**:
```powershell
# Clear rider location
Invoke-RestMethod -Uri "$baseUrl/internal/rider-location/clear/$riderId" -Method Delete -Headers $adminHeaders

# Get ETA
$response = Invoke-RestMethod -Uri "$baseUrl/orders/$orderId/eta" -Method Get -Headers $customerHeaders

Write-Host "ETA: $($response.data.etaMinutes) min"
Write-Host "Source: $($response.data.source)"
```

**Expected Result**:
- PICKED_UP status → 25 minutes default
- OUT_FOR_DELIVERY status → 15 minutes default

**Pass Criteria**:
- Default ETA returned
- No error thrown

---

## 📝 **Category 5: ETA Smoothing**

### **TC-ES-001: ETA Changes Clamped to ±3 Minutes**

**Objective**: Verify ETA smoothing prevents jumpy UI

**Test Steps**:
```powershell
# Initial ETA: 25 minutes
$eta1 = (Invoke-RestMethod -Uri "$baseUrl/orders/$orderId/eta" -Method Get -Headers $customerHeaders).data.etaMinutes

# Simulate rider moving closer (Google Maps would return 10 min)
# Wait 60 seconds, trigger recalculation
Start-Sleep -Seconds 65
Invoke-RestMethod -Uri "$baseUrl/internal/orders/$orderId/recalculate-eta" -Method Post -Headers $adminHeaders

$eta2 = (Invoke-RestMethod -Uri "$baseUrl/orders/$orderId/eta" -Method Get -Headers $customerHeaders).data.etaMinutes

Write-Host "ETA 1: $eta1 min"
Write-Host "ETA 2: $eta2 min"
Write-Host "Change: $($eta1 - $eta2) min"
```

**Expected Behavior**:
```
Time:     0s    60s   120s  180s
Raw ETA:  25    10    10    10
Smoothed: 25    22    19    16
```

**Pass Criteria**:
- ETA change ≤ 3 minutes per update

---

### **TC-ES-002: First ETA Calculation No Smoothing**

**Objective**: Verify first ETA returned without smoothing

**Test Steps**:
```powershell
# Calculate ETA for order with no previous ETA
$response = Invoke-RestMethod -Uri "$baseUrl/internal/orders/$orderId/recalculate-eta" -Method Post -Headers $adminHeaders

Write-Host "First ETA: $($response.data.newEtaMinutes) min"
```

**Pass Criteria**:
- First ETA = raw calculated ETA (no clamping)

---

### **TC-ES-003: ETA Gradually Converges to Actual**

**Objective**: Verify smoothed ETA eventually reaches actual ETA

**Test Steps**:
```powershell
# Simulate 5 recalculations with constant raw ETA of 10 min
for ($i = 1; $i -le 5; $i++) {
    Start-Sleep -Seconds 65
    Invoke-RestMethod -Uri "$baseUrl/internal/orders/$orderId/recalculate-eta" -Method Post -Headers $adminHeaders
    $eta = (Invoke-RestMethod -Uri "$baseUrl/orders/$orderId/eta" -Method Get -Headers $customerHeaders).data.etaMinutes
    Write-Host "Update $i ETA: $eta min"
}
```

**Expected Progression**:
```
Update 1: 22 min (25 - 3)
Update 2: 19 min (22 - 3)
Update 3: 16 min (19 - 3)
Update 4: 13 min (16 - 3)
Update 5: 10 min (13 - 3, reaches actual)
```

**Pass Criteria**:
- Gradual convergence observed

---

## 📝 **Category 6: Lifecycle Hooks**

### **TC-LH-001: ETA Calculated on Status Change to PICKED_UP**

**Objective**: Verify ETA calculated when rider picks up order

**Test Steps**:
```powershell
# Update status to PICKED_UP
$statusBody = @{
    status = "PICKED_UP"
} | ConvertTo-Json

Invoke-RestMethod -Uri "$baseUrl/orders/$orderId/status" -Method Put -Headers $adminHeaders -Body $statusBody

# Wait for hook execution
Start-Sleep -Seconds 3

# Check ETA calculated
$response = Invoke-RestMethod -Uri "$baseUrl/orders/$orderId/eta" -Method Get -Headers $customerHeaders
Write-Host "ETA after pickup: $($response.data.etaMinutes) min"
```

**Pass Criteria**:
- ETA calculated automatically
- `source` = "maps" or "fallback"

---

### **TC-LH-002: ETA Cleared on Status Change to DELIVERED**

**Objective**: Verify ETA cleared when delivery completed

**Test Steps**:
```powershell
# Update status to DELIVERED
$statusBody = @{
    status = "DELIVERED"
} | ConvertTo-Json

Invoke-RestMethod -Uri "$baseUrl/orders/$orderId/status" -Method Put -Headers $adminHeaders -Body $statusBody

# Wait for hook execution
Start-Sleep -Seconds 2

# Check ETA cleared
$response = Invoke-RestMethod -Uri "$baseUrl/orders/$orderId/eta" -Method Get -Headers $customerHeaders
Write-Host "ETA after delivery: $($response.data.etaMinutes)"
Write-Host "Source: $($response.data.source)"
```

**Expected Result**:
- `etaMinutes` = null
- `source` = "none"
- `lastUpdatedAt` = null

**Pass Criteria**:
- ETA fields cleared

---

### **TC-LH-003: ETA Recalculated on Rider Location Update**

**Objective**: Verify ETA recalculated when rider moves

**Test Steps**:
```powershell
# Initial ETA
$eta1 = (Invoke-RestMethod -Uri "$baseUrl/orders/$orderId/eta" -Method Get -Headers $customerHeaders).data.etaMinutes

# Update rider location (simulate movement)
$locationBody = @{
    riderId = "uuid-rider-001"
    orderId = $orderId
    latitude = 12.9750
    longitude = 77.5965
} | ConvertTo-Json

Invoke-RestMethod -Uri "$baseUrl/internal/rider-location/update" -Method Post -Headers $adminHeaders -Body $locationBody

# Wait 65 seconds (rate limit)
Start-Sleep -Seconds 65

# Get new ETA
$eta2 = (Invoke-RestMethod -Uri "$baseUrl/orders/$orderId/eta" -Method Get -Headers $customerHeaders).data.etaMinutes

Write-Host "ETA before: $eta1 min"
Write-Host "ETA after location update: $eta2 min"
```

**Pass Criteria**:
- ETA updated after location change
- Change reflects new distance

---

### **TC-LH-004: ETA Recalculated on Rider Reassignment**

**Objective**: Verify ETA recalculated when order reassigned to new rider

**Test Steps**:
```powershell
# Reassign order to different rider
$reassignBody = @{
    newRiderId = "uuid-rider-002"
} | ConvertTo-Json

Invoke-RestMethod -Uri "$baseUrl/orders/$orderId/reassign" -Method Post -Headers $adminHeaders -Body $reassignBody

# Wait for hook execution
Start-Sleep -Seconds 3

# Check ETA recalculated
$response = Invoke-RestMethod -Uri "$baseUrl/orders/$orderId/eta" -Method Get -Headers $customerHeaders
Write-Host "ETA after reassignment: $($response.data.etaMinutes) min"
```

**Pass Criteria**:
- ETA recalculated with new rider location
- `lastUpdatedAt` updated

---

## 📝 **Category 7: Rate Limiting**

### **TC-RL-001: Minimum 60-Second Interval Between Calculations**

**Objective**: Verify ETA recalculation rate limited to 60 seconds

**Test Steps**:
```powershell
# Trigger location updates rapidly (simulate GPS updates every 10 seconds)
for ($i = 1; $i -le 6; $i++) {
    $locationBody = @{
        riderId = "uuid-rider-001"
        orderId = $orderId
        latitude = 12.9716 + ($i * 0.001)
        longitude = 77.5946 + ($i * 0.001)
    } | ConvertTo-Json
    
    Invoke-RestMethod -Uri "$baseUrl/internal/rider-location/update" -Method Post -Headers $adminHeaders -Body $locationBody
    Start-Sleep -Seconds 10
}

# Check logs for ETA recalculation frequency
# Expected: Only 1 recalculation (at 60-second mark)
```

**Pass Criteria**:
- Only 1 recalculation in 60 seconds
- Subsequent recalculations skipped

---

### **TC-RL-002: Rate Limit Not Applied to First Calculation**

**Objective**: Verify first ETA calculated immediately without rate limit

**Test Steps**:
```powershell
# Clear existing ETA
Invoke-RestMethod -Uri "$baseUrl/orders/$orderId/clear-eta" -Method Delete -Headers $adminHeaders

# Trigger immediate calculation
Invoke-RestMethod -Uri "$baseUrl/internal/orders/$orderId/recalculate-eta" -Method Post -Headers $adminHeaders

# Check calculated immediately
$response = Invoke-RestMethod -Uri "$baseUrl/orders/$orderId/eta" -Method Get -Headers $customerHeaders
Write-Host "First ETA calculated: $($response.data.etaMinutes -ne $null)"
```

**Pass Criteria**:
- First calculation immediate (no wait)

---

## ⚡ **Performance Tests**

### **PT-001: Get ETA Response Time**

**Objective**: Verify ETA retrieval under 200ms

**Test Steps**:
```powershell
$startTime = Get-Date
$response = Invoke-RestMethod -Uri "$baseUrl/orders/$orderId/eta" -Method Get -Headers $customerHeaders
$endTime = Get-Date
$duration = ($endTime - $startTime).TotalMilliseconds

Write-Host "Response Time: $duration ms"
```

**Pass Criteria**:
- Response time < 200ms

---

### **PT-002: Google Maps API Response Time**

**Objective**: Verify Google Maps API call completes within timeout

**Test Steps**:
```powershell
# Trigger ETA calculation
$startTime = Get-Date
Invoke-RestMethod -Uri "$baseUrl/internal/orders/$orderId/recalculate-eta" -Method Post -Headers $adminHeaders
$endTime = Get-Date
$duration = ($endTime - $startTime).TotalMilliseconds

Write-Host "ETA Calculation Duration: $duration ms"
```

**Pass Criteria**:
- Duration < 5000ms (5-second timeout)

---

### **PT-003: Fallback ETA Calculation Speed**

**Objective**: Verify fallback calculation faster than Google Maps

**Test Steps**:
```powershell
# Disable Google Maps API, measure fallback time
$startTime = Get-Date
Invoke-RestMethod -Uri "$baseUrl/internal/orders/$orderId/recalculate-eta" -Method Post -Headers $adminHeaders
$endTime = Get-Date
$duration = ($endTime - $startTime).TotalMilliseconds

Write-Host "Fallback Calculation Duration: $duration ms"
```

**Pass Criteria**:
- Duration < 100ms

---

## 🧩 **Edge Case Tests**

### **EC-001: ETA for Order at Same Location as Rider**

**Objective**: Verify ETA when rider and customer at same location

**Test Steps**:
```powershell
# Set rider location = customer location
$locationBody = @{
    riderId = "uuid-rider-001"
    orderId = $orderId
    latitude = 12.9800  # Same as customer
    longitude = 77.6000 # Same as customer
} | ConvertTo-Json

Invoke-RestMethod -Uri "$baseUrl/internal/rider-location/update" -Method Post -Headers $adminHeaders -Body $locationBody

# Calculate ETA
Invoke-RestMethod -Uri "$baseUrl/internal/orders/$orderId/recalculate-eta" -Method Post -Headers $adminHeaders

$response = Invoke-RestMethod -Uri "$baseUrl/orders/$orderId/eta" -Method Get -Headers $customerHeaders
Write-Host "ETA (same location): $($response.data.etaMinutes) min"
Write-Host "Distance: $($response.data.distanceKm) km"
```

**Expected Result**:
- ETA: ~2-5 minutes (minimum buffer)
- Distance: ~0 km

---

### **EC-002: ETA with Stale Rider Location**

**Objective**: Verify handling of old rider location data

**Test Steps**:
```powershell
# Set rider location with old timestamp (10 minutes ago)
# Mock via admin endpoint with timestamp

# Calculate ETA
$response = Invoke-RestMethod -Uri "$baseUrl/internal/orders/$orderId/recalculate-eta" -Method Post -Headers $adminHeaders

Write-Host "ETA calculation result: $($response.message)"
```

**Expected Behavior**:
- Stale location treated as unavailable
- Falls back to default ETA (25 min)

---

### **EC-003: Multiple Concurrent ETA Requests**

**Objective**: Verify thread safety for concurrent ETA retrievals

**Test Steps**:
```powershell
# Launch 10 concurrent GET /eta requests
$jobs = 1..10 | ForEach-Object {
    Start-Job -ScriptBlock {
        Invoke-RestMethod -Uri "$using:baseUrl/orders/$using:orderId/eta" -Method Get -Headers $using:customerHeaders
    }
}

# Wait for all jobs
Wait-Job $jobs
$results = Receive-Job $jobs

Write-Host "All requests succeeded: $($results | Where-Object { $_.success -eq $true } | Measure-Object | Select-Object -ExpandProperty Count)"
```

**Pass Criteria**:
- All 10 requests succeed
- Consistent ETA returned

---

## ✅ **Test Checklist**

### **ETA Calculation (4 tests)**
- [ ] TC-EC-001: Calculate ETA using Google Maps
- [ ] TC-EC-002: ETA includes 2-minute buffer
- [ ] TC-EC-003: ETA not calculated for non-picked-up orders
- [ ] TC-EC-004: ETA calculation with traffic data

### **ETA Retrieval (4 tests)**
- [ ] TC-ER-001: Get ETA for order
- [ ] TC-ER-002: Get ETA returns distance
- [ ] TC-ER-003: Get ETA for order not found
- [ ] TC-ER-004: Get ETA returns last updated timestamp

### **ETA Recalculation (3 tests)**
- [ ] TC-RC-001: Admin force recalculate ETA
- [ ] TC-RC-002: Non-admin cannot recalculate ETA
- [ ] TC-RC-003: Recalculation bypasses rate limit

### **Fallback Logic (3 tests)**
- [ ] TC-FL-001: Fallback ETA when Google Maps fails
- [ ] TC-FL-002: Fallback ETA calculation accuracy
- [ ] TC-FL-003: Default ETA when no rider location

### **ETA Smoothing (3 tests)**
- [ ] TC-ES-001: ETA changes clamped to ±3 minutes
- [ ] TC-ES-002: First ETA calculation no smoothing
- [ ] TC-ES-003: ETA gradually converges to actual

### **Lifecycle Hooks (4 tests)**
- [ ] TC-LH-001: ETA calculated on status change to PICKED_UP
- [ ] TC-LH-002: ETA cleared on status change to DELIVERED
- [ ] TC-LH-003: ETA recalculated on rider location update
- [ ] TC-LH-004: ETA recalculated on rider reassignment

### **Rate Limiting (2 tests)**
- [ ] TC-RL-001: Minimum 60-second interval between calculations
- [ ] TC-RL-002: Rate limit not applied to first calculation

### **Performance (3 tests)**
- [ ] PT-001: Get ETA response time < 200ms
- [ ] PT-002: Google Maps API response time < 5s
- [ ] PT-003: Fallback ETA calculation speed < 100ms

### **Edge Cases (3 tests)**
- [ ] EC-001: ETA for order at same location as rider
- [ ] EC-002: ETA with stale rider location
- [ ] EC-003: Multiple concurrent ETA requests

**Total**: 29 test cases

---

## 📊 **Test Execution Summary**

```powershell
# Run all tests and generate report
$totalTests = 29
$passedTests = 0
$failedTests = 0

# Execute each category...
# (Test execution script here)

Write-Host "===== DELIVERY-ETA MODULE TEST REPORT ====="
Write-Host "Total Tests: $totalTests"
Write-Host "Passed: $passedTests ($([math]::Round($passedTests/$totalTests*100, 2))%)"
Write-Host "Failed: $failedTests ($([math]::Round($failedTests/$totalTests*100, 2))%)"
```

---

**[DELIVERY-ETA_QA_TEST_CASES_COMPLETE ✅]**

*For feature overview, see `01_FEATURE_OVERVIEW.md`. For technical details, see `02_TECHNICAL_GUIDE.md`.*

---

**Document Version**: 1.0  
**Last Updated**: February 22, 2026  
**Module**: Delivery-ETA (Week 7 - Chef Fulfillment)  
**Test Cases**: 29 (7 categories)  
**Status**: ✅ Complete
