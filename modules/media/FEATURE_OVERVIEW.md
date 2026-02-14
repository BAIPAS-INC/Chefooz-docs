# Media Upload Module - Feature Overview

> **Module**: `apps/chefooz-apis/src/modules/media`  
> **Purpose**: Media upload infrastructure for reels, stories, and images  
> **Last Updated**: February 14, 2026  
> **Status**: ✅ Production Active

---

## 📋 Executive Summary

The **Media Module** provides enterprise-grade upload infrastructure for Chefooz's content creation ecosystem. It handles video uploads for reels and stories, image uploads for profiles/products/chat, and orchestrates video processing pipelines through FFmpeg and AWS MediaConvert.

### Key Capabilities
- **Direct-to-S3 Uploads**: Presigned URLs for secure client-to-S3 transfers (no backend passthrough)
- **Video Processing Pipeline**: FFmpeg-first with MediaConvert fallback for transcoding, thumbnail generation, and quality variants
- **Type-Specific Upload Quotas**: Separate limits for review reels, menu showcase reels, and promotional content
- **Dual-Role Support**: Users and chefs have different quota pools for promotional content
- **Idempotent Uploads**: Client-generated upload IDs prevent duplicate uploads on network retries
- **Rich Metadata Support**: Captions, hashtags, geolocation, user tagging, text overlays, and filters

---

## 🎯 Business Value

### For Users
- **Fast Upload Experience**: Direct S3 upload eliminates backend bottlenecks
- **Resume Support**: Network failures don't lose progress (idempotent retries)
- **Rich Content Creation**: Add overlays, filters, music, and tags like Instagram/TikTok
- **Clear Quota Visibility**: Know exactly how many reels you can upload before hitting limits

### For Chefs
- **Menu Showcase Reels**: Create professional food videos linked to menu items
- **Unlimited Review Reels**: Share customer order reviews (1 reel per delivered order)
- **Promotional Content**: Separate quota pool from user content (10/week in production)

### For Business
- **Cost Efficiency**: Direct uploads reduce bandwidth costs by ~70%
- **Scalability**: S3 and MediaConvert handle unlimited concurrent uploads
- **Content Quality**: Automated transcoding ensures optimal viewing across devices
- **Compliance Ready**: File validation and moderation hooks built-in

---

## 🔄 Upload Workflows

### 1️⃣ **Standard Reel Upload Flow** (Production)

```
┌─────────────┐
│   Client    │
│  (Mobile)   │
└──────┬──────┘
       │ 1. POST /api/v1/media/upload-reel
       │    (caption, duration, file size, uploadId)
       ▼
┌─────────────────────────────────────────────┐
│         Backend - Media Service             │
│                                             │
│  ✅ Validate quota (review/menu/promo)     │
│  ✅ Validate file size & duration limits   │
│  ✅ Create Media + Reel documents          │
│  ✅ Return mediaId + uploadUrl stub        │
└──────┬──────────────────────────────────────┘
       │
       │ 2. POST /api/v1/media/presigned-upload-url
       │    (mediaId, fileType='video/mp4')
       ▼
┌─────────────────────────────────────────────┐
│         S3 Presigned URL Generator          │
│                                             │
│  ✅ Verify media ownership                 │
│  ✅ Generate 15-min presigned PUT URL      │
│  ✅ Return uploadUrl + s3Key               │
└──────┬──────────────────────────────────────┘
       │
       │ 3. PUT <presignedUrl>
       │    (video file bytes)
       ▼
┌─────────────────────────────────────────────┐
│         AWS S3 Upload Bucket                │
│     (chefooz-media-uat)                     │
│                                             │
│  📁 uploads/{mediaId}/original.mp4         │
│  📁 uploads/{mediaId}/thumbnail.jpg        │
└──────┬──────────────────────────────────────┘
       │
       │ 4. POST /api/v1/media/complete-upload
       │    (mediaId, s3Key, trim metadata)
       ▼
┌─────────────────────────────────────────────┐
│         Upload Completion Handler           │
│                                             │
│  ✅ Verify file exists in S3               │
│  ✅ Copy upload → input bucket             │
│  ✅ Update media status → PROCESSING       │
│  ✅ Enqueue FFmpeg job (primary)           │
└──────┬──────────────────────────────────────┘
       │
       │ 5. FFmpeg Worker (Bull Queue)
       ▼
┌─────────────────────────────────────────────┐
│         Video Processing Pipeline           │
│                                             │
│  🎬 Trim video (if startSec/endSec)       │
│  🎬 Generate 720p/480p/360p variants       │
│  🎬 Extract cover thumbnail                │
│  🎬 Burn text overlays                     │
│  🎬 Apply image filters                    │
│  🎬 Upload to output bucket                │
│  ✅ Update Media + Reel → READY           │
└─────────────────────────────────────────────┘
       │
       │ 6. Client polls GET /api/v1/media/:id/status
       ▼
┌─────────────────────────────────────────────┐
│         Media Status Response               │
│                                             │
│  {                                          │
│    status: "ready",                         │
│    variants: [                              │
│      { quality: "720p", url: "..." },      │
│      { quality: "480p", url: "..." }       │
│    ],                                       │
│    thumbnailUrl: "..."                      │
│  }                                          │
└─────────────────────────────────────────────┘
```

**Timeline**: 15-90 seconds from upload complete to READY state (varies by video length)

---

### 2️⃣ **Image Upload Flow** (Profiles, Chat, Products)

```
Client → POST /api/v1/media/chat/upload-init
         (mimeType='image/jpeg')
      ↓
Backend generates presigned S3 URL
      ↓
Client uploads directly to S3
      ↓
Backend returns CDN URL immediately (no processing needed)
```

**Use Cases**:
- Chef profile pictures
- Product menu item images
- Chat media attachments
- Story image posts

---

## 📊 Upload Quota System (Type-Specific)

### **Review Reels** (User Content)
**Business Rule**: 1 reel per completed order, no expiry

| Metric | Rule |
|--------|------|
| **Quota Formula** | `completedOrders × 1` |
| **Validation** | Count DELIVERED orders from `orders` table |
| **Counting Method** | Count review reels with `reelPurpose='USER_REVIEW'` |
| **Example** | 5 completed orders → Can upload 5 review reels total |
| **Enforcement Point** | `initReelUpload()` → `canUploadReviewReel()` |
| **Error Code** | `REVIEW_REEL_QUOTA_EXCEEDED` |

**Linking Requirements**:
- Must provide `orderId` in `UploadReelDto`
- Order must belong to uploader (`order.userId === userId`)
- Order must be DELIVERED (`order.deliveryStatus === 'DELIVERED'`)
- Order data snapshotted: `linkedOrderId`, `creatorOrderValue`

---

### **Menu Showcase Reels** (Chef Content)
**Business Rule**: 1 reel per menu item, lifetime

| Metric | Rule |
|--------|------|
| **Quota Formula** | `1 reel per menuItemId` |
| **Validation** | Menu item must exist and be active |
| **Counting Method** | Count showcase reels with `linkedMenu.menuItemIds` containing target item |
| **Example** | Chef has 10 menu items → Can upload 10 showcase reels (1 per item) |
| **Enforcement Point** | `initReelUpload()` → `canUploadMenuShowcaseReel()` |
| **Error Code** | `MENU_SHOWCASE_QUOTA_EXCEEDED` |

**Linking Requirements**:
- Must provide `menuItemId` in `UploadReelDto`
- Menu item must exist and be active (`chefMenuItem.isActive === true`)
- Menu data snapshotted: `chefId`, `menuItemIds[]`, `estimatedPaise`, `previewImage`

---

### **Promotional Reels** (User & Chef - Separate Pools)
**Business Rule**: Weekly rolling quota, resets after 7 days

| Metric | User Role | Chef Role |
|--------|-----------|-----------|
| **Weekly Quota (UAT)** | 100 reels | 100 reels |
| **Weekly Quota (Prod)** | 10 reels | 10 reels |
| **Rolling Window** | Last 7 days | Last 7 days |
| **Counting Method** | `contentSource='user'` | `contentSource='chef'` |
| **Reset Mechanism** | Automatic (sliding window) | Automatic (sliding window) |
| **Enforcement Point** | `initReelUpload()` → `canUploadPromotionalReel()` |
| **Error Code** | `PROMOTIONAL_REEL_QUOTA_EXCEEDED_USER` or `_CHEF` |

**Key Insight**: Users and chefs have **separate quota pools** for promotional content:
- User creating food review content → Uses **user pool** (10/week prod)
- Chef promoting their business → Uses **chef pool** (10/week prod)
- Same person can exhaust both pools independently

**No Linking Required**: Promotional reels don't require order or menu linking

---

### **File Size & Duration Limits** (Tier-Based)

| Tier | Max File Size (MB) | Max Duration (sec) | Notes |
|------|-------------------|-------------------|-------|
| **BRONZE** | 200 | 120 | Default tier for all users |
| **SILVER** | 200 | 120 | Same as Bronze (no upgrade) |
| **GOLD** | 200 | 120 | Same as Bronze (no upgrade) |
| **PLATINUM** | 200 | 120 | Same as Bronze (no upgrade) |

**Note**: All tiers currently have identical limits. Tier-based differentiation reserved for future monetization.

---

## 🎨 Advanced Features

### **1. Video Trimming** (Client-Side Preview, Server-Side Processing)
- **Client Flow**: User trims video in mobile editor (1-120 seconds)
- **Backend Processing**: FFmpeg uses `trim.startSec` and `trim.endSec` to crop video
- **DTO Field**: `CompleteUploadDto.trim: { startSec: number, endSec: number }`
- **Example**: Upload 60s video, trim to 15s-45s → Final output is 30s

### **2. Custom Cover Frame Selection**
- **Client Flow**: User scrubs timeline and selects cover frame timestamp
- **Backend Processing**: FFmpeg extracts frame at `coverTimestampSec`
- **DTO Field**: `UploadReelDto.coverTimestampSec: number` (default: 1.0s)
- **Use Case**: Choose most appealing frame instead of auto-generated first frame

### **3. Text Overlays** (Instagram-Style)
- **Client Flow**: User adds text stickers with position, scale, rotation
- **Backend Processing**: FFmpeg burns text into video (not removable after processing)
- **DTO Field**: `UploadReelDto.textOverlays: TextOverlayDto[]`
- **Max Limit**: 10 overlays per video
- **Overlay Properties**:
  - `content`: Text string (max 200 chars)
  - `position`: `{ x: 0-1, y: 0-1 }` (normalized screen coordinates)
  - `scale`: Size multiplier (e.g., 1.2 = 120%)
  - `rotation`: Degrees (0-360)
  - `startTime`/`endTime`: Visibility window in seconds
  - `style`: Font, color, shadow, background

### **4. Image Filters** (TikTok/Instagram-Style)
- **Client Flow**: User selects filter preset (vibrant, vintage, cinematic, etc.)
- **Backend Processing**: FFmpeg applies color grading (brightness, contrast, saturation, vignette)
- **DTO Field**: `UploadReelDto.filter: VideoFilterDto`
- **Filter Options**:
  - **Presets**: `vibrant`, `vintage`, `cinematic`, `noir`, `warm`, `cool`
  - **Manual Adjustments**: brightness, contrast, saturation, warmth, vignette (0-1 range)

### **5. User Tagging** (Dual System)
- **Caption Mentions** (`@username`): 
  - Parsed from reel caption
  - Stored in `taggedUserIds` array (max 10)
- **Positioned Tags** (Instagram tap-to-tag):
  - User taps on video to tag specific people
  - Stored in `positionedTags` array with `{ userId, x, y }` (max 20)
- **Unified Notifications**: Backend deduplicates and sends ONE notification per tagged user

### **6. Geolocation Tagging**
- **Purpose**: Enable location-based reel discovery ("See reels near you")
- **Storage**: `locationLat` and `locationLng` stored in Reel schema
- **Use Case**: Users searching "Italian food in New York" see reels from that area
- **Privacy**: Users can opt-out of sharing precise location

---

## 🔐 Security & Validation

### **Pre-Upload Validation**
✅ **File Type**: Only `video/mp4`, `video/quicktime`, `video/x-msvideo` allowed  
✅ **File Size**: Max 200MB (enforced client-side and backend)  
✅ **Duration**: 1-120 seconds (enforced client-side and backend)  
✅ **Quota Check**: Verify user hasn't exceeded type-specific limits  
✅ **Ownership**: Validate linked orders/menu items belong to uploader  

### **Presigned URL Security**
✅ **Expiration**: 15-minute validity window (900 seconds)  
✅ **Single-Use**: URLs tied to specific media document (can't be reused)  
✅ **Content-Type Validation**: MIME type verified on S3 PUT  
✅ **Bucket Access**: Presigned URLs grant temporary write-only access  

### **Post-Upload Verification**
✅ **File Existence**: `HeadObjectCommand` verifies S3 upload success  
✅ **ETag Verification**: Optional checksum validation (if provided by client)  
✅ **Status Tracking**: Media document transitions: UPLOADING → PROCESSING → READY/FAILED  

---

## 📡 API Endpoints

### **POST** `/api/v1/media/upload-reel`
**Purpose**: Initialize reel upload with metadata  
**Auth**: Required (JWT)  
**Request Body**:
```json
{
  "uploadId": "550e8400-e29b-41d4-a716-446655440000",
  "fileName": "my-reel.mp4",
  "mimeType": "video/mp4",
  "durationSec": 45,
  "sizeBytes": 50000000,
  "caption": "Check out this amazing dish! 🍕 #foodie",
  "hashtags": ["foodie", "delicious", "homemade"],
  "orderId": "order-uuid",
  "menuItemId": "menu-item-uuid",
  "locationLat": 28.6139,
  "locationLng": 77.2090,
  "taggedUserIds": ["user-uuid-1", "user-uuid-2"],
  "positionedTags": [
    { "userId": "user-uuid-1", "x": 0.5, "y": 0.3 }
  ],
  "coverTimestampSec": 5.5,
  "textOverlays": [
    {
      "id": "overlay-1",
      "content": "Fresh from the oven! 🔥",
      "position": { "x": 0.5, "y": 0.8 },
      "startTime": 0,
      "endTime": 10
    }
  ],
  "filter": {
    "name": "vibrant",
    "saturation": 0.3,
    "warmth": 0.2
  }
}
```
**Response**:
```json
{
  "success": true,
  "message": "Reel upload initialized",
  "data": {
    "mediaId": "65abc12ef34567890abcdef1",
    "reelId": "65abc12ef34567890abcdef2",
    "uploadUrl": "https://stub-upload.chefooz.com/...",
    "expiresAt": "2026-02-14T12:15:00.000Z"
  }
}
```

---

### **POST** `/api/v1/media/presigned-upload-url`
**Purpose**: Get secure S3 upload URL  
**Auth**: Required (JWT)  
**Request Body**:
```json
{
  "mediaId": "65abc12ef34567890abcdef1",
  "fileType": "video/mp4"
}
```
**Response**:
```json
{
  "success": true,
  "message": "Presigned URL generated",
  "data": {
    "uploadUrl": "https://chefooz-media-uat.s3.ap-south-1.amazonaws.com/uploads/...",
    "s3Key": "uploads/65abc12ef34567890abcdef1/original.mp4",
    "expiresIn": 900
  }
}
```

---

### **POST** `/api/v1/media/complete-upload`
**Purpose**: Finalize upload and trigger processing  
**Auth**: Required (JWT)  
**Request Body**:
```json
{
  "mediaId": "65abc12ef34567890abcdef1",
  "s3Key": "uploads/65abc12ef34567890abcdef1/original.mp4",
  "etag": "abc123def456",
  "trim": {
    "startSec": 15.5,
    "endSec": 45.2
  }
}
```
**Response**:
```json
{
  "success": true,
  "message": "Upload complete, processing started",
  "data": {
    "success": true,
    "mediaId": "65abc12ef34567890abcdef1",
    "inputKey": "uploads/65abc12ef34567890abcdef1/original.mp4"
  }
}
```

---

### **GET** `/api/v1/media/:id/status`
**Purpose**: Poll media processing status  
**Auth**: Required (JWT)  
**Response**:
```json
{
  "success": true,
  "message": "Media status retrieved",
  "data": {
    "status": "ready",
    "variants": [
      {
        "quality": "720p",
        "url": "https://chefooz-media-output.s3.amazonaws.com/...",
        "width": 1280,
        "height": 720,
        "bitrate": 2500000
      },
      {
        "quality": "480p",
        "url": "https://chefooz-media-output.s3.amazonaws.com/...",
        "width": 854,
        "height": 480,
        "bitrate": 1200000
      }
    ],
    "thumbnailUrl": "https://chefooz-media-output.s3.amazonaws.com/..."
  }
}
```

---

### **GET** `/api/v1/media/upload-quotas`
**Purpose**: Get comprehensive quota status for all reel types  
**Auth**: Required (JWT)  
**Response**:
```json
{
  "success": true,
  "message": "Quota status retrieved",
  "data": {
    "review": {
      "used": 2,
      "max": 5,
      "remaining": 3,
      "perOrderQuota": 1,
      "completedOrders": 5
    },
    "menuShowcase": {
      "perItemQuota": 1
    },
    "promotional": {
      "user": {
        "used": 3,
        "max": 10,
        "remaining": 7
      },
      "chef": {
        "used": 5,
        "max": 10,
        "remaining": 5
      }
    }
  }
}
```

---

### **POST** `/api/v1/media/chat/upload-init`
**Purpose**: Initialize image upload for chat attachments  
**Auth**: Required (JWT)  
**Request Body**:
```json
{
  "mimeType": "image/jpeg"
}
```
**Response**:
```json
{
  "success": true,
  "message": "Upload initialized",
  "data": {
    "uploadUrl": "https://chefooz-media-output.s3.amazonaws.com/...",
    "imageKey": "chat/uuid/original.jpg",
    "cdnUrl": "https://cdn.chefooz.com/chat/uuid/original.jpg"
  }
}
```

---

## 🎬 Video Processing Pipeline

### **Primary Pipeline: FFmpeg** (apps/media-processing)
**Use Cases**: All video processing (trim, overlays, filters)  
**Advantages**:
- ✅ Fine-grained control over encoding parameters
- ✅ Real-time trim support (start/end seconds)
- ✅ Text overlay burning with custom fonts
- ✅ Image filter application (color grading)
- ✅ Faster processing for short videos (<60s)

**Processing Steps**:
1. Download original video from S3 input bucket
2. Apply trim (if `trim.startSec`/`trim.endSec` provided)
3. Burn text overlays with position/rotation/timing
4. Apply image filters (brightness, contrast, saturation)
5. Generate 720p, 480p, 360p variants
6. Extract cover frame at `coverTimestampSec`
7. Upload variants + thumbnail to output bucket
8. Update Media document with URLs and mark READY

**Queue**: Bull queue `video-processing`  
**Worker**: `apps/chefooz-apis/src/modules/media-processing/workers/ffmpeg.worker.ts`  
**Retry**: 3 attempts with exponential backoff (5s, 25s, 125s)

---

### **Fallback Pipeline: AWS MediaConvert**
**Use Cases**: FFmpeg failure fallback, legacy support  
**Advantages**:
- ✅ Cloud-native, infinite scalability
- ✅ HLS playlist generation (adaptive bitrate)
- ✅ Automatic thumbnail extraction
- ✅ No server compute required

**Processing Steps**:
1. Create MediaConvert job with input S3 URL
2. MediaConvert transcodes to HLS + MP4 variants
3. Output written to S3 output bucket
4. MediaConvert webhook triggers status update
5. Media document updated with variant URLs

**Limitations**:
- ❌ No trim support (full video only)
- ❌ No text overlay burning
- ❌ No custom cover frame selection
- ❌ Slower for short videos (30-90s processing time)

**Migration Path**: Phase out MediaConvert as FFmpeg stabilizes in production

---

## 📁 S3 Bucket Strategy

### **3-Bucket Architecture**

| Bucket | Purpose | Lifecycle | Access |
|--------|---------|-----------|--------|
| **chefooz-media-uat** | User uploads (presigned URLs) | 7 days → Delete | Write-only (presigned) |
| **chefooz-media-input** | MediaConvert input | 7 days → Delete | Read-only (MediaConvert) |
| **chefooz-media-output** | Processed videos (public CDN) | 90 days → Glacier | Public read |

**Workflow**:
```
Client → Upload → [UAT Bucket]
                        ↓ Copy (on complete-upload)
                  [Input Bucket]
                        ↓ MediaConvert/FFmpeg
                  [Output Bucket] → CDN → Client
```

**Cost Optimization**:
- UAT & Input buckets auto-delete after 7 days (temporary storage)
- Output bucket transitions to Glacier after 90 days (archive)
- CloudFront CDN caches popular videos (reduces S3 egress costs)

---

## 🔔 Notification System

### **Reel Ready Notification**
**Trigger**: Media status changes to READY  
**Recipient**: Reel creator (`media.userId`)  
**Event Type**: `reel.ready`  
**Payload**: `{ reelId: string }`  
**Delivery**: Push notification + in-app alert

---

### **Tag Notifications** (Unified, Deduplicated)
**Trigger**: Reel created with user tags  
**Recipients**: All unique tagged users (caption + positioned tags, deduplicated)  
**Event Type**: `engagement` → `type: 'tag'`  
**Business Rules**:
- ✅ Self-tags filtered out (can't tag yourself)
- ✅ Duplicates removed (same user in caption AND positioned tag = 1 notification)
- ✅ Max 20 tags per reel (platform limit)
- ✅ Sent async (doesn't block reel creation)

**Notification Format**:
- **Title**: `@{username} tagged you`
- **Body**: `@{username} tagged you in a reel`
- **Deep Link**: Opens tagged reel in app

---

## 📊 Analytics & Monitoring

### **Key Metrics**
- **Upload Success Rate**: % of uploads reaching READY state
- **Average Processing Time**: Time from complete-upload to READY
- **Quota Utilization**: % of users hitting weekly promotional limits
- **FFmpeg vs MediaConvert Split**: % processed by each pipeline
- **Presigned URL Expiry Rate**: % of URLs expiring before use

### **Error Tracking**
- `MEDIA_NOT_FOUND`: Invalid mediaId in presigned-url or complete-upload
- `REVIEW_REEL_QUOTA_EXCEEDED`: User out of review reel slots
- `MENU_SHOWCASE_QUOTA_EXCEEDED`: Chef exceeded 1 reel per menu item
- `PROMOTIONAL_REEL_QUOTA_EXCEEDED_USER/CHEF`: Weekly promo limit hit
- `S3_PRESIGN_FAILED`: AWS S3 presigned URL generation failed
- `UPLOAD_COMPLETION_FAILED`: File not found in S3 or copy failed
- `PROCESSING_FAILED`: FFmpeg/MediaConvert job failed (check logs)

---

## 🚀 Performance Characteristics

### **Upload Speed**
- **Direct S3 Upload**: ~500KB/s - 5MB/s (depends on user network)
- **Presigned URL Generation**: <50ms (in-memory, no DB query)
- **Upload Initialization**: ~200ms (DB write + quota checks)

### **Processing Speed**
| Video Length | FFmpeg (Primary) | MediaConvert (Fallback) |
|--------------|------------------|-------------------------|
| **15 sec** | ~10s | ~30s |
| **30 sec** | ~20s | ~45s |
| **60 sec** | ~40s | ~90s |
| **120 sec** | ~80s | ~180s |

**Note**: Times include trim, overlay, filter, variant generation, and S3 upload.

---

## 🔄 Idempotency & Retry Logic

### **Client-Side Retry Safety**
**Problem**: Network failures during upload → User retries → Duplicate reels?  
**Solution**: Client-generated `uploadId` (UUID) in `UploadReelDto`

**Backend Behavior**:
1. Check if `Media` document with same `userId` + `uploadId` exists
2. If exists → Return existing media/reel IDs with `resumeState`
3. `resumeState` tells client what to do:
   - `status: UPLOADING, canResume: true` → Resume S3 upload
   - `status: PROCESSING, canResume: false` → Wait for completion
   - `status: READY, canResume: false` → Navigate to reel view
   - `status: FAILED, canResume: false` → Show error, retry with new uploadId

**MongoDB Index**: `{ userId: 1, uploadId: 1 }` (unique, sparse) prevents duplicate documents

---

## 🎓 Example User Journeys

### **Journey 1: User Uploads Review Reel for Delivered Order**
1. User orders "Paneer Tikka" from Chef Ramesh → Order #12345
2. Order delivered successfully (status: DELIVERED)
3. User opens Chefooz app → "Create Reel" button
4. Selects video from gallery, adds caption "Amazing tikka from @ChefRamesh! 🔥"
5. Tags Chef Ramesh, adds location "Mumbai, India"
6. Links to Order #12345 (app auto-suggests recent delivered orders)
7. Uploads video → Backend validates:
   - ✅ Order belongs to user
   - ✅ Order is DELIVERED
   - ✅ User hasn't exceeded review reel quota (5 orders = 5 slots, only used 2)
8. Video processes → Reel goes live → Chef Ramesh notified of tag

---

### **Journey 2: Chef Uploads Menu Showcase Reel**
1. Chef adds "Butter Chicken" to menu → Menu Item #6789
2. Chef creates reel showing cooking process
3. Links reel to Menu Item #6789
4. Backend validates:
   - ✅ Menu item exists and active
   - ✅ No existing showcase reel for this item (1 reel per item limit)
5. Reel published → Appears in chef's profile → Customers discover via explore feed

---

### **Journey 3: Chef Exhausts Promotional Quota**
1. Chef uploads 10 promotional reels this week (no order/menu linking)
2. Attempts 11th promotional reel
3. Backend rejects:
   - ❌ `contentSource='chef'` pool exhausted (10/10 used)
   - ❌ Error: `PROMOTIONAL_REEL_QUOTA_EXCEEDED_CHEF`
4. App shows: "You've reached your weekly promotional limit (10 reels). Quota resets in 3 days."
5. Chef can still upload review reels (different quota pool) or wait for reset

---

## 🔮 Future Enhancements

### **Planned Features** (Roadmap 2026)
- [ ] **Live Streaming**: RTMP ingest → HLS output → Real-time reel creation
- [ ] **Collaborative Reels**: Multi-user video stitching (duets/remixes)
- [ ] **Music Library Integration**: Licensed background music for reels
- [ ] **AI Auto-Captions**: Speech-to-text for accessibility
- [ ] **Smart Crop**: Auto-detect face/food and crop to 9:16 aspect ratio
- [ ] **Thumbnail A/B Testing**: Upload multiple covers, AI picks best performer

### **Under Consideration**
- [ ] **Video Ads**: Mid-roll ads for monetization (YouTube model)
- [ ] **Creator Fund**: Pay top creators based on views (TikTok model)
- [ ] **Premium Filters**: Paid filter packs for chefs (Snapchat model)

---

## 📚 Related Documentation

- **[Media Processing Technical Guide](./TECHNICAL_GUIDE.md)** - FFmpeg pipeline, AWS setup, video encoding
- **[Media QA Test Cases](./QA_TEST_CASES.md)** - Test scenarios, automation scripts
- **[Reels Module](../reels/FEATURE_OVERVIEW.md)** - Reel discovery, feed algorithm, engagement
- **[Stories Module](../stories/FEATURE_OVERVIEW.md)** - 24-hour ephemeral content
- **[Notification Module](../notification/FEATURE_OVERVIEW.md)** - Push notification delivery

---

**[SLICE_COMPLETE ✅]**  
*Media Module Feature Overview - Comprehensive business documentation complete*
