# Media Upload Module - QA Test Cases

> **Module**: `apps/chefooz-apis/src/modules/media`  
> **Test Coverage Target**: 85%+ (Unit + Integration + E2E)  
> **Last Updated**: February 14, 2026  
> **Test Environment**: UAT (chefooz-media-uat S3 bucket)

---

## 📋 Test Scope Overview

### **Functional Testing**
- ✅ Reel upload initialization (review, menu showcase, promotional)
- ✅ S3 presigned URL generation and security
- ✅ Upload completion and processing triggers
- ✅ Status polling and state transitions
- ✅ Type-specific quota enforcement (review, menu, promo)
- ✅ Order and menu item linking validation
- ✅ User tagging and notification dispatch
- ✅ Idempotency and retry behavior
- ✅ Video processing pipeline (FFmpeg + MediaConvert)
- ✅ Image upload for chat/profiles

### **Non-Functional Testing**
- ✅ Performance (upload speed, processing latency)
- ✅ Security (ownership, presigned URL expiry, MIME type validation)
- ✅ Scalability (concurrent uploads, large file handling)
- ✅ Error handling (S3 failures, quota violations, invalid inputs)

---

## 🧪 Test Case Structure

Each test case follows this format:
```
TC-XXX: [Test Name]
├── Priority: P0 (Critical) / P1 (High) / P2 (Medium) / P3 (Low)
├── Preconditions: Setup required before test
├── Test Steps: Numbered actions to perform
├── Expected Result: What should happen
├── Automation: Yes/No + Tool (Jest/Cypress/Postman)
└── Tags: [api, quota, security, performance]
```

---

## 🎯 Functional Test Cases

### **Category 1: Upload Initialization** (8 Test Cases)

#### **TC-001: Initialize Review Reel Upload (Happy Path)**
- **Priority**: P0 (Critical)
- **Preconditions**:
  - User has JWT token
  - User has at least 1 completed order (DELIVERED status)
  - User has not exhausted review reel quota (used < completed orders)
- **Test Steps**:
  1. Generate unique `uploadId` using UUID
  2. POST `/api/v1/media/upload-reel` with payload:
     ```json
     {
       "uploadId": "550e8400-e29b-41d4-a716-446655440000",
       "fileName": "review-reel.mp4",
       "mimeType": "video/mp4",
       "durationSec": 30,
       "sizeBytes": 15000000,
       "caption": "Amazing food from @ChefRamesh!",
       "hashtags": ["foodie", "delicious"],
       "orderId": "<valid-delivered-order-id>"
     }
     ```
  3. Verify response status 201
- **Expected Result**:
  ```json
  {
    "success": true,
    "message": "Reel upload initialized",
    "data": {
      "mediaId": "<mongodb-objectid>",
      "reelId": "<mongodb-objectid>",
      "uploadUrl": "https://stub-upload.chefooz.com/...",
      "expiresAt": "<timestamp>"
    }
  }
  ```
  - Media document created in MongoDB with status `UPLOADING`
  - Reel document created with `reelPurpose='USER_REVIEW'`
  - `linkedOrderId` set to provided order ID
- **Automation**: ✅ Yes (Jest unit test + Postman E2E)
- **Tags**: `[api, upload, quota, review-reel]`

---

#### **TC-002: Initialize Menu Showcase Reel Upload**
- **Priority**: P0 (Critical)
- **Preconditions**:
  - User has chef role
  - User has at least 1 active menu item
  - Menu item does not have existing showcase reel (1 reel per item limit)
- **Test Steps**:
  1. POST `/api/v1/media/upload-reel` with `menuItemId` field:
     ```json
     {
       "uploadId": "<uuid>",
       "fileName": "butter-chicken-showcase.mp4",
       "mimeType": "video/mp4",
       "durationSec": 45,
       "sizeBytes": 20000000,
       "caption": "My signature Butter Chicken recipe 🍗",
       "menuItemId": "<valid-active-menu-item-id>"
     }
     ```
  2. Verify response 201
- **Expected Result**:
  - Reel created with `reelPurpose='MENU_SHOWCASE'`
  - `linkedMenu` object populated with:
    - `chefId`: Menu item owner
    - `menuItemIds`: Array containing provided menu item ID
    - `estimatedPaise`: Menu item price × 100
    - `previewImage`: Menu item image URL
- **Automation**: ✅ Yes (Jest)
- **Tags**: `[api, upload, quota, menu-showcase]`

---

#### **TC-003: Initialize Promotional Reel Upload (User Role)**
- **Priority**: P1 (High)
- **Preconditions**:
  - User has not uploaded 10 promotional reels in last 7 days (production limit)
  - No `orderId` or `menuItemId` provided (identifies as promotional)
- **Test Steps**:
  1. POST `/api/v1/media/upload-reel` without linking fields:
     ```json
     {
       "uploadId": "<uuid>",
       "fileName": "promo-reel.mp4",
       "mimeType": "video/mp4",
       "durationSec": 60,
       "sizeBytes": 25000000,
       "caption": "Check out my cooking vlog! 🔥"
     }
     ```
  2. Verify response 201
- **Expected Result**:
  - Reel created with `reelPurpose='PROMOTIONAL'`
  - `contentSource='user'` (uses user promotional quota pool)
  - No `linkedOrderId` or `linkedMenu`
- **Automation**: ✅ Yes (Jest)
- **Tags**: `[api, upload, quota, promotional]`

---

#### **TC-004: Reject Upload - Review Reel Quota Exceeded**
- **Priority**: P0 (Critical)
- **Preconditions**:
  - User has 5 completed orders
  - User has already uploaded 5 review reels (quota full)
- **Test Steps**:
  1. Attempt POST `/api/v1/media/upload-reel` with `orderId`
  2. Verify response 403 (Forbidden)
- **Expected Result**:
  ```json
  {
    "success": false,
    "message": "Review reel quota exceeded. You've used 5 of 5 available slots (1 per delivered order).",
    "errorCode": "REVIEW_REEL_QUOTA_EXCEEDED",
    "data": {
      "used": 5,
      "max": 5,
      "completedOrders": 5
    }
  }
  ```
  - No Media or Reel document created
- **Automation**: ✅ Yes (Jest)
- **Tags**: `[api, quota, error-handling]`

---

#### **TC-005: Reject Upload - Menu Showcase Quota Exceeded**
- **Priority**: P1 (High)
- **Preconditions**:
  - Menu item already has 1 showcase reel (limit = 1 per item)
- **Test Steps**:
  1. Attempt POST `/api/v1/media/upload-reel` with same `menuItemId`
  2. Verify response 403
- **Expected Result**:
  ```json
  {
    "success": false,
    "message": "Menu showcase quota exceeded. You've already uploaded 1 reel for this item (max 1 per menu item).",
    "errorCode": "MENU_SHOWCASE_QUOTA_EXCEEDED",
    "data": {
      "menuItemId": "<id>",
      "existingReels": 1,
      "maxPerItem": 1
    }
  }
  ```
- **Automation**: ✅ Yes (Jest)
- **Tags**: `[api, quota, error-handling, menu-showcase]`

---

#### **TC-006: Reject Upload - Promotional Quota Exceeded (Weekly Limit)**
- **Priority**: P1 (High)
- **Preconditions**:
  - User has uploaded 10 promotional reels in last 7 days (production limit)
- **Test Steps**:
  1. Attempt 11th promotional reel upload
  2. Verify response 403
- **Expected Result**:
  ```json
  {
    "success": false,
    "message": "Promotional reel quota exceeded. You've used 10 of 10 weekly slots. Quota resets after 7 days.",
    "errorCode": "PROMOTIONAL_REEL_QUOTA_EXCEEDED_USER",
    "data": {
      "used": 10,
      "max": 10,
      "resetDate": "<timestamp>"
    }
  }
  ```
- **Automation**: ✅ Yes (Jest)
- **Tags**: `[api, quota, error-handling, promotional]`

---

#### **TC-007: Reject Upload - File Size Exceeds Tier Limit**
- **Priority**: P0 (Critical)
- **Preconditions**: User is BRONZE tier (max 200MB)
- **Test Steps**:
  1. Attempt upload with `sizeBytes: 250000000` (250MB)
  2. Verify response 403
- **Expected Result**:
  ```json
  {
    "success": false,
    "message": "File exceeds tier limits. Reduce file size or duration.",
    "errorCode": "FILE_SIZE_EXCEEDS_TIER_LIMIT"
  }
  ```
- **Automation**: ✅ Yes (Jest)
- **Tags**: `[api, validation, file-limits]`

---

#### **TC-008: Reject Upload - Duration Exceeds Limit**
- **Priority**: P1 (High)
- **Preconditions**: None
- **Test Steps**:
  1. Attempt upload with `durationSec: 150` (150 seconds, max is 120)
  2. Verify response 400 (Bad Request)
- **Expected Result**:
  ```json
  {
    "success": false,
    "message": "Validation failed",
    "errors": [
      {
        "property": "durationSec",
        "constraints": {
          "max": "durationSec must not be greater than 120"
        }
      }
    ]
  }
  ```
- **Automation**: ✅ Yes (Jest)
- **Tags**: `[api, validation, dto]`

---

### **Category 2: Idempotency** (4 Test Cases)

#### **TC-009: Idempotent Upload - Return Existing Media (UPLOADING State)**
- **Priority**: P0 (Critical)
- **Preconditions**:
  - User has initiated upload with `uploadId='test-uuid'`
  - Media status is `UPLOADING` (client hasn't completed upload yet)
- **Test Steps**:
  1. POST `/api/v1/media/upload-reel` with SAME `uploadId='test-uuid'`
  2. Verify response 201 (not 409 conflict)
- **Expected Result**:
  ```json
  {
    "success": true,
    "message": "Reel upload initialized",
    "data": {
      "mediaId": "<same-media-id-as-before>",
      "reelId": "<same-reel-id-as-before>",
      "uploadUrl": "...",
      "expiresAt": "...",
      "resumeState": {
        "status": "uploading",
        "canResume": true,
        "message": "Upload in progress. You can continue uploading the file."
      }
    }
  }
  ```
  - No duplicate Media/Reel documents created
  - Client knows to resume S3 upload
- **Automation**: ✅ Yes (Jest)
- **Tags**: `[api, idempotency, retry-logic]`

---

#### **TC-010: Idempotent Upload - Return Existing Media (PROCESSING State)**
- **Priority**: P1 (High)
- **Preconditions**:
  - User has completed upload and media is `PROCESSING`
- **Test Steps**:
  1. Retry POST `/api/v1/media/upload-reel` with same `uploadId`
  2. Verify response 201
- **Expected Result**:
  ```json
  {
    "resumeState": {
      "status": "processing",
      "canResume": false,
      "message": "Video is being processed. Please wait for completion."
    }
  }
  ```
  - Client knows to poll status endpoint
- **Automation**: ✅ Yes (Jest)
- **Tags**: `[api, idempotency, processing]`

---

#### **TC-011: Idempotent Upload - Return Existing Media (READY State)**
- **Priority**: P1 (High)
- **Preconditions**: Media is `READY` (processing complete)
- **Test Steps**:
  1. Retry POST `/api/v1/media/upload-reel` with same `uploadId`
  2. Verify response 201
- **Expected Result**:
  ```json
  {
    "resumeState": {
      "status": "ready",
      "canResume": false,
      "message": "Upload already completed successfully."
    }
  }
  ```
  - Client navigates to reel view page
- **Automation**: ✅ Yes (Jest)
- **Tags**: `[api, idempotency, ready-state]`

---

#### **TC-012: Idempotent Upload - Reject FAILED State Retry**
- **Priority**: P2 (Medium)
- **Preconditions**: Media is `FAILED` (processing error)
- **Test Steps**:
  1. Retry POST `/api/v1/media/upload-reel` with same `uploadId`
  2. Verify response 400 (Bad Request)
- **Expected Result**:
  ```json
  {
    "success": false,
    "message": "Previous upload failed. Please start a new upload.",
    "errorCode": "PREVIOUS_UPLOAD_FAILED"
  }
  ```
  - Client must generate NEW `uploadId` and retry
- **Automation**: ✅ Yes (Jest)
- **Tags**: `[api, idempotency, error-handling]`

---

### **Category 3: Presigned URL Generation** (6 Test Cases)

#### **TC-013: Generate Presigned S3 URL (Video Upload)**
- **Priority**: P0 (Critical)
- **Preconditions**:
  - User has initialized upload (has `mediaId`)
  - Media status is `UPLOADING`
- **Test Steps**:
  1. POST `/api/v1/media/presigned-upload-url` with:
     ```json
     {
       "mediaId": "<valid-media-id>",
       "fileType": "video/mp4"
     }
     ```
  2. Verify response 200
- **Expected Result**:
  ```json
  {
    "success": true,
    "message": "Presigned URL generated",
    "data": {
      "uploadUrl": "https://chefooz-media-uat.s3.ap-south-1.amazonaws.com/uploads/<mediaId>/original.mp4?X-Amz-Algorithm=...",
      "s3Key": "uploads/<mediaId>/original.mp4",
      "expiresIn": 900
    }
  }
  ```
  - URL contains AWS signature parameters
  - URL expires in 15 minutes (900 seconds)
- **Automation**: ✅ Yes (Jest + Postman)
- **Tags**: `[api, s3, presigned-url, security]`

---

#### **TC-014: Generate Presigned URL for Thumbnail Upload**
- **Priority**: P1 (High)
- **Preconditions**: Same as TC-013
- **Test Steps**:
  1. POST `/api/v1/media/presigned-upload-url` with:
     ```json
     {
       "mediaId": "<valid-media-id>",
       "fileType": "image/jpeg",
       "isThumbnail": true
     }
     ```
  2. Verify response 200
- **Expected Result**:
  - `s3Key`: `uploads/<mediaId>/thumbnail.jpg` (not `original.jpg`)
  - Media document updated with `thumbnailUrl` immediately
- **Automation**: ✅ Yes (Jest)
- **Tags**: `[api, s3, thumbnail, image-upload]`

---

#### **TC-015: Reject Presigned URL - Media Not Found**
- **Priority**: P1 (High)
- **Preconditions**: None
- **Test Steps**:
  1. POST `/api/v1/media/presigned-upload-url` with invalid `mediaId='invalid-id'`
  2. Verify response 404
- **Expected Result**:
  ```json
  {
    "success": false,
    "message": "Media document not found",
    "errorCode": "MEDIA_NOT_FOUND"
  }
  ```
- **Automation**: ✅ Yes (Jest)
- **Tags**: `[api, error-handling, validation]`

---

#### **TC-016: Reject Presigned URL - Unauthorized Access**
- **Priority**: P0 (Critical - Security)
- **Preconditions**:
  - User A has created media document
  - User B has valid JWT token
- **Test Steps**:
  1. User B attempts POST `/api/v1/media/presigned-upload-url` with User A's `mediaId`
  2. Verify response 403 (Forbidden)
- **Expected Result**:
  ```json
  {
    "success": false,
    "message": "Not authorized to upload to this media",
    "errorCode": "MEDIA_ACCESS_DENIED"
  }
  ```
  - Prevents cross-user upload attacks
- **Automation**: ✅ Yes (Jest)
- **Tags**: `[api, security, ownership, authorization]`

---

#### **TC-017: Reject Presigned URL - Invalid Media State**
- **Priority**: P1 (High)
- **Preconditions**: Media status is `READY` (upload already complete)
- **Test Steps**:
  1. Attempt POST `/api/v1/media/presigned-upload-url`
  2. Verify response 400
- **Expected Result**:
  ```json
  {
    "success": false,
    "message": "Cannot upload to media in ready state",
    "errorCode": "INVALID_MEDIA_STATE"
  }
  ```
- **Automation**: ✅ Yes (Jest)
- **Tags**: `[api, state-machine, validation]`

---

#### **TC-018: Presigned URL Expiry - Verify Time Limit**
- **Priority**: P2 (Medium - Security)
- **Preconditions**: Generated presigned URL
- **Test Steps**:
  1. Generate presigned URL
  2. Wait 16 minutes (past 15-minute expiry)
  3. Attempt PUT to expired URL
  4. Verify S3 returns 403 (Access Denied)
- **Expected Result**:
  - S3 rejects upload with `<Code>AccessDenied</Code>` XML error
  - Client must request new presigned URL
- **Automation**: ⚠️ Partial (Manual for timing, automated for URL parsing)
- **Tags**: `[security, s3, expiry, time-based]`

---

### **Category 4: Upload Completion** (5 Test Cases)

#### **TC-019: Complete Upload - Trigger FFmpeg Processing**
- **Priority**: P0 (Critical)
- **Preconditions**:
  - User has uploaded video to S3 using presigned URL
  - File exists in upload bucket at `uploads/<mediaId>/original.mp4`
- **Test Steps**:
  1. POST `/api/v1/media/complete-upload` with:
     ```json
     {
       "mediaId": "<valid-media-id>",
       "s3Key": "uploads/<mediaId>/original.mp4"
     }
     ```
  2. Verify response 200
  3. Check Bull queue for enqueued job
- **Expected Result**:
  ```json
  {
    "success": true,
    "message": "Upload complete, processing started",
    "data": {
      "success": true,
      "mediaId": "<mediaId>",
      "inputKey": "uploads/<mediaId>/original.mp4"
    }
  }
  ```
  - Media status updated to `PROCESSING`
  - FFmpeg job added to `video-processing` queue with:
    - `mediaId`, `s3Key`, `trim`, `coverTimestampSec`, `textOverlays`, `filter`
  - File copied from upload bucket → input bucket
- **Automation**: ✅ Yes (Jest + Bull queue mock)
- **Tags**: `[api, upload-completion, ffmpeg, queue]`

---

#### **TC-020: Complete Upload with Trim Metadata**
- **Priority**: P1 (High)
- **Preconditions**: Same as TC-019
- **Test Steps**:
  1. POST `/api/v1/media/complete-upload` with trim:
     ```json
     {
       "mediaId": "<mediaId>",
       "s3Key": "uploads/<mediaId>/original.mp4",
       "trim": {
         "startSec": 15.5,
         "endSec": 45.0
       }
     }
     ```
  2. Verify FFmpeg job includes trim metadata
- **Expected Result**:
  - FFmpeg worker receives `trim: { startSec: 15.5, endSec: 45.0 }`
  - Final video duration = 45.0 - 15.5 = 29.5 seconds
- **Automation**: ✅ Yes (Jest)
- **Tags**: `[api, ffmpeg, trim, video-editing]`

---

#### **TC-021: Reject Complete Upload - File Not Found in S3**
- **Priority**: P0 (Critical)
- **Preconditions**:
  - User has NOT uploaded file to S3 (skipped presigned URL step)
- **Test Steps**:
  1. Attempt POST `/api/v1/media/complete-upload`
  2. Verify response 400
- **Expected Result**:
  ```json
  {
    "success": false,
    "message": "Failed to complete upload. File may not exist or copy failed.",
    "errorCode": "UPLOAD_COMPLETION_FAILED"
  }
  ```
  - Media status updated to `FAILED`
  - No FFmpeg job enqueued
- **Automation**: ✅ Yes (Jest with S3 mock)
- **Tags**: `[api, error-handling, s3, validation]`

---

#### **TC-022: Reject Complete Upload - Unauthorized Access**
- **Priority**: P0 (Critical - Security)
- **Preconditions**: User B attempts to complete User A's upload
- **Test Steps**:
  1. User B POSTs `/api/v1/media/complete-upload` with User A's `mediaId`
  2. Verify response 403
- **Expected Result**:
  ```json
  {
    "success": false,
    "message": "Not authorized to complete upload for this media",
    "errorCode": "MEDIA_ACCESS_DENIED"
  }
  ```
- **Automation**: ✅ Yes (Jest)
- **Tags**: `[api, security, ownership]`

---

#### **TC-023: Complete Upload - S3 Copy Failure Handling**
- **Priority**: P2 (Medium)
- **Preconditions**: S3 CopyObjectCommand fails (network error, IAM permissions, etc.)
- **Test Steps**:
  1. Mock S3 client to throw error on copy
  2. Attempt POST `/api/v1/media/complete-upload`
  3. Verify response 400
- **Expected Result**:
  - Media marked as `FAILED` with `errorMessage`
  - Error logged to monitoring system
  - User receives `UPLOAD_COMPLETION_FAILED` error
- **Automation**: ✅ Yes (Jest with S3 mock)
- **Tags**: `[api, error-handling, s3, resilience]`

---

### **Category 5: Status Polling** (4 Test Cases)

#### **TC-024: Poll Status - UPLOADING State**
- **Priority**: P1 (High)
- **Preconditions**: Media initialized but file not yet uploaded
- **Test Steps**:
  1. GET `/api/v1/media/<mediaId>/status`
  2. Verify response 200
- **Expected Result**:
  ```json
  {
    "success": true,
    "message": "Media status retrieved",
    "data": {
      "status": "uploading",
      "variants": [],
      "thumbnailUrl": null
    }
  }
  ```
  - Client continues showing upload progress bar
- **Automation**: ✅ Yes (Jest)
- **Tags**: `[api, status, polling]`

---

#### **TC-025: Poll Status - PROCESSING State**
- **Priority**: P0 (Critical)
- **Preconditions**: Upload complete, FFmpeg job running
- **Test Steps**:
  1. GET `/api/v1/media/<mediaId>/status`
  2. Verify response 200
- **Expected Result**:
  ```json
  {
    "data": {
      "status": "processing",
      "variants": [],
      "thumbnailUrl": null
    }
  }
  ```
  - Client shows "Processing video..." spinner
  - Client continues polling every 5 seconds
- **Automation**: ✅ Yes (Jest)
- **Tags**: `[api, status, processing]`

---

#### **TC-026: Poll Status - READY State with Variants**
- **Priority**: P0 (Critical)
- **Preconditions**: FFmpeg processing complete
- **Test Steps**:
  1. GET `/api/v1/media/<mediaId>/status`
  2. Verify response 200
- **Expected Result**:
  ```json
  {
    "data": {
      "status": "ready",
      "variants": [
        {
          "quality": "720p",
          "url": "https://chefooz-media-output.s3.amazonaws.com/converted/<mediaId>/720p.mp4",
          "width": 1280,
          "height": 720,
          "bitrate": 2500000
        },
        {
          "quality": "480p",
          "url": "https://chefooz-media-output.s3.amazonaws.com/converted/<mediaId>/480p.mp4",
          "width": 854,
          "height": 480,
          "bitrate": 1200000
        }
      ],
      "thumbnailUrl": "https://chefooz-media-output.s3.amazonaws.com/converted/<mediaId>/thumbnail.jpg"
    }
  }
  ```
  - Client stops polling
  - Client navigates to reel view page with `videoUrl = variants[0].url`
- **Automation**: ✅ Yes (Jest + E2E)
- **Tags**: `[api, status, ready, variants]`

---

#### **TC-027: Poll Status - FAILED State**
- **Priority**: P1 (High)
- **Preconditions**: FFmpeg processing failed (corrupt video, unsupported codec, etc.)
- **Test Steps**:
  1. GET `/api/v1/media/<mediaId>/status`
  2. Verify response 200
- **Expected Result**:
  ```json
  {
    "data": {
      "status": "failed",
      "variants": [],
      "thumbnailUrl": null
    }
  }
  ```
  - Client stops polling
  - Client shows error: "Video processing failed. Please try again."
- **Automation**: ✅ Yes (Jest)
- **Tags**: `[api, status, error-handling]`

---

### **Category 6: Order & Menu Linking** (6 Test Cases)

#### **TC-028: Validate Order Linking - Order Belongs to User**
- **Priority**: P0 (Critical)
- **Preconditions**:
  - User has delivered order with ID `order-123`
- **Test Steps**:
  1. POST `/api/v1/media/upload-reel` with `orderId='order-123'`
  2. Verify backend validates order ownership
  3. Verify response 201
- **Expected Result**:
  - Reel document has `linkedOrderId='order-123'`
  - `creatorOrderValue` snapshotted (order total in paise)
- **Automation**: ✅ Yes (Jest)
- **Tags**: `[api, order-linking, validation]`

---

#### **TC-029: Reject Order Linking - Order Not Found**
- **Priority**: P1 (High)
- **Preconditions**: None
- **Test Steps**:
  1. POST `/api/v1/media/upload-reel` with `orderId='non-existent-id'`
  2. Verify response 404
- **Expected Result**:
  ```json
  {
    "success": false,
    "message": "Order not found",
    "errorCode": "ORDER_NOT_FOUND"
  }
  ```
- **Automation**: ✅ Yes (Jest)
- **Tags**: `[api, order-linking, error-handling]`

---

#### **TC-030: Reject Order Linking - Order Does Not Belong to User**
- **Priority**: P0 (Critical - Security)
- **Preconditions**:
  - User A has order `order-123`
  - User B attempts to link to it
- **Test Steps**:
  1. User B POSTs `/api/v1/media/upload-reel` with `orderId='order-123'`
  2. Verify response 403
- **Expected Result**:
  ```json
  {
    "success": false,
    "message": "Can only link your own orders",
    "errorCode": "ORDER_NOT_YOURS"
  }
  ```
- **Automation**: ✅ Yes (Jest)
- **Tags**: `[api, order-linking, security, ownership]`

---

#### **TC-031: Reject Order Linking - Order Not Delivered**
- **Priority**: P1 (High)
- **Preconditions**: User has order with `deliveryStatus='IN_PROGRESS'`
- **Test Steps**:
  1. POST `/api/v1/media/upload-reel` with in-progress order ID
  2. Verify response 403
- **Expected Result**:
  ```json
  {
    "success": false,
    "message": "Can only link delivered orders",
    "errorCode": "ORDER_NOT_DELIVERED"
  }
  ```
  - Users can't review orders before receiving them
- **Automation**: ✅ Yes (Jest)
- **Tags**: `[api, order-linking, business-rules]`

---

#### **TC-032: Validate Menu Linking - Menu Item Active and Exists**
- **Priority**: P0 (Critical)
- **Preconditions**: Chef has active menu item with ID `item-123`
- **Test Steps**:
  1. POST `/api/v1/media/upload-reel` with `menuItemId='item-123'`
  2. Verify response 201
- **Expected Result**:
  - Reel has `linkedMenu` object with:
    - `chefId`: Menu item owner
    - `menuItemIds`: `['item-123']`
    - `estimatedPaise`: Item price × 100
    - `previewImage`: Item image URL
- **Automation**: ✅ Yes (Jest)
- **Tags**: `[api, menu-linking, validation]`

---

#### **TC-033: Reject Menu Linking - Menu Item Inactive**
- **Priority**: P1 (High)
- **Preconditions**: Chef has menu item with `isActive=false`
- **Test Steps**:
  1. POST `/api/v1/media/upload-reel` with inactive menu item ID
  2. Verify response 403
- **Expected Result**:
  ```json
  {
    "success": false,
    "message": "Cannot link inactive menu items",
    "errorCode": "MENU_ITEM_INACTIVE"
  }
  ```
- **Automation**: ✅ Yes (Jest)
- **Tags**: `[api, menu-linking, validation]`

---

### **Category 7: User Tagging & Notifications** (5 Test Cases)

#### **TC-034: Send Unified Tag Notifications - Caption Mentions**
- **Priority**: P1 (High)
- **Preconditions**:
  - Reel caption contains `@user123` and `@user456`
  - `taggedUserIds` array includes valid user IDs
- **Test Steps**:
  1. POST `/api/v1/media/upload-reel` with `taggedUserIds=['<user123-id>', '<user456-id>']`
  2. Verify reel creation succeeds
  3. Check notification queue for 2 tag notifications
- **Expected Result**:
  - 2 push notifications sent:
    - To `user123`: "@actorUsername tagged you in a reel"
    - To `user456`: "@actorUsername tagged you in a reel"
  - Notifications contain deep link to reel
- **Automation**: ✅ Yes (Jest with notification mock)
- **Tags**: `[api, notifications, user-tagging]`

---

#### **TC-035: Send Unified Tag Notifications - Positioned Tags**
- **Priority**: P1 (High)
- **Preconditions**: Reel has positioned tags (Instagram tap-to-tag)
- **Test Steps**:
  1. POST `/api/v1/media/upload-reel` with:
     ```json
     {
       "positionedTags": [
         { "userId": "<user123-id>", "x": 0.5, "y": 0.3 },
         { "userId": "<user456-id>", "x": 0.7, "y": 0.6 }
       ]
     }
     ```
  2. Verify 2 notifications sent
- **Expected Result**:
  - Both `user123` and `user456` notified
  - Notification metadata includes `tagContext: { hasPositionedTag: true }`
- **Automation**: ✅ Yes (Jest)
- **Tags**: `[api, notifications, positioned-tags]`

---

#### **TC-036: Deduplicate Tag Notifications - Same User in Caption & Positioned**
- **Priority**: P0 (Critical - Prevent Spam)
- **Preconditions**:
  - Reel caption mentions `@user123`
  - Positioned tag also tags `user123` at coordinate `(0.5, 0.3)`
- **Test Steps**:
  1. POST `/api/v1/media/upload-reel` with both tag types for same user
  2. Verify only 1 notification sent to `user123`
- **Expected Result**:
  - Backend deduplicates `taggedUserIds` and `positionedTags` arrays
  - `user123` receives ONLY 1 notification (not 2)
  - Notification metadata: `tagContext: { hasCaptionMention: true, hasPositionedTag: true }`
- **Automation**: ✅ Yes (Jest)
- **Tags**: `[api, notifications, deduplication, user-experience]`

---

#### **TC-037: Filter Self-Tags - User Cannot Tag Themselves**
- **Priority**: P1 (High - Business Rule)
- **Preconditions**: User attempts to tag themselves
- **Test Steps**:
  1. POST `/api/v1/media/upload-reel` with `taggedUserIds` containing own user ID
  2. Verify no notification sent to self
- **Expected Result**:
  - Backend filters out actor's own user ID from tag list
  - No self-notification sent
  - Other tagged users still notified normally
- **Automation**: ✅ Yes (Jest)
- **Tags**: `[api, notifications, business-rules]`

---

#### **TC-038: Limit Tags - Max 20 Total Tags**
- **Priority**: P2 (Medium)
- **Preconditions**: User attempts to tag 25 users (15 caption + 10 positioned)
- **Test Steps**:
  1. POST `/api/v1/media/upload-reel` with 25 total tags
  2. Verify only first 20 tags processed
- **Expected Result**:
  - Backend slices tag arrays to 20 max
  - Only 20 users notified
  - Reel document stores max 20 tags
- **Automation**: ✅ Yes (Jest)
- **Tags**: `[api, notifications, rate-limiting]`

---

### **Category 8: Quota API** (3 Test Cases)

#### **TC-039: Get Upload Quotas - All Reel Types**
- **Priority**: P1 (High)
- **Preconditions**:
  - User has 5 completed orders, uploaded 2 review reels
  - User uploaded 3 promotional reels this week (user role)
  - Chef uploaded 5 promotional reels this week (chef role)
- **Test Steps**:
  1. GET `/api/v1/media/upload-quotas`
  2. Verify response 200
- **Expected Result**:
  ```json
  {
    "success": true,
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
        "user": { "used": 3, "max": 10, "remaining": 7 },
        "chef": { "used": 5, "max": 10, "remaining": 5 }
      }
    }
  }
  ```
  - Client uses this to show quota UI: "You have 3 review reels left"
- **Automation**: ✅ Yes (Jest + E2E)
- **Tags**: `[api, quota, user-experience]`

---

#### **TC-040: Quota Calculation - Rolling Weekly Window**
- **Priority**: P1 (High)
- **Preconditions**:
  - User uploaded 10 promotional reels:
    - 5 reels uploaded 8 days ago (outside 7-day window)
    - 5 reels uploaded 3 days ago (inside 7-day window)
- **Test Steps**:
  1. GET `/api/v1/media/upload-quotas`
  2. Verify `promotional.user.used = 5` (not 10)
- **Expected Result**:
  - Old reels (8+ days ago) not counted against quota
  - User has 5 slots remaining (10 max - 5 used)
- **Automation**: ✅ Yes (Jest with date mocking)
- **Tags**: `[api, quota, time-based, rolling-window]`

---

#### **TC-041: Quota Calculation - Separate User vs Chef Pools**
- **Priority**: P0 (Critical - Business Logic)
- **Preconditions**:
  - Same person has uploaded:
    - 10 promotional reels as USER (user quota full)
    - 5 promotional reels as CHEF (chef quota not full)
- **Test Steps**:
  1. GET `/api/v1/media/upload-quotas`
  2. Verify both quotas tracked separately
- **Expected Result**:
  ```json
  {
    "promotional": {
      "user": { "used": 10, "max": 10, "remaining": 0 },
      "chef": { "used": 5, "max": 10, "remaining": 5 }
    }
  }
  ```
  - User can still upload 5 chef promotional reels
  - User cannot upload more user promotional reels
- **Automation**: ✅ Yes (Jest)
- **Tags**: `[api, quota, dual-role, business-rules]`

---

### **Category 9: Image Upload (Chat/Profiles)** (3 Test Cases)

#### **TC-042: Initialize Chat Image Upload**
- **Priority**: P1 (High)
- **Preconditions**: User wants to upload image in chat
- **Test Steps**:
  1. POST `/api/v1/media/chat/upload-init` with:
     ```json
     {
       "mimeType": "image/jpeg"
     }
     ```
  2. Verify response 201
- **Expected Result**:
  ```json
  {
    "success": true,
    "message": "Upload initialized",
    "data": {
      "uploadUrl": "https://chefooz-media-output.s3.amazonaws.com/chat/<uuid>/original.jpg?...",
      "imageKey": "chat/<uuid>/original.jpg",
      "cdnUrl": "https://cdn.chefooz.com/chat/<uuid>/original.jpg"
    }
  }
  ```
  - Client uploads directly to S3 using presigned URL
  - Client uses `cdnUrl` in chat message
- **Automation**: ✅ Yes (Jest)
- **Tags**: `[api, image-upload, chat]`

---

#### **TC-043: Reject Image Upload - Invalid MIME Type**
- **Priority**: P2 (Medium)
- **Preconditions**: None
- **Test Steps**:
  1. POST `/api/v1/media/chat/upload-init` with `mimeType='video/mp4'`
  2. Verify response 400
- **Expected Result**:
  - Error: "Only image MIME types allowed for chat uploads"
- **Automation**: ✅ Yes (Jest)
- **Tags**: `[api, validation, image-upload]`

---

#### **TC-044: Image Upload - Max File Size Limit**
- **Priority**: P2 (Medium)
- **Preconditions**: Client has 15MB image file (limit = 10MB)
- **Test Steps**:
  1. Attempt to upload 15MB image to presigned URL
  2. Verify S3 rejects upload
- **Expected Result**:
  - S3 returns 400 (Bad Request) or 403 (Forbidden)
  - Client shows error: "Image too large (max 10MB)"
- **Automation**: ⚠️ Partial (S3 integration test)
- **Tags**: `[image-upload, validation, file-size]`

---

## 🔐 Security Test Cases

### **Category 10: Authorization & Ownership** (5 Test Cases)

#### **TC-045: Cross-User Upload Prevention**
- **Priority**: P0 (Critical - Security)
- **Preconditions**:
  - User A has `mediaId='media-123'`
  - User B has valid JWT token
- **Test Steps**:
  1. User B attempts:
     - POST `/api/v1/media/presigned-upload-url` with `mediaId='media-123'`
     - POST `/api/v1/media/complete-upload` with `mediaId='media-123'`
     - GET `/api/v1/media/media-123/status`
  2. Verify all requests return 403
- **Expected Result**:
  - All endpoints verify `media.userId === req.user.id`
  - No cross-user data leakage
- **Automation**: ✅ Yes (Jest)
- **Tags**: `[security, authorization, ownership, critical]`

---

#### **TC-046: JWT Token Validation - Missing Token**
- **Priority**: P0 (Critical - Security)
- **Preconditions**: None
- **Test Steps**:
  1. Send POST `/api/v1/media/upload-reel` WITHOUT `Authorization` header
  2. Verify response 401 (Unauthorized)
- **Expected Result**:
  ```json
  {
    "statusCode": 401,
    "message": "Unauthorized"
  }
  ```
  - No media document created
- **Automation**: ✅ Yes (Jest)
- **Tags**: `[security, authentication, jwt]`

---

#### **TC-047: JWT Token Validation - Expired Token**
- **Priority**: P0 (Critical - Security)
- **Preconditions**: User has JWT token expired 1 day ago
- **Test Steps**:
  1. Send POST `/api/v1/media/upload-reel` with expired token
  2. Verify response 401
- **Expected Result**:
  - Error: "Token expired"
  - User redirected to login
- **Automation**: ✅ Yes (Jest with JWT mock)
- **Tags**: `[security, jwt, expiry]`

---

#### **TC-048: MIME Type Enforcement - S3 Upload**
- **Priority**: P1 (High - Security)
- **Preconditions**: Presigned URL generated for `video/mp4`
- **Test Steps**:
  1. Attempt to upload `image/png` file to video presigned URL
  2. Verify S3 rejects upload
- **Expected Result**:
  - S3 returns 403 (Signature mismatch - Content-Type header doesn't match)
  - Prevents file type spoofing attacks
- **Automation**: ⚠️ Partial (S3 integration test)
- **Tags**: `[security, mime-type, s3]`

---

#### **TC-049: Rate Limiting - Upload Endpoint**
- **Priority**: P2 (Medium - Security)
- **Preconditions**: Rate limiter configured (e.g., 100 requests/min per user)
- **Test Steps**:
  1. Send 110 POST `/api/v1/media/upload-reel` requests in 60 seconds
  2. Verify requests 101-110 return 429 (Too Many Requests)
- **Expected Result**:
  ```json
  {
    "statusCode": 429,
    "message": "Too many requests, please try again later"
  }
  ```
  - Prevents upload spam / DDoS
- **Automation**: ⚠️ Partial (requires rate limiter middleware)
- **Tags**: `[security, rate-limiting, ddos-protection]`

---

## ⚡ Performance Test Cases

### **Category 11: Load & Stress Testing** (4 Test Cases)

#### **TC-050: Concurrent Uploads - 100 Users**
- **Priority**: P1 (High)
- **Preconditions**: Load test environment with 100 test users
- **Test Steps**:
  1. Simulate 100 concurrent POST `/api/v1/media/upload-reel` requests
  2. Measure response time (p50, p95, p99)
  3. Verify all requests succeed (200/201 status)
- **Expected Result**:
  - **p50 (median)**: <300ms
  - **p95**: <800ms
  - **p99**: <1500ms
  - Success rate: 100% (no 5xx errors)
  - MongoDB connection pool handles load
- **Automation**: ⚠️ Partial (Artillery/k6 load test)
- **Tags**: `[performance, load-test, concurrency]`

---

#### **TC-051: Large File Upload - 200MB Video**
- **Priority**: P1 (High)
- **Preconditions**: User has 200MB video file (at tier limit)
- **Test Steps**:
  1. Initialize upload → Get presigned URL
  2. Upload 200MB file to S3 (measure time)
  3. Complete upload → Verify processing succeeds
- **Expected Result**:
  - Upload time: <5 minutes (depends on network, target: ~1MB/s)
  - S3 direct upload works without backend timeout
  - FFmpeg processing completes within 3 minutes
- **Automation**: ⚠️ Manual (requires real 200MB file + S3 bucket)
- **Tags**: `[performance, large-file, upload-speed]`

---

#### **TC-052: FFmpeg Processing Latency - Various Video Lengths**
- **Priority**: P1 (High)
- **Preconditions**: Videos of 15s, 30s, 60s, 120s uploaded
- **Test Steps**:
  1. Complete upload for each video length
  2. Measure time from `status=PROCESSING` → `status=READY`
  3. Calculate avg processing time per second of video
- **Expected Result**:
  | Video Length | Processing Time (Target) | Ratio |
  |--------------|-------------------------|-------|
  | 15s | ~10s | 0.67x |
  | 30s | ~20s | 0.67x |
  | 60s | ~40s | 0.67x |
  | 120s | ~80s | 0.67x |
  - Processing should scale linearly with video length
- **Automation**: ⚠️ Partial (requires FFmpeg worker + test videos)
- **Tags**: `[performance, ffmpeg, processing-time]`

---

#### **TC-053: Presigned URL Generation - Throughput**
- **Priority**: P2 (Medium)
- **Preconditions**: Load test environment
- **Test Steps**:
  1. Simulate 500 POST `/api/v1/media/presigned-upload-url` requests/second
  2. Measure response time and success rate
- **Expected Result**:
  - **p50**: <50ms (in-memory operation, no DB query)
  - **p95**: <150ms
  - Success rate: 100%
  - AWS SDK handles signature generation efficiently
- **Automation**: ⚠️ Partial (Artillery/k6)
- **Tags**: `[performance, presigned-url, throughput]`

---

## 🎭 Edge Case Test Cases

### **Category 12: Edge Cases & Error Scenarios** (6 Test Cases)

#### **TC-054: Upload Reel with Empty Caption**
- **Priority**: P2 (Medium)
- **Preconditions**: None
- **Test Steps**:
  1. POST `/api/v1/media/upload-reel` with `caption=""` (empty string)
  2. Verify response 400 (validation error)
- **Expected Result**:
  - DTO validation fails: "caption should not be empty"
- **Automation**: ✅ Yes (Jest)
- **Tags**: `[validation, edge-case]`

---

#### **TC-055: Upload Reel with 500+ Character Caption**
- **Priority**: P2 (Medium)
- **Preconditions**: Caption exceeds 500 character limit
- **Test Steps**:
  1. POST `/api/v1/media/upload-reel` with 600-character caption
  2. Verify response 400
- **Expected Result**:
  - DTO validation fails: "caption must be shorter than or equal to 500 characters"
- **Automation**: ✅ Yes (Jest)
- **Tags**: `[validation, edge-case, caption]`

---

#### **TC-056: Upload Reel with Invalid Geolocation (Out of Range)**
- **Priority**: P2 (Medium)
- **Preconditions**: None
- **Test Steps**:
  1. POST `/api/v1/media/upload-reel` with:
     ```json
     {
       "locationLat": 91.0,  // Valid range: -90 to 90
       "locationLng": 200.0  // Valid range: -180 to 180
     }
     ```
  2. Verify response 400
- **Expected Result**:
  - DTO validation fails:
    - "locationLat must not be greater than 90"
    - "locationLng must not be greater than 180"
- **Automation**: ✅ Yes (Jest)
- **Tags**: `[validation, edge-case, geolocation]`

---

#### **TC-057: S3 Upload Interrupted - Resume Capability**
- **Priority**: P1 (High)
- **Preconditions**:
  - User uploads 50MB of 100MB file
  - Network disconnects
- **Test Steps**:
  1. User reopens app, retries POST `/api/v1/media/upload-reel` with SAME `uploadId`
  2. Backend returns existing `mediaId`
  3. User requests new presigned URL
  4. User resumes upload from 50MB (S3 multipart upload)
- **Expected Result**:
  - Upload resumes without re-uploading first 50MB
  - S3 multipart upload handles resume (client-side logic)
  - Backend idempotency prevents duplicate reel creation
- **Automation**: ⚠️ Manual (requires network simulation)
- **Tags**: `[edge-case, network-failure, resume]`

---

#### **TC-058: FFmpeg Processing Timeout - Fallback to MediaConvert**
- **Priority**: P1 (High)
- **Preconditions**: FFmpeg worker times out (corrupt video, unsupported codec)
- **Test Steps**:
  1. Upload video with corrupt frames
  2. FFmpeg processing fails 3 times (retry exhausted)
  3. Verify backend triggers MediaConvert fallback (if configured)
- **Expected Result**:
  - Media status: `FAILED` with `processingError` logged
  - If MediaConvert enabled: fallback job created
  - User notified of processing error
- **Automation**: ⚠️ Partial (requires FFmpeg failure simulation)
- **Tags**: `[edge-case, error-handling, fallback, ffmpeg]`

---

#### **TC-059: MongoDB Connection Loss During Upload Init**
- **Priority**: P2 (Medium)
- **Preconditions**: MongoDB becomes unavailable
- **Test Steps**:
  1. Attempt POST `/api/v1/media/upload-reel`
  2. Verify response 500 (Internal Server Error)
- **Expected Result**:
  - Error: "Database connection failed"
  - User sees: "Service temporarily unavailable, please try again"
  - No partial data created (transaction rollback)
- **Automation**: ⚠️ Partial (requires MongoDB failure injection)
- **Tags**: `[edge-case, error-handling, database, resilience]`

---

## 📊 Test Coverage Summary

### **By Priority**
| Priority | Test Cases | % of Total |
|----------|-----------|-----------|
| **P0 (Critical)** | 18 | 31% |
| **P1 (High)** | 27 | 46% |
| **P2 (Medium)** | 13 | 22% |
| **P3 (Low)** | 1 | 2% |
| **Total** | **59** | **100%** |

---

### **By Category**
| Category | Test Cases | Automation % |
|----------|-----------|--------------|
| Upload Initialization | 8 | 100% |
| Idempotency | 4 | 100% |
| Presigned URL | 6 | 83% |
| Upload Completion | 5 | 100% |
| Status Polling | 4 | 100% |
| Order/Menu Linking | 6 | 100% |
| User Tagging | 5 | 100% |
| Quota API | 3 | 100% |
| Image Upload | 3 | 67% |
| Security | 5 | 80% |
| Performance | 4 | 25% |
| Edge Cases | 6 | 50% |
| **Total** | **59** | **~82%** |

---

### **Automation Tools**
| Tool | Test Types | Test Count |
|------|-----------|-----------|
| **Jest (Unit + Integration)** | Controller, Service, DTO validation | 45 |
| **Postman/Newman (E2E)** | Full API workflows | 8 |
| **Artillery/k6 (Load)** | Performance, concurrency | 4 |
| **Manual** | S3 integration, network simulation | 2 |

---

## 🚀 Test Execution Guide

### **Setup Test Environment**
```bash
# 1. Start dependencies
docker-compose up -d mongodb postgres redis

# 2. Set test environment variables
export NODE_ENV=test
export MONGODB_URI=mongodb://localhost:27017/chefooz-test
export DB_HOST=localhost
export AWS_ACCESS_KEY_ID=test-key
export AWS_SECRET_ACCESS_KEY=test-secret
export S3_BUCKET_NAME=chefooz-media-test

# 3. Run database migrations
npm run migration:run

# 4. Seed test data (users, orders, menu items)
npm run seed:test
```

---

### **Run Unit + Integration Tests**
```bash
# Run all Media module tests
npm test -- apps/chefooz-apis/src/modules/media

# Run with coverage report
npm test -- --coverage apps/chefooz-apis/src/modules/media

# Run specific test file
npm test -- media.service.spec.ts

# Watch mode (during development)
npm test -- --watch media.service.spec.ts
```

**Expected Output**:
```
PASS  apps/chefooz-apis/src/modules/media/media.service.spec.ts
  MediaService
    initReelUpload
      ✓ should create media + reel documents for valid review reel (123ms)
      ✓ should throw FORBIDDEN if review reel quota exceeded (45ms)
      ✓ should return existing upload if uploadId matches (67ms)
      ...

Test Suites: 3 passed, 3 total
Tests:       45 passed, 45 total
Time:        15.234s
Coverage:    87.2% statements, 82.5% branches
```

---

### **Run E2E Tests (Postman)**
```bash
# 1. Start backend server (UAT mode)
npm run start:dev

# 2. Run Postman collection
newman run tests/e2e/media-module.postman_collection.json \
  --environment tests/e2e/uat.postman_environment.json \
  --reporters cli,html \
  --reporter-html-export ./test-reports/media-e2e.html
```

**Key E2E Flows**:
1. **Full Upload Flow**: Init → Presigned URL → S3 Upload → Complete → Poll Status
2. **Quota Enforcement**: Exhaust review reel quota → Verify rejection
3. **Idempotency**: Retry upload with same `uploadId` → Verify no duplicates
4. **Order Linking**: Link to delivered order → Verify snapshot

---

### **Run Load Tests (Artillery)**
```bash
# 1. Create Artillery config: tests/load/media-upload.yml
artillery run tests/load/media-upload.yml --output ./test-reports/load-test-results.json

# 2. Generate HTML report
artillery report ./test-reports/load-test-results.json --output ./test-reports/load-test.html
```

**Artillery Config Example**:
```yaml
config:
  target: 'https://api-uat.chefooz.com'
  phases:
    - duration: 60
      arrivalRate: 10  # 10 virtual users per second
  variables:
    authToken: '<jwt-token>'

scenarios:
  - name: 'Upload Reel Init'
    flow:
      - post:
          url: '/api/v1/media/upload-reel'
          headers:
            Authorization: 'Bearer {{ authToken }}'
          json:
            uploadId: '{{ $uuid }}'
            fileName: 'test-reel.mp4'
            mimeType: 'video/mp4'
            durationSec: 30
            sizeBytes: 10000000
            caption: 'Load test reel'
```

---

## 📈 Quality Gates

### **Release Criteria**
Before deploying to production, verify:

✅ **Code Coverage**: ≥85% for Media module  
✅ **Unit Tests**: 100% pass rate  
✅ **E2E Tests**: 100% pass rate  
✅ **Load Tests**: p95 latency <1s for upload init  
✅ **Security Tests**: All P0 security tests pass  
✅ **Manual QA**: Smoke test on UAT environment  

---

## 🐛 Bug Report Template

When reporting media upload bugs, include:

```markdown
**Bug ID**: MED-XXX
**Severity**: Critical / High / Medium / Low
**Environment**: UAT / Production
**Reproducibility**: Always / Intermittent

**Steps to Reproduce**:
1. Login as user with ID `<user-id>`
2. POST `/api/v1/media/upload-reel` with payload: <json>
3. Observe response: <actual-response>

**Expected Behavior**:
<what should happen>

**Actual Behavior**:
<what actually happened>

**Logs**:
<paste relevant backend logs>

**Request Trace ID**: <if available from response headers>

**Additional Context**:
- User tier: BRONZE / SILVER / GOLD / PLATINUM
- Quota status: <review quota>, <promo quota>
- Device: iOS 17 / Android 14 / Web
- Network: 4G / 5G / WiFi
```

---

**[SLICE_COMPLETE ✅]**  
*Media Module QA Test Cases - Comprehensive test documentation complete (59 test cases, ~82% automation coverage)*
