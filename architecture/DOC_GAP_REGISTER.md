# Documentation Gap Register

**Last Updated:** March 2026
**Maintained by:** All contributors — update this file whenever a gap is filled or a new gap is discovered.

---

## Purpose

This register tracks **known documentation gaps** in `/docs/` — features, modules, UI changes, and bug fixes that have been implemented (evidenced by entries in `application-guides/`) but whose documentation in `/docs/modules/` or `/docs/architecture/` is missing, incomplete, or outdated.

> **Rule:** When you fill a gap, mark it **RESOLVED** and record the date and the doc file created/updated.
> When you discover a new gap during a code review or fix, ADD it to this register immediately.

---

## Status Legend

| Status | Meaning |
|---|---|
| 🔴 MISSING | No doc file exists at all |
| 🟠 INCOMPLETE | Doc exists but is outdated or missing sections |
| 🟢 RESOLVED | Gap has been filled — doc created/updated |

---

## Gap Register

### Architecture & Navigation

| # | Feature | Expected Location | Status | Source Guide | Resolved Date |
|---|---|---|---|---|---|
| AG-001 | AppShell implementation (tab layout, safe area, bottom nav) | `docs/architecture/SYSTEM_OVERVIEW.md` (AppShell section) | 🔴 MISSING | `application-guides/APPSHELL_IMPLEMENTATION.md` | — |
| AG-002 | AppHeader rollout (header component across all screens) | `docs/architecture/SYSTEM_OVERVIEW.md` (AppHeader section) | 🔴 MISSING | `application-guides/APPHEADER_ROLLOUT_COMPLETE.md` | — |
| AG-003 | Tab navigation fixes (tab switch bugs, state resets) | `docs/architecture/SYSTEM_OVERVIEW.md` (Navigation section) | 🔴 MISSING | `application-guides/TAB_NAVIGATION_FIX_COMPLETE.md` | — |
| AG-004 | Theme migration Phase 1 (token system, colour overhaul) | `docs/architecture/TECH_STACK_DECISIONS.md` (UI Theme section) | ✅ RESOLVED | `application-guides/THEME_MIGRATION_PHASE1_COMPLETE.md` | 2026-03-04 |
| AG-005 | StatusBar consolidation (expo-status-bar + auto dark mode) | `docs/architecture/TECH_STACK_DECISIONS.md` (ADR-020) | ✅ RESOLVED | — | 2026-03-05 |

---

### Media Module

| # | Feature | Expected Location | Status | Source Guide | Resolved Date |
|---|---|---|---|---|---|
| AM-001 | Upload UX v2.5 (progressive upload, draft autosave) | `docs/modules/media/TECHNICAL_GUIDE.md` | 🟠 INCOMPLETE | `application-guides/UPLOAD_UX_V2_5_COMPLETE.md` | — |
| AM-002 | Upload Select screen redesign | `docs/modules/media/FEATURE_OVERVIEW.md` | 🔴 MISSING | `application-guides/UPLOAD_SELECT_REDESIGN_COMPLETE.md` | — |
| AM-003 | Camera First Edit flow (trim before upload) | `docs/modules/media/FEATURE_OVERVIEW.md` | 🔴 MISSING | `application-guides/CAMERA_FIRST_EDIT_COMPLETE.md` | — |
| AM-004 | Video trimming / WhatsApp-style trim UI | `docs/modules/media/TECHNICAL_GUIDE.md` | 🔴 MISSING | `application-guides/TRIM_WHATSAPP_STYLE_COMPLETE.md` | — |
| AM-005 | Upload flow responsive normalization | `docs/modules/media/TECHNICAL_GUIDE.md` | 🔴 MISSING | `application-guides/UPLOAD_FLOW_RESPONSIVE_NORMALIZATION_COMPLETE.md` | — |

---

### Reels Module

| # | Feature | Expected Location | Status | Source Guide | Resolved Date |
|---|---|---|---|---|---|
| AR-001 | Text mentions overlay / user tagging on reels | `docs/modules/reels/FEATURE_OVERVIEW.md` | 🔴 MISSING | `application-guides/TEXT_MENTIONS_OVERLAY_COMPLETE.md` | — |
| AR-002 | Engagement race condition fix (like/comment count desync) | `docs/modules/reels/QA_TEST_CASES.md` | 🔴 MISSING | `ENGAGEMENT_RACE_CONDITION_FIX.md` | — |
| AR-003 | Text overlay integration (stickers, text on video) | `docs/modules/reels/FEATURE_OVERVIEW.md` | 🟠 INCOMPLETE | `application-guides/TEXT_OVERLAY_INTEGRATION_COMPLETE.md` | — |

---

### Social Module

| # | Feature | Expected Location | Status | Source Guide | Resolved Date |
|---|---|---|---|---|---|
| AS-001 | Block/Unblock feature (user blocking, blocked list, visibility) | `docs/modules/social/QA_TEST_CASES.md` + `FEATURE_OVERVIEW.md` | 🔴 MISSING | `application-guides/BLOCK_UNBLOCK_IMPLEMENTATION.md` | — |
| AS-002 | User blocking — production-ready checklist | `docs/modules/social/TECHNICAL_GUIDE.md` | 🔴 MISSING | `application-guides/USER_BLOCKING_PRODUCTION_READY.md` | — |

---

### Profile / Auth Module

| # | Feature | Expected Location | Status | Source Guide | Resolved Date |
|---|---|---|---|---|---|
| AA-001 | Auth screen beautification (login/register UI redesign) | `docs/modules/auth/FEATURE_OVERVIEW.md` | 🟠 INCOMPLETE | `application-guides/AUTH_BEAUTIFICATION_COMPLETE.md` | — |
| AA-002 | Celebration modal (post-onboarding, post-order) | `docs/modules/profile/FEATURE_OVERVIEW.md` | 🔴 MISSING | `application-guides/CELEBRATION_MODAL_COMPLETE.md` | — |
| AA-003 | Username screen redesign | `docs/modules/user/FEATURE_OVERVIEW.md` | 🔴 MISSING | `application-guides/USERNAME_SCREEN_REDESIGN_COMPLETE.md` | — |

---

### Feed Module

| # | Feature | Expected Location | Status | Source Guide | Resolved Date |
|---|---|---|---|---|---|
| AF-001 | Smart polling implementation (auto-refresh, backoff, visibility) | `docs/modules/feed/TECHNICAL_GUIDE.md` | 🔴 MISSING | `application-guides/SMART_POLLING_IMPLEMENTATION_COMPLETE.md` | — |

---

### Chef Module

| # | Feature | Expected Location | Status | Source Guide | Resolved Date |
|---|---|---|---|---|---|
| AC-001 | Chef module redesign (profile page visual overhaul) | `docs/modules/chef/FEATURE_OVERVIEW.md` | 🟠 INCOMPLETE | `application-guides/CHEF_MODULE_REDESIGN_COMPLETE.md` | — |
| AC-002 | Chef orders design modernization | `docs/modules/chef-orders/FEATURE_OVERVIEW.md` | 🟠 INCOMPLETE | `application-guides/CHEF_ORDERS_DESIGN_MODERNIZATION.md` | — |

---

### Checkout Module

| # | Feature | Expected Location | Status | Source Guide | Resolved Date |
|---|---|---|---|---|---|
| ACH-001 | Checkout UI polish (payment confirmation, order summary) | `docs/modules/checkout/FEATURE_OVERVIEW.md` | 🟠 INCOMPLETE | `application-guides/CHECKOUT_UI_POLISH_SLICE_COMPLETE.md` | — |

---

### Android-Specific Fixes

| # | Feature | Expected Location | Status | Source Guide | Resolved Date |
|---|---|---|---|---|---|
| AND-001 | Android date format fix (locale-aware formatting) | `docs/modules/auth/QA_TEST_CASES.md` | 🔴 MISSING | `application-guides/ANDROID_DATE_FORMAT_COMPLETE.md` | — |
| AND-002 | Android launcher icons fix | `docs/architecture/SYSTEM_OVERVIEW.md` (Build/Deploy section) | 🔴 MISSING | `application-guides/ANDROID_LAUNCHER_ICONS_FIX.md` | — |

---

### Category / Search

| # | Feature | Expected Location | Status | Source Guide | Resolved Date |
|---|---|---|---|---|---|
| ACT-001 | Category search and navigation improvements | `docs/modules/search/FEATURE_OVERVIEW.md` | 🟠 INCOMPLETE | `application-guides/CATEGORY_SEARCH_COMPLETE.md` | — |

---

## How to Use This Register

### To fill a gap

1. Find the gap entry (e.g., `AG-001`)
2. Create or update the doc at the expected location
3. Update the gap's **Status** to `🟢 RESOLVED`
4. Fill in the **Resolved Date**
5. Commit the doc update alongside your code change

### To add a new gap

When you discover a missing doc (e.g., you're fixing a bug and notice the module has no `QA_TEST_CASES.md`):

1. Add a new row with the next available ID in the relevant section
2. Set status to `🔴 MISSING` or `🟠 INCOMPLETE`
3. Reference the relevant `application-guides/` file if it exists

---

## Stats (March 2026)

| Status | Count |
|---|---|
| 🔴 MISSING | 18 |
| 🟠 INCOMPLETE | 7 |
| 🟢 RESOLVED | 0 |
| **Total gaps** | **25** |
