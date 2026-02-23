# Chefooz — Data Flow Diagrams

**Version:** 1.0  
**Last Updated:** February 2026  
**Scope:** All major cross-module data flows across the Chefooz platform  
**Business Rule Summary:** This document illustrates how data moves through the Chefooz system for the six critical platform flows: Order Lifecycle, Video Upload & Processing, Notification Orchestration, Search, Authentication, and Delivery. Each diagram shows the exact sequence of service calls, queue events, and external API interactions.  
**Constraint:** All flows pass through the NestJS backend at `/api/v1/...` — no direct database access from client apps.

---

## 📋 Table of Contents

1. [Order Lifecycle Flow](#order-lifecycle-flow)
2. [Video Upload & Processing Flow](#video-upload--processing-flow)
3. [Notification Orchestration Flow](#notification-orchestration-flow)
4. [Search Flow](#search-flow)
5. [Authentication Flow](#authentication-flow)
6. [Delivery Assignment & Tracking Flow](#delivery-assignment--tracking-flow)
7. [Payment & Commission Settlement Flow](#payment--commission-settlement-flow)
8. [Moderation & Content Review Flow](#moderation--content-review-flow)

---

## Order Lifecycle Flow

This is the primary commercial flow — from item selection through delivery and post-order settlement.

```
CUSTOMER MOBILE APP                    CHEFOOZ-APIS (NestJS)                  EXTERNAL
        │                                       │
        │  1. GET /api/v1/chef/menu/:chefId      │
        │──────────────────────────────────────►│
        │◄──────────────────────────────────────│  ← menu items from PostgreSQL
        │                                        │     (cache: Valkey 10 min TTL)
        │
        │  2. POST /api/v1/cart/add              │
        │──────────────────────────────────────►│  ← CartModule validates:
        │◄──────────────────────────────────────│    - chef-scoped lock
        │  { cartId, items[], total }            │    - menu item availability
        │                                        │    - item pricing (PostgreSQL)
        │
        │  3. POST /api/v1/checkout/create-session│
        │──────────────────────────────────────►│  ← CheckoutModule:
        │◄──────────────────────────────────────│    - Haversine distance calc
        │  { sessionId, breakdown:              │    - delivery fee calculation
        │    subtotal, deliveryFee,             │    - platform fee
        │    tax, total }                        │    - session stored PostgreSQL
        │
        │  4. POST /api/v1/payment/create-order  │
        │──────────────────────────────────────►│                           ┌────────────────┐
        │                                        │──── createOrder() ──────►│   Razorpay API │
        │◄──────────────────────────────────────│◄────────────────────────│ ({ orderId,    │
        │  { razorpayOrderId, amount, currency } │  razorpay order object   │   amount })    │
        │                                        │                           └────────────────┘
        │
        │  5. [User pays via Razorpay SDK]       │
        │   (Razorpay hosted checkout)           │
        │                                        │
        │  6. POST /api/v1/payment/verify        │
        │──────────────────────────────────────►│  ← PaymentModule:
        │  { orderId, paymentId, signature }    │    - HMAC-SHA256 verify
        │                                        │    - PaymentIntent → CAPTURED
        │                                        │    ┌─ OrderModule.createOrder()
        │                                        │    │  - Row in `orders` table
        │                                        │    │  - Status: PAYMENT_SUCCESS
        │                                        │    └─ emit('order.created')
        │◄──────────────────────────────────────│
        │  { success: true, orderId }            │
        │                                        │
        │                                        │  ← ORDER events fan-out:
        │                                        │    ├─ NotificationModule
        │                                        │    │  ├─ push to Customer (receipt)
        │                                        │    │  ├─ push to Chef (new order)
        │                                        │    │  └─ email to Customer
        │                                        │    │
        │                                        │    └─ CommissionModule
        │                                        │       ← schedules commission
        │                                        │         calculation post-delivery
```

### Order Status Lifecycle

```
                    PAYMENT_SUCCESS
                          │
                          ▼
                    ORDER_PLACED
                          │
           ┌──────────────┴──────────────┐
           │                             │
           ▼                             ▼
    CHEF_ACCEPTED                  CHEF_REJECTED
           │                             │
           ▼                             ▼
    PREPARING                     [ORDER_CANCELLED]
           │
           ▼
    READY_FOR_PICKUP
           │
           ▼
    RIDER_ASSIGNED
           │
           ▼
    PICKED_UP
           │
           ▼
    OUT_FOR_DELIVERY
           │
           ▼
    DELIVERED
           │
       ┌───┴───────────────────┐
       ▼                       ▼
  COMMISSION                REVIEW
  CALCULATED             REQUESTED
       │
       ▼
 SETTLEMENT_PENDING
       │
       ▼
  RECONCILED
```

---

## Video Upload & Processing Flow

The reel creation pipeline spans mobile upload through AWS transcoding to final publishing.

```
CHEF MOBILE APP                 CHEFOOZ-APIS                    AWS                  MONGODB
      │                               │                            │
      │  1. POST /api/v1/media/upload-url  │                       │
      │──────────────────────────────►│                            │
      │                               │──── S3.createPresignedPost()──►│ S3 INPUT BUCKET
      │◄──────────────────────────────│                            │
      │  { uploadUrl, mediaJobId }    │                            │
      │                               │                            │
      │  2. PUT <S3 pre-signed URL>   │                            │
      │───────────────────────────────────────────────────────────►│ (direct to S3,
      │◄──────────────────────────────────────────────────────────│  bypasses API)
      │  { ETag }                     │                            │
      │                               │                            │
      │  3. POST /api/v1/media/process│                            │
      │──────────────────────────────►│                            │
      │  { mediaJobId, s3Key,         │                            │
      │    title, hashtags, ... }     │                            │
      │                               │                            │
      │                               │── Bull Queue: 'mediaconvert'
      │                               │   { jobId, inputKey }      │
      │◄──────────────────────────────│                            │
      │  { jobId, status: 'queued' }  │                            │
      │                               │                            │
      │                               │                            │
      │              [BACKGROUND: Bull Worker processes job]        │
      │                               │                            │
      │                               │─ MediaConvert.createJob() ─►│ AWS MediaConvert
      │                               │  (720p, 1080p HLS presets) │
      │                               │◄────────────────────────────│ { jobId }
      │                               │                            │
      │                               │  [Job runs in AWS cloud]   │
      │                               │                            │
      │                               │        SQS Notification    │
      │                               │◄──── SQS.receiveMessage() ─│ (1 min cron poll)
      │                               │   { status: 'COMPLETE',    │
      │                               │     outputUrls: [...] }    │
      │                               │                            │
      │                               │                                          │
      │                               │── MongoDB: mediaconvert_jobs.update()───►│
      │                               │── MongoDB: reels.create({               │
      │                               │     videoUrls, thumbnailUrl,             │
      │                               │     status: 'PUBLISHED' })  ────────────►│
      │                               │                                          │
      │                               │── Elasticsearch: index reel  ──────────► ES
      │                               │                                          │
      │                               │── NotificationModule.emit('reel.published')
      │                               │   → push to chef followers
      │
      │  4. GET /api/v1/media/status/:jobId
      │──────────────────────────────►│
      │◄──────────────────────────────│
      │  { status: 'PUBLISHED',       │
      │    reelId, videoUrls }        │
```

### Media Processing State Machine

```
UPLOADING → UPLOAD_COMPLETE → TRANSCODING_QUEUED → TRANSCODING
    → THUMBNAIL_EXTRACTED → PUBLISHED
    
    (error paths) → FAILED → [manual retry available]
```

---

## Notification Orchestration Flow

The notification module acts as a central event bus fan-out hub.

```
SOURCE MODULE             NOTIFICATION ORCHESTRATOR           DELIVERY CHANNELS
     │                            │                              │
     │  emit('order.created',     │                              │
     │    { userId, orderId, ... })│                             │
     │──────────────────────────►│                              │
     │                            │                              │
     │                            │ 1. Load user preferences    │
     │                            │    ← PostgreSQL (cache: 5min Valkey)
     │                            │                              │
     │                            │ 2. Fan out to handlers:     │
     │                            │    ┌──────────────────────┐ │
     │                            │    │  PUSH HANDLER        │ │
     │                            │    │  ← fetch push tokens  │ │
     │                            │    │  ← PostgreSQL         │ │
     │                            │    │  → Expo Push API ──────►│ Expo Push
     │                            │    │    exp.host/v2/push    │ (APNs + FCM)
     │                            │    └──────────────────────┘ │
     │                            │    ┌──────────────────────┐ │
     │                            │    │  EMAIL HANDLER        │ │
     │                            │    │  ← email template     │ │
     │                            │    │  → Resend API ─────────►│ Resend → SMTP
     │                            │    └──────────────────────┘ │
     │                            │    ┌──────────────────────┐ │
     │                            │    │  ACTIVITY HANDLER    │ │
     │                            │    │  → MongoDB           ──►│ activity_feed
     │                            │    │    (activity_feed)    │ │ collection
     │                            │    └──────────────────────┘ │
     │                            │    ┌──────────────────────┐ │
     │                            │    │  IN-APP HANDLER       │ │
     │                            │    │  → PostgreSQL        ──►│ notifications
     │                            │    │    (notifications)    │ │ table
     │                            │    └──────────────────────┘ │

EVENT TYPES AND THEIR FAN-OUT:

  order.created        → push(customer+chef) + email(customer) + activity
  order.accepted       → push(customer)
  order.rejected       → push(customer) + email(customer)
  order.ready_pickup   → push(rider)
  order.delivered      → push(customer) + email(customer) + activity
  reel.published       → push(followers batch) + activity(followers)
  reel.liked           → activity(reel.owner)
  reel.commented       → push(reel.owner) + activity(reel.owner)
  user.followed        → push(followed.user) + activity(followed.user)
  payment.failed       → push(customer) + email(customer)
  payout.initiated     → push(chef) + email(chef)
  report.resolved      → push(reporter) + email(reporter)
```

### Push Notification Batching

```
Multiple recipients (e.g., reel published to 500 followers)
       │
       ▼
NotificationDispatcher
       │
       ├── Chunk into batches of 100 (Expo Push API limit)
       ├── POST https://exp.host/--/api/v2/push/send (batch)
       ├── Handle per-ticket errors
       │   ├── DeviceNotRegistered → remove token from PostgreSQL
       │   └── MessageRateExceeded → retry with Bull queue
       └── Log delivery status
```

---

## Search Flow

Search has a tiered architecture: Elasticsearch first, MongoDB/PostgreSQL fallback.

```
MOBILE APP                    CHEFOOZ-APIS                  DATA STORES
    │                               │
    │  GET /api/v1/search-elastic/  │
    │    dishes?query=biryani       │
    │    &lat=19.07&lng=72.87       │
    │    &radius=10&sortBy=relevance│
    │──────────────────────────────►│
    │                               │
    │                               │ 1. Rate limit check (Valkey)
    │                               │    10 req/s per user
    │                               │
    │                               │ 2. Cache check (Valkey)
    │                               │    key: search:dishes:{hash}
    │                               │    TTL: 60 seconds
    │                               │
    │                               │ ┌─ CACHE HIT ─────────────────►│ Valkey
    │◄──────────────────────────────│ │  return cached results       │
    │                               │ │                               │
    │                               │ └─ CACHE MISS:                 │
    │                               │    ↓                           │
    │                               │ 3. Elasticsearch query:        │
    │                               │    index: chefooz_reels       ►│ Elasticsearch
    │                               │    bool.must:                  │
    │                               │      multi_match: [title,     │
    │                               │        description, hashtags] │
    │                               │    filter:                     │
    │                               │      geo_distance (radius)    │
    │                               │      isAvailable: true        │
    │                               │    sort: [_score, distance]   │
    │                               │                               │
    │                               │ ┌─ ES ERROR / NO RESULTS:     │
    │                               │ │  fallback MongoDB query ───►│ MongoDB
    │                               │ └──────────────────────────── │
    │                               │                               │
    │                               │ 4. Hydrate missing chef details│
    │                               │    ← PostgreSQL (chef profiles)│ PostgreSQL
    │                               │                               │
    │                               │ 5. Cache results (Valkey)     │
    │                               │    TTL: 60 seconds            │
    │                               │                               │
    │◄──────────────────────────────│
    │  { results: [                 │
    │    { reelId, title, chef,    │
    │      distance, relevanceScore }]}

SEARCH TYPE ROUTING:

  /search-elastic/dishes     → Elasticsearch (reels) + chef hydration
  /search-elastic/chefs      → Elasticsearch (users) — PLANNED
  /search/users              → PostgreSQL (current, pre-ES)
  /search/suggest            → Elasticsearch suggestions (20 req/s limit)
  /explore                   → Feed-ranked discovery (no text query)
```

---

## Authentication Flow

Chefooz uses OTP-based login (no passwords). JWTs are stored securely on-device.

```
MOBILE APP                        CHEFOOZ-APIS                 EXTERNAL
    │                                   │
    │  1. POST /api/v1/auth/v2/send-otp │
    │──────────────────────────────────►│
    │  { phoneNumber: "+91XXXXXXXXXX" } │
    │                                   │
    │                                   │ ← AuthModule / OtpService:
    │                                   │   - Generate 6-digit OTP
    │                                   │   - Store in PostgreSQL (otp_requests)
    │                                   │   - TTL: 5 minutes
    │                                   │
    │                                   │ ┌─ Try WhatsApp Cloud API (primary) ──►│ WhatsApp
    │                                   │ │   { to: phone, template: otp_v1 }   │ Cloud API
    │                                   │ │                                      │
    │                                   │ └─ If WhatsApp fails → Twilio SMS ───►│ Twilio SMS
    │                                   │     { to: phone, body: "Your OTP..." }│ (fallback)
    │◄──────────────────────────────────│
    │  { success: true,                 │
    │    requestId, expiresIn: 300,     │
    │    channel: 'whatsapp'|'sms' }   │
    │                                   │
    │  2. POST /api/v1/auth/v2/verify-otp│
    │──────────────────────────────────►│
    │  { requestId, otp: "123456" }     │
    │                                   │
    │                                   │ ← AuthModule:
    │                                   │   - Lookup otp_requests by requestId
    │                                   │   - Validate OTP + not expired
    │                                   │   - Create/fetch user record (PostgreSQL)
    │                                   │   - Issue JWT (HS256 / RS256)
    │                                   │     { userId, phone, role, profileMode }
    │                                   │   - Store session (PostgreSQL)
    │                                   │
    │◄──────────────────────────────────│
    │  { accessToken, refreshToken,    │
    │    user: { id, phone, mode } }   │
    │                                   │
    │  [Store in expo-secure-store]     │
    │  - accessToken  (key: 'jwt')     │
    │  - refreshToken (key: 'refresh') │
    │                                   │
    │  3. All subsequent requests:      │
    │──────────────────────────────────►│
    │  Authorization: Bearer <token>    │
    │                                   │ ← JwtAuthGuard validates:
    │                                   │   - Signature
    │                                   │   - Expiry
    │                                   │   - userId existence
    │◄──────────────────────────────────│
    │                                   │
    │  4. Token refresh (on 401):       │
    │  POST /api/v1/auth/refresh-token  │
    │──────────────────────────────────►│
    │  { refreshToken }                 │
    │◄──────────────────────────────────│
    │  { accessToken }                  │

OTP DELIVERY CHAIN:

  WhatsApp Cloud API (primary)
      │ fails (delivery timeout / non-WA number)
      ▼
  Twilio SMS (automatic fallback)
      │
      ▼
  6-digit OTP delivered to phone number (E.164 format)
  Valid 5 minutes | Rate-limited per device

TOKEN ARCHITECTURE:

  Mobile Access Token:  { role: 'customer' | 'chef' | 'rider' }
  Admin Access Token:   { role: 'admin', adminId, scope: 'admin-portal' }
  
  → Separate secrets
  → Different expiry times
  → Guards enforce role separation
```

---

## Delivery Assignment & Tracking Flow

```
BACKEND (ORDER FLOW)          DELIVERY MODULE               RIDER APP               CUSTOMER APP
        │                           │                           │                         │
        │ emit('order.accepted',    │                           │                         │
        │   { orderId, pickupLat,  │                           │                         │
        │     pickupLng, ... })     │                           │                         │
        │──────────────────────────►│                           │                         │
        │                           │                           │                         │
        │                           │ 1. Find nearby riders:   │                         │
        │                           │    ← rider_locations     │                         │
        │                           │    (PostgreSQL, updated  │                         │
        │                           │     by rider GPS pings)  │                         │
        │                           │                           │                         │
        │                           │ 2. FIFO Assignment:      │                         │
        │                           │    - Distributed lock    │                         │
        │                           │      (Valkey, 30s TTL)   │                         │
        │                           │    - Assign to rider[0]  │                         │
        │                           │    - Fallback to rider[1]│                         │
        │                           │      if no response      │                         │
        │                           │                           │                         │
        │                           │ 3. NotificationModule:   │                         │
        │                           │    push to rider ────────►│ [New order alert]       │
        │                           │                           │                         │
        │                           │                           │ 4. POST /rider-orders/accept
        │                           │◄──────────────────────────│                         │
        │                           │                           │                         │
        │                           │ 5. Update order status:  │                         │
        │                           │    → RIDER_ASSIGNED       │                         │
        │                           │    → emit('rider.assigned')──────────────────────►│ push
        │                           │                           │                         │
        │                           │                           │ 6. GPS tracking begins  │
        │                           │◄────── POST /rider-location/update ────────────────│
        │                           │  { lat, lng } every 30s  │                         │
        │                           │                           │                         │
        │                           │ 7. DeliveryEta recalculate│                         │
        │                           │    (Haversine, re-trigger)│                         │
        │                           │───────────────────────────────────────────────────►│ ETA push
        │                           │                           │                         │
        │                           │                           │ 8. POST /rider-orders/picked-up
        │                           │◄──────────────────────────│                         │
        │                           │    → Order: PICKED_UP    │                         │
        │                           │    → push to customer ───────────────────────────►│
        │                           │                           │                         │
        │                           │                           │ 9. POST /rider-orders/delivered
        │                           │◄──────────────────────────│                         │
        │                           │    → Order: DELIVERED    │                         │
        │                           │    → emit('order.delivered')                        │
        │                           │    → CommissionModule calculates                    │
        │                           │    → ReviewModule queues review prompt               │
        │                           │    → push + email to customer ───────────────────►│
```

---

## Payment & Commission Settlement Flow

Full financial lifecycle from order payment to chef payout.

```
PAYMENT CAPTURE                    COMMISSION CALC              WITHDRAWAL
     │                                   │                          │
     │ PaymentIntent → CAPTURED          │                          │
     ▼                                   │                          │
CommissionModule.calculateForOrder()     │                          │
     │                                   │                          │
     │  Inputs:                          │                          │
     │  - order.totalAmount              │                          │
     │  - chef.subscriptionTier          │                          │
     │  - delivery.fee                   │                          │
     │  - tip (if any)                   │                          │
     │                                   │                          │
     │  Formula (V2):                    │                          │
     │  platformFee = total * rate       │                          │
     │  chefEarning = total - platformFee│                          │
     │              - deliveryFee        │                          │
     │                                   │                          │
     ▼                                   │                          │
CommissionLedger.creditChef()           │                          │
  → INSERT commission_ledger             │                          │
    { orderId, chefId,                  │                          │
      gross, net, commission,           │                          │
      status: 'PENDING_SETTLEMENT' }   │                          │
     │                                   │                          │
     │  (after order DELIVERED)          │                          │
     ▼                                   │                          │
CommissionLedger.settleOrder()          │                          │
  → UPDATE: status = 'SETTLED'          │                          │
  → UPDATE chef wallet balance          │                          │
     │                                   │                          │
     │                                   │                          │
     ▼                                   │                          │
  Chef requests payout                   │                          │
  POST /api/v1/withdrawal/request        │                    ──────┘
     │                                   │                   │
     │  WithdrawalModule validates:      │                   │
     │  - min withdrawal amount          │                   │
     │  - bank/UPI details verified      │                   │
     │  - no pending disputes            │                   │
     │                                   │                   │
     ▼                                   │                   │
  WithdrawalRequest created              │                   │
  { status: 'PENDING' }                 │                   │
     │                                   │                   │
  [Admin review in portal]               │                   │
     │                                   │                   │
  Admin: POST /api/v1/admin/withdrawal/:id/approve           │
     │                                   │                   │
     ▼                                   │                   │
  Payout provider (UPI/NEFT)            │                   │
  { status: 'PROCESSING' }              │                   │
     │                                   │                   │
     ▼                                   │                   │
  Webhook: payment_processed            │                   │
     │                                   │                   │
     ▼                                   │                   │
  WithdrawalRequest.status = 'COMPLETED'│                   │
  CommissionLedger.reconcile()          │                   │
  ReconciliationLog.create()            ──────────────────►│
  NotificationModule.emit('payout.completed')
```

---

## Moderation & Content Review Flow

Three-tier AI-assisted content review pipeline.

```
CONTENT UPLOAD            AI SCREENING              HUMAN REVIEW              APPEAL
(reel/comment/story)           │                         │                      │
        │                       │                         │                      │
        │ 1. Pre-publish screen │                         │                      │
        │──────────────────────►│ ImageSafetyCheck        │                      │
        │                       │  (vision API)           │                      │
        │                       │                         │                      │
        │   CLEAN ◄─────────────│                         │                      │
        │   → PUBLISHED         │                         │                      │
        │                       │                         │                      │
        │   FLAGGED ◄───────────│                         │                      │
        │   → PENDING_REVIEW    │──────────────────────── │ → AdminQueue        │
        │                       │                         │                      │
        │ 2. User report        │                         │                      │
        │──────────────────────►│ ModerationModule        │                      │
        │  POST /report         │  auto-assess severity   │                      │
        │  { contentId, type }  │                         │                      │
        │                       │  LOW → log only         │                      │
        │                       │  MEDIUM → review queue  │──────────────────── │ → AdminQueue
        │                       │  HIGH → auto-hide +     │                      │
        │                       │    review queue          │                      │
        │                       │                         │                      │
        │                       │                         │ 3. Admin reviews     │
        │                       │                         │──────────────────────│►
        │                       │                         │  /admin/moderation  │
        │                       │                         │                      │
        │                       │                         │  APPROVED → restore │
        │                       │                         │  REJECTED → remove  │
        │                       │                         │   + notify user      │
        │                       │                         │                      │
        │                       │                         │ 4. User appeals ─────►│
        │                       │                         │                      │
        │                       │                         │                      │ AppealModule
        │                       │                         │◄─────────────────────│
        │                       │                         │  Admin re-reviews    │
        │                       │                         │  Final decision      │
        │                       │                         │  (no further appeal) │
        │                       │                         │                      │
        │                       │                     ModerationAudit.create()
        │                       │                     (all decisions logged)
```

---

## Cross-Flow Data Sharing Summary

The following table shows which data stores are written and read in each major flow:

| Flow | PostgreSQL (writes) | MongoDB (writes) | Valkey (reads/writes) | External API calls |
|------|---------------------|------------------|-----------------------|-------------------|
| Order | orders, cart_items, payment_intents, commission_ledger | — | cart cache, rate limits | Razorpay |
| Video Upload | media_jobs | reels, mediaconvert_jobs | Bull queues | S3, MediaConvert, SQS |
| Notification | notifications, email_logs | activity_feed | prefs cache | Expo Push, Resend |
| Search | — | — | result cache, rate limits | Elasticsearch |
| Authentication | users, otp_requests, sessions | — | — | WhatsApp Cloud API (primary), Twilio SMS (fallback) |
| Delivery | delivery_requests, rider_locations, active_deliveries | — | assignment locks | — |
| Commission | commission_ledger, withdrawal_requests, wallets | — | — | Payout provider |
| Moderation | reports, moderation_records, appeals, moderation_audits | — | — | Vision AI API |

---

[SLICE_COMPLETE ✅]
