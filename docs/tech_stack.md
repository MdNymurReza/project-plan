# tech_stack.md — Recipe Sharing App

---

## Mobile Client

### React Native (with Expo Managed / Bare Workflow)
**Justification:** The architecture doc explicitly selects React Native + Expo as the mobile client technology. A single codebase targets both iOS and Android, reducing team size at launch. The Expo ecosystem provides first-class access to camera, push notifications (APNs + FCM), the native share sheet, and secure token storage — all required by the design doc's flows. The bare workflow escape hatch satisfies the architecture's note that deep native modules may be needed later.

### WatermelonDB (SQLite via SQLCipher)
**Justification:** The architecture doc mandates an encrypted local database for offline-first storage of recipes, drafts, bookmarks, comments, and user profiles. WatermelonDB's lazy-loading and sync-adapter interface directly support the custom outbox pattern described in section 3.1. SQLCipher satisfies the security-at-rest requirement stated in the architecture principles.

### Zustand
**Justification:** The architecture doc designates Zustand as the global UI and offline queue state manager. Its minimal API keeps the sync engine's outbox state and connectivity status cleanly separated from server-fetched data without the boilerplate overhead of Redux.

### TanStack Query (React Query)
**Justification:** The architecture doc assigns React Query to server-state management and cache invalidation for online API responses. Its stale-while-revalidate semantics align with the eventual-consistency principle and the design doc's requirement to show local results instantly before server responses arrive (e.g., Discover search).

### React Navigation v6
**Justification:** Specified in the architecture doc. Implements the bottom tab bar with five independent navigation stacks described in the design doc's navigation architecture section, including the elevated Create tab and deep-link URL handling for recipe sharing.

### Expo Notifications (APNs + FCM)
**Justification:** Directly specified in the architecture doc for handling foreground and background push events triggered by `user.followed`, `comment.created`, `rating.created`, and `recipe.published` broker events. Required for App Store and Play Store compliance.

### expo-secure-store
**Justification:** Specified in the architecture doc for storing JWT refresh tokens in the platform keychain (iOS) and keystore (Android), satisfying the security-by-design principle for auth token protection.

### react-native-fast-image
**Justification:** Specified in the architecture doc as the media cache layer. Provides LRU disk caching (default 500 MB cap) so recipe photos in the feed, library, and saved collections remain viewable offline, as required by the offline-first UX principle in the design doc.

### React Native Share API (built-in)
**Justification:** Specified in the architecture doc and required by the design doc's share flow — triggers the native share sheet with a deep link (`recipeshare://recipe/{id}`) and plain-text fallback on both iOS and Android.

### React Native NetInfo
**Justification:** Specified in the architecture doc as the connectivity monitor that signals the sync engine to flush the outbox. Enables the passive offline banner and sync status indicator described in the design doc's offline UX section without blocking any UI.

---

## API Gateway

### Kong (self-hosted) or AWS API Gateway
**Justification:** The architecture doc offers both options. Kong is preferred when self-hosted control over plugins (rate limiting, JWT validation, request logging) is required; AWS API Gateway is preferred to reduce operational overhead on a small initial team. Either handles TLS termination, JWT identity injection into upstream headers, per-user rate limiting (100 req/min standard, 10 req/min for media upload initiation), and routing to the four downstream services as described in section 3.2.

---

## Backend Services

### Node.js with TypeScript
**Justification:** The architecture doc mandates TypeScript across all backend services for type safety, IDE tooling, and consistency with the React Native client codebase (shared type definitions for API contracts are practical). Node.js's non-blocking I/O suits the async-heavy workload (outbox sync, media events, feed fan-out).

### Fastify
**Justification:** Explicitly chosen over Express in the architecture doc for schema-first request validation (JSON Schema / Zod integration), measurably higher throughput, and built-in TypeScript support. Schema-first validation is especially important for the Recipe Service, where ingredient/step payloads are structured and must be validated before PostgreSQL writes.

### Zod
**Justification:** Referenced in the architecture doc alongside JSON Schema for request validation in Fastify. Zod provides runtime type safety and generates TypeScript types from the same schema, enabling shared validation logic between backend services and potentially the mobile client.

### bcrypt
**Justification:** Specified in the architecture doc for password hashing in the Auth Service. Industry-standard adaptive hashing algorithm appropriate for email/password registration and login.

### OAuth 2.0 (Google + Apple Sign-In)
**Justification:** The architecture doc requires social login via OAuth 2.0 with Apple Sign-In mandatory for App Store compliance (as reinforced in the design doc's onboarding UX notes). Apple must appear before Google on iOS per App Store Review Guidelines.

### JSON Web Tokens (short-lived access + long-lived refresh)
**Justification:** Specified in the architecture doc — 15-minute access tokens and 30-day refresh tokens stored in Redis with rotation. Paired with `expo-secure-store` on the client for secure token persistence as required by the security-by-design principle.

### SendGrid
**Justification:** Specified in the architecture doc for transactional email delivery in the Auth Service's password reset flow. Managed delivery with deliverability monitoring reduces operational burden.

---

## Data Stores

### PostgreSQL 15 (AWS RDS Multi-AZ)
**Justification:** The architecture doc designates PostgreSQL as the primary persistent store for all domain data: users, recipes, ingredients, steps, tags, follows, comments, ratings, and bookmarks. Multi-AZ deployment satisfies high-availability requirements. The schema design (normalized tables, adjacency list for follows, soft deletes via `deleted_at`, `sync_log` for idempotent replay) is detailed in section 3.4.

### Redis 7 (AWS ElastiCache cluster)
**Justification:** Specified in the architecture doc for feed sorted sets (materialized per user, scored by publish timestamp), refresh token storage with rotation, rating aggregates, frequently-read recipe metadata, and rate limit counters. The Feed Service's fan-out-on-write model depends on Redis sorted sets for sub-millisecond feed reads, supporting the eventual-consistency architecture principle.

### Elasticsearch 8
**Justification:** The architecture doc explicitly selects Elasticsearch over PostgreSQL full-text search for the Search Service, citing superior relevance scoring, faceted filtering (cuisine, cook time ranges, dietary tags simultaneously), and horizontal scalability as the recipe corpus grows. The `recipes` index supports `multi_match` keyword queries combined with `bool` filter clauses for all filter types exposed in the design doc's Discover screen (ingredient, cuisine, dietary tags, cook time range, rating threshold).

### AWS S3 + CloudFront CDN
**Justification:** Specified in the architecture doc for recipe photos and user avatars. Direct-to-S3 client uploads via pre-signed URLs avoid proxying binary payloads through the backend. CloudFront edge-caches media globally for low-latency image loads in the feed and recipe detail views. Signed URLs restrict access to private draft media as required by the security considerations.

### RabbitMQ (Amazon MQ)
**Justification:** Specified in the architecture doc as the durable async event bus connecting services and workers. Decouples the Recipe Service from downstream consumers (media processing, search indexing, feed fan-out, notification dispatch), keeping API response times low as required by the async-by-default architecture principle. Amazon MQ reduces operational management compared to a self-hosted cluster.

---

## Media Pipeline

### Sharp (Node.js image processing)
**Justification:** Used inside the async Media Worker to transcode and resize uploaded photos to three resolutions (thumbnail 200 px, card 600 px, full 1200 px) as described in the architecture doc's media pipeline. Sharp is the fastest pure-Node image processing library, suitable for a worker that processes photos asynchronously after S3 upload.

### AWS S3 Pre-signed URLs
**Justification:** Specified in the architecture doc's media pipeline. The Recipe Service generates pre-signed URLs so the mobile client uploads directly to S3, bypassing the backend and eliminating binary payload proxying load on the Recipe Service.

---

## Notification Service

### Expo Push Notifications Server SDK (`expo-server-sdk-node`)
**Justification:** Specified in the architecture doc for server-side dispatch to APNs (iOS) and FCM (Android). Wraps both platform SDKs behind a unified API, batches notifications, and handles token invalidation — reducing the implementation surface for the Notification Service while supporting multiple devices per user.

---

## Search

### Elasticsearch 8 (shared with Data Stores above)
**Justification:** As described in the architecture doc's Search Service section, the async indexing worker consumes `recipe.published`, `recipe.updated`, and `recipe.deleted` broker events to keep the index current. The Recipe Service proxies enriched search responses to the client. The design doc's 300 ms debounced search input and local-cache-first result display are implemented at the mobile client layer, not requiring any additional server-side technology.

---

## Infrastructure & Deployment

### AWS (primary cloud provider)
**Justification:** The architecture doc references AWS-managed services throughout: RDS Multi-AZ for PostgreSQL, ElastiCache for Redis, S3 for object storage, CloudFront for CDN, and Amazon MQ for RabbitMQ. Consolidating on AWS reduces integration complexity, simplifies IAM-based access control, and enables VPC-level network isolation between services.

### Docker + Docker Compose (local development)
**Justification:** Provides environment parity between local development and production for the four backend services, PostgreSQL, Redis, Elasticsearch, and RabbitMQ. Essential for a service-oriented backend where developers must run multiple services simultaneously.

### Kubernetes (AWS EKS) or AWS ECS (Fargate)
**Justification:** The architecture doc calls for containerized, independently scalable services. ECS Fargate reduces cluster management overhead for a small initial team while still providing per-service horizontal scaling (e.g., scaling Feed Workers independently during high-follower fan-out bursts). EKS is the upgrade path if orchestration complexity warrants it.

### Terraform
**Justification:** Infrastructure-as-code for reproducible provisioning of RDS, ElastiCache, ECS, S3, CloudFront distributions, and Amazon MQ. Supports the architecture's emphasis on independent service deployment and enables staging/production environment parity.

### GitHub Actions
**Justification:** CI/CD pipeline for automated testing, Docker image builds, and deployment to ECS/EKS on merge to main. Supports separate pipelines per service, consistent with the service-oriented deployment model described in the architecture doc.

---

## Security

### TLS (enforced at API Gateway)
**Justification:** The architecture doc mandates TLS termination at the API Gateway layer for all client-server communication. All internal service-to-service traffic within the VPC uses private networking.

### SQLCipher (via WatermelonDB)
**Justification:** Specified in the architecture doc for encrypting the local SQLite database at rest on the mobile device, protecting offline recipe, bookmark, and draft data if a device is compromised.

### AWS IAM + VPC
**Justification:** The architecture doc's security considerations require IAM roles for fine-grained S3 and service access control, with all backend services isolated within a VPC. Pre-signed S3 URLs scope media upload/download permissions to individual operations without exposing bucket credentials.

---

## Observability

### OpenTelemetry (tracing + metrics collection)
**Justification:** Vendor-neutral instrumentation for distributed traces across the four backend services and async workers. Supports the architecture's need to trace request flows through the API Gateway → services → message queue → workers chain.

### AWS CloudWatch (metrics + logs) or Datadog
**Justification:** The architecture doc requires centralized logging of all API Gateway requests and service-level metrics. CloudWatch integrates natively with all AWS-managed services. Datadog is the preferred upgrade if cross-service dashboards, APM, and alerting sophistication are prioritized (e.g., tracking the <1% sync failure rate and ≥99.5% crash-free session success metrics from the PRD).

### Sentry
**Justification:** Crash reporting and error tracking for both the React Native mobile client and the Node.js backend services. Directly supports the PRD's ≥99.5% crash-free session rate target by surfacing regressions immediately on release.

---

## Summary Table

| Layer | Technology | Key Reason |
|---|---|---|
| Mobile framework | React Native + Expo | Specified in architecture; cross-platform, Expo ecosystem |
| Local database | WatermelonDB + SQLCipher | Offline-first encrypted storage with sync adapter |
| Global state | Zustand | Specified; manages outbox and connectivity state |
| Server state / cache | TanStack Query | Specified; stale-while-revalidate, cache invalidation |
| Navigation | React Navigation v6 | Specified; deep links, tab + stack navigation |
| Push notifications (client) | Expo Notifications | Specified; unified APNs + FCM |
| Secure token storage | expo-secure-store | Specified; platform keychain/keystore |
| Image caching | react-native-fast-image | Specified; LRU disk cache for offline media |
| Backend language | Node.js + TypeScript | Specified; async I/O, shared types with client |
| Backend framework | Fastify | Specified; schema-first, higher performance than Express |
| Input validation | Zod | Specified alongside Fastify for schema + type generation |
| Auth hashing | bcrypt | Specified for email/password auth |
| Social auth | OAuth 2.0 (Google, Apple) | Specified; Apple required for App Store |
| API gateway | Kong or AWS API Gateway | Specified; TLS, JWT validation, rate limiting, routing |
| Primary database | PostgreSQL 15 (RDS Multi-AZ) | Specified; all persistent domain data, HA |
| Cache / feed store | Redis 7 (ElastiCache) | Specified; feed sorted sets, tokens, rate limits |
| Search | Elasticsearch 8 | Specified; faceted filtering, relevance scoring |
| Object storage | AWS S3 + CloudFront | Specified; media storage, CDN, pre-signed uploads |
| Message broker | RabbitMQ (Amazon MQ) | Specified; async event bus between services/workers |
| Image processing | Sharp (Node.js) | Media Worker; fastest Node image transcoding |
| Push notifications (server) | expo-server-sdk-node | Specified; unified APNs + FCM dispatch |
| Email | SendGrid | Specified; password reset transactional email |
| Cloud provider | AWS | All managed services reference AWS throughout |
| Containerization | Docker + Docker Compose | Local dev parity for multi-service backend |
| Orchestration | AWS ECS (Fargate) / EKS | Independent per-service scaling |
| Infrastructure-as-code | Terraform | Reproducible environment provisioning |
| CI/CD | GitHub Actions | Per-service pipelines, automated deployment |
| Distributed tracing | OpenTelemetry | Vendor-neutral; traces cross-service flows |
| Metrics + logs | AWS CloudWatch / Datadog | Centralized observability; supports PRD KPIs |
| Crash reporting | Sentry | Mobile + backend; supports 99.5% crash-free target |