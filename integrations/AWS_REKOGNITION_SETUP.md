# AWS Rekognition Setup Guide — Chefooz Moderation

> Last Updated: March 2026
> Environment: `ap-south-1` (Mumbai)
> Affected module: `apps/chefooz-apis/src/modules/moderation/`

---

## Overview

Chefooz uses **Amazon Rekognition Content Moderation** to automatically scan user-uploaded media for inappropriate content (nudity, violence, drugs, weapons, hate symbols). Two distinct APIs are used:

| Content Type | API Used | Approach |
|---|---|---|
| Reels (video, `.mp4`) | `StartContentModeration` + `GetContentModeration` | Async (multi-frame) |
| Profile photos (image) | `DetectModerationLabels` | Sync (single frame) |
| Stories (image or video) | `DetectModerationLabels` | Sync |
| Cover images (image) | `DetectModerationLabels` | Sync |

### How It Fits Into Chefooz

```
Media Upload
    │
    ▼
NestJS API (media/stories/profile modules)
    │  calls startModeration() — non-blocking
    ▼
ModerationService.startModeration()
    │  creates ModerationRecord (PostgreSQL, status=pending)
    │  syncs MongoDB moderationStatus = 'pending' (shadow-hidden)
    │  enqueues Bull job: { s3Key, s3Bucket, contentType }
    ▼
Bull Queue ('moderation', concurrency=2)
    │
    ▼
ModerationProcessor.handleAnalyzeContent()
    │
    ├─ if reel → AWSRekognitionProvider.analyzeVideo()
    │               StartContentModeration → polls GetContentModeration
    │               polls every 2s, max 30s
    │
    └─ if image → AWSRekognitionProvider.analyzeContent()
                    DetectModerationLabels (immediate response)
    │
    ▼
ModerationRulesService.evaluateContent()
    │  applies business thresholds (env-specific)
    │  decision: approved | needs_review | rejected
    ▼
Update PostgreSQL (ModerationRecord status)
Update MongoDB (Reel/Media moderationStatus) → shadow-ban
Send notification to user
Auto-ban check (3+ rejections in 24h → accountStatus = suspended)
```

---

## AWS Architecture Comparison: Our Approach vs AWS Sample

The AWS sample repo you referenced (`aws-samples/amazon-rekognition-serverless-large-scale-image-and-video-processing`) uses a **fully serverless CDK pipeline**: S3 → Lambda → SQS → Lambda workers → Rekognition → SNS → Results Lambda → DynamoDB.

**We intentionally do NOT use that architecture.** Here's why and what we use instead:

| Dimension | AWS Sample | Chefooz |
|---|---|---|
| Job orchestration | SQS + Lambda | Bull Queue + NestJS Worker |
| Trigger | S3 `ObjectCreated` event | Programmatic (after upload completes) |
| Results storage | DynamoDB + S3 JSON | PostgreSQL + MongoDB (integrated with app data) |
| Video polling | SNS completion callback | Polling (`GetContentModeration`, 2s interval, 30s max) |
| Deployment | CDK / CloudFormation | Runs inside `chefooz-apis` NestJS process |
| Scale | Millions of items (backfill) | App-scale (~100 uploads/day) |
| Setup complexity | High (deploy CDK stack) | Low (just IAM permissions) |

The AWS sample is the right choice for **bulk backfill** of millions of existing items. For Chefooz's scale and integrated architecture, Bull Queue inside NestJS is the correct choice.

---

## Prerequisites

- AWS account with `ap-south-1` (Mumbai) region available
- An existing S3 bucket where uploaded media is stored (see `S3_BUCKET_NAME` in env)
- Node.js 18+ and AWS CLI v2 installed locally for verification

---

## Step 1 — IAM Setup

Rekognition needs IAM credentials to call its APIs. The IAM entity must also be able to read from S3 (Rekognition accesses the media file directly using your credentials).

### Option A: IAM User (simpler, current approach)

1. Go to **AWS Console → IAM → Users**
2. Create user: `chefooz-rekognition-worker` (or add permissions to existing `chefooz-media` user)
3. Attach the following **inline policy** or create a managed policy named `ChefoozRekognitionPolicy`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "RekognitionImageModeration",
      "Effect": "Allow",
      "Action": [
        "rekognition:DetectModerationLabels"
      ],
      "Resource": "*"
    },
    {
      "Sid": "RekognitionVideoModeration",
      "Effect": "Allow",
      "Action": [
        "rekognition:StartContentModeration",
        "rekognition:GetContentModeration"
      ],
      "Resource": "*"
    },
    {
      "Sid": "S3ReadForRekognition",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject"
      ],
      "Resource": [
        "arn:aws:s3:::chefooz-media-staging/*",
        "arn:aws:s3:::chefooz-media-production/*",
        "arn:aws:s3:::chefooz-media-uat/*"
      ]
    }
  ]
}
```

> **Note**: Replace bucket names with your actual bucket ARNs. Wildcard `/*` is required — Rekognition reads the specific object by key.

4. Generate **Access Key** (Access key ID + Secret access key) under the user's Security credentials tab
5. Save these as your `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`

### Option B: IAM Role + Instance Profile (recommended for EC2/ECS)

If `chefooz-apis` runs on EC2 or ECS:
1. Attach the `ChefoozRekognitionPolicy` above directly to the **EC2 Instance Role** or **ECS Task Role**
2. Remove `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` from your `.env` — the SDK will use instance metadata automatically

---

## Step 2 — S3 Bucket Configuration

Rekognition reads the media file from S3 using your credentials. No additional bucket policy is needed beyond what your IAM user already has (`s3:GetObject`).

**Important**: The S3 object must exist and be readable by the time the Rekognition job runs. Our Bull Queue job fires only after the upload confirmation from MediaConvert / the upload service, so timing is not an issue.

**Verify bucket access**:
```bash
aws s3api head-object \
  --bucket chefooz-media-staging \
  --key uploads/some-mediaId/original.mp4 \
  --region ap-south-1
```

---

## Step 3 — Environment Variables

Add or verify this block in your `.env` file:

```bash
# ──────────────────────────────────────────────────────
# AWS Rekognition — Content Moderation
# ──────────────────────────────────────────────────────

# Core AWS credentials (shared with S3, MediaConvert, SNS)
AWS_REGION=ap-south-1
AWS_ACCESS_KEY_ID=<your-iam-access-key>
AWS_SECRET_ACCESS_KEY=<your-iam-secret-key>

# S3 bucket where uploaded media is stored
# Rekognition reads media directly from this bucket
S3_BUCKET_NAME=chefooz-media-staging        # staging
# S3_BUCKET_NAME=chefooz-media-production   # production

# Rekognition tuning (optional — defaults are used if not set)
# These are baked into AWSRekognitionProvider and ModerationRulesService
# Override via libs/domain getEnvConfig() if needed
# REKOGNITION_MIN_CONFIDENCE=50             # labels below this are ignored
```

> The code reads `S3_BUCKET_NAME` (primary) with `AWS_S3_BUCKET` as legacy fallback. 
> Do NOT use `AWS_S3_BUCKET` in new deployments — use `S3_BUCKET_NAME`.

---

## Step 4 — Verify Rekognition Is Enabled in Your Region

Rekognition Content Moderation is available in `ap-south-1` (Mumbai). ✅

Verify via AWS CLI (requires the same IAM user credentials):

```bash
# Test image moderation (requires a real JPEG in your bucket)
aws rekognition detect-moderation-labels \
  --image '{"S3Object":{"Bucket":"chefooz-media-staging","Name":"uploads/test.jpg"}}' \
  --min-confidence 50 \
  --region ap-south-1

# Expected output: {"ModerationLabels": [...], "ModerationModelVersion": "7.0"}
# If empty array: image is clean ✅
# If labels present: image contains flagged content
```

```bash
# Test video moderation (submit a job)
aws rekognition start-content-moderation \
  --video '{"S3Object":{"Bucket":"chefooz-media-staging","Name":"uploads/test.mp4"}}' \
  --min-confidence 50 \
  --region ap-south-1

# Returns: {"JobId": "abc123..."}
# Then poll:
aws rekognition get-content-moderation \
  --job-id abc123... \
  --region ap-south-1
```

---

## Step 5 — Content Thresholds

Thresholds are environment-specific and managed in `libs/domain/src/lib/` via `getEnvConfig()`. The `ModerationRulesService` calls these functions:

| Function | Meaning | Typical Production Value |
|---|---|---|
| `shouldAutoRejectExplicit(score)` | Hard reject (auto-block) | `score >= 80` |
| `shouldReviewExplicit(score)` | Flag for human review | `score >= 50` |
| `shouldAutoRejectViolence(score)` | Hard reject violence | `score >= 80` |
| `shouldReviewViolence(score)` | Flag violence for review | `score >= 60` |

Additionally, these Rekognition label names **always trigger rejection** regardless of score:
- `Weapons`
- `Drugs`
- `Hate Symbols`
- `Graphic Violence`

---

## Step 6 — Service Limits & Costs

### API Limits (AWS default, ap-south-1)

| API | Default Limit | Our Usage |
|---|---|---|
| `DetectModerationLabels` | 5 TPS | Bull concurrency=2 → safe |
| `StartContentModeration` | 20 concurrent jobs/account | Bull concurrency=2 → safe |
| `GetContentModeration` (poll) | 25 TPS | 1 poll per 2s per job → safe |
| Max video duration | 10 hours | Chefooz reels: ≤60s → safe |
| Max video file size | 10 GB | Chefooz reels: ≤500MB → safe |

If you exceed `DetectModerationLabels` TPS, you'll get `LimitExceededException`. To raise limits: AWS Console → Service Quotas → Amazon Rekognition → Request quota increase.

### Approximate Cost (ap-south-1 pricing, March 2026)

| API | Cost |
|---|---|
| `DetectModerationLabels` | ~$0.001 per image (~$1 per 1,000 images) |
| `StartContentModeration` | ~$0.10 per minute of video analyzed |

For a 30-second reel: **~$0.05 per reel moderated**. At 100 new reels/day: ~$5/day, ~$150/month.

---

## Step 7 — Monitoring & Debugging

### Check if a moderation job is stuck (> 30s, no result)

Our polling times out after 30s and throws, which routes the content to `needs_review`. This is intentional — Rekognition Video jobs for very long videos can take > 30s.

If you see many `needs_review` items in the admin queue:
```bash
# Check CloudWatch for Rekognition job failures
aws logs filter-log-events \
  --log-group-name /aws/lambda/rekognition \
  --filter-pattern "FAILED" \
  --region ap-south-1
```

### Check Bull Queue for stalled moderation jobs

Bull Dashboard is available if `BULL_BOARD_ENABLED=true` in your env:
- URL: `http://your-api-host/admin/queues`
- Queue name: `moderation`
- Failed jobs will show the Rekognition error (e.g., access denied = wrong IAM permissions)

### Common Errors

| Error | Cause | Fix |
|---|---|---|
| `InvalidS3ObjectException` | S3 key doesn't exist yet | Ensure file upload completes before moderation trigger |
| `AccessDeniedException` | IAM user missing `rekognition:*` or `s3:GetObject` | Review Step 1 |
| `InvalidParameterException: unsupported image type` | Profile photo not JPEG/PNG | Convert before analysis (image-upload.service.ts handles this) |
| `LimitExceededException` | Exceeding 5 TPS on DetectModerationLabels | Reduce Bull concurrency or request quota increase |
| Timeout (> 30s) | Video job didn't complete within our poll window | Content routes to `needs_review` — admin handles it |
| `S3ObjectExceedsSizeLimit` | Video > 10 GB | Never happens with Chefooz reels (≤ 500MB) |

---

## Comparison: Polling vs SNS Notification (AWS Sample Approach)

Our implementation polls `GetContentModeration` every 2s (max 30s). The AWS sample uses an **SNS completion notification** instead.

| | Polling (current) | SNS notification |
|---|---|---|
| Setup complexity | None (just IAM) | Need: SNS topic, IAM role for Rekognition, Lambda/endpoint to receive callback |
| Latency | Up to 30s wait per job in the Bull worker | Near-instant when job completes |
| Worker thread blocked? | Yes — 1 Bull concurrency slot holds for up to 30s | No — fire-and-forget, job resumes on SNS event |
| Suitable for Chefooz scale | ✅ Yes (< 100 reels/day) | Overkill |
| Required for > 500 concurrent jobs | No — Amazon handles queue | Yes |

**When to switch to SNS**: If concurrent video moderation jobs exceed 20/day regularly, or you need < 5s latency on content going live, switch to SNS. Until then, polling is correct.

To enable SNS notifications for Rekognition Video, you would need:
1. Create SNS topic: `chefooz-rekognition-video-complete`
2. Create IAM role `ChefoozRekognitionVideoRole` with `sns:Publish` on that topic
3. Trust policy: `{"Service": "rekognition.amazonaws.com"}`
4. Pass the role ARN in `StartContentModerationCommand.NotificationChannel`
5. Subscribe an SQS queue or HTTP endpoint to the SNS topic
6. Call `GetContentModeration` only after the SNS event fires

---

## Required IAM Policy — Minimal Copy-Paste

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "rekognition:DetectModerationLabels",
        "rekognition:StartContentModeration",
        "rekognition:GetContentModeration"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::chefooz-media-*/*"
    }
  ]
}
```

> Replace `chefooz-media-*` with your exact bucket names for tighter security.

---

## Files Reference

| File | Purpose |
|---|---|
| `apps/chefooz-apis/src/modules/moderation/providers/aws-rekognition.provider.ts` | Wraps AWS SDK — `analyzeContent()` and `analyzeVideo()` |
| `apps/chefooz-apis/src/modules/moderation/services/moderation.service.ts` | Orchestrates pipeline, reads `S3_BUCKET_NAME` |
| `apps/chefooz-apis/src/jobs/moderation.processor.ts` | Bull Queue processor — concurrency=2 |
| `apps/chefooz-apis/src/modules/moderation/services/moderation-rules.service.ts` | Applies content thresholds |
| `libs/domain/src/lib/` | `getEnvConfig()` — environment-specific thresholds |
| `apps/chefooz-apis/.env.production.template` | Production env var template |
