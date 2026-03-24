# Packaged Food Delivery — Feature Overview

> **Module:** `packaged-delivery`
> **Status:** Implemented ✅ (March 2026)
> **Scope:** PAN-India shelf-life-aware delivery feasibility for packaged food items

---

## Business Purpose

Chefooz supports home chefs who produce packaged foods (pickles, masalas, chutneys, dry snacks, sweets, etc.) that can be shipped across India via courier. However, perishable products have finite shelf lives, and a product that takes 4 days to arrive doesn't make sense if it expires in 3 days.

This feature:
1. Lets chefs mark individual menu items as nationally shippable and configure their shelf life / dispatch buffer
2. Checks whether a specific user location can receive the item before it expires
3. Shows clear UI feedback ("Ships India 🇮🇳", "Delivers in N days", or "Not Available")
4. Guards the cart against adding items that cannot reach the user in time

This is **critical to Chefooz's vision** as a platform for India's home food economy:
- Opens orders from any corner of India (not just 25 km radius)
- Prevents bad customer experiences from expired deliveries
- Builds trust in the national delivery proposition

---

## Key Concepts

### Shelf Life Constraint (domain rule)

```
isOrderable = (dispatchBufferDays + estimatedDeliveryDays) < shelfLifeDays
```

- `dispatchBufferDays`: days the chef needs to pack and hand off to the courier (default: 1)
- `estimatedDeliveryDays`: transit time from kitchen's pincode to user's pincode (via Shiprocket or fallback)
- `shelfLifeDays`: product's remaining shelf life at dispatch (set on menu item, enforced by parent `isPackagedFood=true`)
- **Strictly less than**: arriving on the expiry day is NOT orderable — product must arrive with at least 1 day of shelf life remaining

### DeliveryType

| Value | Meaning |
|---|---|
| `local` | Standard delivery within the chef's service radius |
| `national` | Courier-based PAN-India shipping via Shiprocket |

---

## User-Facing Features

### In Explore Feed

| UI Element | Trigger |
|---|---|
| "🇮🇳 Ships India" badge on card image | Item has `nationalDeliveryEnabled=true` |
| "Delivers in N days" (replaces ETA minutes) | Item is national + orderable |
| "Not Available" CTA (grey, disabled) | Item is national + not orderable to user's location |
| Red reason text below price | `unorderableReason` from domain rule |
| "Order Now" CTA (enabled) | User has no address set (optimistic) |

### In Cart

- If a nationally-enabled item becomes non-orderable at cart validation time (e.g., user changes address), it is **automatically removed**
- A `CartValidationChange` of type `not_deliverable` is returned to the frontend
- Users see a notification explaining why the item was removed

---

## Chef-Facing Configuration

Chefs configure per menu item (via `UpdateMenuItemRequest`):
- `nationalDeliveryEnabled: boolean` — toggles PAN-India mode
- `dispatchBufferDays: number` — 0–7 days (how long to pack before handoff)

Chefs also set their kitchen's 6-digit `pincode` (used as the origin for Shiprocket serviceability checks).

> **Note**: A Chef UI for toggling these settings (in the Chef Dashboard) is a follow-on task.

---

## Logistics Provider

**Primary**: Shiprocket — queries serviceability and estimated transit days per pincode pair, caches results for 6 hours.

**Fallback** (when Shiprocket is unavailable or unconfigured): Distance-based zone estimate using Haversine:
| Distance | Estimated Days |
|---|---|
| 0–200 km | 2 days |
| 200–600 km | 3 days |
| 600–1500 km | 4 days |
| 1500+ km | 5 days |

---

## Screens Affected

| Screen / Component | Change |
|---|---|
| `ExploreMenuShowcase.tsx` | National badge, days label, disabled CTA |
| Cart screen | `not_deliverable` removal notification |

---

**Last Updated**: March 2026
