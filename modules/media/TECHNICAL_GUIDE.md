# Media Upload Module - Technical Guide

> **Module**: `apps/chefooz-apis/src/modules/media`  
> **Tech Stack**: NestJS, MongoDB (Media/Reel schemas), PostgreSQL (Orders/Users), AWS S3, Bull Queue, FFmpeg  
> **Last Updated**: April 25, 2026  
> **Maintainer**: Backend Team

---

## April 25, 2026 — Camera Filters & Live Preview (Food-Optimised Presets)

### What was implemented

A full filter pipeline for the upload flow — from live camera preview to server-side baking.

| Layer | Component / File | What it does |
|---|---|---|
| Preset definitions | `libs/types/src/lib/reel.types.ts` | `VideoFilter` interface + `VIDEO_FILTER_PRESETS` — 9 presets total |
| Live camera overlay | `LiveCameraView.tsx` — `activeFilter` prop | Applies `renderFilterOverlays()` View-tinting on top of the camera feed in real time |
| Camera strip UI | `CameraFilterSelector.tsx` (new) | Horizontal scroll strip at bottom of camera view — picks filter BEFORE recording |
| Post-media picker | `FilterPickerSheet.tsx` | Full bottom sheet with Presets + Custom sliders — triggered by the Filter rail button |
| Post-media preview | `VideoFilterPreview.tsx` | Same View-tinting approach applied over the recorded video/image preview |
| Upload store | `upload-v2.store.ts` | `filter?: VideoFilter` + `setFilter()` — carries selected filter to the share screen |
| Edit screen wiring | `edit.tsx` | `selectedCameraFilter` state, `CameraFilterSelector` in camera view, Filter button in action rail, pre-populate store on recording end |

### Filter presets (9 total, food-centric order)

| Key | Display name | Best for |
|---|---|---|
| `original` | Original | No filter |
| `vibrant` | Vibrant | Curries, salads, colourful dishes — +saturation +contrast |
| `warm` | Warm | Home cooking, cosy plating — warm orange tones |
| `golden` | Golden | Baked goods, chai, Indian sweets — deep gold warmth +brightness |
| `vintage` | Vintage | Artisan bakery, nostalgic food photography — warm +vignette |
| `cool` | Cool | Seafood, sushi, clean-eating plates — slight blue shift |
| `fresh` | Fresh | Salads, smoothies, healthy bowls — crisp green boost |
| `dramatic` | Dramatic | Fine-dining, dark plating — high contrast +vignette |
| `bw` | B&W | Editorial / artisan mono — full desaturation |

### Preview approach (no Skia dependency)

Filters are previewed as a stack of semi-transparent `View` overlays placed above the camera/video using `StyleSheet.absoluteFill`. Each `VideoFilter` field maps to a colour tint:

| Field | Overlay colour | Opacity scaling |
|---|---|---|
| `brightness > 0` | `#FFFFFF` (white) | `abs(brightness) * 0.28` |
| `brightness < 0` | `#000000` (black) | `abs(brightness) * 0.28` |
| `saturation < 0` | `#808080` (grey) | `abs(saturation) * 0.38` |
| `warmth > 0` | `#FFA726` (orange) | `abs(warmth) * 0.20` |
| `warmth < 0` | `#42A5F5` (blue) | `abs(warmth) * 0.20` |
| `contrast > 0` | `#000000` (darken) | `abs(contrast) * 0.10` |
| `vignette > 0` | `#000000` (darken) | `vignette * 0.22` |

This is a visual approximation. Actual LUT-grade colour transforms happen server-side in FFMPEG (see MediaConvert LUT section below when implemented).

**Why not Skia?** `@shopify/react-native-skia` is not installed. The View-tinting approach is zero-dependency and has near-zero performance overhead (no extra GPU passes). It is sufficient for a "preview" that communicates the filter mood to the user before uploading.

### Camera filter UX flow

1. **Camera mode** (no media selected): `CameraFilterSelector` strip is visible at the bottom of the camera view. User swipes through 9 filter thumbnails. Selected filter is shown as a View overlay on the live preview via `LiveCameraView.activeFilter`.
2. **Recording**: `CameraFilterSelector` is hidden while recording is in progress (`!isRecording` guard). The overlay remains active.
3. **Recording ends**: `handleRecordingEnd` pre-populates `setFilter(VIDEO_FILTER_PRESETS[selectedCameraFilter])` into the store so the filter is already selected on the share screen.
4. **Video/image editing**: A **Filter** button (palette icon) in the right action rail opens `FilterPickerSheet` for full control — 9 presets + custom adjustment sliders (brightness, contrast, saturation, warmth, vignette).
5. **Share**: The `filter` field in the store is sent to the backend as part of the reel creation DTO.

### Server-side filter baking (pending — backend sprint)

The `filter` field (`{ name, preset, brightness, contrast, saturation, warmth, vignette }`) is passed to the media service with the upload request. FFMPEG filter baking implementation (LUT `.cube` files per preset, `MediaConvert LUTSettings`) is a separate backend sprint.

---

## April 23, 2026 — LiveCameraView migrated to React Native Vision Camera v5

### Motivation

`expo-camera` is a thin wrapper around AVFoundation / Camera2 with no Frame Processor API.
For a UGC platform competing with TikTok/Instagram, it is insufficient once real-time filters,
multi-camera streaming, or per-capture flash control are required.

### Changes

| Area | Before | After |
|---|---|---|
| Camera library | `expo-camera ~16.1.11` | `react-native-vision-camera ^5.0.4` |
| Camera component | `<CameraView>` | `<Camera>` (from rnvc) |
| Permission hooks | `useCameraPermissions()` / `useMicrophonePermissions()` (plural) | `useCameraPermission()` / `useMicrophonePermission()` (singular) |
| Device selection | `facing` prop | `useCameraDevice('back'\|'front')` hook, both loaded; selected by prop |
| Photo capture | `takePictureAsync()` → `{ uri }` | `takePhoto()` → `PhotoFile { path }`, normalised to `file://` URI |
| Video recording | `recordAsync()` (Promise, race condition prone) | `startRecording({ onRecordingFinished, onRecordingError })` + `stopRecording()` — callback-based |
| Zoom | `0–1` passed directly to expo-camera | `device.neutralZoom + (norm * clamp(maxZoom, 8))` — true device range |
| Torch | `enableTorch` prop | `torch='on'\|'off'` prop on Camera |
| Frame Processors | ❌ not available | ✅ architecture ready via `react-native-worklets-core ^1.6.3` |

### File changed

`apps/chefooz-app/src/components/upload/LiveCameraView.tsx`

### app.json plugin added

```json
["react-native-vision-camera", {
  "cameraPermissionText": "Allow $(PRODUCT_NAME) to access your camera for recording reels.",
  "microphonePermissionText": "Allow $(PRODUCT_NAME) to use your microphone for recording audio.",
  "enableMicrophonePermission": true
}]
```

### URI normalisation constraint

VisionCamera `PhotoFile.path` and `VideoFile.path` may or may not carry a `file://` prefix depending
on platform and device. Both are normalised in `LiveCameraView.tsx` before any URI is passed to callbacks:

```ts
const uri = path.startsWith('file://') ? path : `file://${path}`;
```

All callers in `edit.tsx` continue to receive `string` URIs — no change to parent code.

### Zoom range constraint

The `MAX_ZOOM_MULTIPLIER = 8` cap prevents exposing extreme digital zoom (some devices report
`device.maxZoom = 128`). Gesture input `0–1` is linearly mapped to `neutralZoom → min(8, maxZoom)`.
`neutralZoom` is the 1× focal length, so gesture `0` always equals no zoom, not wide-angle.

### Frame Processor readiness

`react-native-worklets-core ^1.6.3` is already in `apps/chefooz-app/package.json`.
To add a real-time filter:
1. Import `useFrameProcessor` from `react-native-vision-camera`
2. Implement your worklet with the `'worklet'` directive
3. Pass it as the `frameProcessor` prop on `<Camera />`
4. Add `react-native-skia` for canvas-based drawing (Snapchat-style stickers/masks)

### Camera performance constraints (April 2026)

Three VisionCamera v5-native performance tweaks applied to `LiveCameraView.tsx` to achieve native-like smoothness:

| Optimisation | API | Rationale |
|---|---|---|
| Single-lens back camera | `useCameraDevice('back', { physicalDevices: ['wide-angle'] })` | Preference scoring that steers toward a single-lens device instead of the triple-camera system, reducing session startup time. Soft filter — won't fail if only a multi-lens device is available. |
| YUV pixel format | `constraints: [{ pixelFormat: 'yuv-420-8-bit-video' }]` | Camera's native format. Prevents a YUV→RGB conversion stage that adds CPU/memory overhead, especially relevant when a frame processor is added later. |
| SDR dynamic range | `constraints: [{ videoDynamicRange: CommonDynamicRanges.ANY_SDR }]` | Explicitly opts out of 10-bit HDR (Dolby Vision / HLG) which newer iPhones may auto-select. SDR is lighter to encode and sufficient for 1080p UGC reels. |

**Why `enableBufferCompression` / `videoHdr` are not used**: Those are VisionCamera v4 props. In v5 the equivalent control is the `constraints` array on `<Camera>`. There is no `enableBufferCompression` in v5.

**Note on `physicalDevices` filter**: The string in v5 is `'wide-angle'` (not `'wide-angle-camera'` which was the v4 string). Using the wrong string silently falls back to default device selection.

---

## March 14, 2026 — PostCropModal portrait ratio changed to 4:5

### Problem
`9:16` was too tall for the post-photo editing flow. It did not align well with the upload edit preview,
and it provided a portrait crop option that was less consistent with standard feed-post proportions.

### Root cause
The post editor was offering a portrait preset optimized for reels rather than feed posts. That made
the portrait crop mode taller than necessary, harder to fit edge-to-edge within the available post layout,
and more difficult to validate visually against the edit preview.

An additional preview mismatch existed in `PostCropModal.tsx`: pan and scale were applied on the same
animated layer. On device, this could couple horizontal translation to the current zoom level and make the
saved crop look shifted left even when the crop rectangle math itself was correct.

### Fix
- Replaced the portrait post ratio option from `9:16` to `4:5`
- Updated portrait crop output from `1080 × 1920` to `1080 × 1350`
- Extracted shared geometry helpers into `apps/chefooz-app/src/components/upload/PostCropModal.utils.ts`
- Kept frame sizing, contained bitmap sizing, base scale, clamp bounds, and crop rect generation on the same helper path
- Updated `PostCropModal.tsx` and the upload edit preview so portrait posts now use `4:5` consistently
- Separated pan and scale into nested animated layers in `PostCropModal.tsx` so preview translation matches stored crop offsets exactly
- Added regression coverage in `apps/chefooz-app/src/components/upload/PostCropModal.utils.spec.ts`

### Constraint
For post-photo cropping, preview layout and crop-output math must always derive from the same
rendered bitmap dimensions. The supported post portrait preset is now `4:5`, matching standard Instagram portrait posts more closely than `9:16`.

---

## March 2026 — PostCropModal UX Redesign

### Problem
The original `PostCropModal` used a sequential "Next →" flow — the user had to crop image 1,
press Next, crop image 2, press Next, and could not return to a previous image once confirmed.
This was unusable for multi-image POST uploads.

### Solution
`PostCropModal.tsx` was rewritten with a thumbnail-driven, non-sequential UX:

| Aspect | Old behaviour | New behaviour |
|---|---|---|
| Navigation | Sequential Next → Done | Thumbnail strip — tap any image to jump freely |
| Crop state | Single Animated pair, reset on Next | Per-image `cropStatesRef[]` persisted when switching |
| Processing | Per-image on Next press | All images processed together when Done pressed |
| Header button | "Next →" / "Done" (dynamic) | Always "Done" |
| Progress feedback | None | Orange progress bar (0–100%) during batch |
| Ratio change | Reset current image only | Rebuilds cover-fill defaults for ALL images |

### Key implementation details

**Per-image crop state** (`cropStatesRef: Ref<Array<{panX, panY, scale, baseScale}>>`)
- Initialised by `initCropStates()` on open with cover-fill defaults for each image
- `saveCropState(idx)` writes current animated values into `cropStatesRef.current[idx]`
- `applyCropState(state)` restores animated values from a saved state
- Switching: `saveCropState(current) → setCurrentIndex(new) → applyCropState(new)`
- Ratio change: `saveCropState(current) → initCropStates(all) → applyCropState(current)`

**Batch processing on Done**
- `handleDone` saves current state, then runs `Promise.all` for every image simultaneously
- Progress bar increments as each image resolves

**Clamp math fix**
The old `_clamp` used `img.width * s` (incorrect — `resizeMode="contain"` means
rendered size ≠ pixel size × scale). The correct version computes "contain" rendered size first:
```ts
const scaledW = renderedW * s;  // renderedW = contain-fitted width
const maxX = Math.max(0, (scaledW - frame.width) / 2);
```

**File**: `apps/chefooz-app/src/components/upload/PostCropModal.tsx`

## March 12, 2026 — Upload Enhancements

### 1. Reel/Post Toggle Visibility (edit.tsx)
- The Reel/Post content-type toggle in the upload edit screen is now **only shown when the selected media is an image**.
- When a **video** is selected, the toggle is hidden and `contentType` is automatically forced to `'REEL'` (videos can only be Reels).
- A `useEffect` in `EditScreen` watches `media?.type` and calls `setContentType('REEL')` immediately if a video is detected.

### 2. Multiple Image Selection for POST (useVideoPicker + store)
- `useVideoPicker` now exposes `pickMultipleImages()` which uses `allowsMultipleSelection: true` (up to 10 images).
- The upload store (`upload-v2.store.ts`) has a new `images: VideoAsset[]` field and `setImages(images)` action.
- `setImages` auto-sets `media` to the first image for backward compatibility with the single-upload pipeline.
- When `contentType === 'POST'` and gallery button is tapped in the edit screen, `pickMultipleImages()` is invoked instead of `pickVideo()`.
- All selected images are shown as a thumbnail strip at the bottom of the edit preview, with a count badge.
- **Note**: The current upload pipeline uploads only the first (primary) image. True multi-image carousel post upload is deferred pending backend `imageUrls[]` support.

### 3. ContentType End-to-End (DTO → DB → Feed)
- `UploadReelDto` now has `contentType?: string` (`'REEL'` | `'POST'`, default `'REEL'`).
- `media.service.ts` passes `contentType: dto.contentType || 'REEL'` when creating the reel document in MongoDB.
- `MediaUploadRequest` (shared types) has `contentType?: 'REEL' | 'POST'`.
- `share.tsx` passes `contentType` from the Zustand store to the upload request.
- Post-upload navigation: `'POST'` → `/(tabs)/home`; `'REEL'` → `/(tabs)/feed`.
- `home.tsx` now maps `reel.contentType === 'POST'` items as `type: 'post'` so `FeedPostCard` renders them instead of the video reel card.

---

## �🏗️ Architecture Overview

### **Module Structure**
```
apps/chefooz-apis/src/modules/media/
├── media.module.ts              # Module definition + dependency injection
├── media.controller.ts          # REST API endpoints
├── media.service.ts             # Core business logic (1378 lines)
├── media.schema.ts              # MongoDB schema for Media documents
├── image-upload.service.ts      # Image upload utilities (profiles/chat)
├── dto/
│   ├── upload-init.dto.ts       # Legacy upload DTO
│   ├── upload-reel.dto.ts       # Modern reel upload with metadata
│   ├── presigned-upload.dto.ts  # S3 presigned URL DTOs
│   ├── upload-complete.dto.ts   # Upload finalization DTO
│   ├── chat-upload-init.dto.ts  # Chat image upload DTO
│   └── linkable-product.dto.ts  # Product linking DTO
└── jobs/
    ├── generateThumbnail.worker.ts  # Thumbnail extraction (legacy)
    └── transcode.worker.ts          # Video transcoding (legacy)
```

**Related Modules**:
- `media-processing/` - FFmpeg video processing workers
- `reels/` - Reel document management (MongoDB)
- `notification/` - Push notifications for reel ready/tags
- `integrations/media-convert/` - AWS MediaConvert service wrapper

---

## 🗄️ Data Models

### **Media Schema** (MongoDB - Flexible Document Storage)
**Location**: `apps/chefooz-apis/src/modules/media/media.schema.ts`  
**Collection**: `media`  
**Purpose**: Metadata for uploaded videos/images

```typescript
export enum MediaType {
  REEL = 'reel',
  STORY = 'story',
  POST = 'post',
}

export enum MediaStatus {
  UPLOADING = 'uploading',    // Client uploading to S3
  PROCESSING = 'processing',  // FFmpeg/MediaConvert processing
  READY = 'ready',           // Video ready for playback
  FAILED = 'failed',         // Processing failed
}

export class MediaVariant {
  quality: string;   // '720p', '480p', '360p'
  url: string;       // S3 URL
  width: number;     // 1280, 854, 640
  height: number;    // 720, 480, 360
  bitrate: number;   // 2500000, 1200000, 800000
}

@Schema({ timestamps: true, collection: 'media' })
export class Media {
  @Prop({ required: true, index: true })
  userId: string;  // PostgreSQL user ID

  @Prop({ required: false })
  uploadId?: string;  // Client UUID for idempotency

  @Prop({ required: true, enum: MediaType, default: MediaType.REEL })
  type: MediaType;

  @Prop({ required: true, enum: MediaStatus, default: MediaStatus.UPLOADING })
  status: MediaStatus;

  @Prop({ required: true })
  durationSec: number;  // Video length in seconds

  @Prop()
  thumbnailUrl?: string;  // S3 URL for cover image

  @Prop({ type: [MediaVariantSchema], default: [] })
  variants?: MediaVariant[];  // Quality variants

  @Prop()
  linkedOrderItemId?: string;  // Legacy: Order item linking

  @Prop({ type: String, enum: ['user', 'chef'], default: 'user' })
  contentSource?: 'user' | 'chef';  // Quota pool identifier

  @Prop()
  fileName?: string;  // Original filename

  @Prop()
  mimeType?: string;  // 'video/mp4', 'image/jpeg'

  @Prop()
  sizeBytes?: number;  // File size

  @Prop()
  checksum?: string;  // MD5/SHA256 hash

  // MediaConvert Integration
  @Prop()
  jobId?: string;  // MediaConvert job ID

  @Prop({ type: String, enum: ['SUBMITTED', 'PROGRESSING', 'COMPLETE', 'ERROR'] })
  jobStatus?: string;

  @Prop({ type: Object })
  jobMetadata?: Record<string, any>;

  @Prop()
  errorMessage?: string;

  @Prop({ type: String, enum: ['ffmpeg', 'mediaconvert'] })
  processingMethod?: 'ffmpeg' | 'mediaconvert';

  @Prop()
  processingError?: string;

  @Prop({ type: String, enum: ['pending', 'approved', 'rejected', 'reviewing'] })
  moderationStatus?: 'pending' | 'approved' | 'rejected' | 'reviewing';

  @Prop({ type: Date, index: true })
  createdAt?: Date;

  @Prop({ type: Date })
  updatedAt?: Date;
}
```

**Indexes**:
```typescript
MediaSchema.index({ userId: 1, createdAt: -1 });  // User's media timeline
MediaSchema.index({ status: 1 });                 // Status filtering
MediaSchema.index({ userId: 1, uploadId: 1 }, { unique: true, sparse: true });  // Idempotency
```

---

### **Reel Schema** (MongoDB - Social Content)
**Location**: `apps/chefooz-apis/src/database/schemas/reel.schema.ts`  
**Collection**: `reels`  
**Purpose**: Social reel metadata (created in parallel with Media)

```typescript
export class Reel {
  @Prop({ required: true, index: true })
  userId: string;  // Creator (PostgreSQL user ID)

  @Prop({ required: true, index: true })
  mediaId: string;  // Reference to Media document

  @Prop({ required: true })
  caption: string;  // User-written caption

  @Prop({ type: [String], default: [] })
  hashtags: string[];  // For discovery (#foodie)

  @Prop()
  videoUrl?: string;  // Final processed video URL

  @Prop()
  thumbnailUrl?: string;  // Cover image URL

  @Prop({ required: true })
  durationSec: number;

  // Order Linking (Review Reels)
  @Prop()
  linkedOrderId?: string;  // Reference to Order table

  @Prop()
  creatorOrderValue?: number;  // Order total in paise (snapshotted)

  // Menu Linking (Showcase Reels)
  @Prop({ type: Object })
  linkedMenu?: {
    chefId: string;
    menuItemIds: string[];
    estimatedPaise: number;
    previewImage?: string;
  };

  // Reel Purpose (for quota enforcement)
  @Prop({ type: String, enum: ['USER_REVIEW', 'MENU_SHOWCASE', 'PROMOTIONAL'] })
  reelPurpose?: 'USER_REVIEW' | 'MENU_SHOWCASE' | 'PROMOTIONAL';

  // User Tagging
  @Prop({ type: [String], default: [] })
  taggedUserIds?: string[];  // Caption mentions (max 10)

  @Prop({ type: Array, default: [] })
  positionedTags?: Array<{
    userId: string;
    username: string;
    fullName?: string;
    x: number;  // 0-1 normalized
    y: number;  // 0-1 normalized
  }>;

  // Geolocation
  @Prop({ type: Object })
  location?: {
    lat: number;
    lng: number;
  };

  // FFMPEG Enhancements
  @Prop({ default: 1.0 })
  coverTimestampSec?: number;  // User-selected cover frame

  @Prop({ type: Array, default: [] })
  textOverlays?: Array<{
    id: string;
    content: string;
    position: { x: number; y: number };
    scale: number;
    rotation: number;
    startTime: number;
    endTime: number;
    style: any;
  }>;

  @Prop({ type: Object })
  filter?: {
    name: string;
    brightness?: number;
    contrast?: number;
    saturation?: number;
    warmth?: number;
    vignette?: number;
    preset?: string;
  };

  // Engagement Stats
  @Prop({ type: Object, default: () => ({ views: 0, likes: 0, comments: 0, saves: 0 }) })
  stats: {
    views: number;
    likes: number;
    comments: number;
    saves: number;
  };
}
```

---

## 🔌 API Implementation Details

### **Controller**: `media.controller.ts`

```typescript
@ApiTags('media')
@Controller('v1/media')
@UseGuards(JwtAuthGuard)
@ApiBearerAuth()
export class MediaController {
  constructor(
    private readonly mediaService: MediaService,
    private readonly imageUploadService: ImageUploadService,
  ) {}
}
```

**Key Endpoints**:

#### 1️⃣ **POST** `/api/v1/media/upload-reel`
**Handler**: `uploadReel()`  
**Purpose**: Initialize reel upload with full metadata  
**Flow**:
```typescript
async uploadReel(@Request() req: any, @Body() dto: UploadReelDto) {
  const userId = req.user.id;
  const data = await this.mediaService.initReelUpload(userId, dto);
  return { success: true, message: 'Reel upload initialized', data };
}
```

**Service Logic**: `mediaService.initReelUpload()`
1. **Idempotency Check**: If `uploadId` exists, return existing media/reel
2. **Quota Validation**: Check type-specific limits (review/menu/promo)
3. **File Validation**: Verify size/duration within tier limits
4. **Order/Menu Linking**: Validate and snapshot linked entities
5. **Create Documents**: Insert Media + Reel documents in MongoDB
6. **Send Notifications**: Unified tag notifications (deduplicated)
7. **Return URLs**: Stub upload URL (replaced by presigned URL in next step)

---

#### 2️⃣ **POST** `/api/v1/media/presigned-upload-url`
**Handler**: `getPresignedUploadUrl()`  
**Purpose**: Generate secure S3 upload URL  
**Flow**:
```typescript
async getPresignedUploadUrl(@Request() req: any, @Body() dto: GetPresignedUrlDto) {
  const userId = req.user.id;
  const data = await this.mediaService.getPresignedUploadUrl(userId, dto);
  return { success: true, message: 'Presigned URL generated', data };
}
```

**Service Logic**: `mediaService.getPresignedUploadUrl()`
```typescript
async getPresignedUploadUrl(userId: string, dto: GetPresignedUrlDto) {
  // 1. Verify media document exists
  const media = await this.mediaModel.findById(dto.mediaId);
  if (!media) throw new NotFoundException('Media document not found');

  // 2. Verify ownership
  if (String(media.userId) !== userId) {
    throw new ForbiddenException('Not authorized');
  }

  // 3. Verify status (must be UPLOADING)
  if (media.status !== MediaStatus.UPLOADING) {
    throw new BadRequestException(`Cannot upload in ${media.status} state`);
  }

  // 4. Generate S3 key
  const fileExtension = dto.fileType.split('/')[1];  // 'video/mp4' → 'mp4'
  const isThumbnail = (dto as any).isThumbnail === true;
  const s3Key = isThumbnail 
    ? `uploads/${dto.mediaId}/thumbnail.${fileExtension}`
    : `uploads/${dto.mediaId}/original.${fileExtension}`;

  // 5. Generate presigned PUT URL
  const command = new PutObjectCommand({
    Bucket: this.uploadBucket,  // 'chefooz-media-uat'
    Key: s3Key,
    ContentType: dto.fileType,
  });
  const expiresIn = 900;  // 15 minutes
  const uploadUrl = await getSignedUrl(this.s3Client, command, { expiresIn });

  // 6. Update media with thumbnail URL (if thumbnail upload)
  if (isThumbnail) {
    const thumbnailUrl = `https://${this.uploadBucket}.s3.${this.region}.amazonaws.com/${s3Key}`;
    await this.mediaModel.findByIdAndUpdate(dto.mediaId, { thumbnailUrl });
  }

  return { uploadUrl, s3Key, expiresIn };
}
```

**Security Features**:
- ✅ 15-minute expiration (prevents URL reuse)
- ✅ Content-Type validation (only video MIME types allowed)
- ✅ Ownership verification (can't upload to other user's media)
- ✅ Status check (can't re-upload READY/PROCESSING media)

---

#### **POST** `/api/v1/media/menu-image/presign` *(Added March 2026)*
**Handler**: `presignMenuImages()`  
**Purpose**: Generate 3 presigned PUT URLs for menu item image variants (1:1, 4:5, 16:9)  
**Auth**: JWT required (class-level `@UseGuards(JwtAuthGuard)`)  
**Response**:
```json
{
  "uploadId": "uuid-v4",
  "ratio1x1": { "uploadUrl": "...", "s3Key": "menu-images/{uploadId}/1x1.jpg", "finalUrl": "..." },
  "ratio4x5": { "uploadUrl": "...", "s3Key": "menu-images/{uploadId}/4x5.jpg", "finalUrl": "..." },
  "ratio16x9": { "uploadUrl": "...", "s3Key": "menu-images/{uploadId}/16x9.jpg", "finalUrl": "..." },
  "expiresIn": 900
}
```
**Bucket**: `chefooz-media-output` (output bucket — no MediaConvert processing)  
**Expiry**: 15 minutes  
**Method**: `ImageUploadService.generateMenuImagePresignedUrls()` (no user ID required — static prefix)

---

#### 3️⃣ **POST** `/api/v1/media/complete-upload`
**Handler**: `completeUpload()`  
**Purpose**: Verify S3 upload and trigger processing  
**Flow**:
```typescript
async completeUpload(@Request() req: any, @Body() dto: CompleteUploadDto) {
  const userId = req.user.id;
  const data = await this.mediaService.completeUpload(userId, dto);
  return { success: true, message: 'Upload complete, processing started', data };
}
```

**Service Logic**: `mediaService.completeUpload()`
```typescript
async completeUpload(userId: string, dto: CompleteUploadDto) {
  const media = await this.mediaModel.findById(dto.mediaId);
  
  // 1. Verify ownership
  if (String(media.userId) !== userId) throw new ForbiddenException();

  // 2. Verify file exists in upload bucket
  const headCommand = new HeadObjectCommand({
    Bucket: this.uploadBucket,
    Key: dto.s3Key,
  });
  await this.s3Client.send(headCommand);  // Throws if file doesn't exist

  // 3. Copy upload bucket → input bucket (for MediaConvert)
  const inputKey = `uploads/${dto.mediaId}/original.mp4`;
  const copyCommand = new CopyObjectCommand({
    CopySource: `${this.uploadBucket}/${dto.s3Key}`,
    Bucket: this.inputBucket,  // 'chefooz-media-input'
    Key: inputKey,
  });
  await this.s3Client.send(copyCommand);

  // 4. Update media status
  await this.mediaModel.findByIdAndUpdate(dto.mediaId, {
    status: MediaStatus.PROCESSING,
    s3Key: inputKey,
    uploadCompletedAt: new Date(),
  });

  // 5. Enqueue FFmpeg processing job
  const reel = await this.reelModel.findOne({ mediaId: dto.mediaId });
  await this.videoProcessingQueue.add('process-video', {
    mediaId: dto.mediaId,
    s3Key: inputKey,
    trim: dto.trim,  // Optional: { startSec, endSec }
    coverTimestampSec: reel?.coverTimestampSec,
    textOverlays: reel?.textOverlays,
    filter: reel?.filter,
    photoDurationSec: dto.photoDurationSec,  // Image-to-video conversion
    userId,
  }, {
    attempts: 3,
    backoff: { type: 'exponential', delay: 5000 },
  });

  return { success: true, mediaId: dto.mediaId, inputKey };
}
```

**Error Handling**:
- `HeadObjectCommand` throws if file not in S3 → `UPLOAD_COMPLETION_FAILED`
- `CopyObjectCommand` throws if copy fails → `UPLOAD_COMPLETION_FAILED`
- Media marked as FAILED if any step throws

---

#### 4️⃣ **GET** `/api/v1/media/:id/status`
**Handler**: `getMediaStatus()`  
**Purpose**: Poll processing status (for client loading screen)  
**Flow**:
```typescript
async getMediaStatus(@Request() req: any, @Param('id') mediaId: string) {
  const userId = req.user.id;
  const data = await this.mediaService.getMediaStatus(mediaId, userId);
  return { success: true, message: 'Media status retrieved', data };
}
```

**Service Logic**:
```typescript
async getMediaStatus(mediaId: string, userId: string) {
  const media = await this.mediaModel.findById(mediaId);
  if (!media) throw new NotFoundException();
  if (media.userId !== userId) throw new ForbiddenException();

  return {
    status: media.status,  // 'uploading', 'processing', 'ready', 'failed'
    variants: media.variants,  // Array of quality variants (if ready)
    thumbnailUrl: media.thumbnailUrl,  // Cover image URL
  };
}
```

**Client Polling Strategy**:
```typescript
// Recommended polling intervals
const pollInterval = {
  uploading: 2000,    // 2s (fast, waiting for client upload)
  processing: 5000,   // 5s (medium, waiting for FFmpeg)
  ready: null,        // Stop polling
  failed: null,       // Stop polling, show error
};
```

---

#### 5️⃣ **GET** `/api/v1/media/upload-quotas`
**Handler**: `getUploadQuotas()`  
**Purpose**: Check available upload slots for all reel types  
**Flow**:
```typescript
async getUploadQuotas(@Request() req: any) {
  const userId = req.user.id;
  const data = await this.mediaService.getUploadQuotas(userId);
  return { success: true, message: 'Quota status retrieved', data };
}
```

**Service Logic**: `mediaService.getUploadQuotas()`
```typescript
async getUploadQuotas(userId: string) {
  // 1. Count completed orders
  const completedOrdersCount = await this.orderRepository.count({
    where: { userId, status: OrderStatus.DELIVERED },
  });

  // 2. Count existing review reels
  const existingReviewReelsCount = await this.reelModel.countDocuments({
    userId,
    reelPurpose: 'USER_REVIEW',
    status: { $ne: 'DELETED' },
  });

  // 3. Count promotional reels this week (user role)
  const oneWeekAgo = new Date(Date.now() - 7 * 24 * 60 * 60 * 1000);
  const weeklyPromotionalCountUser = await this.reelModel.countDocuments({
    userId,
    reelPurpose: 'PROMOTIONAL',
    contentSource: 'user',
    createdAt: { $gte: oneWeekAgo },
    status: { $ne: 'DELETED' },
  });

  // 4. Count promotional reels this week (chef role)
  const weeklyPromotionalCountChef = await this.reelModel.countDocuments({
    userId,
    reelPurpose: 'PROMOTIONAL',
    contentSource: 'chef',
    createdAt: { $gte: oneWeekAgo },
    status: { $ne: 'DELETED' },
  });

  // 5. Call domain policy function
  const { getReelQuotaStatus } = await import('@chefooz-app/domain');
  const envConfig = getEnvConfig();
  
  return getReelQuotaStatus(
    completedOrdersCount,
    existingReviewReelsCount,
    weeklyPromotionalCountUser,
    weeklyPromotionalCountChef,
    envConfig
  );
}
```

**Response Format**:
```json
{
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
    "user": { "used": 3, "max": 10, "remaining": 7 },
    "chef": { "used": 5, "max": 10, "remaining": 5 }
  }
}
```

---

## 🎬 Video Processing Pipeline

### **FFmpeg Worker** (Primary Pipeline)
**Location**: `apps/chefooz-apis/src/modules/media-processing/workers/ffmpeg.worker.ts`  
**Queue**: Bull queue `video-processing`  
**Job Data**:
```typescript
interface VideoProcessingJob {
  mediaId: string;
  s3Key: string;
  trim?: { startSec: number; endSec: number };
  coverTimestampSec?: number;
  textOverlays?: TextOverlay[];
  filter?: VideoFilter;
  photoDurationSec?: number;
  userId: string;
}
```

**Processing Steps**:
1. **Download**: Fetch original video from S3 input bucket
2. **Trim** (if provided): `ffmpeg -i input.mp4 -ss {startSec} -to {endSec} -c copy trimmed.mp4`
3. **Generate Variants**:
   ```bash
   # 720p variant
   ffmpeg -i input.mp4 -vf scale=1280:720 -b:v 2500k -c:a aac -b:a 128k 720p.mp4
   
   # 480p variant
   ffmpeg -i input.mp4 -vf scale=854:480 -b:v 1200k -c:a aac -b:a 96k 480p.mp4
   
   # 360p variant
   ffmpeg -i input.mp4 -vf scale=640:360 -b:v 800k -c:a aac -b:a 64k 360p.mp4
   ```
4. **Extract Thumbnail**: `ffmpeg -i input.mp4 -ss {coverTimestampSec} -vframes 1 thumbnail.jpg`
5. **Apply Filters** (if provided):
   ```bash
   ffmpeg -i input.mp4 -vf "eq=brightness={b}:contrast={c}:saturation={s}" output.mp4
   ```
6. **Burn Text Overlays** (if provided):
   ```bash
   ffmpeg -i input.mp4 -vf "drawtext=text='{content}':x={x}:y={y}:fontsize={size}" output.mp4
   ```
7. **Upload to S3**: Upload all variants + thumbnail to output bucket
8. **Update Media**: Mark as READY with variant URLs
9. **Update Reel**: Set `videoUrl` and `thumbnailUrl`
10. **Send Notification**: Dispatch `reel.ready` push notification

**Retry Logic**:
- 3 attempts with exponential backoff (5s, 25s, 125s)
- If all attempts fail → Mark media as FAILED → Log error → Alert ops team

---

### **AWS MediaConvert** (Fallback Pipeline)
**Location**: `apps/chefooz-apis/src/integrations/media-convert/mediaconvert.service.ts`  
**Trigger**: FFmpeg failure OR explicitly requested (legacy)

**Job Configuration**:
```typescript
const jobSettings = {
  Inputs: [{
    FileInput: `s3://${inputBucket}/${s3Key}`,
    VideoSelector: {},
    AudioSelectors: { 'Audio Selector 1': {} },
  }],
  OutputGroups: [
    {
      Name: 'HLS',
      OutputGroupSettings: {
        Type: 'HLS_GROUP_SETTINGS',
        HlsGroupSettings: {
          Destination: `s3://${outputBucket}/${outputPrefix}hls/`,
          SegmentLength: 10,
          MinSegmentLength: 0,
        },
      },
      Outputs: [
        {
          NameModifier: '-720p',
          VideoDescription: {
            Width: 1280,
            Height: 720,
            CodecSettings: { Codec: 'H_264', H264Settings: { Bitrate: 2500000 } },
          },
        },
        {
          NameModifier: '-480p',
          VideoDescription: {
            Width: 854,
            Height: 480,
            CodecSettings: { Codec: 'H_264', H264Settings: { Bitrate: 1200000 } },
          },
        },
      ],
    },
    {
      Name: 'Thumbnails',
      OutputGroupSettings: {
        Type: 'FILE_GROUP_SETTINGS',
        FileGroupSettings: {
          Destination: `s3://${outputBucket}/${outputPrefix}thumbnails/`,
        },
      },
      Outputs: [{
        ContainerSettings: { Container: 'RAW' },
        VideoDescription: {
          CodecSettings: { Codec: 'FRAME_CAPTURE', FrameCaptureSettings: {} },
        },
      }],
    },
  ],
};
```

**Limitations**:
- ❌ No trim support (processes full video)
- ❌ No custom cover frame selection (auto-extracts at 1s)
- ❌ No text overlay burning
- ❌ No filter application
- ✅ HLS playlist generation (adaptive bitrate)
- ✅ Cloud-native (no server compute)

**Migration Strategy**: Deprecate MediaConvert as FFmpeg stabilizes

---

## 🔐 Security Implementation

### **1. Presigned URL Security**
```typescript
// Generate presigned PUT URL with 15-minute expiration
const command = new PutObjectCommand({
  Bucket: 'chefooz-media-uat',
  Key: `uploads/${mediaId}/original.mp4`,
  ContentType: 'video/mp4',  // Enforces MIME type
});
const uploadUrl = await getSignedUrl(s3Client, command, { expiresIn: 900 });
```

**Security Properties**:
- ✅ **Temporary Access**: URL expires in 15 minutes
- ✅ **Write-Only**: Grants PUT permission only (can't read other files)
- ✅ **Path Scoped**: Can only upload to specified S3 key
- ✅ **Content-Type Locked**: S3 rejects uploads with wrong MIME type

---

### **2. Ownership Validation**
```typescript
// Every endpoint verifies media belongs to user
const media = await this.mediaModel.findById(mediaId);
if (String(media.userId) !== userId) {
  throw new ForbiddenException('Not authorized to access this media');
}
```

**Prevents**:
- ❌ User A uploading to User B's media document
- ❌ User A polling User B's processing status
- ❌ User A completing User B's upload

---

### **3. Quota Enforcement** (Type-Specific)
```typescript
// Review Reel Quota Check
const completedOrdersCount = await this.orderRepository.count({
  where: { userId, status: OrderStatus.DELIVERED },
});
const existingReviewReelsCount = await this.reelModel.countDocuments({
  userId,
  reelPurpose: 'USER_REVIEW',
  status: { $ne: 'DELETED' },
});

const quotaCheck = canUploadReviewReel(
  completedOrdersCount,
  existingReviewReelsCount,
  envConfig
);

if (!quotaCheck.allowed) {
  throw new HttpException(
    {
      success: false,
      message: quotaCheck.reason,
      errorCode: quotaCheck.errorCode,
      data: quotaCheck.metadata,
    },
    HttpStatus.FORBIDDEN
  );
}
```

**Policy Functions** (from `libs/domain`):
- `canUploadReviewReel()` - Validates order-based quota
- `canUploadMenuShowcaseReel()` - Validates menu item quota
- `canUploadPromotionalReel()` - Validates weekly rolling quota

---

### **4. File Validation**
```typescript
// Validate file size and duration within tier limits
const fileSizeMb = dto.sizeBytes / (1024 * 1024);
const fileConstraintCheck = isFileWithinTierLimits(
  userTier,  // BRONZE, SILVER, GOLD, PLATINUM
  fileSizeMb,
  dto.durationSec,
  envConfig
);

if (!fileConstraintCheck.allowed) {
  throw new HttpException(
    {
      success: false,
      message: 'File exceeds tier limits',
      errorCode: fileConstraintCheck.errorCode,
    },
    HttpStatus.FORBIDDEN
  );
}
```

**Tier Limits** (configured in `libs/domain`):
```typescript
const tierLimits = {
  BRONZE: { maxFileSizeMB: 200, maxDurationSec: 120 },
  SILVER: { maxFileSizeMB: 200, maxDurationSec: 120 },
  GOLD: { maxFileSizeMB: 200, maxDurationSec: 120 },
  PLATINUM: { maxFileSizeMB: 200, maxDurationSec: 120 },
};
```

---

## 🔄 Idempotency Implementation

### **Problem**: Network Retries → Duplicate Reels
**Example**:
1. Client calls `POST /upload-reel` → Network timeout
2. Client retries → Backend creates 2nd Media + Reel document
3. User now has duplicate reels (bad UX)

### **Solution**: Client-Generated Upload ID
```typescript
// Client generates UUID on first attempt
const uploadId = uuidv4();  // '550e8400-e29b-41d4-a716-446655440000'

// Include in all upload-reel requests
const payload = {
  uploadId,
  fileName: 'my-reel.mp4',
  ...
};
```

### **Backend Implementation**:
```typescript
async initReelUpload(userId: string, dto: UploadReelDto) {
  // Check if upload already initiated
  if (dto.uploadId) {
    const existingMedia = await this.mediaModel.findOne({
      userId,
      uploadId: dto.uploadId,
    });

    if (existingMedia) {
      const existingReel = await this.reelModel.findOne({ mediaId: String(existingMedia._id) });
      
      // Return existing upload with resume state
      return {
        mediaId: String(existingMedia._id),
        reelId: String(existingReel._id),
        uploadUrl: `https://stub-upload.chefooz.com/${existingMedia._id}`,
        expiresAt: new Date(Date.now() + 3600000).toISOString(),
        resumeState: {
          status: existingMedia.status,
          canResume: existingMedia.status === MediaStatus.UPLOADING,
          message: getResumeMessage(existingMedia.status),
        },
      };
    }
  }

  // Continue with normal upload creation...
}
```

**Resume State Logic**:
| Status | canResume | Message | Client Action |
|--------|-----------|---------|---------------|
| `UPLOADING` | ✅ true | "Upload in progress. Continue uploading." | Resume S3 upload |
| `PROCESSING` | ❌ false | "Video is being processed. Please wait." | Poll status endpoint |
| `READY` | ❌ false | "Upload already completed." | Navigate to reel view |
| `FAILED` | ❌ false | "Previous upload failed. Retry with new uploadId." | Show error, start fresh |

**MongoDB Index**: Ensures uniqueness
```typescript
MediaSchema.index({ userId: 1, uploadId: 1 }, { unique: true, sparse: true });
```

---

## 📊 Monitoring & Observability

### **Logging Strategy**
```typescript
// Structured logging with context
this.logger.log(`✅ Upload eligibility verified: userId=${userId}, tier=${userTier}, reelType=${reelType}`);
this.logger.warn(`⚠️ Quota exceeded: ${quotaCheck.reason}`);
this.logger.error(`❌ S3 upload failed: ${errorMessage}`);
```

**Key Events to Log**:
- ✅ Upload initialization (userId, mediaId, quotaCheck)
- ✅ Presigned URL generation (s3Key, expiresIn)
- ✅ Upload completion (mediaId, s3Key, processingMethod)
- ✅ Processing job enqueued (mediaId, jobId, features)
- ✅ Processing complete (mediaId, duration, variants)
- ❌ Quota violations (userId, quotaType, reason)
- ❌ S3 errors (operation, bucket, key, error)
- ❌ Processing failures (mediaId, error, retryCount)

---

### **Metrics to Track**
```typescript
// Upload metrics
uploadInitiations.inc({ quotaType: 'review' });
uploadCompletions.inc({ success: true });
uploadDuration.observe(Date.now() - uploadStartTime);

// Processing metrics
processingJobs.inc({ pipeline: 'ffmpeg' });
processingDuration.observe({ quality: '720p' }, processingTime);
processingFailures.inc({ error: 'FFMPEG_TIMEOUT' });

// Quota metrics
quotaUtilization.set({ type: 'promotional', role: 'user' }, usedSlots / maxSlots);
quotaViolations.inc({ type: 'menu_showcase' });
```

**Recommended Dashboards**:
1. **Upload Funnel**: Init → Presigned URL → Complete → Ready
2. **Processing Performance**: FFmpeg vs MediaConvert latency
3. **Quota Health**: % users near limits by type
4. **Error Rate**: Upload failures, processing failures, S3 errors

---

### **Alerting Rules**
```yaml
# Upload success rate < 95%
- alert: UploadSuccessRateLow
  expr: rate(uploadCompletions{success="true"}[5m]) / rate(uploadInitiations[5m]) < 0.95
  for: 10m
  annotations:
    summary: "Upload success rate dropped below 95%"

# Processing failures spike
- alert: ProcessingFailuresHigh
  expr: rate(processingFailures[5m]) > 10
  for: 5m
  annotations:
    summary: "More than 10 processing failures per minute"

# Quota violations surge (potential abuse)
- alert: QuotaViolationsSurge
  expr: rate(quotaViolations[1h]) > 100
  for: 10m
  annotations:
    summary: "Unusual spike in quota violations (possible abuse)"
```

---

## 🧪 Testing Strategy

### **Unit Tests**: `media.service.spec.ts`
```typescript
describe('MediaService', () => {
  let service: MediaService;
  let mediaModel: Model<MediaDocument>;
  let reelModel: Model<Reel>;
  let orderRepository: Repository<Order>;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        MediaService,
        { provide: getModelToken(Media.name), useValue: mockMediaModel },
        { provide: getModelToken(Reel.name), useValue: mockReelModel },
        { provide: getRepositoryToken(Order), useValue: mockOrderRepository },
        // ... other mocks
      ],
    }).compile();

    service = module.get<MediaService>(MediaService);
  });

  describe('initReelUpload', () => {
    it('should create media + reel documents for valid review reel', async () => {
      // Arrange
      const userId = 'user-123';
      const dto: UploadReelDto = {
        uploadId: 'upload-uuid',
        fileName: 'test.mp4',
        mimeType: 'video/mp4',
        durationSec: 30,
        sizeBytes: 10000000,
        caption: 'Test reel',
        orderId: 'order-123',
      };
      mockOrderRepository.count.mockResolvedValue(5);  // 5 delivered orders
      mockReelModel.countDocuments.mockResolvedValue(2);  // 2 review reels uploaded

      // Act
      const result = await service.initReelUpload(userId, dto);

      // Assert
      expect(result.mediaId).toBeDefined();
      expect(result.reelId).toBeDefined();
      expect(mockMediaModel.prototype.save).toHaveBeenCalled();
      expect(mockReelModel.create).toHaveBeenCalledWith(
        expect.objectContaining({
          userId,
          reelPurpose: 'USER_REVIEW',
          linkedOrderId: 'order-123',
        })
      );
    });

    it('should throw FORBIDDEN if review reel quota exceeded', async () => {
      const userId = 'user-123';
      const dto: UploadReelDto = { ...validDto, orderId: 'order-123' };
      mockOrderRepository.count.mockResolvedValue(5);  // 5 delivered orders
      mockReelModel.countDocuments.mockResolvedValue(5);  // 5 review reels (quota full)

      await expect(service.initReelUpload(userId, dto)).rejects.toThrow(HttpException);
    });

    it('should return existing upload if uploadId matches (idempotency)', async () => {
      const userId = 'user-123';
      const dto: UploadReelDto = { ...validDto, uploadId: 'existing-uuid' };
      const existingMedia = { _id: 'media-123', status: MediaStatus.UPLOADING };
      const existingReel = { _id: 'reel-123', mediaId: 'media-123' };
      mockMediaModel.findOne.mockResolvedValue(existingMedia);
      mockReelModel.findOne.mockResolvedValue(existingReel);

      const result = await service.initReelUpload(userId, dto);

      expect(result.mediaId).toBe('media-123');
      expect(result.reelId).toBe('reel-123');
      expect(result.resumeState).toEqual({
        status: 'uploading',
        canResume: true,
        message: expect.any(String),
      });
    });
  });

  describe('getPresignedUploadUrl', () => {
    it('should generate valid presigned S3 URL', async () => {
      const userId = 'user-123';
      const dto: GetPresignedUrlDto = { mediaId: 'media-123', fileType: 'video/mp4' };
      const media = { _id: 'media-123', userId, status: MediaStatus.UPLOADING };
      mockMediaModel.findById.mockResolvedValue(media);

      const result = await service.getPresignedUploadUrl(userId, dto);

      expect(result.uploadUrl).toContain('https://');
      expect(result.uploadUrl).toContain('chefooz-media-uat');
      expect(result.s3Key).toBe('uploads/media-123/original.mp4');
      expect(result.expiresIn).toBe(900);
    });

    it('should throw FORBIDDEN if media does not belong to user', async () => {
      const userId = 'user-123';
      const dto: GetPresignedUrlDto = { mediaId: 'media-123', fileType: 'video/mp4' };
      const media = { _id: 'media-123', userId: 'other-user', status: MediaStatus.UPLOADING };
      mockMediaModel.findById.mockResolvedValue(media);

      await expect(service.getPresignedUploadUrl(userId, dto)).rejects.toThrow(ForbiddenException);
    });

    it('should throw BAD_REQUEST if media status is not UPLOADING', async () => {
      const userId = 'user-123';
      const dto: GetPresignedUrlDto = { mediaId: 'media-123', fileType: 'video/mp4' };
      const media = { _id: 'media-123', userId, status: MediaStatus.READY };
      mockMediaModel.findById.mockResolvedValue(media);

      await expect(service.getPresignedUploadUrl(userId, dto)).rejects.toThrow(BadRequestException);
    });
  });

  describe('completeUpload', () => {
    it('should verify S3 file, copy to input bucket, and enqueue processing', async () => {
      const userId = 'user-123';
      const dto: CompleteUploadDto = { mediaId: 'media-123', s3Key: 'uploads/media-123/original.mp4' };
      const media = { _id: 'media-123', userId, status: MediaStatus.UPLOADING };
      mockMediaModel.findById.mockResolvedValue(media);
      mockS3Client.send.mockResolvedValue({});  // HeadObjectCommand + CopyObjectCommand succeed

      const result = await service.completeUpload(userId, dto);

      expect(result.success).toBe(true);
      expect(mockMediaModel.findByIdAndUpdate).toHaveBeenCalledWith(
        'media-123',
        expect.objectContaining({ status: MediaStatus.PROCESSING })
      );
      expect(mockVideoProcessingQueue.add).toHaveBeenCalledWith(
        'process-video',
        expect.objectContaining({ mediaId: 'media-123' }),
        expect.any(Object)
      );
    });

    it('should throw BAD_REQUEST if S3 file does not exist', async () => {
      const userId = 'user-123';
      const dto: CompleteUploadDto = { mediaId: 'media-123', s3Key: 'uploads/media-123/original.mp4' };
      const media = { _id: 'media-123', userId, status: MediaStatus.UPLOADING };
      mockMediaModel.findById.mockResolvedValue(media);
      mockS3Client.send.mockRejectedValue(new Error('NoSuchKey'));

      await expect(service.completeUpload(userId, dto)).rejects.toThrow(BadRequestException);
    });
  });
});
```

---

### **Integration Tests**: `media.e2e-spec.ts`
```typescript
describe('Media API (e2e)', () => {
  let app: INestApplication;
  let authToken: string;

  beforeAll(async () => {
    const moduleFixture = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = moduleFixture.createNestApplication();
    await app.init();

    // Login to get auth token
    const loginResponse = await request(app.getHttpServer())
      .post('/api/v1/auth/login')
      .send({ phone: '+919876543210', otp: '123456' });
    authToken = loginResponse.body.data.accessToken;
  });

  it('POST /api/v1/media/upload-reel should initialize reel upload', async () => {
    const payload = {
      uploadId: uuidv4(),
      fileName: 'test-reel.mp4',
      mimeType: 'video/mp4',
      durationSec: 30,
      sizeBytes: 10000000,
      caption: 'Test reel from E2E test',
      hashtags: ['test', 'automation'],
    };

    const response = await request(app.getHttpServer())
      .post('/api/v1/media/upload-reel')
      .set('Authorization', `Bearer ${authToken}`)
      .send(payload)
      .expect(201);

    expect(response.body.success).toBe(true);
    expect(response.body.data.mediaId).toBeDefined();
    expect(response.body.data.reelId).toBeDefined();
    expect(response.body.data.uploadUrl).toContain('https://');
  });

  it('POST /api/v1/media/presigned-upload-url should generate S3 URL', async () => {
    // First initialize upload
    const initPayload = { ...validUploadReelDto };
    const initResponse = await request(app.getHttpServer())
      .post('/api/v1/media/upload-reel')
      .set('Authorization', `Bearer ${authToken}`)
      .send(initPayload)
      .expect(201);

    const mediaId = initResponse.body.data.mediaId;

    // Then get presigned URL
    const presignedPayload = { mediaId, fileType: 'video/mp4' };
    const presignedResponse = await request(app.getHttpServer())
      .post('/api/v1/media/presigned-upload-url')
      .set('Authorization', `Bearer ${authToken}`)
      .send(presignedPayload)
      .expect(200);

    expect(presignedResponse.body.data.uploadUrl).toContain('amazonaws.com');
    expect(presignedResponse.body.data.s3Key).toContain(mediaId);
    expect(presignedResponse.body.data.expiresIn).toBe(900);
  });

  it('Full upload flow: init → presigned → complete → status poll', async () => {
    // 1. Initialize upload
    const initPayload = { ...validUploadReelDto };
    const initResponse = await request(app.getHttpServer())
      .post('/api/v1/media/upload-reel')
      .set('Authorization', `Bearer ${authToken}`)
      .send(initPayload)
      .expect(201);
    const mediaId = initResponse.body.data.mediaId;

    // 2. Get presigned URL
    const presignedPayload = { mediaId, fileType: 'video/mp4' };
    const presignedResponse = await request(app.getHttpServer())
      .post('/api/v1/media/presigned-upload-url')
      .set('Authorization', `Bearer ${authToken}`)
      .send(presignedPayload)
      .expect(200);
    const s3Key = presignedResponse.body.data.s3Key;

    // 3. Simulate S3 upload (mock in test environment)
    // (In real E2E, would actually upload to S3)

    // 4. Complete upload
    const completePayload = { mediaId, s3Key };
    await request(app.getHttpServer())
      .post('/api/v1/media/complete-upload')
      .set('Authorization', `Bearer ${authToken}`)
      .send(completePayload)
      .expect(200);

    // 5. Poll status until READY
    let status = 'processing';
    let attempts = 0;
    while (status === 'processing' && attempts < 30) {
      const statusResponse = await request(app.getHttpServer())
        .get(`/api/v1/media/${mediaId}/status`)
        .set('Authorization', `Bearer ${authToken}`)
        .expect(200);
      status = statusResponse.body.data.status;
      if (status === 'processing') {
        await new Promise(resolve => setTimeout(resolve, 2000));  // Wait 2s
        attempts++;
      }
    }

    expect(status).toBe('ready');
  });
});
```

---

## 🚀 Deployment Configuration

### **Environment Variables**
```bash
# AWS S3 Configuration
AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
AWS_REGION=ap-south-1
S3_BUCKET_NAME=chefooz-media-uat        # Upload bucket
S3_INPUT_BUCKET=chefooz-media-input     # MediaConvert input
S3_OUTPUT_BUCKET=chefooz-media-output   # Processed output
CDN_BASE_URL=https://cdn.chefooz.com

# AWS MediaConvert (Optional - Fallback Pipeline)
AWS_MEDIACONVERT_ENDPOINT=https://abc123def.mediaconvert.ap-south-1.amazonaws.com
AWS_MEDIACONVERT_ROLE_ARN=arn:aws:iam::123456789012:role/MediaConvertRole

# Redis (Bull Queue for FFmpeg Jobs)
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=secret

# MongoDB (Media & Reel Documents)
MONGODB_URI=mongodb://localhost:27017/chefooz

# PostgreSQL (Orders, Users, MenuItems)
DB_HOST=localhost
DB_PORT=5432
DB_USERNAME=postgres
DB_PASSWORD=postgres
DB_DATABASE=chefooz
```

---

### **S3 Bucket Lifecycle Policies**
```json
{
  "Rules": [
    {
      "Id": "CleanupUploadBucket",
      "Status": "Enabled",
      "Filter": { "Prefix": "uploads/" },
      "Expiration": { "Days": 7 },
      "NoncurrentVersionExpiration": { "NoncurrentDays": 1 }
    },
    {
      "Id": "CleanupInputBucket",
      "Status": "Enabled",
      "Filter": { "Prefix": "uploads/" },
      "Expiration": { "Days": 7 }
    },
    {
      "Id": "ArchiveOutputBucket",
      "Status": "Enabled",
      "Filter": { "Prefix": "converted/" },
      "Transitions": [
        {
          "Days": 90,
          "StorageClass": "GLACIER"
        }
      ]
    }
  ]
}
```

---

### **Bull Queue Configuration**
```typescript
// apps/chefooz-apis/src/app.module.ts
BullModule.forRoot({
  redis: {
    host: process.env.REDIS_HOST || 'localhost',
    port: parseInt(process.env.REDIS_PORT || '6379'),
    password: process.env.REDIS_PASSWORD,
    maxRetriesPerRequest: null,  // Required for Bull
  },
  defaultJobOptions: {
    removeOnComplete: 100,  // Keep last 100 completed jobs
    removeOnFail: 500,      // Keep last 500 failed jobs
    attempts: 3,
    backoff: {
      type: 'exponential',
      delay: 5000,
    },
  },
}),
```

---

## 📚 Related Technical Documentation

- **[Media Processing Module](../media-processing/TECHNICAL_GUIDE.md)** - FFmpeg worker implementation
- **[Reels Schema](../../database/schemas/reel.schema.ts)** - MongoDB reel document structure
- **[AWS MediaConvert Integration](../../integrations/media-convert/mediaconvert.service.ts)** - MediaConvert API wrapper
- **[Domain Policy Library](../../../libs/domain/src/upload-policy/)** - Quota calculation functions

---

## March 2026 — Multi-Image Carousel POST Upload

> **Last Updated**: March 2026

### Problem
When a user selected multiple images for a POST, only the first image (`media.uri`) was uploaded.
Extra images in `images[1..N]` were silently ignored. The backend always queued FFmpeg for every
upload, causing POST photos to enter a permanent "processing" state and never appear in the feed.
Additionally, the feed query filtered on `videoUrl` existence, hiding all POST content.

### Solution: End-to-End Multi-Image Pipeline

#### Upload Flow (POST type)

```
share.tsx
  1. Destructures images[] from upload-v2.store
  2. Calls uploadReel(request, images[0].uri, mimeType, ..., extraImages=images[1..N])

useUploadReel.ts
  Step 1: Init (creates Media + Reel docs with imageUrls=[])
  Step 2: Get presigned URL for primary image (images[0]) → uploads/{mediaId}/original.jpeg
  Step 3: Upload primary image to S3
  Step 3.5: For each extra image (index 1..N):
    - Get presigned URL with imageIndex param → uploads/{mediaId}/image_{i}.jpeg
    - Upload to S3
    - Collect S3 key → extraImageS3Keys[]
  Step 4: completeS3Upload(mediaId, s3Key, ..., extraImageS3Keys)

Backend completeUpload (media.service.ts)
  - Verifies primary file in upload bucket
  - Copies primary to input bucket (for audit/storage)
  - Detects contentType === 'POST' → skips FFmpeg entirely
  - For each image in [s3Key, ...extraImageS3Keys]:
      copies from upload bucket to output bucket as photos/{mediaId}/image_{i}.ext
      builds HTTPS URL from output bucket
  - Updates reel.imageUrls = [url_0, ..., url_n]
  - Updates reel.thumbnailUrl = url_0
  - Marks media.status = 'ready' immediately
  - Sends reel.ready notification

feed.service.ts
  - Feed query now allows: (videoUrl exists) OR (contentType=POST AND thumbnailUrl exists)
  - mapReelToFeedItem now includes imageUrls[] in response
```

#### Key Files Changed

| File | Change |
|---|---|
| `reel.schema.ts` | Added `imageUrls: string[]` Mongoose prop |
| `libs/types/src/lib/reel.types.ts` | Added `imageUrls?: string[]` to `Reel` interface |
| `dto/presigned-upload.dto.ts` | Relaxed `fileType` to include image MIME types; added `imageIndex?`; added `extraImageS3Keys?` to `CompleteUploadDto` |
| `media.service.ts` (backend) | `getPresignedUploadUrl` supports `imageIndex`; `completeUpload` has POST fast-path that skips FFmpeg |
| `feed.service.ts` | Fixed feed filter to include POST content; maps `imageUrls[]` in response |
| `media.service.ts` (frontend) | Added `imageIndex?` and `extraImageS3Keys?` params |
| `useUploadReel.ts` | Added `extraImages?` param; step 3.5 uploads images 1..N |
| `share.tsx` | Passes `images.slice(1)` as `extraImages` |
| `FeedPostCard.tsx` | Horizontal paginating `FlatList` carousel when `imageUrls.length > 1`, with dot indicators |

#### S3 Key Convention

| Purpose | Upload bucket key | Output bucket key |
|---|---|---|
| Primary image (image[0]) | `uploads/{mediaId}/original.{ext}` | `photos/{mediaId}/image_0.{ext}` |
| Extra image (image[i]) | `uploads/{mediaId}/image_{i}.{ext}` | `photos/{mediaId}/image_{i}.{ext}` |
| Thumbnail (REEL) | `uploads/{mediaId}/thumbnail.jpg` | `converted/{mediaId}/thumbnail.jpg` |

#### MIME Type Validation (updated)

`GetPresignedUrlDto.fileType` now accepts:
- `video/mp4`, `video/quicktime`, `video/x-msvideo` (REELs)
- `image/jpeg`, `image/jpg`, `image/png`, `image/heic`, `image/heif` (POSTs)

#### Constraints / Edge Cases

- If an extra image copy fails it is **skipped** (warning logged) — the POST still completes with fewer images.
- POST uploads never enter FFmpeg queue. `media.status` transitions: `UPLOADING → PROCESSING → READY` (all within `completeUpload`).
- Feed filter uses `$and` to add the POST visibility condition, avoiding conflict with the existing moderation `$or`.
- `imageUrls` on the Reel document is empty `[]` until `completeUpload` — never query on it during upload.

---

**[SLICE_COMPLETE ✅]**  
*Media Module Technical Guide - Comprehensive developer documentation complete*

---

## Android Build Fix — VisionCamera v5 + Kotlin 2.0 + CameraX Compatibility

> **Date**: April 24, 2026  
> **Type**: Build infrastructure fix (no user-facing changes)

### Root Causes

The Android build was failing with two separate issues after migrating to `react-native-vision-camera@5`:

#### Issue 1: CameraX alpha dependency requires AGP 8.9.1 + compileSdk 36

VisionCamera v5 hardcodes `androidx.camera:*:1.7.0-alpha01` in its `android/build.gradle`. This version:
- Requires `compileSdk >= 36`
- Requires Android Gradle Plugin `>= 8.9.1`

The app was using `compileSdk 35` and AGP `8.8.2` (via React Native's `libs.versions.toml`).

#### Issue 2: Kotlin 2.0 compiler rejects `protected const val` in companion objects

VisionCamera's Nitrogen-generated `*Spec.kt` files contain companion objects with `protected const val TAG = "..."`. Kotlin 2.0 (in use via React Native 0.79+'s toolchain) treats accessing these from subclasses without `@JvmStatic` as a compilation error. `@JvmStatic` cannot be applied to `const val`, so the fix is to remove the `protected` modifier.

### Fixes Applied

#### `apps/chefooz-app/android/build.gradle`

1. **AGP version**: Pinned `classpath('com.android.tools.build:gradle:8.9.1')` explicitly, overriding the RN version catalog's `8.8.2`
2. **compileSdk**: Added `ext { compileSdkVersion = 36; targetSdkVersion = 35; buildToolsVersion = "35.0.0" }` before `apply plugin: "expo-root-project"` — Expo's `ExpoRootProjectPlugin` uses `extra.setIfNotExist(...)` so this override is respected

#### `scripts/postinstall.js` 

Added a post-install step that removes `protected` from `const val TAG` in all `nitrogen/generated/android/kotlin/**/*Spec.kt` files. This runs on every `npm install` to survive VisionCamera upgrades.

#### `react-native-vision-camera` version

Upgraded from `5.0.4` → `5.0.6` which fixes additional Kotlin 2.0 compatibility issues (nullable invocations, generated view manager type incompatibilities).

### Configuration Summary

| Setting | Before | After |
|---|---|---|
| Android Gradle Plugin | `8.8.2` (from RN libs.versions.toml) | `8.9.1` (pinned in build.gradle) |
| `compileSdk` | `35` (Expo default) | `36` (ext override) |
| `targetSdk` | `35` | `35` (unchanged) |
| `buildTools` | `35.0.0` | `35.0.0` (unchanged) |
| VisionCamera version | `5.0.4` | `5.0.6` |
| Gradle wrapper | `8.13` | `8.13` (unchanged — already compatible with AGP 8.9.1) |

### Constraints / Edge Cases

- The `ext { compileSdkVersion = 36 }` block must be placed **before** `apply plugin: "expo-root-project"` to take effect
- `targetSdk` is kept at 35 intentionally — bumping `targetSdk` to 36 requires testing all Android behavior changes for API 36
- The Kotlin companion `protected const val TAG` fix uses a regex that matches exactly 4 spaces indent — if VisionCamera changes its code style, update the regex in `postinstall.js`
- Gradle 8.13 is already compatible with AGP 8.9.1 (requires Gradle 8.11.1+)

---

## Android Runtime Fix — Media3 Version Conflicts (Video Recording Crashes, April 2026)

Three cascading crashes were resolved via an **explicit pin-list strategy** for Media3 artifacts.

### Crash 1 — Upload FAB: AbstractMethodError on video playback

**Symptom**:
```
FATAL EXCEPTION: ExoPlayer:Playback
java.lang.AbstractMethodError: abstract method
  "Allocator LoadControl.getAllocator(PlayerId)"
  at ExoPlayerImplInternal.createMediaPeriodHolder
```

**Root cause**: CameraX `1.7.0-alpha01` (via VisionCamera v5) transitively depends on `media3-muxer:1.9.0`. Media3's BOM constraints promoted all `androidx.media3:*` to `1.9.0`. `expo-video` v3.0.14 compiled its `LoadControl` implementation against `1.8.0`. Media3 `1.9.0` added `LoadControl.getAllocator(PlayerId)` as a new abstract method — expo-video's compiled binary doesn't implement it → `AbstractMethodError`.

### Crash 2 — Video recording: NoClassDefFoundError for MediaMuxerCompat

**Symptom** (after blanket-pinning all media3 to 1.8.0):
```
FATAL EXCEPTION: CameraX-camerax_io_1
java.lang.NoClassDefFoundError: Failed resolution of: Landroidx/media3/muxer/MediaMuxerCompat;
  at androidx.camera.video.internal.muxer.Media3MuxerImpl.setOutput
```

**Root cause**: `MediaMuxerCompat` was introduced in `media3-muxer:1.9.0`. Blanket-pinning all `media3-*` to `1.8.0` removes it from the APK's DEX.

### Crash 3 — Video recording: NoSuchMethodError in MediaFormatUtil

**Symptom** (after exempting only media3-muxer and media3-container):
```
FATAL EXCEPTION: CameraX-camerax_io_1
java.lang.NoSuchMethodError: No static method getFloatFromIntOrFloat(...)
  in Landroidx/media3/common/util/MediaFormatUtil;
  at MediaMuxerCompat.addTrack (media3-muxer:1.9.0)
```

**Root cause**: `media3-muxer:1.9.0` was compiled against `media3-common:1.9.0` and calls `MediaFormatUtil.getFloatFromIntOrFloat()` — a static method added in `media3-common:1.9.0`. Keeping `media3-common` pinned at `1.8.0` makes this call fail. Each new version of `media3-muxer` may call new APIs on `media3-common`, making the exclusion-list approach inherently fragile.

### Final Fix — Explicit pin list (not exclusion list)

Rather than naming what to *exclude* from the pin (fragile — any new `media3-muxer` API call on a newer common artifact will break), we **only pin the ExoPlayer playback artifacts** that expo-video compiled against `1.8.0`. Everything else floats to whatever version Gradle resolves (1.9.0 from CameraX).

**Why this is safe**: `media3-exoplayer:1.8.0` was compiled against `media3-common:1.8.0`. At runtime with `media3-common:1.9.0`, it only calls APIs it knows about (1.8.0 era), which are all still present (backward-compatible additions only). The `AbstractMethodError` is fixed because *ExoPlayer itself* (the caller of `LoadControl.getAllocator`) stays at `1.8.0` where that method doesn't exist in the interface.

```groovy
// apps/chefooz-app/android/build.gradle
allprojects {
  configurations.all {
    resolutionStrategy.eachDependency { DependencyResolveDetails details ->
      if (details.requested.group == 'androidx.media3') {
        // Only pin the ExoPlayer playback artifacts that expo-video compiled against 1.8.0.
        // media3-common, media3-muxer, media3-container, media3-extractor etc. float to 1.9.0.
        def pinTo180 = [
          'media3-exoplayer',
          'media3-exoplayer-dash',
          'media3-exoplayer-hls',
          'media3-session',
          'media3-ui',
          'media3-datasource-okhttp',
        ]
        if (pinTo180.contains(details.requested.name)) {
          details.useVersion '1.8.0'
          details.because 'expo-video compiled against 1.8.0; media3 1.9.0 adds abstract LoadControl.getAllocator(PlayerId) breaking expo-video binary'
        }
      }
    }
  }
}
```

**Verified dependency resolutions**:
| Artifact | Resolved | Reason |
|---|---|---|
| `media3-exoplayer` | **`1.8.0`** (pinned) | expo-video LoadControl compiled against 1.8.0 |
| `media3-exoplayer-dash` | **`1.8.0`** (pinned) | expo-video dep at 1.8.0 |
| `media3-exoplayer-hls` | **`1.8.0`** (pinned) | expo-video dep at 1.8.0 |
| `media3-session` | **`1.8.0`** (pinned) | expo-video dep at 1.8.0 |
| `media3-ui` | **`1.8.0`** (pinned) | expo-video dep at 1.8.0 |
| `media3-datasource-okhttp` | **`1.8.0`** (pinned) | expo-video dep at 1.8.0 |
| `media3-muxer` | **`1.9.0`** (floats) | CameraX needs MediaMuxerCompat (1.9.0+) |
| `media3-container` | **`1.9.0`** (floats) | muxer transitive dep |
| `media3-common` | **`1.9.0`** (floats) | muxer 1.9.0 calls getFloatFromIntOrFloat (1.9.0+) |
| `media3-extractor` | **`1.9.0`** (floats) | CameraX transitive dep |

**Note**: If `expo-video` is updated to a version compiled against Media3 `1.9.0`+, the entire `pinTo180` list can be removed.

---

## iOS Runtime Fix — VisionCamera v5 `invalidAVFileType` on Recording (April 2026)

**Symptom**: On iOS, attempting to record video fails immediately:
```
ERROR  VisionCamera createRecorder failed: [Error: invalidAVFileType]
```

**Root cause**: `URL+createTempURL.swift` in VisionCamera v5.0.6 has a type mismatch bug:

```swift
// BUGGY in VisionCamera v5.0.6:
let mimeType = fileType.rawValue  // "com.apple.quicktime-movie" — a UTI, NOT a MIME type
guard let utFileType = UTType(mimeType: mimeType) else {
  throw TemporaryFileError.invalidAVFileType  // throws every time
}
```

`AVFileType.rawValue` is a UTI identifier string:
- `.mov` → `"com.apple.quicktime-movie"` (UTI)
- `.mp4` → `"public.mpeg-4"` (UTI)

`UTType(mimeType:)` is a **failable** initializer that only accepts true MIME-type strings (e.g. `"video/quicktime"`, `"video/mp4"`). UTI strings always return `nil` → the error is thrown before any recording begins. The default `fileType` on iOS is `.mov`, so every recording fails.

**Why `fileType: 'mp4'` does NOT help**: `AVFileType.mp4.rawValue = "public.mpeg-4"` (also a UTI) → same failure.

**Fix**: Use `UTType(_ identifier: String)` — the UTI identifier initializer. Note: this initializer is also failable (`UTType?`), so `guard let` is required:

```swift
// FIXED:
guard let utFileType = UTType(fileType.rawValue) else {
  throw TemporaryFileError.invalidAVFileType
}
return try createTempURL(fileType: utFileType)
// "com.apple.quicktime-movie" → UTType.quickTimeMovie → extension "mov"
// "public.mpeg-4"             → UTType.mpeg4Movie     → extension "mp4"
```

**Where applied**:
1. `node_modules/react-native-vision-camera/ios/Extensions/URL+createTempURL.swift` — direct source patch
2. `scripts/postinstall.js` — string-replace block re-applies the fix on every `yarn install`

After applying: `cd ios && pod install` is required for CocoaPods to pick up the patched source.

**Upgrade note**: Check if this bug is still present when upgrading VisionCamera beyond v5.0.6. If the upstream fix is merged, remove the postinstall patch block.
