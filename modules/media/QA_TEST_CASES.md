# Media Upload Module - QA Test Cases

> **Module**: `apps/chefooz-apis/src/modules/media`  
> **Test Coverage Target**: 85%+ (Unit + Integration + E2E)  
> **Last Updated**: March 14, 2026  
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

### TC-MEDIA-BUG-001: Photo POST uploaded as video in profile grid

**Type:** Bug Regression  
**Feature area:** Upload pipeline — POST content type  
**Priority:** P0

**Preconditions:**
- User is logged in
- User selects 2+ images and shares as a POST (not a REEL)

**Steps:**
1. Open upload flow, select 2 images
2. Tap share — observe frontend logs confirm `contentType: 'POST'` is passed from `share.tsx`
3. Wait for upload to complete
4. Open profile grid

**Expected result:** Profile grid shows image badge (not play icon); tapping opens `/post/[id]` with a carousel  
**Actual result (before fix):** Images were processed as a 5-second video by FFmpeg; profile showed a video player  
**Root cause:** `initReelUpload()` in `apps/chefooz-app/src/services/media.service.ts` built the API payload manually but **omitted** `contentType`. Backend received `contentType: undefined`, defaulted to `'REEL'`, stored reel as `contentType: 'REEL'`. In `completeUpload()`, the check `reel?.contentType === 'POST'` returned false → FFmpeg pipeline ran, converting the image to a 5-second video.  
**Fix applied:**  
1. Added `contentType: data.contentType` to the payload in `apps/chefooz-app/src/services/media.service.ts` `initReelUpload()`  
2. Added defensive mime-type fallback in `apps/chefooz-apis/src/modules/media/media.service.ts` `completeUpload()`: if `media.mimeType` is an image MIME type, treat as POST regardless of stored `contentType`; also corrects the stored `contentType` on the reel document  
**Regression test:** Verify backend log shows `📸 POST detected — skipping FFmpeg` (not `🎬 Enqueueing FFmpeg processing job`)  
**Status:** Fixed ✅

---

### TC-MEDIA-BUG-002: PostCropModal WYSIWYG — pinch-zoom release no longer causes position jump

**Type:** Bug Regression  
**Feature area:** PostCropModal — crop editor for photo posts  
**Priority:** P1

**Preconditions:**
- User opens PostCropModal with one or more images
- User performs a pinch-zoom gesture (two fingers) while slightly translating (common on physical devices)

**Steps:**
1. Upload a POST \u2014 PostCropModal opens
2. Use two fingers to zoom in
3. Lift both fingers simultaneously — observe if the crop frame jumps
4. Verify by tapping Done and confirming the saved crop matches what was shown

**Expected result:** Crop frame position is stable on finger lift; saved crop pixel coordinates match the visual exactly

**Actual result (before fix):** At the moment of lifting both fingers after a pinch, the stored pan offset jumped by the centroid drift (`gesture.dx/dy`). This caused the crop to be saved at a different position than displayed. The visual briefly jumped at release and the resulting cropped image did not match what the user had framed.

**Root cause:** `onPanResponderRelease` always computed `rawX = panOffsetRef.current.x + gesture.dx`. For a pinch gesture, `gesture.dx` is the two-finger centroid movement from grant, which the `onPanResponderMove` pinch handler never applied to pan. This created a stored-state vs visual-state mismatch on every pinch release.

**Fix applied:** In `apps/chefooz-app/src/components/upload/PostCropModal.tsx`: check `wasPinch = pinchRef.current !== null` before clearing `pinchRef`. For pinch releases, clamp existing `panOffsetRef` at new scale (no `gesture.dx/dy` added). For single-finger pan releases, behaviour is unchanged.

**Regression test:** Pinch-zoom in PostCropModal \u2014 confirm no position jump on release; crop result matches visible frame
**Status:** Fixed \u2705

---

### TC-MEDIA-BUG-003: PostCropModal 4:5 crop result matches the framed preview

**Type:** Bug Regression / Automated
**Feature area:** PostCropModal — photo post crop editor
**Priority:** P1

**Preconditions:**
- User selects one or more images for a POST upload
- User changes the crop ratio to `4:5`
- Test image includes enough horizontal detail to detect left/right drift after crop

**Steps:**
1. Open POST upload and select images from the gallery
2. In `PostCropModal`, switch the crop ratio to `4:5`
3. Pan or zoom the image horizontally until a distinct subject is centered in the crop frame
4. Tap Done and return to the upload edit screen
5. Compare the resulting preview with the framing that was shown inside the crop modal

**Expected result:** The saved `4:5` image matches the crop framed in the modal with no horizontal left shift
**Actual result (before fix):** The old portrait crop path used `9:16`, which was a poor fit for post preview and could be harder to verify visually after processing
**Fix applied:** `PostCropModal` now uses `4:5` for portrait posts, with shared geometry helpers for frame size, rendered bitmap size, clamp bounds, and crop rect generation so preview transforms and crop output are derived from the same dimensions
**Regression test:** `apps/chefooz-app/src/components/upload/PostCropModal.utils.spec.ts`
**Status:** Fixed ✅

---

### TC-MEDIA-BUG-004: POST image preview stretches / overflows when crop modal is cancelled

**Type:** Bug Regression / Manual
**Feature area:** Edit screen — POST image preview after crop cancel
**Priority:** P1

**Preconditions:**
- User is on the upload edit screen with content type set to POST
- No images have been selected yet

**Steps:**
1. Tap the gallery icon to open image picker
2. Select one or more images
3. The crop modal opens — tap **Cancel** without cropping
4. Observe the image preview on the edit screen

**Expected result:** Images appear in a `4:5` (portrait) aspect ratio preview that fits within the screen bounds
**Actual result (before fix):** Images had no aspect ratio constraint applied so they stretched or overflowed the preview frame
**Fix applied:** `handleCropCancel` in `edit.tsx` now calls `setPostAspectRatio('4:5')` + `setImages(pendingCropImages)` when this is the first-time selection cancel, ensuring images are accepted with the `4:5` default
**Regression test:** Manual — select images for POST, cancel crop, verify preview is bounded correctly
**Status:** Fixed ✅

---

### TC-MEDIA-BUG-005: No way to re-adjust crop after POST images are selected

**Type:** UX / Manual
**Feature area:** Edit screen — POST image re-crop
**Priority:** P2

**Preconditions:**
- User has selected images for a POST upload (crop modal already completed or cancelled)

**Steps:**
1. Select images for a POST upload
2. Complete or cancel the crop modal
3. Return to the edit screen
4. Observe bottom-left of the image preview area

**Expected result:** A "Crop" button is visible at the bottom-left of the image, allowing the user to re-open the crop modal at any time
**Actual result (before fix):** No re-crop affordance existed; users would have to discard images and re-select to change the crop
**Fix applied:** Added `handleReCrop` handler and a `reCropButton` `TouchableOpacity` (icon + label) rendered at the bottom-left of the POST image preview in `edit.tsx`; position lifts up when the multi-image thumbnail strip is also visible
**Regression test:** Manual — select POST images, verify "Crop" button appears; tap it, verify crop modal opens with current images; crop and verify preview updates
**Status:** Fixed ✅

---

### TC-MEDIA-BUG-006: Share screen sub-components white in dark mode

**Type:** Bug Regression / Manual
**Feature area:** Share screen — UploadQuotaBanner, CaptionInputWithSuggestions, MetadataRow, LocationRow, VisibilitySelector
**Priority:** P1

**Preconditions:**
- Device is set to dark mode (system or in-app)
- User is on the Share reel upload screen (Stage 3)

**Steps:**
1. Enable dark mode on the device
2. Open the app and navigate to reel upload
3. Reach the Share / metadata stage (Stage 3)
4. Observe: promotional banner, caption input, Details section, Add Location row, Visibility bottom sheet

**Expected result:** All components render with dark backgrounds (`colors.surface` or `colors.background`), theme-aware text colours, and visible borders — matching the rest of the dark UI.
**Actual result (before fix):** All 5 sub-components used `CHEFOOZ_COLORS.*` static constants (deprecated), rendering with hardcoded light colours (`#F8F8F8`, `#FFFFFF`, `#333333`, `#F5F5F5`, `#FFF8E1`, `#FFEBEE`) regardless of theme — appearing as bright white blocks in dark mode.
**Fix applied:** Migrated all 5 components to `makeStyles(colors, isDark)` factory pattern called via `useMemo`. Replaced all `CHEFOOZ_COLORS.*` references with adaptive `useChefoozTheme()` tokens (`colors.surface`, `colors.background`, `colors.textPrimary`, `colors.textSecondary`, `colors.textMuted`, `colors.border`, `colors.interactiveSubtle`, `colors.warning`, `colors.danger`). Removed `CHEFOOZ_TYPOGRAPHY` spread color conflicts by inlining explicit `fontSize`/`color` properties where needed.
**Regression test:** Manual — toggle dark mode and verify all 5 components adapt: UploadQuotaBanner, CaptionInputWithSuggestions, MetadataRow, LocationRow, VisibilitySelector
**Status:** Fixed ✅

---

---

### TC-MEDIA-BUG-007: Review reel upload crashes with 500 — "Cannot read properties of undefined (reading 'count')"

**Type:** Bug Regression  
**Feature area:** `apps/chefooz-apis/src/modules/media/media.service.ts` — `initReelUpload`  
**Priority:** P0

**Preconditions:**
- User has at least one completed (DELIVERED) order
- User attempts to upload a review reel (DTO includes `orderId`)

**Steps:**
1. Open the upload flow on staging
2. Record or select a video
3. Set the reel type as a review (link to an order)
4. Tap "Share" / publish

**Expected result:** Upload initialises successfully; S3 pre-signed URL returned  
**Actual result (before fix):** Server responds with HTTP 500: `TypeError: Cannot read properties of undefined (reading 'count')`  
**Root cause:** `initReelUpload` references `this.ordersRepository` (with an extraneous 's') inside the `USER_REVIEW` quota check branch. The injected property is `this.orderRepository` (singular). The typo meant the property was always `undefined` at runtime; calling `.count()` on it crashed every review reel upload attempt.  
Line 325 in `media.service.ts`:
```ts
// BEFORE (broken)
const completedOrdersCount = await this.ordersRepository.count({ ... });
// AFTER (fixed)
const completedOrdersCount = await this.orderRepository.count({ ... });
```
**Fix applied:** Corrected typo `ordersRepository` → `orderRepository` in `media.service.ts` line 325. Also corrected hardcoded `'DELIVERED'` → `OrderStatus.DELIVERED` (enum value is `'delivered'` lowercase) — the case mismatch caused the order count to always return 0, blocking all review reel uploads with a false "quota full" error even when users had completed orders.  
**Regression test:** Manual — upload a review reel on staging; confirm 200 + presigned URL returned.  
**Status:** Fixed ✅

---

### TC-MEDIA-BUG: Food video incorrectly flagged as `needs_review` on staging

**Type:** Bug Regression  
**Feature area:** `aws-rekognition.provider.ts` — explicit score calculation  
**Priority:** P1

**Preconditions:**
- Staging environment (`CHEFOOZ_ENV=staging`)
- Chef uploads a legitimate food video (e.g. brownie, dark chocolate sauce, grilled meat)

**Steps:**
1. Chef records and uploads a food video
2. Video processing completes (FFmpeg / MediaConvert)
3. Moderation pipeline runs `analyzeVideo()` via Rekognition
4. AWS Rekognition returns `Suggestive` label at 65-75% confidence for dark/brown food tones
5. `explicitScore` exceeds the staging `explicitContentReviewThreshold` (65%)

**Expected result:** Video is approved; chef's content goes live.

**Actual result (before fix):** Video receives `moderationStatus = needs_review` despite showing entirely safe food content. The video is still visible to customers (feed includes `needs_review`) but it lands in the admin human-review queue as a false positive, creating noise and review workload.

**Root cause:** `Suggestive` was included in the four labels used to compute `explicitScore`. AWS Rekognition's `Suggestive` category is too broad — it matches dark/brown textures (chocolate, sauces, brownie batter) commonly found in food photography. The explicit review threshold on staging (65%) is below the typical `Suggestive` confidence for food close-ups (65-75%).

**Fix applied:** Removed `'Suggestive'` from the explicit-score label arrays in both `analyzeContent()` (image) and `analyzeVideo()` (video) in `aws-rekognition.provider.ts`. Truly explicit content in a food-app context will still be caught by `'Explicit Nudity'`, `'Nudity'`, and `'Sexual Activity'`.

**Regression test:** Upload a dark-food video (brownie, grilled meat, chocolate sauce) on staging; confirm `moderationStatus = approved`.  
**Status:** Fixed ✅

---

---

### TC-MEDIA-BUG-008: Crop modal shows blank preview when uploading images from gallery

**Type:** Bug Regression
**Feature area:** `PostCropModal.tsx`, `useVideoPicker.ts`
**Priority:** P1

**Preconditions:**
- User is on the Upload tab, content type is POST
- User taps the gallery button to select multiple images

**Steps:**
1. Tap the gallery button on the Upload/Edit screen
2. Select 1–10 images from the photo library
3. The crop modal opens
4. Observe the main crop frame preview and thumbnails at the bottom

**Expected result:** Selected image(s) are displayed in the crop frame and thumbnail strip. User can pan and pinch to adjust crop.
**Actual result (before fix):** The crop frame showed a solid dark (#111) background with no image visible. Thumbnails were also blank. User could not see anything to crop.
**Root cause:** `pickMultipleImages` used `quality: 0.8` in `launchImageLibraryAsync`. Combining a quality setting with `allowsMultipleSelection: true` triggers expo-image-picker's pre-compression step. On some Android devices and iOS versions with limited photo library permissions, this produces empty or partially-written temporary files. The resulting URIs appeared structurally valid but the underlying file data was absent, causing React Native's `Image` component to render blank. PostCropModal already applies 0.85 quality compression via `expo-image-manipulator` on Done, making the pre-compression step redundant and harmful.
**Fix applied:**
- `useVideoPicker.ts` — removed `quality: 0.8` from `pickMultipleImages` `launchImageLibraryAsync` options; added `exif: false`. PostCropModal's `ImageManipulator` handles final compression at Done.
- `PostCropModal.tsx` — added `imageLoadError` state. The crop frame `Image` now has an `onError` callback that sets `imageLoadError = true` to surface failures. When `imageLoadError` is true, a visible error placeholder is shown instead of a silent blank frame. State resets to `false` when the modal opens and when switching between images in `switchToImage`.
**Regression test:** Manual — select multiple images from the gallery; confirm crop preview shows immediately. Test on Android 13+ and iOS with limited photo access.
**Status:** Fixed ✅

---

### TC-MEDIA-BUG-009: LiveCameraView migrated from expo-camera to react-native-vision-camera v5

**Type:** Architecture Enhancement  
**Feature area:** `apps/chefooz-app/src/components/upload/LiveCameraView.tsx`  
**Priority:** P0

**Preconditions:**
- Fresh EAS build after the VisionCamera plugin is added to app.json (requires new native binary)
- Camera and microphone permissions granted on device

**Steps:**

**Photo capture:**
1. Open Upload tab — camera preview should appear immediately
2. Tap the shutter button once
3. Confirm a photo is taken and preview appears in edit.tsx
4. Test with flash = off / on / auto modes

**Video recording:**
1. Long-press the shutter button
2. Confirm recording indicator (red dot + timer) appears
3. Release after 5–10 seconds
4. Confirm video preview loads in edit.tsx
5. Confirm duration is passed correctly to the parent

**Camera flip:**
1. Tap the flip button
2. Confirm front/back switching works without crash

**Zoom:**
1. Slide finger up on the camera preview
2. Confirm zoom increases
3. Confirm zoom releases back to 0 when gesture ends

**Permission denied — camera:**
1. Revoke camera permission in device settings
2. Open the Upload tab
3. Confirm "Camera permission required" + tappable "Allow Camera" button

**Permission denied — microphone:**
1. Grant camera permission but deny microphone
2. Open the Upload tab — camera preview should show
3. Long-press to record
4. Confirm alert appears with "Microphone Permission Required" + "Open Settings"

**Expected result:** All flows above pass without crash, lag, or missing URI errors.
**Actual result (before fix):** expo-camera had no Frame Processor API, Promise-based recording race conditions on Android stopRecording(), and no real device zoom range.
**Fix applied:** Full rewrite of `LiveCameraView.tsx` using VisionCamera v5; VisionCamera plugin added to `app.json`; `CameraType` import from expo-camera removed in `edit.tsx`. Public ref API (`startRecording`, `stopRecording`, `takePhoto`) is unchanged — no changes required in parent screens.
**Regression test:** Manual — full camera flow on both iOS and Android staging builds.
**Status:** Fixed ✅

---

### TC-MEDIA-067: Upload FAB tap crashes app on Android (Media3 version conflict)

**Type:** Bug Regression  
**Feature area:** Upload FAB / Video playback (ExoPlayer)  
**Priority:** P0

**Preconditions:**
- Android device/emulator
- User is logged in and on the feed screen

**Steps:**
1. Open the app on Android
2. Tap the Upload FAB (bottom + button)

**Expected result:** Upload flow or camera opens without crash  
**Actual result (before fix):** App immediately crashed with `AbstractMethodError: LoadControl.getAllocator(PlayerId)`  
**Root cause:** CameraX 1.7.0-alpha01 → `media3-muxer:1.9.0` → BOM forces all Media3 to 1.9.0 → `expo-video` compiled against 1.8.0's `LoadControl` interface is missing a new abstract method added in 1.9.0  
**Fix applied:** Force all `androidx.media3:*` to `1.8.0` via `resolutionStrategy.eachDependency` in `android/build.gradle` — with `media3-muxer` and `media3-container` **exempt** (see TC-MEDIA-068)  
**Regression test:** Manual — tap upload FAB on Android; verify no crash, camera opens  
**Status:** Fixed ✅

---

### TC-MEDIA-068: Video recording crashes on Android (MediaMuxerCompat not found)

**Type:** Bug Regression  
**Feature area:** Camera / Video Recording (CameraX + media3-muxer)  
**Priority:** P0

**Preconditions:**
- Android device (tested on Samsung SM-A505F, Android 11 / API 30)
- App built after TC-MEDIA-067 fix (all media3 forced to 1.8.0)
- Camera opened successfully

**Steps:**
1. Open the app on Android
2. Tap the Upload FAB to open the camera
3. Switch to video mode (if not default)
4. Press and hold the record button to start recording

**Expected result:** Video records without crash  
**Actual result (before fix):** App crashed immediately on recording start with:
```
FATAL EXCEPTION: CameraX-camerax_io_1
java.lang.NoClassDefFoundError: Failed resolution of: Landroidx/media3/muxer/MediaMuxerCompat;
  at androidx.camera.video.internal.muxer.Media3MuxerImpl.setOutput
```  
**Root cause:** `MediaMuxerCompat` was introduced in `media3-muxer:1.9.0`. CameraX 1.7.0-alpha01's `Media3MuxerImpl` uses it for video muxing. The TC-MEDIA-067 fix blanket-pinned all `androidx.media3:*` to `1.8.0`, including `media3-muxer`, removing the class from the APK's DEX.  
**Fix applied:** Intermediate fix — exempted `media3-muxer` and `media3-container` from the 1.8.0 pin. Superseded by TC-MEDIA-069 final fix.  
**Status:** Fixed ✅ (via TC-MEDIA-069 final strategy)

---

### TC-MEDIA-069: Video recording crashes on Android (NoSuchMethodError in MediaFormatUtil)

**Type:** Bug Regression  
**Feature area:** Camera / Video Recording (CameraX + media3-muxer + media3-common)  
**Priority:** P0

**Preconditions:**
- Android device (tested on Samsung SM-A505F, Android 11 / API 30)
- App built after TC-MEDIA-068 intermediate fix (media3-muxer/container exempt, media3-common still at 1.8.0)
- Camera opened successfully

**Steps:**
1. Open the app on Android
2. Tap the Upload FAB to open the camera
3. Press record button to start recording a video

**Expected result:** Video records without crash  
**Actual result (before fix):**
```
FATAL EXCEPTION: CameraX-camerax_io_1
java.lang.NoSuchMethodError: No static method getFloatFromIntOrFloat(...)
  in Landroidx/media3/common/util/MediaFormatUtil;
  at MediaMuxerCompat.addTrack (media3-muxer:1.9.0)
```  
**Root cause:** `media3-muxer:1.9.0` was compiled against `media3-common:1.9.0` and calls `MediaFormatUtil.getFloatFromIntOrFloat()` added in 1.9.0. The exclusion-list approach (exempt only muxer+container) still left `media3-common` at 1.8.0. Any new Media3 muxer version calling new common APIs will continue to break with this approach — it is inherently fragile.  
**Fix applied:** Replaced the exclusion-list strategy with an **explicit pin-list** strategy. Only artifacts that expo-video directly compiled against 1.8.0 are pinned (`media3-exoplayer`, `media3-exoplayer-dash`, `media3-exoplayer-hls`, `media3-session`, `media3-ui`, `media3-datasource-okhttp`). Everything else (`media3-common`, `media3-muxer`, `media3-container`, `media3-extractor`) floats to 1.9.0.  
**Regression test:** Manual — open camera, record a short video, verify no crash, verify media saves correctly  
**Status:** Fixed ✅

---

### TC-MEDIA-070: Video recording crashes on iOS (invalidAVFileType)

**Type:** Bug Regression  
**Feature area:** Camera / Video Recording (iOS VisionCamera v5)  
**Priority:** P0

**Preconditions:**
- iOS device (any — reproducible on all devices running VisionCamera v5.0.6)
- Camera screen opened, camera permission granted

**Steps:**
1. Open the app on iOS
2. Tap the Upload FAB to open the camera
3. Press and hold the record button to start recording a video

**Expected result:** Video recording starts without error  
**Actual result (before fix):**
```
LOG  📹 Recording started
ERROR  ❌ VisionCamera createRecorder failed: [Error: invalidAVFileType]
WARN  ⚠️ Recording failed — resetting recording state
```  
**Root cause:** VisionCamera v5.0.6 `URL+createTempURL.swift` has a type mismatch bug:
```swift
// Bug: AVFileType.rawValue is a UTI ("com.apple.quicktime-movie"), NOT a MIME type
let mimeType = fileType.rawValue
guard let utFileType = UTType(mimeType: mimeType) else {
  throw TemporaryFileError.invalidAVFileType  // <-- thrown every time for .mov (the default)
}
```
`AVFileType.rawValue` returns a UTI identifier string. `UTType(mimeType:)` strictly requires MIME type strings (e.g. `"video/quicktime"`). The default `fileType` is `.mov` (`"com.apple.quicktime-movie"` UTI) → `UTType(mimeType:)` returns `nil` → `invalidAVFileType` is always thrown.  
**Fix applied:** Patched `node_modules/react-native-vision-camera/ios/Extensions/URL+createTempURL.swift` to use `UTType(fileType.rawValue)` (non-failable UTI identifier initializer) instead of `UTType(mimeType:)`. Fix persisted via `scripts/postinstall.js` to survive `yarn install`.  
**Regression test:** Manual — open camera on iOS, hold record button, verify recording starts and video saves without error  
**Status:** Fixed ✅

---

**[SLICE_COMPLETE ✅]**  
*Media Module QA Test Cases - Updated April 2026 (70 test cases)*

