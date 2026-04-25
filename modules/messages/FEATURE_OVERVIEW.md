# Messages Module - Feature Overview

**Module**: `messages`
**Type**: Core Feature
**Category**: Social & Discovery
**Status**: Production
**Last Updated**: 2026-04-25

---

## Purpose

1:1 messaging lets users chat socially after a follow relationship exists.

## Key Screens

- Conversation inbox
- Message thread
- Shared media preview cards inside chat threads

## Shared Content in Chat

- Direct Share from feed cards now sends a structured media attachment instead of a plain text link.
- Shared reels render with a thumbnail, play affordance, and tap-through back into the feed.
- Shared flicks/posts render with a thumbnail, flick label, and tap-through into the dedicated post screen.
- Shared content cards are designed to avoid the friction of copy-pasting or parsing raw URLs in chat.

## Business Rules

- Messaging remains gated by the social follow relationship.
- Shared content cards are supported for `REEL` and `POST` content types.
- Tapping a shared reel opens the feed highlight route.
- Tapping a shared flick opens the post detail route when the author username is available; otherwise it falls back to the feed highlight route.