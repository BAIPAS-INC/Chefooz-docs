# Reputation API Reference

**Last Updated**: 2026-05-02
**Base path**: `/api/v1/crs`

**Phase Status**: MVP Phase 1 (May 2, 2026) — 9/16 events production-ready

---

## MVP Phase 1 Production-Ready Events

Only the 9 events below emit production events end-to-end via their respective producer modules.
The remaining 7 events are deferred to Phase 2 & 3 and can only be triggered via admin API for now.

| Event | Producer Module | Line |
|---|---|---|
| REVIEW_SUBMITTED | review.service.ts | 168 |
| HYGIENE_POSITIVE | review.service.ts | 175 |
| REEL_UPLOADED_FROM_ORDER | media.service.ts | 1245 |
| CONVERSION_INFLUENCE | order.service.ts | 1580 |
| DELIVERY_HELPFUL | rider-rating.service.ts | 122 |
| FOLLOWER_MILESTONE | social.service.ts | 790 |
| ENGAGEMENT_HEALTHY | reputation.service.ts | 964 (cron) |
| CONSISTENCY_WEEK | reputation.service.ts | 964 (cron) |
| ADMIN_OVERRIDE | reputation.controller.ts | 123 |

**Deferred to future phases:**
- Phase 2 (Q3 2026): HELPFUL_VOTES, SPAM_REPORTED, BIASED_REVIEW
- Phase 3 (Q4 2026): CHEF_COMPLAINT_VALID, FORCED_ENGAGEMENT, LOCATION_MISMATCH

---


---

## Profile Endpoints

### `GET /api/v1/crs/me`

Get authenticated user's reputation profile.

**Guards**: `JwtAuthGuard`

### `GET /api/v1/crs/user/:userId`

Get public profile for a target user with privacy-filtered fields.

**Guards**: Public endpoint (optional auth context if available)

### `GET /api/v1/crs/admin/user/:userId`

Get full reputation details for any user (admin only).

**Guards**: `JwtAuthGuard` + admin role check

---

## Leaderboard Endpoints

### `GET /api/v1/crs/leaderboard`

Get paginated leaderboard.

#### Query Parameters

| Param | Type | Required | Default | Description |
|---|---|---|---|---|
| `period` | `weekly \| monthly` | Yes | - | Leaderboard period |
| `limit` | number | No | 20 | Max rows returned |
| `cursor` | string | No | - | Next-page cursor (rank) |

#### Response

```json
{
  "success": true,
  "message": "Leaderboard retrieved successfully",
  "data": {
    "entries": [
      {
        "rank": 1,
        "userId": "uuid",
        "username": "chef_name",
        "level": "gold",
        "score": 12000,
        "createdAt": "2026-05-02T10:10:00.000Z"
      }
    ],
    "nextCursor": "20",
    "period": "weekly"
  }
}
```

### `POST /api/v1/crs/admin/rebuild-leaderboard`

Rebuild weekly and monthly leaderboard rows from current reputation scores.

**Guards**: `JwtAuthGuard` + admin role check

**Automation note**: A scheduled rebuild also runs daily via cron; this endpoint is the manual recovery trigger.

#### Response

```json
{
  "success": true,
  "message": "Leaderboard rebuilt with 2000 entries",
  "data": {
    "entries": 2000
  }
}
```

---

## Event Endpoints

### `POST /api/v1/crs/events`

Restricted ingestion endpoint for trusted internal/admin event producers.

**Guards**: `JwtAuthGuard` + admin role check

#### Request Body

| Field | Type | Required | Constraints |
|---|---|---|---|
| `userId` | UUID string | Yes | Target user receiving the event |
| `type` | enum | Yes | One of `ReputationEventType` |
| `source` | enum | Yes | `review-module` \| `media-module` \| `order-module` \| `rider-rating-module` \| `social-module` \| `reputation-cron` \| `admin-api` \| `internal-api` |
| `referenceId` | string | No | Required for event types tied to an entity (`REVIEW_SUBMITTED`, `REEL_UPLOADED_FROM_ORDER`, `HYGIENE_POSITIVE`, `DELIVERY_HELPFUL`, `CONVERSION_INFLUENCE`) |
| `meta` | object | No | Event-specific metadata (e.g. `linkedReelId`, `followerCount`, `reelPurpose`) |

#### Runtime Validation Rules

- Source is validated against event type allow-lists (for example, `CONVERSION_INFLUENCE` allows only `order-module`, `admin-api`, `internal-api`).
- `CONVERSION_INFLUENCE` requires `meta.linkedReelId`.
- `REEL_UPLOADED_FROM_ORDER` requires `meta.reelPurpose` of `USER_REVIEW` or `PROMOTIONAL`.
- `FOLLOWER_MILESTONE` requires a valid milestone `meta.followerCount` in `{10,50,100,500,1000,5000,10000}`.

#### Response

```json
{
  "success": true,
  "message": "Reputation event queued for processing",
  "data": {
    "jobId": "12345"
  }
}
```

**Reliability note**: Event writes are asynchronous via Bull queue (`reputation-events`) with exponential retry and terminal dead-letter logging on final failure.

---

## Admin Override Endpoint

### `POST /api/v1/crs/admin/override`

Manually set a user's reputation score (admin only).

**Guards**: `JwtAuthGuard` + admin role check

#### Request Body

| Field | Type | Required | Constraints |
|---|---|---|---|
| `userId` | UUID string | Yes | Valid UUID |
| `newScore` | number | Yes | Integer, min `0`, max `1,000,000` |
| `reason` | string | Yes | Non-empty text |

#### Response

```json
{
  "success": true,
  "message": "Reputation override applied. Score changed from 4200 to 75000",
  "data": {
    "score": 75000,
    "level": "diamond",
    "coinMultiplier": 1.2,
    "feedBoostWeight": 1.5,
    "perks": {
      "earlyAccess": true,
      "chefDiscountTier": "A+",
      "exclusiveThemes": ["neon", "chef-pro"]
    },
    "updatedAt": "2026-05-02T10:10:00.000Z"
  }
}
```

---

## Manual Job Endpoints

### `POST /api/v1/crs/admin/run-weekly-decay`

Manual trigger for weekly inactivity decay.

### `POST /api/v1/crs/admin/run-engagement-check`

Manual trigger for weekly engagement and consistency scoring.

---

## Scheduled Background Jobs

The following jobs run automatically when scheduler is active:

- `0 3 * * 0` — weekly snapshot generation
- `0 6 * * 0` — weekly digest generation
- `0 4 * * *` — leaderboard rebuild

Snapshot persistence is idempotent per `userId + weekStart` via upsert and unique DB index.
