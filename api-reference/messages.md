# Messages API Reference

**Last Updated**: 2026-04-25
**Base path**: `/api/v1/messages`

---

## `POST /api/v1/messages/:id/send`

Send a message in an existing conversation.

### Request Body

| Field | Type | Required | Description |
|---|---|---|---|
| `text` | string | No | Plain text message body |
| `imageUrl` | string | No | Single image URL |
| `imageUrls` | string[] | No | Multiple image URLs |
| `sharedContent` | object | No | Structured attachment for direct-shared reel or flick previews |

At least one of `text`, `imageUrl`, `imageUrls`, or `sharedContent` must be present.

### `sharedContent` shape

```json
{
  "contentType": "REEL",
  "mediaId": "69ebc90323940843040fc931",
  "authorUsername": "taste_guru2",
  "authorName": "Taste Guru",
  "caption": "Freshly plated biryani",
  "thumbnailUrl": "https://cdn.chefooz.com/media/thumb.jpg",
  "imageUrls": [],
  "videoUrl": "https://cdn.chefooz.com/media/video.mp4",
  "durationSec": 18
}
```

### Response

```json
{
  "success": true,
  "message": "Message sent",
  "data": {
    "id": "mongo-message-id",
    "conversationId": "mongo-conversation-id",
    "senderId": "uuid",
    "text": null,
    "imageUrl": null,
    "imageUrls": [],
    "sharedContent": {
      "contentType": "POST",
      "mediaId": "69ebc90323940843040fc932",
      "authorUsername": "jagrity",
      "thumbnailUrl": "https://cdn.chefooz.com/media/post.jpg"
    },
    "readBy": ["uuid"],
    "isRead": true,
    "isEdited": false,
    "createdAt": "2026-04-25T12:00:00.000Z",
    "updatedAt": "2026-04-25T12:00:00.000Z"
  }
}
```