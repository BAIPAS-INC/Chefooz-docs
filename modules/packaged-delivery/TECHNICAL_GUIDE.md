# Packaged Food Delivery — Technical Guide

> **Module:** `packaged-delivery`
> **Location:** `apps/chefooz-apis/src/modules/packaged-delivery/`
> **Status:** Implemented ✅ (March 2026)

---

## Module Structure

```
apps/chefooz-apis/src/modules/packaged-delivery/
  packaged-delivery.module.ts      NestJS module (HttpModule, CacheModule, ConfigModule)
  packaged-delivery.service.ts     Feasibility orchestration (Shiprocket + fallback + domain rule)
  shiprocket.client.ts             Shiprocket API auth + serviceability client

libs/domain/src/lib/
  packaged-food-delivery.ts        Pure domain functions (no side effects)
  packaged-food-delivery.spec.ts   Unit tests (~20 cases)

libs/types/src/lib/
  packaged-delivery.types.ts       Shared response/result types

apps/chefooz-apis/src/database/migrations/
  1775000000000-AddPackagedDeliveryFields.ts   DB migration (idempotent)
```

---

## Domain Rule

**File:** `libs/domain/src/lib/packaged-food-delivery.ts`

### `isPackagedFoodOrderable(shelfLifeDays, dispatchBufferDays, estimatedDeliveryDays): boolean`

```ts
return (dispatchBufferDays + estimatedDeliveryDays) < shelfLifeDays;
```

**STRICT**: `===` returns `false`. Arriving on the exact expiry day is unacceptable.

### `getUnorderableReason(shelfLifeDays, dispatchBufferDays, estimatedDeliveryDays): string`

Returns a human-readable message for the UI. Examples:
- `"This item cannot be delivered to your location within its shelf life"` (general)
- `"Shelf life too short: item would take X day(s) to arrive but only has Y day shelf life"` (specific)

### `estimateDeliveryDaysFromDistanceKm(distanceKm): number`

Haversine-based zone fallback when Shiprocket is unavailable:
| Distance | Days |
|---|---|
| ≤ 200 km | 2 |
| 201–600 km | 3 |
| 601–1500 km | 4 |
| > 1500 km | 5 |

---

## Shared Types

**File:** `libs/types/src/lib/packaged-delivery.types.ts`

### `DeliveryFeasibility`

```ts
interface DeliveryFeasibility {
  isOrderable: boolean;
  deliveryType: DeliveryType;           // 'local' | 'national'
  estimatedDeliveryDays?: number;       // transit days only (courier)
  totalDaysFromOrder?: number;          // dispatchBuffer + transit
  unorderableReason?: string;           // human-readable reason when !isOrderable
  courierName?: string;                 // Shiprocket courier name if available
  noLocationData?: boolean;             // true when user has no address; CTA stays enabled
}
```

### `CourierServiceabilityResult`

Internal type returned by `ShiprocketClient.checkServiceability()`:
```ts
interface CourierServiceabilityResult {
  isServiceable: boolean;
  estimatedDeliveryDays?: number;
  courierName?: string;
  source: 'shiprocket' | 'distance_fallback';
}
```

---

## ShiprocketClient

**File:** `apps/chefooz-apis/src/modules/packaged-delivery/shiprocket.client.ts`

### Authentication

- `POST /v1/external/auth/login` with `{email, password}`
- JWT token cached in Redis under `shiprocket:token` for **9 days** (token validity minus 1 day safety)
- `getToken()` reads from cache first; fetches and caches if expired
- On HTTP 401 from any API call: token key is deleted, single retry attempted

### Serviceability Check

```
GET /v1/external/courier/serviceability
  ?pickup_postcode={originPin}
  &delivery_postcode={destPin}
  &weight=0.5        (default 500g)
  &cod=0
```

- Result cached: `courier:svc:{originPin}:{destPin}` for **6 hours**
- Picks the courier with the lowest `estimated_delivery_days` from available couriers
- Returns `isServiceable=false` with `source='shiprocket'` when no couriers available for the route

### Fallback

When Shiprocket is unreachable, credentials are not configured, or an unexpected error occurs:
1. `isServiceable=true` (optimistic — do not block on infrastructure failure)
2. `estimatedDeliveryDays` from `estimateDeliveryDaysFromDistanceKm(haversineDistance)`
3. `source='distance_fallback'`

---

## PackagedDeliveryService

**File:** `apps/chefooz-apis/src/modules/packaged-delivery/packaged-delivery.service.ts`

### `checkFeasibility(params: CheckFeasibilityParams): Promise<DeliveryFeasibility>`

```ts
interface CheckFeasibilityParams {
  shelfLifeDays: number;
  dispatchBufferDays: number;
  originPincode?: string;          // chef kitchen pincode
  destinationPincode?: string;     // user's default address pincode
  originLat?: number;              // kitchen lat (for distance fallback)
  originLng?: number;
  destinationLat?: number;         // user lat
  destinationLng?: number;
}
```

**Flow:**
1. If either pincode is missing → attempt distance-based fallback using lat/lng
2. If both lat/lng missing → return `{ isOrderable: true, noLocationData: true }` (optimistic)
3. Call `ShiprocketClient.checkServiceability(originPin, destPin)`
4. Apply domain rule: `isPackagedFoodOrderable(shelfLifeDays, dispatchBufferDays, transitDays)`
5. Return fully populated `DeliveryFeasibility`

### `checkFeasibilityBatch(request: BatchFeasibilityRequest): Promise<BatchFeasibilityResult>`

Runs all items via `Promise.all` for concurrent checks. Intended for potential checkout batch use.

---

## Database Schema

### New Columns

**`chef_menu_item` table:**
```sql
national_delivery_enabled  BOOLEAN   DEFAULT false  NOT NULL
dispatch_buffer_days       SMALLINT  DEFAULT 1      NOT NULL
```

**`chef_kitchen` table:**
```sql
pincode  VARCHAR(10)  NULL
```

### TypeORM Entities

- `chef-menu-item.entity.ts`: `nationalDeliveryEnabled!: boolean`, `dispatchBufferDays!: number`
- `chef-kitchen.entity.ts`: `pincode?: string`

### Migration

File: `1775000000000-AddPackagedDeliveryFields.ts`

Uses `ADD COLUMN IF NOT EXISTS` — safe to re-run; idempotent.

---

## Module Registration

### `PackagedDeliveryModule`

Imports:
- `ConfigModule` — for `SHIPROCKET_*` env vars
- `HttpModule.register({ timeout: 8000 })` — for Shiprocket HTTP calls
- `CacheModule` — for Redis token + serviceability caching

Exports: `PackagedDeliveryService`

### Consuming Modules

| Module | How it uses PackagedDeliveryService |
|---|---|
| `ExploreModule` | `enrichWithDeliveryFeasibility()` — per reel item enrichment |
| `CartModule` | `validateCart()` — `not_deliverable` check before checkout |

---

## Explore Integration

**File:** `apps/chefooz-apis/src/modules/explore/explore.service.ts`

`enrichWithDeliveryFeasibility(sections, userId)`:
1. Collects all `menuItemIds` from items with `linkedMenu`
2. Queries `ChefMenuItemEntity` where `id IN menuItemIds AND nationalDeliveryEnabled = true` (single batched SELECT)
3. Fetches user's default `UserAddress` (pincode, lat, lng)
4. Fetches `ChefKitchen.pincode` for all relevant chef IDs
5. For each matching reel item: calls `checkFeasibility()`, attaches result to `item.linkedMenu.deliveryFeasibility`
6. Called after full mapping but **before** Redis caching of sections

Error handling: TypeORM column-not-found error for `nationalDeliveryEnabled` (pre-migration) is caught; enrichment silently skipped.

---

## Cart Integration

**File:** `apps/chefooz-apis/src/modules/cart/cart.service.ts`

In `validateCart()`, after the existing stock/availability checks:
1. Fetches user's default address (once, shared across all items)
2. For each cart item where `isPackagedFood && nationalDeliveryEnabled`:
   - Fetches `ChefKitchen.pincode` for the item's chef
   - Calls `PackagedDeliveryService.checkFeasibility()`
   - If `!isOrderable && !noLocationData`: removes item from cart, pushes `CartValidationChange` with `type: 'not_deliverable'`

`cart.types.ts` `CartValidationChange.type` union now includes `'not_deliverable'`.

---

## DTO Validation

**`create-menu-item.dto.ts` / `update-menu-item.dto.ts`:**
```ts
@IsOptional()
@IsBoolean()
nationalDeliveryEnabled?: boolean;

@IsOptional()
@IsNumber()
@Min(0)
@Max(7)
dispatchBufferDays?: number;
```

**`create-kitchen.dto.ts`:**
```ts
@IsOptional()
@Matches(/^\d{6}$/)
pincode?: string;
```

---

## Frontend Types (`libs/utils/src/lib/explore-data-mapper.ts`)

`ExploreFoodItem` now has:
```ts
deliveryFeasibility?: {
  isOrderable: boolean;
  estimatedDeliveryDays?: number;
  totalDaysFromOrder?: number;
  unorderableReason?: string;
  courierName?: string;
  noLocationData?: boolean;
};
deliveryType?: 'local' | 'national';
```

Set in `mapReelToFoodItem()` from `reel.linkedMenu.deliveryFeasibility`.

---

## Required Environment Variables

| Variable | Required | Description |
|---|---|---|
| `SHIPROCKET_BASE_URL` | Yes (for live) | Shiprocket API base URL |
| `SHIPROCKET_EMAIL` | Yes (for live) | Shiprocket account email |
| `SHIPROCKET_PASSWORD` | Yes (for live) | Shiprocket account password |
| `EXPLORE_NEAR_YOU_RADIUS_KM` | No (default 25) | Near-you radius (unrelated but same service) |

If `SHIPROCKET_EMAIL` or `SHIPROCKET_PASSWORD` are missing, `ShiprocketClient` falls back to distance zones automatically.

---

## Constraints & Edge Cases

| Scenario | Handling |
|---|---|
| User has no default address | `noLocationData=true`, CTA remains enabled |
| Chef kitchen has no `pincode` | Falls back to lat/lng Haversine estimate |
| Haversine fallback: no lat/lng either | `noLocationData=true` returned |
| Shiprocket HTTP 401 | Token cache cleared, single retry |
| Shiprocket timeout / server error | Distance-based fallback activates |
| No available couriers for route | `isServiceable=false`, item blocked |
| `shelfLifeDays=0` or unset | Domain rule: `0 > 0` = `false`, always blocked |
| `dispatch + delivery === shelfLife` | Domain rule STRICT: blocked (arrives on expiry day) |
| Migration not yet run | TypeORM query for `nationalDeliveryEnabled` catches error, enrichment skipped |

---

**Last Updated**: March 2026
