# Packaged Food Delivery — QA Test Cases

> **Module:** `packaged-delivery`
> **Status:** Implemented ✅ (March 2026)
> **Priority legend:** P0 = blocker, P1 = high, P2 = medium

---

## TC-PD-001: Domain rule — orderable when shelf life has sufficient margin

**Type:** Automated unit test
**Feature area:** `libs/domain/src/lib/packaged-food-delivery.ts`
**Priority:** P0

**Preconditions:** N/A (pure function)

**Test input:** `shelfLifeDays=7`, `dispatchBufferDays=1`, `estimatedDeliveryDays=3`

**Expected result:** `isPackagedFoodOrderable` returns `true`
**Actual result (before feature):** N/A
**Fix applied:** Domain function implemented
**Regression test:** `libs/domain/src/lib/packaged-food-delivery.spec.ts`
**Status:** Passing ✅

---

## TC-PD-002: Domain rule — strict boundary (buffer + delivery === shelfLife)

**Type:** Automated unit test
**Feature area:** `libs/domain/src/lib/packaged-food-delivery.ts`
**Priority:** P0

**Test input:** `shelfLifeDays=3`, `dispatchBufferDays=1`, `estimatedDeliveryDays=2` (sum = 3 = shelfLife)

**Expected result:** `isPackagedFoodOrderable` returns `false` (item arrives on expiry day — NOT orderable)
**Regression test:** `libs/domain/src/lib/packaged-food-delivery.spec.ts` → "returns false when buffer + delivery === shelfLife"
**Status:** Passing ✅

---

## TC-PD-003: Domain rule — shelfLifeDays = 0 always blocked

**Type:** Automated unit test
**Feature area:** `libs/domain/src/lib/packaged-food-delivery.ts`
**Priority:** P0

**Test input:** `shelfLifeDays=0`, `dispatchBufferDays=0`, `estimatedDeliveryDays=0`

**Expected result:** `isPackagedFoodOrderable` returns `false`
**Regression test:** `libs/domain/src/lib/packaged-food-delivery.spec.ts`
**Status:** Passing ✅

---

## TC-PD-004: Domain rule — negative shelf life always blocked

**Type:** Automated unit test
**Feature area:** `libs/domain/src/lib/packaged-food-delivery.ts`
**Priority:** P0

**Test input:** `shelfLifeDays=-1`, `dispatchBufferDays=1`, `estimatedDeliveryDays=2`

**Expected result:** `isPackagedFoodOrderable` returns `false`
**Regression test:** `libs/domain/src/lib/packaged-food-delivery.spec.ts`
**Status:** Passing ✅

---

## TC-PD-005: Distance zone estimation — correct bucket selection

**Type:** Automated unit test
**Feature area:** `libs/domain/src/lib/packaged-food-delivery.ts`
**Priority:** P1

**Test inputs and expected outputs:**
| Distance | Expected days |
|---|---|
| 0 km | 2 |
| 199 km | 2 |
| 200 km | 2 |
| 201 km | 3 |
| 600 km | 3 |
| 601 km | 4 |
| 1500 km | 4 |
| 1501 km | 5 |

**Regression test:** `libs/domain/src/lib/packaged-food-delivery.spec.ts`
**Status:** Passing ✅

---

## TC-PD-006: Shiprocket token cached after first auth

**Type:** Manual / Integration
**Feature area:** `ShiprocketClient.getToken()`
**Priority:** P1

**Steps:**
1. Clear Redis (or set no existing `shiprocket:token` key)
2. Call `checkServiceability()` which triggers `getToken()`
3. Verify a `POST /v1/external/auth/login` was sent once
4. Call `checkServiceability()` again
5. Verify no second auth call made (token returned from cache)

**Expected result:** Auth API called once; subsequent calls reuse cached token
**Status:** Verified ✅

---

## TC-PD-007: Shiprocket 401 — token invalidated and retried

**Type:** Manual / Integration
**Feature area:** `ShiprocketClient`
**Priority:** P1

**Steps:**
1. Pre-seed Redis with an expired/invalid Shiprocket token
2. Call `checkServiceability()`
3. Shiprocket returns 401

**Expected result:**
- Redis key `shiprocket:token` is deleted
- Client immediately re-authenticates and retries
- On retry success: serviceability result returned normally

**Status:** Verified ✅

---

## TC-PD-008: Shiprocket unavailable — falls back to distance zones

**Type:** Manual (mock Shiprocket down)
**Feature area:** `PackagedDeliveryService.checkFeasibility()`
**Priority:** P1

**Steps:**
1. Set `SHIPROCKET_BASE_URL` to an unreachable host
2. Call `checkFeasibility()` with a valid origin and destination

**Expected result:**
- No thrown exception
- `source = 'distance_fallback'` in result
- `isOrderable` depends on distance-zone estimate + shelf life (not defaulted to true/false)

**Status:** Verified ✅

---

## TC-PD-009: No user address → optimistic CTA

**Type:** Manual
**Feature area:** `PackagedDeliveryService.checkFeasibility()`
**Priority:** P1

**Preconditions:** Both pincode and lat/lng absent from params

**Expected result:**
- Returns `{ isOrderable: true, noLocationData: true }`
- Frontend renders CTA as "Order Now" (not "Not Available")

**Status:** Verified ✅

---

## TC-PD-010: Explore enrichment attaches deliveryFeasibility to reel items

**Type:** Manual / Integration
**Feature area:** `explore.service.ts → enrichWithDeliveryFeasibility()`
**Priority:** P0

**Preconditions:**
- A reel in the DB has `linkedMenu.menuItemId` for an item with `nationalDeliveryEnabled=true`
- User has a default address set

**Steps:**
1. Call `GET /api/v1/explore/sections` (authenticated)
2. Inspect the reel item in the response

**Expected result:** `item.linkedMenu.deliveryFeasibility` is present with `isOrderable`, `estimatedDeliveryDays`, `totalDaysFromOrder`

**Status:** Verified ✅

---

## TC-PD-011: Cart removes not_deliverable item on validation

**Type:** Manual
**Feature area:** `cart.service.ts → validateCart()`
**Priority:** P0

**Preconditions:**
- Cart has a `nationalDeliveryEnabled=true` item
- User's location makes `isOrderable=false`

**Steps:**
1. Navigate to cart
2. Cart validation runs automatically

**Expected result:**
- Item is removed from cart
- Response includes change with `type: 'not_deliverable'`

**Status:** Verified ✅

---

## TC-PD-012: Local items unaffected by national delivery logic

**Type:** Manual
**Feature area:** All layers
**Priority:** P1

**Preconditions:**
- Cart and explore contain items with `nationalDeliveryEnabled=false` (the default)

**Steps:**
1. Load explore feed
2. Load cart
3. Complete checkout

**Expected result:** No `deliveryFeasibility` enrichment applied; no `not_deliverable` validation; ordering works as before

**Status:** Verified ✅

---

## TC-PD-013: Pre-migration graceful degradation

**Type:** Manual
**Feature area:** `explore.service.ts → enrichWithDeliveryFeasibility()`
**Priority:** P1

**Preconditions:** `chef_menu_item` does NOT yet have the `nationalDeliveryEnabled` column (migration not run)

**Steps:**
1. Trigger the explore endpoint

**Expected result:**
- TypeORM query error caught silently
- Explore sections returned normally (no enrichment, no crash)

**Status:** Verified ✅

---

## TC-PD-014: DTO validation rejects invalid dispatchBufferDays

**Type:** Automated / Integration
**Feature area:** `create-menu-item.dto.ts`
**Priority:** P2

**Test inputs:**
- `dispatchBufferDays: -1` → should reject (Min 0)
- `dispatchBufferDays: 8` → should reject (Max 7)
- `dispatchBufferDays: 3` → should accept

**Status:** class-validator decorators in place ✅

---

## TC-PD-015: DTO validation rejects malformed pincode

**Type:** Automated / Integration
**Feature area:** `create-kitchen.dto.ts`
**Priority:** P2

**Test inputs:**
- `pincode: "12345"` → reject (5 digits)
- `pincode: "ABCDEF"` → reject (letters)
- `pincode: "400001"` → accept
- `pincode: "110011"` → accept

**Status:** `@Matches(/^\d{6}$/)` in place ✅

---

**Last Updated**: March 2026
