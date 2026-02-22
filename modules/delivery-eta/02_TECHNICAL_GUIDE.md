# 🕒 Delivery-ETA Module - Technical Guide

## 📋 **Table of Contents**
- [Architecture Overview](#architecture-overview)
- [Module Structure](#module-structure)
- [API Endpoints](#api-endpoints)
- [Service Layer](#service-layer)
- [DTOs](#dtos)
- [Database Schema](#database-schema)
- [Integration Patterns](#integration-patterns)
- [Algorithms](#algorithms)
- [Error Handling](#error-handling)
- [Testing Strategy](#testing-strategy)

---

## 🏗️ **Architecture Overview**

### **System Components**

```
┌──────────────────────────────────────────────────────────────┐
│                DELIVERY-ETA MODULE ECOSYSTEM                  │
└──────────────────────────────────────────────────────────────┘
                                │
        ┌───────────────────────┼───────────────────────┐
        │                       │                       │
        ▼                       ▼                       ▼
┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐
│  DeliveryEta     │   │  DeliveryEta     │   │  EtaHooks        │
│  Controller      │   │  Service         │   │  Service         │
│                  │   │                  │   │                  │
│ GET /eta         │   │ Main Logic:      │   │ Lifecycle:       │
│ POST /recalc     │   │ - calculateEta() │   │ - onStatusChange │
└──────────────────┘   │ - getOrderEta()  │   │ - onLocation     │
        │              │                  │   │ - onReassignment │
        │              │ Helpers:         │   └──────────────────┘
        │              │ - smoothEta()    │            │
        │              │ - updateEta()    │            │
        │              │ - fallbackEta()  │            │
        └──────────────┴──────────────────┴────────────┘
                                │
        ┌───────────────────────┼───────────────────────┐
        │                       │                       │
        ▼                       ▼                       ▼
┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐
│  Google Maps     │   │  RiderLocation   │   │  Order Entity    │
│  Directions API  │   │  Service         │   │  (TypeORM)       │
│                  │   │                  │   │                  │
│ - Route calc     │   │ - getLocation()  │   │ Fields:          │
│ - Traffic data   │   │ - staleness chk  │   │ - etaMinutes     │
│ - Duration       │   │                  │   │ - etaCalculated  │
└──────────────────┘   └──────────────────┘   │ - etaSource      │
                                              └──────────────────┘
```

### **Request Flow: ETA Calculation**

```mermaid
sequenceDiagram
    participant C as Customer
    participant API as ETA Controller
    participant ES as ETA Service
    participant RL as Rider Location
    participant GM as Google Maps
    participant DB as Order DB

    C->>API: GET /orders/{id}/eta
    API->>ES: getOrderEta(orderId)
    ES->>DB: Find order
    DB-->>ES: Order data
    
    alt Has rider location
        ES->>RL: getLocationForOrder(orderId)
        RL-->>ES: { lat, lng, timestamp }
        ES->>GM: GET /directions (rider → customer)
        GM-->>ES: duration_in_traffic: 780s
        
        ES->>ES: Convert to minutes: 13 + 2 buffer = 15 min
        ES->>ES: smoothEta(15) → 15 min (first calc)
        ES->>DB: Update order (etaMinutes: 15, source: "maps")
    else No rider location
        ES->>ES: calculateFallbackEta() → 25 min
        ES->>DB: Update order (etaMinutes: 25, source: "fallback")
    end
    
    ES-->>API: { etaMinutes: 15, distanceKm: 3.2, source: "maps" }
    API-->>C: 200 OK with ETA data
```

---

## 📁 **Module Structure**

```
apps/chefooz-apis/src/modules/delivery-eta/
├── delivery-eta.controller.ts       # 2 HTTP endpoints (customer + admin)
├── delivery-eta.service.ts          # Core ETA calculation logic (381 lines)
├── eta-hooks.service.ts             # Lifecycle hooks (status changes, location updates)
├── delivery-eta.module.ts           # Module definition + dependencies
└── dto/
    └── delivery-eta.dto.ts          # 4 DTOs (GetOrderEtaResponse, RecalculateEtaResponse, etc.)
```

### **Module Definition**

```typescript
// delivery-eta.module.ts
import { Module, forwardRef } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { HttpModule } from '@nestjs/axios';
import { Order } from '../../database/entities/order.entity';
import { DeliveryEtaService } from './delivery-eta.service';
import { DeliveryEtaController } from './delivery-eta.controller';
import { EtaHooksService } from './eta-hooks.service';
import { RiderLocationModule } from '../rider-location/rider-location.module';

@Module({
  imports: [
    TypeOrmModule.forFeature([Order]),
    HttpModule.register({
      timeout: 5000,      // 5-second timeout for Google Maps API
      maxRedirects: 3,
    }),
    forwardRef(() => RiderLocationModule), // Circular dependency
  ],
  controllers: [DeliveryEtaController],
  providers: [DeliveryEtaService, EtaHooksService],
  exports: [DeliveryEtaService, EtaHooksService],
})
export class DeliveryEtaModule {}
```

**Key Dependencies**:
- **TypeORM Order Entity**: Store ETA fields
- **HttpModule**: Google Maps API calls
- **RiderLocationModule**: Fetch rider's current GPS coordinates (forwardRef to avoid circular dependency)

---

## 🔌 **API Endpoints**

**Base URL**: `/v1`  
**Authentication**: JWT Bearer token  
**Content-Type**: `application/json`

---

### **Endpoint 1: Get Order ETA (Customer)**

**URL**: `GET /v1/orders/:id/eta`

**Description**: Retrieve current ETA for an order

**Authentication**: Required (JWT)

**Rate Limit**: None (lightweight read operation)

**Request Headers**:
```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**URL Parameters**:
- `id` (required): Order UUID

**Response** (200 OK):
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

**Response Fields**:
- `etaMinutes`: Estimated delivery time in minutes from now (null if not available)
- `distanceKm`: Current distance from rider to customer in kilometers (null if rider location unavailable)
- `lastUpdatedAt`: When ETA was last calculated (ISO string, null if never calculated)
- `source`: ETA calculation method
  - `"maps"`: Google Maps Directions API
  - `"fallback"`: Distance-based calculation
  - `"none"`: No ETA calculated yet
  - `"error"`: Error occurred

**Error Response** (404 - Order Not Found):
```json
{
  "success": false,
  "message": "Order not found",
  "errorCode": "ORDER_NOT_FOUND"
}
```

**Error Response** (500 - ETA Fetch Failed):
```json
{
  "success": false,
  "message": "Failed to retrieve ETA",
  "errorCode": "ETA_FETCH_FAILED"
}
```

**Business Logic**:
```typescript
async getOrderEta(orderId: string): Promise<GetOrderEtaResponseDto> {
  try {
    const etaData = await this.deliveryEtaService.getOrderEta(orderId);

    return {
      success: true,
      message: 'ETA retrieved successfully',
      data: {
        etaMinutes: etaData.etaMinutes,
        distanceKm: etaData.distanceKm,
        lastUpdatedAt: etaData.lastUpdatedAt?.toISOString() || null,
        source: etaData.source,
      },
    };
  } catch (error) {
    this.logger.error(`Failed to get ETA for order ${orderId}:`, error);
    return {
      success: false,
      message: 'Failed to retrieve ETA',
      errorCode: 'ETA_FETCH_FAILED',
    };
  }
}
```

**Performance**: ~100-200ms (database read + distance calculation if rider location available)

---

### **Endpoint 2: Recalculate ETA (Admin/Internal)**

**URL**: `POST /v1/internal/orders/:id/recalculate-eta`

**Description**: Force ETA recalculation for an order (bypass rate limiting)

**Authentication**: Required (JWT with admin or system role)

**Rate Limit**: None

**Request Headers**:
```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**URL Parameters**:
- `id` (required): Order UUID

**Response** (200 OK):
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

**Error Response** (400 - ETA Calculation Failed):
```json
{
  "success": false,
  "message": "ETA calculation failed or not applicable",
  "errorCode": "ETA_CALCULATION_FAILED"
}
```

**Business Logic**:
```typescript
async recalculateEta(orderId: string): Promise<RecalculateEtaResponseDto> {
  try {
    const newEta = await this.deliveryEtaService.calculateEta(orderId);

    if (newEta === null) {
      return {
        success: false,
        message: 'ETA calculation failed or not applicable',
        errorCode: 'ETA_CALCULATION_FAILED',
      };
    }

    return {
      success: true,
      message: 'ETA recalculated successfully',
      data: {
        newEtaMinutes: newEta,
        calculatedAt: new Date().toISOString(),
      },
    };
  } catch (error) {
    this.logger.error(`Failed to recalculate ETA for order ${orderId}:`, error);
    return {
      success: false,
      message: 'ETA recalculation failed',
      errorCode: 'ETA_RECALC_FAILED',
    };
  }
}
```

**Use Cases**:
- Admin manually triggers recalculation
- Testing ETA accuracy
- Debugging ETA issues

---

## 🔧 **Service Layer**

### **DeliveryEtaService Methods**

#### **Method 1: calculateEta**

**Signature**:
```typescript
async calculateEta(orderId: string): Promise<number | null>
```

**Purpose**: Calculate ETA for an order using Google Maps or fallback

**Steps**:
1. **Fetch order** from database
2. **Validate order status** (must be PICKED_UP or OUT_FOR_DELIVERY)
3. **Get rider location** from RiderLocationService
4. **Extract delivery coordinates** from order.addressSnapshot
5. **Calculate ETA using Google Maps** (if available)
6. **Smooth ETA** to prevent jumpy UI
7. **Update order** with new ETA
8. **Return ETA** in minutes

**Implementation**:
```typescript
async calculateEta(orderId: string): Promise<number | null> {
  try {
    const order = await this.orderRepository.findOne({
      where: { id: orderId },
      relations: ['user'],
    });

    if (!order) {
      this.logger.warn(`Order ${orderId} not found for ETA calculation`);
      return null;
    }

    // ETA only applies after pickup
    if (order.deliveryStatus !== 'PICKED_UP' && order.deliveryStatus !== 'OUT_FOR_DELIVERY') {
      this.logger.debug(`Order ${orderId} not picked up yet, skipping ETA`);
      return null;
    }

    // Get rider current location
    const riderLocation = await this.riderLocationService.getLocationForOrder(orderId);

    if (!riderLocation) {
      this.logger.warn(`No rider location for order ${orderId}`);
      return this.calculateFallbackEta(order);
    }

    // Get customer delivery address
    const deliveryCoords = this.extractDeliveryCoordinates(order);
    if (!deliveryCoords) {
      this.logger.warn(`No delivery coordinates for order ${orderId}`);
      return this.calculateFallbackEta(order);
    }

    // Calculate ETA using Google Maps
    const mapsEta = await this.calculateGoogleMapsEta(
      riderLocation.lat,
      riderLocation.lng,
      deliveryCoords.lat,
      deliveryCoords.lng,
    );

    if (mapsEta) {
      const smoothedEta = this.smoothEta(orderId, mapsEta);
      await this.updateOrderEta(orderId, smoothedEta, 'maps');
      return smoothedEta;
    }

    // Fallback to distance-based calculation
    const fallbackEta = this.calculateDistanceBasedEta(
      riderLocation.lat,
      riderLocation.lng,
      deliveryCoords.lat,
      deliveryCoords.lng,
    );

    await this.updateOrderEta(orderId, fallbackEta, 'fallback');
    return fallbackEta;
  } catch (error) {
    this.logger.error(`ETA calculation failed for order ${orderId}:`, error);
    return null;
  }
}
```

**Return Value**:
- `number`: ETA in minutes
- `null`: ETA calculation not applicable or failed

---

#### **Method 2: getOrderEta**

**Signature**:
```typescript
async getOrderEta(orderId: string): Promise<{
  etaMinutes: number | null;
  distanceKm: number | null;
  lastUpdatedAt: Date | null;
  source: string;
}>
```

**Purpose**: Retrieve current ETA data for an order (read-only, no calculation)

**Implementation**:
```typescript
async getOrderEta(orderId: string): Promise<{
  etaMinutes: number | null;
  distanceKm: number | null;
  lastUpdatedAt: Date | null;
  source: string;
}> {
  try {
    const order = await this.orderRepository.findOne({
      where: { id: orderId },
    });

    if (!order) {
      return {
        etaMinutes: null,
        distanceKm: null,
        lastUpdatedAt: null,
        source: 'none',
      };
    }

    // Calculate distance if we have rider location and delivery address
    let distanceKm: number | null = null;
    if (order.deliveryPartnerId) {
      const riderLocation = await this.riderLocationService.getLocationForOrder(orderId);
      const deliveryCoords = this.extractDeliveryCoordinates(order);

      if (riderLocation && deliveryCoords) {
        distanceKm = this.calculateDistance(
          riderLocation.lat,
          riderLocation.lng,
          deliveryCoords.lat,
          deliveryCoords.lng,
        );
      }
    }

    return {
      etaMinutes: order.estimatedDeliveryMinutes ?? null,
      distanceKm,
      lastUpdatedAt: order.etaLastCalculatedAt ?? null,
      source: order.etaSource || 'none',
    };
  } catch (error) {
    this.logger.error(`Failed to get ETA for order ${orderId}:`, error);
    return {
      etaMinutes: null,
      distanceKm: null,
      lastUpdatedAt: null,
      source: 'error',
    };
  }
}
```

**Performance**: ~80-150ms (database read + optional distance calculation)

---

#### **Method 3: recalculateEtaIfNeeded**

**Signature**:
```typescript
async recalculateEtaIfNeeded(orderId: string): Promise<boolean>
```

**Purpose**: Recalculate ETA only if rate limit allows (60-second minimum interval)

**Implementation**:
```typescript
async recalculateEtaIfNeeded(orderId: string): Promise<boolean> {
  try {
    const order = await this.orderRepository.findOne({
      where: { id: orderId },
    });

    if (!order || !order.deliveryPartnerId) {
      return false;
    }

    // Check if recalculation is needed
    const shouldRecalculate = this.shouldRecalculateEta(order);
    if (!shouldRecalculate) {
      return false;
    }

    const newEta = await this.calculateEta(orderId);
    return newEta !== null;
  } catch (error) {
    this.logger.error(`ETA recalculation check failed for order ${orderId}:`, error);
    return false;
  }
}

// Helper: Check if 60 seconds elapsed since last calculation
private shouldRecalculateEta(order: Order): boolean {
  // Always calculate if no ETA exists
  if (!order.estimatedDeliveryMinutes || !order.etaLastCalculatedAt) {
    return true;
  }

  // Recalculate if last calculation was more than 60 seconds ago
  const timeSinceLastCalculation = Date.now() - order.etaLastCalculatedAt.getTime();
  return timeSinceLastCalculation > 60000; // 60 seconds
}
```

**Return Value**:
- `true`: ETA recalculated
- `false`: Skipped (rate limited or not applicable)

---

#### **Method 4: calculateGoogleMapsEta (Private)**

**Signature**:
```typescript
private async calculateGoogleMapsEta(
  originLat: number,
  originLng: number,
  destLat: number,
  destLng: number,
): Promise<number | null>
```

**Purpose**: Call Google Maps Directions API to get traffic-aware ETA

**Google Maps API Request**:
```http
GET https://maps.googleapis.com/maps/api/directions/json
  ?origin=12.9716,77.5946
  &destination=12.9800,77.6000
  &mode=driving
  &traffic_model=best_guess
  &departure_time=now
  &key=AIza...
```

**Implementation**:
```typescript
private async calculateGoogleMapsEta(
  originLat: number,
  originLng: number,
  destLat: number,
  destLng: number,
): Promise<number | null> {
  try {
    const apiKey = this.configService.get<string>('GOOGLE_MAPS_API_KEY');
    if (!apiKey) {
      this.logger.warn('Google Maps API key not configured');
      return null;
    }

    const url = 'https://maps.googleapis.com/maps/api/directions/json';
    const params = {
      origin: `${originLat},${originLng}`,
      destination: `${destLat},${destLng}`,
      mode: 'driving',
      traffic_model: 'best_guess',
      departure_time: 'now',
      key: apiKey,
    };

    const response = await firstValueFrom(
      this.httpService.get(url, { params, timeout: 5000 }),
    );

    if (response.data.status !== 'OK' || !response.data.routes?.length) {
      this.logger.warn('Google Maps API returned no routes');
      return null;
    }

    const route = response.data.routes[0];
    const leg = route.legs[0];

    // Use duration_in_traffic if available, otherwise duration
    const durationSeconds = leg.duration_in_traffic?.value || leg.duration?.value;
    if (!durationSeconds) {
      return null;
    }

    // Convert to minutes and add small buffer
    const etaMinutes = Math.ceil(durationSeconds / 60) + 2; // +2 min buffer

    this.logger.debug(`Google Maps ETA: ${etaMinutes} minutes`);
    return etaMinutes;
  } catch (error) {
    this.logger.error('Google Maps API call failed:', error);
    return null;
  }
}
```

**Error Handling**:
- API key not configured → return `null` (fallback to distance-based)
- Network timeout (5s) → return `null`
- No routes found → return `null`
- Invalid response → return `null`

**Google Maps Response Example**:
```json
{
  "status": "OK",
  "routes": [
    {
      "legs": [
        {
          "distance": { "value": 3200, "text": "3.2 km" },
          "duration": { "value": 780, "text": "13 mins" },
          "duration_in_traffic": { "value": 900, "text": "15 mins" }
        }
      ]
    }
  ]
}
```

**Performance**: ~500-1500ms (external API call)

---

#### **Method 5: calculateDistanceBasedEta (Private)**

**Signature**:
```typescript
private calculateDistanceBasedEta(
  originLat: number,
  originLng: number,
  destLat: number,
  destLng: number,
): number
```

**Purpose**: Calculate fallback ETA using Haversine distance + average speed

**Implementation**:
```typescript
private calculateDistanceBasedEta(
  originLat: number,
  originLng: number,
  destLat: number,
  destLng: number,
): number {
  const distanceKm = this.calculateDistance(originLat, originLng, destLat, destLng);
  const etaMinutes = Math.ceil((distanceKm / this.avgSpeedKmh) * 60) + 3;
  
  this.logger.debug(`Fallback ETA: ${etaMinutes} minutes for ${distanceKm}km`);
  return etaMinutes;
}
```

**Constants**:
- `avgSpeedKmh = 20`: Average delivery speed (validated from historical data)
- `+3 minutes`: Buffer for non-straight roads

**Formula**:
```
etaMinutes = ⌈(distance_km / 20 km/h) × 60 min/h⌉ + 3 min

Example:
distance = 3.2 km
etaMinutes = ⌈(3.2 / 20) × 60⌉ + 3
           = ⌈9.6⌉ + 3
           = 10 + 3
           = 13 minutes
```

---

#### **Method 6: smoothEta (Private)**

**Signature**:
```typescript
private smoothEta(orderId: string, newEta: number): number
```

**Purpose**: Smooth ETA changes to prevent jumpy UI (max ±3 minutes per update)

**Implementation**:
```typescript
private smoothEta(orderId: string, newEta: number): number {
  const cached = this.etaCache.get(orderId);
  
  if (!cached) {
    // First calculation
    this.etaCache.set(orderId, { eta: newEta, calculatedAt: new Date() });
    return newEta;
  }

  // Apply smoothing
  const maxChange = this.maxEtaChangeMinutes; // 3 minutes
  const previousEta = cached.eta;
  
  const smoothedEta = Math.max(
    Math.min(newEta, previousEta + maxChange),
    previousEta - maxChange,
  );

  this.etaCache.set(orderId, { eta: smoothedEta, calculatedAt: new Date() });
  
  this.logger.debug(
    `ETA smoothing: ${previousEta} → ${newEta} → ${smoothedEta} minutes`,
  );
  
  return smoothedEta;
}
```

**Cache Structure**:
```typescript
etaCache: Map<string, { eta: number; calculatedAt: Date }> = new Map();
```

**Smoothing Logic**:
```typescript
// Clamp new ETA to [previousEta - 3, previousEta + 3]
smoothedEta = max(
  min(newEta, previousEta + 3),
  previousEta - 3
)

// Examples:
previousEta = 25, newEta = 18 → smoothedEta = 22  (25 - 3)
previousEta = 22, newEta = 18 → smoothedEta = 19  (22 - 3)
previousEta = 19, newEta = 18 → smoothedEta = 18  (exact match)
```

---

#### **Method 7: calculateDistance (Private)**

**Signature**:
```typescript
private calculateDistance(lat1: number, lng1: number, lat2: number, lng2: number): number
```

**Purpose**: Calculate straight-line distance using Haversine formula

**Implementation**:
```typescript
private calculateDistance(lat1: number, lng1: number, lat2: number, lng2: number): number {
  const R = 6371; // Earth's radius in km
  const dLat = this.toRadians(lat2 - lat1);
  const dLng = this.toRadians(lng2 - lng1);
  
  const a = 
    Math.sin(dLat / 2) * Math.sin(dLat / 2) +
    Math.cos(this.toRadians(lat1)) * Math.cos(this.toRadians(lat2)) *
    Math.sin(dLng / 2) * Math.sin(dLng / 2);
  
  const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
  return R * c;
}

private toRadians(degrees: number): number {
  return degrees * (Math.PI / 180);
}
```

**Haversine Formula**:
```
a = sin²(Δlat/2) + cos(lat1) × cos(lat2) × sin²(Δlng/2)
c = 2 × atan2(√a, √(1-a))
distance = R × c

where:
  R = 6371 km (Earth's radius)
  Δlat = lat2 - lat1 (in radians)
  Δlng = lng2 - lng1 (in radians)
```

**Example**:
```
lat1 = 12.9716, lng1 = 77.5946  (Rider)
lat2 = 12.9800, lng2 = 77.6000  (Customer)

distance ≈ 1.03 km
```

---

### **EtaHooksService Methods**

#### **Method 1: onDeliveryStatusChange**

**Signature**:
```typescript
async onDeliveryStatusChange(orderId: string, newStatus: string): Promise<void>
```

**Purpose**: Trigger ETA calculation when delivery status changes

**Triggers**:
1. **Status: PICKED_UP** → Calculate initial ETA
2. **Status: OUT_FOR_DELIVERY** → Continue ETA calculation
3. **Status: DELIVERED** → Clear ETA
4. **Status: CANCELLED** → Clear ETA

**Implementation**:
```typescript
async onDeliveryStatusChange(orderId: string, newStatus: string): Promise<void> {
  try {
    // Start ETA calculation when order is picked up
    if (newStatus === 'PICKED_UP' || newStatus === 'OUT_FOR_DELIVERY') {
      this.logger.debug(`Order ${orderId} ${newStatus}, triggering ETA calculation`);
      
      // Calculate initial ETA
      await this.deliveryEtaService.calculateEta(orderId);
    }
    
    // Clear ETA when delivery is complete
    if (newStatus === 'DELIVERED' || newStatus === 'CANCELLED') {
      this.logger.debug(`Order ${orderId} ${newStatus}, clearing ETA`);
      await this.clearOrderEta(orderId);
    }
  } catch (error) {
    this.logger.error(`ETA hook failed for order ${orderId} status ${newStatus}:`, error);
  }
}
```

---

#### **Method 2: onLocationUpdate**

**Signature**:
```typescript
async onLocationUpdate(orderId: string, riderId: string): Promise<void>
```

**Purpose**: Trigger ETA recalculation when rider location updates

**Called By**: `RiderLocationService` (every time rider GPS updates)

**Implementation**:
```typescript
async onLocationUpdate(orderId: string, riderId: string): Promise<void> {
  try {
    const shouldRecalculate = await this.deliveryEtaService.recalculateEtaIfNeeded(orderId);
    if (shouldRecalculate) {
      this.logger.debug(`Location update triggered ETA recalculation for order ${orderId}`);
    }
  } catch (error) {
    this.logger.error(`ETA location hook failed for order ${orderId}:`, error);
  }
}
```

**Rate Limiting**: Built-in via `recalculateEtaIfNeeded()` (60-second minimum interval)

---

#### **Method 3: onRiderReassignment**

**Signature**:
```typescript
async onRiderReassignment(orderId: string, newRiderId: string): Promise<void>
```

**Purpose**: Recalculate ETA when order reassigned to different rider

**Implementation**:
```typescript
async onRiderReassignment(orderId: string, newRiderId: string): Promise<void> {
  try {
    this.logger.debug(`Order ${orderId} reassigned to rider ${newRiderId}, recalculating ETA`);
    
    // Clear existing ETA and recalculate with new rider
    await this.clearOrderEta(orderId);
    
    // If order is already picked up, calculate new ETA
    const order = await this.orderRepository.findOne({
      where: { id: orderId },
      select: ['deliveryStatus'],
    });

    if (order && (order.deliveryStatus === 'PICKED_UP' || order.deliveryStatus === 'OUT_FOR_DELIVERY')) {
      await this.deliveryEtaService.calculateEta(orderId);
    }
  } catch (error) {
    this.logger.error(`ETA reassignment hook failed for order ${orderId}:`, error);
  }
}
```

---

## 📦 **DTOs**

### **GetOrderEtaResponseDto**

```typescript
export class EtaDataDto {
  @ApiProperty({
    description: 'Estimated delivery time in minutes from now',
    example: 15,
    nullable: true,
  })
  etaMinutes!: number | null;

  @ApiProperty({
    description: 'Distance to delivery location in kilometers',
    example: 2.3,
    nullable: true,
  })
  distanceKm!: number | null;

  @ApiProperty({
    description: 'When ETA was last calculated (ISO string)',
    example: '2024-01-15T10:30:00Z',
    nullable: true,
  })
  lastUpdatedAt!: string | null;

  @ApiProperty({
    description: 'Source of ETA calculation',
    example: 'maps',
    enum: ['maps', 'fallback', 'none', 'error'],
  })
  source!: string;
}

export class GetOrderEtaResponseDto extends BaseResponseDto {
  @ApiProperty({
    description: 'ETA data',
    type: EtaDataDto,
    required: false,
  })
  data?: EtaDataDto;
}
```

---

### **RecalculateEtaResponseDto**

```typescript
export class RecalculateEtaDataDto {
  @ApiProperty({
    description: 'Newly calculated ETA in minutes',
    example: 18,
  })
  newEtaMinutes!: number;

  @ApiProperty({
    description: 'When the calculation was performed (ISO string)',
    example: '2024-01-15T10:35:00Z',
  })
  calculatedAt!: string;
}

export class RecalculateEtaResponseDto extends BaseResponseDto {
  @ApiProperty({
    description: 'Recalculation result data',
    type: RecalculateEtaDataDto,
    required: false,
  })
  data?: RecalculateEtaDataDto;
}
```

---

## 💾 **Database Schema**

### **Order Entity (ETA Fields)**

**Table**: `orders`

**ETA-Related Columns**:
```sql
ALTER TABLE orders ADD COLUMN estimated_delivery_minutes INT;
ALTER TABLE orders ADD COLUMN eta_last_calculated_at TIMESTAMPTZ;
ALTER TABLE orders ADD COLUMN eta_source VARCHAR(20) DEFAULT 'none';

-- Indexes (if needed for analytics)
CREATE INDEX idx_orders_eta_source ON orders(eta_source);
CREATE INDEX idx_orders_eta_calculated ON orders(eta_last_calculated_at) WHERE eta_last_calculated_at IS NOT NULL;
```

**TypeORM Entity Fields**:
```typescript
@Entity('orders')
export class Order {
  // ... other fields ...

  @Column({ type: 'int', nullable: true, name: 'estimated_delivery_minutes' })
  estimatedDeliveryMinutes?: number;

  @Column({ type: 'timestamptz', nullable: true, name: 'eta_last_calculated_at' })
  etaLastCalculatedAt?: Date;

  @Column({ type: 'varchar', length: 20, default: 'none', name: 'eta_source' })
  etaSource?: string; // 'maps' | 'fallback' | 'none' | 'error'
}
```

---

## 🔗 **Integration Patterns**

### **Integration 1: Rider Location Module**

**Purpose**: Fetch rider's current GPS coordinates

**Usage**:
```typescript
import { RiderLocationService } from '../rider-location/rider-location.service';

// Get rider location for order
const location = await riderLocationService.getLocationForOrder(orderId);
// Returns: { lat: number, lng: number, timestamp: Date } | null
```

**Circular Dependency**: Resolved using `forwardRef(() => RiderLocationModule)`

---

### **Integration 2: Order Module**

**Purpose**: Update order ETA fields

**Usage**:
```typescript
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { Order } from '../../database/entities/order.entity';

// Update order ETA
await orderRepository.update(orderId, {
  estimatedDeliveryMinutes: 15,
  etaLastCalculatedAt: new Date(),
  etaSource: 'maps',
});
```

---

### **Integration 3: Google Maps API**

**Purpose**: Traffic-aware route calculation

**Configuration**:
```bash
# .env
GOOGLE_MAPS_API_KEY=AIzaSy...
```

**API Endpoint**:
```
GET https://maps.googleapis.com/maps/api/directions/json
```

**Rate Limits**:
- Free tier: 40,000 requests/month
- Billing: $5 per 1,000 requests after free tier

---

## 🧮 **Algorithms**

### **Algorithm 1: Haversine Distance**

**Purpose**: Calculate great-circle distance between two lat/lng points

**Formula**:
```
a = sin²(Δφ/2) + cos φ1 × cos φ2 × sin²(Δλ/2)
c = 2 × atan2(√a, √(1−a))
d = R × c

where:
  φ = latitude (radians)
  λ = longitude (radians)
  R = 6371 km (Earth's radius)
```

**Implementation**:
```typescript
function haversineDistance(lat1, lng1, lat2, lng2) {
  const R = 6371; // km
  const φ1 = lat1 * Math.PI / 180;
  const φ2 = lat2 * Math.PI / 180;
  const Δφ = (lat2 - lat1) * Math.PI / 180;
  const Δλ = (lng2 - lng1) * Math.PI / 180;

  const a = Math.sin(Δφ/2) * Math.sin(Δφ/2) +
            Math.cos(φ1) * Math.cos(φ2) *
            Math.sin(Δλ/2) * Math.sin(Δλ/2);
  
  const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a));
  
  return R * c;
}
```

**Complexity**: O(1) - constant time

---

### **Algorithm 2: ETA Smoothing**

**Purpose**: Prevent jumpy UI by limiting ETA changes

**Formula**:
```
smoothed_eta = clamp(new_eta, prev_eta - max_change, prev_eta + max_change)

where:
  max_change = 3 minutes
  clamp(x, min, max) = max(min, min(x, max))
```

**Implementation**:
```typescript
function smoothEta(orderId, newEta) {
  const cached = etaCache.get(orderId);
  if (!cached) {
    return newEta; // First calculation, no smoothing
  }

  const maxChange = 3;
  const prevEta = cached.eta;
  
  return Math.max(
    Math.min(newEta, prevEta + maxChange),
    prevEta - maxChange
  );
}
```

**Complexity**: O(1) - constant time

---

## ⚠️ **Error Handling**

### **Error Codes**

| Error Code | HTTP Status | Description |
|------------|-------------|-------------|
| `ORDER_NOT_FOUND` | 404 | Order does not exist |
| `ETA_FETCH_FAILED` | 500 | Failed to retrieve ETA |
| `ETA_CALCULATION_FAILED` | 400 | ETA calculation not applicable or failed |
| `ETA_RECALC_FAILED` | 500 | ETA recalculation failed |

### **Common Error Patterns**

**Google Maps API Timeout**:
```typescript
try {
  const response = await firstValueFrom(
    this.httpService.get(url, { params, timeout: 5000 }),
  );
} catch (error) {
  this.logger.error('Google Maps API call failed:', error);
  return null; // Fallback to distance-based ETA
}
```

**Missing Rider Location**:
```typescript
const location = await riderLocationService.getLocationForOrder(orderId);
if (!location) {
  return calculateFallbackEta(order); // Default ETA
}
```

**Order Not Picked Up**:
```typescript
if (order.deliveryStatus !== 'PICKED_UP' && order.deliveryStatus !== 'OUT_FOR_DELIVERY') {
  return null; // ETA not applicable
}
```

---

## 🧪 **Testing Strategy**

### **Unit Tests**

**File**: `delivery-eta.service.spec.ts`

**Test Cases**:
1. **calculateEta**:
   - Calculates ETA using Google Maps
   - Falls back to distance-based when API fails
   - Returns null for non-picked-up orders
   - Handles missing rider location

2. **smoothEta**:
   - Clamps ETA changes to ±3 minutes
   - Returns raw ETA for first calculation
   - Gradually converges to actual ETA

3. **calculateDistance**:
   - Calculates correct Haversine distance
   - Handles edge cases (same location, antipodes)

4. **calculateGoogleMapsEta**:
   - Parses Google Maps API response
   - Adds 2-minute buffer
   - Returns null on API failure

---

### **Integration Tests**

**File**: `delivery-eta.e2e.spec.ts`

**Test Scenarios**:
1. Full ETA calculation flow (pickup → ETA calculated)
2. ETA recalculation on location update
3. Fallback ETA when Google Maps fails
4. ETA clearing on delivery completion

---

**[DELIVERY-ETA_TECHNICAL_GUIDE_COMPLETE ✅]**

*For business overview, see `01_FEATURE_OVERVIEW.md`. For testing scenarios, see `03_QA_TEST_CASES.md`.*

---

**Document Version**: 1.0  
**Last Updated**: February 22, 2026  
**Module**: Delivery-ETA (Week 7 - Chef Fulfillment)  
**Status**: ✅ Complete
