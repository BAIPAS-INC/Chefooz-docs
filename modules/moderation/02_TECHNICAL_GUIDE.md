# Moderation Module - Technical Guide

**Module**: `moderation` + `report`  
**Type**: Content Safety & Community Guidelines  
**Last Updated**: March 2026

---

## 📋 Table of Contents

1. [Module Architecture](#module-architecture)
2. [Database Schema](#database-schema)
3. [API Endpoints](#api-endpoints)
4. [AI Integration (AWS Rekognition)](#ai-integration-aws-rekognition)
5. [Business Rules Engine](#business-rules-engine)
6. [Moderation Pipeline Implementation](#moderation-pipeline-implementation)
7. [Reporting System Implementation](#reporting-system-implementation)
8. [Admin Dashboard Implementation](#admin-dashboard-implementation)
9. [Graceful Failure Handling](#graceful-failure-handling)
10. [Testing Strategy](#testing-strategy)
11. [Performance Optimization](#performance-optimization)
12. [Security Considerations](#security-considerations)

---

## 🏗️ Module Architecture

### High-Level Architecture

```
┌───────────────────────────────────────────────────────────────────┐
│                      MODERATION SYSTEM                            │
├───────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌─────────────────┐     ┌──────────────────┐                   │
│  │  MediaService   │────▶│ ModerationService│                   │
│  │ (Upload Compl.) │     │  (Orchestrator)  │                   │
│  └─────────────────┘     └────────┬─────────┘                   │
│                                    │                              │
│                          ┌─────────▼─────────┐                   │
│                          │  Create Pending   │                   │
│                          │     Record        │                   │
│                          └─────────┬─────────┘                   │
│                                    │                              │
│                    ┌───────────────▼───────────────┐             │
│                    │    AI Analysis Pipeline       │             │
│                    ├───────────────────────────────┤             │
│                    │  1. AWS Rekognition API       │             │
│                    │  2. Extract Scores/Labels     │             │
│                    │  3. Apply Business Rules      │             │
│                    │  4. Make Decision             │             │
│                    │  5. Update Status             │             │
│                    │  6. Send Notification         │             │
│                    └───────────────┬───────────────┘             │
│                                    │                              │
│                      ┌─────────────▼──────────────┐              │
│                      │      Decision Outcomes      │              │
│                      ├─────────────────────────────┤              │
│                      │  • Approved (87%)           │              │
│                      │  • Rejected (12%)           │              │
│                      │  • Needs Review (0.8%)      │              │
│                      │  • AI Failure → Review (0.2%)│             │
│                      └─────────────────────────────┘              │
│                                                                   │
│  ┌──────────────────┐                                            │
│  │ User Reports     │────▶ ReportService                         │
│  │ (Community)      │      (Anti-spam + Escalation)              │
│  └──────────────────┘                                            │
│                                                                   │
│  ┌──────────────────┐                                            │
│  │ Admin Dashboard  │────▶ ModerationController                  │
│  │ (Manual Review)  │      (Approve/Reject + Audit)              │
│  └──────────────────┘                                            │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

### Component Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                     MODERATION COMPONENTS                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ModerationController                                           │
│  ├─ GET /moderation/my/:mediaId (user check status)            │
│  ├─ GET /moderation/pending (admin list queue)                 │
│  ├─ POST /moderation/:id/approve (admin approve)               │
│  ├─ POST /moderation/:id/reject (admin reject)                 │
│  └─ GET /moderation/:id/audit (admin view history)             │
│                                                                 │
│  ModerationService (Orchestrator)                               │
│  ├─ startModeration(mediaId, userId, s3Key)                    │
│  ├─ runModerationPipeline() [private]                          │
│  ├─ getModerationResult(mediaId, userId)                       │
│  ├─ getPendingReviews(cursor, limit)                           │
│  ├─ approveModerationManually(id, moderatorId, notes)          │
│  ├─ rejectModerationManually(id, moderatorId, notes)           │
│  ├─ getAuditLog(moderationId)                                  │
│  └─ checkForAutoBan(userId) [private]                          │
│                                                                 │
│  AWSRekognitionProvider (AI Integration)                        │
│  ├─ analyzeContent(s3Key, s3Bucket)                            │
│  └─ calculateCategoryScore(labels, categories) [private]       │
│                                                                 │
│  ModerationRulesService (Business Logic)                        │
│  ├─ evaluateContent(explicitScore, violenceScore, labels)      │
│  └─ shouldAutoBan(recentViolationsCount)                       │
│                                                                 │
│  ReportController                                               │
│  ├─ POST /report (user create report)                          │
│  ├─ GET /report/my (user list my reports)                      │
│  ├─ GET /admin/reports (admin list all reports)                │
│  ├─ POST /admin/reports/:id/review (admin take action)         │
│  └─ GET /admin/reports/:id (admin view details)                │
│                                                                 │
│  ReportService                                                  │
│  ├─ createReport(userId, dto)                                  │
│  ├─ getMyReports(userId, limit, cursor)                        │
│  ├─ getReportsForModerator(filters, limit, cursor)             │
│  ├─ reviewReport(reportId, adminId, dto)                       │
│  ├─ getReportById(reportId)                                    │
│  └─ countSimilarReports(dto) [private]                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🗄️ Database Schema

### Table: `moderation_records`

**Purpose**: Store AI moderation verdicts and human review decisions for each reel

```sql
CREATE TABLE moderation_records (
  id                              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  media_id                        VARCHAR(255) NOT NULL,
  user_id                         UUID NOT NULL,
  status                          VARCHAR(20) NOT NULL DEFAULT 'pending',
                                  -- Values: pending | ai_flagged | approved | rejected | needs_review
  
  -- AI Analysis Results
  ai_score_explicit               FLOAT NULL COMMENT 'AI explicit content score (0-100)',
  ai_score_violence               FLOAT NULL COMMENT 'AI violence score (0-100)',
  ai_labels                       JSONB NULL COMMENT 'Raw AI labels from Rekognition',
  rekognition_response_time       INTEGER NULL COMMENT 'API response time in milliseconds',
  
  -- Business Rules
  rules_triggered                 JSONB NULL COMMENT 'Rules that triggered flags/rejections',
  
  -- Decision Tracking
  final_decision_by               VARCHAR(20) NULL COMMENT 'ai | moderator',
  moderator_id                    UUID NULL,
  moderator_notes                 TEXT NULL,
  
  -- Graceful Failure Handling
  ai_failure_reason               TEXT NULL COMMENT 'Why AI moderation failed',
  moderation_fallback_triggered   BOOLEAN DEFAULT FALSE COMMENT 'Routed to human review due to AI failure',
  
  created_at                      TIMESTAMP DEFAULT NOW(),
  updated_at                      TIMESTAMP DEFAULT NOW(),
  
  -- Indexes
  CONSTRAINT idx_moderation_records_media_id         INDEX (media_id),
  CONSTRAINT idx_moderation_records_user_id          INDEX (user_id),
  CONSTRAINT idx_moderation_records_status           INDEX (status),
  CONSTRAINT idx_moderation_records_created_at       INDEX (created_at)
);
```

**Key Fields**:
- `media_id`: References `reels.id` (foreign key in application layer)
- `status`: Current moderation state (pending → ai_flagged/approved/needs_review → approved/rejected)
- `ai_score_explicit`: 0-100 score for explicit content (higher = more explicit)
- `ai_score_violence`: 0-100 score for violence (higher = more violent)
- `ai_labels`: Raw Rekognition labels (e.g., `["Explicit Nudity", "Suggestive"]`)
- `rules_triggered`: Array of business rules that triggered (e.g., `[{rule: "EXPLICIT_HARD_REJECT", severity: "critical"}]`)
- `final_decision_by`: Who made final decision (`ai` for auto-approve/reject, `moderator` for manual review)
- `ai_failure_reason`: Error message if Rekognition API failed (e.g., "timeout", "rate limit exceeded")
- `moderation_fallback_triggered`: Set to `true` if AI failed and content routed to human review

**Status Transitions**:
```
pending
  ├─ ai_flagged (borderline content, needs review)
  ├─ approved (auto-approved by AI or manually by moderator)
  ├─ rejected (auto-rejected by AI or manually by moderator)
  └─ needs_review (AI confidence low, or AI failed)
```

---

### Table: `moderation_audit`

**Purpose**: Audit trail for all moderation actions (compliance requirement)

```sql
CREATE TABLE moderation_audit (
  id                         BIGSERIAL PRIMARY KEY,
  moderation_record_id       UUID NOT NULL,
  event                      VARCHAR(50) NOT NULL,
                             -- Values: MODERATION_STARTED | AI_ANALYZED | RULES_EVALUATED | 
                             --         STATUS_CHANGED | AI_FAILED
  old_status                 VARCHAR(20) NULL,
  new_status                 VARCHAR(20) NULL,
  payload                    JSONB NULL COMMENT 'Event-specific data',
  actor_id                   UUID NULL COMMENT 'User or admin who triggered action',
  timestamp                  TIMESTAMP DEFAULT NOW(),
  
  -- Indexes
  CONSTRAINT idx_moderation_audit_record_id  INDEX (moderation_record_id),
  CONSTRAINT idx_moderation_audit_timestamp  INDEX (timestamp)
);
```

**Audit Events**:
1. **MODERATION_STARTED**: Record created
   ```json
   { "mediaId": "reel-123", "userId": "user-456" }
   ```

2. **AI_ANALYZED**: Rekognition analysis complete
   ```json
   { 
     "explicitScore": 65, 
     "violenceScore": 30, 
     "labels": ["Suggestive", "Revealing Clothes"],
     "responseTimeMs": 450
   }
   ```

3. **RULES_EVALUATED**: Business rules applied
   ```json
   {
     "decision": "needs_review",
     "rulesTriggered": [
       { "rule": "EXPLICIT_SOFT_FLAG", "reason": "Borderline explicit content (score 65)", "severity": "warning" }
     ]
   }
   ```

4. **STATUS_CHANGED**: Status updated
   ```json
   {
     "oldStatus": "pending",
     "newStatus": "approved",
     "reason": "AI auto-approve",
     "moderatorId": null
   }
   ```

5. **AI_FAILED**: Rekognition error
   ```json
   {
     "error": "Rekognition API timeout",
     "fallbackAction": "human_review_required",
     "timestamp": "2026-02-22T14:35:00Z"
   }
   ```

---

### Table: `reports`

**Purpose**: User-generated reports about problematic content or behavior

```sql
CREATE TABLE reports (
  id                      SERIAL PRIMARY KEY,
  reporter_id             UUID NOT NULL,
  reported_user_id        UUID NULL COMMENT 'User being reported (if applicable)',
  
  -- Content Targets (only one should be set)
  reel_id                 UUID NULL,
  message_id              UUID NULL,
  review_id               INTEGER NULL,
  profile_id              UUID NULL,
  
  -- Report Details
  category                VARCHAR(50) NOT NULL,
                          -- Values: spam | scam | nudity | violence | hate | 
                          --         harassment | copyright | impersonation | other
  message                 TEXT NOT NULL COMMENT 'User explanation',
  
  -- Status Tracking
  status                  VARCHAR(50) NOT NULL DEFAULT 'submitted',
                          -- Values: submitted | under_review | action_taken | rejected
  moderator_decision      TEXT NULL,
  moderator_id            UUID NULL,
  decision_at             TIMESTAMP NULL,
  
  -- Escalation Metadata
  is_escalated            BOOLEAN DEFAULT FALSE COMMENT '5+ reports in 1h for same target',
  similar_reports_count   INTEGER DEFAULT 0 COMMENT 'Count at report creation time',
  
  created_at              TIMESTAMP DEFAULT NOW(),
  updated_at              TIMESTAMP DEFAULT NOW(),
  
  -- Indexes
  CONSTRAINT idx_reports_reporter_id       INDEX (reporter_id),
  CONSTRAINT idx_reports_reported_user_id  INDEX (reported_user_id),
  CONSTRAINT idx_reports_category          INDEX (category),
  CONSTRAINT idx_reports_status            INDEX (status),
  CONSTRAINT idx_reports_reel_id           INDEX (reel_id) WHERE reel_id IS NOT NULL,
  CONSTRAINT idx_reports_created_at        INDEX (created_at)
);
```

**Key Fields**:
- `reporter_id`: User who submitted report
- `reported_user_id`: User being reported (null for system-level reports)
- **Content Targets**: Only one should be set (reel OR message OR review OR profile)
- `category`: Type of violation (9 enum values)
- `status`: Current state of report (submitted → under_review → action_taken/rejected)
- `is_escalated`: Set to `true` if ≥ 5 reports for same target in 1 hour
- `similar_reports_count`: Snapshot count at report creation time (for analytics)

**Constraints**:
- At least one target must be set: `CHECK (reel_id IS NOT NULL OR message_id IS NOT NULL OR review_id IS NOT NULL OR profile_id IS NOT NULL)`
- Cannot report yourself: Application-level validation in `ReportService`

---

## 🔌 API Endpoints

### User Endpoints

#### 1. GET /v1/moderation/my/:mediaId

**Purpose**: User checks moderation status of their own content

**Auth**: JWT required (user must own the reel)

**Request**:
```http
GET /api/v1/moderation/my/reel-12345 HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Response (Approved)**:
```json
{
  "success": true,
  "message": "Moderation result retrieved",
  "data": {
    "status": "approved",
    "explicitScore": 15,
    "violenceScore": 10,
    "labels": ["Food", "Kitchen", "Cooking"],
    "rulesTriggered": [],
    "moderatorNotes": null,
    "createdAt": "2026-02-22T10:15:30Z",
    "updatedAt": "2026-02-22T10:15:31Z"
  }
}
```

**Response (Rejected)**:
```json
{
  "success": true,
  "message": "Moderation result retrieved",
  "data": {
    "status": "rejected",
    "explicitScore": 85,
    "violenceScore": 20,
    "labels": ["Explicit Nudity", "Suggestive"],
    "rulesTriggered": [
      {
        "rule": "EXPLICIT_HARD_REJECT",
        "reason": "Explicit content score 85 exceeds threshold 80",
        "severity": "critical"
      }
    ],
    "moderatorNotes": null,
    "createdAt": "2026-02-22T10:15:30Z",
    "updatedAt": "2026-02-22T10:15:31Z"
  }
}
```

**Response (Under Review)**:
```json
{
  "success": true,
  "message": "Moderation result retrieved",
  "data": {
    "status": "needs_review",
    "explicitScore": 65,
    "violenceScore": 30,
    "labels": ["Suggestive", "Revealing Clothes"],
    "rulesTriggered": [
      {
        "rule": "EXPLICIT_SOFT_FLAG",
        "reason": "Borderline explicit content (score 65)",
        "severity": "warning"
      }
    ],
    "moderatorNotes": null,
    "createdAt": "2026-02-22T10:15:30Z",
    "updatedAt": "2026-02-22T10:15:31Z"
  }
}
```

**Errors**:
- `404 Not Found`: Moderation record not found (reel doesn't exist or not owned by user)
- `401 Unauthorized`: No JWT token or invalid token

---

#### 2. POST /v1/report

**Purpose**: User submits report about problematic content/behavior

**Auth**: JWT required

**Request**:
```http
POST /api/v1/report HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json

{
  "reelId": "reel-12345",
  "reportedUserId": "user-789",
  "category": "nudity",
  "message": "This reel contains inappropriate sexual content that violates community guidelines."
}
```

**Request Body Schema**:
```typescript
{
  // At least one target required
  reelId?: string;
  messageId?: string;
  reviewId?: number;
  profileId?: string;
  
  // Optional (null for system-level reports)
  reportedUserId?: string;
  
  // Required
  category: 'spam' | 'scam' | 'nudity' | 'violence' | 'hate' | 
            'harassment' | 'copyright' | 'impersonation' | 'other';
  message: string; // 10-2000 characters
}
```

**Response (Success)**:
```json
{
  "success": true,
  "message": "Report submitted successfully",
  "data": {
    "id": 12345,
    "reporterId": "user-456",
    "reportedUserId": "user-789",
    "reelId": "reel-12345",
    "category": "nudity",
    "message": "This reel contains inappropriate sexual content...",
    "status": "submitted",
    "isEscalated": false,
    "similarReportsCount": 2,
    "createdAt": "2026-02-22T14:35:00Z"
  }
}
```

**Response (Escalated)**:
```json
{
  "success": true,
  "message": "Report submitted successfully",
  "data": {
    "id": 12346,
    "reporterId": "user-456",
    "reportedUserId": "user-789",
    "reelId": "reel-12345",
    "category": "nudity",
    "message": "This reel contains inappropriate sexual content...",
    "status": "submitted",
    "isEscalated": true,
    "similarReportsCount": 6,
    "createdAt": "2026-02-22T14:35:00Z"
  }
}
```

**Errors**:
- `400 Bad Request`: Validation error
  - At least one target required
  - Cannot report yourself
  - Duplicate report within 24h
- `401 Unauthorized`: No JWT token

---

#### 3. GET /v1/report/my

**Purpose**: User lists their submitted reports

**Auth**: JWT required

**Query Parameters**:
- `limit` (number, default: 20): Max results per page
- `cursor` (number, optional): Pagination cursor (report ID)

**Request**:
```http
GET /api/v1/report/my?limit=20 HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Response**:
```json
{
  "success": true,
  "message": "Reports retrieved successfully",
  "data": {
    "items": [
      {
        "id": 12345,
        "reportedUserId": "user-789",
        "reelId": "reel-12345",
        "category": "nudity",
        "message": "This reel contains...",
        "status": "action_taken",
        "moderatorDecision": "Content removed for explicit nudity",
        "decisionAt": "2026-02-22T16:00:00Z",
        "createdAt": "2026-02-22T14:35:00Z"
      },
      {
        "id": 12340,
        "reportedUserId": "user-777",
        "messageId": "msg-5678",
        "category": "harassment",
        "message": "User sent threatening messages",
        "status": "submitted",
        "moderatorDecision": null,
        "decisionAt": null,
        "createdAt": "2026-02-21T10:00:00Z"
      }
    ],
    "nextCursor": 12340
  }
}
```

---

### Admin Endpoints

#### 4. GET /v1/admin/moderation/pending

**Purpose**: Admin lists pending moderation items (AI-flagged content)

**Auth**: JWT + Admin role required

**Query Parameters**:
- `cursor` (string, optional): ISO timestamp for pagination
- `limit` (number, default: 20): Max results per page

**Request**:
```http
GET /api/v1/admin/moderation/pending?limit=20 HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Response**:
```json
{
  "success": true,
  "message": "Pending moderation items retrieved",
  "data": {
    "items": [
      {
        "id": "uuid-12345",
        "mediaId": "reel-67890",
        "userId": "user-456",
        "status": "needs_review",
        "explicitScore": 65,
        "violenceScore": 30,
        "labels": ["Suggestive", "Revealing Clothes"],
        "rulesTriggered": [
          {
            "rule": "EXPLICIT_SOFT_FLAG",
            "reason": "Borderline explicit content (score 65)",
            "severity": "warning"
          }
        ],
        "createdAt": "2026-02-22T10:15:30Z"
      },
      {
        "id": "uuid-12346",
        "mediaId": "reel-67891",
        "userId": "user-457",
        "status": "needs_review",
        "explicitScore": null,
        "violenceScore": null,
        "labels": [],
        "rulesTriggered": [],
        "aiFailureReason": "Rekognition API timeout",
        "moderationFallbackTriggered": true,
        "createdAt": "2026-02-22T11:00:00Z"
      }
    ],
    "nextCursor": "2026-02-22T11:00:00Z"
  }
}
```

**Errors**:
- `403 Forbidden`: Not admin role

---

#### 5. POST /v1/admin/moderation/:id/approve

**Purpose**: Admin manually approves flagged content

**Auth**: JWT + Admin role required

**Request**:
```http
POST /api/v1/admin/moderation/uuid-12345/approve HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json

{
  "notes": "Content is artistic fashion photography, not explicit"
}
```

**Request Body Schema**:
```typescript
{
  notes?: string; // Optional explanation
}
```

**Response**:
```json
{
  "success": true,
  "message": "Moderation approved successfully"
}
```

**Side Effects**:
- Update `moderation_records.status = 'approved'`
- Set `final_decision_by = 'moderator'`
- Store `moderator_id` and `moderator_notes`
- Create audit event `STATUS_CHANGED`
- Send notification to content creator: "Your reel is live!"
- Make reel visible in feeds

**Errors**:
- `404 Not Found`: Moderation record not found
- `403 Forbidden`: Not admin role

---

#### 6. POST /v1/admin/moderation/:id/reject

**Purpose**: Admin manually rejects flagged content

**Auth**: JWT + Admin role required

**Request**:
```http
POST /api/v1/admin/moderation/uuid-12345/reject HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json

{
  "notes": "Explicit nudity violates community guidelines Section 2.3"
}
```

**Request Body Schema**:
```typescript
{
  notes: string; // Mandatory reason for rejection
}
```

**Response**:
```json
{
  "success": true,
  "message": "Moderation rejected successfully"
}
```

**Side Effects**:
- Update `moderation_records.status = 'rejected'`
- Set `final_decision_by = 'moderator'`
- Store `moderator_id` and `moderator_notes`
- Create audit event `STATUS_CHANGED`
- Send notification to content creator: "Your reel was removed: [reason]"
- Check for auto-ban (3 strikes in 24h)
- Hide reel from feeds

**Errors**:
- `400 Bad Request`: Notes required
- `404 Not Found`: Moderation record not found
- `403 Forbidden`: Not admin role

---

#### 7. GET /v1/admin/moderation/:id/audit

**Purpose**: Admin views full audit trail for moderation decision

**Auth**: JWT + Admin role required

**Request**:
```http
GET /api/v1/admin/moderation/uuid-12345/audit HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Response**:
```json
{
  "success": true,
  "message": "Audit log retrieved",
  "data": {
    "events": [
      {
        "event": "MODERATION_STARTED",
        "oldStatus": null,
        "newStatus": "pending",
        "payload": {
          "mediaId": "reel-67890",
          "userId": "user-456"
        },
        "actorId": null,
        "timestamp": "2026-02-22T10:15:30Z"
      },
      {
        "event": "AI_ANALYZED",
        "oldStatus": null,
        "newStatus": null,
        "payload": {
          "explicitScore": 65,
          "violenceScore": 30,
          "labels": ["Suggestive", "Revealing Clothes"],
          "responseTimeMs": 450
        },
        "actorId": null,
        "timestamp": "2026-02-22T10:15:31Z"
      },
      {
        "event": "RULES_EVALUATED",
        "oldStatus": null,
        "newStatus": null,
        "payload": {
          "decision": "needs_review",
          "rulesTriggered": [
            {
              "rule": "EXPLICIT_SOFT_FLAG",
              "reason": "Borderline explicit content (score 65)",
              "severity": "warning"
            }
          ]
        },
        "actorId": null,
        "timestamp": "2026-02-22T10:15:31Z"
      },
      {
        "event": "STATUS_CHANGED",
        "oldStatus": "pending",
        "newStatus": "needs_review",
        "payload": {
          "reason": "Borderline content requires human review"
        },
        "actorId": null,
        "timestamp": "2026-02-22T10:15:31Z"
      },
      {
        "event": "STATUS_CHANGED",
        "oldStatus": "needs_review",
        "newStatus": "approved",
        "payload": {
          "moderatorId": "admin-001",
          "notes": "Content is artistic fashion, not explicit"
        },
        "actorId": "admin-001",
        "timestamp": "2026-02-22T14:30:00Z"
      }
    ]
  }
}
```

**Errors**:
- `404 Not Found`: Moderation record not found
- `403 Forbidden`: Not admin role

---

#### 8. GET /v1/admin/reports

**Purpose**: Admin lists user-submitted reports with filters

**Auth**: JWT + Admin role required

**Query Parameters**:
- `status` (string, optional): Filter by status (submitted | under_review | action_taken | rejected)
- `category` (string, optional): Filter by category (spam | scam | nudity | violence | hate | harassment | copyright | impersonation | other)
- `isEscalated` (boolean, optional): Filter by escalation status
- `limit` (number, default: 20): Max results per page
- `cursor` (number, optional): Pagination cursor (report ID)

**Request**:
```http
GET /api/v1/admin/reports?status=submitted&isEscalated=true&limit=20 HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Response**:
```json
{
  "success": true,
  "message": "Reports retrieved successfully",
  "data": {
    "items": [
      {
        "id": 12346,
        "reporterId": "user-456",
        "reportedUserId": "user-789",
        "reelId": "reel-12345",
        "category": "nudity",
        "message": "This reel contains inappropriate sexual content...",
        "status": "submitted",
        "moderatorDecision": null,
        "moderatorId": null,
        "decisionAt": null,
        "isEscalated": true,
        "similarReportsCount": 6,
        "createdAt": "2026-02-22T14:35:00Z",
        "reporter": {
          "id": "user-456",
          "displayName": "Reporter Name",
          "username": "reporter_username"
        },
        "reportedUser": {
          "id": "user-789",
          "displayName": "Accused Name",
          "username": "accused_username"
        }
      }
    ],
    "nextCursor": 12346
  }
}
```

**Errors**:
- `403 Forbidden`: Not admin role

---

#### 9. POST /v1/admin/reports/:id/review

**Purpose**: Admin takes action on user report

**Auth**: JWT + Admin role required

**Request**:
```http
POST /api/v1/admin/reports/12346/review HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json

{
  "status": "action_taken",
  "moderatorDecision": "Content removed for explicit nudity. User warned."
}
```

**Request Body Schema**:
```typescript
{
  status: 'action_taken' | 'rejected';
  moderatorDecision: string; // Mandatory explanation
}
```

**Response**:
```json
{
  "success": true,
  "message": "Report reviewed successfully",
  "data": {
    "id": 12346,
    "status": "action_taken",
    "moderatorDecision": "Content removed for explicit nudity. User warned.",
    "moderatorId": "admin-001",
    "decisionAt": "2026-02-22T16:00:00Z"
  }
}
```

**Side Effects**:
- Update `reports.status = 'action_taken' | 'rejected'`
- Store `moderator_id`, `moderator_decision`, `decision_at`
- Send notification to reporter: "Action taken on your report" OR "Report dismissed"
- Send notification to accused user (if action_taken): "Your content was removed"

**Errors**:
- `400 Bad Request`: Report already reviewed, or moderatorDecision missing
- `404 Not Found`: Report not found
- `403 Forbidden`: Not admin role

---

#### 10. GET /v1/admin/reports/:id

**Purpose**: Admin views single report details

**Auth**: JWT + Admin role required

**Request**:
```http
GET /api/v1/admin/reports/12346 HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Response**:
```json
{
  "success": true,
  "message": "Report retrieved successfully",
  "data": {
    "id": 12346,
    "reporterId": "user-456",
    "reportedUserId": "user-789",
    "reelId": "reel-12345",
    "messageId": null,
    "reviewId": null,
    "profileId": null,
    "category": "nudity",
    "message": "This reel contains inappropriate sexual content that violates community guidelines.",
    "status": "action_taken",
    "moderatorDecision": "Content removed for explicit nudity. User warned.",
    "moderatorId": "admin-001",
    "decisionAt": "2026-02-22T16:00:00Z",
    "isEscalated": true,
    "similarReportsCount": 6,
    "createdAt": "2026-02-22T14:35:00Z",
    "updatedAt": "2026-02-22T16:00:00Z",
    "reporter": {
      "id": "user-456",
      "displayName": "Reporter Name",
      "username": "reporter_username",
      "email": "reporter@example.com"
    },
    "reportedUser": {
      "id": "user-789",
      "displayName": "Accused Name",
      "username": "accused_username",
      "email": "accused@example.com"
    }
  }
}
```

**Errors**:
- `404 Not Found`: Report not found
- `403 Forbidden`: Not admin role

---

## 🤖 AI Integration (AWS Rekognition)

### AWSRekognitionProvider Implementation

**File**: `apps/chefooz-apis/src/modules/moderation/providers/aws-rekognition.provider.ts`

```typescript
import { Injectable, Logger } from '@nestjs/common';
import {
  RekognitionClient,
  DetectModerationLabelsCommand,
  ModerationLabel,
} from '@aws-sdk/client-rekognition';

@Injectable()
export class AWSRekognitionProvider {
  private readonly logger = new Logger(AWSRekognitionProvider.name);
  private readonly client: RekognitionClient;

  constructor() {
    this.client = new RekognitionClient({
      region: process.env.AWS_REGION || 'us-east-1',
      credentials: {
        accessKeyId: process.env.AWS_ACCESS_KEY_ID || '',
        secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY || '',
      },
    });
  }

  /**
   * Analyze image/video frame for inappropriate content
   * @param s3Key - S3 object key (e.g., 'reels/thumbnails/abc123.jpg')
   * @param s3Bucket - S3 bucket name
   * @returns Normalized moderation result
   */
  async analyzeContent(
    s3Key: string,
    s3Bucket: string,
  ): Promise<{
    explicitScore: number;
    violenceScore: number;
    labels: string[];
    rawLabels: ModerationLabel[];
  }> {
    try {
      const command = new DetectModerationLabelsCommand({
        Image: {
          S3Object: {
            Bucket: s3Bucket,
            Name: s3Key,
          },
        },
        MinConfidence: 50, // Only return labels with 50%+ confidence
      });

      const response = await this.client.send(command);
      const labels = response.ModerationLabels || [];

      this.logger.debug(`Rekognition analyzed ${s3Key}: ${labels.length} labels found`);

      // Calculate category scores
      const explicitScore = this.calculateCategoryScore(labels, [
        'Explicit Nudity',
        'Nudity',
        'Sexual Activity',
        'Suggestive',
      ]);

      const violenceScore = this.calculateCategoryScore(labels, [
        'Violence',
        'Visually Disturbing',
        'Weapons',
        'Explosions and Blasts',
      ]);

      // Extract label names
      const labelNames = labels
        .filter((l: ModerationLabel) => l.Confidence && l.Confidence >= 60)
        .map((l: ModerationLabel) => l.Name || 'Unknown')
        .slice(0, 10); // Top 10 labels

      return {
        explicitScore,
        violenceScore,
        labels: labelNames,
        rawLabels: labels,
      };
    } catch (error) {
      this.logger.error(`Rekognition analysis failed for ${s3Key}:`, error);
      throw new Error(`AI moderation failed: ${error instanceof Error ? error.message : 'Unknown error'}`);
    }
  }

  /**
   * Calculate normalized score (0-100) for a category
   * Takes highest confidence label in category
   * @private
   */
  private calculateCategoryScore(
    labels: ModerationLabel[],
    categoryNames: string[],
  ): number {
    const categoryLabels = labels.filter((label: ModerationLabel) =>
      categoryNames.some((cat) =>
        (label.Name || '').toLowerCase().includes(cat.toLowerCase()),
      ),
    );

    if (categoryLabels.length === 0) {
      return 0;
    }

    // Take highest confidence label in category
    const maxConfidence = Math.max(
      ...categoryLabels.map((l: ModerationLabel) => l.Confidence || 0),
    );

    return Math.round(maxConfidence);
  }
}
```

### Rekognition API Details

**API**: `DetectModerationLabels`

**Input**:
```json
{
  "Image": {
    "S3Object": {
      "Bucket": "chefooz-media",
      "Name": "reels/thumbnails/reel-12345.jpg"
    }
  },
  "MinConfidence": 50
}
```

**Output Example**:
```json
{
  "ModerationLabels": [
    {
      "Confidence": 95.5,
      "Name": "Explicit Nudity",
      "ParentName": "Nudity"
    },
    {
      "Confidence": 78.3,
      "Name": "Suggestive",
      "ParentName": ""
    },
    {
      "Confidence": 65.2,
      "Name": "Revealing Clothes",
      "ParentName": "Suggestive"
    }
  ],
  "ModerationModelVersion": "6.0"
}
```

**Category Mapping**:

| Rekognition Label | Our Category | Score Calculation |
|-------------------|--------------|-------------------|
| Explicit Nudity | Explicit | Max confidence in category |
| Nudity | Explicit | Max confidence in category |
| Sexual Activity | Explicit | Max confidence in category |
| Suggestive | Explicit | Max confidence in category |
| Violence | Violence | Max confidence in category |
| Visually Disturbing | Violence | Max confidence in category |
| Weapons | Violence | Max confidence in category |
| Explosions and Blasts | Violence | Max confidence in category |

**Score Normalization**:
- Rekognition returns confidence 0-100
- We use max confidence within category as category score
- Example: Labels ["Explicit Nudity" (95.5), "Suggestive" (78.3)] → explicitScore = 95

---

## ⚙️ Business Rules Engine

### ModerationRulesService Implementation

**File**: `apps/chefooz-apis/src/modules/moderation/services/moderation-rules.service.ts`

```typescript
import { Injectable, Logger } from '@nestjs/common';
import { 
  getEnvConfig, 
  shouldAutoRejectExplicit, 
  shouldReviewExplicit, 
  shouldAutoRejectViolence, 
  shouldReviewViolence 
} from '@chefooz-app/domain';

@Injectable()
export class ModerationRulesService {
  private readonly logger = new Logger(ModerationRulesService.name);

  /**
   * Evaluate AI scores against business rules
   * @returns Decision and triggered rules
   */
  evaluateContent(
    explicitScore: number,
    violenceScore: number,
    labels: string[],
  ): {
    decision: 'approved' | 'rejected' | 'needs_review';
    rulesTriggered: Array<{ rule: string; reason: string; severity: string }>;
  } {
    const rulesTriggered: Array<{ rule: string; reason: string; severity: string }> = [];
    const envConfig = getEnvConfig();

    // Rule 1: Hard reject explicit content
    if (shouldAutoRejectExplicit(explicitScore, envConfig)) {
      rulesTriggered.push({
        rule: 'EXPLICIT_HARD_REJECT',
        reason: `Explicit content score ${explicitScore} exceeds threshold ${envConfig.moderation.explicitContentRejectThreshold}`,
        severity: 'critical',
      });
    }

    // Rule 2: Hard reject violence
    if (shouldAutoRejectViolence(violenceScore, envConfig)) {
      rulesTriggered.push({
        rule: 'VIOLENCE_HARD_REJECT',
        reason: `Violence score ${violenceScore} exceeds threshold ${envConfig.moderation.violenceRejectThreshold}`,
        severity: 'critical',
      });
    }

    // Rule 3: Soft flag borderline explicit content
    if (shouldReviewExplicit(explicitScore, envConfig)) {
      rulesTriggered.push({
        rule: 'EXPLICIT_SOFT_FLAG',
        reason: `Borderline explicit content (score ${explicitScore})`,
        severity: 'warning',
      });
    }

    // Rule 4: Soft flag moderate violence
    if (shouldReviewViolence(violenceScore, envConfig)) {
      rulesTriggered.push({
        rule: 'VIOLENCE_SOFT_FLAG',
        reason: `Moderate violence detected (score ${violenceScore})`,
        severity: 'warning',
      });
    }

    // Rule 5: Check for specific prohibited labels
    const prohibitedLabels = ['Weapons', 'Drugs', 'Hate Symbols', 'Graphic Violence'];
    const foundProhibited = labels.filter((label) =>
      prohibitedLabels.some((prohibited) => label.toLowerCase().includes(prohibited.toLowerCase())),
    );

    if (foundProhibited.length > 0) {
      rulesTriggered.push({
        rule: 'PROHIBITED_CONTENT',
        reason: `Prohibited content detected: ${foundProhibited.join(', ')}`,
        severity: 'critical',
      });
    }

    // Determine final decision
    const hasCritical = rulesTriggered.some((r) => r.severity === 'critical');
    const hasWarning = rulesTriggered.some((r) => r.severity === 'warning');

    let decision: 'approved' | 'rejected' | 'needs_review';

    if (hasCritical) {
      decision = 'rejected';
    } else if (hasWarning) {
      decision = 'needs_review';
    } else {
      decision = 'approved';
    }

    this.logger.debug(
      `Rules evaluation: ${decision} (${rulesTriggered.length} rules triggered)`,
    );

    return { decision, rulesTriggered };
  }

  /**
   * Check if user should be auto-banned for repeat violations
   * @param recentViolationsCount - Number of rejected content in last 24h
   * @returns true if user should be banned
   */
  shouldAutoBan(recentViolationsCount: number): boolean {
    const threshold = 3;
    return recentViolationsCount >= threshold;
  }
}
```

### Environment-Specific Thresholds

**Production** (`libs/domain/src/lib/config/production.config.ts`):
```typescript
export const productionConfig = {
  moderation: {
    explicitContentRejectThreshold: 80,  // Auto-reject if ≥ 80
    explicitContentReviewThreshold: 50,  // Human review if 50-79
    violenceRejectThreshold: 80,          // Auto-reject if ≥ 80
    violenceReviewThreshold: 50,          // Human review if 50-79
  },
};
```

**Staging** (`libs/domain/src/lib/config/staging.config.ts`):
```typescript
export const stagingConfig = {
  moderation: {
    explicitContentRejectThreshold: 70,  // More aggressive for testing
    explicitContentReviewThreshold: 40,  // Catch more borderline cases
    violenceRejectThreshold: 70,
    violenceReviewThreshold: 40,
  },
};
```

### 5 Business Rules

**Rule 1: EXPLICIT_HARD_REJECT** (Critical Severity)
```typescript
if (explicitScore >= envConfig.moderation.explicitContentRejectThreshold) {
  // Production: ≥ 80, Staging: ≥ 70
  rulesTriggered.push({
    rule: 'EXPLICIT_HARD_REJECT',
    reason: `Explicit content score ${explicitScore} exceeds threshold ${threshold}`,
    severity: 'critical',
  });
}
```
**Action**: Auto-reject, notify user, check auto-ban

**Rule 2: VIOLENCE_HARD_REJECT** (Critical Severity)
```typescript
if (violenceScore >= envConfig.moderation.violenceRejectThreshold) {
  // Production: ≥ 80, Staging: ≥ 70
  rulesTriggered.push({
    rule: 'VIOLENCE_HARD_REJECT',
    reason: `Violence score ${violenceScore} exceeds threshold ${threshold}`,
    severity: 'critical',
  });
}
```
**Action**: Auto-reject, notify user, check auto-ban

**Rule 3: EXPLICIT_SOFT_FLAG** (Warning Severity)
```typescript
if (explicitScore >= envConfig.moderation.explicitContentReviewThreshold &&
    explicitScore < envConfig.moderation.explicitContentRejectThreshold) {
  // Production: 50-79, Staging: 40-69
  rulesTriggered.push({
    rule: 'EXPLICIT_SOFT_FLAG',
    reason: `Borderline explicit content (score ${explicitScore})`,
    severity: 'warning',
  });
}
```
**Action**: Route to human review queue

**Rule 4: VIOLENCE_SOFT_FLAG** (Warning Severity)
```typescript
if (violenceScore >= envConfig.moderation.violenceReviewThreshold &&
    violenceScore < envConfig.moderation.violenceRejectThreshold) {
  // Production: 50-79, Staging: 40-69
  rulesTriggered.push({
    rule: 'VIOLENCE_SOFT_FLAG',
    reason: `Moderate violence detected (score ${violenceScore})`,
    severity: 'warning',
  });
}
```
**Action**: Route to human review queue

**Rule 5: PROHIBITED_CONTENT** (Critical Severity)
```typescript
const prohibitedLabels = ['Weapons', 'Drugs', 'Hate Symbols', 'Graphic Violence'];
const foundProhibited = labels.filter((label) =>
  prohibitedLabels.some((prohibited) => label.toLowerCase().includes(prohibited.toLowerCase())),
);

if (foundProhibited.length > 0) {
  rulesTriggered.push({
    rule: 'PROHIBITED_CONTENT',
    reason: `Prohibited content detected: ${foundProhibited.join(', ')}`,
    severity: 'critical',
  });
}
```
**Action**: Auto-reject regardless of score

---

## 🔄 Moderation Pipeline Implementation

### ModerationService: startModeration()

**Entry Point**: Called by MediaService after MediaConvert job completes

```typescript
async startModeration(
  mediaId: string,
  userId: string,
  s3ThumbnailKey: string,
): Promise<ModerationRecord> {
  this.logger.log(`Starting moderation for media ${mediaId}`);

  // 1. Create pending record
  const record = this.moderationRepo.create({
    mediaId,
    userId,
    status: 'pending',
  });

  await this.moderationRepo.save(record);
  await this.createAudit(record.id, 'MODERATION_STARTED', null, 'pending', {
    mediaId,
    userId,
  });

  // 2. Run AI + Rules pipeline asynchronously (don't block MediaService)
  setImmediate(() => this.runModerationPipeline(record, s3ThumbnailKey, userId));

  return record;
}
```

**Key Design Decisions**:
- **Async Execution**: `setImmediate()` runs pipeline in next event loop tick (non-blocking)
- **MediaService doesn't wait**: Upload completes immediately, moderation runs in background
- **Audit Trail Start**: Log `MODERATION_STARTED` event before async work begins

---

### ModerationService: runModerationPipeline() (Private)

**6-Step Pipeline**:

```typescript
private async runModerationPipeline(
  record: ModerationRecord,
  s3ThumbnailKey: string,
  userId: string,
): Promise<void> {
  const startTime = Date.now();
  
  try {
    // STEP 1: Run AI analysis
    const s3Bucket = process.env.AWS_S3_BUCKET || 'chefooz-media';
    const aiResult = await this.aiProvider.analyzeContent(s3ThumbnailKey, s3Bucket);
    
    const responseTime = Date.now() - startTime;

    record.aiScoreExplicit = aiResult.explicitScore;
    record.aiScoreViolence = aiResult.violenceScore;
    record.aiLabels = aiResult.rawLabels;
    record.rekognitionResponseTime = responseTime;

    await this.moderationRepo.save(record);
    await this.createAudit(record.id, 'AI_ANALYZED', null, null, {
      explicitScore: aiResult.explicitScore,
      violenceScore: aiResult.violenceScore,
      labels: aiResult.labels,
      responseTimeMs: responseTime,
    });

    this.logger.debug(
      `AI analysis complete (${responseTime}ms): explicit=${aiResult.explicitScore}, violence=${aiResult.violenceScore}`,
    );

    // STEP 2: Apply business rules
    const { decision, rulesTriggered } = this.rulesService.evaluateContent(
      aiResult.explicitScore,
      aiResult.violenceScore,
      aiResult.labels,
    );

    record.rulesTriggered = rulesTriggered;
    record.finalDecisionBy = 'ai';

    await this.createAudit(record.id, 'RULES_EVALUATED', null, null, {
      decision,
      rulesTriggered,
    });

    // STEP 3-6: Update status based on decision
    if (decision === 'rejected') {
      // Auto-reject
      record.status = 'rejected';
      await this.moderationRepo.save(record);
      await this.createAudit(record.id, 'STATUS_CHANGED', 'pending', 'rejected', {
        reason: 'AI auto-reject',
      });

      await this.notificationDispatcher.send(userId, 'moderation.rejected', {
        mediaId: record.mediaId,
        reason: 'Content violated community guidelines',
      });

      await this.checkForAutoBan(userId);
      
    } else if (decision === 'needs_review') {
      // Human review required
      record.status = 'needs_review';
      await this.moderationRepo.save(record);
      await this.createAudit(record.id, 'STATUS_CHANGED', 'pending', 'needs_review', {
        reason: 'Borderline content requires human review',
      });

      await this.notificationDispatcher.send(userId, 'moderation.under_review', {
        mediaId: record.mediaId,
      });
      
    } else {
      // Auto-approve
      record.status = 'approved';
      await this.moderationRepo.save(record);
      await this.createAudit(record.id, 'STATUS_CHANGED', 'pending', 'approved', {
        reason: 'AI auto-approve',
      });

      await this.notificationDispatcher.send(userId, 'moderation.approved', {
        mediaId: record.mediaId,
      });
    }

    this.logger.log(`Moderation complete for ${record.mediaId}: ${record.status}`);
    
  } catch (error) {
    // ✅ GRACEFUL DEGRADATION: Route to human review
    await this.handleAIFailure(record, userId, error);
  }
}
```

**Performance**:
- **Typical Duration**: 1.5 seconds (300-600ms Rekognition + 100ms rules + 900ms DB/notifications)
- **Target SLA**: < 2 seconds
- **Concurrency**: Can handle 100+ simultaneous moderations (Rekognition rate limit: 5 TPS default, can request increase)

---

### ModerationService: handleAIFailure() (Graceful Degradation)

```typescript
private async handleAIFailure(
  record: ModerationRecord,
  userId: string,
  error: any,
): Promise<void> {
  const errorMessage = error instanceof Error ? error.message : 'Unknown error';
  
  this.logger.error(
    `⚠️  AI moderation failed for ${record.mediaId}, routing to human review:`,
    errorMessage,
  );

  // Update record with failure details
  record.status = 'needs_review';
  record.aiFailureReason = errorMessage;
  record.moderationFallbackTriggered = true;
  record.finalDecisionBy = null; // No AI decision available

  await this.moderationRepo.save(record);
  await this.createAudit(record.id, 'AI_FAILED', 'pending', 'needs_review', {
    error: errorMessage,
    fallbackAction: 'human_review_required',
    timestamp: new Date().toISOString(),
  });

  // Alert admin dashboard (async, don't block)
  this.alertAdminDashboard({
    type: 'moderation_ai_failure',
    mediaId: record.mediaId,
    userId: record.userId,
    error: errorMessage,
    timestamp: new Date().toISOString(),
  }).catch((err) => {
    this.logger.error('Failed to alert admin dashboard:', err);
  });

  // Notify user with transparency
  await this.notificationDispatcher.send(userId, 'moderation.under_review', {
    mediaId: record.mediaId,
    reason: 'Your content is being reviewed by our team',
  });

  this.logger.log(`✅ Media ${record.mediaId} routed to human review (AI failure)`);
}
```

**Why This Matters**:
- **Zero upload blocking**: External service failure doesn't impact user experience
- **Admin visibility**: Dashboard alerts enable proactive monitoring
- **Compliance maintained**: All content eventually reviewed by humans
- **User trust preserved**: Transparent "under review" status (not scary error)

---

### ModerationService: checkForAutoBan()

```typescript
private async checkForAutoBan(userId: string): Promise<void> {
  const oneDayAgo = new Date(Date.now() - 24 * 60 * 60 * 1000);

  const recentViolations = await this.moderationRepo.count({
    where: {
      userId,
      status: 'rejected',
      createdAt: LessThan(oneDayAgo),
    },
  });

  if (this.rulesService.shouldAutoBan(recentViolations)) {
    this.logger.warn(`User ${userId} exceeded violation threshold: ${recentViolations} violations`);

    // TODO: Integrate with user.service to ban user
    // await this.userService.banUser(userId, 'Repeated content violations');

    await this.notificationDispatcher.send(userId, 'account.suspended', {
      reason: 'Multiple community guideline violations',
    });
  }
}
```

**Auto-Ban Logic**:
- **Threshold**: 3 rejected content in 24 hours
- **Action**: Ban user account (TODO: integrate with user.service)
- **Notification**: "Account suspended for repeat violations"
- **Why**: Prevents repeat violators from continuing to upload harmful content

---

## 📋 Reporting System Implementation

### ReportService: createReport()

```typescript
async createReport(userId: string, dto: CreateReportDto): Promise<Report> {
  this.logger.log(`User ${userId} creating report for category: ${dto.category}`);

  // Validation 1: at least one target must be specified
  const hasTarget = dto.reelId || dto.messageId || dto.reviewId || dto.profileId;
  if (!hasTarget) {
    throw new BadRequestException('At least one target must be specified');
  }

  // Validation 2: cannot report yourself
  if (dto.reportedUserId && dto.reportedUserId === userId) {
    throw new BadRequestException('You cannot report yourself');
  }

  // Anti-spam: check for duplicate reports within 24h
  const oneDayAgo = new Date(Date.now() - 24 * 60 * 60 * 1000);
  const where: any = {
    reporterId: userId,
    createdAt: MoreThan(oneDayAgo),
  };
  if (dto.reelId) where.reelId = dto.reelId;
  if (dto.messageId) where.messageId = dto.messageId;
  if (dto.reviewId) where.reviewId = dto.reviewId;
  if (dto.profileId) where.profileId = dto.profileId;

  const existingReport = await this.reportRepo.findOne({ where });

  if (existingReport) {
    throw new BadRequestException('You have already reported this content within the last 24 hours');
  }

  // Count similar reports for escalation
  const similarReportsCount = await this.countSimilarReports(dto);

  // Create report
  const report = this.reportRepo.create({
    reporterId: userId,
    reportedUserId: dto.reportedUserId || null,
    reelId: dto.reelId || null,
    messageId: dto.messageId || null,
    reviewId: dto.reviewId || null,
    profileId: dto.profileId || null,
    category: dto.category,
    message: dto.message,
    status: 'submitted',
    similarReportsCount,
    isEscalated: similarReportsCount >= 5,
  });

  const savedReport = await this.reportRepo.save(report);

  // Send notification to reporter
  await this.notificationDispatcher.send(userId, 'report.submitted', {
    reportId: savedReport.id,
    category: dto.category,
  });

  // If escalated, log high priority
  if (report.isEscalated) {
    this.logger.warn(`🚨 Report ${savedReport.id} escalated: ${similarReportsCount} similar reports`);
  }

  this.logger.log(`✅ Report ${savedReport.id} created by user ${userId}`);

  return savedReport;
}
```

### ReportService: countSimilarReports()

```typescript
private async countSimilarReports(dto: CreateReportDto): Promise<number> {
  const oneHourAgo = new Date(Date.now() - 60 * 60 * 1000);

  const where: any = {
    createdAt: MoreThan(oneHourAgo),
  };

  if (dto.reelId) where.reelId = dto.reelId;
  if (dto.messageId) where.messageId = dto.messageId;
  if (dto.reviewId) where.reviewId = dto.reviewId;
  if (dto.profileId) where.profileId = dto.profileId;

  return await this.reportRepo.count({ where });
}
```

**Auto-Escalation Logic**:
- Count reports for same target (reelId, messageId, reviewId, OR profileId) in last 1 hour
- If count ≥ 5, set `isEscalated = true`
- Log warning with 🚨 emoji for admin visibility

---

## 👨‍💼 Admin Dashboard Implementation

### ModerationService: getPendingReviews()

```typescript
async getPendingReviews(
  cursor?: string,
  limit = 20,
): Promise<{ items: ModerationRecord[]; nextCursor: string | null }> {
  const queryBuilder = this.moderationRepo
    .createQueryBuilder('moderation')
    .where('moderation.status IN (:...statuses)', {
      statuses: ['needs_review', 'ai_flagged'],
    })
    .orderBy('moderation.createdAt', 'DESC')
    .take(limit + 1);

  if (cursor) {
    queryBuilder.andWhere('moderation.createdAt < :cursor', { cursor: new Date(cursor) });
  }

  const items = await queryBuilder.getMany();
  const hasMore = items.length > limit;
  const results = hasMore ? items.slice(0, limit) : items;

  const nextCursor =
    hasMore && results.length > 0 ? results[results.length - 1].createdAt.toISOString() : null;

  return { items: results, nextCursor };
}
```

**Cursor-Based Pagination**:
- Use `createdAt` as cursor (ISO timestamp)
- Fetch `limit + 1` to check if more items exist
- Return `nextCursor` if `hasMore = true`

---

### ModerationService: approveModerationManually()

```typescript
async approveModerationManually(
  moderationId: string,
  moderatorId: string,
  notes?: string,
): Promise<void> {
  const record = await this.moderationRepo.findOne({ where: { id: moderationId } });

  if (!record) {
    throw new NotFoundException('Moderation record not found');
  }

  const oldStatus = record.status;
  record.status = 'approved';
  record.finalDecisionBy = 'moderator';
  record.moderatorId = moderatorId;
  record.moderatorNotes = notes || null;

  await this.moderationRepo.save(record);
  await this.createAudit(moderationId, 'STATUS_CHANGED', oldStatus, 'approved', {
    moderatorId,
    notes,
  });

  // Notify user
  await this.notificationDispatcher.send(record.userId, 'moderation.approved', {
    mediaId: record.mediaId,
  });

  this.logger.log(`Moderation ${moderationId} approved by moderator ${moderatorId}`);
}
```

---

### ModerationService: rejectModerationManually()

```typescript
async rejectModerationManually(
  moderationId: string,
  moderatorId: string,
  notes: string,
): Promise<void> {
  const record = await this.moderationRepo.findOne({ where: { id: moderationId } });

  if (!record) {
    throw new NotFoundException('Moderation record not found');
  }

  const oldStatus = record.status;
  record.status = 'rejected';
  record.finalDecisionBy = 'moderator';
  record.moderatorId = moderatorId;
  record.moderatorNotes = notes;

  await this.moderationRepo.save(record);
  await this.createAudit(moderationId, 'STATUS_CHANGED', oldStatus, 'rejected', {
    moderatorId,
    notes,
  });

  // Notify user
  await this.notificationDispatcher.send(record.userId, 'moderation.rejected', {
    mediaId: record.mediaId,
    reason: notes,
  });

  // Check for auto-ban
  await this.checkForAutoBan(record.userId);

  this.logger.log(`Moderation ${moderationId} rejected by moderator ${moderatorId}`);
}
```

---

## 🛡️ Graceful Failure Handling

### Why Graceful Degradation Matters

**Problem**: AWS Rekognition is an external service that can fail:
- API timeouts (> 30 seconds)
- Rate limit exceeded (5 TPS default)
- Service outage (rare but possible)
- Network errors
- Invalid credentials

**Bad Approach** (Blocking):
```typescript
try {
  const aiResult = await rekognition.analyzeContent(s3Key, s3Bucket);
  // ... normal flow
} catch (error) {
  throw new ServiceUnavailableException('AI moderation failed. Please try again later.');
}
```
**Problem**: User upload fails with cryptic error, bad UX

**Good Approach** (Graceful Degradation):
```typescript
try {
  const aiResult = await rekognition.analyzeContent(s3Key, s3Bucket);
  // ... normal flow
} catch (error) {
  // Route to human review (don't block upload)
  record.status = 'needs_review';
  record.aiFailureReason = error.message;
  record.moderationFallbackTriggered = true;
  
  await alertAdminDashboard({ type: 'ai_failure', ... });
  await notificationDispatcher.send(userId, 'moderation.under_review', { ... });
}
```
**Result**: User upload succeeds, content reviewed by humans, admin alerted for monitoring

---

### Monitoring AI Failures

**CloudWatch Metric**:
```typescript
// In alertAdminDashboard()
await cloudWatch.putMetricData({
  Namespace: 'Chefooz/Moderation',
  MetricData: [{
    MetricName: 'AIFailureRate',
    Value: 1,
    Unit: 'Count',
    Timestamp: new Date(),
  }],
});
```

**Alarm Configuration**:
- Trigger: AIFailureRate > 5% in 1 hour (e.g., > 5 failures per 100 uploads)
- Action: Send PagerDuty alert to on-call engineer
- Reason: Potential Rekognition outage or quota exhaustion

---

## 🧪 Testing Strategy

### Unit Tests

**ModerationRulesService Tests**:
```typescript
describe('ModerationRulesService', () => {
  let service: ModerationRulesService;

  beforeEach(() => {
    service = new ModerationRulesService();
  });

  describe('evaluateContent', () => {
    it('should auto-reject explicit content (score ≥ 80)', () => {
      const result = service.evaluateContent(85, 20, []);
      
      expect(result.decision).toBe('rejected');
      expect(result.rulesTriggered).toHaveLength(1);
      expect(result.rulesTriggered[0].rule).toBe('EXPLICIT_HARD_REJECT');
      expect(result.rulesTriggered[0].severity).toBe('critical');
    });

    it('should flag borderline explicit content for review (score 50-79)', () => {
      const result = service.evaluateContent(65, 20, []);
      
      expect(result.decision).toBe('needs_review');
      expect(result.rulesTriggered).toHaveLength(1);
      expect(result.rulesTriggered[0].rule).toBe('EXPLICIT_SOFT_FLAG');
      expect(result.rulesTriggered[0].severity).toBe('warning');
    });

    it('should auto-approve clean content (score < 50)', () => {
      const result = service.evaluateContent(20, 20, []);
      
      expect(result.decision).toBe('approved');
      expect(result.rulesTriggered).toHaveLength(0);
    });

    it('should auto-reject prohibited labels regardless of score', () => {
      const result = service.evaluateContent(30, 30, ['Weapons', 'Drugs']);
      
      expect(result.decision).toBe('rejected');
      expect(result.rulesTriggered).toHaveLength(1);
      expect(result.rulesTriggered[0].rule).toBe('PROHIBITED_CONTENT');
    });
  });

  describe('shouldAutoBan', () => {
    it('should return true if 3+ violations in 24h', () => {
      expect(service.shouldAutoBan(3)).toBe(true);
      expect(service.shouldAutoBan(5)).toBe(true);
    });

    it('should return false if < 3 violations', () => {
      expect(service.shouldAutoBan(0)).toBe(false);
      expect(service.shouldAutoBan(1)).toBe(false);
      expect(service.shouldAutoBan(2)).toBe(false);
    });
  });
});
```

---

### Integration Tests

**ModerationService Tests**:
```typescript
describe('ModerationService (Integration)', () => {
  let service: ModerationService;
  let repo: Repository<ModerationRecord>;
  let aiProvider: AWSRekognitionProvider;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        ModerationService,
        { provide: AWSRekognitionProvider, useValue: mockAIProvider },
        // ... other providers
      ],
    }).compile();

    service = module.get<ModerationService>(ModerationService);
    repo = module.get<Repository<ModerationRecord>>(getRepositoryToken(ModerationRecord));
  });

  describe('startModeration', () => {
    it('should create pending record and run AI pipeline', async () => {
      const record = await service.startModeration('reel-123', 'user-456', 's3-key');
      
      expect(record.mediaId).toBe('reel-123');
      expect(record.status).toBe('pending');
      
      // Wait for async pipeline
      await new Promise((resolve) => setTimeout(resolve, 2000));
      
      const updated = await repo.findOne({ where: { id: record.id } });
      expect(updated.status).not.toBe('pending'); // Should be approved/rejected/needs_review
    });
  });

  describe('Graceful AI failure', () => {
    it('should route to human review if AI fails', async () => {
      // Mock AI provider to throw error
      jest.spyOn(aiProvider, 'analyzeContent').mockRejectedValueOnce(
        new Error('Rekognition API timeout'),
      );

      const record = await service.startModeration('reel-123', 'user-456', 's3-key');
      
      // Wait for async pipeline
      await new Promise((resolve) => setTimeout(resolve, 2000));
      
      const updated = await repo.findOne({ where: { id: record.id } });
      expect(updated.status).toBe('needs_review');
      expect(updated.aiFailureReason).toBe('Rekognition API timeout');
      expect(updated.moderationFallbackTriggered).toBe(true);
    });
  });
});
```

---

### E2E Tests

**Admin Moderation Flow**:
```typescript
describe('Admin Moderation (E2E)', () => {
  it('should allow admin to approve flagged content', async () => {
    // 1. Create moderation record
    const record = await createModerationRecord({
      status: 'needs_review',
      explicitScore: 65,
    });

    // 2. Admin approves
    const response = await request(app.getHttpServer())
      .post(`/v1/admin/moderation/${record.id}/approve`)
      .set('Authorization', `Bearer ${adminToken}`)
      .send({ notes: 'Content is artistic, not explicit' })
      .expect(200);

    expect(response.body.success).toBe(true);

    // 3. Verify status updated
    const updated = await repo.findOne({ where: { id: record.id } });
    expect(updated.status).toBe('approved');
    expect(updated.finalDecisionBy).toBe('moderator');
    expect(updated.moderatorNotes).toBe('Content is artistic, not explicit');
  });

  it('should require admin role', async () => {
    const record = await createModerationRecord({ status: 'needs_review' });

    await request(app.getHttpServer())
      .post(`/v1/admin/moderation/${record.id}/approve`)
      .set('Authorization', `Bearer ${userToken}`) // Non-admin token
      .send({ notes: 'Test' })
      .expect(403);
  });
});
```

---

## ⚡ Performance Optimization

### Database Indexes

**Critical Indexes**:
```sql
-- moderation_records
CREATE INDEX idx_moderation_records_media_id ON moderation_records(media_id);
CREATE INDEX idx_moderation_records_user_id ON moderation_records(user_id);
CREATE INDEX idx_moderation_records_status ON moderation_records(status);
CREATE INDEX idx_moderation_records_created_at ON moderation_records(created_at);

-- reports
CREATE INDEX idx_reports_reporter_id ON reports(reporter_id);
CREATE INDEX idx_reports_reported_user_id ON reports(reported_user_id);
CREATE INDEX idx_reports_category ON reports(category);
CREATE INDEX idx_reports_status ON reports(status);
CREATE INDEX idx_reports_reel_id ON reports(reel_id) WHERE reel_id IS NOT NULL; -- Sparse index
CREATE INDEX idx_reports_created_at ON reports(created_at);

-- moderation_audit
CREATE INDEX idx_moderation_audit_record_id ON moderation_audit(moderation_record_id);
CREATE INDEX idx_moderation_audit_timestamp ON moderation_audit(timestamp);
```

**Why These Indexes**:
- `media_id`: User checks moderation status of their reel (high frequency)
- `user_id`: Admin checks all moderation records for a user
- `status`: Admin filters by `needs_review` + `ai_flagged` (queue view)
- `created_at`: Cursor-based pagination in admin dashboard
- `category`: Admin filters reports by violation type
- `reel_id` (sparse): Only index non-null values (saves space)

---

### Caching Strategy

**Cache Moderation Results** (Redis):
```typescript
// In getModerationResult()
const cacheKey = `moderation:${mediaId}`;
const cached = await redis.get(cacheKey);

if (cached) {
  return JSON.parse(cached);
}

const record = await moderationRepo.findOne({ where: { mediaId, userId } });

if (record) {
  await redis.set(cacheKey, JSON.stringify(record), 'EX', 3600); // Cache for 1 hour
}

return record;
```

**Why Cache**:
- User may check moderation status multiple times (e.g., refresh app)
- Moderation result rarely changes after initial decision
- Reduces DB load for high-traffic reels

**Cache Invalidation**:
- When admin approves/rejects (manual review)
- TTL expires (1 hour)

---

### Async Processing

**setImmediate() vs Promise**:
```typescript
// ❌ Bad: Blocks MediaService until moderation completes
await this.runModerationPipeline(record, s3ThumbnailKey, userId);

// ✅ Good: Returns immediately, moderation runs in background
setImmediate(() => this.runModerationPipeline(record, s3ThumbnailKey, userId));
```

**Why `setImmediate()`**:
- MediaService returns immediately (upload completes)
- Moderation runs in next event loop tick (non-blocking)
- User doesn't wait for AI analysis (better UX)

---

## 🔐 Security Considerations

### 1. Admin-Only Endpoints

**RolesGuard**:
```typescript
@UseGuards(JwtAuthGuard, RolesGuard)
@Roles('admin')
async getPendingReviews() {
  // Admin-only endpoint
}
```

**Why**:
- Only admins can view moderation queue
- Only admins can approve/reject content
- Prevents unauthorized access to sensitive data

---

### 2. User Cannot Report Themselves

**Validation**:
```typescript
if (dto.reportedUserId && dto.reportedUserId === userId) {
  throw new BadRequestException('You cannot report yourself');
}
```

**Why**: Prevents abuse of reporting system

---

### 3. Anti-Spam (24h Duplicate Prevention)

**Validation**:
```typescript
const existingReport = await reportRepo.findOne({
  where: {
    reporterId: userId,
    reelId: dto.reelId,
    createdAt: MoreThan(oneDayAgo),
  },
});

if (existingReport) {
  throw new BadRequestException('You have already reported this content within the last 24 hours');
}
```

**Why**: Prevents report spam by single user

---

### 4. Audit Trail (Immutable Logs)

**Compliance Requirement**:
- Every moderation decision logged in `moderation_audit`
- WHO made decision (actorId)
- WHEN decision made (timestamp)
- WHAT changed (oldStatus → newStatus)
- WHY changed (payload with reason)

**Why**:
- IT Act Section 79 requires action logs
- DMCA safe harbor compliance
- Moderator accountability

---

**[TECHNICAL_GUIDE_COMPLETE ✅]**

**Module**: Moderation + Report  
**Lines**: ~18,500  
**Coverage**: Complete implementation guide with AI integration, graceful failure handling, admin dashboard, and testing strategy

---

## Next Steps

**QA Test Cases**: 40+ test scenarios covering:
1. AI moderation pipeline (auto-approve, auto-reject, needs review, AI failure)
2. Business rules (5 rules with thresholds)
3. Reporting system (anti-spam, escalation, admin actions)
4. Admin dashboard (approve, reject, audit trail)
5. Edge cases (concurrent uploads, race conditions, DB failures)
6. Performance tests (1000+ concurrent moderations)

---

## Implementation Update — March 2026

**Last Updated**: 2026-03-14

---

## Critical Constraints — Appeal Flow

### `ModerationResult.id` is the Appeal Identifier

The appeal endpoint `POST /v1/moderation/:id/appeal` takes the **`ModerationRecord` postgres primary key**, NOT the `mediaId`.

This means:
- `getMyModerationResult` (`GET /v1/moderation/my/:mediaId`) **must** return the `id` field in its response
- The `ModerationResult` shared type (`libs/types/src/lib/moderation.types.ts`) **must** include `id: string`
- The mobile screen must NOT attempt to submit an appeal without `moderation.id` being truthy

**Root cause of TC-MOD-BUG-007:** The `id` field was absent from both the type and the controller response. The screen used `(moderation as any).id` which silently resolved to `undefined`, causing the appeal guard `if (!moderationId) return;` to short-circuit before the API call every time.

### `ModerationResult` Response Shape (controller mapping)

```ts
data: {
  id: result.id,           // ← REQUIRED for appeal submission
  status: result.status,
  contentType: result.contentType,
  explicitScore: result.aiScoreExplicit,
  violenceScore: result.aiScoreViolence,
  labels: /* .Name extracted from Rekognition objects */,
  rulesTriggered: result.rulesTriggered || [],
  moderatorNotes: result.moderatorNotes,
  decidedAt: ...,
  createdAt: ...,
  updatedAt: ...,
}
```

If the `id` field is ever omitted from this mapping again, appeal submission will silently fail — add a unit test for this endpoint.

### Two Separate Appeal Flows — Do Not Confuse

There are **two distinct appeal systems** in Chefooz with separate clients, hooks, and admin pages:

| Dimension | Report Appeals | Content-Moderation Appeals |
|---|---|---|
| **What it covers** | User-reported content (reports against reels/users/comments) | AI/admin rejection decisions (moderation records) |
| **Backend controller** | `apps/chefooz-apis/src/modules/appeal/appeal.controller.ts` | `apps/chefooz-apis/src/modules/moderation/moderation.controller.ts` |
| **Backend routes** | `POST /v1/appeal`, `GET /v1/admin/appeals` | `POST /v1/moderation/:id/appeal`, `GET /v1/moderation/appeals` |
| **API client file** | `libs/api-client/src/lib/appeal.client.ts` | `libs/api-client/src/lib/clients/moderation.client.ts` |
| **Hook file** | `libs/api-client/src/lib/useAppeal.ts` | `libs/api-client/src/lib/hooks/moderation/useModeration.ts` |
| **Admin page** | `/dashboard/moderation/appeals` | `/dashboard/moderation/content-appeals` |
| **Entity** | `Appeal` (PostgreSQL, `appeal` module) | `ModerationAppeal` (PostgreSQL, `moderation` module) |
| **Cursor type** | `number \| null` (integer PK cursor) | `string \| null` (ISO date cursor) |

**Critical constraint:** Both clients MUST use `apiClient` from `libs/api-client/src/lib/axios-config.ts` — NOT raw `axios`. The `apiClient` interceptor injects the JWT `Authorization` header and normalises `/v1/` URLs to `/api/v1/`. Using raw `axios` results in 401 on all admin-guarded endpoints. (See TC-MOD-BUG-014 — this was the root cause of report appeals showing empty.)

### Mobile Screen Responsive Contract

`apps/chefooz-app/src/app/moderation/[mediaId].tsx` requires:
- Root: `SafeAreaView` — prevents notch overlap
- StatusBar: `expo-status-bar`-compatible style based on `theme.isDark`
- Appeal form: wrapped in `KeyboardAvoidingView` with `behavior={Platform.OS === 'ios' ? 'padding' : 'height'}` — prevents keyboard covering `TextInput`
- All colors: via `theme.colors.*` inline — never hardcoded hex values

### Architecture Changes

#### Bull Queue Integration
```
startModeration(mediaId, userId, s3Key, contentType)
  → creates ModerationRecord (status: pending)
  → syncMongoModerationStatus(mediaId, 'pending', contentType)  ← MongoDB shadow-ban sync
  → moderationQueue.add({ moderationRecordId, s3Key, s3Bucket, contentType, userId })
        ↓
ModerationProcessor.process() [concurrency: 2, attempts: 3, exponential backoff 5s]
  → runModerationPipelineById(...)
  → contentType === 'reel' → analyzeVideo() [Rekognition Video, polls 2s/30s max]
  → other → analyzeContent() [Rekognition DetectModerationLabels]
  → apply business rules → approve/reject/needsReview
  → syncMongoModerationStatus(mediaId, newStatus, contentType)
```

#### Shadow-Ban Feed Filter (feed.service.ts)
```typescript
// Public feed — hide rejected content
filter.$or = [
  { moderationStatus: { $in: ['approved', 'pending', 'reviewing'] } },
  { moderationStatus: { $exists: false } },
];
// Owner's own feed — they can still see their own rejected content
filter.$or = [...above..., { userId: query.userId }];
```

#### JWT Account Status Enforcement (jwt.strategy.ts)
```typescript
if (user.accountStatus === 'suspended' || user.accountStatus === 'banned') {
  throw new UnauthorizedException(`Account is ${user.accountStatus}`);
}
```

### New Entities

#### `moderation_appeals` table
| Column | Type | Notes |
|--------|------|-------|
| id | uuid | PK |
| moderationRecordId | uuid | FK → moderation_records |
| userId | uuid | FK → users (appellant) |
| message | text | Appeal reason (max 500 chars) |
| status | enum | submitted / under_review / approved / rejected |
| moderatorId | uuid | nullable |
| moderatorDecision | text | nullable (decision notes) |
| decidedAt | timestamptz | nullable |
| createdAt | timestamptz | |

#### `moderation_records` additions
| Column | Type | Notes |
|--------|------|-------|
| contentType | varchar | reel / profile_photo / story / cover_image |
| rekognitionVideoJobId | varchar | nullable — for async video analysis |
| decidedAt | timestamptz | nullable — when manual decision made |

### New API Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/v1/moderation/:id/appeal` | User | Submit moderation appeal |
| GET | `/v1/moderation/appeals/my` | User | List own appeals |
| GET | `/v1/moderation/banned` | Admin | Banned content list (paginated, by contentType) |
| GET | `/v1/moderation/stats` | Admin | Rolling stats (days param) |
| POST | `/v1/moderation/:id/restore` | Admin | Restore + re-moderate content |
| GET | `/v1/moderation/appeals` | Admin | All appeals with status filter |
| POST | `/v1/moderation/appeals/:id/review` | Admin | Approve or reject appeal |

### Content Type Moderation Trigger Map

| Content Type | Trigger Point | S3 Key Pattern |
|---|---|---|
| `reel` | `video-processing.processor.ts` step 9 | `uploads/{mediaId}/original.mp4` |
| `profile_photo` | `ProfileService.updateProfile()` after DB save | `avatarKey` from DTO |
| `cover_image` | `ProfileService.updateProfile()` after DB save | `coverKey` from DTO |
| `story` | `StoriesService.createStory()` after MongoDB save | `uploads/{mediaId}/original.{ext}` |

### Key Constraints

- One active appeal per moderation record (enforced: `status NOT IN ('submitted', 'under_review')` check)
- Appeal only allowed for `status === 'rejected'` records
- Restoring content sets it back to `pending` and re-enqueues in Bull
- MongoDB sync failures are logged but do not throw — shadow-ban is best-effort
- Auto-ban threshold: 3 rejections within 24 hours → `accountStatus = 'suspended'`
- Profile/cover/story moderation triggered fire-and-forget (non-blocking) — does not block profile save

---

## 🐛 Bug Fixes & Constraint Updates (March 2026)

### Admin Score Normalization

Rekognition returns confidence scores in the **0–100 range**. The admin portal UI renders scores as `score * 100` to display percentages. Therefore ALL controller responses that expose `explicitScore` / `violenceScore` **MUST divide by 100** before returning:

```ts
explicitScore: item.aiScoreExplicit != null ? item.aiScoreExplicit / 100 : null,
violenceScore: item.aiScoreViolence != null ? item.aiScoreViolence / 100 : null,
```

Affected: `getPendingReviews` and `getBannedContent` handlers in `moderation.controller.ts`. The entity stores raw Rekognition values (0–100); division happens at the API boundary.

### Thumbnail URL Path

`VideoProcessingProcessor` uploads thumbnails to S3 as:
```
uploads/{mediaId}/thumbnail.jpg   (in S3_OUTPUT_BUCKET)
```

The correct CloudFront URL is:
```
${AWS_CLOUDFRONT_URL}/uploads/${mediaId}/thumbnail.jpg
```

❌ **Historical bug**: `${cdnUrl}/thumbnails/${mediaId}.jpg`

When `AWS_CLOUDFRONT_URL` is not set, falls back to:
```
https://${S3_OUTPUT_BUCKET}.s3.ap-south-1.amazonaws.com/uploads/${mediaId}/thumbnail.jpg
```

### ModerationAppeal Entity Must Be in `forRoot` entities

`ModerationAppeal` MUST appear in both:

1. `TypeOrmModule.forFeature([..., ModerationAppeal, ...])` in `ModerationModule` — for repository injection
2. `entities: [..., ModerationAppeal, ...]` in `TypeOrmModule.forRoot(...)` in `AppModule` — for schema synchronization

In NestJS TypeORM 11, `forFeature()` entities are **not** included in the root DataSource entity list for schema sync (`synchronize: true`). Missing this registration prevents `moderation_appeals` table creation in development, causing `GET /v1/moderation/appeals` to return 500.

In production (`synchronize: false`), migration `1741440000000-ModerationSystemHardening.ts` must be run to create the table.
