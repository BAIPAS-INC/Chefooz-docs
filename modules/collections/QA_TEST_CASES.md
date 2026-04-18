# Collections Module — QA Test Cases

**Module**: `apps/chefooz-apis/src/modules/collections`  
**Frontend**: `apps/chefooz-app/src/app/(tabs)/profile.tsx` (Saved tab)  
**Components**: `apps/chefooz-app/src/components/collections/CollectionCard.tsx`  
**Last Updated**: 2026-04-18

---

## Bug Regression Tests

### TC-COLLECTIONS-BUG-001: Saved tab shows collections with thumbnail previews

**Type:** UI Enhancement + Bug Regression  
**Feature area:** Profile → Saved tab  
**Priority:** P1

**Preconditions:**
- User is logged in
- User has at least one named collection with saved reels

**Steps:**
1. Navigate to Profile
2. Tap the **Saved** tab
3. Observe the collection grid

**Expected result:**
- Two-column grid of collection cards is shown (not a flat reel grid)
- Each collection card shows:
  - If 4+ reels: 2×2 thumbnail grid
  - If 1–3 reels: single full-size thumbnail
  - If 0 reels: emoji or folder icon
- Collection name and item count shown below the preview area
- Long-press reveals Edit / Delete options

**Actual result (before fix):** Saved tab showed individual saved reels in a 3-column flat grid instead of the collection cards view. No collection names or grouping was visible.

**Root cause:** `profile.tsx` used `useSavedReels()` + flat `FlatList(numColumns=3)`. The `CollectionCard` component existed but was never wired to the Saved tab. Backend `getMyCollections()` returned no `previewUrls`.

**Fix applied:**
1. Backend `getMyCollections()` — batch-fetches up to 4 thumbnail URLs per collection from MongoDB using the same 3-path lookup as `getSavedReels` (primary ObjectId, ObjectId fallback, UUID fallback)
2. `CollectionResponseDto` + `Collection` interface — added `previewUrls: string[]`
3. `CollectionCard.tsx` — redesigned with conditional render: 2×2 grid (4 previews), single image (1–3), emoji/icon fallback (0)
4. `profile.tsx` Saved tab — swapped `useSavedReels` → `useMyCollections`; FlatList now renders `CollectionCard` at 2 columns

**Regression test:** Manual — verify collections with thumbnails appear in 2-column grid  
**Status:** Fixed ✅

---

### TC-COLLECTIONS-BUG-002: Saved tab old UUID saves get thumbnails

**Type:** Bug Regression  
**Feature area:** `getSavedReels` backend, Saved tab  
**Priority:** P1

**Preconditions:**
- User has saved reels that were saved before the `ReelCard.tsx` mediaId fix (UUIDs stored as mediaId)

**Steps:**
1. Navigate to Profile → Saved tab (legacy reel grid, now replaced by collections)

**Expected result:** N/A (Saved tab now shows collections, not individual saved reels).  
The `previewUrls` in collections follow the same 3-path lookup and handle UUID-format items.  
For `getSavedReels` (used by collection item screens): thumbnails still resolve via UUID fallback.

**Fix applied:** See TC-COLLECTIONS-BUG-001 for `getMyCollections`. `getSavedReels` already has UUID fallback from previous fix.  
**Status:** Fixed ✅

---

## Test Coverage Notes

### Preview thumbnail lookup paths (both `getMyCollections` and `getSavedReels`)

1. **Primary** — `saved.mediaId` is a valid MongoDB ObjectId → `reel._id` match
2. **ObjectId fallback** — valid ObjectId but no `_id` match → lookup by `reel.mediaId` field (handles old ReelCard bug where Media._id was stored)
3. **UUID fallback** — Postgres UUID fails `ObjectId.isValid()` → lookup by `reel.mediaId` string field directly

All three paths apply `s3UriToHttps()` before returning to the client.

### `CollectionCard` preview rendering rules

| `previewUrls.length` | Render |
|---|---|
| 0 | Emoji (if set) or folder icon (accent colour) |
| 1–3 | `previewUrls[0]` as full-size square image |
| 4 | 2×2 grid of `previewUrls[0..3]` |
