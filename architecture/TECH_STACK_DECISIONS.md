# Chefooz — Tech Stack Decisions

**Version:** 1.0  
**Last Updated:** February 2026  
**Scope:** Architecture Decision Records (ADRs) for all major technology choices in the Chefooz platform  
**Business Rule Summary:** This document records the *why* behind every major technology choice in the Chefooz monorepo. Each decision is documented in ADR format with context, the decision made, the alternatives considered, and the rationale. These decisions are final for the QA release and should only be revisited via a new ADR with team consensus.

---

## 📋 Table of Contents

1. [ADR-001: NestJS for Backend API](#adr-001-nestjs-for-backend-api)
2. [ADR-002: Dual Database — PostgreSQL + MongoDB](#adr-002-dual-database--postgresql--mongodb)
3. [ADR-003: Valkey (Redis-Compatible) over Redis](#adr-003-valkey-redis-compatible-over-redis)
4. [ADR-004: Elasticsearch for Search](#adr-004-elasticsearch-for-search)
5. [ADR-005: Expo (Managed) for Mobile](#adr-005-expo-managed-for-mobile)
6. [ADR-006: Nx Monorepo Toolchain](#adr-006-nx-monorepo-toolchain)
7. [ADR-007: Razorpay as Payment Gateway](#adr-007-razorpay-as-payment-gateway)
8. [ADR-008: Resend for Email](#adr-008-resend-for-email)
9. [ADR-009: BullMQ for Job Queues](#adr-009-bullmq-for-job-queues)
10. [ADR-010: OpenTelemetry + AWS ADOT for Observability](#adr-010-opentelemetry--aws-adot-for-observability)
11. [ADR-011: TanStack Query for Client Data Fetching](#adr-011-tanstack-query-for-client-data-fetching)
12. [ADR-012: Zustand for Mobile State Management](#adr-012-zustand-for-mobile-state-management)
13. [ADR-013: SWC for TypeScript Compilation](#adr-013-swc-for-typescript-compilation)
14. [ADR-014: Expo Router for Mobile Navigation](#adr-014-expo-router-for-mobile-navigation)
15. [ADR-015: Material UI (MUI) for Admin Portal](#adr-015-material-ui-mui-for-admin-portal)
16. [ADR-016: AWS MediaConvert for Video Transcoding](#adr-016-aws-mediaconvert-for-video-transcoding)
17. [ADR-017: OTP-Only Authentication (No Passwords)](#adr-017-otp-only-authentication-no-passwords)
18. [ADR-018: JWT Stored in expo-secure-store](#adr-018-jwt-stored-in-expo-secure-store)
19. [Tech Stack Summary Table](#tech-stack-summary-table)

---

## ADR-001: NestJS for Backend API

| Attribute | Value |
|-----------|-------|
| **Status** | ✅ Accepted |
| **Date** | Early 2025 |
| **Deciders** | Core backend team |

### Context

We needed a backend framework for a production-grade API that would serve mobile, web, and admin clients. The API needed to support 52+ modules with consistent patterns, dependency injection, background jobs, and websockets.

### Decision

Use **NestJS 11** as the backend framework.

### Rationale

| Factor | Why NestJS wins |
|--------|----------------|
| **Modularity** | Module system enforces domain boundaries — 52 modules all follow the same `module.ts + controller + service + dto` pattern without convention drift |
| **TypeScript-first** | Full TypeScript from day 1 with decorators for guards, interceptors, pipes |
| **Dependency Injection** | Built-in DI container eliminates manual wiring; makes unit testing clean with `@nestjs/testing` |
| **Ecosystem** | First-class `@nestjs/typeorm`, `@nestjs/mongoose`, `@nestjs/bull`, `@nestjs/schedule`, `@nestjs/swagger` — no adapter glue needed |
| **OpenAPI** | Auto-generated Swagger documentation from decorators (`@ApiTags`, `@ApiResponse`, `@ApiBearerAuth`) |
| **Consistent patterns** | Guards, Interceptors, Pipes, and Filters are framework-level — every module uses the same security, logging, and validation primitives |
| **Team familiarity** | Team has prior NestJS production experience |

### Alternatives Considered

| Alternative | Why Rejected |
|-------------|-------------|
| **Express.js** | Too low-level; no DI, no module system, inconsistent patterns across 52 modules would lead to high maintenance overhead |
| **Fastify** | Good performance, but lacks the module/DI ecosystem NestJS provides; building module architecture from scratch not worth it |
| **Hono** | Too new, minimal ecosystem, not suitable for a complex 52-module API needing TypeORM, Mongoose, Bull integrations |

---

## ADR-002: Dual Database — PostgreSQL + MongoDB

| Attribute | Value |
|-----------|-------|
| **Status** | ✅ Accepted |
| **Date** | Early 2025 |
| **Deciders** | Full architecture team |

### Context

Chefooz serves two fundamentally different data access patterns:
1. **Transactional data** — orders, payments, commissions, user accounts, chef profiles, delivery records. Must have ACID guarantees, foreign key integrity, and support for complex joins.
2. **Content/media data** — reels, stories, chat messages, activity feeds. High write volume, flexible schemas (reel metadata changes frequently), and TTL-managed documents.

### Decision

Use **PostgreSQL** (AWS RDS) for transactional data and **MongoDB Atlas** for content/media data.

### Rationale

#### PostgreSQL for Transactional Data

| Reason | Explanation |
|--------|-------------|
| **ACID transactions** | Order creation, payment capture, and commission calculation must be atomic. Partial failures must roll back cleanly |
| **Foreign key constraints** | Chef profiles must reference users; order items must reference menu items. Referential integrity enforced by DB |
| **TypeORM integration** | Code-first entity design with NestJS TypeORM works seamlessly. Migrations track schema changes |
| **Complex joins** | Commission reports and reconciliation queries join orders + chef profiles + payments + ledger — SQL joins are the right tool |
| **AWS RDS managed** | Automated backups, Multi-AZ failover, read replicas when needed |

#### MongoDB for Content Data

| Reason | Explanation |
|--------|-------------|
| **Schema flexibility** | Reel documents have evolved — hashtags, location, AR filters, collaboration fields. MongoDB handles schema changes without migrations |
| **High-write throughput** | Reel view count increments, story viewer tracking, activity feed writes — MongoDB handles these without row-level locks |
| **TTL indexes** | Stories expire after 24 hours. MongoDB's TTL index auto-deletes documents. PostgreSQL would need a cron job |
| **Document model** | A chat message naturally serializes as a document (reactions, reply_to, read_receipts nested). Flat tables would require many joins |
| **Atlas managed** | Global cluster, automatic backups, performance advisor |

### Alternatives Considered

| Alternative | Why Rejected |
|-------------|-------------|
| **PostgreSQL only** | JSON columns can handle flexible data, but performance at scale for high-write content (millions of reel view increments) is poor vs MongoDB |
| **MongoDB only** | Cannot do ACID transactions across order + payment + commission atomically. Lack of referential integrity for financial data is a compliance risk |
| **PlanetScale** | MySQL dialect, no TypeORM support for branching workflow, limited NestJS ecosystem |

---

## ADR-003: Valkey (Redis-Compatible) over Redis

| Attribute | Value |
|-----------|-------|
| **Status** | ✅ Accepted |
| **Date** | Mid 2025 |
| **Deciders** | Backend + infrastructure team |

### Context

We needed an in-memory data store for: job queues (BullMQ), API response caching, distributed locking (delivery assignment), rate limiting, and pub/sub. The question was which Redis-compatible option to use.

### Decision

Use **Valkey** on **AWS ElastiCache Serverless** as the Redis-compatible backend.

### Rationale

| Factor | Reasoning |
|--------|----------|
| **Cost** | ElastiCache Serverless (Valkey) scales to zero — no minimum instance cost vs provisioned Redis cluster |
| **Redis-compatible** | 100% API-compatible with Redis 7. `ioredis` client works without changes. BullMQ works without changes |
| **AWS-native** | Same VPC, IAM security, CloudWatch monitoring as the rest of AWS infrastructure — no separate cloud account/billing |
| **TLS + ACL auth** | Enterprise-grade security: ACL username+password auth + TLS in transit. Supported in ElastiCache Serverless |
| **Cluster mode** | Hash tag prefix `{bull}:` ensures BullMQ keys hash to the same cluster slot — required for multi-key operations |
| **No vendor lock-in** | Valkey (Linux Foundation fork) is open source; if we migrate off AWS, the same ioredis config works with any Redis 7-compatible server |

### Alternatives Considered

| Alternative | Why Rejected |
|-------------|-------------|
| **Redis Cloud** | Additional monthly cost on top of AWS; separate vendor to manage; no benefit over ElastiCache in AWS-native deployment |
| **AWS ElastiCache for Redis** | Provisioned instances require minimum node size even at low traffic. Serverless (Valkey) is cheaper for pre-production and spiky loads |
| **Upstash** | Managed Redis-compatible, good API, but extra vendor; does not have the same VPC placement as RDS/ECS |

---

## ADR-004: Elasticsearch for Search

| Attribute | Value |
|-----------|-------|
| **Status** | ✅ Accepted |
| **Date** | Mid 2025 |
| **Deciders** | Backend team |

### Context

The core search experience — customers searching for dishes, chefs, reels — needs to be fast, typo-tolerant, and relevance-ranked. Early implementation used PostgreSQL `ILIKE` but this was inadequate for:
- Fuzzy matching ("buter chiken" → "Butter Chicken")
- Geospatial filtering (dishes within 10km)
- TF/IDF relevance ranking
- Multi-field search across title, description, hashtags

### Decision

Use **Elasticsearch 8+** as the primary search engine, with MongoDB/PostgreSQL fallback.

### Rationale

| Feature | Elasticsearch advantage |
|---------|------------------------|
| **Fuzzy matching** | `fuzziness: AUTO` handles typos, phonetic variations without custom code |
| **Geospatial** | `geo_distance` filter is a native query type — no PostGIS extension needed |
| **Relevance scoring** | TF/IDF + BM25 ranking gives contextually relevant results; `_score` boosts better matches |
| **Multi-field search** | `multi_match` across `title`, `description`, `hashtags`, `chefName` in one query |
| **Aggregations** | Faceted search (cuisine type, dietary tags, price range) via ES aggregations |
| **Autocomplete** | Edge n-gram tokenizer for real-time search suggestions |
| **Fallback available** | All search endpoints fall back to MongoDB/PostgreSQL if ES is unavailable — no hard dependency |

### Alternatives Considered

| Alternative | Why Rejected |
|-------------|-------------|
| **PostgreSQL full-text search** | No fuzzy matching, no geospatial, no relevance tuning; `tsvector` is good but insufficient for this use case |
| **Algolia** | Excellent product but expensive at scale ($1+ per 1000 search operations); not cost-appropriate for a launch-stage startup |
| **Typesense** | Promising open-source alternative, but less mature ecosystem and fewer advanced geospatial features than ES 8 |
| **MeiliSearch** | Good for simple text search but lacks the geospatial and aggregation capabilities needed |

---

## ADR-005: Expo (Managed) for Mobile

| Attribute | Value |
|-----------|-------|
| **Status** | ✅ Accepted |
| **Date** | Early 2025 |
| **Deciders** | Mobile team |

### Context

Chefooz needed a mobile app for iOS and Android with: video playback/recording, maps/location, camera, push notifications, and complex navigation. The choice was between React Native CLI (bare), Expo Managed, or Expo Bare workflow.

### Decision

Use **Expo Managed Workflow** (SDK 53).

### Rationale

| Factor | Reasoning |
|--------|----------|
| **OTA updates** | Expo EAS Update lets us push JavaScript-layer fixes without App Store review. Critical for a startup needing rapid iteration |
| **SDK ecosystem** | `expo-video`, `expo-location`, `expo-image-picker`, `expo-audio`, `expo-secure-store` — all battle-tested, maintained by Expo team |
| **No native linking** | Managed workflow means no manual `pod install` / Gradle config for the team. Lower maintenance burden |
| **EAS Build** | Cloud-based iOS/Android builds without needing a Mac in CI (no macOS runner cost for Android builds) |
| **Camera & media** | Expo Camera and expo-video cover reel recording and playback requirements without native modules |
| **Web compatibility** | Expo supports `expo-router` on web for potential future web MVP of the customer app |

### Constraints (accepted)

- Cannot use arbitrary native modules that require prebuild
- All new native capabilities must use Expo SDK or request explicit approval
- No custom native Objective-C/Swift/Kotlin code without switching to Bare workflow

### Alternatives Considered

| Alternative | Why Rejected |
|-------------|-------------|
| **React Native CLI (bare)** | Full control, but significant native build maintenance overhead — not justified at this team size |
| **Flutter** | Strong performance but separate language (Dart), no code sharing with web admin/backend TypeScript ecosystem |
| **Native iOS + Android** | Highest performance but 2x codebase maintenance; not viable for a small cross-functional team |

---

## ADR-006: Nx Monorepo Toolchain

| Attribute | Value |
|-----------|-------|
| **Status** | ✅ Accepted |
| **Date** | Early 2025 |
| **Deciders** | Full team |

### Context

The project has 5 applications (apis, app, admin, 2x e2e) and 5 shared libraries. Managing these as separate repositories would mean duplicating types, utilities, and build configurations, and making cross-app changes cumbersome.

### Decision

Use **Nx 22** as the monorepo toolchain.

### Rationale

| Feature | Value |
|---------|-------|
| **Affected builds** | `nx affected:build` only rebuilds changed projects and their dependents — drastically faster CI |
| **Task graph** | Nx understands that `chefooz-app` depends on `libs/api-client` depends on `libs/types` — runs tasks in the right order |
| **Code generators** | Custom generators for creating modules, components, API clients consistently (same structure every time) |
| **Caching** | Computation caching — identical inputs produce cached outputs. Repeated builds on unchanged code are near-instant |
| **Shared libs** | `libs/types`, `libs/api-client`, `libs/ui` are first-class project members with proper type-checking and import paths |
| **Consistent tooling** | One `jest.config.ts`, one `tsconfig.base.json`, one `eslint.config.mjs` for the entire repo |

### Alternatives Considered

| Alternative | Why Rejected |
|-------------|-------------|
| **Turborepo** | Good option, but less mature than Nx for multi-framework (NestJS + Expo + Next.js) monorepos; fewer generators |
| **Separate repositories** | Cross-cutting type changes require coordinated PRs across 5 repos. Shared lib versioning becomes a maintenance burden |
| **Yarn/pnpm workspaces only** | No task graph, no affected computation, no generators — just package hoisting |

---

## ADR-007: Razorpay as Payment Gateway

| Attribute | Value |
|-----------|-------|
| **Status** | ✅ Accepted |
| **Date** | Mid 2025 |
| **Deciders** | Product + backend team |

### Context

Chefooz operates exclusively in India (INR). Payment methods must include UPI (the dominant payment method in India), credit/debit cards, net banking, and digital wallets. International payment gateways have inferior UPI support.

### Decision

Use **Razorpay** as the sole payment gateway (INR only).

### Rationale

| Factor | Reasoning |
|--------|----------|
| **UPI-first** | Razorpay has the best UPI integration — deep VPA support, UPI Intent flow for mobile, UPI AutoPay for subscriptions |
| **Indian regulatory compliance** | Razorpay is a licensed Payment Aggregator regulated by RBI — handles PCI-DSS, NPCI, RBI mandates on our behalf |
| **Developer API** | Razorpay's REST API and webhook system are well-documented; HMAC-SHA256 signature verification is clean |
| **Dashboard** | Real-time payment analytics, dispute management, refund processing via Razorpay dashboard |
| **Payout API** | Razorpay X (payout product) can be used for chef payouts (NEFT/IMPS/UPI) — same vendor for in+out money movement |
| **INR focus** | No multi-currency complexity needed; Razorpay is INR-native |

### Webhook Security

```typescript
// HMAC-SHA256 verification
const generatedSignature = crypto
  .createHmac('sha256', process.env.RAZORPAY_KEY_SECRET)
  .update(`${orderId}|${paymentId}`)
  .digest('hex');
  
if (generatedSignature !== receivedSignature) throw new UnauthorizedException();
```

### Alternatives Considered

| Alternative | Why Rejected |
|-------------|-------------|
| **Stripe** | Excellent developer experience but inferior UPI support; UPI is 70%+ of Indian transactions — this is a non-starter |
| **PayU** | India-native but worse developer documentation and SDK quality than Razorpay |
| **Cashfree** | Good API but less market penetration and fewer supported payment methods than Razorpay |
| **Paytm** | Complicated regulatory history in 2024; dependent on Paytm Payments Bank (shut down); compliance risk |

---

## ADR-008: Resend for Email

| Attribute | Value |
|-----------|-------|
| **Status** | ✅ Accepted |
| **Date** | Mid 2025 |
| **Deciders** | Backend team |

### Context

The platform needs transactional emails for: order receipts, payout confirmations, OTP fallback, and review requests. Email volume is low (thousands, not millions per day). We need a simple, reliable API without complex SMTP setup.

### Decision

Use **Resend** for all transactional emails.

### Rationale

| Factor | Reasoning |
|--------|----------|
| **Developer-first API** | Single `resend.emails.send()` call with TypeScript SDK — no SMTP config, no server setup |
| **Transactional philosophy** | Resend is built specifically for transactional email — no campaign/marketing features bleeding into the API |
| **React Email compatible** | Official support for `@react-email/components` — email templates as React components (consistent with our stack) |
| **Generous free tier** | 3,000 emails/month free. Sufficient for QA launch. Paid plans are competitive |
| **Deliverability** | Resend uses dedicated IPs and actively manages sender reputation |
| **Simple API** | `https://api.resend.com` — REST POST. No SDK bloat |

### Alternatives Considered

| Alternative | Why Rejected |
|-------------|-------------|
| **SendGrid** | Industry standard but more complex API for simple transactional use; higher cost at lower volumes |
| **AWS SES** | Good deliverability, AWS-native, but requires SMTP or complex SDK setup; no template management UI |
| **Postmark** | Excellent for transactional email but costs ~$1.25 per 1000 emails from the start (no free tier) |
| **Nodemailer + SMTP** | DIY setup requires managing SMTP servers, deliverability, bounce handling — not worth it |

---

## ADR-009: BullMQ for Job Queues

| Attribute | Value |
|-----------|-------|
| **Status** | ✅ Accepted |
| **Date** | Early 2025 |
| **Deciders** | Backend team |

### Context

The platform has several async workloads that must not block HTTP responses:
- Video transcoding job dispatch
- Push notification batching (up to 500 recipients per reel publish)
- Delivery assignment with retry logic
- Elasticsearch index sync

These need reliable at-least-once execution, retry with backoff, priority queuing, and failure visibility.

### Decision

Use **BullMQ** (via `@nestjs/bull` / `bull`) with Valkey as the backing store.

### Rationale

| Feature | Value |
|---------|-------|
| **Redis-backed persistence** | Jobs survive server restarts; stored in Valkey until acknowledged |
| **Retry with backoff** | Configurable attempts, delay, and exponential backoff — MediaConvert jobs auto-retry on transient failures |
| **Job priorities** | Notification jobs can be prioritized (payment-related > social activity) |
| **Concurrency control** | Worker concurrency is configured per queue — prevents overwhelming external APIs |
| **Delayed jobs** | Schedule review prompt 30 minutes after delivery (`delay: 1800000`) |
| **NestJS integration** | `@nestjs/bull` provides `@Processor`, `@Process` decorators — jobs are DI-aware |
| **Bull Board** | Optional UI for monitoring queue health (available in dev/staging) |

### Alternatives Considered

| Alternative | Why Rejected |
|-------------|-------------|
| **AWS SQS as job queue** | Good for distributed systems, but adds latency, no built-in job prioritization, more complex consumer setup |
| **Agenda (MongoDB)** | MongoDB-backed job queue; adds load to MongoDB which is already handling content at volume |
| **Custom cron + DB flag** | No retry logic, no distributed locking, no worker isolation — poor reliability |

---

## ADR-010: OpenTelemetry + AWS ADOT for Observability

| Attribute | Value |
|-----------|-------|
| **Status** | ✅ Accepted (Phase 3.7.1) |
| **Date** | Late 2025 |
| **Deciders** | Backend + infrastructure team |

### Context

As we approach QA/production, we need distributed tracing, application metrics, and centralized logging. The system spans NestJS, multiple databases, AWS services, and external APIs — traces need to correlate across all of these.

### Decision

Use **OpenTelemetry SDK** (vendor-neutral) with **AWS ADOT Collector** forwarding to **CloudWatch Logs + CloudWatch Metrics + X-Ray**.

### Rationale

| Factor | Reasoning |
|--------|----------|
| **Vendor neutrality** | OpenTelemetry is the CNCF standard — if we want to switch from X-Ray to Jaeger or Honeycomb, only the exporter changes; no app code changes |
| **AWS-native export** | ADOT (AWS Distro for OpenTelemetry) is the officially supported bridge from OTel to CloudWatch/X-Ray — maintained by AWS |
| **Single pane** | CloudWatch Logs + X-Ray Traces + CloudWatch Metrics — one AWS console for all observability data |
| **Pre-NestJS init** | OTel SDK initializes before NestJS so that all NestJS operations (including module initialization) are traced |
| **Sampling control** | `OTEL_SAMPLING_PERCENTAGE` env var controls cost — 100% in staging, reduced in production |
| **Dev mode** | `ConsoleSpanExporter` and `ConsoleLogRecordExporter` in development — no ADOT needed locally |
| **Graceful shutdown** | `shutdownOpenTelemetry()` called on SIGTERM ensures all pending spans are exported before process exits |

### Alternatives Considered

| Alternative | Why Rejected |
|-------------|-------------|
| **Datadog** | Excellent product but expensive ($23+/host/month); external vendor added cost for a launch-stage startup |
| **New Relic** | Similar cost concerns; adds another vendor in the stack |
| **Sentry** | Good for error tracking but not distributed tracing; would need to combine with another solution |
| **Self-hosted Jaeger** | Requires running and maintaining Jaeger infrastructure on AWS — operational burden not justified |

---

## ADR-011: TanStack Query for Client Data Fetching

| Attribute | Value |
|-----------|-------|
| **Status** | ✅ Accepted |
| **Date** | Early 2025 |
| **Deciders** | Mobile + admin team |

### Context

Both the mobile app and admin portal need a consistent approach to: fetching data from the API, caching responses, showing loading/error states, invalidating cache after mutations, and refetching stale data.

### Decision

Use **TanStack Query v5** (`@tanstack/react-query`) in both `chefooz-app` (mobile) and `chefooz-admin` (Next.js).

### Rationale

| Feature | Value |
|---------|-------|
| **Intelligent caching** | `staleTime: 30s` — data is fresh for 30 seconds, serving cached results instantly; `gcTime: 5min` for memory management |
| **Automatic refetch** | Data refetches on window focus and network reconnect — critical for mobile users who background/foreground the app |
| **Background updates** | Stale data shown immediately while fresh data loads in background (not a blank loading screen) |
| **Optimistic updates** | Cart additions show immediately while POST request is in-flight |
| **Retry logic** | `retry: 1` — one retry on transient failures before showing error state |
| **Shared hooks via libs/api-client** | React Query hooks live in `libs/api-client` and are consumed by both mobile and admin — one implementation, two consumers |
| **DevTools** | `@tanstack/react-query-devtools` for debugging cache state in development |

### Query Key Convention

```typescript
['orders', userId]           // user-scoped
['reels', reelId]            // item-specific
['chef', 'menu', chefId]    // hierarchical
['search', 'dishes', hash]  // parameterized
```

### Alternatives Considered

| Alternative | Why Rejected |
|-------------|-------------|
| **SWR** | Similar capabilities, but TanStack Query has richer mutation handling, better TypeScript inference, and more active development |
| **Redux Toolkit Query** | RTK Query is powerful but adds Redux overhead to mobile; Zustand + TanStack Query covers the same ground without Redux boilerplate |
| **Apollo Client** | GraphQL client — not applicable since Chefooz uses REST |

---

## ADR-012: Zustand for Mobile State Management

| Attribute | Value |
|-----------|-------|
| **Status** | ✅ Accepted |
| **Date** | Early 2025 |
| **Deciders** | Mobile team |

### Context

The mobile app needs client-side state for: authentication (JWT), cart, multi-step onboarding wizard, camera settings, video player, profile mode (customer/chef toggle), upload pipeline, and UI state (modals, toasts).

### Decision

Use **Zustand v5** for all mobile state management.

### Rationale

| Factor | Reasoning |
|--------|----------|
| **Minimal boilerplate** | Zustand store is just a `create()` call with state and actions — no reducers, action creators, or providers required |
| **No context wrapping** | TanStack Query covers server state; Zustand covers client state. No need to wrap components in context |
| **React Native compatible** | Works identically on React Native and web — no platform-specific adjustments |
| **Devtools support** | `zustand/middleware/devtools` for Redux DevTools integration during development |
| **Persist middleware** | `zustand/middleware/persist` with `expo-secure-store` adapter for JWT persistence |
| **Small bundle** | ~1.4KB gzipped — important for mobile app initial load |
| **11 focused stores** | One store per domain (cart, auth, media, etc.) — modular, independently testable |

### Alternatives Considered

| Alternative | Why Rejected |
|-------------|-------------|
| **Redux Toolkit** | More powerful but significant boilerplate overhead; team velocity priority favors Zustand for a startup |
| **Jotai** | Atom-based model is excellent but adds more conceptual overhead than Zustand's slice model; team familiarity factor |
| **Context API + useReducer** | No devtools, no persistence middleware, renders issues at scale — not suitable for 11 distinct state domains |
| **MobX** | Observable model is powerful but requires class decorators; conflicts with our functional/arrow-function preference |

---

## ADR-013: SWC for TypeScript Compilation

| Attribute | Value |
|-----------|-------|
| **Status** | ✅ Accepted |
| **Date** | Early 2025 |
| **Deciders** | DevOps + backend team |

### Context

The monorepo has 52+ NestJS modules, each with multiple TypeScript files. The default `tsc` compiler is slow for large projects — long initial build times and slow Jest runs hurt developer velocity.

### Decision

Use **SWC** (`@swc/core`) for TypeScript compilation across the monorepo.

### Rationale

| Factor | Reasoning |
|--------|----------|
| **20x faster** | SWC is a Rust-based compiler; typical Jest runs go from ~60s to ~3s for the same test suite |
| **Jest transform** | `@swc/jest` as the Jest transformer replaces `ts-jest` — no type-checking during tests (type checks run separately) |
| **NestJS compatibility** | `@swc/cli` supports decorators (TypeScript `experimentalDecorators` mode) — NestJS `@Module`, `@Controller`, `@Injectable` all work correctly |
| **Same output** | SWC produces identical JavaScript output to `tsc` for the features we use |
| **Nx integration** | Nx has native SWC support in the `@nx/js` plugin; `@swc/core` version is managed centrally in `package.json` |

### Trade-offs (Accepted)

- SWC does **not** type-check — TypeScript errors only surface from separate `tsc --noEmit` runs (separate CI step)
- Some advanced TypeScript features (conditional types, complex mapped types) may behave slightly differently in edge cases

### Alternatives Considered

| Alternative | Why Rejected |
|-------------|-------------|
| **ts-jest** | 10-20x slower than SWC; was the previous default but replaced for velocity |
| **esbuild** | Faster than SWC for bundling but less mature decorator support for NestJS |
| **babel + @babel/preset-typescript** | Slower than SWC; no first-class TypeScript decorator support without plugins |

---

## ADR-014: Expo Router for Mobile Navigation

| Attribute | Value |
|-----------|-------|
| **Status** | ✅ Accepted |
| **Date** | Early 2025 |
| **Deciders** | Mobile team |

### Context

The mobile app has 30+ screens across customer, chef, and rider roles. Navigation needs to support: tab navigation, stack navigation, modals, deep linking, and role-based route protection.

### Decision

Use **Expo Router v5** (file-system based routing).

### Rationale

| Feature | Value |
|---------|-------|
| **File-system routing** | Screen at `src/app/orders/[id].tsx` automatically maps to route `/orders/:id` — zero manual registration |
| **Deep linking** | Expo Router handles `chefooz://orders/123` deep links automatically via file structure |
| **Type-safe navigation** | `router.push('/orders/123')` with TypeScript path inference — no string typos |
| **Layout files** | `_layout.tsx` at each directory level allows stacked providers, guards, and tab config |
| **Web compatible** | Same routing works on web via Next.js adapter — future web version of the app uses the same screens |
| **Auth guards** | Root `_layout.tsx` handles auth state and redirects unauthenticated users to `/auth/login` |

### Alternatives Considered

| Alternative | Why Rejected |
|-------------|-------------|
| **React Navigation v7** | Industry standard but requires manual route registration and extra setup for deep linking and type safety |
| **React Native Navigation (Wix)** | Requires native module integration — incompatible with Expo Managed workflow |

---

## ADR-015: Material UI (MUI) for Admin Portal

| Attribute | Value |
|-----------|-------|
| **Status** | ✅ Accepted |
| **Date** | Mid 2025 |
| **Deciders** | Admin team |

### Context

The admin portal needs a rich set of pre-built components: data tables, forms, dialogs, charts, badges, and color-coded status indicators. Development speed is the priority — internal tools should be built quickly.

### Decision

Use **Material UI (MUI) v7** for all admin portal components.

### Rationale

| Factor | Reasoning |
|--------|----------|
| **Component richness** | `DataGrid`, `Dialog`, `Chip`, `Alert`, `Card`, `Autocomplete`, `DateRangePicker` — all enterprise-grade and built-in |
| **Next.js 15 compatible** | MUI v7 supports React 19 and Next.js App Router with proper SSR support |
| **Theming system** | Color-coded status system (🟢🟠🔴) built via MUI `Palette` and `Chip` `color` prop |
| **Table support** | `DataGrid` handles large financial reconciliation tables with sorting, filtering, pagination out of the box |
| **Team velocity** | Team has prior MUI production experience — zero learning curve |
| **Accessibility** | MUI components are WCAG-compliant — internal tool compliance requirement |

### Alternatives Considered

| Alternative | Why Rejected |
|-------------|-------------|
| **shadcn/ui + Tailwind** | Excellent for consumer apps but requires more assembly for complex data tables/grids; not the right choice for data-heavy admin portal |
| **Ant Design** | Strong table/form components but Chinese company concerns and slightly dated visual design |
| **Chakra UI** | Good DX but lacks the enterprise data grid and date picker capabilities needed for reconciliation dashboards |

---

## ADR-016: AWS MediaConvert for Video Transcoding

| Attribute | Value |
|-----------|-------|
| **Status** | ✅ Accepted |
| **Date** | Mid 2025 |
| **Deciders** | Backend + infrastructure team |

### Context

Chefs upload videos that need to be transcoded to multiple quality levels (720p/1080p) in HLS format before publishing as reels. Mobile devices have varying network conditions — adaptive bitrate streaming via HLS is required.

### Decision

Use **AWS MediaConvert** for server-side video transcoding.

### Rationale

| Factor | Reasoning |
|--------|----------|
| **Managed service** | No FFmpeg server to provision, scale, or maintain |
| **HLS output** | Native HLS (HTTP Live Streaming) output with multiple quality levels (adaptive bitrate) |
| **S3 integration** | Input from `S3_INPUT_BUCKET`, output to `S3_OUTPUT_BUCKET` — no data egress from AWS |
| **SQS notifications** | Job completion events via SQS — our cron polls every 1 minute for completed jobs |
| **Pay-per-minute** | Charged per minute of video processed — zero cost when no videos are being transcoded |
| **Thumbnail extraction** | MediaConvert can extract thumbnail frames — reduces need for separate FFmpeg calls |
| **Global availability** | Available in the same AWS region as our other infrastructure |

### Alternatives Considered

| Alternative | Why Rejected |
|-------------|-------------|
| **Self-hosted FFmpeg** | Cost-effective at scale but requires managing EC2 instances, auto-scaling, queue management — too much ops overhead for launch |
| **Cloudinary** | Good video API but expensive at video volume; external vendor outside AWS ecosystem |
| **Mux** | Excellent developer experience but pricing is high for a startup; external service vs AWS-native |
| **AWS Elastic Transcoder** | Legacy AWS service; MediaConvert is the recommended replacement with better codec support |

---

## ADR-017: OTP-Only Authentication (No Passwords)

| Attribute | Value |
|-----------|-------|
| **Status** | ✅ Accepted |
| **Date** | Early 2025 |
| **Deciders** | Product + security team |

### Context

The platform is mobile-first, targeting Indian consumers. Phone-number based OTP login is the dominant pattern in Indian apps (Swiggy, Zomato, Ola all use this). Password-based auth adds friction and security complexity (password resets, breach exposure).

### Decision

Use **OTP-only authentication** via SMS to the user's phone number.

### Rationale

| Factor | Reasoning |
|--------|----------|
| **Frictionless onboarding** | One phone number entry + OTP = account created. No password to forget or reset |
| **Indian UX norm** | Indian users expect phone-number-based login; password auth feels outdated in the food delivery context |
| **Security** | Eliminates password breach risk, credential stuffing attacks, and weak password vulnerabilities |
| **Phone as identity** | Phone number is the verified identifier; no email required (email only for transactional notifications) |
| **OTP TTL** | 5-minute expiry + rate limiting prevents brute force attacks |

### Trade-offs (Accepted)

- SMS delivery cost (per OTP sent)
- Dependency on SMS gateway reliability
- Users without phone access cannot log in (edge case for target market)

---

## ADR-018: JWT Stored in expo-secure-store

| Attribute | Value |
|-----------|-------|
| **Status** | ✅ Accepted |
| **Date** | Early 2025 |
| **Deciders** | Security + mobile team |

### Context

After OTP authentication, the JWT access token and refresh token need to be persisted on the device so users stay logged in across app restarts.

### Decision

Store JWT tokens exclusively in **`expo-secure-store`** (hardware-backed encryption).

### Rationale

| Factor | Reasoning |
|--------|----------|
| **Hardware-backed** | `expo-secure-store` uses iOS Keychain (Secure Enclave) and Android Keystore — tokens encrypted at hardware level |
| **Not extractable** | Unlike `AsyncStorage` (plain text SQLite), tokens stored in Secure Store cannot be read by other apps or extracted from device backups |
| **OWASP compliance** | OWASP MASVS requires tokens be stored in secure system storage, not plain text — `expo-secure-store` satisfies this |
| **Expo Managed compatible** | Available as a first-class Expo SDK API — no custom native module needed |
| **Zero extra config** | `SecureStore.setItemAsync('jwt', token)` — same API as `AsyncStorage` but hardware-secured |

### Policy

```
✅ expo-secure-store → for JWT accessToken, refreshToken
❌ AsyncStorage     → never for auth tokens
❌ MMKV             → never for auth tokens (not hardware-backed)
❌ memory only      → never (tokens lost on app restart)
```

---

## Tech Stack Summary Table

| Layer | Technology | Version | Decision |
|-------|-----------|---------|---------|
| **Backend framework** | NestJS | 11 | ADR-001 |
| **Backend language** | TypeScript | ~5.9 | (project default) |
| **Transactional DB** | PostgreSQL (AWS RDS) | 15+ | ADR-002 |
| **Content DB** | MongoDB Atlas | 7+ | ADR-002 |
| **Cache / Queue** | Valkey (AWS ElastiCache) | Redis-compatible | ADR-003 |
| **Job queue** | BullMQ | via @nestjs/bull | ADR-009 |
| **Search engine** | Elasticsearch | 8+ | ADR-004 |
| **Mobile framework** | Expo (Managed) | SDK 53 | ADR-005 |
| **Mobile navigation** | Expo Router | ~5.1.7 | ADR-014 |
| **Mobile language** | React Native + TypeScript | 0.79.6 / React 19 | (project default) |
| **Admin framework** | Next.js | 15 (App Router) | (Next.js standard) |
| **Admin UI library** | Material UI (MUI) | v7 | ADR-015 |
| **Client data fetching** | TanStack Query | v5 | ADR-011 |
| **Mobile state** | Zustand | v5 | ADR-012 |
| **Monorepo toolchain** | Nx | 22 | ADR-006 |
| **TS compiler** | SWC | ~1.5.7 | ADR-013 |
| **Payment gateway** | Razorpay | latest | ADR-007 |
| **Email service** | Resend | latest | ADR-008 |
| **Push notifications** | Expo Push API | (Expo managed) | — |
| **Video transcoding** | AWS MediaConvert | — | ADR-016 |
| **Media storage** | AWS S3 | — | (AWS standard) |
| **Observability** | OpenTelemetry + ADOT | — | ADR-010 |
| **Auth storage (mobile)** | expo-secure-store | — | ADR-018 |
| **Auth strategy** | OTP + JWT | — | ADR-017 |
| **OTP delivery (primary)** | WhatsApp Cloud API (Meta) | — | ADR-017 |
| **OTP delivery (fallback)** | Twilio SMS | — | ADR-017 |

---

[SLICE_COMPLETE ✅]
