# User API Reference

**Last Updated**: 2026-03-XX  
**Base path**: `/api/v1`

---

## Admin Endpoints

### `GET /api/v1/admin/users`

List all platform users (customers, chefs, riders) with pagination, search, and filtering.

**Guards**: `JwtAuthGuard` + `RolesGuard` (`admin` role required)

#### Query Parameters

| Param | Type | Required | Default | Description |
|---|---|---|---|---|
| `page` | number | No | 1 | Page number (1-based) |
| `limit` | number | No | 20 | Items per page (max 100) |
| `search` | string | No | — | ILIKE match on `fullName`, `username`, `phone`, `displayName` |
| `role` | `user \| chef \| rider` | No | — | Filter by role (admin always excluded) |
| `trustState` | `NORMAL \| WARNED \| RESTRICTED \| FRICTION_REQUIRED` | No | — | Filter by trust state |

#### Response

```json
{
  "success": true,
  "message": "Users retrieved",
  "data": {
    "items": [
      {
        "id": "uuid",
        "fullName": "Priya Sharma",
        "username": "priya_eats",
        "phone": "+919876543210",
        "avatarUrl": "https://cdn.chefooz.com/avatars/uuid.jpg",
        "role": "user",
        "trustState": "NORMAL",
        "createdAt": "2025-10-14T08:22:00Z",
        "chefBusinessName": null,
        "chefVerified": null,
        "reportCount": 0,
        "orderCount": 12
      }
    ],
    "total": 1540,
    "page": 1,
    "limit": 20,
    "totalPages": 77
  }
}
```

#### Notes

- `chefBusinessName` and `chefVerified` are only populated when `role = 'chef'`
- `reportCount` = number of times this user was the **target** of a report (not filed by them)
- `orderCount` = total orders placed/associated with this user
- Both counts are batch-fetched per page — no N+1 queries

---

## Customer-facing Endpoints

> These endpoints use the authenticated user's JWT — not an admin token.

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/v1/user/addresses` | List user delivery addresses |
| `POST` | `/api/v1/user/addresses` | Add a new delivery address |
| `PUT` | `/api/v1/user/addresses/:id` | Update a delivery address |
| `DELETE` | `/api/v1/user/addresses/:id` | Delete a delivery address |
| `GET` | `/api/v1/user/username/check` | Check username availability |
| `PUT` | `/api/v1/user/username` | Set or update username |

> Full request/response details for customer endpoints are documented in the [User Technical Guide](../modules/user/TECHNICAL_GUIDE.md).
