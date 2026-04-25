# Messages Module - QA Test Cases

**Module**: `messages`
**Type**: Core Feature
**Category**: Social & Discovery
**Status**: Production
**Last Updated**: 2026-04-25

---

### TC-messages-01: Shared reel renders as a tappable media card

**Type:** Bug Regression / Automated / Manual
**Feature area:** Direct share in chat thread
**Priority:** P1

**Preconditions:**
- Sender can create or reuse a conversation with the target user
- Sender opens Share With from a reel card

**Steps:**
1. Share a reel directly to another user.
2. Open the target conversation thread.
3. Inspect the newly sent message.
4. Tap the media card.

**Expected result:** The message shows a thumbnail with reel affordance, and tapping it opens the reel in feed.
**Actual result (before fix):** The thread showed a plain text link with no media preview or in-app tap target.
**Fix applied:** Direct shares now send `sharedContent` metadata and the thread renders a tappable preview card.
**Regression test:** apps/chefooz-app/src/components/share/DirectShareModal.spec.tsx
**Status:** Fixed ✅

### TC-messages-02: Shared flick renders as a tappable media card

**Type:** Bug Regression / Manual
**Feature area:** Direct share in chat thread
**Priority:** P1

**Preconditions:**
- Sender opens Share With from a Flick/post card
- Post has thumbnail or image data available

**Steps:**
1. Share the flick directly to another user.
2. Open the target conversation thread.
3. Tap the media card.

**Expected result:** The message shows a flick thumbnail card and tapping it opens the post detail route.
**Actual result (before fix):** The thread showed a raw text message with no thumbnail, and the content type was indistinguishable from reel shares.
**Fix applied:** Flick shares now use the same shared-content attachment pipeline with post-specific navigation.
**Regression test:** apps/chefooz-app/src/components/messaging/SharedContentCard.spec.tsx
**Status:** Fixed ✅

### TC-messages-03: Shared flick from post viewer reopens the flick route

**Type:** Bug Regression / Automated / Manual
**Feature area:** Direct share from dedicated flick viewer
**Priority:** P1

**Preconditions:**
- User opens a flick via `/post/[postId]`
- The flick belongs to a profile with a public username

**Steps:**
1. Tap share from the flick detail screen.
2. Choose Share With.
3. Send the flick to another user.
4. Open the chat thread and tap the shared flick card.

**Expected result:** The share action opens the local share sheet, the message renders as a flick card, and tapping it opens `/post/[postId]?username=<authorUsername>`.
**Actual result (before fix):** The flick detail screen pushed `/reels/:id?action=share`, and shared flick cards could fall back into the reel feed.
**Fix applied:** The flick detail screen now uses `ShareSheet` + `DirectShareModal` directly, and shared-content payloads keep post routing metadata.
**Regression test:** apps/chefooz-app/src/app/post/[postId].spec.tsx
**Status:** Fixed ✅