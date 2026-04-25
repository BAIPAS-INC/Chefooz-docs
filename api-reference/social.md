# Social API Reference

**Last Updated**: 2026-04-25
**Base path**: `/api/v1`

---

## Reel Share Endpoints

### `GET /api/v1/reels/:reelId/share/targetable-users`

Returns the users that the authenticated user can directly share a reel with.

**Guards**: Authenticated mobile user

#### Query Parameters

| Param | Type | Required | Default | Description |
|---|---|---|---|---|
| `cursor` | string | No | - | ISO 8601 timestamp returned as `nextCursor` from the previous page. Invalid or empty cursors are ignored safely. |
| `limit` | number | No | 20 | Number of follow relationships to read before deduplicating target users. |

#### Response

```json
{
  "success": true,
  "message": "Targetable users retrieved",
  "data": {
    "items": [
      {
        "id": "uuid",
        "username": "chef_maria",
        "fullName": "Maria Fernandes",
        "avatar": "https://cdn.chefooz.com/avatar.jpg",
        "isFollowing": true,
        "isFollowedBy": false
      }
    ],
    "nextCursor": "2026-04-25T10:15:30.000Z"
  }
}
```

#### Notes

- Pagination is based on `user_follow.createdAt DESC`.
- `nextCursor` is a timestamp cursor, not a reel ID or user ID.
- The service deduplicates returned users after reading follow rows, so the number of `items` can be lower than `limit` when the user has mutual follow relationships.