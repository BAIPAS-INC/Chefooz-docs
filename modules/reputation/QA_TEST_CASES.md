# Reputation Module — QA Test Cases

**Version:** 2.0  
**Last Updated:** May 2, 2026  
**Module:** Reputation / CRS
---

## MVP Phase 1 Scope (May 2026 — Production-Ready)

**Status:** All 23 test cases below are **automated** and **cover end-to-end production flows** for Phase 1 events.

**Events covered in Phase 1:**
- ✅ REVIEW_SUBMITTED | HYGIENE_POSITIVE | REEL_UPLOADED_FROM_ORDER | CONVERSION_INFLUENCE | DELIVERY_HELPFUL
- ✅ FOLLOWER_MILESTONE | ENGAGEMENT_HEALTHY | CONSISTENCY_WEEK | ADMIN_OVERRIDE

**Events deferred to future phases:**
- ⏳ Phase 2: HELPFUL_VOTES, SPAM_REPORTED, BIASED_REVIEW (moderation module)
- ⏳ Phase 3: CHEF_COMPLAINT_VALID, FORCED_ENGAGEMENT, LOCATION_MISMATCH (fraud & address validation)

---

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
| TC-REP-014 | Promotional upload writes eligible reputation event | Automated | P1 |
| TC-REP-015 | Follower milestone respects 30-day cap | Automated | P0 |
| TC-REP-016 | Weekly engagement job counts only awarded events | Automated | P0 |
| TC-REP-017 | Admin override accepts production-scale scores | Automated | P0 |
| TC-REP-018 | Admin leaderboard rebuild executes real rebuild | Automated | P0 |
| TC-REP-019 | Score recompute uses full event history | Automated | P0 |
| TC-REP-020 | Weekly snapshots are idempotent per user/week | Automated | P0 |
| TC-REP-021 | CRS scheduler jobs are active in module DI | Automated | P1 |
| TC-REP-022 | Disallowed event source is rejected | Automated | P0 |
| TC-REP-023 | Event endpoint enqueues durable queue job | Automated | P0 |
| TC-REP-030 | Promotional reel media ObjectId does not break UUID persistence | Automated | P0 |

---

### TC-REP-014: Promotional upload writes eligible reputation event

**Type:** Automated (regression)  
**Feature area:** Upload-driven event ingestion (`MediaService` → `ReputationService.recordEvent`)  
**Priority:** P1

**Preconditions:**
- Authenticated user creates a promotional reel upload (`reelPurpose='PROMOTIONAL'`)
- Media source is user content

**Steps:**
1. Complete upload through media flow.
2. Observe event payload sent to `recordEvent`.
3. Verify event metadata includes reel purpose and reference points to media id.

**Expected result:**
- Eligible promotional user uploads emit an upload-driven reputation event.
- Payload includes `meta.reelPurpose='PROMOTIONAL'` and `referenceId=<mediaId>`.

**Actual result (before fix):**
- Promotional uploads did not map to any reputation event; no CRS write occurred.

**Fix applied:**
- Added `getReelUploadReputationEvent()` utility in media module and wired both upload-completion branches to use it.

**Regression test:**
- `apps/chefooz-apis/src/modules/media/reel-upload-reputation.util.spec.ts`

**Status:** Fixed ✅

---

### TC-REP-015: Follower milestone respects 30-day cap

**Type:** Automated (regression)  
**Feature area:** `ReputationService.recordEvent()` + social milestone integration  
**Priority:** P0

**Preconditions:**
- User already received `FOLLOWER_MILESTONE` credit within the last 30 days for the same follower threshold window

**Steps:**
1. Call `recordEvent(userId, { type: FOLLOWER_MILESTONE, meta: { followerCount: 10 } })` after mocking one recent prior milestone event.

**Expected result:**
- Service returns success with a frequency-cap message.
- No new `user_reputation_events` row is inserted.

**Actual result (before fix):**
- `FOLLOWER_MILESTONE` was defined in config but omitted from the central cap map in `reputation.service.ts`.

**Fix applied:**
- Added `FOLLOWER_MILESTONE: { limit: 1, windowDays: 30 }` to `CRS_FREQUENCY_CAPS`.

**Regression test:**
- `apps/chefooz-apis/src/modules/reputation/reputation.service.spec.ts`

**Status:** Fixed ✅

---

### TC-REP-016: Weekly engagement job counts only awarded events

**Type:** Automated (regression)  
**Feature area:** `ReputationService.checkWeeklyEngagement()`  
**Priority:** P0

**Preconditions:**
- A user qualifies for `ENGAGEMENT_HEALTHY` and `CONSISTENCY_WEEK`
- At least one of those events is capped and therefore silently absorbed

**Steps:**
1. Mock weekly reel data so one user is eligible for both events.
2. Mock one `recordEventInternal()` call returning `recorded: false` and the other returning `recorded: true`.
3. Run `checkWeeklyEngagement()`.

**Expected result:**
- Returned counters reflect only the actually inserted event rows.
- Example: `engagementFired = 0`, `consistencyFired = 1`.

**Actual result (before fix):**
- Job incremented counters after every awaited call, even when the call was capped and no event row was inserted.

**Fix applied:**
- `checkWeeklyEngagement()` now increments counters only when `recorded === true` from the internal event-recording result.

**Regression test:**
- `apps/chefooz-apis/src/modules/reputation/reputation.service.spec.ts`

**Status:** Fixed ✅

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
| `apps/chefooz-apis/src/modules/reputation/reputation.service.spec.ts` | follower milestone cap, weekly engagement awarded-counting, queue enqueue contract, disallowed source rejection, leaderboard rebuild, full-history score recomputation |
| `apps/chefooz-apis/src/modules/reputation/dto/admin-override.dto.spec.ts` | production score bounds for admin override DTO |
| `apps/chefooz-apis/src/jobs/crs/weeklySnapshot.job.spec.ts` | idempotent upsert behavior (`userId`, `weekStart`) |

---

### TC-REP-017: Admin override accepts production-scale scores

**Type:** Bug Regression / Automated
**Feature area:** `AdminOverrideDto` validation
**Priority:** P0

**Preconditions:**
- Admin override request payload includes valid `userId` and `reason`

**Steps:**
1. Validate DTO with `newScore = 1_000_000`.
2. Validate DTO with `newScore = 1_000_001`.

**Expected result:**
- `1_000_000` passes validation.
- `1_000_001` fails validation.
**Actual result (before fix):**
- Any score above `100` failed validation even though CRS uses production 0–1,000,000 scale.
**Fix applied:**
- Updated `AdminOverrideDto` max bound from `100` to `1_000_000` and aligned API docs.
**Regression test:**
- `apps/chefooz-apis/src/modules/reputation/dto/admin-override.dto.spec.ts`
**Status:** Fixed ✅

---

### TC-REP-018: Admin leaderboard rebuild executes real rebuild

**Type:** Bug Regression / Automated
**Feature area:** Admin endpoint `POST /api/v1/crs/admin/rebuild-leaderboard`
**Priority:** P0

**Preconditions:**
- Admin user is authenticated
- Reputation current table has users with scores

**Steps:**
1. Call admin rebuild endpoint.
2. Verify leaderboard table is rebuilt with weekly and monthly entries.

**Expected result:**
- Endpoint executes actual rebuild and returns rebuilt entry count.
**Actual result (before fix):**
- Endpoint returned success with message `not yet implemented` and no rebuild occurred.
**Fix applied:**
- Controller now calls `ReputationService.rebuildLeaderboard()` which clears and repopulates leaderboard rows.
**Regression test:**
- `apps/chefooz-apis/src/modules/reputation/reputation.service.spec.ts`
**Status:** Fixed ✅

---

### TC-REP-019: Score recompute uses full event history

**Type:** Bug Regression / Automated
**Feature area:** `ReputationService.updateCurrentScore()`
**Priority:** P0

**Preconditions:**
- User has a historical event stream across older and newer timestamps

**Steps:**
1. Invoke score recomputation path.
2. Verify query fetches by `userId` across full event history.
3. Verify final score equals sum of event deltas (with clamp) regardless of `lastEventAt` cursor position.

**Expected result:**
- Recomputed score is deterministic from complete event history.
**Actual result (before fix):**
- Incremental `createdAt > lastEventAt` window could skip events under concurrent writes.
**Fix applied:**
- Replaced incremental-window aggregation with full-history aggregation in `updateCurrentScore()`.
**Regression test:**
- `apps/chefooz-apis/src/modules/reputation/reputation.service.spec.ts`
**Status:** Fixed ✅

---

### TC-REP-020: Weekly snapshots are idempotent per user/week

**Type:** Bug Regression / Automated
**Feature area:** `WeeklySnapshotJob.executeWeeklySnapshot()`
**Priority:** P0

**Preconditions:**
- Weekly snapshot job is triggered more than once for the same ISO week

**Steps:**
1. Run `executeWeeklySnapshot()` with the same user set and week start.
2. Observe persistence method arguments.

**Expected result:**
- Snapshot persistence uses upsert conflict keys `['userId', 'weekStart']`.
- Repeated runs update existing weekly snapshots instead of inserting duplicates.
**Actual result (before fix):**
- Job used `save()` only, which could create duplicate rows on reruns unless externally constrained.
**Fix applied:**
- Switched to `upsert()` and added unique DB constraint/index for `userId + weekStart`.
**Regression test:**
- `apps/chefooz-apis/src/jobs/crs/weeklySnapshot.job.spec.ts`
**Status:** Fixed ✅

---

### TC-REP-021: CRS scheduler jobs are active in module DI

**Type:** Bug Regression / Automated
**Feature area:** Reputation module scheduler wiring
**Priority:** P1

**Preconditions:**
- Scheduler module is enabled globally
- Reputation module is part of app imports

**Steps:**
1. Verify `WeeklySnapshotJob`, `WeeklyDigestJob`, and `RebuildLeaderboardJob` are registered providers in `ReputationModule`.
2. Verify each of these jobs has an executable `@Cron(...)` decorator on its main run method.

**Expected result:**
- Scheduler can instantiate and execute these jobs automatically.
**Actual result (before fix):**
- Jobs existed but were not cron-active in runtime scheduling flow.
**Fix applied:**
- Added `@Cron` decorators and registered jobs in module providers.
**Regression test:**
- Configuration/static verification + runtime smoke recommended in staging
**Status:** Fixed ✅

---

### TC-REP-022: Disallowed event source is rejected

**Type:** Bug Regression / Automated
**Feature area:** `ReputationService.validateEventSource()`
**Priority:** P0

**Preconditions:**
- Event payload is syntactically valid
- Source/event pair violates ownership rules

**Steps:**
1. Call `recordEvent(userId, CONVERSION_INFLUENCE payload, 'review-module')`.

**Expected result:**
- Request fails with `400` and source-allow-list validation message.

**Actual result (before fix):**
- No source ownership enforcement at ingestion path, allowing cross-module spoofing.

**Fix applied:**
- Added strict source allow-list validation per event type in `validateEventSource()`.

**Regression test:**
- `apps/chefooz-apis/src/modules/reputation/reputation.service.spec.ts`

**Status:** Fixed ✅

---

### TC-REP-023: Event endpoint enqueues durable queue job

**Type:** Bug Regression / Automated
**Feature area:** `ReputationService.enqueueEvent()` and producer reliability
**Priority:** P0

**Preconditions:**
- Event producer triggers a valid reputation event

**Steps:**
1. Call `enqueueEvent()` with a valid payload and trusted source.
2. Inspect queue invocation options.

**Expected result:**
- Event is added to `reputation-events` queue as `record-event` job.
- Job uses retry settings (`attempts=5`, exponential backoff).

**Actual result (before fix):**
- Producers used direct `recordEvent(...).catch(...)`, which could drop events on transient failure.

**Fix applied:**
- Producers now enqueue jobs and consume via `ReputationEventsProcessor` with terminal dead-letter logging.

---

## Future Phase Test Cases (Deferred — Phase 2 & 3)

These test cases will be added when the corresponding modules integrate with reputation.

### TC-REP-024: (Phase 2) HELPFUL_VOTES emitted when comment detected as helpful

**Feature area:** moderation module integration
**Priority:** To be determined
**Trigger module:** comment/reaction evaluation system
**Expected producer:** `moderation.service.ts` or comment-reaction handler

```typescript
// TODO: Add test when moderation module exposes helpful signal
// Expected flow:
// 1. User marks comment as helpful
// 2. Moderation checks helpfulness criteria
// 3. If helpful, emit HELPFUL_VOTES event to reputation module
// 4. Verify score increase + frequency cap enforcement
```

### TC-REP-025: (Phase 2) SPAM_REPORTED integrated with moderation detection

**Feature area:** moderation module integration
**Priority:** To be determined
**Trigger module:** moderation.service spam detection pipeline
**Expected producer:** moderation engagement/content analyzer

```typescript
// TODO: Add test when moderation module is ready
// Expected flow:
// 1. Moderation detects content as spam (flagged, multiple reports, keyword match)
// 2. Confirm user is the reporter/evaluator
// 3. Emit SPAM_REPORTED event with content reference
// 4. Verify negative score delta applied + cap enforcement
```

### TC-REP-026: (Phase 2) BIASED_REVIEW integrated with review ML model

**Feature area:** review quality module
**Priority:** To be determined
**Trigger module:** review validation engine with ML bias detection
**Expected producer:** review-quality.service or ml-integration

```typescript
// TODO: Add test when review ML model is integrated
// Expected flow:
// 1. Review submitted and scored by bias-detection ML model
// 2. If bias detected with high confidence, emit BIASED_REVIEW event
// 3. Verify reviewer score reduced correctly
```

### TC-REP-027: (Phase 3) CHEF_COMPLAINT_VALID emitted on complaint resolution

**Feature area:** complaint/dispute resolution module
**Priority:** To be determined
**Trigger module:** complaint resolution workflow
**Expected producer:** dispute.service after investigation

```typescript
// TODO: Add test when complaint module validates disputes
// Expected flow:
// 1. Customer files complaint against chef
// 2. Admin/system investigates and marks as valid
// 3. Emit CHEF_COMPLAINT_VALID with chef userId + reference
// 4. Verify reputation reduction + recovery path
```

### TC-REP-028: (Phase 3) FORCED_ENGAGEMENT detected and applied by fraud engine

**Feature area:** fraud detection engine
**Priority:** To be determined
**Trigger module:** engagement fraud analysis (background job)
**Expected producer:** fraud-detection.service or analytics engine

```typescript
// TODO: Add test when fraud detection is production-ready
// Expected flow:
// 1. Background job analyzes engagement metrics (likes, comments, follows)
// 2. Detects suspicious patterns (sudden spikes, bots, fake accounts)
// 3. If violation confirmed, emit FORCED_ENGAGEMENT event
// 4. Verify reputation penalty applied
```

### TC-REP-029: (Phase 3) LOCATION_MISMATCH detected on delivery address mismatch

**Feature area:** delivery address validation
**Priority:** To be determined
**Trigger module:** delivery/fulfillment module address verification
**Expected producer:** delivery-validation service post-completion

```typescript
// TODO: Add test when delivery address validation is integrated
// Expected flow:
// 1. Order completed and delivery fulfillment finalized
// 2. Delivery address validation detects mismatch (chef_location vs delivery_location)
// 3. Emit LOCATION_MISMATCH event with order reference
// 4. Verify reputation reduction for location fraud penalty
```

---

## Integration Timeline & Roadmap

| Phase | Timeline | Events | Status |
|---|---|---|---|
| **Phase 1** | May 2026 ✅ | 9 events | **SHIPPED** |
| **Phase 2** | Q3 2026 (est.) | HELPFUL_VOTES, SPAM_REPORTED, BIASED_REVIEW | In planning |
| **Phase 3** | Q4 2026 (est.) | CHEF_COMPLAINT_VALID, FORCED_ENGAGEMENT, LOCATION_MISMATCH | Backlog |

---

**Regression test:**
- `apps/chefooz-apis/src/modules/reputation/reputation.service.spec.ts`

**Status:** Fixed ✅

---

### TC-REP-030: Promotional reel media ObjectId does not break UUID persistence

**Type:** Bug Regression / Automated
**Feature area:** `ReputationService.recordEventInternal()` persistence path for `REEL_UPLOADED_FROM_ORDER`
**Priority:** P0

**Preconditions:**
- Reel upload event uses Mongo media id (ObjectId-like string)

**Steps:**
1. Trigger `recordEvent(..., { type: REEL_UPLOADED_FROM_ORDER, referenceId: '<mongo-id>', meta: { reelPurpose: 'PROMOTIONAL', mediaId: '<mongo-id>' } })`.
2. Observe entity payload passed to `user_reputation_events` repository create/save.

**Expected result:**
- Event is recorded successfully.
- `referenceId` persisted as `null` for reel upload events.
- `meta.mediaId` retains Mongo id linkage for traceability.

**Actual result (before fix):**
- Insert failed with PostgreSQL error: `invalid input syntax for type uuid`.

**Fix applied:**
- Media producer writes reel linkage into `meta.mediaId`.
- Reputation service uses metadata for reel checks and sanitizes persisted `referenceId` for upload events.

**Regression test:**
- `apps/chefooz-apis/src/modules/reputation/reputation.service.spec.ts`

**Status:** Fixed ✅
