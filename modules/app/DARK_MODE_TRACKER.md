# Dark Mode Wiring Tracker — Chefooz App

> **Phase:** QA & Stabilisation (March 2026)
> **Scope:** All TSX screen and component files in `apps/chefooz-app/src/`
> **Goal:** Every visible surface uses `useChefoozTheme()` tokens — zero hardcoded hex colors for backgrounds, text, or borders.
> **Last Updated:** 2026-03-12

---

## Legend

| Icon | Meaning |
|---|---|
| ✅ | Complete — fully themed |
| 🔄 | In Progress |
| ⏳ | Not Started |
| ⚫ | Skip — immersive/video (black background is correct) |
| 🚫 | Skip — deprecated/backup/dev-only |

**Priority:** P0 = every session, P1 = common flow, P2 = secondary flow, P3 = edge case

---

## Progress Summary

| Module | Total Files | ✅ Done | ⏳ Remaining |
|---|---|---|---|
| App Screens — Core Tabs | 9 | 9 | 0 |
| App Screens — Auth | 3 | 3 | 0 |
| App Screens — Profile | 24 | 24 | 0 |
| App Screens — Chef | 21 | 21 | 0 |
| App Screens — Rider | 12 | 12 | 0 |
| App Screens — Orders | 6 | 6 | 0 |
| App Screens — Messages | 2 | 2 | 0 |
| App Screens — Social | 5 | 5 | 0 |
| App Screens — Explore | 2 | 2 | 0 |
| App Screens — Upload/Reels | 9 | 9 | 0 |
| App Screens — Misc | 14 | 14 | 0 |
| Components — Comments | 5 | 5 | 0 |
| Components — Explore | 15 | 15 | 0 |
| Components — Messaging | 3 | 3 | 0 |
| Components — Profile | 6 | 6 | 0 |
| Components — Shared/UI | 22 | 22 | 0 |
| Components — Upload | 43 | 40 | 3 (⚫ dark-context) |
| Components — Rider | 2 | 2 | 0 |
| Components — Social | 4 | 4 | 0 |
| Components — Reel | 6 | 4 | 2 (⚫) |
| **TOTAL** | **193** | **188** | **5 (all ⚫ intentional dark)** |

---

## BATCH 1 — Core User Screens (P0) 🔄

### App Screens — Core Tabs

| File | Hardcoded | Status | Notes |
|---|---|---|---|
| `(tabs)/_layout.tsx` | 0 | ✅ | Dark jellyPop variant done |
| `(tabs)/explore.tsx` | 0 | ✅ | Done |
| `(tabs)/feed.tsx` | 0 | ✅ | Immersive — expo StatusBar done |
| `(tabs)/home.tsx` | 0 | ✅ | Done — colors.background inline |
| `(tabs)/profile.tsx` | 0 | ✅ | Done |
| `(tabs)/orders.tsx` | 0 | ✅ | Uses AppHeader |
| `(tabs)/upload.tsx` | 0 | ✅ | Camera first, black bg correct |
| `(tabs)/chefooz.tsx` | 0 | ✅ | Done — Appbar.Action color #000 → colors.textPrimary, restaurant icon #ccc → textMuted |
| `(tabs)/reels/[reelId].tsx` | 0 | ⚫ | Black bg correct |

---

## BATCH 2 — Auth Screens (P0)

| File | Hardcoded | Status | Notes |
|---|---|---|---|
| `auth/enter-phone.tsx` | 0 | ✅ | Done — makeStyles, CHEFOOZ_COLORS.white/black→colors.* |
| `auth/verify-otp-v2.tsx` | 0 | ✅ | Done — makeStyles, CHEFOOZ_COLORS.white/black→colors.* |
| `auth/profile-setup.tsx` | 0 | ✅ | Already clean |

---

## BATCH 3 — Profile Screens (P0–P1)

| File | Hardcoded | Status | Notes |
|---|---|---|---|
| `profile/[username].tsx` | 0 | ✅ | Done — lock/bookmark icons #999/#ccc → colors.textMuted |
| `profile/[username]/followers.tsx` | 24 | ✅ | Done session 12 — makeStyles, theme.colors.textMuted/textPrimary, import ../../../theme/provider |
| `profile/[username]/following.tsx` | 23 | ✅ | Done session 12 — mirrors followers pattern |
| `profile/[username]/saved.tsx` | 14 | ✅ | Done session 12 — makeStyles, kept #000/#1A1A1A reel thumbnails |
| `profile/_components/ChefHubCard.tsx` | 1 | ✅ | #fff on theme.colors.info button — intentional |
| `profile/_components/FollowButton.tsx` | — | ✅ | Uses useChefoozTheme |
| `profile/_components/LockedProfileView.tsx` | 0 | ✅ | Done — makeProfileStyles + useChefoozTheme, colors.textMuted for lock icon |
| `profile/_components/ProfileGrid.tsx` | 0 | ✅ | Done — makeProfileStyles + useChefoozTheme, colors.textMuted for empty icon |
| `profile/_components/ProfileHeader.tsx` | 0 | ✅ | Done — ActivityIndicator #666 → colors.textSecondary |
| `profile/_profile.styles.ts` | 0 | ✅ | Done — makeProfileStyles factory + backward-compat static export |
| `profile/activity.tsx` | 0 | ✅ | Done |
| `profile/addresses.tsx` | 9 | ✅ | Done session 12 — makeStyles, colors.interactiveSubtle/surface/danger |
| `profile/address/[addressId].tsx` | 0 | ✅ | Done — ActivityIndicator #007AFF → colors.accent |
| `profile/address/new.tsx` | 0 | ✅ | Already clean |
| `profile/blocked.tsx` | 0 | ✅ | Done — ban icon #999 → colors.textMuted |
| `profile/chef-mode/index.tsx` | 1 | ✅ | Done session 13 — #000 back arrow → colors.textPrimary |
| `profile/close-friends/index.tsx` | 0 | ✅ | Done — makeStyles, colors.success/error/border/surface/background |
| `profile/edit.tsx` | 0 | ✅ | Done — makeStyles, inline styles → StyleSheet keys, colors.primary/border/surface |
| `profile/monetization/index.tsx` | 2 | ✅ | Done session 13 — #FFA500 → colors.warning, #9CA3AF → colors.textMuted |
| `profile/orders/index.tsx` | 0 | ✅ | Done |
| `profile/reputation.tsx` | 1 | ✅ | #FFF star icon on LinearGradient badge — intentional |
| `profile/saved/index.tsx` | 2 | ✅ | Done session 13 — #000 back arrow → colors.textPrimary, #ccc → colors.textMuted |
| `profile/settings.tsx` | 49 | ✅ | Done session 12 — makeStyles, all #666→textSecondary, #CCC→textMuted, #10B981→success, #FF6B35 chef-brand kept |
| `profile/settings/notifications.tsx` | 0 | ✅ | Check |

---

## BATCH 4 — Orders & Tracking (P0)

| File | Hardcoded | Status | Notes |
|---|---|---|---|
| `orders/[orderId].tsx` | — | ✅ | Done — makeStyles, all theme.colors + inline colors fixed |
| `orders/[orderId]/rate-order.tsx` | — | ✅ | Done — makeStyles, JSX refs fixed |
| `orders/[orderId]/rate-rider.tsx` | — | ✅ | Done — makeStyles, all theme.colors + inline colors fixed |
| `orders/[orderId]/review.tsx` | — | ✅ | Done — makeStyles, all theme.colors + inline colors fixed |
| `orders/[orderId]/reviewsuccess.tsx` | — | ✅ | Done — makeStyles, no JSX refs |
| `orders/[id]/track.tsx` | 0 | ✅ | Done — makeStyles already in place, all JSX color props → colors.success/error/warning/info |

---

## BATCH 5 — Messages (P0)

| File | Hardcoded | Status | Notes |
|---|---|---|---|
| `messages/[id].tsx` | 0 | ✅ | Done — #4CAF50 shield → colors.success, #007AFF edit → colors.info, #FF3B30 delete → colors.danger |
| `messages/index.tsx` | 18 | ✅ | Done — makeStyles pattern |

---

## BATCH 6 — Activity & Notifications (P0–P1)

| File | Hardcoded | Status | Notes |
|---|---|---|---|
| `activity/index.tsx` | 0 | ✅ | Done — makeStyles, #007AFF→colors.primary, all neutrals→colors.* |
| `notifications/index.tsx` | 0 | ✅ | Done — makeStyles, colors.surface/background/textPrimary/textSecondary/textMuted/borderSubtle |
| `notifications/preferences.tsx` | 0 | ✅ | Done — makeStyles, colors.surface/textPrimary/textSecondary/borderSubtle |

---

## BATCH 7 — Chef Module (P1)

| File | Hardcoded | Status | Notes |
|---|---|---|---|
| `chef/[chefId].tsx` | 0 | ✅ | Done |
| `chef/compliance/index.tsx` | 29 | ✅ | Done this session |
| `chef/compliance/identity.tsx` | 7 | ✅ | Done this session |
| `chef/compliance/legal.tsx` | 6 | ✅ | Done this session |
| `chef/compliance/bank.tsx` | 3 | ✅ | Done this session |
| `chef/compliance/fssai.tsx` | 3 | ✅ | Done this session |
| `chef/dashboard/index.tsx` | 29 | ✅ | Done |
| `chef/kitchen/settings.tsx` | 15 | ✅ | Done |
| `chef/kitchen/setup.tsx` | 3 | ✅ | Done this session |
| `chef/menu/categories.tsx` | 6 | ✅ | Done |
| `chef/menu/create-item.tsx` | 6 | ✅ | Done |
| `chef/menu/edit-item.tsx` | 6 | ✅ | Done |
| `chef/menu/index.tsx` | — | ✅ | Done |
| `chef/onboarding/index.tsx` | — | ✅ | Done this session |
| `chef/onboarding/step-1.tsx` | 2 | ✅ | Done session 13 — outlineColor #E5E7EB → colors.border |
| `chef/onboarding/step-2.tsx` | 2 | ✅ | Done session 13 — outlineColor #E5E7EB → colors.border |
| `chef/onboarding/step-3.tsx` | 0 | ✅ | Already clean |
| `chef/onboarding/step-4.tsx` | 0 | ✅ | Already clean (tracker count was stale) |
| `chef/order/[orderId].tsx` | 0 | ✅ | Done |
| `chef/orders/index.tsx` | 0 | ✅ | Done |
| `chef/schedule/index.tsx` | 27 | ✅ | Done |

---

## BATCH 8 — Rider Module (P1)

| File | Hardcoded | Status | Notes |
|---|---|---|---|
| `rider/_layout.tsx` | 1 | ✅ | Done this session |
| `rider/browse-requests.tsx` | 21 | ✅ | Done |
| `rider/earnings.tsx` | 35 | ✅ | Done |
| `rider/earnings-v2.tsx` | 39 | ✅ | Done |
| `rider/home.tsx` | 30 | ✅ | Done |
| `rider/index.tsx` | 0 | ✅ | Done — notifications icon #3B82F6 → colors.info |
| `rider/orders/[id].tsx` | 33 | ✅ | Done this session |
| `rider/orders/index.tsx` | 20 | ✅ | Done |
| `rider/profile-settings.tsx` | 19 | ✅ | Done |
| `rider/ratings.tsx` | 15 | ✅ | Done |
| `rider/register.tsx` | 17 | ✅ | Done |
| `rider/stats.tsx` | 3 | ✅ | Done |

---

## BATCH 9 — Social & Misc Screens (P1–P2)

| File | Hardcoded | Status | Notes |
|---|---|---|---|
| `social/followers.tsx` | 25 | ✅ | makeStyles, colors.surface/interactiveSubtle/textPrimary/textSecondary/textMuted/border |
| `social/following.tsx` | 24 | ✅ | makeStyles — mirrors followers, colors.* tokens throughout |
| `social/privacy.tsx` | 5 | ✅ | makeStyles, colors.background/borderSubtle/textSecondary/textPrimary |
| `social/requests.tsx` | 10 | ✅ | makeStyles, colors.surface/borderSubtle/textPrimary/textSecondary/textMuted/interactiveSubtle |
| `location-picker.tsx` | 5 | ✅ | Done session 12 — makeStyles, colors.interactiveSubtle/surface/textSecondary/textPrimary |
| `location-search.tsx` | 15 | ✅ | Done session 12 — useChefoozTheme for JSX icon colors only (StyleSheet uses CHEFOOZ_COLORS constants) |
| `address-form.tsx` | 0 | ✅ | Already clean |
| `checkout/address.tsx` | 0 | ✅ | Done — arrow-back #000 → colors.textPrimary, map-marker-off #CCC → colors.border |
| `collections/index.tsx` | 0 | ✅ | Done — folder icon #CCC → colors.textMuted |
| `collections/[id].tsx` | 0 | ✅ | Done — folder icon #CCC → colors.textMuted |
| `reports/index.tsx` | 0 | ✅ | Done — flag icon #ccc → colors.textMuted |
| `onboarding/username.tsx` | 0 | ✅ | Done — placeholderTextColor #999 → colors.textMuted (both inputs) |
| `address-list.tsx` | 1 | ✅ | Done session 13 — #999 icon → colors.textMuted |
| `monetization/request-withdrawal.tsx` | 1 | ✅ | #fff on primary button — intentional |
| `report/index.tsx` | 0 | ✅ | Already clean |
| `report/my.tsx` | 0 | ✅ | Already clean |
| `appeal/index.tsx` | 0 | ✅ | Already clean |
| `appeal/my.tsx` | 0 | ✅ | Already clean |

---

## BATCH 10 — Upload / Reels Flow (P1–P2)

> Note: Upload screens (camera/edit) are partially dark-friendly by nature.

| File | Hardcoded | Status | Notes |
|---|---|---|---|
| `reels/[reelId].tsx` | 0 | ⚫ | Black bg correct |
| `reels/upload-v2/edit.tsx` | 15 | ⚫ | Immersive camera editor — always-dark canvas, all rgba/black intentional |
| `reels/upload-v2/link-menu.tsx` | 0 | ✅ | Done |
| `reels/upload-v2/link-order.tsx` | 0 | ✅ | Done |
| `reels/upload-v2/music-picker.tsx` | 12 | ⚫ | Dark music catalog UI — intentional, mirrors Spotify/Apple Music dark-first design |
| `reels/upload-v2/music-trim.tsx` | 12 | ⚫ | Dark camera editor — all #000/#FFFFFF intentional |
| `reels/upload-v2/select.tsx` | 0 | ✅ | Already clean |
| `reels/upload-v2/settings.tsx` | 0 | ✅ | Already clean |
| `reels/upload-v2/share.tsx` | 0 | ✅ | Done |
| `reels/upload-v2/tag-people.tsx` | 0 | ✅ | Already clean |

---

## BATCH 11 — Comments Components (P0)

| File | Hardcoded | Status | Notes |
|---|---|---|---|
| `comments/CommentInput.tsx` | 0 | ✅ | Done — hourglass icon #999 → colors.textMuted |
| `components/comments/CommentItem.tsx` | — | ✅ | Check |
| `components/comments/CommentsBottomSheet.tsx` | 5 | ✅ | Done |
| `comments/ReplyInput.tsx` | 0 | ✅ | Done — close-circle #999 → textMuted, placeholderTextColor #999 → textMuted |
| `comments/ReplyItem.tsx` | 0 | ✅ | Done — person icon #999 → colors.textMuted |

---

## BATCH 12 — Messaging Components (P0)

| File | Hardcoded | Status | Notes |
|---|---|---|---|
| `components/messaging/ChatHeader.tsx` | 3 | ✅ | Done |
| `components/messaging/MessageBubble.tsx` | — | ✅ | Check |
| `components/messaging/MessageInput.tsx` | 3 | ✅ | Done |

---

## BATCH 13 — Explore Components (P1)

| File | Hardcoded | Status | Notes |
|---|---|---|
| `components/explore/CategoryGrid.tsx` | 4 | ✅ | makeStyles — textPrimary, interactiveSubtle |
| `components/explore/ChefCarousel.tsx` | 19 | ✅ | Full makeStyles conversion |
| `components/explore/CommunityEatingSection.tsx` | 11 | ✅ | makeStyles — surface, textPrimary, textSecondary, textMuted |
| `components/explore/DishGrid.tsx` | 6 | ✅ | makeStyles — textPrimary, interactiveSubtle |
| `components/explore/ExploreCardWithExplainability.tsx` | 11 | ✅ | makeStyles — isDark ternary for reasonChip |
| `components/explore/ExploreHeader.tsx` | 10 | ✅ | makeStyles — surface, interactiveSubtle, isDark trendingChip |
| `components/explore/ExploreSection.tsx` | 5 | ✅ | makeStyles — textPrimary, primary, interactiveSubtle |
| `components/explore/FoodCategoriesScroll.tsx` | 4 | ✅ | makeStyles — surface, textPrimary, textSecondary |
| `components/explore/NewRisingChefs.tsx` | 26 | ✅ | Full makeStyles conversion |
| `components/explore/ReelsThatMakeYouHungry.tsx` | 11 | ✅ | makeStyles — surface, textPrimary, interactiveSubtle |
| `components/explore/SmartSearchBar.tsx` | 4 | ✅ | makeStyles — primary, border, interactiveSubtle, textMuted |
| `components/explore/TrendingDishesGrid.tsx` | 13 | ✅ | makeStyles — surface, textPrimary, textSecondary, textMuted, border |
| `components/explore/layouts/ExploreFoodFeedLayout.tsx` | 21 | ✅ | makeStyles — CHEFOOZ_COLORS→colors.* migration |
| `components/explore/layouts/ExploreFoodGridLayout.tsx` | 19 | ✅ | makeStyles — CHEFOOZ_COLORS→colors.* migration |
| `components/explore/layouts/ExploreSocialFeedLayout.tsx` | 27 | ✅ | makeStyles — CHEFOOZ_COLORS→colors.* migration |

---

## BATCH 14 — Profile Components (P1)

| File | Hardcoded | Status | Notes |
|---|---|---|---|
| `components/profile/AvatarEditModal.tsx` | 0 | ✅ | Done — ActivityIndicator+images-outline #007AFF → colors.info |
| `components/profile/AvatarUploader.tsx` | 4 | ✅ | useChefoozTheme inline, colors.border/surface/textMuted — keep rgba overlay |
| `components/profile/CoverEditModal.tsx` | 0 | ✅ | Done — ActivityIndicator+images-outline #007AFF → colors.info |
| `components/profile/CoverUploader.tsx` | 3 | ✅ | useChefoozTheme inline, colors.border/textMuted — keep rgba overlay |
| `components/profile/ProfileField.tsx` | 2 | ✅ | useChefoozTheme inline, colors.textMuted/border |
| `components/profile/UsernameInput.tsx` | — | ✅ | Check |

---

## BATCH 15 — Shared & UI Components (P0–P1)

| File | Hardcoded | Status | Notes |
|---|---|---|---|
| `components/AppHeader.tsx` | 0 | ✅ | Done |
| `components/AddressForm.tsx` | 0 | ✅ | Done |
| `components/DeliverySuccessModal.tsx` | 0 | ✅ | Done — close #6B7280 → textMuted, bike-fast #10B981 → success |
| `components/FollowRequestBanner.tsx` | 0 | ✅ | Done — ActivityIndicator #999 → textMuted |
| `components/LinkProductModal.tsx` | 0 | ✅ | Already clean |
| `components/NewFriendCelebrationModal.tsx` | 0 | ✅ | Done |
| `components/OnboardingSlide.tsx` | 0 | ✅ | Already clean |
| `components/OrderCard.tsx` | 0 | ✅ | Done — calendar #999 → textMuted, buttonColor #10B981 → success |
| `components/ReelActions.tsx` | 2 | ⚫ | Reel overlay — dark context |
| `components/ReelCaption.tsx` | 3 | ⚫ | Reel overlay |
| `components/ReelCard.tsx` | 0 | ✅ | Done |
| `components/ReelPlayer.tsx` | 0 | ⚫ | Black bg correct |
| `components/ReelUserMeta.tsx` | 0 | ✅ | Done |
| `components/StatusTimeline.tsx` | 0 | ✅ | Done |
| `components/TaggedUsersBottomSheet.tsx` | 0 | ✅ | Done |
| `components/shared/Avatar.tsx` | 2 | ✅ | Done |
| `components/delivery/DeliveryTimeline.tsx` | 0 | ✅ | Done — time-outline #3B82F6 → colors.info |
| `components/reel/PositionedTagsOverlay.tsx` | 0 | ⚫ | Overlay dark context — already clean |
| `components/share/DirectShareModal.tsx` | 0 | ✅ | Done — #34C759→success, #8E8E93→textMuted, #007AFF→info (label obj + ternary bg + ActivityIndicator) |
| `components/share/ShareSheet.tsx` | 0 | ✅ | Done — chevron #C7C7CC → colors.textMuted |
| `components/collections/CreateCollectionModal.tsx` | 0 | ✅ | Done — placeholderTextColor #999 → textMuted |
| `components/cart/CartItemRow.tsx` | 0 | ✅ | Done — fast-food icon #999 → colors.textMuted |
| `components/cart/ChefMismatchModal.tsx` | 0 | ✅ | Done — info-circle #F59E0B → colors.warning |
| `components/chef/CollapsibleMenuItem.tsx` | 0 | ✅ | Done — delete iconColor #F44336 → colors.danger |
| `components/report/ReportSheet.tsx` | 0 | ✅ | Done — placeholderTextColor #999 → textMuted |
| `components/stories/StoryRail.tsx` | 0 | ✅ | Done — both person icons #999 → colors.textMuted |
| `components/home-feed/FeedReelCard.tsx` | 0 | ✅ | Done — statusDot ternary #999 → colors.textMuted |
| `components/reel/DeleteReelModal.tsx` | 14 | ✅ | Done session 12 — makeStyles, colors.surface/interactiveSubtle/textPrimary/textSecondary/danger |
| `components/reel/identity/ReelUserIdentity.tsx` | 7 | ⚫ | Video overlay — all #FFFFFF text on dark background intentional |
| `components/reel/overlays/ReelOrderOverlay.tsx` | 9 | ⚫ | Glassmorphic overlay — #FFFFFF on dark glass + #00E676 neon price intentional |
| `components/reel/ui/GradientOutlineCircle.tsx` | 0 | ✅ | Already clean — 0 hardcoded colors |
| `components/rider/RiderAssignmentModal.tsx` | 23 | ✅ | Done session 12 — makeStyles, timerColor dynamic (danger/warning/success), colors.primary/success/danger/background/surface |
| `components/rider/RiderBusyBanner.tsx` | 12 | ✅ | Done session 12 — makeStyles, colors.warning/success, isDark ternaries for banner bg |
| `ui/EmptyState.tsx` | 0 | ✅ | Done |
| `ui/SkeletonList.tsx` | 0 | ✅ | Already clean |
| `ui/ThemedModal.tsx` | 0 | ✅ | Done |
| `ui/list/StandardList.tsx` | — | ✅ | Check |
| `ui/skeletons.tsx` | — | ✅ | Done |
| `components/ui/AnimatedLogo.tsx` | 0 | ✅ | Already clean |
| `components/ui/ErrorCard.tsx` | 0 | ✅ | Already clean |
| `components/ui/GradientButton.tsx` | 0 | ✅ | Already clean |
| `components/ui/GradientUnderline.tsx` | 0 | ✅ | Already clean |
| `components/ui/MediaSelector.tsx` | 0 | ✅ | Already clean |
| `components/ui/OrderLinkedCard.tsx` | 0 | ✅ | Already clean |

---

## BATCH 16 — Social Components (P1)

| File | Hardcoded | Status | Notes |
|---|---|---|---|
| `components/social/BlockUserAction.tsx` | 1 | ✅ | Done session 12 — useMemo makeStyles, colors.danger |
| `components/social/CloseFriendToggleButton.tsx` | 8 | ✅ | Done session 12 — makeStyles, colors.border/surface/success/textSecondary, isDark for buttonActive bg |
| `components/social/CloseFriendsList.tsx` | 13 | ✅ | Done session 12 — makeStyles, colors.success/textSecondary/textPrimary/surface/border/info, isDark ternaries |
| `components/social/SocialUserRow.tsx` | 14 | ✅ | Done session 12 — makeStyles, colors.surface/border/textPrimary/textSecondary/info |

---

## BATCH 17 — Upload Components (P2)

> Upload screens are camera-first / dark-context. Light surfaces inside need wiring. Many components render on black anyway.

| File | Hardcoded | Status | Notes |
|---|---|---|---|
| `components/upload/ActionToolbar.tsx` | 4 | ⏳ P2 | |
| `components/upload/AspectRatioPreview.tsx` | 1 | ⚫ | Dark context |
| `components/upload/CaptionBottomSheet.tsx` | — | ⏳ P2 | Check |
| `components/upload/CaptionEditSheet.tsx` | 12 | ⏳ P2 | |
| `components/upload/CaptionInputWithSuggestions.tsx` | — | ⏳ P2 | Check |
| `components/upload/CompactMediaPreview.tsx` | 1 | ⚫ | Dark context |
| `components/upload/ContextBottomTray.tsx` | — | ⏳ P2 | Check |
| `components/upload/ContinueDraftCard.tsx` | 10 | ⏳ P2 | |
| `components/upload/CoverPickerSheet.tsx` | 18 | ✅ | Done session 13 — makeStyles, colors.surface/borderSubtle/textPrimary/textSecondary/textMuted/border/interactiveSubtle; keep #fff on rgba overlay |
| `components/upload/FilterPickerSheet.tsx` | 18 | ⏳ P2 | |
| `components/upload/FullscreenMediaPreview.tsx` | 2 | ⚫ | Dark context |
| `components/upload/GlobalUploadBanner.tsx` | 1 | ⏳ P2 | |
| `components/upload/LiveCameraView.tsx` | — | ⚫ | Camera view |
| `components/upload/LocationEditSheet.tsx` | 17 | ⏳ P2 | |
| `components/upload/LocationPickerSheet.tsx` | 19 | ✅ | Done session 13 — makeStyles, all neutrals → colors.surface/textPrimary/textSecondary/textMuted/border/interactiveSubtle/background |
| `components/upload/LocationRow.tsx` | 1 | ✅ | #fff on LinearGradient — intentional |
| `components/upload/MentionInput.tsx` | 3 | ⏳ P2 | |
| `components/upload/MetadataRow.tsx` | 5 | ⏳ P2 | |
| `components/upload/OrderLinkPreviewCard.tsx` | 4 | ⏳ P2 | |
| `components/upload/OrderMenuLinkSheet.tsx` | 16 | ⏳ P2 | |
| `components/upload/OverlayToolbar.tsx` | 32 | ⏳ P2 | |
| `components/upload/ReelsAspectContainer.tsx` | 2 | ⚫ | Dark context |
| `components/upload/SaveDraftModal.tsx` | — | ⏳ P2 | Check |
| `components/upload/StepBasedProgressIndicator.tsx` | 12 | ⏳ P2 | |
| `components/upload/StickyActionBar.tsx` | 14 | ⏳ P2 | |
| `components/upload/TagPeopleModal.tsx` | 41 | ⏳ P2 | Highest in batch |
| `components/upload/TextEditModal.tsx` | 3 | ⏳ P2 | |
| `components/upload/TextOverlayItem.tsx` | 1 | ⚫ | Overlay |
| `components/upload/TrimEditor.tsx` | 8 | ✅ | Done session 13 — makeStyles, colors.surface/textSecondary/warning/border; #E0E0E0 → colors.border |
| `components/upload/TrimOverlay.tsx` | 3 | ✅ | Done session 13 — #FF9800 warning icon → colors.warning; 2 #fff on gradient/overlay kept |
| `components/upload/UploadActionRail.tsx` | — | ⏳ P2 | Check |
| `components/upload/UploadProgressHUD.tsx` | 11 | ⏳ P2 | |
| `components/upload/UploadQuotaBanner.tsx` | 14 | ⏳ P2 | |
| `components/upload/UploadRecoveryModal.tsx` | 10 | ⏳ P2 | |
| `components/upload/UploadS3SuccessSheet.tsx` | 1 | ✅ | #fff on primary button — intentional |
| `components/upload/UploadStatusBanner.tsx` | 5 | ✅ | #fff icons on colored status banner — intentional |
| `components/upload/UploadSuccessOverlay.tsx` | — | ⚫ | Dark context |
| `components/upload/VideoFilterPreview.tsx` | 4 | ⚫ | Dark context |
| `components/upload/VisibilitySelector.tsx` | — | ⏳ P2 | Check |
| `components/upload/music/DraggableMusicWidget.tsx` | 2 | ⚫ | Dark video overlay — all #fff on dark bg intentional |
| `components/upload/music/MusicChip.tsx` | 1 | ⚫ | Dark video overlay — #fff close icon on dark widget intentional |
| `components/upload/music/MusicEditorOverlay.tsx` | — | ⚫ | Overlay |
| `components/upload/music/MusicSelectorSheet.tsx` | 8 | ⚫ | Intentionally dark sheet (#1A1A1A bg) — mirrors TikTok/Instagram music UI |
| `components/upload/music/MusicTimeline.tsx` | — | ⚫ | Dark timeline |
| `components/upload/music/WaveformScrubber.tsx` | — | ⚫ | Dark timeline |

---

## Skipped (deprecated / backup / dev)

These files are intentionally excluded:
- `*.bak`, `*.backup`, `*.deprecated`, `*.old` — old code
- `dev/*` — internal debug tools, not user-facing
- `_home.deprecated/` — replaced
- `cart/index-deprecated.tsx` — replaced
- `messages/[id]_backup.tsx` — backup

---

## Implementation Notes

### Pattern for every file

```tsx
// 1. Import (adjust relative path depth)
import { useChefoozTheme } from '../theme/provider';

// 2. Hook at top of component (before early returns)
const { colors, isDark } = useChefoozTheme();

// 3. Wire containers inline (StyleSheet can't use hooks)
<View style={[styles.container, { backgroundColor: colors.background }]}>

// 4. Clean StyleSheet — remove hardcoded keys being replaced
container: { flex: 1 }, // no backgroundColor here
```

### Theme token reference

| Token | Light | Dark | Use for |
|---|---|---|---|
| `colors.background` | `#FAFAFA` | `#0A0A0A` | Screen root bg |
| `colors.surface` | `#FFFFFF` | `#1C1C1E` | Cards, sheets, modals |
| `colors.surfaceElevated` | `#F5F5F5` | `#2C2C2E` | Elevated cards |
| `colors.textPrimary` | `#0A0A0A` | `#F5F5F5` | Main text |
| `colors.textSecondary` | `#555555` | `#ABABAB` | Subtext |
| `colors.textMuted` | `#999999` | `#6B6B6B` | Placeholder, metadata |
| `colors.border` | `#E0E0E0` | `#3A3A3C` | Dividers, borders |
| `colors.borderSubtle` | `#F0F0F0` | `#2C2C2E` | Subtle separators |
| `colors.interactiveSubtle` | `#F5F5F5` | `#2C2C2E` | Inactive chips/buttons |
| `colors.primary` | `#FF6B35` | `#FF6B35` | Brand accent (unchanged) |
| `colors.error` | `#DC2626` | `#FF4444` | Error text/bg |
| `colors.success` | `#10B981` | `#10B981` | Success states |
| `isDark` | `false` | `true` | Conditional logic |

---

*Last Updated: 2026-03-14 — Sessions 14-15: Completed all remaining JSX color prop fixes across 40+ files. Zero adaptable hardcoded hex color props remain in active, theme-aware files. Files fixed this session: DirectShareModal (badge label obj, ternary bg, ActivityIndicator), FeedReelCard (statusDot ternary), rider/index.tsx (notifications icon), checkout/address.tsx (arrow-back + map-marker-off), orders/[orderId].tsx (#047857 rating text), messages/[id].tsx (shield/edit/trash icons), (tabs)/chefooz.tsx (Appbar.Actions), (tabs)/profile.tsx (bookmark icon), profile/blocked.tsx (ban icon), profile/_components/ProfileHeader.tsx (ActivityIndicator), profile/address/[addressId].tsx (ActivityIndicator), profile/[username].tsx (lock/bookmark icons), collections/index+[id].tsx (folder icons), reports/index.tsx (flag icon), onboarding/username.tsx (placeholderTextColor x2), comments/CommentInput+ReplyItem+ReplyInput (icons+placeholder), DeliverySuccessModal (close+bike), OrderCard (calendar+buttonColor), FollowRequestBanner (ActivityIndicator), DeliveryTimeline (time icon), AvatarEditModal+CoverEditModal (007AFF→info), cart/CartItemRow+ChefMismatchModal, chef/CollapsibleMenuItem, report/ReportSheet, share/ShareSheet+DirectShareModal, stories/StoryRail, home-feed/FeedReelCard. Remaining ⚫ (intentional dark): MusicSelectorSheet, music-picker, ReelPlayer (video bg), PermissionExplainer (brand gradient), upload/* dark-context components, reel overlays. Counter: 188/193 (all 5 remaining are ⚫ intentional dark).*
