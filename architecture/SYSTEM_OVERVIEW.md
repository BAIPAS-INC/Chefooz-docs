# Chefooz — System Architecture Overview

**Version:** 1.0  
**Last Updated:** February 2026  
**Scope:** Complete system architecture across all apps, services, infrastructure, and integrations  
**Business Rule Summary:** Chefooz is a food-tech platform that combines a social content feed (reels/stories) with a home-chef ordering marketplace. The architecture is built as a managed Nx monorepo serving three distinct client surfaces: a React Native mobile app, a Next.js admin portal, and a NestJS backend API.  
**Target Release:** QA by mid-February 2026 (first production release)

---

## 📋 Table of Contents

1. [System Overview](#system-overview)
2. [Monorepo Structure](#monorepo-structure)
3. [Application Layer](#application-layer)
4. [Backend Module Map](#backend-module-map)
5. [Data Layer](#data-layer)
6. [Infrastructure & Cloud Services](#infrastructure--cloud-services)
7. [Observability Stack](#observability-stack)
8. [Security Architecture](#security-architecture)
9. [Network & API Design](#network--api-design)
10. [Module Dependency Overview](#module-dependency-overview)
11. [Deployment Architecture](#deployment-architecture)
12. [Health & Monitoring Endpoints](#health--monitoring-endpoints)

---

## System Overview

Chefooz is a **social food ordering platform** where:
- **Customers** discover home chefs through a TikTok-style reel feed, browse menus, and place orders
- **Home Chefs** create cooking content, manage their virtual kitchen, accept orders, and earn via payouts
- **Delivery Riders** receive order assignments and track/complete deliveries
- **Admins** manage platform operations through a dedicated web portal

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         CLIENT SURFACES                              │
├──────────────────────┬─────────────────────┬────────────────────────┤
│  📱 Mobile App        │  🌐 Admin Portal     │  🔧 Dev / E2E          │
│  (Expo React Native) │  (Next.js 15)        │  (Cypress)            │
│  iOS + Android + Web │  Internal Control    │  chefooz-app-e2e       │
│  chefooz-app         │  chefooz-admin       │  chefooz-apis-e2e      │
└──────────┬───────────┴──────────┬──────────┴────────────────────────┘
           │                      │
           │   REST + HTTPS       │   REST + HTTPS (Admin JWT)
           │   /api/v1/...        │   /api/v1/admin/...
           ▼                      ▼
┌──────────────────────────────────────────────────────────────────────┐
│                         chefooz-apis                                  │
│                    NestJS 11 Backend (Port 3333)                       │
│                   URI Versioned: /api/v1/...                           │
│                                                                        │
│  52+ Modules across 9 functional domains                               │
│  PostgreSQL (TypeORM) + MongoDB (Mongoose) + Valkey/Redis (Bull)      │
└──────────────────────────────────────────────────────────────────────┘
           │
           ├── PostgreSQL (AWS RDS)        ── Transactional data
           ├── MongoDB Atlas               ── Content (reels, chat, stories)
           ├── Valkey/ElastiCache          ── Cache + Bull job queues
           ├── Elasticsearch              ── Full-text search
           ├── AWS S3                     ── Media storage (raw + processed)
           ├── AWS MediaConvert           ── Video transcoding pipeline
           ├── AWS SQS                    ── Async job notifications
           ├── Razorpay                   ── Payment processing (INR)
           ├── Resend                     ── Transactional emails
           ├── Expo Push API              ── Mobile push notifications
           ├── WhatsApp Cloud API         ── OTP delivery (primary)
           ├── Twilio SMS                 ── OTP delivery (fallback)
           └── AWS ADOT / CloudWatch / X-Ray ── Observability
```

---

## Monorepo Structure

The project uses **Nx 22** as the monorepo toolchain with **SWC** for fast TypeScript compilation.

```
chefooz-app/                           ← Nx Workspace Root
├── apps/
│   ├── chefooz-app/                   ← Expo React Native (iOS + Android + Web)
│   ├── chefooz-apis/                  ← NestJS REST API Backend
│   ├── chefooz-admin/                 ← Next.js 15 Admin Portal
│   ├── chefooz-app-e2e/               ← Cypress E2E for Mobile
│   └── chefooz-apis-e2e/              ← E2E for Backend APIs
│
├── libs/
│   ├── api-client/                    ← Axios clients + React Query hooks
│   ├── domain/                        ← Pure domain logic + validation
│   ├── types/                         ← Shared TypeScript types + interfaces
│   ├── ui/                            ← Shared UI components (cross-platform)
│   └── utils/                         ← Pure utility functions
│
├── migrations/                        ← TypeORM migration files
├── infra/                             ← Infrastructure config
│   ├── observability/                 ← AWS ADOT Collector config
│   └── alerting/                      ← CloudWatch alerting rules
├── scripts/                           ← Build + deployment scripts
├── tools/                             ← Custom Nx generators
└── docs/                              ← This documentation
    ├── modules/                       ← Per-module docs
    ├── journeys/                      ← End-to-end user flows
    ├── integrations/                  ← Cross-module integration guides
    └── architecture/                  ← System-level docs (this file)
```

### Shared Libraries

| Library | Path | Purpose | Consumers |
|---------|------|---------|----------|
| `@chefooz-app/types` | `libs/types` | Shared TypeScript DTOs, enums, interfaces | All apps |
| `@chefooz-app/api-client` | `libs/api-client` | Axios wrappers + React Query hooks | Mobile + Admin |
| `@chefooz-app/domain` | `libs/domain` | Validation, computations, CRS scoring | All apps |
| `@chefooz-app/ui` | `libs/ui` | Shared React Native/Web components | Mobile |
| `@chefooz-app/utils` | `libs/utils` | Pure helper functions | All apps |

---

## Application Layer

### 1. Mobile App (`apps/chefooz-app`)

**Framework**: Expo SDK 53 (Managed Workflow)  
**Navigation**: Expo Router 5 (file-system based routing)  
**State Management**: Zustand 5  
**Data Fetching**: TanStack React Query 5  
**Auth Storage**: `expo-secure-store` (JWT stored securely)

#### Screen Structure

```
src/app/
├── _layout.tsx              ← Root layout (QueryClient, Theme, Auth guard)
├── index.tsx                ← Entry redirect
├── (tabs)/                  ← Bottom tab navigator
│   ├── _layout.tsx          ← Tab bar config
│   ├── index.tsx            ← Home feed
│   ├── explore.tsx          ← Explore/discover
│   ├── search.tsx           ← Search
│   ├── orders.tsx           ← My orders
│   └── profile.tsx          ← User profile
├── auth/                    ← Login, OTP, onboarding
├── cart/                    ← Cart + checkout
├── checkout/                ← Checkout + payment
├── chef/                    ← Chef profile view
├── reels/                   ← Reel viewer
├── orders/                  ← Order detail, tracking
├── messages/                ← Chat / messaging
├── search/                  ← Search screens
├── social/                  ← Follow, followers, blocked
├── profile/                 ← User profile edit
├── settings/                ← App settings, preferences
├── notifications/           ← Notification list
├── activity/                ← Activity feed
├── collections/             ← Saved reels
├── stories/                 ← Story viewer
├── rider/                   ← Rider-specific screens
├── admin/                   ← Admin-debug screens (dev only)
└── moderation/              ← Content moderation (self-service)
```

#### Zustand Stores

| Store | Purpose |
|-------|---------|
| `auth.store.ts` | JWT token, authenticated user, login state |
| `cart.store.ts` | Cart items, chef lock, pricing |
| `onboarding.store.ts` | Chef onboarding wizard state |
| `profile-mode.store.ts` | Customer ↔ Chef mode toggle |
| `media.store.ts` | Upload state, reel creation |
| `upload-v2.store.ts` | V2 upload pipeline state |
| `camera-settings.store.ts` | Camera/recording preferences |
| `address.store.ts` | User addresses, selected address |
| `review.store.ts` | Review submission state |
| `ui.store.ts` | Global UI state (modals, toasts) |
| `video-player.store.ts` | Reel player state |

#### API Client Pattern (libs/api-client)

```typescript
// Each module has: <module>.client.ts + <module>.hooks.ts
// Example:
import { useSearchDishes } from '@chefooz-app/api-client';

const { data, isLoading } = useSearchDishes({
  query: 'butter chicken',
  sortBy: 'relevance',
  lat: 19.07,
  lng: 72.87,
});
```

---

### 2. Backend API (`apps/chefooz-apis`)

**Framework**: NestJS 11  
**Transport**: HTTP/REST (Express adapter)  
**Versioning**: URI versioning (`/api/v1/...`)  
**Port**: 3333 (configurable via `PORT` env var)  
**Auth**: JWT with role-based guards (`@Roles()`)

**Bootstrap order**:
1. OpenTelemetry SDK initialized (before NestJS)
2. NestJS app created
3. CORS enabled (all origins in dev, specific in production)
4. URI versioning enabled
5. Global prefix `/api` set
6. Swagger auto-documentation enabled
7. App listens on `0.0.0.0:3333`

---

### 3. Admin Portal (`apps/chefooz-admin`)

**Framework**: Next.js 15 (App Router)  
**UI Library**: Material UI (MUI) v7  
**State / Data**: TanStack React Query  
**Auth**: Admin JWT (separate scope from mobile tokens)

#### Dashboard Routes

```
/dashboard/
├── page.tsx                 ← Dashboard home
├── analytics/               ← Platform analytics
├── audit-log/               ← Admin audit trail
├── feature-flags/           ← Feature flag management
├── payouts/                 ← Payout monitoring
├── pricing/                 ← Pricing configuration
├── reconciliation/          ← Financial reconciliation
├── reports/                 ← User reports
├── reviews/                 ← Chef review moderation
└── users/                   ← User management
```

---

## Backend Module Map

The 52+ NestJS modules are organized into 9 functional domains:

### Domain 1: Identity & Auth
| Module | Responsibility |
|--------|---------------|
| `auth` | OTP-based login, JWT issue/refresh, session management |
| `user` | User CRUD, preferences, account management |
| `profile` | Chef/customer profile, dual-mode toggle |
| `chef` | Chef profile (compliance status, public/private) |
| `chef-onboarding` | Multi-step chef registration wizard |
| `chef-compliance` | FSSAI license, verification docs |

### Domain 2: Content Creation
| Module | Responsibility |
|--------|---------------|
| `media` | S3 pre-signed URL generation, upload metadata |
| `media-processing` | FFmpeg thumbnail extraction, Bull queue integration |
| `reels` | Reel lifecycle (create, publish, archive), stats |
| `stories` | Story creation, TTL management, viewer tracking |

### Domain 3: Discovery & Search
| Module | Responsibility |
|--------|---------------|
| `feed` | Personalized home feed, ranked content delivery |
| `explore` | Food-first discovery, trending content |
| `search` | Legacy PostgreSQL user search |
| `search-elastic` | Elasticsearch full-text search (reels, menu items, chefs) |

### Domain 4: Social & Engagement
| Module | Responsibility |
|--------|---------------|
| `social` | Follow/unfollow, social graph, block |
| `social-close-friends` | Close friends list for restricted stories |
| `comments` | Reel comments, threaded replies, mentions |
| `collections` | Saved reels, personal collections |
| `activity` | Activity feed (likes, comments, follows) |

### Domain 5: Chef Kitchen & Marketplace
| Module | Responsibility |
|--------|---------------|
| `chef-kitchen` | Menu management (CRUD), availability, pricing |
| `chef-public` | Public chef page — read-only discovery endpoint |
| `platform-categories` | Reference data: cuisine types, dietary tags |
| `pricing` | Dynamic pricing, surge pricing configuration |

### Domain 6: Order Flow
| Module | Responsibility |
|--------|---------------|
| `cart` | Chef-scoped cart, item management, validation |
| `checkout` | Pricing breakdown, Haversine distance, session |
| `payment` | Razorpay create-order, HMAC verify, webhooks |
| `order` | Order lifecycle (10 statuses), snapshots, history |

### Domain 7: Fulfillment & Delivery
| Module | Responsibility |
|--------|---------------|
| `chef-orders` | Chef-side order management, accept/reject, prep time |
| `delivery` | Rider assignment (FIFO), GPS tracking, auto-cancel |
| `delivery-eta` | Real-time ETA calculation, progress updates |
| `rider-orders` | Rider-side pickup/delivery flow |
| `rider-profile` | Rider registration, documents, availability |
| `rider-location` | Real-time GPS location broadcasting |
| `rider-rating` | Post-delivery rider ratings |
| `rider-earnings` | Rider earnings ledger, wallet |

### Domain 8: Post-Order & Finance
| Module | Responsibility |
|--------|---------------|
| `review` | Customer reviews, chef CRS (reputation score) |
| `commission` | Commission calculation (V2 formula), ledger credit |
| `withdrawal` | Chef payout requests, UPI/bank/wallet |
| `reconciliation` | Financial reconciliation, ledger comparison |
| `tips` | Customer tips to riders, wallet credits |

### Domain 9: Platform & Admin
| Module | Responsibility |
|--------|---------------|
| `notification` | Push, email, activity feed orchestration |
| `messaging` | Direct messages, chat |
| `moderation` | Content review, AI pipeline, 3-tier workflow |
| `report` | User/content reports |
| `appeal` | Moderation appeal workflow |
| `feature-flags` | Dynamic feature configuration, A/B testing |
| `cache` | Redis/Valkey caching, distributed locking |
| `location` | Geocoding, Haversine, geospatial utilities |
| `analytics` | Platform analytics, event tracking |
| `reputation` | User trust score, chef CRS calculations |
| `deeplink` | Deep link generation and resolution |
| `subscription` | Chef subscription tiers (planned) |

**Internal modules** (not REST-exposed):
- `audit` — Admin action audit trail
- `telemetry` — OpenTelemetry metric instrumentation
- `health` — Health check endpoints
- `simulator` — Dev-only simulation tools
- `admin-debug` — Admin debugging utilities

---

## Data Layer

### PostgreSQL (AWS RDS)

**Purpose**: All transactional, relational data  
**ORM**: TypeORM (Code-First with migrations)  
**Pool size**: 10 max / 2 min connections  
**SSL**: Enabled in production (AWS RDS)

**Key entities** (40+ tables):

| Entity Group | Tables |
|-------------|--------|
| **Users** | `users`, `otp_requests`, `otp_sessions`, `user_addresses` |
| **Profiles** | `chef_profiles`, `chef_onboardings`, `chef_compliance`, `rider_profiles` |
| **Kitchen** | `chef_kitchens`, `chef_menu_items`, `menu_categories`, `chef_service_schedules`, `platform_categories` |
| **Orders** | `orders`, `cart_items`, `order_status_history`, `order_events`, `payment_intents` |
| **Finance** | `commission_ledger`, `withdrawal_requests`, `reconciliation_logs`, `wallets`, `tip_transactions`, `rider_earnings`, `rider_ratings` |
| **Social** | `user_follows`, `user_privacy`, `user_blocks`, `close_friends` |
| **Notifications** | `notifications`, `notification_preferences`, `push_tokens`, `email_notification_logs` |
| **Content Meta** | `collections`, `collection_items`, `saved_reels` |
| **Moderation** | `reports`, `appeals`, `moderation_records`, `moderation_audits` |
| **Reputation** | `user_reputation_current`, `user_reputation_events`, `user_reputation_snapshots`, `user_reputation_leaderboard` |
| **Delivery** | `delivery_requests`, `active_deliveries`, `rider_locations` |
| **Platform** | `feature_flags`, `activities`, `feed_abuse_tracking` |

**Migration strategy**: TypeORM migrations (`migrations/` directory), `synchronize: false` in production.

---

### MongoDB Atlas

**Purpose**: High-volume, unstructured, or schema-flexible content  
**ODM**: Mongoose  
**Pool**: 50 max / 10 min connections

**Collections**:

| Collection | Data |
|-----------|------|
| `reels` | Reel documents (video metadata, stats, hashtags, location) |
| `stories` | Story documents (TTL-managed, viewer tracking) |
| `messages` | Chat messages (per-conversation, append-heavy) |
| `media_jobs` | Upload/processing job metadata |
| `mediaconvert_jobs` | AWS MediaConvert job status tracking |
| `activity_feed` | Social activity entries |

**Why MongoDB for content**: Schema flexibility for media metadata, high-write throughput for stats (likes, views, comments), TTL indexes for stories.

---

### Valkey / Redis (AWS ElastiCache)

**Purpose**: Caching, Bull job queues, pub/sub, distributed locking  
**Client**: ioredis (via Bull and custom cache module)  
**Mode**: Cluster-compatible (`{bull}:` hash-tag key prefix)

**Usage patterns**:

| Pattern | TTL | Example Keys |
|---------|-----|-------------|
| Notification preferences | 5 min | `notification:prefs:{userId}` |
| Feed cache | 5 min | `feed:{userId}:page:{n}` |
| Chef menu cache | 10 min | `chef:menu:{chefId}` |
| Search results | 1 min | `search:dishes:{hash}` |
| Rate limiting | 1 min | `ratelimit:{ip}:{endpoint}` |
| Distributed locks | 30 sec | `lock:delivery-assign:{orderId}` |
| Bull queues | N/A (job TTL) | `{bull}:media-processing:*` |

**Bull Queues**:
- `media-processing` — Video thumbnail extraction, format conversion
- `mediaconvert` — AWS MediaConvert job dispatch and status polling
- `notifications` — Async notification delivery
- `delivery` — Delivery assignment jobs

---

### Elasticsearch 8+

**Purpose**: Full-text search with fuzzy matching and geospatial filters  
**Client**: `@elastic/elasticsearch` v8  
**Fallback**: MongoDB / PostgreSQL if ES unavailable

**Active Indices**:

| Index | Data | Shards | Replicas |
|-------|------|--------|---------|
| `chefooz_reels` | Reel content for dish/recipe search | 3 | 1 |
| `chefooz_menu_items` | Menu items (planned) | 3 | 1 |
| `chefooz_users` | Chef/user search (planned) | 3 | 1 |

**Sync strategy**: Background cron job syncs MongoDB reel changes to ES every 15 minutes.

---

## Infrastructure & Cloud Services

### AWS Services

| Service | Usage | Module |
|---------|-------|--------|
| **RDS (PostgreSQL)** | Primary relational database | All modules |
| **MongoDB Atlas** | Document store for content | Reels, Stories, Chat, Media |
| **ElastiCache (Valkey)** | Cache + queue backend | Cache, Bull, Notifications |
| **S3** | Raw video upload (`input bucket`), processed output (`output bucket`) | Media, Media-Processing |
| **MediaConvert** | Server-side video transcoding (720p/1080p HLS) | Media-Processing, integrations/media-convert |
| **SQS** | MediaConvert completion event queue (polled every 1 min) | integrations/media-convert |
| **CloudWatch** | Logs + metrics (via ADOT) | Observability |
| **X-Ray** | Distributed tracing (via ADOT) | Observability |
| **CloudFront** | CDN for processed video delivery | Media-Processing |

### Third-Party Services

| Service | Purpose | Integration Point |
|---------|---------|-----------------|
| **Razorpay** | Payment processing (INR only) | `modules/payment` + webhooks |
| **Resend** | Transactional emails (orders, payouts) | `modules/notification/email` |
| **Expo Push API** | iOS/Android push notifications | `modules/notification/dispatcher` |
| **Elasticsearch** | Search engine | `modules/search-elastic` |
| **Twilio SMS** | OTP fallback when WhatsApp delivery fails | `modules/auth/otp.service.ts` |
| **WhatsApp Cloud API** | Primary OTP delivery channel | `modules/auth/otp.service.ts` |

### Payment Gateway: Razorpay

- **Supported methods**: UPI, Cards (Visa/MC/RuPay), Net Banking, Wallets, EMI
- **Currency**: INR only
- **Webhooks**: `payment.captured`, `payment.failed`, `refund.created`, `refund.processed`
- **Signature verification**: HMAC-SHA256 (`orderId|paymentId`)

---

## Observability Stack

**Phase 3.7.1 implementation** — AWS ADOT (OpenTelemetry) wiring.

```
NestJS App
    │ OTLP/HTTP
    ▼
ADOT Collector (OpenTelemetry)
    │
    ├── Traces   → AWS X-Ray
    ├── Metrics  → CloudWatch Metrics
    └── Logs     → CloudWatch Logs
```

### Signal Types

| Signal | Tool | Config |
|--------|------|--------|
| **Traces** | AWS X-Ray (via ADOT) | `OTEL_SAMPLING_PERCENTAGE` (default: 100%) |
| **Metrics** | CloudWatch Metrics | `OTEL_EXPORTER_OTLP_ENDPOINT` |
| **Logs** | CloudWatch Logs | `OBSERVABILITY_ENABLED=true` |
| **Errors** | CrashReporter (mobile) | `apps/chefooz-app/src/observability/` |

### Health Check Endpoints

| Endpoint | Purpose | Used By |
|----------|---------|--------|
| `GET /health` | Basic liveness (200 = running) | Docker HEALTHCHECK, load balancer |
| `GET /health/ready` | Readiness (checks Postgres + MongoDB) | Kubernetes probes, CI/CD |
| `GET /health/info` | Version + environment info | Deployment verification |

---

## Security Architecture

### Authentication

- **Mobile + API**: JWT-based, stored in `expo-secure-store` (hardware-encrypted)
- **Admin Portal**: Separate JWT scope (admin-only claims)
- **OTP delivery**: WhatsApp Cloud API (primary) → Twilio SMS (automatic fallback if WhatsApp fails)
- **Token storage**: Never in `AsyncStorage` — always `expo-secure-store`

### API Security

| Layer | Mechanism |
|-------|----------|
| **Auth guard** | `JwtAuthGuard` on all protected routes |
| **Role guard** | `@Roles('admin')` on admin endpoints |
| **Rate limiting** | Search: 10 req/s, Suggest: 20 req/s, Payment: 5 req/s |
| **CORS** | Origin whitelist in production (localhost + app domains) |
| **Payment** | HMAC-SHA256 webhook signature verification |
| **Webhook** | Razorpay `X-Razorpay-Signature` header validation |
| **Audit trail** | All admin mutations logged to `audit_events` |

### Data Security

- Secrets in environment variables (never committed)
- SSL/TLS for all DB connections in production (`rejectUnauthorized: false` for AWS RDS)
- MongoDB Atlas: TLS enforced at connection level
- Sensitive fields not returned in API responses

---

## Network & API Design

### URL Structure

```
https://api.chefooz.com/api/v1/<module>/<resource>

Examples:
  GET  /api/v1/search/dishes?query=biryani
  POST /api/v1/payment/create-order
  POST /api/v1/payment/verify
  POST /api/v1/payment/webhook        ← Razorpay webhook
  GET  /api/v1/chef/menu/:chefId
  POST /api/v1/cart/add
  GET  /api/v1/feed/personalized
  GET  /health                        ← No /api prefix
  GET  /health/ready
```

### API Response Envelope

All API responses follow a consistent wrapper:

```typescript
{
  success: boolean;
  message: string;
  data?: any;
  errorCode?: string;    // Present on errors
}
```

### Versioning Strategy

- URI versioning: `/api/v1/...`
- Breaking changes: increment to `/api/v2/...`
- Swagger auto-documentation at `/api/docs` (development)
- All controllers use `@ApiTags`, `@ApiResponse`, `@ApiBearerAuth`

---

## Module Dependency Overview

```
                     ┌─────────────────────────────┐
                     │        NOTIFICATION          │
                     │  (orchestrator - hub)        │
                     └──────────┬──────────────────┘
                                │ events
        ┌───────────────────────┼───────────────────────────────┐
        │                       │                               │
┌───────▼──────┐    ┌───────────▼────────┐    ┌───────────────▼──────┐
│   PAYMENT    │    │   SOCIAL MODULE    │    │   MODERATION         │
│  (Razorpay)  │    │ (feed/explore/     │    │  (review queue)      │
│  create-order│    │  search/reels)     │    │                      │
│  verify      │    └────────────────────┘    └──────────────────────┘
│  webhook     │
└──────┬───────┘
       │ on success
       ▼
┌──────────────┐    ┌────────────────────┐    ┌──────────────────────┐
│    ORDER     │───►│   CHEF-ORDERS      │───►│     DELIVERY         │
│  (lifecycle) │    │  (accept/reject)   │    │  (rider assignment   │
│  10 statuses │    │                    │    │   GPS tracking)      │
└──────┬───────┘    └────────────────────┘    └──────────────────────┘
       │ on delivery
       ▼
┌──────────────┐    ┌────────────────────┐    ┌──────────────────────┐
│    REVIEW    │───►│   COMMISSION       │───►│    WITHDRAWAL        │
│  (CRS score) │    │  (ledger credit)   │    │  (payout to chef)    │
└──────────────┘    └────────────────────┘    └──────────────────────┘
                                │
                                ▼
                    ┌────────────────────┐
                    │  RECONCILIATION    │
                    │  (financial audit) │
                    └────────────────────┘
```

---

## Deployment Architecture

### Current State (QA / Pre-Production)

```
Developer Machine
    └── nx run chefooz-apis:serve      → localhost:3333
    └── nx run chefooz-app:start       → Expo Dev Server
    └── nx run chefooz-admin:dev       → localhost:4200

Target (Production AWS)
    ├── ECS/EC2              → chefooz-apis (NestJS)
    ├── Vercel / Amplify     → chefooz-admin (Next.js)
    ├── Expo EAS             → chefooz-app (iOS + Android binaries)
    ├── RDS (Multi-AZ)       → PostgreSQL
    ├── Atlas M10+           → MongoDB
    ├── ElastiCache          → Valkey/Redis cluster
    └── S3 + CloudFront      → Media CDN
```

### Build Commands

```bash
# Mobile app
nx run chefooz-app:start             # Expo dev server
nx run chefooz-app:build:production  # EAS build

# Backend
nx run chefooz-apis:serve            # Dev server
nx run chefooz-apis:build            # Production build

# Admin
nx run chefooz-admin:dev             # Next.js dev
nx run chefooz-admin:build           # Production build

# All tests
nx run-many --target=test
```

### Environment Profiles

| Profile | `CHEFOOZ_ENV` | Databases | Logging | Observability |
|---------|--------------|-----------|---------|--------------|
| `development` | `development` | Local Docker | Verbose | Console only |
| `staging` | `staging` | AWS RDS (staging) | Error+Warn | ADOT → CloudWatch |
| `production` | `production` | AWS RDS (prod) | Error only | ADOT → CloudWatch + X-Ray |

---

[SLICE_COMPLETE ✅]
