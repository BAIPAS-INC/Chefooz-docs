# Reputation Module — Technical Guide

**Version:** 2.0 (Production-Grade Scale)
**Last Updated:** March 2026

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
4. **Batch decay:** `applyReputationDecay` is safe for batch jobs — it is stateless and does not write to DB. The job calls `recordEvent(INACTIVITY_DECAY)` after computing the delta.
5. **`mockEnvironmentConfig()` export:** Added to `environment.config.ts` to support test-file imports that previously silently failed. Returns `DEVELOPMENT_CONFIG` with optional deep-merge overrides.
6. **Follower milestone delta resolution:** `FOLLOWER_MILESTONE` is the only event type where the delta comes from `FOLLOWER_MILESTONE_REWARDS[meta.followerCount]` instead of `CRS_WEIGHTS.eventWeights`. The caller (social service background job) must pass `meta.followerCount` equal to the exact milestone threshold crossed (e.g. `100`, not `"102"`). An unrecognised threshold throws HTTP 400 listing valid thresholds.
7. **PostgreSQL enum:** `FOLLOWER_MILESTONE` was added to `reputation_event_type_enum` via migration `1778300000000`. `ADD VALUE IF NOT EXISTS` is used so the migration is idempotent.
