# Architecture.md — Recipe Sharing Mobile App

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Architectural Principles](#2-architectural-principles)
3. [Major Components](#3-major-components)
   - 3.1 [Mobile Client](#31-mobile-client)
   - 3.2 [Backend Services](#32-backend-services)
   - 3.3 [Data Stores](#33-data-stores)
   - 3.4 [Infrastructure & Platform Services](#34-infrastructure--platform-services)
4. [Data Flow](#4-data-flow)
   - 4.1 [Recipe Creation (Online)](#41-recipe-creation-online)
   - 4.2 [Recipe Creation (Offline → Sync)](#42-recipe-creation-offline--sync)
   - 4.3 [Community Feed & Discovery](#43-community-feed--discovery)
   - 4.4 [Social Interactions & Push Notifications](#44-social-interactions--push-notifications)
5. [Offline-First Strategy](#5-offline-first-strategy)
   - 5.1 [Local Storage Model](#51-local-storage-model)
   - 5.2 [Sync Engine](#52-sync-engine)
   - 5.3 [Conflict Resolution](#53-conflict-resolution)
6. [Key Technical Decisions](#6-key-technical-decisions)
7. [Security Considerations](#7-security-considerations)
8. [Scalability & Reliability](#8-scalability--reliability)
9. [Component Dependency Diagram](#9-component-dependency-diagram)

---

## 1. System Overview

The Recipe Sharing App is a mobile-first, offline-capable platform that allows home cooks to create, organize, and share personal recipes. The system is designed around an **offline-first** philosophy: every core action—creating, editing, browsing, and reading recipes—works without network access, with data reconciled against the cloud transparently when connectivity is restored.

At a high level, the system consists of:

- A **cross-platform mobile client** (iOS & Android) with an embedded local database and sync engine.
- A **cloud backend** composed of focused microservices handling users, recipes, social interactions, search, media, and notifications.
- A **managed cloud infrastructure** providing object storage, a relational database, a search index, a message broker, and a push notification gateway.

```
┌─────────────────────────────────────────────────────┐
│                   Mobile Client                     │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────┐  │
│  │  Local DB   │  │  Sync Engine │  │  UI Layer │  │
│  └─────────────┘  └──────────────┘  └───────────┘  │
└─────────────────────────┬───────────────────────────┘
                          │ HTTPS / WebSocket
┌─────────────────────────▼───────────────────────────┐
│                  API Gateway (REST)                 │
└──┬───────────┬───────────┬────────────┬─────────────┘
   │           │           │            │
┌──▼──┐  ┌────▼───┐  ┌────▼──┐  ┌─────▼──────┐
│User │  │Recipe  │  │Social │  │Notification│
│Svc  │  │Svc     │  │Svc    │  │Svc         │
└──┬──┘  └────┬───┘  └────┬──┘  └─────┬──────┘
   │          │            │           │
┌──▼──────────▼────────────▼───────────▼──────────────┐
│         Shared Infrastructure                       │
│  PostgreSQL │ Redis │ S3 │ Elasticsearch │ FCM/APNs  │
└─────────────────────────────────────────────────────┘
```

---

## 2. Architectural Principles

| Principle | Rationale |
|---|---|
| **Offline-first** | ≥ 25% of sessions are expected to be fully offline; the device is the primary source of truth for the user's own data. |
| **Eventually consistent** | The system tolerates temporary divergence between local and server state, resolving conflicts deterministically on sync. |
| **Single responsibility services** | Each backend service owns a bounded domain; this limits blast radius and enables independent scaling and deployment. |
| **Optimistic UI** | User actions are applied locally immediately and reconciled in the background, giving a fast, responsive feel. |
| **Secure by default** | Authentication tokens, media access, and API endpoints are protected; user data is encrypted at rest on device. |
| **Cross-platform code sharing** | A single shared codebase for both iOS and Android reduces maintenance surface without sacrificing native feel. |

---

## 3. Major Components

### 3.1 Mobile Client

The mobile client is the primary surface for all user interaction. It is built with **React Native** (see decision rationale in §6) and structured into the following internal layers.

#### 3.1.1 UI Layer

- Built with React Native components targeting iOS and Android.
- Follows a feature-based folder structure: `features/recipes`, `features/feed`, `features/social`, `features/profile`, `features/notifications`.
- Navigation managed by **React Navigation** with a tab-bar root and stack sub-navigators per feature.
- State management via **Redux Toolkit** with feature-scoped slices; async thunks dispatch to the local repository layer, not directly to the network.

#### 3.1.2 Local Repository Layer

Acts as the single source of truth for the client. The UI always reads from and writes to this layer; the Sync Engine handles propagation to the server.

- **SQLite via expo-sqlite / Watermelon DB**: Stores recipes, ingredients, instructions, tags, user profiles, comments, likes, follow relationships, and the sync change log.
- **File system cache**: Stores downloaded and locally captured photo files.
- **Async Storage / Secure Store**: Stores authentication tokens, user preferences, and small configuration values.

#### 3.1.3 Sync Engine

A background service within the client responsible for bidirectional synchronization. Detailed in §5.

#### 3.1.4 Media Manager

Handles photo capture (device camera), selection (photo library), local resizing/compression before upload, and caching of remote images.

- Uses `expo-image-picker` for capture/selection.
- Compresses images to a maximum of 1200 × 1200 px and 85% JPEG quality before upload.
- Downloaded community recipe images are cached to disk with an LRU eviction policy (configurable limit, default 500 MB).

#### 3.1.5 Push Notification Handler

Registers device tokens with the Notification Service on login and handles foreground/background notification receipt, routing the user to the relevant recipe or comment on tap.

---

### 3.2 Backend Services

All services are deployed as containerized workloads behind the API Gateway. Each service is independently deployable and communicates with others via the internal message broker (async events) or direct HTTP calls (synchronous queries).

#### 3.2.1 API Gateway

- Single HTTPS entry point for all mobile client traffic.
- Responsibilities: TLS termination, JWT validation, rate limiting, request routing, response caching headers.
- Technology: **Kong** or **AWS API Gateway** (managed option preferred for operational simplicity at launch).

#### 3.2.2 User Service

Owns all identity and profile data.

| Responsibility | Detail |
|---|---|
| Registration / login | Email+password and OAuth (Google, Apple Sign-In). Issues short-lived JWTs (15 min) + refresh tokens (90 days). |
| Profile management | Display name, bio, profile photo, follower/following counts. |
| Follow graph | Stores follow relationships; emits `user.followed` events. |
| Token refresh | Validates refresh tokens, rotates on each use. |

#### 3.2.3 Recipe Service

The core domain service. Owns all recipe data on the server side.

| Responsibility | Detail |
|---|---|
| CRUD for recipes | Create, read, update, soft-delete. Recipes have a `visibility` flag: `private` or `public`. |
| Versioning | Each recipe mutation increments a `server_version` counter used for conflict resolution. |
| Ingredient & instruction management | Stored as ordered JSON arrays within the recipe record. |
| Tagging | Tags are stored in a normalized `tags` table; many-to-many join with recipes. |
| Sync endpoint | Accepts batched change-sets from the client sync engine; returns server-authoritative records. |

#### 3.2.4 Media Service

Decouples photo handling from business logic.

- Accepts multipart photo uploads from the client.
- Stores originals in **S3-compatible object storage**.
- Triggers async generation of thumbnail variants (320 px, 640 px) via a worker queue (e.g., **BullMQ** on Redis).
- Returns CDN-backed URLs for each variant to the Recipe Service, which stores them in the recipe record.

#### 3.2.5 Feed & Discovery Service

Generates personalized and global community feeds and powers search.

| Responsibility | Detail |
|---|---|
| Community feed | Aggregates public recipes from followed users + ranked global feed. Fan-out on write for small follower counts; fan-out on read for high-follower accounts (hybrid). |
| Search | Delegates full-text search (title, ingredients, tags) to **Elasticsearch**. Recipes are indexed asynchronously when published or updated. |
| Trending / recommended | Basic ranking by recency and like count in v1; extensible for ML ranking later. |

#### 3.2.6 Social Service

Handles all engagement features.

| Responsibility | Detail |
|---|---|
| Likes | Idempotent like/unlike operations. Emits `recipe.liked` events. |
| Comments | Threaded comments (single level in v1). Emits `recipe.commented` events. |
| Bookmarks | Records a user's saved community recipes; instructs the sync engine to download the full recipe for offline access. |

#### 3.2.7 Notification Service

Listens to domain events from the message broker and dispatches push notifications.

- Consumes: `recipe.liked`, `recipe.commented`, `user.followed`.
- Looks up device tokens from User Service.
- Sends via **Firebase Cloud Messaging (FCM)** for Android and **Apple Push Notification Service (APNs)** for iOS.
- Stores a notification history record per user for in-app notification inbox.

---

### 3.3 Data Stores

| Store | Technology | Owned By | Purpose |
|---|---|---|---|
| Primary relational DB | **PostgreSQL 15** | All services (separate schemas or databases per service) | Recipes, users, comments, likes, follows, bookmarks, tags |
| Cache / message broker | **Redis 7** | API Gateway, Feed Service, Media Service | Rate limit counters, session tokens, BullMQ job queues, feed cache |
| Object storage | **AWS S3** (or compatible) | Media Service | Original and resized recipe photos |
| CDN | **CloudFront** (or equivalent) | Media Service | Low-latency global photo delivery |
| Search index | **Elasticsearch 8** | Feed & Discovery Service | Full-text recipe search |
| Local device DB | **SQLite** (via WatermelonDB) | Mobile Client | All on-device data |
| Local file cache | Device file system | Mobile Client | Cached photos |

---

### 3.4 Infrastructure & Platform Services

- **Container orchestration**: Kubernetes (EKS or GKE) for production. Docker Compose for local development.
- **CI/CD**: GitHub Actions — lint, test, build, and deploy pipelines. Mobile builds via Expo EAS Build.
- **Observability**: Structured JSON logging to a log aggregation service (e.g., Datadog or ELK). Distributed tracing via OpenTelemetry. Crash reporting on the client via **Sentry**.
- **Secrets management**: AWS Secrets Manager (or equivalent). No secrets in source code or environment files.
- **Feature flags**: LaunchDarkly (or open-source equivalent) for controlled rollouts.

---

## 4. Data Flow

### 4.1 Recipe Creation (Online)

```
User fills form → UI dispatches action
  → Local Repository writes recipe (status: synced_pending_upload)
  → UI immediately shows recipe in local list (optimistic)
  → Sync Engine picks up pending record
  → POST /recipes to Recipe Service (with photo multipart to Media Service)
  → Recipe Service persists, returns server_id + server_version
  → Media Service returns CDN URLs
  → Sync Engine updates local record (status: synced, server_id populated)
  → If recipe published: Recipe Service emits recipe.published event
    → Feed & Discovery Service indexes recipe in Elasticsearch
    → Feed Service adds to follower feeds
```

### 4.2 Recipe Creation (Offline → Sync)

```
User fills form → UI dispatches action
  → Local Repository writes recipe (status: pending_sync, local_id assigned)
  → Change log entry created (operation: CREATE, entity: recipe, local_id)
  → UI immediately shows recipe (optimistic, no network call)

[Later — connectivity restored]
  → Network monitor detects online state
  → Sync Engine reads change log, ordered by timestamp
  → For each pending change: POST/PATCH/DELETE to appropriate service
  → On success: update local record status to synced, store server_id/version
  → On conflict: apply conflict resolution policy (see §5.3)
  → On persistent failure after retries: mark as sync_error, surface to user
```

### 4.3 Community Feed & Discovery

```
User opens Feed tab
  → UI requests feed from Local Repository (cached feed records)
  → Simultaneously Sync Engine fetches latest feed page from Feed & Discovery Service
  → New/updated recipes written to local cache
  → UI reactively updates via WatermelonDB observable queries

User performs search
  → If online: query Feed & Discovery Service → Elasticsearch → ranked results
  → If offline: query local SQLite FTS5 index over downloaded recipes
  → Results rendered; user can save (bookmark) any recipe

User bookmarks a recipe
  → Social Service records bookmark
  → Sync Engine downloads full recipe payload + photos to local DB/file cache
  → Recipe now available fully offline
```

### 4.4 Social Interactions & Push Notifications

```
User A likes User B's recipe
  → POST /likes to Social Service (optimistic local increment)
  → Social Service persists like, emits recipe.liked event to message broker
  → Notification Service consumes event
  → Notification Service fetches User B's device token from User Service
  → Sends push via FCM/APNs
  → User B's device receives push, Notification Service stores in-app record
  → User B taps notification → deep link opens recipe detail screen
```

---

## 5. Offline-First Strategy

### 5.1 Local Storage Model

**WatermelonDB** is used as the local database. It is built on SQLite, is React Native native, and is designed for high-performance, reactive offline-first apps.

Key local tables:

| Table | Key Columns |
|---|---|
| `recipes` | `local_id`, `server_id`, `user_id`, `title`, `description`, `serving_size`, `prep_time`, `cook_time`, `visibility`, `server_version`, `sync_status`, `updated_at_local` |
| `ingredients` | `local_id`, `recipe_local_id`, `name`, `quantity`, `unit`, `order` |
| `instructions` | `local_id`, `recipe_local_id`, `step_number`, `body` |
| `tags` | `local_id`, `name` |
| `recipe_tags` | `recipe_local_id`, `tag_local_id` |
| `feed_items` | `server_recipe_id`, `author_id`, `cached_at`, `expires_at` |
| `comments` | `local_id`, `server_id`, `recipe_id`, `body`, `sync_status` |
| `likes` | `local_id`, `recipe_id`, `sync_status` |
| `change_log` | `id`, `operation (CREATE/UPDATE/DELETE)`, `entity_type`, `local_id`, `payload_snapshot`, `created_at`, `status (pending/synced/error)` |

`sync_status` values: `local_only` | `pending_sync` | `synced` | `sync_error`

### 5.2 Sync Engine

The Sync Engine runs as a background task