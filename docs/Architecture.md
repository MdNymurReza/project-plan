# Architecture.md — Recipe Sharing Mobile App

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Architecture Principles](#2-architecture-principles)
3. [Major Components](#3-major-components)
   - 3.1 [Mobile Client](#31-mobile-client)
   - 3.2 [API Gateway](#32-api-gateway)
   - 3.3 [Core Backend Services](#33-core-backend-services)
   - 3.4 [Data Stores](#34-data-stores)
   - 3.5 [Media Pipeline](#35-media-pipeline)
   - 3.6 [Notification Service](#36-notification-service)
   - 3.7 [Search Service](#37-search-service)
4. [Data Flow](#4-data-flow)
   - 4.1 [Recipe Creation (Online)](#41-recipe-creation-online)
   - 4.2 [Recipe Creation (Offline → Sync)](#42-recipe-creation-offline--sync)
   - 4.3 [Feed & Discovery](#43-feed--discovery)
   - 4.4 [Search](#44-search)
   - 4.5 [Social Interactions (Comments, Ratings, Follows)](#45-social-interactions-comments-ratings-follows)
   - 4.6 [Push Notifications](#46-push-notifications)
5. [Data Models](#5-data-models)
6. [Key Technical Decisions](#6-key-technical-decisions)
7. [Infrastructure & Deployment](#7-infrastructure--deployment)
8. [Security Considerations](#8-security-considerations)
9. [Observability](#9-observability)
10. [Open Questions & Future Considerations](#10-open-questions--future-considerations)

---

## 1. System Overview

The Recipe Sharing App is an **offline-first, community-driven mobile application** for iOS and Android that allows home cooks to create, store, share, and discover personal recipes. The system must support full core functionality without internet connectivity and seamlessly synchronize data when connectivity is restored.

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Mobile Clients (iOS / Android)               │
│                                                                     │
│  ┌──────────────────┐   ┌─────────────────┐   ┌─────────────────┐  │
│  │  Local SQLite DB │   │  Media Cache    │   │  Sync Queue     │  │
│  │  (SQLCipher)     │   │  (Disk / CDN)   │   │  (Outbox)       │  │
│  └──────────────────┘   └─────────────────┘   └─────────────────┘  │
└────────────────────────────────┬────────────────────────────────────┘
                                 │ HTTPS / REST + WebSocket
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                          API Gateway (Kong / AWS API GW)            │
│              Auth · Rate Limiting · Routing · TLS Termination       │
└──────┬──────────────┬──────────────┬──────────────┬────────────────┘
       │              │              │              │
       ▼              ▼              ▼              ▼
  ┌─────────┐  ┌──────────┐  ┌──────────┐  ┌─────────────┐
  │  Auth   │  │  Recipe  │  │  Social  │  │    Feed     │
  │ Service │  │ Service  │  │ Service  │  │   Service   │
  └─────────┘  └──────────┘  └──────────┘  └─────────────┘
       │              │              │              │
       ▼              ▼              ▼              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        Shared Infrastructure                        │
│   PostgreSQL  │  Redis  │  Elasticsearch  │  S3 + CloudFront CDN   │
│   (Primary DB)│  (Cache)│  (Search Index) │  (Media Storage)       │
└─────────────────────────────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────┐
│  Async Workers (Message Queue)   │
│  - Media processing              │
│  - Search index updates          │
│  - Push notification dispatch    │
│  - Feed fan-out                  │
└──────────────────────────────────┘
```

---

## 2. Architecture Principles

| Principle | Rationale |
|---|---|
| **Offline-first** | A core PRD requirement; ≥20% of sessions must work without connectivity. The client is the source of truth for local data; the server reconciles on sync. |
| **Mobile-only at launch** | No web client reduces surface area and allows investment in native-quality UX and deep platform integration (share sheet, camera, push). |
| **Service-oriented backend** | Bounded services per domain (Auth, Recipe, Social, Feed) allow independent scaling, deployment, and ownership without the full overhead of microservices. |
| **Async by default for heavy work** | Photo processing, feed fan-out, search indexing, and notifications are decoupled via a message queue to keep API response times low. |
| **Eventual consistency is acceptable** | Feed freshness, follower counts, and rating averages do not need to be real-time. This allows caching and async updates without distributed transactions. |
| **Security by design** | User-generated content, auth tokens, and offline data require encryption at rest and in transit from day one. |

---

## 3. Major Components

### 3.1 Mobile Client

**Technology: React Native (with Expo managed workflow)**

Chosen over fully native Swift/Kotlin for the following reasons:
- Single codebase for iOS and Android reduces initial team size.
- React Native's ecosystem (Expo, MMKV, WatermelonDB) has mature offline-first libraries.
- Can be ejected to bare workflow if deep native modules are needed later.

#### Key Sub-components

| Sub-component | Technology | Purpose |
|---|---|---|
| **Local Database** | WatermelonDB (SQLite via SQLCipher) | Stores all user recipes, saved recipes, drafts, comments, and user profile data locally. Encrypted at rest. |
| **Sync Engine** | Custom Outbox Pattern + WatermelonDB Sync Adapter | Queues mutations made offline; replays against the server when connectivity returns. Uses server-assigned vector clocks for conflict resolution. |
| **Media Cache** | react-native-fast-image + local disk cache | Caches recipe photos for offline viewing. Implements LRU eviction with a configurable cap (default 500 MB). |
| **State Management** | Zustand (global) + React Query (server state) | Zustand handles UI and offline queue state; React Query manages cache invalidation for online API responses. |
| **Navigation** | React Navigation v6 | Stack and tab-based navigation. |
| **Auth** | expo-secure-store | Stores JWT refresh tokens securely in the platform keychain/keystore. |
| **Push Notifications** | Expo Notifications (APNs + FCM) | Handles foreground and background push events. |
| **External Share** | React Native Share API | Generates deep-link URLs and triggers native share sheet. |

#### Offline-First Strategy

The client uses a **local-write-first** model:

1. All writes (recipe create/edit, comments, ratings, follows) are committed to the local SQLite database immediately.
2. Each write is appended to an **outbox table** with a pending status.
3. A background sync worker monitors connectivity (via NetInfo) and flushes the outbox in FIFO order when online.
4. Server responses update local records with server-canonical IDs and timestamps.
5. Conflicts are resolved with a **last-write-wins** strategy based on `updated_at` timestamps, with a server-side merge log for auditability.

---

### 3.2 API Gateway

**Technology: Kong (self-hosted) or AWS API Gateway**

Responsibilities:
- TLS termination.
- JWT validation and identity injection into upstream service headers.
- Rate limiting per user (e.g., 100 req/min standard, 10 req/min for media upload initiation).
- Request routing to downstream services.
- Logging all requests for audit and debugging.

---

### 3.3 Core Backend Services

All services are written in **Node.js (TypeScript) with Fastify**. Fastify is chosen over Express for its schema-first validation (JSON Schema / Zod), better performance, and built-in TypeScript support. Services communicate with each other via internal HTTP calls for synchronous needs and a shared message broker (RabbitMQ) for asynchronous events.

#### Auth Service

Responsibilities:
- User registration (email/password with bcrypt hashing).
- Social login via OAuth 2.0 (Google, Apple Sign-In — required for App Store compliance).
- Issue short-lived JWTs (access token: 15 min) and long-lived refresh tokens (30 days, stored in Redis with rotation).
- Token revocation (logout, account deletion).
- Password reset flow via email (SendGrid).

#### Recipe Service

Responsibilities:
- CRUD for recipes (title, ingredients, steps, photos references, tags, cook time, cuisine, dietary tags).
- Draft management — drafts are stored in the same table with a `status` enum (`draft | published | deleted`).
- Recipe versioning — a `recipe_versions` table records each edit for potential future audit/restore.
- Generates and manages **public deep-link URLs** (e.g., `recipeshare://recipe/{id}` and `https://app.recipeshare.io/r/{id}`).
- Triggers `recipe.published` and `recipe.updated` events to the message broker for downstream fan-out.
- Photo upload flow: returns pre-signed S3 URLs to the client; records media references after client confirms upload.

#### Social Service

Responsibilities:
- Follow/unfollow relationships (stored as a directed graph in PostgreSQL).
- Comments (threaded, depth-limited to 2 levels at launch).
- Star ratings (1–5); computes and caches recipe aggregate rating in Redis.
- Bookmarks/saves (user ↔ recipe associations).
- Emits events: `user.followed`, `comment.created`, `rating.created`.

#### Feed Service

Responsibilities:
- Generates a **personalized feed** per user showing recipes from followed cooks, ordered by recency.
- Maintains a **discovery feed** (curated by recency and rating for users with fewer than 5 follows).
- Uses a **fan-out on write** (push model) for users with ≤ 1,000 followers; switches to **fan-out on read** (pull model) for high-follower accounts to avoid write amplification.
- Feed entries are materialized per user in Redis sorted sets (score = publish timestamp).
- Feed data is also written to the client's local DB during sync so the feed is available offline.

---

### 3.4 Data Stores

| Store | Technology | Usage |
|---|---|---|
| **Primary Database** | PostgreSQL 15 (RDS Multi-AZ) | All persistent domain data: users, recipes, ingredients, steps, tags, follows, comments, ratings, bookmarks. |
| **Cache** | Redis 7 (ElastiCache cluster) | Feed sorted sets, session/refresh tokens, rating aggregates, frequently-read recipe metadata, rate limit counters. |
| **Search Index** | Elasticsearch 8 | Full-text search over recipe title, ingredients, tags, cuisine, cook time. Kept in sync via async indexing worker. |
| **Object Storage** | AWS S3 + CloudFront CDN | Recipe photos and user avatars. CloudFront edge-caches media globally; signed URLs control access to private drafts. |
| **Message Broker** | RabbitMQ (Amazon MQ) | Durable async event bus between services and workers. |
| **Client Local DB** | SQLite via WatermelonDB (mobile) | Offline recipe, bookmark, and draft data per user device. |

#### PostgreSQL Schema Highlights

- `users`, `recipes`, `ingredients`, `steps` are core normalized tables.
- `recipe_tags` is a join table supporting many-to-many tagging.
- `follows` is a self-referential adjacency list on `users`.
- `sync_log` table records outbox events processed per device for idempotent replay detection.
- All tables include `created_at`, `updated_at` (indexed), and soft-delete via `deleted_at`.

---

### 3.5 Media Pipeline

```
Mobile Client
    │
    │ 1. Request pre-signed S3 upload URL (Recipe Service)
    ▼
Recipe Service ──────────────────────► S3 (private bucket, temp prefix)
    │ 2. Return pre-signed URL
    ▼
Mobile Client
    │ 3. PUT image directly to S3 (bypasses backend)
    │ 4. Notify Recipe Service of successful upload (key reference)
    ▼
Recipe Service publishes `media.uploaded` event
    │
    ▼
Media Worker (async)
    ├── Transcode / resize to multiple resolutions (thumbnail 200px, card 600px, full 1200px)
    ├── Strip EXIF metadata (privacy)
    ├── Move from temp → permanent S3 prefix
    └── Invalidate CloudFront cache if replacing existing media
```

**Why direct-to-S3 upload?** Avoids proxying large binary payloads through the backend, reducing server load and latency.

---

### 3.6 Notification Service

- Listens to broker events: `user.followed`, `comment.created`, `rating.created`, `recipe.published` (for followers).
- Maintains a `device_tokens` table per user (multiple devices supported).
- Dispatches via **FCM** (Android) and **APNs** (iOS) using the Expo Push Notifications SDK server-side helper.
- Respects user notification preference settings stored in the user profile.
- Notification payloads include a deep-link URL for in-app navigation on tap.

---

### 3.7 Search Service

- Elasticsearch index: `recipes` with fields for `title`, `ingredients.name`, `tags`, `cuisine`, `dietary_tags`, `cook_time_minutes`, `author_id`, `published_at`, `average_rating`.
- Index is populated/updated via an **async indexing worker** that consumes `recipe.published`, `recipe.updated`, and `recipe.deleted` events from the broker.
- The Recipe Service proxies search queries to Elasticsearch and returns results enriched with author profile summaries.
- Filters are implemented as Elasticsearch `bool` query `filter` clauses (exact match) combined with `multi_match` for keyword search.
- **Why not PostgreSQL full-text search?** Elasticsearch provides superior relevance scoring, faceted filtering (cuisine, cook time ranges, dietary tags simultaneously), and horizontal scaling for search-heavy workloads as the recipe corpus grows.

---

## 4. Data Flow

### 4.1 Recipe Creation (Online)

```
1. User fills recipe form → local DB write (status: draft)
2. User taps "Publish"
3. Client requests pre-signed S3 URL for each photo → Recipe Service → S3
4. Client uploads photos directly to S3
5. Client POSTs recipe payload (text fields + S3 keys) → API Gateway → Recipe Service
6. Recipe Service writes to PostgreSQL (status: published)
7. Recipe Service responds 201 with canonical recipe ID and server timestamps
8. Client updates local DB record with server ID and clears outbox entry
9. Recipe Service emits `recipe.published` to broker
10. Workers consume event:
    a. Media Worker: transcode images, move to permanent prefix
    b. Search Worker: index recipe in Elasticsearch
    c. Feed Worker: fan-out recipe to followers' feed sorted sets in Redis
    d. (No notification for own publish; notifications sent for follower events)
```

### 4.2 Recipe Creation (Offline → Sync)

```
1. User fills and "publishes" recipe while offline
2. Local DB write (status: published_pending_sync), outbox entry created
3. App displays recipe immediately from local DB (optimistic UI)
4. Connectivity restored → Sync Engine detects outbox entries
5. Sync Engine:
    a. Checks for local photos → uploads to S3 via pre-signed URL
    b. POSTs recipe to Recipe Service with idempot