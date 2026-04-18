# Reputation Module — QA Test Cases

**Version:** 2.0  
**Last Updated:** March 2026  
**Module:** Reputation / CRS

---

## Test Case Index

| ID | Title | Type | Priority |
|----|-------|------|----------|
| TC-REP-001 | Bronze user earns Silver after 4 weeks | Manual | P1 |
| TC-REP-002 | Frequency cap prevents review farming | Automated | P0 |
| TC-REP-003 | Weekly decay reduces score after 7+ days inactive | Automated | P0 |
| TC-REP-004 | Score does not decay below 100 | Automated | P0 |
| TC-REP-005 | Promotion cooldown blocks rapid tier jumps | Automated | P1 |
| TC-REP-006 | Demotion grace period allows score recovery | Automated | P1 |
| TC-REP-007 | Negative events reduce score correctly | Automated | P0 |
| TC-REP-008 | Diamond tier cannot be reached in less than 1 year | Manual | P1 |
| TC-REP-009 | Capped event silently absorbed (no error shown) | Manual | P2 |
| TC-REP-010 | BUG REGRESSION: platinum tier bug (fixed March 2026) | Automated | P0 |
| TC-REP-011 | Follower milestone awards correct points | Automated | P1 |
| TC-REP-012 | Follow/unfollow gaming blocked by 30-day cap | Automated | P0 |
| TC-REP-013 | Missing meta.followerCount returns 400 | Automated | P1 |

---

### TC-REP-001: Bronze user earns Silver after ~4 weeks

**Type:** Manual  
**Feature area:** Reputation score accumulation  
**Priority:** P1

**Preconditions:**
- User starts at Bronze (score = 0)
- User is active: 2 reviews/day + 1 engagement/day + 1 delivery credit/day ≈ 200 pts/day

**Steps:**
1. Submit 2 reviews per day for 10 days
2. Record `ENGAGEMENT_HEALTHY` once per day
3. Record `DELIVERY_HELPFUL` once per day
4. Check score after 10 days

**Expected result:** Score ~2,000+ → Silver tier reached  
**Notes:** 200 pts/day × 10 days = 2,000. Frequency caps apply (2 reviews/day max credited).  
**Status:** Verified ✅

---

### TC-REP-002: Frequency cap prevents review farming

**Type:** Automated (regression)  
**Feature area:** `ReputationService.recordEvent()`  
**Priority:** P0

**Preconditions:** User at Bronze tier

**Steps:**
1. Call `recordEvent(userId, REVIEW_SUBMITTED)` 5 times within the same day

**Expected result:**
- First 2 calls: `{ success: true, pointsAwarded: 60 }` each
- Calls 3–5: `{ success: true, pointsAwarded: 0, message: 'frequency cap reached' }`
- Score increases by only 120 pts total (not 300)

**Actual result (before fix):** No cap existed — 5 calls × 8 pts = 40 pts (also wrong weight)  
**Fix applied:** Added `CRS_FREQUENCY_CAPS` constant + `checkEventFrequencyLimit()` method in `reputation.service.ts`  
**Regression test:** `apps/chefooz-apis/src/modules/reputation/reputation.service.spec.ts`  
**Status:** Fixed ✅

---

### TC-REP-003: Weekly decay reduces score after 7+ days inactive

**Type:** Automated  
**Feature area:** `applyReputationDecay()`  
**Priority:** P0

**Preconditions:** User with score 10,000 (Gold tier), no activity for 14 days

**Steps:**
1. Calculate `applyReputationDecay({ currentScore: 10000, lastActiveAt: 14daysAgo, recentActivityCount: 0 })`

**Expected result:** `{ shouldDecay: true, newScore: 9500, decayAmount: 500 }` (250/week × 2 weeks)  
**Status:** Verified ✅

---

### TC-REP-004: Score does not decay below 100

**Type:** Automated  
**Feature area:** `applyDecay()` minimum floor  
**Priority:** P0

**Preconditions:** User with score 100

**Steps:**
1. Apply 30 days of inactive decay

**Expected result:** `shouldDecay: false`, `newScore: 100` — decay stops exactly at the floor  
**Status:** Verified ✅

---

### TC-REP-005: Promotion cooldown blocks rapid tier jumps

**Type:** Automated  
**Feature area:** `shouldPromoteTier()`  
**Priority:** P1

**Preconditions:** User at Bronze tier, last promotion 10 days ago, current score 2,100

**Steps:**
1. Call `shouldPromoteTier(2100, BRONZE, 10daysAgo, config)`

**Expected result:** `{ allowed: false, errorCode: 'TIER_LOCKED', reason contains 'cooldown' }`  
**Notes:** 14-day cooldown, 10 days elapsed → blocked  
**Status:** Verified ✅

---

### TC-REP-006: Demotion grace period allows score recovery

**Type:** Automated  
**Feature area:** `shouldDemoteTier()`  
**Priority:** P1

**Preconditions:** User at Silver tier (score dropped to 500), score dropped 2 days ago

**Steps:**
1. Call `shouldDemoteTier(500, SILVER, 2daysAgo, config)`

**Expected result:** `{ allowed: false, reason contains 'Grace period' }`  
**Notes:** 3-day grace period — 2 days elapsed means still protected  
**Status:** Verified ✅

---

### TC-REP-007: Negative events reduce score correctly

**Type:** Automated  
**Feature area:** `CRS_WEIGHTS` negative entries  
**Priority:** P0

**Preconditions:** User with score 5,000

**Steps:**
1. Record `SPAM_REPORTED` event
2. Check new score

**Expected result:** `5,000 - 300 = 4,700`  
**Status:** Verified ✅

---

### TC-REP-008: Diamond tier not reachable in under 1 year

**Type:** Manual (simulation)  
**Feature area:** CRS balance overall  
**Priority:** P1

**Preconditions:** Fresh user starting at 0

**Steps:**
1. Simulate maximum legitimate daily earning: 2 reviews (120) + 1 engagement (50) + 5 delivery (200) + 2 helpful votes (100) + 2 conversion (400) = 870 pts/day
2. Apply 14-day cooldown between tiers
3. Calculate weeks to Diamond (35,000 pts)

**Expected result:** Even at maximum earning rate, Diamond requires 40+ days minimum (35,000 / 870 ≈ 40 days). At realistic rates (~500 pts/week), ~70 weeks (~16 months)  
**Status:** Verified ✅

---

### TC-REP-009: Capped event silently absorbed (no error shown)

**Type:** Manual  
**Feature area:** Frequency cap UX  
**Priority:** P2

**Preconditions:** User has already submitted 2 reviews today

**Steps:**
1. User submits a 3rd review
2. Observe API response
3. Observe user-facing feedback

**Expected result:**
- API returns `{ success: true, message: 'Event frequency cap reached', pointsAwarded: 0 }`
- No error toast shown to user
- Review is still published (review creation is separate from reputation event)

**Status:** Verified ✅

---

### TC-REP-010: BUG REGRESSION — platinum tier fix (March 2026)

**Type:** Automated (bug regression)  
**Feature area:** `ReputationService.updateCurrentScore()` tierOrder  
**Priority:** P0

**Preconditions:** User at Diamond tier (score ≥ 35,000)

**Steps:**
1. Record a score that is in the "recovery buffer" (score just above Diamond min after a dip)
2. Call `updateCurrentScore()` internally

**Expected result:** Tier correctly identified as `diamond` or `legend` — no silent fallback to `bronze`

**Actual result (before fix):** `tierOrder` array contained `'platinum'` which is not a valid `UserTier`. Diamond/Legend users were skipped in the old tier-recovery loop, causing incorrect tier downgrades.

**Fix applied:** Replaced `['bronze', 'silver', 'gold', 'platinum']` with `['bronze', 'silver', 'gold', 'diamond', 'legend']` in `updateCurrentScore()`. Also fixed hardcoded threshold map that used `platinum` key and old 0–100 values.

**Regression test:** `apps/chefooz-apis/src/modules/reputation/reputation.service.spec.ts` — test: "should correctly identify Diamond tier in updateCurrentScore"  
**Status:** Fixed ✅

---

---

### TC-REP-011: Follower milestone awards correct points

**Type:** Automated  
**Feature area:** `ReputationService.recordEvent()` — FOLLOWER_MILESTONE  
**Priority:** P1

**Preconditions:** User at Bronze tier, score = 500

**Steps:**
1. Fire `recordEvent(userId, { type: 'FOLLOWER_MILESTONE', meta: { followerCount: 100 } })`
2. Check new score

**Expected result:** `score = 500 + 400 = 900`  
**Status:** Verified ✅

---

### TC-REP-012: Follow/unfollow gaming blocked by 30-day cap

**Type:** Automated  
**Feature area:** Frequency cap — FOLLOWER_MILESTONE  
**Priority:** P0

**Preconditions:** User already received a FOLLOWER_MILESTONE credit 10 days ago

**Steps:**
1. Fire another `FOLLOWER_MILESTONE` event (e.g. threshold 500)

**Expected result:** `{ success: true, pointsAwarded: 0, message: 'frequency cap reached' }` — no delta applied  
**Status:** Verified ✅

---

### TC-REP-013: Missing meta.followerCount returns 400

**Type:** Automated  
**Feature area:** `recordEvent` validation  
**Priority:** P1

**Steps:**
1. Fire `recordEvent(userId, { type: 'FOLLOWER_MILESTONE' })` with no meta

**Expected result:** HTTP 400 — `FOLLOWER_MILESTONE requires meta.followerCount`  
**Status:** Verified ✅

| File | Coverage |
|------|---------|
| `libs/domain/src/policies/reputation.policy.spec.ts` | shouldPromoteTier, shouldDemoteTier, applyDecay, applyReputationDecay, getTierForScore, getScoreRangeForTier |
| `libs/domain/src/reputation/reputation.config.spec.ts` | isValidScore, clampScore, applyWeeklyDecay, calculateTier (numeric), integration decay tests |
| `apps/chefooz-apis/src/modules/reputation/reputation.service.spec.ts` | recordEvent, frequency caps, updateCurrentScore, platinum-bug regression |
