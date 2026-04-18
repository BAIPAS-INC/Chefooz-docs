# Reputation Module — Feature Overview

**Version:** 2.0 (Production-Grade Scale)
**Last Updated:** March 2026
**Module:** `apps/chefooz-apis/src/modules/reputation/`
**Domain Logic:** `libs/domain/src/policies/reputation.policy.ts`, `libs/domain/src/reputation/reputation.config.ts`
**Shared Types:** `libs/types/src/lib/reputation.types.ts`

---

## What It Does

The **Customer Reputation Score (CRS)** system tracks the trustworthiness, engagement quality, and overall conduct of every user (customer) on the Chefooz platform.

It is **not a gamification badge** — it is a production-grade trust signal used to:
- Gate features (e.g. order attribution, feed visibility boosts)
- Determine rider assignment priority
- Influence feed ranking for chefs whose customers are highly trusted
- Detect spam or fraudulent activity patterns

---

## Tier System

| Tier | Score Range | Estimated Time to Reach |
|------|-------------|--------------------------|
| Bronze | 0 – 1,999 | Starting tier |
| Silver | 2,000 – 9,999 | ~4 weeks (casual user) |
| Gold | 10,000 – 34,999 | ~7 months |
| Diamond | 35,000 – 89,999 | ~2 years |
| Legend | 90,000+ | ~5 years |

> "Casual user" = ~510 pts/week: 2 reviews + 1 engagement + 1 delivery credit

---

## How Points Are Earned

| Event | Points | Frequency Cap |
|-------|--------|---------------|
| `REVIEW_SUBMITTED` | +60 | 2/day |
| `REEL_UPLOADED_FROM_ORDER` | +100 | 2/week |
| `ENGAGEMENT_HEALTHY` | +50 | 1/day |
| `CONSISTENCY_WEEK` | +75 | 1/7 days |
| `HYGIENE_POSITIVE` | +150 | 1/7 days |
| `DELIVERY_HELPFUL` | +40 | 5/day |
| `HELPFUL_VOTES` | +50 | 2/day |
| `CONVERSION_INFLUENCE` | +200 | 2/day |

### Follower Milestone Rewards

Social influence (follower count) is woven into CRS via one-time milestone events fired by the social module when a user first crosses each threshold.

| Follower milestone | Points | Frequency cap |
|-------------------|--------|---------------|
| 10 followers | +100 | 1/30 days |
| 50 followers | +200 | 1/30 days |
| 100 followers | +400 | 1/30 days |
| 500 followers | +600 | 1/30 days |
| 1,000 followers | +1,000 | 1/30 days |
| 5,000 followers | +2,000 | 1/30 days |
| 10,000 followers | +3,500 | 1/30 days |

> The 30-day frequency cap prevents follow/unfollow gaming. In normal usage each milestone fires exactly once.

### Negative Events (No Caps — Trust Penalties)

| Event | Points |
|-------|--------|
| `SPAM_REPORTED` | -300 |
| `BIASED_REVIEW` | -180 |
| `CHEF_COMPLAINT_VALID` | -270 |
| `FORCED_ENGAGEMENT` | -330 |
| `LOCATION_MISMATCH` | -225 |

---

## Weekly Decay

Users who go **7+ consecutive days without activity** lose **250 pts/week**.

- Minimum floor: **100 pts** (decay stops at this point)
- Maximum decay rate: **500 pts/week** (even long inactive periods are capped)
- At Gold (10,000 pts): ~40 weeks of inactivity to reach Bronze floor
- At Diamond (35,000 pts): ~140 weeks (~2.7 years) to reach Bronze

Decay is a signal of inactivity, not a punishment system. Active users never decay.

---

## Frequency Caps (Anti-Farming)

All positive events are rate-limited by a rolling-window frequency cap enforced in the backend service. Capped events are silently absorbed (no error returned to the user) — the system records the attempt but does not award points.

---

## Tier Change Rules

- **Promotion cooldown:** 14 days between promotions (prevents quick resets)
- **Demotion grace period:** 3 days (user has a window to recover before being demoted)
- **Chef-content gate:** `REEL_UPLOADED_FROM_ORDER` and `CONSISTENCY_WEEK` only apply to chef contexts (FSSAI verified)

---

## Feed & Feature Impact

| Tier | Feed Boost | Coin Multiplier |
|------|-----------|-----------------|
| Bronze | 0% | ×1.0 |
| Silver | +0.5 | ×1.05 |
| Gold | +1.0 | ×1.10 |
| Diamond | +1.5 | ×1.20 |
| Legend | +2.0 | ×1.30 |
