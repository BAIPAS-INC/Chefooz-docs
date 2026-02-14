# Media-Processing Module - Technical Guide

**Version:** 1.0  
**Last Updated:** February 14, 2026  
**Status:** Production  
**Target Audience:** Backend Engineers, DevOps Engineers

---

## 📋 Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Module Structure](#module-structure)
3. [Core Services](#core-services)
4. [Bull Queue Integration](#bull-queue-integration)
5. [FFmpeg Processing](#ffmpeg-processing)
6. [S3 Integration](#s3-integration)
7. [Database Schema](#database-schema)
8. [API Reference](#api-reference)
9. [Error Handling](#error-handling)
10. [Testing Strategy](#testing-strategy)
11. [Deployment & Configuration](#deployment--configuration)
12. [Monitoring & Observability](#monitoring--observability)
13. [Performance Optimization](#performance-optimization)
14. [Security Considerations](#security-considerations)
15. [Troubleshooting Guide](#troubleshooting-guide)

---

## 1. Architecture Overview

### 1.1 High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                         CHEFOOZ BACKEND                               │
├──────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  ┌────────────────┐         ┌──────────────────┐                    │
│  │  Media Module  │────────▶│  Bull Queue      │                    │
│  │                │  Enqueue│  (Redis)         │                    │
│  │  POST /upload  │         │                  │                    │
│  └────────────────┘         └────────┬─────────┘                    │
│                                       │                               │
│                              ┌────────▼──────────────────┐           │
│                              │ VideoProcessingProcessor  │           │
│                              │  (@Processor decorator)   │           │
│                              └────────┬──────────────────┘           │
│                                       │                               │
│                       ┌───────────────┼────────────────┐             │
│                       │               │                │             │
│              ┌────────▼─────┐ ┌──────▼──────┐ ┌───────▼──────┐     │
│              │ FFmpegUtils  │ │MediaProcess │ │VideoFilters  │     │
│              │   Service    │ │  Service    │ │   Service    │     │
│              └──────────────┘ └─────────────┘ └──────────────┘     │
│                       │               │                │             │
│                       └───────────────┼────────────────┘             │
│                                       │                               │
│                              ┌────────▼─────────┐                    │
│                              │   AWS S3         │                    │
│                              │  Input/Output    │                    │
│                              └──────────────────┘                    │
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘
```

### 1.2 Technology Stack

| Component | Technology | Version | Purpose |
|-----------|-----------|---------|---------|
| **Runtime** | Node.js | 20.x | JavaScript runtime |
| **Framework** | NestJS | 10.x | API framework |
| **Queue** | Bull | 4.x | Job queue management |
| **Cache** | Redis | 7.x | Queue backend |
| **Video Processing** | FFmpeg | 6.x | Video encoding/filtering |
| **FFmpeg Wrapper** | fluent-ffmpeg | 2.x | Node.js FFmpeg API |
| **Storage** | AWS S3 | SDK v3 | Media file storage |
| **Database (Docs)** | MongoDB | 6.x | Media/Reel documents |
| **Database (Audit)** | PostgreSQL | 15.x | Audit logs |

### 1.3 Processing Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│ MEDIA-PROCESSING MODULE FLOW                                         │
└─────────────────────────────────────────────────────────────────────┘

    [1] Job Received (Bull Queue)
           │
           ↓
    [2] VideoProcessingProcessor.handleVideoProcessing()
           │
           ├─→ Download from S3 (input bucket)
           │
           ├─→ FFmpegUtilsService.detectVideoMetadata()
           │    └─→ Aspect ratio, codec, duration, rotation
           │
           ├─→ MediaProcessingService.processVideo()
           │    │
           │    ├─→ [Optional] Convert image to video (if isImage)
           │    ├─→ [Optional] Trim video (if trim metadata)
           │    ├─→ [Optional] VideoFiltersService.applyMultipleEffects()
           │    │    ├─→ Image filters (brightness, contrast, etc.)
           │    │    └─→ Text overlays (drawtext filter)
           │    │
           │    ├─→ Generate 3 quality variants (720p, 480p, 360p)
           │    │    └─→ Parallel encoding (Promise.all)
           │    │
           │    └─→ FFmpegUtilsService.generateCover()
           │         └─→ Extract JPEG at user-selected timestamp
           │
           ├─→ Upload to S3 (output bucket)
           │    ├─→ master.mp4
           │    ├─→ variant_720p.mp4
           │    ├─→ variant_480p.mp4
           │    ├─→ variant_360p.mp4
           │    └─→ thumbnail.jpg
           │
           ├─→ Update MongoDB (Media & Reel documents)
           │
           ├─→ [Optional] Trigger ModerationService
           │
           ├─→ [Optional] Invalidate feed cache (Redis pub/sub)
           │
           └─→ Cleanup temp files (/tmp/video-processing-<mediaId>)
```

---

## 2. Module Structure

### 2.1 File Organization

```
apps/chefooz-apis/src/modules/media-processing/
├── video-processing.processor.ts        # Bull queue worker (job handler)
├── media-processing.service.ts          # Core processing logic
├── ffmpeg-utils.service.ts              # FFmpeg utilities (metadata, cover)
├── video-filters.service.ts             # Text overlays & filters
├── media-processing.module.ts           # Module definition
└── media-processing-test.controller.ts  # Test endpoints (dev only)
```

### 2.2 Dependency Graph

```
MediaProcessingModule
  │
  ├─→ VideoProcessingProcessor (Bull Worker)
  │    ├─→ MediaProcessingService
  │    │    ├─→ FFmpegUtilsService
  │    │    └─→ VideoFiltersService
  │    │
  │    ├─→ MongooseModels (Media, Reel, MediaConvertJob)
  │    └─→ S3Client (AWS SDK)
  │
  └─→ MediaProcessingTestController (Optional)
       ├─→ FFmpegUtilsService
       └─→ VideoFiltersService
```

### 2.3 Module Configuration

**`media-processing.module.ts`**:
```typescript
@Module({
  imports: [
    MongooseModule.forFeature([
      { name: Media.name, schema: MediaSchema },
      { name: Reel.name, schema: ReelSchema },
      { name: MediaConvertJob.name, schema: MediaConvertJobSchema },
    ]),
    BullModule.registerQueue({
      name: 'video-processing',
      defaultJobOptions: {
        attempts: 3,
        backoff: { type: 'exponential', delay: 5000 },
        removeOnComplete: 100,
        removeOnFail: 500,
      },
    }),
  ],
  controllers: [MediaProcessingTestController],
  providers: [
    FFmpegUtilsService,
    VideoFiltersService,
    MediaProcessingService,
    VideoProcessingProcessor,
  ],
  exports: [MediaProcessingService, BullModule],
})
export class MediaProcessingModule {}
```

**Key Configuration**:
- **Retry Policy**: 3 attempts with exponential backoff (5s, 10s, 20s)
- **Job Retention**: Last 100 completed jobs (debugging), last 500 failed jobs (analysis)
- **Exports**: Service and queue available to other modules

---

## 3. Core Services

### 3.1 VideoProcessingProcessor

**Purpose**: Bull queue worker that orchestrates the entire processing pipeline.

**Key Methods**:

#### `handleVideoProcessing(job: Job<VideoProcessingJobData>)`
Main job handler decorated with `@Process('process-video')`.

**Input** (Job Data):
```typescript
interface VideoProcessingJobData {
  mediaId: string;              // Media document ID
  s3Key: string;                // S3 key (input bucket)
  trim?: { startSec: number; endSec: number };
  coverTimestampSec?: number;   // User-selected cover frame time
  textOverlays?: ReelOverlay[]; // Text overlays to burn
  musicOverlay?: MusicOverlay;  // Music track (planned)
  filter?: VideoFilter;         // Image filter to apply
  photoDurationSec?: number;    // Image-to-video duration
  userId: string;
}
```

**Processing Steps**:
```typescript
async handleVideoProcessing(job: Job<VideoProcessingJobData>): Promise<any> {
  const { mediaId, s3Key, trim, coverTimestampSec, textOverlays, filter, userId } = job.data;
  const workDir = path.join('/tmp', `video-processing-${mediaId}`);

  try {
    // 1. Create work directory
    await fs.mkdir(workDir, { recursive: true });

    // 2. Download from S3
    await this.downloadFromS3(s3Key, path.join(workDir, 'original.mp4'));
    await job.progress(20);

    // 3. Process video (metadata → trim → effects → variants → cover)
    const processed = await this.mediaProcessingService.processVideo(
      path.join(workDir, 'original.mp4'),
      { trim, generateVariants: true, generateThumbnail: true, coverTimestampSec, textOverlays, filter }
    );
    await job.progress(70);

    // 4. Upload to S3 (master + variants + thumbnail)
    const masterS3Key = s3Key.replace('original.mp4', 'master.mp4');
    await this.uploadToS3(processed.masterPath, masterS3Key);
    // ... upload variants and thumbnail

    await job.progress(90);

    // 5. Update database (Media & Reel)
    await this.mediaModel.findByIdAndUpdate(mediaId, {
      status: MediaStatus.READY,
      masterVideoUrl: s3UriToHttps(`s3://${this.outputBucket}/${masterS3Key}`, cdnUrl),
      variants: variantKeys,
      thumbnailUrl,
      processedAt: new Date(),
      processingMethod: 'ffmpeg',
    });

    await this.reelModel.findOneAndUpdate(
      { mediaId },
      { videoUrl: variantKeys[0].url, thumbnailUrl, durationSec: processed.metadata.duration }
    );

    // 6. Trigger moderation (async, don't block)
    await this.moduleRef.resolve('ModerationService').startModeration(mediaId, userId, thumbnailKey);

    // 7. Invalidate feed cache
    await this.moduleRef.resolve('CacheService').publish('invalidate:feed', { mediaId, userId });

    // 8. Cleanup
    await this.mediaProcessingService.cleanupWorkDir(workDir);

    await job.progress(100);
    return { success: true, mediaId, processingTime: Date.now() - job.timestamp };

  } catch (error) {
    // Fallback to MediaConvert (if available)
    await this.triggerMediaConvertFallback(mediaId, s3Key, error);
    throw error;
  }
}
```

**S3 Helper Methods**:
```typescript
private async downloadFromS3(s3Key: string, localPath: string): Promise<void> {
  const command = new GetObjectCommand({ Bucket: this.inputBucket, Key: s3Key });
  const response = await this.s3Client.send(command);
  const readableStream = response.Body as Readable;
  const writeStream = fs.createWriteStream(localPath);
  await new Promise((resolve, reject) => {
    readableStream.pipe(writeStream);
    writeStream.on('finish', resolve);
    writeStream.on('error', reject);
  });
}

private async uploadToS3(localPath: string, s3Key: string): Promise<void> {
  const fileContent = await fs.readFile(localPath);
  const command = new PutObjectCommand({
    Bucket: this.outputBucket,
    Key: s3Key,
    Body: fileContent,
    ContentType: s3Key.endsWith('.jpg') ? 'image/jpeg' : 'video/mp4',
  });
  await this.s3Client.send(command);
}
```

### 3.2 MediaProcessingService

**Purpose**: Core FFmpeg processing logic (trim, effects, variants, cover).

**Key Methods**:

#### `processVideo(inputPath, options)`
Main processing pipeline orchestrator.

```typescript
async processVideo(
  inputPath: string,
  options: {
    trim?: { startSec: number; endSec: number };
    generateVariants?: boolean;
    generateThumbnail?: boolean;
    coverTimestampSec?: number;
    textOverlays?: ReelOverlay[];
    musicOverlay?: MusicOverlay;
    filter?: VideoFilter;
    photoDurationSec?: number;
  }
): Promise<{
  masterPath: string;
  variants: { resolution: string; path: string }[];
  thumbnail?: string;
  metadata: VideoMetadata;
}> {
  const workDir = path.dirname(inputPath);
  let currentFile = inputPath;

  // STEP 0: Detect metadata
  const metadata = await this.ffmpegUtils.detectVideoMetadata(inputPath);
  const isImage = metadata.duration < 1.0 || ['mjpeg', 'png', 'bmp'].includes(metadata.codec);

  // STEP 0.5: Convert image to video (if needed)
  if (isImage) {
    const videoPath = path.join(workDir, 'image_as_video.mp4');
    await this.ffmpegUtils.convertImageToVideo(currentFile, videoPath, options.photoDurationSec || 5);
    currentFile = videoPath;
  }

  // STEP 1: Trim (if specified)
  if (options.trim) {
    const trimmedPath = path.join(workDir, 'trimmed.mp4');
    await this.trimVideo(currentFile, trimmedPath, options.trim);
    currentFile = trimmedPath;
  }

  // STEP 1.5: Apply effects (text overlays + filters)
  if (options.textOverlays || options.filter) {
    const effectsPath = path.join(workDir, 'with_effects.mp4');
    await this.videoFilters.applyMultipleEffects(currentFile, effectsPath, {
      textOverlays: options.textOverlays,
      filter: options.filter,
      videoMetadata: metadata,
    });
    currentFile = effectsPath;
  }

  // STEP 2: Generate variants (720p, 480p, 360p)
  const variants = [];
  if (options.generateVariants) {
    const variantPromises = [
      this.encodeVariant(currentFile, '720p', workDir),
      this.encodeVariant(currentFile, '480p', workDir),
      this.encodeVariant(currentFile, '360p', workDir),
    ];
    const variantPaths = await Promise.all(variantPromises);
    variants.push(
      { resolution: '720p', path: variantPaths[0] },
      { resolution: '480p', path: variantPaths[1] },
      { resolution: '360p', path: variantPaths[2] }
    );
  }

  // STEP 3: Generate cover thumbnail
  let thumbnail: string | undefined;
  if (options.generateThumbnail) {
    const coverPath = path.join(workDir, 'cover.jpg');
    const coverTimestamp = options.coverTimestampSec ?? 1.0;
    await this.ffmpegUtils.generateCover(currentFile, coverPath, {
      timestampSec: coverTimestamp,
      maxWidth: 1080,
      maxHeight: 1920,
    });
    thumbnail = coverPath;
  }

  return { masterPath: currentFile, variants, thumbnail, metadata };
}
```

#### `trimVideo(inputPath, outputPath, trim)`
Fast video trimming using FFmpeg copy mode.

```typescript
private async trimVideo(
  inputPath: string,
  outputPath: string,
  trim: { startSec: number; endSec: number }
): Promise<void> {
  return new Promise((resolve, reject) => {
    ffmpeg(inputPath)
      .setStartTime(trim.startSec)
      .setDuration(trim.endSec - trim.startSec)
      .outputOptions('-c copy') // Fast copy (no re-encode)
      .output(outputPath)
      .on('end', () => resolve())
      .on('error', (err) => reject(err))
      .run();
  });
}
```

**Why `-c copy`?**  
Copies video/audio streams without re-encoding → 10× faster trimming (1 second vs 10 seconds for 30s video).

#### `encodeVariant(inputPath, resolution, workDir)`
Generate quality variant at specific resolution.

```typescript
private async encodeVariant(
  inputPath: string,
  resolution: '720p' | '480p' | '360p',
  workDir: string
): Promise<string> {
  const configs = {
    '720p': { width: 1280, height: 720, bitrate: '2000k' },
    '480p': { width: 854, height: 480, bitrate: '1000k' },
    '360p': { width: 640, height: 360, bitrate: '500k' },
  };
  const config = configs[resolution];
  const outputPath = path.join(workDir, `variant_${resolution}.mp4`);

  return new Promise((resolve, reject) => {
    ffmpeg(inputPath)
      .outputOptions([
        `-vf scale=${config.width}:${config.height}:force_original_aspect_ratio=decrease:force_divisible_by=2`,
        `-b:v ${config.bitrate}`,
        '-preset veryfast',
        '-profile:v main',
        '-level 4.0',
        '-pix_fmt yuv420p',
        '-movflags +faststart',
      ])
      .videoCodec('libx264')
      .audioCodec('aac')
      .audioBitrate('128k')
      .output(outputPath)
      .on('end', () => resolve(outputPath))
      .on('error', (err) => reject(err))
      .run();
  });
}
```

**Key FFmpeg Flags**:
- `force_original_aspect_ratio=decrease`: Scale to fit within dimensions (no cropping)
- `force_divisible_by=2`: Ensure even dimensions (H.264 requirement)
- `preset veryfast`: Balance speed vs compression (faster = larger files)
- `movflags +faststart`: Move metadata to start of file (enables progressive download)

### 3.3 FFmpegUtilsService

**Purpose**: Video metadata detection, cover generation, image-to-video conversion.

**Key Methods**:

#### `detectVideoMetadata(videoPath)`
Industry-grade aspect ratio and metadata detection.

```typescript
async detectVideoMetadata(videoPath: string): Promise<VideoMetadata> {
  return new Promise((resolve, reject) => {
    ffmpeg.ffprobe(videoPath, (err, metadata) => {
      if (err) return reject(err);

      const videoStream = metadata.streams.find(s => s.codec_type === 'video');
      if (!videoStream) return reject(new Error('No video stream found'));

      const width = videoStream.width || 0;
      const height = videoStream.height || 0;
      const aspectRatio = width / height;
      const rotation = this.detectRotation(videoStream);

      // Swap dimensions if rotated 90° or 270°
      const isRotated = rotation === 90 || rotation === 270;
      const effectiveWidth = isRotated ? height : width;
      const effectiveHeight = isRotated ? width : height;

      resolve({
        width,
        height,
        aspectRatio,
        aspectCategory: this.classifyAspectRatio(aspectRatio),
        effectiveWidth,
        effectiveHeight,
        effectiveAspectRatio: effectiveWidth / effectiveHeight,
        effectiveAspectCategory: this.classifyAspectRatio(effectiveWidth / effectiveHeight),
        rotation,
        duration: metadata.format.duration || 0,
        codec: videoStream.codec_name || 'unknown',
        bitrate: parseInt(metadata.format.bit_rate?.toString() || '0', 10),
        fps: this.extractFPS(videoStream),
        colorSpace: videoStream.pix_fmt || 'unknown',
        hasAudio: metadata.streams.some(s => s.codec_type === 'audio'),
      });
    });
  });
}

private classifyAspectRatio(aspectRatio: number): AspectCategory {
  if (aspectRatio < 0.6) return 'vertical';   // 9:16 and taller
  if (aspectRatio > 1.6) return 'horizontal'; // 16:9 and wider
  if (aspectRatio >= 0.9 && aspectRatio <= 1.1) return 'square';
  return aspectRatio < 1 ? 'vertical' : 'horizontal';
}
```

**Output Example**:
```typescript
{
  width: 1080,
  height: 1920,
  aspectRatio: 0.5625,
  aspectCategory: 'vertical',
  effectiveWidth: 1080,
  effectiveHeight: 1920,
  effectiveAspectRatio: 0.5625,
  effectiveAspectCategory: 'vertical',
  rotation: 0,
  duration: 32.5,
  codec: 'h264',
  bitrate: 8500000,
  fps: 30,
  colorSpace: 'yuv420p',
  hasAudio: true
}
```

#### `generateCover(videoPath, outputPath, options)`
Extract cover thumbnail with aspect ratio preservation.

```typescript
async generateCover(
  videoPath: string,
  outputPath: string,
  options: CoverGenerationOptions
): Promise<void> {
  const metadata = await this.detectVideoMetadata(videoPath);
  const timestampSec = options.timestampSec ?? 0;
  const safeTimestamp = Math.min(timestampSec, Math.max(0, metadata.duration - 0.1));

  const targetSize = this.calculateCoverSize(
    metadata.effectiveWidth,
    metadata.effectiveHeight,
    options.maxWidth || 1080,
    options.maxHeight || 1920
  );

  return new Promise((resolve, reject) => {
    ffmpeg(videoPath)
      .seekInput(safeTimestamp)
      .frames(1)
      .size(`${targetSize.width}x${targetSize.height}`)
      .outputOptions([
        '-q:v 2', // High quality JPEG (90%)
        `-vf scale='min(${targetSize.width}\\,iw)':'min(${targetSize.height}\\,ih)':force_original_aspect_ratio=decrease`,
      ])
      .output(outputPath)
      .on('end', () => resolve())
      .on('error', (err) => reject(err))
      .run();
  });
}

private calculateCoverSize(
  sourceWidth: number,
  sourceHeight: number,
  maxWidth: number,
  maxHeight: number
): { width: number; height: number } {
  const sourceAspect = sourceWidth / sourceHeight;
  const targetAspect = maxWidth / maxHeight;

  let width: number, height: number;
  if (sourceAspect > targetAspect) {
    width = maxWidth;
    height = Math.round(maxWidth / sourceAspect);
  } else {
    height = maxHeight;
    width = Math.round(maxHeight * sourceAspect);
  }

  // Ensure even dimensions
  width = width % 2 === 0 ? width : width - 1;
  height = height % 2 === 0 ? height : height - 1;

  return { width, height };
}
```

#### `convertImageToVideo(imagePath, outputPath, durationSec)`
Convert static image to video (Instagram-style).

```typescript
async convertImageToVideo(
  imagePath: string,
  outputPath: string,
  durationSec: number
): Promise<void> {
  return new Promise((resolve, reject) => {
    ffmpeg(imagePath)
      .inputOptions(['-loop 1']) // Loop input indefinitely
      .fps(30)
      .outputOptions([
        '-t', durationSec.toString(),
        '-c:v', 'libx264',
        '-pix_fmt', 'yuv420p',
        '-preset', 'fast',
        '-crf', '23',
        '-movflags', '+faststart',
        '-vf', 'scale=min(1080\\,iw):min(1920\\,ih):force_original_aspect_ratio=decrease,pad=1080:1920:(ow-iw)/2:(oh-ih)/2:black',
      ])
      .output(outputPath)
      .on('end', () => resolve())
      .on('error', (err) => reject(err))
      .run();
  });
}
```

**FFmpeg Command Breakdown**:
- `-loop 1`: Loop input image
- `-t 5`: Duration 5 seconds
- `-vf scale=...`: Scale to fit 1080x1920, then pad with black bars

### 3.4 VideoFiltersService

**Purpose**: Text overlay burning, image filters.

**Key Methods**:

#### `applyTextOverlays(inputPath, outputPath, overlays, videoMetadata)`
Burn text overlays permanently into video.

```typescript
async applyTextOverlays(
  inputPath: string,
  outputPath: string,
  overlays: ReelOverlay[],
  videoMetadata: VideoMetadata
): Promise<void> {
  const TARGET_WIDTH = 1080;
  const TARGET_HEIGHT = 1920;

  // Step 1: Normalize aspect ratio to 9:16 (force 1080x1920 canvas)
  const normalizationFilters = [
    `scale=${TARGET_WIDTH}:${TARGET_HEIGHT}:force_original_aspect_ratio=decrease`,
    `pad=${TARGET_WIDTH}:${TARGET_HEIGHT}:(ow-iw)/2:(oh-ih)/2:black`
  ];

  let filterChain = normalizationFilters.join(',');
  const tempFiles: string[] = [];

  try {
    // Step 2: Build drawtext filters for each overlay
    if (overlays.length > 0) {
      const textFilePaths: string[] = [];
      for (let i = 0; i < overlays.length; i++) {
        const filePath = path.join(workDir, `overlay_text_${Date.now()}_${i}.txt`);
        await fs.writeFile(filePath, overlays[i].content, 'utf8');
        tempFiles.push(filePath);
        textFilePaths.push(filePath);
      }

      const textFilters = this.buildTextFilterChain(overlays, TARGET_WIDTH, TARGET_HEIGHT, textFilePaths);
      filterChain += ',' + textFilters;
    }

    // Step 3: Apply filter chain with FFmpeg
    await new Promise<void>((resolve, reject) => {
      ffmpeg(inputPath)
        .complexFilter(filterChain)
        .outputOptions([
          '-c:v libx264',
          '-preset medium',
          '-crf 23',
          '-profile:v main',
          '-pix_fmt yuv420p',
          '-c:a copy',
          '-movflags +faststart',
        ])
        .output(outputPath)
        .on('end', () => resolve())
        .on('error', (err) => reject(err))
        .run();
    });
  } finally {
    // Cleanup temp text files
    await Promise.all(tempFiles.map(f => fs.unlink(f).catch(() => {})));
  }
}
```

#### `buildTextFilterChain(overlays, canvasWidth, canvasHeight, textFiles)`
Build FFmpeg `drawtext` filter chain for all overlays.

```typescript
private buildTextFilterChain(
  overlays: ReelOverlay[],
  canvasWidth: number,
  canvasHeight: number,
  textFiles: string[]
): string {
  const REFERENCE_PREVIEW_WIDTH = 390;
  const scaleFactor = canvasWidth / REFERENCE_PREVIEW_WIDTH; // 1080 / 390 = 2.77

  const filters = overlays.map((overlay, index) => {
    // Font size scaling
    const previewFontSize = overlay.style.fontSize || 24;
    const userScale = Math.max(0.3, Math.min(overlay.scale, 5.0));
    const videoFontSize = Math.round(previewFontSize * scaleFactor * userScale);
    const clampedFontSize = Math.max(16, Math.min(videoFontSize, 300));

    // Position mapping (normalized 0-1 → absolute pixels)
    const normalizedX = Math.max(0, Math.min(1, overlay.position.x));
    const normalizedY = Math.max(0, Math.min(1, overlay.position.y));
    const leftX = Math.round(normalizedX * canvasWidth);
    const topY = Math.round(normalizedY * canvasHeight);

    const xExpr = `max(0, min(${leftX}, ${canvasWidth}-text_w))`;
    const yExpr = `max(0, min(${topY}, ${canvasHeight}-text_h))`;

    // Font and color
    const fontColor = this.convertColorToFFmpeg(overlay.style.color);
    const fontFile = this.selectFontFile(overlay.style.fontWeight);

    // Background style
    let boxOpts = '';
    switch (overlay.style.background) {
      case 'pill':
        const boxPadding = Math.max(8, Math.round(clampedFontSize * 0.5));
        boxOpts = `:box=1:boxcolor=black@0.6:boxborderw=${boxPadding}`;
        break;
      case 'shadow':
        const shadowY = Math.max(2, Math.round(clampedFontSize * 0.08));
        boxOpts = `:shadowx=0:shadowy=${shadowY}:shadowcolor=black@0.75`;
        break;
    }

    // Text content (use textfile for multi-line support)
    const escapedPath = this.escapeFFmpegPath(textFiles[index]);
    const contentOption = `textfile='${escapedPath}'`;

    // Timing
    const enableExpr = `between(t,${overlay.startTime},${overlay.endTime})`;

    return `drawtext=${contentOption}:expansion=none:fontfile=${fontFile}:fontsize=${clampedFontSize}:fontcolor=${fontColor}:x='${xExpr}':y='${yExpr}'${boxOpts}:enable='${enableExpr}'`;
  });

  return filters.join(',');
}
```

**FFmpeg `drawtext` Filter Anatomy**:
```
drawtext=
  textfile='/tmp/overlay_text_123.txt':  ← Multi-line text from file
  expansion=none:                        ← Don't expand variables
  fontfile=/usr/share/fonts/...:         ← Font path
  fontsize=48:                           ← Font size (pixels)
  fontcolor=0xFFFFFF:                    ← Hex color
  x='max(0, min(108, 1080-text_w))':     ← X position (clamped)
  y='max(0, min(192, 1920-text_h))':     ← Y position (clamped)
  box=1:boxcolor=black@0.6:boxborderw=24: ← Pill background
  enable='between(t,0,5.5)'              ← Timing (0s to 5.5s)
```

#### `applyImageFilter(inputPath, outputPath, filter)`
Apply image filters (brightness, contrast, saturation, warmth, vignette).

```typescript
async applyImageFilter(
  inputPath: string,
  outputPath: string,
  filter: VideoFilter
): Promise<void> {
  const filterChain = this.buildImageFilterChain(filter);

  return new Promise((resolve, reject) => {
    ffmpeg(inputPath)
      .videoFilters(filterChain)
      .outputOptions(['-c:v libx264', '-preset medium', '-crf 23', '-c:a copy', '-movflags +faststart'])
      .output(outputPath)
      .on('end', () => resolve())
      .on('error', (err) => reject(err))
      .run();
  });
}

private buildImageFilterChain(filter: VideoFilter): string {
  const filters: string[] = [];

  if (filter.brightness !== undefined && filter.brightness !== 0) {
    filters.push(`eq=brightness=${filter.brightness}`);
  }

  if (filter.contrast !== undefined && filter.contrast !== 0) {
    const contrastValue = 1.0 + filter.contrast; // Map -1 to 1 → 0 to 2
    filters.push(`eq=contrast=${contrastValue}`);
  }

  if (filter.saturation !== undefined && filter.saturation !== 0) {
    const satValue = 1.0 + filter.saturation;
    filters.push(`eq=saturation=${satValue}`);
  }

  if (filter.warmth !== undefined && filter.warmth !== 0) {
    const tempAdjust = filter.warmth > 0 ? 6500 + filter.warmth * 3000 : 6500 - Math.abs(filter.warmth) * 2000;
    filters.push(`colortemperature=temperature=${tempAdjust}`);
  }

  if (filter.vignette !== undefined && filter.vignette > 0) {
    filters.push('vignette=angle=PI/3:mode=forward:eval=frame:dither=1');
  }

  if (filter.preset) {
    filters.push(...this.getPresetFilters(filter.preset));
  }

  return filters.join(',');
}
```

**Preset Filters**:
```typescript
private getPresetFilters(preset: string): string[] {
  const presets: Record<string, string[]> = {
    vintage: ['eq=saturation=0.7:contrast=1.1', 'colortemperature=temperature=7500', 'vignette'],
    vibrant: ['eq=saturation=1.4:contrast=1.2:brightness=0.05'],
    bw: ['hue=s=0', 'eq=contrast=1.15'],
    cool: ['colortemperature=temperature=4500', 'eq=contrast=1.1'],
    warm: ['colortemperature=temperature=8000', 'eq=saturation=1.1'],
    dramatic: ['eq=contrast=1.3:brightness=-0.05', 'vignette'],
  };
  return presets[preset] || [];
}
```

---

## 4. Bull Queue Integration

### 4.1 Queue Configuration

**Queue Name**: `video-processing`  
**Redis Configuration**: Inherited from app-level Redis connection.

**Job Options**:
```typescript
{
  attempts: 3,                      // Retry failed jobs 3 times
  backoff: {
    type: 'exponential',           // Exponential backoff
    delay: 5000,                   // Start with 5s delay (5s, 10s, 20s)
  },
  removeOnComplete: 100,           // Keep last 100 completed jobs (for debugging)
  removeOnFail: 500,               // Keep last 500 failed jobs (for analysis)
}
```

### 4.2 Job Lifecycle

```
[1] Job Created (Media module)
      │
      ↓
[2] Job Enqueued (Redis)
      │
      ↓
[3] Worker Picks Job (VideoProcessingProcessor)
      │
      ├─→ [Success] → Update database → Clean up → Mark complete
      │
      └─→ [Failure] → Log error → Retry (if attempts < 3)
                           │
                           ├─→ [Retry 1] Backoff 5s → Re-process
                           ├─→ [Retry 2] Backoff 10s → Re-process
                           └─→ [Retry 3] Backoff 20s → Re-process
                                      │
                                      ├─→ [Success] → Complete
                                      └─→ [Final Failure] → Fallback to MediaConvert → Clean up ghost reel
```

### 4.3 Job Progress Tracking

**Progress Updates** (0-100%):
```typescript
await job.progress(20);  // Downloaded from S3
await job.progress(70);  // Processing complete
await job.progress(90);  // Uploaded to S3
await job.progress(100); // Database updated
```

**Monitoring**: Frontend can poll job progress using Bull Board (dev tool) or custom progress API.

### 4.4 Event Handlers

#### `@OnQueueCompleted()`
```typescript
@OnQueueCompleted()
async onCompleted(job: Job, result: any) {
  this.logger.log(`✅ Job ${job.id} completed successfully`);
  this.logger.debug(`Result:`, result);
}
```

#### `@OnQueueFailed()`
```typescript
@OnQueueFailed()
async onFailed(job: Job, error: Error) {
  this.logger.error(`❌ Job ${job.id} failed after ${job.attemptsMade} attempts`);
  
  if (job.attemptsMade >= 3) {
    const { mediaId } = job.data as VideoProcessingJobData;
    
    // Clean up ghost reel (prevent blank entries in user profile)
    await this.reelModel.deleteOne({ mediaId });
    await this.mediaModel.findByIdAndUpdate(mediaId, {
      status: MediaStatus.FAILED,
      errorMessage: error.message,
    });
    
    this.logger.log(`🧹 Cleaned up failed Reel/Media for ${mediaId}`);
  }
}
```

---

## 5. FFmpeg Processing

### 5.1 FFmpeg Installation

**Alpine Linux (Production Docker)**:
```dockerfile
RUN apk add --no-cache ffmpeg
```

**Ubuntu/Debian**:
```bash
sudo apt-get install -y ffmpeg
```

**macOS (Development)**:
```bash
brew install ffmpeg
```

**Verification**:
```bash
ffmpeg -version
ffprobe -version
```

### 5.2 FFmpeg Command Patterns

#### Trim Video (Fast Copy)
```bash
ffmpeg -i input.mp4 -ss 5 -t 30 -c copy output.mp4
```
- `-ss 5`: Start at 5 seconds
- `-t 30`: Duration 30 seconds
- `-c copy`: Copy streams (no re-encode)

#### Generate Quality Variant
```bash
ffmpeg -i input.mp4 \
  -vf "scale=1280:720:force_original_aspect_ratio=decrease:force_divisible_by=2" \
  -b:v 2000k \
  -preset veryfast \
  -profile:v main \
  -level 4.0 \
  -pix_fmt yuv420p \
  -movflags +faststart \
  -c:a aac \
  -b:a 128k \
  output_720p.mp4
```

#### Extract Thumbnail
```bash
ffmpeg -ss 12.5 -i input.mp4 -frames:v 1 -q:v 2 \
  -vf "scale='min(1080\,iw)':'min(1920\,ih)':force_original_aspect_ratio=decrease" \
  thumbnail.jpg
```

#### Burn Text Overlay
```bash
ffmpeg -i input.mp4 \
  -vf "drawtext=textfile='overlay.txt':fontfile=/usr/share/fonts/DejaVuSans.ttf:fontsize=48:fontcolor=0xFFFFFF:x='(w-text_w)/2':y='(h-text_h)/2':enable='between(t,0,5)'" \
  -c:v libx264 -preset medium -crf 23 -c:a copy \
  output.mp4
```

#### Apply Image Filter (Vintage)
```bash
ffmpeg -i input.mp4 \
  -vf "eq=saturation=0.7:contrast=1.1,colortemperature=temperature=7500,vignette" \
  -c:v libx264 -preset medium -crf 23 -c:a copy \
  output.mp4
```

### 5.3 FFmpeg Performance Tuning

**Preset Selection** (speed vs compression):
| Preset | Speed | File Size | Use Case |
|--------|-------|-----------|----------|
| ultrafast | 10× | 3× larger | Not recommended |
| veryfast | 5× | 1.5× larger | Variant generation |
| fast | 3× | 1.2× larger | Image-to-video conversion |
| medium | 1× | 1× (baseline) | Effects application |
| slow | 0.5× | 0.9× smaller | Not used (too slow) |

**CRF (Constant Rate Factor)** (quality vs size):
| CRF | Quality | File Size | Use Case |
|-----|---------|-----------|----------|
| 18 | Visually lossless | 2× baseline | Not used (too large) |
| 23 | High quality | 1× baseline | **Default (balanced)** |
| 28 | Medium quality | 0.5× baseline | Not used (visible artifacts) |

**Recommended Configuration**:
- Variant generation: `-preset veryfast -crf 23`
- Effects application: `-preset medium -crf 23`
- Image-to-video: `-preset fast -crf 23`

---

## 6. S3 Integration

### 6.1 Bucket Architecture

**Input Bucket**: `chefooz-media-input`
- **Purpose**: Store raw uploaded videos from mobile app
- **Lifecycle**: Delete original after processing (if trimmed)
- **Access**: Private (presigned URLs for upload)

**Output Bucket**: `chefooz-media-output`
- **Purpose**: Store processed videos (master + variants + thumbnails)
- **Lifecycle**: Permanent (until user deletes reel)
- **Access**: Public read (via CloudFront CDN)

### 6.2 S3 Key Structure

**Input Bucket**:
```
uploads/<userId>/original.mp4
uploads/<userId>/IMG_1234.jpg
```

**Output Bucket**:
```
videos/<mediaId>/master.mp4
videos/<mediaId>/variant_720p.mp4
videos/<mediaId>/variant_480p.mp4
videos/<mediaId>/variant_360p.mp4
thumbnails/<mediaId>/cover.jpg
```

### 6.3 S3 URL Conversion

**S3 URI → HTTPS URL** (with CDN support):

```typescript
import { s3UriToHttps } from '../../utils/s3-url.util';

const s3Uri = 's3://chefooz-media-output/videos/123/master.mp4';
const cdnUrl = this.configService.get<string>('CDN_URL'); // https://cdn.chefooz.com

const httpsUrl = s3UriToHttps(s3Uri, cdnUrl);
// Result: https://cdn.chefooz.com/videos/123/master.mp4
```

**Implementation**:
```typescript
export function s3UriToHttps(s3Uri: string, cdnUrl?: string): string {
  const match = s3Uri.match(/s3:\/\/([^\/]+)\/(.*)/);
  if (!match) return s3Uri;

  const [, bucket, key] = match;
  
  if (cdnUrl) {
    return `${cdnUrl}/${key}`;
  } else {
    return `https://${bucket}.s3.amazonaws.com/${key}`;
  }
}
```

### 6.4 S3 SDK v3 Usage

**Client Initialization**:
```typescript
this.s3Client = new S3Client({
  region: this.configService.get<string>('AWS_REGION') || 'ap-south-1',
  credentials: {
    accessKeyId: this.configService.get<string>('AWS_ACCESS_KEY_ID')!,
    secretAccessKey: this.configService.get<string>('AWS_SECRET_ACCESS_KEY')!,
  },
});
```

**Download from S3**:
```typescript
const command = new GetObjectCommand({ Bucket: 'chefooz-media-input', Key: 'uploads/user123/original.mp4' });
const response = await this.s3Client.send(command);
const readableStream = response.Body as Readable;
```

**Upload to S3**:
```typescript
const fileContent = await fs.readFile(localPath);
const command = new PutObjectCommand({
  Bucket: 'chefooz-media-output',
  Key: 'videos/123/master.mp4',
  Body: fileContent,
  ContentType: 'video/mp4',
});
await this.s3Client.send(command);
```

**Delete from S3**:
```typescript
const command = new DeleteObjectCommand({ Bucket: 'chefooz-media-input', Key: 'uploads/user123/original.mp4' });
await this.s3Client.send(command);
```

---

## 7. Database Schema

### 7.1 Media Document (MongoDB)

**Collection**: `media`  
**Schema**: `apps/chefooz-apis/src/modules/media/media.schema.ts`

```typescript
{
  _id: ObjectId("507f1f77bcf86cd799439011"),
  userId: "user123",
  uploadId: "upload-uuid-1234",
  type: "reel_video", // 'reel_video' | 'review_video' | 'menu_item_image' | 'promo_video'
  status: "ready",    // 'uploading' | 'uploaded' | 'processing' | 'ready' | 'failed'
  originalS3Key: "uploads/user123/original.mp4",
  masterVideoUrl: "https://cdn.chefooz.com/videos/123/master.mp4",
  variants: [
    {
      quality: "720p",
      url: "https://cdn.chefooz.com/videos/123/variant_720p.mp4",
      width: 1280,
      height: 720,
      bitrate: 2000000
    },
    {
      quality: "480p",
      url: "https://cdn.chefooz.com/videos/123/variant_480p.mp4",
      width: 854,
      height: 480,
      bitrate: 1000000
    },
    {
      quality: "360p",
      url: "https://cdn.chefooz.com/videos/123/variant_360p.mp4",
      width: 640,
      height: 360,
      bitrate: 500000
    }
  ],
  thumbnailUrl: "https://cdn.chefooz.com/thumbnails/123/cover.jpg",
  durationSec: 32.5,
  sizeBytes: 15728640,
  processingMethod: "ffmpeg", // 'ffmpeg' | 'mediaconvert'
  processingError: null,
  moderationStatus: "approved", // 'pending' | 'approved' | 'rejected' | 'flagged'
  moderationReason: null,
  uploadedAt: "2026-02-14T10:25:00Z",
  processedAt: "2026-02-14T10:25:35Z",
  createdAt: "2026-02-14T10:24:30Z",
  updatedAt: "2026-02-14T10:25:40Z"
}
```

**Key Fields**:
- `status`: Tracks processing lifecycle
- `processingMethod`: 'ffmpeg' (primary) or 'mediaconvert' (fallback)
- `variants[]`: Array of quality variants with URLs and dimensions
- `moderationStatus`: AI/manual content moderation result

### 7.2 Reel Document (MongoDB)

**Collection**: `reels`  
**Schema**: `apps/chefooz-apis/src/database/schemas/reel.schema.ts`

```typescript
{
  _id: ObjectId("507f1f77bcf86cd799439012"),
  userId: "user123",
  mediaId: "507f1f77bcf86cd799439011",
  videoUrl: "https://cdn.chefooz.com/videos/123/variant_720p.mp4", // Primary playback (720p)
  thumbnailUrl: "https://cdn.chefooz.com/thumbnails/123/cover.jpg",
  durationSec: 32.5,
  caption: "Check out my signature biryani! 🍛",
  hashtags: ["biryani", "chefooz", "homemade"],
  mentions: ["@user456"],
  location: { type: "Point", coordinates: [77.5946, 12.9716] },
  locationName: "Bangalore, Karnataka",
  visibility: "public", // 'public' | 'followers' | 'private'
  likesCount: 0,
  commentsCount: 0,
  sharesCount: 0,
  viewsCount: 0,
  createdAt: "2026-02-14T10:25:40Z",
  updatedAt: "2026-02-14T10:25:40Z"
}
```

**Update Flow**:
1. Media status changes to 'ready'
2. VideoProcessingProcessor updates `videoUrl`, `thumbnailUrl`, `durationSec`
3. Reel becomes visible in user profile and feeds

### 7.3 MediaConvertJob Document (MongoDB)

**Collection**: `mediaconvert_jobs`  
**Schema**: `apps/chefooz-apis/src/integrations/media-convert/schemas/mediaconvert-job.schema.ts`

```typescript
{
  _id: ObjectId("507f1f77bcf86cd799439013"),
  jobId: "1234567890123-abcdef", // AWS MediaConvert job ID
  mediaId: "507f1f77bcf86cd799439011",
  userId: "user123",
  status: "PROGRESSING", // 'SUBMITTED' | 'PROGRESSING' | 'COMPLETE' | 'ERROR' | 'CANCELED'
  inputS3Key: "uploads/user123/original.mp4",
  outputS3KeyPrefix: "converted/123/",
  ffmpegFailureReason: "Unsupported codec: hevc with HDR metadata",
  submittedAt: "2026-02-14T10:26:00Z",
  completedAt: null,
  errorMessage: null,
  createdAt: "2026-02-14T10:26:00Z",
  updatedAt: "2026-02-14T10:26:30Z"
}
```

**Purpose**: Track MediaConvert fallback jobs for smart polling.

---

## 8. API Reference

### 8.1 Test Endpoints (Development Only)

**Controller**: `MediaProcessingTestController`  
**Base Path**: `/api/v1/media-processing/test`  
**Auth**: JWT required

**⚠️ WARNING**: These endpoints are for development testing only. Remove or secure before production.

#### POST `/api/v1/media-processing/test/metadata`
Detect video metadata.

**Request**:
```json
{
  "videoPath": "/tmp/test-video.mp4"
}
```

**Response**:
```json
{
  "success": true,
  "data": {
    "width": 1080,
    "height": 1920,
    "aspectRatio": 0.5625,
    "aspectCategory": "vertical",
    "duration": 32.5,
    "codec": "h264",
    "fps": 30,
    "hasAudio": true
  }
}
```

#### POST `/api/v1/media-processing/test/cover`
Generate cover thumbnail.

**Request**:
```json
{
  "videoPath": "/tmp/test-video.mp4",
  "outputPath": "/tmp/cover.jpg",
  "timestampSec": 12.5,
  "maxWidth": 1080,
  "maxHeight": 1920
}
```

**Response**:
```json
{
  "success": true,
  "message": "Cover generated successfully",
  "outputPath": "/tmp/cover.jpg"
}
```

#### POST `/api/v1/media-processing/test/text-overlay`
Apply text overlays.

**Request**:
```json
{
  "videoPath": "/tmp/test-video.mp4",
  "outputPath": "/tmp/output.mp4",
  "overlays": [
    {
      "content": "Taste the Flavors",
      "startTime": 0,
      "endTime": 5.5,
      "position": { "x": 0.1, "y": 0.1 },
      "scale": 1.0,
      "style": {
        "fontSize": 32,
        "fontWeight": "700",
        "color": "#FFFFFF",
        "background": "pill"
      }
    }
  ],
  "videoMetadata": { "width": 1080, "height": 1920, ... }
}
```

**Response**:
```json
{
  "success": true,
  "message": "Applied 1 text overlays",
  "outputPath": "/tmp/output.mp4"
}
```

#### POST `/api/v1/media-processing/test/filter`
Apply image filter.

**Request**:
```json
{
  "videoPath": "/tmp/test-video.mp4",
  "outputPath": "/tmp/output.mp4",
  "filter": {
    "name": "vintage",
    "brightness": 0.1,
    "contrast": 0.2,
    "saturation": -0.3,
    "warmth": 0.5,
    "vignette": 0.4
  }
}
```

**Response**:
```json
{
  "success": true,
  "message": "Applied filter: vintage",
  "outputPath": "/tmp/output.mp4"
}
```

### 8.2 Integration with Media Module

**Job Enqueue** (Media Module):
```typescript
import { InjectQueue } from '@nestjs/bull';
import { Queue } from 'bull';

@Injectable()
export class MediaService {
  constructor(
    @InjectQueue('video-processing') private videoQueue: Queue
  ) {}

  async enqueueProcessing(mediaId: string, uploadData: any) {
    await this.videoQueue.add('process-video', {
      mediaId,
      s3Key: uploadData.s3Key,
      trim: uploadData.trim,
      coverTimestampSec: uploadData.coverTimestampSec,
      textOverlays: uploadData.textOverlays,
      filter: uploadData.filter,
      userId: uploadData.userId,
    });
  }
}
```

---

## 9. Error Handling

### 9.1 Error Categories

| Error Type | Cause | Recovery Strategy |
|------------|-------|-------------------|
| **FFmpeg Error** | Corrupt video, unsupported codec, invalid command | Retry (3 attempts) → MediaConvert fallback |
| **S3 Error** | Network timeout, invalid credentials, bucket not found | Retry (3 attempts) → Alert ops team |
| **Thumbnail Error** | Invalid timestamp, corrupt frames, seek error | Fallback to timestamp 0 → Use original image (if image-video) |
| **Disk Space Error** | `/tmp` full, large video files | Cleanup incomplete jobs → Alert ops team |
| **Out of Memory** | Large video files (> 500 MB), insufficient RAM | Kill worker pod → Restart → Process with higher memory limit |

### 9.2 Error Logging

**Structured Logging**:
```typescript
this.logger.error(`❌ FFmpeg processing failed for media ${mediaId}:`, {
  mediaId,
  s3Key,
  errorMessage: error.message,
  errorStack: error.stack,
  attemptsMade: job.attemptsMade,
  jobId: job.id,
});
```

**Log Aggregation**:
- Production: AWS CloudWatch Logs
- Development: Console output

**Log Retention**: 30 days (production), 7 days (development).

### 9.3 Fallback: MediaConvert

**Trigger Conditions**:
- FFmpeg fails after 3 retry attempts
- Specific codec known to fail (HEVC with HDR, VP9, AV1)

**Implementation**:
```typescript
try {
  await this.mediaProcessingService.processVideo(inputPath, options);
} catch (ffmpegError) {
  this.logger.warn(`⚠️ FFmpeg failed, attempting MediaConvert fallback`);
  
  await this.mediaModel.findByIdAndUpdate(mediaId, {
    status: MediaStatus.PROCESSING,
    processingMethod: 'mediaconvert',
    processingError: `FFmpeg failed: ${ffmpegError.message}`,
  });

  const mediaConvertService = this.moduleRef.get('MediaConvertService');
  const result = await mediaConvertService.createJob(s3Key, mediaId, `converted/${mediaId}/`);
  
  await this.mediaConvertJobModel.create({
    jobId: result.jobId,
    mediaId,
    userId,
    status: 'SUBMITTED',
    inputS3Key: s3Key,
    ffmpegFailureReason: ffmpegError.message,
  });

  const mediaConvertConsumer = this.moduleRef.get('MediaConvertConsumerService');
  await mediaConvertConsumer.resumePolling();
  
  return { success: true, fallback: true, jobId: result.jobId };
}
```

---

## 10. Testing Strategy

### 10.1 Unit Tests

**Test Files**: `*.spec.ts` (colocated with source files)

**Example** (`media-processing.service.spec.ts`):
```typescript
describe('MediaProcessingService', () => {
  let service: MediaProcessingService;
  let ffmpegUtils: FFmpegUtilsService;
  let videoFilters: VideoFiltersService;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        MediaProcessingService,
        {
          provide: FFmpegUtilsService,
          useValue: {
            detectVideoMetadata: jest.fn(),
            generateCover: jest.fn(),
            convertImageToVideo: jest.fn(),
          },
        },
        {
          provide: VideoFiltersService,
          useValue: {
            applyMultipleEffects: jest.fn(),
          },
        },
      ],
    }).compile();

    service = module.get<MediaProcessingService>(MediaProcessingService);
    ffmpegUtils = module.get<FFmpegUtilsService>(FFmpegUtilsService);
    videoFilters = module.get<VideoFiltersService>(VideoFiltersService);
  });

  it('should detect image and convert to video', async () => {
    jest.spyOn(ffmpegUtils, 'detectVideoMetadata').mockResolvedValue({
      duration: 0.04,
      codec: 'mjpeg',
      // ... other metadata
    });

    jest.spyOn(ffmpegUtils, 'convertImageToVideo').mockResolvedValue();

    await service.processVideo('/tmp/image.jpg', { photoDurationSec: 5 });

    expect(ffmpegUtils.convertImageToVideo).toHaveBeenCalledWith(
      '/tmp/image.jpg',
      expect.stringContaining('image_as_video.mp4'),
      5
    );
  });

  it('should trim video before generating variants', async () => {
    // ... test trim-first optimization
  });

  it('should apply text overlays and filters', async () => {
    // ... test effects application
  });
});
```

**Run Unit Tests**:
```bash
npm run test -- media-processing.service.spec.ts
```

### 10.2 Integration Tests

**Test File**: `apps/chefooz-apis-e2e/src/media-processing.e2e-spec.ts`

**Test Scenarios**:
1. Upload video → Enqueue job → Process → Verify variants uploaded to S3
2. Upload video with trim → Verify original deleted after processing
3. Upload video with text overlays → Verify overlays burned into output
4. Upload image → Verify converted to 5-second video
5. FFmpeg failure → Verify MediaConvert fallback triggered

**Example**:
```typescript
describe('Media Processing E2E', () => {
  it('should process video with text overlays', async () => {
    // 1. Upload video
    const uploadResult = await request(app.getHttpServer())
      .post('/api/v1/media/upload')
      .set('Authorization', `Bearer ${authToken}`)
      .send({ type: 'reel_video', fileName: 'test.mp4' });

    const { mediaId, uploadUrl } = uploadResult.body.data;

    // 2. Upload file to S3 presigned URL
    await axios.put(uploadUrl, videoBuffer, { headers: { 'Content-Type': 'video/mp4' } });

    // 3. Complete upload (triggers processing)
    await request(app.getHttpServer())
      .post(`/api/v1/media/${mediaId}/complete`)
      .set('Authorization', `Bearer ${authToken}`)
      .send({ textOverlays: [{ content: 'Test', startTime: 0, endTime: 5, ... }] });

    // 4. Wait for processing (poll job status)
    await waitForJobCompletion(mediaId, 60000); // 60s timeout

    // 5. Verify variants uploaded
    const media = await request(app.getHttpServer())
      .get(`/api/v1/media/${mediaId}`)
      .set('Authorization', `Bearer ${authToken}`);

    expect(media.body.data.status).toBe('ready');
    expect(media.body.data.variants).toHaveLength(3);
    expect(media.body.data.thumbnailUrl).toBeDefined();
  });
});
```

### 10.3 Performance Tests

**Load Testing** (Artillery):
```yaml
config:
  target: 'https://api.chefooz.com'
  phases:
    - duration: 300
      arrivalRate: 10 # 10 videos/sec
scenarios:
  - name: 'Video Processing Load Test'
    flow:
      - post:
          url: '/api/v1/media/upload'
          json:
            type: 'reel_video'
            fileName: 'test.mp4'
      - think: 5
      - post:
          url: '/api/v1/media/{{ mediaId }}/complete'
```

**Metrics to Track**:
- Average processing time (target: < 40s for 30s video)
- Queue depth (target: < 50 jobs)
- Worker CPU usage (target: < 80%)
- S3 upload/download bandwidth (target: < 100 Mbps per worker)

---

## 11. Deployment & Configuration

### 11.1 Environment Variables

```bash
# AWS Configuration
AWS_REGION=ap-south-1
AWS_ACCESS_KEY_ID=AKIA...
AWS_SECRET_ACCESS_KEY=...
S3_INPUT_BUCKET=chefooz-media-input
S3_OUTPUT_BUCKET=chefooz-media-output
CDN_URL=https://cdn.chefooz.com

# Redis Configuration (Bull Queue)
REDIS_HOST=redis.chefooz.com
REDIS_PORT=6379
REDIS_PASSWORD=...

# Database Configuration
MONGODB_URI=mongodb://...
POSTGRES_URI=postgresql://...

# Processing Configuration (optional)
VIDEO_PROCESSING_CONCURRENCY=2  # Workers per pod
VIDEO_PROCESSING_TIMEOUT=300000 # 5 minutes
```

### 11.2 Docker Configuration

**Dockerfile**:
```dockerfile
FROM node:18-alpine

# Install FFmpeg
RUN apk add --no-cache ffmpeg

# Install fonts for text overlays
RUN apk add --no-cache ttf-dejavu

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY dist ./dist

CMD ["node", "dist/apps/chefooz-apis/main.js"]
```

**Docker Compose** (Local Development):
```yaml
version: '3.8'
services:
  redis:
    image: redis:7-alpine
    ports:
      - '6379:6379'

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: chefooz
      POSTGRES_USER: chefooz
      POSTGRES_PASSWORD: password
    ports:
      - '5432:5432'

  mongodb:
    image: mongo:6
    ports:
      - '27017:27017'

  chefooz-apis:
    build: .
    depends_on:
      - redis
      - postgres
      - mongodb
    environment:
      REDIS_HOST: redis
      MONGODB_URI: mongodb://mongodb:27017/chefooz
      POSTGRES_URI: postgresql://chefooz:password@postgres:5432/chefooz
    ports:
      - '3000:3000'
    volumes:
      - /tmp:/tmp # For FFmpeg temp files
```

### 11.3 Kubernetes Configuration

**Deployment** (`media-processing-deployment.yaml`):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: media-processing-worker
spec:
  replicas: 2
  selector:
    matchLabels:
      app: media-processing-worker
  template:
    metadata:
      labels:
        app: media-processing-worker
    spec:
      containers:
        - name: worker
          image: chefooz/apis:latest
          resources:
            requests:
              cpu: 2000m
              memory: 4Gi
            limits:
              cpu: 4000m
              memory: 8Gi
          env:
            - name: VIDEO_PROCESSING_CONCURRENCY
              value: '2'
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir:
            sizeLimit: 10Gi
```

**Horizontal Pod Autoscaler** (`hpa.yaml`):
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: media-processing-worker-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: media-processing-worker
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

---

## 12. Monitoring & Observability

### 12.1 Metrics

**Bull Queue Metrics** (exposed via Bull Board):
- Queue depth (active + waiting jobs)
- Completed jobs (last 24h)
- Failed jobs (last 24h)
- Average processing time

**Custom Metrics** (Prometheus):
```typescript
import { Counter, Histogram } from 'prom-client';

const processingDuration = new Histogram({
  name: 'video_processing_duration_seconds',
  help: 'Video processing duration',
  labelNames: ['resolution', 'has_overlays', 'has_filter'],
});

const processingErrors = new Counter({
  name: 'video_processing_errors_total',
  help: 'Total video processing errors',
  labelNames: ['error_type'],
});

// Usage:
const timer = processingDuration.startTimer({ resolution: '720p', has_overlays: 'true', has_filter: 'false' });
await encodeVariant(...);
timer();
```

### 12.2 Logging

**Log Levels**:
- `ERROR`: Processing failures, S3 errors, FFmpeg crashes
- `WARN`: Fallback to MediaConvert, retries, invalid timestamps
- `LOG`: Job start/complete, variant generation, database updates
- `DEBUG`: FFmpeg commands, filter chains, progress updates

**Structured Logging Example**:
```typescript
this.logger.log('Starting video processing', {
  mediaId,
  s3Key,
  trim: !!trim,
  overlays: textOverlays?.length || 0,
  filter: filter?.name || 'none',
});
```

### 12.3 Alerts

**PagerDuty/Slack Alerts**:
1. **Queue Depth > 100**: High load, consider scaling workers
2. **Failed Jobs > 10 (last hour)**: Processing issues, investigate logs
3. **Processing Time > 120s (30s video)**: Performance degradation, check CPU/disk
4. **Disk Usage > 80% (/tmp)**: Cleanup incomplete, restart workers
5. **FFmpeg Crashes > 5 (last hour)**: FFmpeg bug or corrupt inputs

**Alert Configuration** (Prometheus Alert Manager):
```yaml
groups:
  - name: media_processing
    rules:
      - alert: HighQueueDepth
        expr: bull_queue_depth{queue="video-processing"} > 100
        for: 5m
        annotations:
          summary: 'Video processing queue depth is high ({{ $value }} jobs)'
```

---

## 13. Performance Optimization

### 13.1 Trim-First Optimization

**Impact**: 70% faster processing for trimmed videos.

**Example**:
```
120s video trimmed to 30s:
- Without optimization: 120s × 3 variants = 360s encoding
- With trim-first: Trim (5s) + 30s × 3 variants = 95s encoding
- Savings: 265s (74%)
```

### 13.2 Parallel Variant Generation

**Impact**: 66% faster variant generation.

**Implementation**:
```typescript
const variantPromises = [
  this.encodeVariant(input, '720p', workDir),
  this.encodeVariant(input, '480p', workDir),
  this.encodeVariant(input, '360p', workDir),
];
const results = await Promise.all(variantPromises); // Parallel execution
```

**Before**: 18s (6s per variant, sequential)  
**After**: 6s (all variants in parallel)

### 13.3 FFmpeg Preset Tuning

**Recommendation**: Use `-preset veryfast` for variants, `-preset medium` for effects.

**Rationale**:
- `veryfast`: 5× faster encoding, 1.5× larger files (acceptable for variants)
- `medium`: Better compression for master/effects output

### 13.4 Disk I/O Optimization

**Use tmpfs for /tmp** (RAM-backed filesystem):
```yaml
volumes:
  - name: tmp
    emptyDir:
      medium: Memory # Use RAM instead of disk
      sizeLimit: 4Gi
```

**Impact**: 2-3× faster file I/O (especially for large videos).

---

## 14. Security Considerations

### 14.1 Input Validation

**Validate S3 Keys**:
```typescript
if (!/^uploads\/[a-zA-Z0-9\-_]+\/[a-zA-Z0-9\-_\.]+$/.test(s3Key)) {
  throw new BadRequestException('Invalid S3 key format');
}
```

**Validate FFmpeg Inputs**:
- Check file size (< 500 MB)
- Check video duration (< 180 seconds)
- Check mime type (video/mp4, image/jpeg, image/png)

### 14.2 S3 Security

**Bucket Policies**:
- Input bucket: Private (presigned URLs only)
- Output bucket: Public read (via CloudFront only, deny direct access)

**Presigned URL Expiry**:
- Upload URLs: 15 minutes
- Download URLs: Not used (worker uses IAM role)

### 14.3 FFmpeg Command Injection Prevention

**Never pass user input directly to FFmpeg**:
```typescript
// ❌ BAD: User input in command
ffmpeg(`-i ${userInput}`)

// ✅ GOOD: Use fluent-ffmpeg API with validated inputs
ffmpeg(validatedPath).setStartTime(validatedTimestamp)
```

**Sanitize Text Overlay Content**:
```typescript
private escapeFFmpegText(text: string): string {
  return text
    .replace(/\\/g, '\\\\')
    .replace(/'/g, "\\'")
    .replace(/\n/g, '\\N');
}
```

### 14.4 Temp File Cleanup

**Always cleanup on success AND error**:
```typescript
try {
  await processVideo();
} catch (error) {
  throw error;
} finally {
  await fs.rm(workDir, { recursive: true, force: true });
}
```

**Cron Job**: Cleanup orphaned /tmp files older than 24 hours.

---

## 15. Troubleshooting Guide

### 15.1 Common Issues

#### Issue: "FFmpeg not found"
**Symptom**: Error `spawn ffmpeg ENOENT`  
**Cause**: FFmpeg not installed or not in PATH  
**Solution**:
```bash
# Alpine Linux
apk add --no-cache ffmpeg

# Verify installation
ffmpeg -version
```

#### Issue: "Out of disk space"
**Symptom**: Error `ENOSPC: no space left on device`  
**Cause**: /tmp directory full  
**Solution**:
```bash
# Check disk usage
df -h /tmp

# Cleanup old processing directories
find /tmp -name "video-processing-*" -mtime +1 -exec rm -rf {} \;

# Increase tmpfs size (Kubernetes)
kubectl edit deployment media-processing-worker
# Set: emptyDir.sizeLimit: 20Gi
```

#### Issue: "Cover generation fails at timestamp"
**Symptom**: Error `frame extraction error`  
**Cause**: Invalid timestamp (exceeds video duration)  
**Solution**: Automatically handled with fallback logic:
```typescript
if (coverTimestampSec >= videoDuration) {
  coverTimestampSec = Math.max(0, videoDuration - 0.5);
}
```

#### Issue: "Text overlay not appearing"
**Symptom**: Video processed but no text visible  
**Cause**: Font file not found, invalid filter syntax  
**Solution**:
```bash
# Check font availability
ls -la /usr/share/fonts/ttf-dejavu/

# Install fonts (if missing)
apk add --no-cache ttf-dejavu

# Check FFmpeg stderr logs for filter errors
```

#### Issue: "Video playback stutters"
**Symptom**: Video plays but stutters on mobile devices  
**Cause**: Missing `-movflags +faststart` flag  
**Solution**: Ensure all FFmpeg encode commands include:
```typescript
.outputOptions('-movflags +faststart')
```

### 15.2 Debugging Commands

**Check FFmpeg Version**:
```bash
ffmpeg -version
ffprobe -version
```

**Test FFmpeg Encoding**:
```bash
ffmpeg -i /tmp/test.mp4 -c:v libx264 -preset veryfast -crf 23 /tmp/output.mp4
```

**Check Redis Queue**:
```bash
redis-cli
> KEYS bull:video-processing:*
> LLEN bull:video-processing:wait
> LLEN bull:video-processing:active
```

**Check S3 Uploads**:
```bash
aws s3 ls s3://chefooz-media-output/videos/123/
```

**Check Job Logs**:
```bash
kubectl logs -f deployment/media-processing-worker | grep "mediaId: 123"
```

### 15.3 Performance Debugging

**Profile FFmpeg Encoding**:
```bash
time ffmpeg -i input.mp4 -c:v libx264 -preset veryfast output.mp4
```

**Check Worker CPU Usage**:
```bash
kubectl top pods -l app=media-processing-worker
```

**Check Queue Metrics**:
```bash
curl http://localhost:3000/bull-board
```

---

## 16. References

**Internal Documentation**:
- [Media Module Overview](../media/FEATURE_OVERVIEW.md)
- [Media-Processing Feature Overview](./FEATURE_OVERVIEW.md)
- [Media-Processing QA Test Cases](./QA_TEST_CASES.md)

**External Resources**:
- [FFmpeg Documentation](https://ffmpeg.org/documentation.html)
- [Bull Queue Documentation](https://github.com/OptimalBits/bull)
- [AWS S3 SDK v3](https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/clients/client-s3/)
- [fluent-ffmpeg API](https://github.com/fluent-ffmpeg/node-fluent-ffmpeg)

---

**[END OF TECHNICAL GUIDE]**  
**Document Status**: ✅ Complete  
**Next Review Date**: May 14, 2026
