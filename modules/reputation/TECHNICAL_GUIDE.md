# Reputation Module — Technical Guide

**Version:** 2.0 (Production-Grade Scale)
**Last Updated:** May 2, 2026

---

## May 2, 2026 — Upload event taxonomy fix: REEL_UPLOADED_GENERIC introduced

- `REEL_UPLOADED_GENERIC` added as a new CRS event covering `PROMOTIONAL` and `MENU_SHOWCASE` reel uploads.
- `REEL_UPLOADED_FROM_ORDER` is now **strictly reserved** for `USER_REVIEW` reels that include a `linkedOrderId`.
- `MENU_SHOWCASE` uploads now earn upload points (previously untracked) — +80 pts per event.
- `PROMOTIONAL` uploads earn +80 pts via `REEL_UPLOADED_GENERIC` (previously incorrectly using `REEL_UPLOADED_FROM_ORDER`).
- Backfill migration: existing `REEL_UPLOADED_FROM_ORDER` rows with `meta.reelPurpose='PROMOTIONAL'` are rewritten to `REEL_UPLOADED_GENERIC`.

## May 2, 2026 — User-facing modal copy no longer shows exact point values

- Reputation modal text in the mobile app now avoids exposing exact per-event point values and exact score thresholds.
- UI copy source of truth:
  - `apps/chefooz-app/src/constants/labels/profile.labels.ts`
  - `PROFILE_LABELS.reputation.modal.earnPoints`
  - `PROFILE_LABELS.reputation.modal.tiers`
- Backend scoring remains unchanged and centralized in:
  - `libs/domain/src/environment/environment.config.ts` (`reputation.eventWeights`)
  - `apps/chefooz-apis/src/modules/reputation/utils/leveling.ts`
- Reason for policy:
  - reduces behavior gaming,
  - avoids drift when ops tunes CRS weights,
  - keeps UX guidance principle-based instead of formula-based.

### Numeric value suppression across all reputation UI surfaces

- The no-numeric-disclosure policy is now enforced beyond modal copy:
  - `apps/chefooz-app/src/app/profile/reputation.tsx`:
    - hero card shows current tier name instead of numeric score.
  - `apps/chefooz-app/src/components/reel/identity/ReelUserIdentity.tsx`:
    - avatar badge shows tier initial only (no score).
  - `apps/chefooz-app/src/app/profile/_components/ProfileHeader.tsx`:
    - profile avatar reputation badge shows tier only.
  - `apps/chefooz-app/src/app/reputation/leaderboard.tsx`:
    - list rows show tier only; no points text rendered.

- Backend and ranking logic are unchanged; only user-facing rendering is suppressed.

## May 2, 2026 — Production hardening audit for event wiring

- `FOLLOWER_MILESTONE` now uses the same central frequency-cap enforcement as other positive events (`1 / 30 days`), matching `crs.weights.json`.
- `checkWeeklyEngagement()` now increments `engagementFired` / `consistencyFired` only when a reputation row was actually inserted, not merely when a user was eligible before cap checks.
- A dedicated Jest regression spec now exists at `apps/chefooz-apis/src/modules/reputation/reputation.service.spec.ts`.

### Event wiring status matrix

| Event | Weight | Live trigger | Status |
|---|---:|---|---|
| `REVIEW_SUBMITTED` | +60 | `review.service.ts` on saved review | Wired |
| `REEL_UPLOADED_FROM_ORDER` | +100 | `media.service.ts` — USER_REVIEW reels with `linkedOrderId` only | Wired |
| `REEL_UPLOADED_GENERIC` | +80 | `media.service.ts` — PROMOTIONAL (user) and MENU_SHOWCASE reels | Wired |
| `ENGAGEMENT_HEALTHY` | +50 | `ReputationService.checkWeeklyEngagement()` cron/manual admin trigger | Wired |
| `CONSISTENCY_WEEK` | +75 | `ReputationService.checkWeeklyEngagement()` cron/manual admin trigger | Wired |
| `HYGIENE_POSITIVE` | +150 | `review.service.ts` when `dto.hygiene >= 4` | Wired |
| `DELIVERY_HELPFUL` | +40 | `rider-rating.service.ts` when rating `>= 4` | Wired |
| `CONVERSION_INFLUENCE` | +200 | `order.service.ts` when attributed order credits a reel owner | Wired |
| `FOLLOWER_MILESTONE` | variable | `social.service.ts` when follower count crosses threshold | Wired |
| `HELPFUL_VOTES` | +50 | No module trigger found | Unwired |
| `SPAM_REPORTED` | -300 | No module trigger found | Unwired |
| `BIASED_REVIEW` | -180 | No module trigger found | Unwired |
| `CHEF_COMPLAINT_VALID` | -270 | No module trigger found | Unwired |
| `FORCED_ENGAGEMENT` | -330 | No module trigger found | Unwired |
| `LOCATION_MISMATCH` | -225 | No module trigger found | Unwired |
| `ADMIN_OVERRIDE` | 0 / manual delta | `reputation.service.ts` admin override / decay path | Wired |

### Release constraint

- The unwired events above are defined in the CRS model but still require business-specific module triggers. They should not be treated as active production signals until corresponding service-level integration and regression coverage exist.
### Production MVP Phase 1 (May 2026)

**Scope: 10 events fully production-ready and end-to-end wired**

| Event | Weight | Live trigger | Phase |
|---|---:|---|---|
| `REVIEW_SUBMITTED` | +60 | `review.service.ts:168` on saved review | ✅ MVP Phase 1 |
| `REEL_UPLOADED_FROM_ORDER` | +100 | `media.service.ts` — USER_REVIEW reels with `linkedOrderId` | ✅ MVP Phase 1 |
| `REEL_UPLOADED_GENERIC` | +80 | `media.service.ts` — PROMOTIONAL (user) + MENU_SHOWCASE reels | ✅ MVP Phase 1 |
| `ENGAGEMENT_HEALTHY` | +50 | `reputation.service.ts:964` cron (Monday 2 AM) | ✅ MVP Phase 1 |
| `CONSISTENCY_WEEK` | +75 | `reputation.service.ts:964` cron (Monday 2 AM) | ✅ MVP Phase 1 |
| `HYGIENE_POSITIVE` | +150 | `review.service.ts:175` when `dto.hygiene >= 4` | ✅ MVP Phase 1 |
| `DELIVERY_HELPFUL` | +40 | `rider-rating.service.ts:122` when rating `>= 4` | ✅ MVP Phase 1 |
| `CONVERSION_INFLUENCE` | +200 | `order.service.ts:1580` on order attribution | ✅ MVP Phase 1 |
| `FOLLOWER_MILESTONE` | variable | `social.service.ts:790` on follower threshold | ✅ MVP Phase 1 |
| `ADMIN_OVERRIDE` | 0 / manual | `reputation.controller.ts:123` admin endpoint | ✅ MVP Phase 1 |

### Future Phases (Roadmap)

**Phase 2: Moderation & Quality (Q3 2026 estimate)**

| Event | Weight | Required Module | Notes |
|---|---:|---|---|
| `HELPFUL_VOTES` | +50 | comment/reaction module | Detect helpful vote signals |
| `SPAM_REPORTED` | -300 | moderation.service | Integrate spam detection pipeline |
| `BIASED_REVIEW` | -180 | review-quality module | Integrate review bias ML model |

**Phase 3: Fraud & Address Validation (Q4 2026 estimate)**

| Event | Weight | Required Module | Notes |
|---|---:|---|---|
| `CHEF_COMPLAINT_VALID` | -270 | complaint module | Customer dispute resolution workflow |
| `FORCED_ENGAGEMENT` | -330 | fraud-detection module | ML-based suspicious engagement patterns |
| `LOCATION_MISMATCH` | -225 | delivery module | Address validation on fulfillment |

### Production Deployment Safety

- **MVP scope is locked for production.** Only Phase 1 events emit events end-to-end.
- **Future events are admin-triggerable** but have no automatic producer.
- **Source validation is enforce** at ingestion boundary, preventing cross-module spoofing.
- **Durability via queue** ensures no event loss under transient producer failures.
- **Frequency caps** prevent farming on all active events.

## May 2, 2026 — QA bug-fix hardening (Top 3 production blockers)

### 1) Admin override score range aligned to production scale

- `AdminOverrideDto.newScore` now validates `0..1,000,000` (was `0..100`).
- This matches the current production CRS range used by `mapScoreToLevel()` and environment config.
- File: `apps/chefooz-apis/src/modules/reputation/dto/admin-override.dto.ts`

### 2) Leaderboard rebuild endpoint now executes real rebuild logic

- `POST /api/v1/crs/admin/rebuild-leaderboard` is no longer a placeholder response.
- Controller now delegates to `ReputationService.rebuildLeaderboard()`.
- Rebuild flow:
  - clear leaderboard table,
  - fetch top users from `user_reputation_current` by score DESC,
  - write weekly and monthly ranked rows.
- Files:
  - `apps/chefooz-apis/src/modules/reputation/reputation.controller.ts`
  - `apps/chefooz-apis/src/modules/reputation/reputation.service.ts`

### 3) Score recomputation switched to full event aggregation

- `updateCurrentScore()` now recomputes from full event history for the user.
- Previous incremental window (`createdAt > lastEventAt`) could miss events in high-concurrency scenarios.
- New approach is deterministic and idempotent; event history is the source of truth.
- File: `apps/chefooz-apis/src/modules/reputation/reputation.service.ts`

### New regression tests

- `apps/chefooz-apis/src/modules/reputation/dto/admin-override.dto.spec.ts`
  - accepts 1,000,000 and rejects >1,000,000
- `apps/chefooz-apis/src/modules/reputation/reputation.service.spec.ts`
  - verifies leaderboard rebuild persists both periods
  - verifies score recomputation uses full event history

## May 2, 2026 — QA bug-fix hardening (Next production gaps)

### 4) Shared event-type contract drift removed

- `libs/types` now includes `FOLLOWER_MILESTONE` in `ReputationEventType`.
- This aligns shared client contracts with backend/domain event support.
- File: `libs/types/src/lib/reputation.types.ts`

### 5) Snapshot idempotency enforced (application + database)

- Weekly snapshot writer now uses repository `upsert(..., ['userId', 'weekStart'])`.
- Snapshot entity now declares a unique index on `userId + weekStart`.
- New migration adds a unique DB index for production safety:
  - `apps/chefooz-apis/src/migrations/1778400000000-AddUniqueIndexToReputationSnapshots.ts`

### 6) CRS jobs activated via DI and cron decorators

- `WeeklySnapshotJob`, `WeeklyDigestJob`, and `RebuildLeaderboardJob` now have executable `@Cron(...)` decorators.
- These jobs are now registered as providers in `ReputationModule`, so scheduler discovery can instantiate and run them.
- Weekly decay remains on `ReputationService.applyWeeklyDecay()` to avoid duplicate decay schedules.

### Additional regression test

- `apps/chefooz-apis/src/jobs/crs/weeklySnapshot.job.spec.ts`
  - verifies snapshot writes use upsert conflict keys `['userId', 'weekStart']`

## May 2, 2026 — P0 safeguards (event endpoint security + durable producer delivery)

### 7) Event ingestion endpoint restricted to admin/system workflows

- `POST /api/v1/crs/events` now accepts `InternalRecordEventDto` and requires admin role.
- Endpoint no longer allows authenticated end users to self-credit by posting arbitrary event payloads.
- Payload now explicitly includes:
  - target `userId`
  - trusted `source`
  - source-owned event payload (`type`, `referenceId`, `meta`)
- File:
  - `apps/chefooz-apis/src/modules/reputation/reputation.controller.ts`
  - `apps/chefooz-apis/src/modules/reputation/dto/internal-record-event.dto.ts`

### 8) Source-aware event validation at ingestion boundary

- `ReputationService.validateEventSource()` now enforces:
  - source allow-list per event type,
  - required `referenceId` for entity-bound events,
  - required `meta.linkedReelId` for `CONVERSION_INFLUENCE`,
  - required `meta.reelPurpose = 'USER_REVIEW'` for `REEL_UPLOADED_FROM_ORDER` (PROMOTIONAL/MENU_SHOWCASE are rejected — they must use `REEL_UPLOADED_GENERIC`),
  - required `meta.reelPurpose` of `PROMOTIONAL` or `MENU_SHOWCASE` for `REEL_UPLOADED_GENERIC`,
  - valid follower thresholds for `FOLLOWER_MILESTONE`.
- This prevents cross-module event spoofing and malformed high-value events.

### 9) Durable queue path for producer event delivery

- Producers now call `ReputationService.enqueueEvent(...)` instead of direct `recordEvent(...)` fire-and-forget writes.
- Queue: `reputation-events` (Bull) with retries and exponential backoff.
- Consumer: `ReputationEventsProcessor` calls `processQueuedEvent(...)`.
- Terminal failure behavior: structured dead-letter log emitted via `@OnQueueFailed` once max retries are exhausted.
- Updated producer integrations:
  - `apps/chefooz-apis/src/modules/review/review.service.ts`
  - `apps/chefooz-apis/src/modules/order/order.service.ts`
  - `apps/chefooz-apis/src/modules/rider-rating/rider-rating.service.ts`
  - `apps/chefooz-apis/src/modules/social/social.service.ts`
  - `apps/chefooz-apis/src/modules/media/media.service.ts`

### New regression tests

- `apps/chefooz-apis/src/modules/reputation/reputation.service.spec.ts`
  - verifies queue enqueue options and payload shape
  - verifies disallowed source/event combinations are rejected

## May 2, 2026 — Centralized event weights in environment config

- Reputation event weights were moved from `ReputationService` constants to centralized environment configuration:
  - `libs/domain/src/environment/environment.config.ts`
  - `reputation.eventWeights`
- Runtime scoring now resolves non-milestone deltas via:
  - `getEnvConfig().reputation.eventWeights[eventType]`
- This enables phase-wise tuning from one place without editing service logic.

### Why `ADMIN_OVERRIDE` is `0`

- `ADMIN_OVERRIDE` does not use a static reward/penalty weight.
- Its effective delta is computed dynamically as:
  - `newScore - oldScore`
- Keeping config weight `0` prevents accidental double-counting while preserving event-type compatibility.

### Why `FOLLOWER_MILESTONE` is `0`

- `FOLLOWER_MILESTONE` delta is intentionally variable by threshold.
- Actual points come from `FOLLOWER_MILESTONE_REWARDS` map (e.g., 10 → 100, 100 → 400, 10_000 → 3500).
- Keeping config weight `0` avoids conflicting with threshold-specific reward logic.

## May 2, 2026 — Promotional reel referenceId UUID mismatch fix

- Root cause: `REEL_UPLOADED_FROM_ORDER` producer passed Mongo `mediaId` (ObjectId string) in `referenceId`, but `user_reputation_events.referenceId` is PostgreSQL UUID.
- Runtime symptom: `invalid input syntax for type uuid` during reputation event insertion.

### Fix applied

- Media producer now sends reel media linkage in metadata:
  - `meta.mediaId = <mongo-media-id>`
- Reputation service now:
  - validates reel upload linkage via `meta.mediaId` (or legacy `referenceId`),
  - resolves reel content-source checks using `meta.mediaId`,
  - persists `referenceId = null` for **both** `REEL_UPLOADED_FROM_ORDER` and `REEL_UPLOADED_GENERIC` to avoid UUID-column violations.

### Backward compatibility

- Validation keeps fallback support for legacy payloads still sending reel media id in `referenceId`.
- Persistence sanitization prevents invalid UUID writes while preserving metadata for traceability.

---

## Scale Design

### Old Scale (Pre-March 2026, RETIRED)
- 0–100 points total
- Diamond reachable in ~3 weeks
- Weekly decay: -5 pts (meaningless)
- No frequency caps → unlimited point farming

### New Scale (Production)
- 0–1,000,000 practical ceiling
- Diamond requires ~2 years of genuine activity
- Weekly decay: -250 pts
- Frequency caps on all positive events

---

## File Map

| File | Purpose |
|------|---------|
| `apps/chefooz-apis/src/modules/reputation/reputation.service.ts` | Event recording, frequency cap enforcement, score updates |
| `apps/chefooz-apis/src/modules/reputation/utils/leveling.ts` | `mapScoreToLevel()` — score → UserTier mapping |
| `apps/chefooz-apis/src/modules/reputation/config/crs.weights.json` | Event weights and frequency caps (reference copy) |
| `libs/domain/src/policies/reputation.policy.ts` | `shouldPromoteTier()`, `shouldDemoteTier()`, `applyDecay()`, `applyReputationDecay()` |
| `libs/domain/src/shared/constants.ts` | `REPUTATION_CONSTANTS`, `EVENT_FREQUENCY_CAPS` — single source of truth |
| `libs/domain/src/shared/constants-compat.ts` | Compatibility fallback values (mirrors constants.ts) |
| `libs/domain/src/environment/environment.config.ts` | Per-environment reputation config (`scoreMax`, `weeklyDecay`) |
| `libs/domain/src/reputation/reputation.config.ts` | Legacy helper: `calculateTier()` (numeric 1–5), `applyWeeklyDecay()` |
| `apps/chefooz-apis/src/database/entities/user-reputation-current.entity.ts` | PostgreSQL entity: current score + tier |
| `apps/chefooz-apis/src/database/entities/user-reputation-event.entity.ts` | PostgreSQL entity: event log (delta: -330 to +200) |

---

## Threshold Sources of Truth

All three files below must stay in sync. The canonical values are:

```
Bronze:  0–1,999     | Silver: 2,000–9,999
Gold: 10,000–34,999  | Diamond: 35,000–89,999
Legend: 90,000+
```

| File | Constant |
|------|---------|
| `libs/domain/src/shared/constants.ts` | `REPUTATION_CONSTANTS.BRONZE_MAX` etc. |
| `libs/domain/src/policies/reputation.policy.ts` | `TIER_SCORE_THRESHOLDS` |
| `apps/chefooz-apis/src/modules/reputation/utils/leveling.ts` | `mapScoreToLevel()` threshold checks |

**When changing thresholds: update ALL THREE files simultaneously.**

---

## Frequency Cap Implementation

Caps are checked in `ReputationService.recordEvent()` via the private method `checkEventFrequencyLimit()`:

```typescript
private async checkEventFrequencyLimit(
  userId: string,
  eventType: ReputationEventType
): Promise<{ allowed: boolean; reason?: string }>
```

- Queries `reputationEventRepository.count()` with `userId`, `eventType`, `createdAt > windowStart`
- Returns `{ allowed: false, reason: 'cap message' }` when limit exceeded
- Capped events are **silently absorbed** — `success: true` is returned but no delta is applied and no event row is inserted
- Window is `Date.now() - windowDays * 86400 * 1000`
- `FOLLOWER_MILESTONE` is centrally capped at `1 / 30 days` in addition to threshold-based triggering in the social module.

---

## Decay Implementation

### `applyDecay()` (policy.ts — stateless)
Used for point-in-time calculations.

```typescript
applyDecay(currentScore, inactivityDays, config)
```

- Threshold: 7 days inactive before decay starts
- Rate: `config.reputation.weeklyDecay` (= -250 in all envs)
- Cap: `DECAY_THRESHOLDS.MAX_DECAY_PER_WEEK = 500` pts/application
- Floor: `DECAY_THRESHOLDS.MIN_SCORE_BEFORE_DECAY_STOPS = 100`

### `applyReputationDecay()` (policy.ts — full decision)
Used by the scheduled decay job. Includes activity-count gate and structured result.

---

## `reputation.config.ts` — Legacy Numeric Tier System

This file (`libs/domain/src/reputation/reputation.config.ts`) implements a **separate, legacy 5-tier numeric system** (tiers 1–5) that divides `scoreMax` into 5 equal buckets.

- It is NOT the same as `UserTier` (Bronze/Silver/Gold/Diamond/Legend)
- `calculateTier(score, config)` returns 1–5, not a `UserTier` enum
- Named as: Novice / Explorer / Contributor / Expert / Champion
- Used only in admin analytics panels; **not used in user-facing tier display**

With `scoreMax = 1,000,000`, the numeric tier thresholds are:
| Numeric Tier | Name | Range |
|---|---|---|
| 1 | Novice | 0–199,999 |
| 2 | Explorer | 200,000–399,999 |
| 3 | Contributor | 400,000–599,999 |
| 4 | Expert | 600,000–799,999 |
| 5 | Champion | 800,000–1,000,000 |

---

## CRS Weights (Production Values)

```json
{
  "eventWeights": {
    "REVIEW_SUBMITTED":         60,
    "REEL_UPLOADED_FROM_ORDER": 100,
    "REEL_UPLOADED_GENERIC":    80,
    "ENGAGEMENT_HEALTHY":       50,
    "CONSISTENCY_WEEK":         75,
    "HYGIENE_POSITIVE":         150,
    "DELIVERY_HELPFUL":         40,
    "HELPFUL_VOTES":            50,
    "CONVERSION_INFLUENCE":     200,
    "SPAM_REPORTED":           -300,
    "BIASED_REVIEW":           -180,
    "CHEF_COMPLAINT_VALID":    -270,
    "FORCED_ENGAGEMENT":       -330,
    "LOCATION_MISMATCH":       -225,
    "FOLLOWER_MILESTONE":      "variable — see followerMilestoneRewards"
  },
  "followerMilestoneRewards": {
    "10":    100,
    "50":    200,
    "100":   400,
    "500":   600,
    "1000":  1000,
    "5000":  2000,
    "10000": 3500
  }
}
```

---

## Score Update Flow

```
User action triggers event
  → ReputationService.recordEvent(userId, eventType)
    → Chef-content gate (FSSAI check for REEL/CONSISTENCY events)
    → checkEventFrequencyLimit(userId, eventType) — return silently if capped
    → Save ReputationEvent row (delta)
    → Recalculate current score: sum all events
    → mapScoreToLevel(newScore) → newTier
    → Update ReputationCurrent row
    → Return { success: true, newScore, tierChanged }
```

---

## Known Constraints & Edge Cases

1. **Negative score prevention:** `updateCurrentScore()` uses `Math.max(0, recalculated)` — scores cannot go below 0.
2. **Platinum bug (fixed March 2026):** Old `tierOrder` array contained `'platinum'` instead of `'diamond'`+`'legend'`. This caused the tier-recovery check to silently skip Diamond/Legend users. Fixed in `updateCurrentScore()`.
3. **No event deduplication:** Two identical events at different times are both recorded. Frequency caps handle this via the rolling window.
4. **Eligibility vs award accounting:** Scheduled engagement jobs must count only `recorded=true` outcomes after cap checks, not raw eligibility scans.
5. **Batch decay:** `applyReputationDecay` is safe for batch jobs — it is stateless and does not write to DB. The job calls `recordEvent(INACTIVITY_DECAY)` after computing the delta.
6. **`mockEnvironmentConfig()` export:** Added to `environment.config.ts` to support test-file imports that previously silently failed. Returns `DEVELOPMENT_CONFIG` with optional deep-merge overrides.
7. **Follower milestone delta resolution:** `FOLLOWER_MILESTONE` is the only event type where the delta comes from `FOLLOWER_MILESTONE_REWARDS[meta.followerCount]` instead of `CRS_WEIGHTS.eventWeights`. The caller (social service background job) must pass `meta.followerCount` equal to the exact milestone threshold crossed (e.g. `100`, not `"102"`). An unrecognised threshold throws HTTP 400 listing valid thresholds.
8. **PostgreSQL enum:** `FOLLOWER_MILESTONE` was added to `reputation_event_type_enum` via migration `1778300000000`. `ADD VALUE IF NOT EXISTS` is used so the migration is idempotent.
