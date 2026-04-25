# Messages Module - Technical Guide

**Module**: `messages`
**Type**: Core Feature
**Category**: Social & Discovery
**Status**: Production
**Last Updated**: 2026-04-25

---

## Message Content Model

The chat message schema now supports four content channels:

- `text`
- `imageUrl`
- `imageUrls`
- `sharedContent`

`sharedContent` is a structured attachment used by direct-share flows.

```ts
type SharedContentAttachment = {
  contentType: 'REEL' | 'POST';
  mediaId: string;
  authorUsername?: string | null;
  authorName?: string | null;
  caption?: string | null;
  thumbnailUrl?: string | null;
  imageUrls?: string[];
  videoUrl?: string | null;
  durationSec?: number | null;
}
```

## Direct Share Flow

1. The share modal records the social share event.
2. The app creates or reuses a DM conversation.
3. The app sends a `sharedContent` message payload.
4. The chat thread renders a tappable media preview card.
5. Flick shares resolve author metadata from `author`, `owner`, or an explicit viewer-route override so post cards can reopen the flick detail screen.

## Rendering Rules

- `REEL` attachments use the thumbnail or video fallback and show a play affordance.
- `POST` attachments use the thumbnail or first image and show a flick label.
- Attachment messages do not depend on parsing visible text links.
- Conversation last-message previews fall back to `Shared a reel` or `Shared a flick` when the message has no text.

## Navigation Rules

- Reel attachments route to `/(tabs)/feed?highlightReel=<mediaId>`.
- Post attachments route to `/post/<mediaId>?username=<authorUsername>`.
- If legacy post attachments are missing `authorUsername`, the client falls back to `/(tabs)/home` instead of the reel feed.

## Post Viewer Share Flow

- The dedicated flick viewer now opens `ShareSheet` in place.
- Choosing direct share from that sheet passes a post-shaped payload into `DirectShareModal` with the current profile username as a routing override.
- This keeps post shares aligned with the feed share flow and prevents `/reels/:id?action=share` from reopening flicks inside reel surfaces.