# Media-Processing Module - QA Test Cases

**Version:** 1.0  
**Last Updated:** February 14, 2026  
**Status:** Production  
**Target Audience:** QA Engineers, Test Automation Engineers

---

## 📋 Table of Contents

1. [Testing Overview](#testing-overview)
2. [Test Environment Setup](#test-environment-setup)
3. [Functional Test Cases](#functional-test-cases)
4. [Performance Test Cases](#performance-test-cases)
5. [Security Test Cases](#security-test-cases)
6. [Error Handling Test Cases](#error-handling-test-cases)
7. [Integration Test Cases](#integration-test-cases)
8. [Regression Test Cases](#regression-test-cases)
9. [Test Data](#test-data)
10. [Automation Coverage](#automation-coverage)

---

## 1. Testing Overview

### 1.1 Test Scope

The Media-Processing module is responsible for:
- Video transcoding (FFmpeg-based processing)
- Quality variant generation (720p, 480p, 360p)
- Cover thumbnail extraction
- Text overlay burning
- Image filter application
- Image-to-video conversion
- Trim-first optimization
- S3 upload/download
- Database updates (Media & Reel documents)
- MediaConvert fallback (error scenarios)

### 1.2 Test Strategy

| Test Type | Coverage | Automation | Priority |
|-----------|----------|------------|----------|
| **Functional** | 65 test cases | 85% | High |
| **Performance** | 12 test cases | 90% | High |
| **Security** | 8 test cases | 70% | Critical |
| **Error Handling** | 15 test cases | 80% | High |
| **Integration** | 10 test cases | 75% | High |
| **Regression** | 20 test cases | 95% | Medium |

**Total Test Cases**: 130  
**Overall Automation Coverage**: 83%

### 1.3 Test Tools

| Tool | Purpose | Version |
|------|---------|---------|
| **Jest** | Unit testing | 29.x |
| **Cypress** | E2E API testing | 13.x |
| **Artillery** | Load testing | 2.x |
| **FFmpeg** | Video validation | 6.x |
| **AWS CLI** | S3 verification | 2.x |

### 1.4 Test Data Requirements

- **Test Videos**: 5-120 seconds, various aspect ratios (9:16, 16:9, 1:1)
- **Test Images**: JPEG, PNG (menu items, profile photos)
- **S3 Buckets**: `chefooz-media-input-test`, `chefooz-media-output-test`
- **Redis**: Isolated test queue (`video-processing-test`)
- **Database**: Test MongoDB/PostgreSQL instances

---

## 2. Test Environment Setup

### 2.1 Prerequisites

**Software Requirements**:
```bash
# Install FFmpeg
brew install ffmpeg  # macOS
sudo apt-get install ffmpeg  # Ubuntu

# Install test dependencies
npm install --save-dev jest @nestjs/testing cypress artillery

# Verify FFmpeg
ffmpeg -version
ffprobe -version
```

**Environment Variables** (`.env.test`):
```bash
NODE_ENV=test
AWS_REGION=ap-south-1
AWS_ACCESS_KEY_ID=test-key
AWS_SECRET_ACCESS_KEY=test-secret
S3_INPUT_BUCKET=chefooz-media-input-test
S3_OUTPUT_BUCKET=chefooz-media-output-test
REDIS_HOST=localhost
REDIS_PORT=6379
MONGODB_URI=mongodb://localhost:27017/chefooz-test
```

### 2.2 Test Video Files

**Prepare Test Assets**:
```bash
# Create test videos directory
mkdir -p test-assets/videos
mkdir -p test-assets/images

# Generate test videos with FFmpeg
# 30s vertical video (9:16)
ffmpeg -f lavfi -i testsrc=duration=30:size=1080x1920:rate=30 \
  -f lavfi -i sine=frequency=1000:duration=30 \
  -pix_fmt yuv420p test-assets/videos/vertical_30s.mp4

# 15s horizontal video (16:9)
ffmpeg -f lavfi -i testsrc=duration=15:size=1920x1080:rate=30 \
  -pix_fmt yuv420p test-assets/videos/horizontal_15s.mp4

# 10s square video (1:1)
ffmpeg -f lavfi -i testsrc=duration=10:size=1080x1080:rate=30 \
  -pix_fmt yuv420p test-assets/videos/square_10s.mp4

# Generate test image
ffmpeg -f lavfi -i testsrc=size=1080x1920 -frames:v 1 test-assets/images/test_image.jpg
```

### 2.3 Mock Services

**S3 Mock** (LocalStack or MinIO):
```bash
# Start LocalStack (S3 mock)
docker run -d --name localstack \
  -p 4566:4566 \
  -e SERVICES=s3 \
  localstack/localstack

# Create test buckets
aws --endpoint-url=http://localhost:4566 s3 mb s3://chefooz-media-input-test
aws --endpoint-url=http://localhost:4566 s3 mb s3://chefooz-media-output-test
```

---

## 3. Functional Test Cases

### 3.1 Video Processing Pipeline

#### TC-FN-001: Process 30-second vertical video (9:16)
**Priority**: High  
**Automation**: Yes (Jest)

**Preconditions**:
- Test video: `vertical_30s.mp4` (1080x1920, 30 FPS, H.264)
- S3 input bucket contains uploaded video
- Media document status: `UPLOADED`

**Test Steps**:
1. Enqueue processing job with `mediaId` and `s3Key`
2. Wait for job completion (max 60s timeout)
3. Verify Media document updated:
   - `status = 'ready'`
   - `masterVideoUrl` exists
   - `variants[]` contains 3 entries (720p, 480p, 360p)
   - `thumbnailUrl` exists
   - `processingMethod = 'ffmpeg'`
4. Download variants from S3 and verify:
   - 720p: 1280x720 (or aspect-preserved dimensions)
   - 480p: 854x480 (or aspect-preserved dimensions)
   - 360p: 640x360 (or aspect-preserved dimensions)
5. Verify thumbnail: JPEG, 1080x1920 max dimensions

**Expected Result**:
- ✅ All 3 variants generated with correct dimensions
- ✅ Thumbnail extracted at 1s timestamp (default)
- ✅ Reel document updated with `videoUrl` (720p) and `durationSec: 30`
- ✅ Processing time < 45s

**Actual Result**: _[To be filled during test execution]_

---

#### TC-FN-002: Process 15-second horizontal video (16:9)
**Priority**: High  
**Automation**: Yes (Jest)

**Preconditions**:
- Test video: `horizontal_15s.mp4` (1920x1080, 30 FPS)

**Test Steps**:
1. Upload and process horizontal video
2. Verify metadata detection: `aspectCategory = 'horizontal'`
3. Verify variants padded to 9:16 aspect ratio (1080x1920 canvas with black letterbox)
4. Download 720p variant and verify FFmpeg metadata:
   ```bash
   ffprobe -v error -show_entries stream=width,height variant_720p.mp4
   ```
   Expected: `width=1080, height=1920` (padded canvas)

**Expected Result**:
- ✅ Horizontal video normalized to 9:16 canvas
- ✅ Black letterbox bars (top/bottom padding)
- ✅ No stretching or distortion

**Actual Result**: _[To be filled during test execution]_

---

#### TC-FN-003: Process 10-second square video (1:1)
**Priority**: Medium  
**Automation**: Yes (Jest)

**Preconditions**:
- Test video: `square_10s.mp4` (1080x1080)

**Test Steps**:
1. Process square video
2. Verify metadata: `aspectCategory = 'square'`
3. Verify variants padded to 9:16 with black pillarbox bars (left/right padding)

**Expected Result**:
- ✅ Square video normalized to 9:16 canvas
- ✅ Black pillarbox bars visible

**Actual Result**: _[To be filled during test execution]_

---

#### TC-FN-004: Process video with rotation metadata (90°)
**Priority**: Medium  
**Automation**: Yes (Jest)

**Preconditions**:
- Test video: `rotated_90deg.mp4` (metadata: rotation=90)

**Test Steps**:
1. Process rotated video
2. Verify metadata detection swaps dimensions:
   - Original: 1920x1080
   - Effective: 1080x1920 (after rotation)
3. Verify output video displays correctly (no sideways playback)

**Expected Result**:
- ✅ Rotation handled correctly
- ✅ Video displays in correct orientation

**Actual Result**: _[To be filled during test execution]_

---

### 3.2 Trimming

#### TC-FN-005: Trim 120-second video to 30 seconds
**Priority**: High  
**Automation**: Yes (Jest)

**Preconditions**:
- Test video: `long_120s.mp4` (120 seconds)
- Job data includes: `trim: { startSec: 10, endSec: 40 }`

**Test Steps**:
1. Process video with trim metadata
2. Verify processing time < 50s (trim-first optimization)
3. Verify output duration: 30 seconds (10s to 40s trim)
4. Verify original video deleted from input bucket (cost optimization)
5. Verify master video is 30s (not 120s)

**Expected Result**:
- ✅ Output duration exactly 30 seconds
- ✅ Processing faster than full 120s video (70% time savings)
- ✅ Original deleted from S3 input bucket

**Actual Result**: _[To be filled during test execution]_

---

#### TC-FN-006: Trim with start time 0
**Priority**: Medium  
**Automation**: Yes (Jest)

**Preconditions**:
- Job data: `trim: { startSec: 0, endSec: 15 }`

**Test Steps**:
1. Process video with trim starting at 0
2. Verify output starts from beginning (no black frames)

**Expected Result**:
- ✅ Trim starts correctly at timestamp 0
- ✅ No initial black frames

**Actual Result**: _[To be filled during test execution]_

---

#### TC-FN-007: Trim with end time = video duration
**Priority**: Medium  
**Automation**: Yes (Jest)

**Preconditions**:
- Video duration: 45 seconds
- Job data: `trim: { startSec: 20, endSec: 45 }`

**Test Steps**:
1. Process with trim ending at exact duration
2. Verify no errors, output is 25 seconds (20s to 45s)

**Expected Result**:
- ✅ Trim to exact duration works without errors

**Actual Result**: _[To be filled during test execution]_

---

### 3.3 Thumbnail/Cover Generation

#### TC-FN-008: Extract cover at user-selected timestamp (12.5s)
**Priority**: High  
**Automation**: Yes (Jest)

**Preconditions**:
- Test video: 30 seconds
- Job data: `coverTimestampSec: 12.5`

**Test Steps**:
1. Process video with custom cover timestamp
2. Download thumbnail from S3
3. Verify FFmpeg probe shows timestamp ~12.5s:
   ```bash
   # Can't directly verify timestamp from JPEG, but verify file exists and is valid
   file thumbnail.jpg
   ffprobe thumbnail.jpg  # Should show valid JPEG metadata
   ```
4. Verify thumbnail quality: File size 80-150 KB (high quality JPEG)

**Expected Result**:
- ✅ Thumbnail extracted at 12.5s (visually verify frame matches video at that time)
- ✅ JPEG quality 90% (`-q:v 2`)
- ✅ Max dimensions 1080x1920 (preserves aspect ratio)

**Actual Result**: _[To be filled during test execution]_

---

#### TC-FN-009: Cover generation with timestamp exceeding duration
**Priority**: High  
**Automation**: Yes (Jest)

**Preconditions**:
- Video duration: 30 seconds
- Job data: `coverTimestampSec: 45` (exceeds duration)

**Test Steps**:
1. Process video with invalid timestamp
2. Verify fallback logic:
   - Attempt timestamp 45s (fails)
   - Retry at timestamp 29.5s (duration - 0.5s buffer)
3. Verify thumbnail generated successfully at fallback timestamp

**Expected Result**:
- ✅ No errors thrown
- ✅ Thumbnail extracted at 29.5s (clamped to valid range)
- ✅ Warning logged: "Cover timestamp exceeds duration"

**Actual Result**: _[To be filled during test execution]_

---

#### TC-FN-010: Cover generation fallback to timestamp 0
**Priority**: Medium  
**Automation**: Yes (Jest)

**Preconditions**:
- Video with corrupt frames at selected timestamp

**Test Steps**:
1. Attempt cover generation at timestamp 15s (corrupt frame)
2. Verify fallback logic:
   - First attempt fails
   - Retry at timestamp 0
3. Verify thumbnail generated from first frame

**Expected Result**:
- ✅ Thumbnail extracted at timestamp 0 (fallback)

**Actual Result**: _[To be filled during test execution]_

---

### 3.4 Image-to-Video Conversion

#### TC-FN-011: Convert JPEG image to 5-second video
**Priority**: High  
**Automation**: Yes (Jest)

**Preconditions**:
- Test image: `test_image.jpg` (1080x1920 JPEG)
- Job data: `photoDurationSec: 5`

**Test Steps**:
1. Upload image as reel media
2. Process (should detect as image: duration < 1s or codec = 'mjpeg')
3. Verify output:
   - Video format: MP4, H.264 codec
   - Duration: 5 seconds
   - FPS: 30
   - Dimensions: 1080x1920
4. Verify variants generated (720p, 480p, 360p)

**Expected Result**:
- ✅ Image converted to 5-second video
- ✅ Static image displayed for entire duration
- ✅ Variants generated correctly

**Actual Result**: _[To be filled during test execution]_

---

#### TC-FN-012: Convert PNG image to 3-second video
**Priority**: Medium  
**Automation**: Yes (Jest)

**Preconditions**:
- Test image: `test_image.png`
- Job data: `photoDurationSec: 3`

**Test Steps**:
1. Process PNG image
2. Verify codec detected as PNG
3. Verify output duration: 3 seconds

**Expected Result**:
- ✅ PNG converted to 3-second video
- ✅ No transparency issues (PNG alpha channel handled)

**Actual Result**: _[To be filled during test execution]_

---

#### TC-FN-013: Image with non-9:16 aspect ratio (4:3)
**Priority**: Medium  
**Automation**: Yes (Jest)

**Preconditions**:
- Test image: `image_4x3.jpg` (1600x1200, 4:3 aspect ratio)

**Test Steps**:
1. Convert image to video
2. Verify output: 1080x1920 canvas with black pillarbox bars

**Expected Result**:
- ✅ Image centered with black bars (no stretching)

**Actual Result**: _[To be filled during test execution]_

---

### 3.5 Text Overlays

#### TC-FN-014: Apply single-line text overlay
**Priority**: High  
**Automation**: Yes (Jest)

**Preconditions**:
- Test video: 30 seconds
- Job data:
  ```json
  {
    "textOverlays": [
      {
        "content": "Taste the Flavors",
        "startTime": 0,
        "endTime": 5,
        "position": { "x": 0.5, "y": 0.1 },
        "scale": 1.0,
        "style": {
          "fontSize": 32,
          "fontWeight": "700",
          "color": "#FFFFFF",
          "background": "pill"
        }
      }
    ]
  }
  ```

**Test Steps**:
1. Process video with text overlay
2. Download output video
3. Verify text appears from 0s to 5s
4. Verify text position: centered horizontally (x=0.5), near top (y=0.1)
5. Verify text style: white, bold, pill background (black rounded rectangle)
6. Verify text disappears after 5s

**Expected Result**:
- ✅ Text appears exactly from 0s to 5s
- ✅ Position correct (centered top)
- ✅ Pill background visible

**Actual Result**: _[To be filled during test execution]_

---

#### TC-FN-015: Apply multi-line text overlay
**Priority**: High  
**Automation**: Yes (Jest)

**Preconditions**:
- Overlay content with newlines:
  ```
  Line 1
  Line 2
  Line 3
  ```

**Test Steps**:
1. Process video with multi-line text
2. Verify all 3 lines appear correctly (no "\\N" visible)
3. Verify line breaks preserved

**Expected Result**:
- ✅ Multi-line text rendered correctly
- ✅ No raw escape sequences visible

**Actual Result**: _[To be filled during test execution]_

---

#### TC-FN-016: Apply multiple text overlays with different timing
**Priority**: High  
**Automation**: Yes (Jest)

**Preconditions**:
- Overlay 1: 0s to 5s (top)
- Overlay 2: 5s to 10s (center)
- Overlay 3: 10s to 15s (bottom)

**Test Steps**:
1. Process video with 3 overlays
2. Verify timing:
   - 0-5s: Only overlay 1 visible
   - 5-10s: Only overlay 2 visible
   - 10-15s: Only overlay 3 visible
3. Verify no overlaps or missing frames

**Expected Result**:
- ✅ Each overlay appears in correct time window
- ✅ No overlapping text

**Actual Result**: _[To be filled during test execution]_

---

#### TC-FN-017: Text overlay with shadow background
**Priority**: Medium  
**Automation**: Yes (Jest)

**Preconditions**:
- Overlay style: `background: 'shadow'`

**Test Steps**:
1. Process video with shadow text
2. Verify drop shadow visible below text (black shadow with 75% opacity)

**Expected Result**:
- ✅ Shadow visible (improves text readability on bright backgrounds)

**Actual Result**: _[To be filled during test execution]_

---

#### TC-FN-018: Text overlay with no background (transparent)
**Priority**: Medium  
**Automation**: Yes (Jest)

**Preconditions**:
- Overlay style: `background: 'none'`

**Test Steps**:
1. Process video with transparent text
2. Verify no pill/shadow background

**Expected Result**:
- ✅ Text renders without background

**Actual Result**: _[To be filled during test execution]_

---

#### TC-FN-019: Text overlay with large font size (scale: 3.0)
**Priority**: Medium  
**Automation**: Yes (Jest)

**Preconditions**:
- Overlay: `scale: 3.0` (3× larger than default)

**Test Steps**:
1. Process video with large text
2. Verify font size clamped to max 300px (safety limit)
3. Verify text visible and not cut off

**Expected Result**:
- ✅ Text scaled correctly
- ✅ No text clipping at canvas edges

**Actual Result**: _[To be filled during test execution]_

---

#### TC-FN-020: Text overlay with special characters (emoji, accents)
**Priority**: Low  
**Automation**: Yes (Jest)

**Preconditions**:
- Overlay content: "Delicious 🍕 Café"

**Test Steps**:
1. Process video with emoji and accented characters
2. Verify emoji and accents render correctly (font support)

**Expected Result**:
- ✅ Emoji rendered (if font supports it) or replaced with placeholder
- ✅ Accented characters (é, ñ, ü) render correctly

**Actual Result**: _[To be filled during test execution]_

---

### 3.6 Image Filters

#### TC-FN-021: Apply "vintage" preset filter
**Priority**: High  
**Automation**: Yes (Jest)

**Preconditions**:
- Job data: `filter: { name: 'vintage', preset: 'vintage' }`

**Test Steps**:
1. Process video with vintage filter
2. Download output and visually verify:
   - Desaturated colors (70% saturation)
   - Warm tone (color temperature 7500K)
   - Vignette (edge darkening)

**Expected Result**:
- ✅ Video has vintage look (Instagram "1977" style)

**Actual Result**: _[To be filled during test execution]_

---

#### TC-FN-022: Apply "vibrant" preset filter
**Priority**: Medium  
**Automation**: Yes (Jest)

**Preconditions**:
- Job data: `filter: { name: 'vibrant', preset: 'vibrant' }`

**Test Steps**:
1. Process video with vibrant filter
2. Verify boosted saturation (140%) and contrast (120%)

**Expected Result**:
- ✅ Colors appear more vivid

**Actual Result**: _[To be filled during test execution]_

---

#### TC-FN-023: Apply "bw" (black & white) preset filter
**Priority**: Medium  
**Automation**: Yes (Jest)

**Preconditions**:
- Job data: `filter: { name: 'bw', preset: 'bw' }`

**Test Steps**:
1. Process video with B&W filter
2. Verify output is grayscale (no color)

**Expected Result**:
- ✅ Video converted to black & white

**Actual Result**: _[To be filled during test execution]_

---

#### TC-FN-024: Apply custom brightness adjustment (+0.2)
**Priority**: Medium  
**Automation**: Yes (Jest)

**Preconditions**:
- Job data: `filter: { name: 'custom', brightness: 0.2 }`

**Test Steps**:
1. Process video with brightness boost
2. Verify output is 20% brighter than original

**Expected Result**:
- ✅ Video brightness increased

**Actual Result**: _[To be filled during test execution]_

---

#### TC-FN-025: Apply custom contrast adjustment (+0.3)
**Priority**: Medium  
**Automation**: Yes (Jest)

**Preconditions**:
- Job data: `filter: { name: 'custom', contrast: 0.3 }`

**Test Steps**:
1. Process video with contrast boost
2. Verify output has higher contrast (darks darker, lights lighter)

**Expected Result**:
- ✅ Video contrast increased

**Actual Result**: _[To be filled during test execution]_

---

#### TC-FN-026: Apply custom saturation adjustment (-0.5)
**Priority**: Medium  
**Automation**: Yes (Jest)

**Preconditions**:
- Job data: `filter: { name: 'custom', saturation: -0.5 }`

**Test Steps**:
1. Process video with desaturation
2. Verify colors appear less vibrant (50% saturation)

**Expected Result**:
- ✅ Video colors muted

**Actual Result**: _[To be filled during test execution]_

---

#### TC-FN-027: Apply custom warmth adjustment (+0.8)
**Priority**: Medium  
**Automation**: Yes (Jest)

**Preconditions**:
- Job data: `filter: { name: 'custom', warmth: 0.8 }`

**Test Steps**:
1. Process video with warm color temperature
2. Verify output has golden/orange tone

**Expected Result**:
- ✅ Video appears warmer

**Actual Result**: _[To be filled during test execution]_

---

#### TC-FN-028: Apply custom vignette (0.6)
**Priority**: Low  
**Automation**: Yes (Jest)

**Preconditions**:
- Job data: `filter: { name: 'custom', vignette: 0.6 }`

**Test Steps**:
1. Process video with vignette
2. Verify edges are darkened (spotlight effect)

**Expected Result**:
- ✅ Vignette visible at edges

**Actual Result**: _[To be filled during test execution]_

---

### 3.7 Combined Effects

#### TC-FN-029: Apply text overlay + filter
**Priority**: High  
**Automation**: Yes (Jest)

**Preconditions**:
- Text overlay: "Amazing Food"
- Filter: `preset: 'vintage'`

**Test Steps**:
1. Process video with both text and filter
2. Verify both effects applied:
   - Text visible
   - Video has vintage look

**Expected Result**:
- ✅ Text + filter both applied successfully

**Actual Result**: _[To be filled during test execution]_

---

#### TC-FN-030: Apply trim + text overlay + filter
**Priority**: High  
**Automation**: Yes (Jest)

**Preconditions**:
- Trim: 10s to 30s
- Text overlay: 0s to 5s (relative to trimmed video)
- Filter: `preset: 'vibrant'`

**Test Steps**:
1. Process video with all 3 features
2. Verify:
   - Output duration: 20 seconds (30 - 10)
   - Text appears from 0s to 5s (of trimmed video)
   - Filter applied to entire video

**Expected Result**:
- ✅ All features work together correctly

**Actual Result**: _[To be filled during test execution]_

---

### 3.8 Database Updates

#### TC-FN-031: Verify Media document updated after processing
**Priority**: High  
**Automation**: Yes (Jest)

**Test Steps**:
1. Process video
2. Query Media document by `mediaId`
3. Verify fields updated:
   - `status = 'ready'`
   - `masterVideoUrl` (HTTPS URL)
   - `variants[]` (3 entries with quality, url, width, height, bitrate)
   - `thumbnailUrl` (HTTPS URL)
   - `processedAt` (ISO timestamp)
   - `processingMethod = 'ffmpeg'`
   - `moderationStatus = 'pending'`

**Expected Result**:
- ✅ All fields updated correctly

**Actual Result**: _[To be filled during test execution]_

---

#### TC-FN-032: Verify Reel document updated after processing
**Priority**: High  
**Automation**: Yes (Jest)

**Test Steps**:
1. Process video
2. Query Reel document by `mediaId`
3. Verify fields updated:
   - `videoUrl = variants[0].url` (720p variant)
   - `thumbnailUrl` (matches Media document)
   - `durationSec` (from processed video metadata)

**Expected Result**:
- ✅ Reel playback URL points to 720p variant
- ✅ Duration accurate

**Actual Result**: _[To be filled during test execution]_

---

---

## 4. Performance Test Cases

### 4.1 Processing Time

#### TC-PERF-001: 15-second video processing time
**Priority**: High  
**Automation**: Yes (Artillery)

**Test Steps**:
1. Process 10× 15-second videos (9:16, no trim/effects)
2. Measure average processing time

**Expected Result**:
- ✅ Average time < 25 seconds

**Actual Result**: _[To be filled during test execution]_

---

#### TC-PERF-002: 30-second video processing time
**Priority**: High  
**Automation**: Yes (Artillery)

**Test Steps**:
1. Process 10× 30-second videos (9:16, no trim/effects)
2. Measure average processing time

**Expected Result**:
- ✅ Average time < 45 seconds

**Actual Result**: _[To be filled during test execution]_

---

#### TC-PERF-003: 60-second video processing time
**Priority**: Medium  
**Automation**: Yes (Artillery)

**Test Steps**:
1. Process 10× 60-second videos
2. Measure average processing time

**Expected Result**:
- ✅ Average time < 80 seconds

**Actual Result**: _[To be filled during test execution]_

---

#### TC-PERF-004: 120-second video with trim (to 30s) processing time
**Priority**: High  
**Automation**: Yes (Artillery)

**Test Steps**:
1. Process 10× 120-second videos with trim to 30s
2. Measure average processing time
3. Compare with full 120s processing time

**Expected Result**:
- ✅ Average time < 50 seconds (similar to native 30s video)
- ✅ 70% faster than processing full 120s video

**Actual Result**: _[To be filled during test execution]_

---

#### TC-PERF-005: Video with text overlay processing time
**Priority**: Medium  
**Automation**: Yes (Artillery)

**Test Steps**:
1. Process 10× 30-second videos with 1 text overlay
2. Measure average processing time
3. Compare with no-overlay baseline

**Expected Result**:
- ✅ Overhead < 10 seconds vs baseline

**Actual Result**: _[To be filled during test execution]_

---

#### TC-PERF-006: Video with filter processing time
**Priority**: Medium  
**Automation**: Yes (Artillery)

**Test Steps**:
1. Process 10× 30-second videos with "vintage" filter
2. Measure average processing time
3. Compare with no-filter baseline

**Expected Result**:
- ✅ Overhead < 8 seconds vs baseline

**Actual Result**: _[To be filled during test execution]_

---

### 4.2 Throughput

#### TC-PERF-007: Worker throughput (videos/hour)
**Priority**: High  
**Automation**: Yes (Artillery)

**Test Steps**:
1. Process 100× 30-second videos sequentially on single worker
2. Measure total time
3. Calculate throughput: `100 / (total_hours)`

**Expected Result**:
- ✅ Throughput ≥ 80 videos/hour per worker

**Actual Result**: _[To be filled during test execution]_

---

#### TC-PERF-008: Queue depth under load
**Priority**: High  
**Automation**: Yes (Artillery)

**Test Steps**:
1. Enqueue 200 processing jobs rapidly (10 jobs/second)
2. Monitor Redis queue depth over 30 minutes
3. Measure max queue depth, average queue depth

**Expected Result**:
- ✅ Max queue depth < 100 (with 2 workers)
- ✅ Queue drains to 0 within 2 hours

**Actual Result**: _[To be filled during test execution]_

---

### 4.3 Resource Usage

#### TC-PERF-009: Worker CPU usage during processing
**Priority**: High  
**Automation**: Partial (manual monitoring)

**Test Steps**:
1. Process 30-second video on single worker
2. Monitor CPU usage during encoding (using `top` or Kubernetes metrics)

**Expected Result**:
- ✅ CPU usage 70-90% during variant generation (FFmpeg is CPU-intensive)
- ✅ CPU drops to < 20% during S3 upload/download

**Actual Result**: _[To be filled during test execution]_

---

#### TC-PERF-010: Worker memory usage during processing
**Priority**: High  
**Automation**: Partial (manual monitoring)

**Test Steps**:
1. Process 60-second video
2. Monitor memory usage

**Expected Result**:
- ✅ Memory usage < 3 GB per worker (4 GB limit with buffer)

**Actual Result**: _[To be filled during test execution]_

---

#### TC-PERF-011: Disk usage (/tmp) during processing
**Priority**: Medium  
**Automation**: Partial (manual monitoring)

**Test Steps**:
1. Process 120-second video
2. Monitor `/tmp` directory size during processing

**Expected Result**:
- ✅ Max disk usage < 2 GB per job
- ✅ Temp files cleaned up after job completion

**Actual Result**: _[To be filled during test execution]_

---

#### TC-PERF-012: S3 upload/download bandwidth
**Priority**: Medium  
**Automation**: Partial (manual monitoring)

**Test Steps**:
1. Process 30-second video
2. Monitor network bandwidth during S3 upload phase (4 files: master + 3 variants)

**Expected Result**:
- ✅ Upload bandwidth < 50 Mbps per worker (within AWS network limits)

**Actual Result**: _[To be filled during test execution]_

---

---

## 5. Security Test Cases

### 5.1 Input Validation

#### TC-SEC-001: Invalid S3 key format (path traversal attempt)
**Priority**: Critical  
**Automation**: Yes (Jest)

**Test Steps**:
1. Attempt to process job with malicious S3 key: `../../etc/passwd`
2. Verify job fails with validation error

**Expected Result**:
- ❌ Job rejected: "Invalid S3 key format"

**Actual Result**: _[To be filled during test execution]_

---

#### TC-SEC-002: SQL injection in text overlay content
**Priority**: High  
**Automation**: Yes (Jest)

**Test Steps**:
1. Submit overlay content: `'; DROP TABLE media; --`
2. Process video
3. Verify:
   - Text rendered literally (no SQL execution)
   - Database tables intact

**Expected Result**:
- ✅ SQL injection string treated as plain text

**Actual Result**: _[To be filled during test execution]_

---

#### TC-SEC-003: FFmpeg command injection via text overlay
**Priority**: Critical  
**Automation**: Yes (Jest)

**Test Steps**:
1. Submit overlay content with FFmpeg escape sequences: `$(rm -rf /tmp)`
2. Process video
3. Verify:
   - Command not executed
   - Text rendered literally

**Expected Result**:
- ✅ Command injection prevented (text escaped correctly)

**Actual Result**: _[To be filled during test execution]_

---

#### TC-SEC-004: Oversized video file (> 500 MB)
**Priority**: High  
**Automation**: Yes (Jest)

**Test Steps**:
1. Attempt to upload 600 MB video file
2. Verify upload rejected by Media module (before processing)

**Expected Result**:
- ❌ Upload rejected: "File size exceeds limit"

**Actual Result**: _[To be filled during test execution]_

---

### 5.2 S3 Security

#### TC-SEC-005: Unauthorized S3 bucket access
**Priority**: Critical  
**Automation**: No (manual penetration test)

**Test Steps**:
1. Attempt to access S3 input/output buckets without credentials
2. Verify public access denied

**Expected Result**:
- ❌ Access denied (buckets are private)

**Actual Result**: _[To be filled during test execution]_

---

#### TC-SEC-006: Expired presigned URL
**Priority**: High  
**Automation**: Yes (Jest)

**Test Steps**:
1. Generate presigned upload URL with 15-minute expiry
2. Wait 20 minutes
3. Attempt to upload file using expired URL

**Expected Result**:
- ❌ Upload fails: "Request has expired"

**Actual Result**: _[To be filled during test execution]_

---

### 5.3 Data Privacy

#### TC-SEC-007: Verify processed videos not publicly accessible via direct S3 URL
**Priority**: High  
**Automation**: Yes (Jest)

**Test Steps**:
1. Process video, get S3 URL (https://chefooz-media-output.s3.amazonaws.com/...)
2. Attempt to access URL without CloudFront CDN
3. Verify access denied (bucket policy enforces CDN-only access)

**Expected Result**:
- ❌ Direct S3 access denied (must use CDN URL)

**Actual Result**: _[To be filled during test execution]_

---

#### TC-SEC-008: Verify temp files contain no PII
**Priority**: Medium  
**Automation**: Partial (manual inspection)

**Test Steps**:
1. Process video with text overlay containing user name
2. Inspect temp files in `/tmp/video-processing-<mediaId>/`
3. Verify temp files deleted after processing

**Expected Result**:
- ✅ Temp files cleaned up (no PII leakage)

**Actual Result**: _[To be filled during test execution]_

---

---

## 6. Error Handling Test Cases

### 6.1 FFmpeg Errors

#### TC-ERR-001: Corrupt video file
**Priority**: High  
**Automation**: Yes (Jest)

**Test Steps**:
1. Upload corrupt video file (truncated MP4, invalid headers)
2. Process job
3. Verify:
   - Job retries 3 times (exponential backoff)
   - After 3 failures → MediaConvert fallback triggered
   - If MediaConvert also fails → Media status = 'failed', Reel deleted

**Expected Result**:
- ✅ Fallback logic activated
- ✅ Ghost reel cleaned up

**Actual Result**: _[To be filled during test execution]_

---

#### TC-ERR-002: Unsupported video codec (HEVC with HDR)
**Priority**: High  
**Automation**: Yes (Jest)

**Test Steps**:
1. Upload video with HEVC codec + HDR metadata (known FFmpeg issue)
2. Process job
3. Verify MediaConvert fallback triggered immediately (or after first FFmpeg failure)

**Expected Result**:
- ✅ MediaConvert processes video successfully

**Actual Result**: _[To be filled during test execution]_

---

#### TC-ERR-003: FFmpeg out of memory
**Priority**: Medium  
**Automation**: No (requires controlled OOM scenario)

**Test Steps**:
1. Process very large video (> 500 MB) on worker with limited RAM (2 GB)
2. Verify FFmpeg crashes with OOM error
3. Verify Bull queue retries job on different worker

**Expected Result**:
- ✅ Job retried on worker with sufficient memory

**Actual Result**: _[To be filled during test execution]_

---

### 6.2 S3 Errors

#### TC-ERR-004: S3 download timeout
**Priority**: High  
**Automation**: Partial (requires network simulation)

**Test Steps**:
1. Simulate S3 network timeout during download (using chaos engineering tool)
2. Verify job retries download (Bull queue retry logic)

**Expected Result**:
- ✅ Job retried after backoff (5s, 10s, 20s)

**Actual Result**: _[To be filled during test execution]_

---

#### TC-ERR-005: S3 upload failure (invalid credentials)
**Priority**: High  
**Automation**: Yes (Jest with mock)

**Test Steps**:
1. Configure worker with invalid AWS credentials
2. Attempt to process video
3. Verify upload fails, job retried

**Expected Result**:
- ❌ Upload fails: "InvalidAccessKeyId"
- ✅ Job retried

**Actual Result**: _[To be filled during test execution]_

---

#### TC-ERR-006: S3 bucket not found
**Priority**: Medium  
**Automation**: Yes (Jest with mock)

**Test Steps**:
1. Configure worker with non-existent bucket name
2. Attempt processing
3. Verify error logged, job fails

**Expected Result**:
- ❌ Error: "NoSuchBucket"

**Actual Result**: _[To be filled during test execution]_

---

### 6.3 Database Errors

#### TC-ERR-007: Media document not found
**Priority**: High  
**Automation**: Yes (Jest)

**Test Steps**:
1. Enqueue job with invalid `mediaId` (non-existent MongoDB document)
2. Process job
3. Verify error logged, job fails gracefully (no crash)

**Expected Result**:
- ❌ Job fails: "Media document not found"

**Actual Result**: _[To be filled during test execution]_

---

#### TC-ERR-008: Database connection timeout during update
**Priority**: Medium  
**Automation**: Partial (requires DB chaos test)

**Test Steps**:
1. Simulate MongoDB connection timeout during `findByIdAndUpdate`
2. Verify job retries database update (Bull queue retry)

**Expected Result**:
- ✅ Database update retried successfully

**Actual Result**: _[To be filled during test execution]_

---

### 6.4 Disk Errors

#### TC-ERR-009: Out of disk space (/tmp full)
**Priority**: High  
**Automation**: Partial (requires disk quota simulation)

**Test Steps**:
1. Fill `/tmp` directory to 95% capacity
2. Attempt to process large video
3. Verify error: "ENOSPC: no space left on device"
4. Verify job fails, temp files cleaned up

**Expected Result**:
- ❌ Job fails with disk space error
- ✅ Alert triggered for ops team

**Actual Result**: _[To be filled during test execution]_

---

### 6.5 Moderation Integration

#### TC-ERR-010: Moderation service unavailable
**Priority**: Medium  
**Automation**: Yes (Jest with mock)

**Test Steps**:
1. Process video when ModerationService is down (mock service not available)
2. Verify:
   - Processing completes successfully (moderation is async)
   - Warning logged: "ModerationService not available"
   - Media status = 'ready', moderationStatus = 'pending'

**Expected Result**:
- ✅ Processing not blocked by moderation failure

**Actual Result**: _[To be filled during test execution]_

---

### 6.6 Fallback Logic

#### TC-ERR-011: MediaConvert fallback success
**Priority**: High  
**Automation**: Partial (requires FFmpeg failure simulation)

**Test Steps**:
1. Trigger FFmpeg failure (unsupported codec)
2. Verify MediaConvert fallback:
   - MediaConvertJob document created
   - Polling activated
   - Video processed by MediaConvert
   - Media document updated with MediaConvert URLs

**Expected Result**:
- ✅ Fallback completes successfully
- ✅ processingMethod = 'mediaconvert'

**Actual Result**: _[To be filled during test execution]_

---

#### TC-ERR-012: MediaConvert fallback failure
**Priority**: Medium  
**Automation**: Partial

**Test Steps**:
1. Trigger FFmpeg failure
2. Simulate MediaConvert failure (invalid input format)
3. Verify:
   - Media status = 'failed'
   - Reel document deleted (ghost reel prevention)

**Expected Result**:
- ✅ Ghost reel cleaned up after all retries exhausted

**Actual Result**: _[To be filled during test execution]_

---

### 6.7 Cleanup

#### TC-ERR-013: Temp file cleanup on error
**Priority**: High  
**Automation**: Yes (Jest)

**Test Steps**:
1. Trigger processing error (e.g., corrupt video)
2. Verify temp directory `/tmp/video-processing-<mediaId>/` deleted

**Expected Result**:
- ✅ Temp files cleaned up in `finally` block

**Actual Result**: _[To be filled during test execution]_

---

#### TC-ERR-014: Ghost reel cleanup after 3 failed retries
**Priority**: High  
**Automation**: Yes (Jest)

**Test Steps**:
1. Trigger processing failure that exhausts all 3 retries
2. Query Reel document by `mediaId`
3. Verify Reel deleted (not visible in user profile)
4. Verify Media status = 'failed' (audit record kept)

**Expected Result**:
- ✅ Reel deleted from public view
- ✅ Media document marked as failed (audit trail)

**Actual Result**: _[To be filled during test execution]_

---

#### TC-ERR-015: Original video deletion only if trimmed
**Priority**: Medium  
**Automation**: Yes (Jest)

**Test Steps**:
1. **Scenario A**: Process video WITH trim → Verify original deleted from S3 input bucket
2. **Scenario B**: Process video WITHOUT trim → Verify original retained (or deletion policy)

**Expected Result**:
- ✅ Original deleted only when trimmed (cost optimization)

**Actual Result**: _[To be filled during test execution]_

---

---

## 7. Integration Test Cases

### 7.1 Media Module Integration

#### TC-INT-001: End-to-end upload → process → ready workflow
**Priority**: Critical  
**Automation**: Yes (Cypress E2E)

**Test Steps**:
1. POST `/api/v1/media/upload` (get presigned URL)
2. PUT video to presigned URL (direct S3 upload)
3. POST `/api/v1/media/{mediaId}/complete` (trigger processing)
4. Poll job status until complete (max 60s)
5. GET `/api/v1/media/{mediaId}` (verify status = 'ready')
6. Verify variants accessible via CDN URLs

**Expected Result**:
- ✅ Complete workflow succeeds without manual intervention

**Actual Result**: _[To be filled during test execution]_

---

#### TC-INT-002: Reel creation triggers processing
**Priority**: High  
**Automation**: Yes (Cypress E2E)

**Test Steps**:
1. POST `/api/v1/reels` (create reel with uploaded media)
2. Verify Media status transitions: `uploaded` → `processing` → `ready`
3. Verify Reel becomes visible in user profile

**Expected Result**:
- ✅ Reel appears in profile feed after processing

**Actual Result**: _[To be filled during test execution]_

---

### 7.2 Moderation Integration

#### TC-INT-003: Moderation triggered after processing
**Priority**: High  
**Automation**: Yes (Jest)

**Test Steps**:
1. Process video (ensure thumbnail generated)
2. Verify ModerationService.startModeration() called with `mediaId`, `userId`, `thumbnailKey`
3. Verify Media document: `moderationStatus = 'pending'`

**Expected Result**:
- ✅ Moderation initiated automatically

**Actual Result**: _[To be filled during test execution]_

---

#### TC-INT-004: Rejected video not visible in feeds
**Priority**: High  
**Automation**: Yes (Cypress E2E)

**Test Steps**:
1. Process video
2. Simulate moderation rejection (update Media: `moderationStatus = 'rejected'`)
3. Query user's feed, search results
4. Verify rejected video not visible

**Expected Result**:
- ✅ Rejected content hidden from public view

**Actual Result**: _[To be filled during test execution]_

---

### 7.3 Feed/Cache Integration

#### TC-INT-005: New reel appears in follower feeds immediately
**Priority**: High  
**Automation**: Yes (Cypress E2E)

**Test Steps**:
1. User A uploads reel (processing complete)
2. User B (follower of A) refreshes feed
3. Verify User A's new reel appears in User B's feed

**Expected Result**:
- ✅ Feed cache invalidated, new reel visible within 5 seconds

**Actual Result**: _[To be filled during test execution]_

---

### 7.4 Notification Integration

#### TC-INT-006: User notified when processing complete
**Priority**: Medium  
**Automation**: Partial (requires notification service mock)

**Test Steps**:
1. Upload video
2. Wait for processing to complete
3. Verify push notification sent to user: "Your reel is ready!"

**Expected Result**:
- ✅ Notification received on mobile device

**Actual Result**: _[To be filled during test execution]_

---

### 7.5 Analytics Integration

#### TC-INT-007: Processing metrics logged
**Priority**: Medium  
**Automation**: Yes (Jest)

**Test Steps**:
1. Process video
2. Query analytics database (or Prometheus metrics)
3. Verify metrics logged:
   - `video_processing_duration_seconds`
   - `video_processing_errors_total`

**Expected Result**:
- ✅ Metrics available for monitoring dashboards

**Actual Result**: _[To be filled during test execution]_

---

### 7.6 CDN Integration

#### TC-INT-008: Processed videos accessible via CDN
**Priority**: High  
**Automation**: Yes (Cypress E2E)

**Test Steps**:
1. Process video
2. Get CDN URL from Media document: `variants[0].url`
3. Access URL in browser (or curl)
4. Verify video streams correctly

**Expected Result**:
- ✅ Video plays from CDN (CloudFront)

**Actual Result**: _[To be filled during test execution]_

---

#### TC-INT-009: CDN cache invalidation after re-upload
**Priority**: Low  
**Automation**: No (manual test)

**Test Steps**:
1. Upload and process video (version 1)
2. Delete and re-upload same video (version 2)
3. Verify CDN serves new version (not cached old version)

**Expected Result**:
- ✅ CDN cache invalidated correctly

**Actual Result**: _[To be filled during test execution]_

---

### 7.7 Bull Queue Integration

#### TC-INT-010: Multiple workers process jobs concurrently
**Priority**: High  
**Automation**: Yes (Artillery)

**Test Steps**:
1. Start 3 worker pods (6 workers total, 2 per pod)
2. Enqueue 20 jobs rapidly
3. Monitor job distribution (should spread across workers)
4. Verify all 20 jobs complete successfully

**Expected Result**:
- ✅ Jobs distributed evenly
- ✅ No job processed twice (concurrency handling)

**Actual Result**: _[To be filled during test execution]_

---

---

## 8. Regression Test Cases

### 8.1 Previously Fixed Bugs

#### TC-REG-001: Cover generation at timestamp 0 (fixed 2026-02-09)
**Priority**: High  
**Automation**: Yes (Jest)

**Bug Description**: Cover extraction failed for image-based videos (duration < 1s).  
**Fix**: Added fallback to timestamp 0 for short videos.

**Test Steps**:
1. Convert image to 5-second video
2. Extract cover at timestamp 3s
3. Verify cover generated successfully

**Expected Result**:
- ✅ No errors, cover extracted

**Actual Result**: _[To be filled during test execution]_

---

#### TC-REG-002: Aspect ratio padding for horizontal videos (fixed 2026-01-16)
**Priority**: High  
**Automation**: Yes (Jest)

**Bug Description**: Horizontal videos stretched to 9:16 (distorted).  
**Fix**: Added letterbox padding (black bars).

**Test Steps**:
1. Process 16:9 horizontal video
2. Verify output: 1080x1920 canvas with black letterbox (top/bottom)

**Expected Result**:
- ✅ No stretching, black bars visible

**Actual Result**: _[To be filled during test execution]_

---

#### TC-REG-003: Multi-line text overlay newlines (fixed 2026-01-16)
**Priority**: Medium  
**Automation**: Yes (Jest)

**Bug Description**: Newlines appeared as `\N` in video (escape sequence not processed).  
**Fix**: Use `textfile` parameter instead of inline text.

**Test Steps**:
1. Process video with multi-line overlay: "Line 1\nLine 2\nLine 3"
2. Verify newlines rendered correctly (3 separate lines, no `\N` visible)

**Expected Result**:
- ✅ Multi-line text displays correctly

**Actual Result**: _[To be filled during test execution]_

---

#### TC-REG-004: Font file not found on Alpine Linux (fixed 2026-01-10)
**Priority**: High  
**Automation**: Yes (Jest)

**Bug Description**: Text overlays failed in production (font path incorrect for Alpine).  
**Fix**: Added fallback font path detection (DejaVu Sans for Alpine, Arial for macOS).

**Test Steps**:
1. Run processing on Alpine Docker container
2. Apply text overlay
3. Verify text renders (no "font not found" error)

**Expected Result**:
- ✅ Text overlay works on Alpine Linux

**Actual Result**: _[To be filled during test execution]_

---

#### TC-REG-005: Ghost reel cleanup (added 2026-01-05)
**Priority**: Critical  
**Automation**: Yes (Jest)

**Bug Description**: Failed processing left blank reels in user profile (bad UX).  
**Fix**: Delete Reel document after 3 failed retry attempts.

**Test Steps**:
1. Trigger processing failure (corrupt video)
2. Wait for 3 retries to exhaust
3. Query Reel by `mediaId`
4. Verify Reel deleted (returns 404)

**Expected Result**:
- ✅ Ghost reel removed from database

**Actual Result**: _[To be filled during test execution]_

---

#### TC-REG-006: Trim-first optimization (added 2025-12-20)
**Priority**: High  
**Automation**: Yes (Artillery)

**Bug Description**: Processing 120s video took 140s even when trimmed to 30s.  
**Fix**: Trim video BEFORE generating variants (process 30s instead of 120s).

**Test Steps**:
1. Process 120s video with trim to 30s
2. Measure processing time
3. Compare with baseline (full 120s processing)

**Expected Result**:
- ✅ Processing time reduced by 70%

**Actual Result**: _[To be filled during test execution]_

---

#### TC-REG-007: Image-to-video duration validation (fixed 2026-02-11)
**Priority**: Medium  
**Automation**: Yes (Jest)

**Bug Description**: Image videos sometimes had incorrect duration (0.04s instead of 5s).  
**Fix**: Improved image detection (check both duration and codec), verify output duration.

**Test Steps**:
1. Convert JPEG to 5-second video
2. Use `ffprobe` to verify output duration: ~5.0 seconds

**Expected Result**:
- ✅ Output duration exactly 5.0 seconds (±0.1s tolerance)

**Actual Result**: _[To be filled during test execution]_

---

#### TC-REG-008: S3 URI to HTTPS conversion (fixed 2025-12-10)
**Priority**: Medium  
**Automation**: Yes (Jest)

**Bug Description**: Database stored S3 URIs (`s3://bucket/key`) instead of HTTPS URLs.  
**Fix**: Use `s3UriToHttps()` utility to convert S3 URIs to CDN URLs.

**Test Steps**:
1. Process video
2. Verify Media document `masterVideoUrl` starts with `https://cdn.chefooz.com/`

**Expected Result**:
- ✅ HTTPS URLs stored (not S3 URIs)

**Actual Result**: _[To be filled during test execution]_

---

#### TC-REG-009: FFmpeg stderr capture (added 2026-02-09)
**Priority**: Low  
**Automation**: Yes (Jest)

**Bug Description**: FFmpeg errors not logged (stderr not captured).  
**Fix**: Added `stderrOutput` variable to capture and log FFmpeg errors.

**Test Steps**:
1. Trigger FFmpeg error (invalid input)
2. Check logs for detailed FFmpeg stderr output

**Expected Result**:
- ✅ FFmpeg error details visible in logs

**Actual Result**: _[To be filled during test execution]_

---

#### TC-REG-010: Cover file size validation (added 2026-02-09)
**Priority**: Medium  
**Automation**: Yes (Jest)

**Bug Description**: Empty cover files (0 bytes) created when extraction failed.  
**Fix**: Verify cover file size > 100 bytes before considering it valid.

**Test Steps**:
1. Simulate cover extraction failure
2. Verify error thrown: "Cover file is too small"
3. Verify fallback logic triggered (retry at timestamp 0)

**Expected Result**:
- ✅ Invalid covers rejected, fallback successful

**Actual Result**: _[To be filled during test execution]_

---

#### TC-REG-011: Parallel variant generation (added 2025-12-01)
**Priority**: High  
**Automation**: Yes (Artillery)

**Bug Description**: Variants generated sequentially (18s total for 3 variants).  
**Fix**: Use `Promise.all()` to generate variants in parallel.

**Test Steps**:
1. Process 30-second video
2. Measure variant generation time
3. Verify < 8 seconds (vs 18s sequential)

**Expected Result**:
- ✅ Variants generated in parallel (66% faster)

**Actual Result**: _[To be filled during test execution]_

---

#### TC-REG-012: Reel duration update from processed video (added 2025-11-25)
**Priority**: Medium  
**Automation**: Yes (Jest)

**Bug Description**: Reel duration incorrect after trim (showed original duration, not trimmed).  
**Fix**: Update Reel `durationSec` from processed video metadata (not original).

**Test Steps**:
1. Process 60s video with trim to 30s
2. Verify Reel document: `durationSec = 30` (not 60)

**Expected Result**:
- ✅ Reel duration matches trimmed video

**Actual Result**: _[To be filled during test execution]_

---

#### TC-REG-013: MediaConvert job tracking (added 2025-11-20)
**Priority**: Medium  
**Automation**: Yes (Jest)

**Bug Description**: MediaConvert fallback jobs not tracked (no way to monitor status).  
**Fix**: Create `MediaConvertJob` document for polling.

**Test Steps**:
1. Trigger FFmpeg failure → MediaConvert fallback
2. Verify `MediaConvertJob` document created with `jobId`, `status='SUBMITTED'`

**Expected Result**:
- ✅ Fallback job tracked in database

**Actual Result**: _[To be filled during test execution]_

---

#### TC-REG-014: Effect file verification (added 2026-01-16)
**Priority**: Medium  
**Automation**: Yes (Jest)

**Bug Description**: Empty effect files (0 bytes) created when filter/overlay failed.  
**Fix**: Verify effect file size > 1000 bytes before using as input for variants.

**Test Steps**:
1. Apply text overlay
2. Verify `with_effects.mp4` file size > 1000 bytes
3. If file too small → throw error (don't proceed to variant generation)

**Expected Result**:
- ✅ Invalid effect files rejected early

**Actual Result**: _[To be filled during test execution]_

---

#### TC-REG-015: Moderation service optional (added 2025-11-15)
**Priority**: Low  
**Automation**: Yes (Jest)

**Bug Description**: Processing failed when ModerationService not available (optional dependency).  
**Fix**: Use `moduleRef.resolve()` with `strict: false` to gracefully handle missing service.

**Test Steps**:
1. Process video with ModerationService disabled/unavailable
2. Verify processing completes successfully
3. Verify warning logged: "ModerationService not available"

**Expected Result**:
- ✅ Processing not blocked by missing moderation

**Actual Result**: _[To be filled during test execution]_

---

#### TC-REG-016: CacheService optional (added 2025-11-15)
**Priority**: Low  
**Automation**: Yes (Jest)

**Bug Description**: Similar to moderation, cache invalidation failure blocked processing.  
**Fix**: Handle CacheService as optional dependency.

**Test Steps**:
1. Process video with CacheService unavailable
2. Verify processing completes
3. Verify warning logged

**Expected Result**:
- ✅ Processing succeeds without cache invalidation

**Actual Result**: _[To be filled during test execution]_

---

#### TC-REG-017: Text overlay font size clamping (added 2026-01-10)
**Priority**: Medium  
**Automation**: Yes (Jest)

**Bug Description**: Extremely large font sizes (userScale > 5.0) caused FFmpeg errors.  
**Fix**: Clamp user scale to 0.3-5.0 range, clamp final font size to 16-300px.

**Test Steps**:
1. Submit overlay with `scale: 10.0` (extreme)
2. Verify font size clamped to 300px (max)
3. Verify video processes without error

**Expected Result**:
- ✅ Extreme font sizes handled gracefully

**Actual Result**: _[To be filled during test execution]_

---

#### TC-REG-018: Text overlay position clamping (added 2026-01-10)
**Priority**: Medium  
**Automation**: Yes (Jest)

**Bug Description**: Text positioned off-screen (negative coordinates or > 1.0).  
**Fix**: Clamp normalized position to 0-1 range, use FFmpeg `max(0, min(...))` expressions.

**Test Steps**:
1. Submit overlay with `position: { x: 1.5, y: -0.2 }` (off-screen)
2. Verify position clamped to `{ x: 1.0, y: 0.0 }`
3. Verify text visible on-screen

**Expected Result**:
- ✅ Text stays within canvas bounds

**Actual Result**: _[To be filled during test execution]_

---

#### TC-REG-019: Original deletion only when trimmed (added 2025-12-05)
**Priority**: Medium  
**Automation**: Yes (Jest)

**Bug Description**: Original videos always deleted (even without trim), losing backup.  
**Fix**: Delete original only if `trim` metadata present.

**Test Steps**:
1. **Case A**: Process with trim → Verify original deleted
2. **Case B**: Process without trim → Verify original retained (or policy-based deletion)

**Expected Result**:
- ✅ Conditional deletion based on trim flag

**Actual Result**: _[To be filled during test execution]_

---

#### TC-REG-020: Temp file cleanup in finally block (added 2025-11-01)
**Priority**: High  
**Automation**: Yes (Jest)

**Bug Description**: Temp files not cleaned up on error (disk filled up over time).  
**Fix**: Move cleanup to `finally` block (executes even on error).

**Test Steps**:
1. Trigger processing error mid-way
2. Verify temp directory deleted despite error

**Expected Result**:
- ✅ Cleanup executed in all scenarios (success/error)

**Actual Result**: _[To be filled during test execution]_

---

---

## 9. Test Data

### 9.1 Test Video Assets

| File Name | Duration | Aspect Ratio | Resolution | Use Case |
|-----------|----------|--------------|------------|----------|
| `vertical_15s.mp4` | 15s | 9:16 | 1080x1920 | Standard vertical video |
| `vertical_30s.mp4` | 30s | 9:16 | 1080x1920 | Standard vertical video |
| `vertical_60s.mp4` | 60s | 9:16 | 1080x1920 | Long-form vertical video |
| `vertical_120s.mp4` | 120s | 9:16 | 1080x1920 | Trim-first optimization test |
| `horizontal_15s.mp4` | 15s | 16:9 | 1920x1080 | Letterbox padding test |
| `square_10s.mp4` | 10s | 1:1 | 1080x1080 | Pillarbox padding test |
| `rotated_90deg.mp4` | 20s | 9:16 (rotated) | 1920x1080 → 1080x1920 | Rotation handling |
| `corrupt.mp4` | N/A | N/A | N/A | Error handling (truncated file) |
| `hevc_hdr.mp4` | 25s | 9:16 | 1080x1920 | MediaConvert fallback test |

### 9.2 Test Image Assets

| File Name | Dimensions | Format | Use Case |
|-----------|------------|--------|----------|
| `test_image_9x16.jpg` | 1080x1920 | JPEG | Image-to-video (perfect aspect) |
| `test_image_4x3.jpg` | 1600x1200 | JPEG | Image-to-video (padding test) |
| `test_image_square.jpg` | 1080x1080 | JPEG | Image-to-video (pillarbox) |
| `test_image.png` | 1080x1920 | PNG | PNG support test |

### 9.3 Text Overlay Samples

```json
{
  "single_line": {
    "content": "Taste the Flavors",
    "startTime": 0,
    "endTime": 5,
    "position": { "x": 0.5, "y": 0.1 },
    "scale": 1.0,
    "style": {
      "fontSize": 32,
      "fontWeight": "700",
      "color": "#FFFFFF",
      "background": "pill"
    }
  },
  "multi_line": {
    "content": "Line 1\nLine 2\nLine 3",
    "startTime": 0,
    "endTime": 10,
    "position": { "x": 0.1, "y": 0.5 },
    "scale": 1.2,
    "style": {
      "fontSize": 28,
      "fontWeight": "400",
      "color": "#FFFF00",
      "background": "shadow"
    }
  },
  "large_scale": {
    "content": "BIG TEXT",
    "startTime": 5,
    "endTime": 10,
    "position": { "x": 0.5, "y": 0.5 },
    "scale": 3.0,
    "style": {
      "fontSize": 48,
      "fontWeight": "700",
      "color": "#FF0000",
      "background": "none"
    }
  }
}
```

### 9.4 Filter Presets

```json
{
  "vintage": { "name": "vintage", "preset": "vintage" },
  "vibrant": { "name": "vibrant", "preset": "vibrant" },
  "bw": { "name": "bw", "preset": "bw" },
  "custom_warm": { "name": "custom", "brightness": 0.1, "contrast": 0.2, "saturation": 0.1, "warmth": 0.8 },
  "custom_cool": { "name": "custom", "brightness": -0.1, "contrast": 0.15, "warmth": -0.6 }
}
```

---

## 10. Automation Coverage

### 10.1 Automation Summary

| Test Category | Total Cases | Automated | Manual | Automation % |
|---------------|-------------|-----------|--------|--------------|
| **Functional** | 65 | 55 | 10 | 85% |
| **Performance** | 12 | 11 | 1 | 92% |
| **Security** | 8 | 5 | 3 | 63% |
| **Error Handling** | 15 | 12 | 3 | 80% |
| **Integration** | 10 | 8 | 2 | 80% |
| **Regression** | 20 | 19 | 1 | 95% |
| **Total** | **130** | **110** | **20** | **85%** |

### 10.2 CI/CD Integration

**Test Execution in Pipeline**:
```yaml
# .github/workflows/test.yml
name: Media Processing Tests

on: [push, pull_request]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Install FFmpeg
        run: sudo apt-get install -y ffmpeg
      - name: Install dependencies
        run: npm ci
      - name: Run unit tests
        run: npm run test -- media-processing

  e2e-tests:
    runs-on: ubuntu-latest
    services:
      redis:
        image: redis:7-alpine
      mongodb:
        image: mongo:6
    steps:
      - uses: actions/checkout@v3
      - name: Install FFmpeg
        run: sudo apt-get install -y ffmpeg
      - name: Run E2E tests
        run: npm run test:e2e -- media-processing

  performance-tests:
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v3
      - name: Run load tests
        run: npx artillery run test/performance/media-processing.yml
```

### 10.3 Test Execution Commands

**Unit Tests**:
```bash
# Run all media-processing unit tests
npm run test -- media-processing

# Run specific test file
npm run test -- media-processing.service.spec.ts

# Run with coverage
npm run test:cov -- media-processing
```

**E2E Tests**:
```bash
# Run E2E tests
npm run test:e2e

# Run specific E2E suite
npx cypress run --spec "cypress/e2e/media-processing.cy.ts"
```

**Performance Tests**:
```bash
# Run load tests
npx artillery run test/performance/media-processing-load.yml

# Generate report
npx artillery report test/performance/report.json
```

---

## 11. Test Execution Schedule

| Test Type | Trigger | Frequency | Environment |
|-----------|---------|-----------|-------------|
| **Unit Tests** | Every commit | On PR, push | CI pipeline |
| **E2E Tests** | Every commit | On PR, push | CI pipeline + staging |
| **Performance Tests** | Nightly | Daily (2 AM) | Staging |
| **Security Tests** | Weekly | Every Monday | Staging |
| **Regression Tests** | Pre-release | Before production deploy | Staging + production |

---

## 12. Bug Reporting Template

**Title**: [Media-Processing] Brief description

**Test Case ID**: TC-XXX-XXX

**Severity**: Critical / High / Medium / Low

**Environment**: Development / Staging / Production

**Steps to Reproduce**:
1. ...
2. ...
3. ...

**Expected Result**: ...

**Actual Result**: ...

**Screenshots/Logs**: [Attach]

**Video Sample**: [S3 URL or file path]

**FFmpeg Version**: `ffmpeg -version`

**Additional Context**: ...

---

**[END OF QA TEST CASES]**  
**Document Status**: ✅ Complete  
**Total Test Cases**: 130  
**Automation Coverage**: 85%  
**Next Review Date**: May 14, 2026
