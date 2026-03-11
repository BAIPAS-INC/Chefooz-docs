# Report Module — QA Test Cases

**Last Updated:** 11 March 2026

---

## Manual Test Cases

### TC-REPORT-001: Submit a report via ReportSheet

**Type:** Manual  
**Feature area:** ReportSheet component  
**Priority:** P1

**Preconditions:**
- User is logged in
- A reel is visible in the feed

**Steps:**
1. Open any reel
2. Tap the 3-dot (overflow) menu
3. Select "Report"
4. Enter a reason (≥ 5 chars)
5. Tap "Submit Report"

**Expected result:** Success message shown, report saved to MongoDB with `status: pending`  
**Status:** Ready for QA ✅

---

### TC-REPORT-002: Duplicate report prevention

**Type:** Manual  
**Feature area:** ReportSheet / backend  
**Priority:** P2

**Preconditions:**
- User has already reported reel ID `REEL123`

**Steps:**
1. Open the same reel again
2. Tap 3-dot → Report
3. Submit a report for the same reel

**Expected result:** No duplicate document in MongoDB. User still sees "success" (silent dedup).  
**Status:** Ready for QA ✅

---

### TC-REPORT-003: Admin can view all reports

**Type:** Manual  
**Feature area:** Admin portal `/dashboard/reports`  
**Priority:** P0

**Preconditions:**
- At least one report has been submitted via the mobile app
- Admin user is logged in

**Steps:**
1. Navigate to `/dashboard/reports`
2. Verify table loads with reports
3. Default filter should be "All" (all statuses)

**Expected result:** Reports table shows all submitted reports regardless of status  
**Actual result (before fix):** "No reports found" was shown if default 'pending' filter matched 0 documents  
**Fix applied:** Changed default `statusFilter` from `'pending'` to `'all'`  
**Regression test:** — (manual only)  
**Status:** Fixed ✅

---

### TC-REPORT-004: Admin "No reports found" empty state

**Type:** Manual  
**Feature area:** Admin portal `/dashboard/reports`  
**Priority:** P2

**Preconditions:**
- Admin is logged in
- No reports exist in MongoDB (empty environment)

**Steps:**
1. Navigate to `/dashboard/reports`
2. Leave filters as default ("All" / "All")

**Expected result:** Alert says "No reports have been submitted yet."  
**Status:** Fixed ✅

---

### TC-REPORT-005: Admin filter shows contextual empty state

**Type:** Manual  
**Feature area:** Admin portal `/dashboard/reports`  
**Priority:** P2

**Preconditions:**
- Admin is logged in
- Only `reviewed` reports exist in MongoDB

**Steps:**
1. Navigate to `/dashboard/reports`
2. Change Status filter to "Pending"

**Expected result:** Alert says "No reports match the selected filters. Try selecting 'All'..."  
**Status:** Fixed ✅

---

### TC-REPORT-006: Admin can take action on a report

**Type:** Manual  
**Feature area:** Admin portal `/dashboard/reports`  
**Priority:** P1

**Preconditions:**
- A `pending` report exists

**Steps:**
1. Navigate to `/dashboard/reports`
2. Find a report with status "pending"
3. Click "Take Action"

**Expected result:** Report status updates to `action_taken`, row updates immediately (optimistic or refetch)  
**Status:** Ready for QA ✅

---

### TC-REPORT-007: Report submission from old ReportScreen — V1 format bug

**Type:** Bug Regression  
**Feature area:** `apps/chefooz-app/src/app/report/` (legacy screen)  
**Priority:** P1

**Preconditions:**
- User navigates directly to the /report path (deep link or test)

**Steps:**
1. Open the legacy report submission screen
2. Select target type "User / Profile"
3. Enter a target ID
4. Enter a reason (≥ 10 chars)
5. Tap "Submit Report"

**Expected result:** Report is created in MongoDB with `targetType: 'user'`  
**Actual result (before fix):** V2 backend rejected the request (400) because old screen sent `{category, message, profileId}` (V1 format). Report was silently lost.  
**Fix applied:** `handleSubmit` now sends V2 format `{targetType, targetId, reason}`. Radio button options updated from `['reel', 'profile', 'message', 'review']` to `['reel', 'user', 'comment']`.  
**Status:** Fixed ✅

---

## Known Issues / Edge Cases

- The AdminLayout on reports page previously showed title "Reviews" — fixed to "Reports" (March 2026)
- Legacy V1 report controller (`/api/v1/report`) still exists in codebase for backward compatibility but is NOT shown in admin portal
- Rate limiting for report submissions is in-memory only (process-scoped); resets on server restart — acceptable for current scale
