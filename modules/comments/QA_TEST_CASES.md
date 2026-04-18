# Comments Module — QA Test Cases

**Module**: `apps/chefooz-apis/src/modules/comments`  
**Frontend**: `apps/chefooz-app/src/components/comments/`  
**Client Hook**: `libs/api-client/src/lib/hooks/useComments.ts`  
**Last Updated**: 2026-03-XX

---

## Bug Regression Tests

### TC-COMMENTS-BUG-001: Comment avatars display correctly (CDN URL, not S3 key)

**Type:** Bug Regression  
**Feature area:** Comments sheet — `CommentItem` avatar  
**Priority:** P1

**Preconditions:**
- User is logged in
- At least one other user has an avatar set (avatar uploaded via profile screen)
- That user has posted a comment on a reel

**Steps:**
1. Open any reel
2. Tap the comment icon to open the comments sheet
3. Observe the avatar circles next to each comment

**Expected result:** Each comment with a non-null `avatarUrl` shows the user's actual profile photo (rounded image). Comments for users with no avatar show the initials fallback circle.

**Actual result (before fix):** All avatar circles showed the primary-color filled circle (black/brand colour). No profile photos appeared even when users had avatars.

**Root cause:** `comments.service.ts` → `mapCommentToResponse()` returned `user.avatarUrl` as a raw S3 key (e.g. `avatars/userId/photo.jpg`). React Native `Image` silently fails on non-HTTP URIs, so `CommentItem` always fell through to the initials fallback. `buildMediaUrl()` was not applied.

**Fix applied:** Added `import { buildMediaUrl } from '../../utils/media-url.util'` and changed:
```ts
// Before:
avatarUrl: user?.avatarUrl ?? null

// After:
avatarUrl: buildMediaUrl(user?.avatarUrl, process.env['CDN_URL']) ?? null
```

**Regression test:** `apps/chefooz-apis/src/modules/comments/comments.service.spec.ts`  
**Status:** Fixed ✅

---

### TC-COMMENTS-BUG-002: Optimistic comment shows current user's avatar

**Type:** Bug Regression  
**Feature area:** Comments sheet — optimistic UI while posting  
**Priority:** P2

**Preconditions:**
- User is logged in and has an avatar uploaded
- User opens a reel's comment sheet

**Steps:**
1. Open any reel and tap the comment icon
2. Type a comment and tap Send
3. Observe the new comment that appears instantly (before server confirms)

**Expected result:** The optimistic (temporary) comment row shows the current user's profile photo in the avatar slot.

**Actual result (before fix):** Optimistic comment always showed the initials fallback circle (avatarUrl was hardcoded to `null`).

**Root cause:** `useCreateComment` hook in `useComments.ts` hardcoded `avatarUrl: null` in the temp comment. The hook had no access to the current user's CDN avatar URL.

**Fix applied:**  
- `useCreateComment` now accepts an optional `currentUserAvatarUrl?: string | null` parameter  
- `CommentInput.tsx` passes `user?.avatarUrl` from `useAuthStore()` at call site:  
  ```ts
  const createCommentMutation = useCreateComment(user?.avatarUrl);
  ```

**Regression test:** Manual  
**Status:** Fixed ✅
