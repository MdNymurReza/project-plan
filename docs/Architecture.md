# Architecture.md — AI Meeting Notes

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Major Components](#2-major-components)
3. [Data Flow](#3-data-flow)
4. [Infrastructure & Deployment](#4-infrastructure--deployment)
5. [Key Technical Decisions](#5-key-technical-decisions)
6. [Security & Privacy Considerations](#6-security--privacy-considerations)
7. [Scalability & Performance](#7-scalability--performance)
8. [Phase 2 Extension Points](#8-phase-2-extension-points)

---

## 1. System Overview

AI Meeting Notes is a fully browser-based SaaS application that accepts meeting recordings, transcribes them with speaker labeling, and produces structured, editable, shareable notes using AI. The system is designed around an **asynchronous processing pipeline**: files are ingested quickly, heavy computation (transcription and AI summarization) runs in the background, and users are notified when results are ready.

The architecture follows a **decoupled services pattern** with a clear boundary between:

- The **user-facing web layer** (upload, editing, sharing, export)
- The **async processing pipeline** (transcription, diarization, AI note generation)
- The **data layer** (structured metadata, note content, user accounts)
- The **storage layer** (temporary file staging, no long-term raw file retention)

```
┌─────────────────────────────────────────────────────────────────┐
│                        Browser Client                           │
│           (React SPA — upload, editor, dashboard, share)        │
└───────────────────────────┬─────────────────────────────────────┘
                            │ HTTPS / WebSocket
┌───────────────────────────▼─────────────────────────────────────┐
│                        API Gateway                              │
│              (Auth, rate limiting, routing)                     │
└──────┬──────────────────┬──────────────────┬────────────────────┘
       │                  │                  │
┌──────▼──────┐  ┌────────▼───────┐  ┌───────▼───────┐
│  Auth       │  │  Core API      │  │  Sharing &    │
│  Service    │  │  Service       │  │  Export API   │
└─────────────┘  └────────┬───────┘  └───────────────┘
                          │
              ┌───────────▼────────────┐
              │    Job Queue           │
              │  (async processing)    │
              └───────────┬────────────┘
                          │
          ┌───────────────┼───────────────┐
          │               │               │
  ┌───────▼──────┐ ┌──────▼──────┐ ┌─────▼──────────┐
  │ Transcription│ │  AI Notes   │ │  Notification  │
  │  Worker      │ │  Worker     │ │  Worker        │
  └───────┬──────┘ └──────┬──────┘ └────────────────┘
          │               │
  ┌───────▼───────────────▼────────┐
  │         Data Layer             │
  │  PostgreSQL │ Object Storage   │
  └────────────────────────────────┘
```

---

## 2. Major Components

### 2.1 Browser Client (Frontend)

**Technology:** React (TypeScript), TailwindCSS, Tiptap (rich text editor)

Responsibilities:
- **Upload UI:** Chunked multipart file upload (handles files up to 2 GB) with progress indication and client-side MIME-type validation before submission.
- **Dashboard:** Paginated list of all meetings showing title, date, duration, participant count, and processing status.
- **Note Editor:** Rich text editor (Tiptap) rendered from structured note data. Users can edit, annotate, add/remove sections, and manually adjust speaker labels.
- **Real-time status updates:** WebSocket subscription to processing job status events so users see transcription and note generation progress without polling.
- **Sharing views:** View-only and editable share link targets that render notes without requiring authentication for view-only access.
- **Export triggers:** Client initiates export requests (Markdown, PDF, plain text); PDF rendering occurs server-side.

### 2.2 API Gateway

**Technology:** Nginx or AWS API Gateway / Kong

Responsibilities:
- TLS termination
- JWT authentication verification (delegates to Auth Service)
- Rate limiting per user and per IP
- Request routing to downstream services
- Static share link routing (bypasses auth for view-only tokens)

### 2.3 Auth Service

**Technology:** Node.js / Express or dedicated auth provider (e.g., Auth0, Supabase Auth)

Responsibilities:
- Email/password registration and login with bcrypt-hashed passwords
- Google OAuth 2.0 integration
- JWT issuance and refresh token rotation
- Session revocation

**Decision note:** Using a managed auth provider (Auth0 or Supabase Auth) is strongly preferred in Phase 1 to reduce security surface area and accelerate delivery. See §5.

### 2.4 Core API Service

**Technology:** Node.js (TypeScript) with Express or Fastify, or Python (FastAPI)

Responsibilities:
- Receive upload initiation request, generate a pre-signed object storage URL, and return it to the client for direct upload (client uploads directly to object storage, not through the API server).
- Record meeting metadata (user ID, filename, duration estimate, upload timestamp) in PostgreSQL.
- Enqueue a processing job upon upload completion confirmation.
- Serve note read/update endpoints (CRUD on meeting notes).
- Expose share link creation and validation endpoints.
- Serve dashboard list and search endpoints.

### 2.5 Object Storage

**Technology:** AWS S3 (or GCS / Cloudflare R2)

Responsibilities:
- Temporary staging of raw recording files during processing.
- Files are **automatically deleted after successful transcription** (lifecycle policy: e.g., 24-hour TTL after processing confirmed) to comply with the non-goal of no long-term raw video retention.
- Storage of generated export artifacts (PDFs) with short-lived pre-signed download URLs.

### 2.6 Job Queue & Workers

**Technology:** BullMQ on Redis (or AWS SQS for managed option)

The queue decouples ingestion from processing, absorbs traffic spikes, enables retries, and allows independent scaling of workers.

#### 2.6.1 Transcription Worker

**Technology:** Python service (async worker consuming queue jobs)

Responsibilities:
- Pulls the file from object storage.
- For audio extraction: uses FFmpeg to extract audio track from video files before sending to the transcription API (reduces API payload size and cost).
- Sends audio to **third-party Speech-to-Text API with diarization** (e.g., AssemblyAI or Deepgram Nova-2).
- Receives transcript with speaker labels and word-level timestamps.
- Writes raw transcript and speaker segments to PostgreSQL.
- Emits a `transcription_complete` job to the queue.
- Updates job status (visible to frontend via WebSocket).

#### 2.6.2 AI Notes Worker

**Technology:** Python service

Responsibilities:
- Reads the structured transcript from PostgreSQL.
- Constructs a prompt with transcript segments and speaker labels.
- Calls the **LLM API** (OpenAI GPT-4o or Anthropic Claude) with a structured output schema requesting: summary, key decisions, action items (with owner and optional due date), and discussion topics.
- Parses the structured response (JSON mode / tool-call output).
- Writes structured note sections to PostgreSQL.
- Emits a `notes_ready` event triggering user notification.
- Updates job status to `complete`.

#### 2.6.3 Notification Worker

Responsibilities:
- Listens for `notes_ready` events.
- Sends in-app WebSocket push to connected client.
- Sends email notification (via SendGrid or AWS SES) if user is not actively connected.

### 2.7 PostgreSQL Database

Primary relational store. Key tables:

| Table | Key Columns |
|---|---|
| `users` | id, email, password_hash, oauth_provider, created_at |
| `meetings` | id, user_id, title, duration_seconds, participant_count, status, created_at |
| `transcripts` | id, meeting_id, raw_json (word segments + speaker labels), created_at |
| `notes` | id, meeting_id, content_json (structured sections), updated_at |
| `share_links` | id, meeting_id, token, access_level (view/edit), expires_at |
| `jobs` | id, meeting_id, type, status, attempts, last_error, created_at |

`content_json` in `notes` stores the structured note document as a JSON column (sections array), which maps directly to the Tiptap editor document model, avoiding a complex relational schema for note structure while remaining queryable.

For Phase 2 keyword search, a **full-text search index** on transcript text and note content is added (PostgreSQL `tsvector` columns or migration to Elasticsearch/pgvector for semantic search).

### 2.8 WebSocket / Real-time Layer

**Technology:** Socket.io on a dedicated real-time service (or integrated into the Core API behind a sticky-session load balancer)

Responsibilities:
- Authenticated WebSocket connections keyed to `user_id`.
- Workers publish job status events to Redis pub/sub; the WebSocket service broadcasts to the appropriate connected client.
- Enables live progress display (upload confirmed → transcribing → generating notes → complete).

### 2.9 Export Service

**Technology:** Node.js microservice or serverless function

Responsibilities:
- Markdown export: serializes the `content_json` note structure into Markdown text, returned directly.
- PDF export: renders the note via Puppeteer (headless Chrome) or a library like `pdf-lib` / WeasyPrint, uploads PDF to object storage, returns a short-lived pre-signed download URL.
- Plain text export: strips formatting from `content_json`.

---

## 3. Data Flow

### 3.1 Recording Upload & Processing

```
1. User selects file in browser
   └─► Client validates MIME type and file size locally

2. Client calls Core API: POST /meetings (title, filename, size)
   └─► API creates meeting record (status: "uploading")
   └─► API generates pre-signed S3 PUT URL
   └─► Returns { meetingId, uploadUrl } to client

3. Client uploads file directly to S3 (multipart for large files)
   └─► Upload progress shown in UI

4. Upload completes → Client calls Core API: POST /meetings/{id}/upload-complete
   └─► API updates meeting status to "queued"
   └─► API enqueues { jobType: "transcribe", meetingId } in BullMQ

5. Transcription Worker picks up job
   └─► Fetches file from S3
   └─► Extracts audio via FFmpeg (if video)
   └─► Sends audio to AssemblyAI / Deepgram
   └─► Polls or webhooks for transcript result
   └─► Stores transcript JSON in PostgreSQL (transcripts table)
   └─► Updates meeting status to "transcribed"
   └─► Enqueues { jobType: "generate_notes", meetingId }
   └─► Triggers S3 lifecycle policy: delete source file

6. AI Notes Worker picks up job
   └─► Reads transcript from PostgreSQL
   └─► Constructs structured prompt with transcript + speaker labels
   └─► Calls LLM API (GPT-4o) with JSON response schema
   └─► Parses and validates structured output
   └─► Writes note sections to PostgreSQL (notes table)
   └─► Updates meeting status to "complete"
   └─► Publishes "notes_ready:{userId}" event to Redis pub/sub

7. Notification Worker receives event
   └─► WebSocket service pushes status update to connected client
   └─► If client disconnected: sends email notification

8. Client receives WebSocket event → navigates to Note Editor
```

### 3.2 Note Editing & Sharing

```
User edits note in Tiptap editor
└─► Debounced PATCH /notes/{id} calls to Core API (autosave)
└─► Core API writes updated content_json to PostgreSQL

User creates share link
└─► POST /share-links { meetingId, accessLevel: "view" | "edit" }
└─► API generates cryptographically random token, stores in share_links table
└─► Returns shareable URL: https://app.domain.com/s/{token}

Recipient visits share link
└─► API Gateway routes /s/{token} to Core API without auth check
└─► Core API validates token, checks expiry, retrieves note
└─► Returns note content (read-only for "view" links)
└─► View-only render served to unauthenticated user
```

### 3.3 Export

```
User clicks "Export as PDF"
└─► POST /export { meetingId, format: "pdf" }
└─► Export Service renders note HTML → PDF via Puppeteer
└─► PDF uploaded to S3 with 15-minute pre-signed URL
└─► API returns download URL to client
└─► Browser triggers download
```

---

## 4. Infrastructure & Deployment

### Environment Layout

| Environment | Purpose |
|---|---|
| Production | Live user traffic |
| Staging | Pre-release validation, mirrors production config |
| Development | Local developer environments via Docker Compose |

### Deployment Platform

**AWS** (primary recommendation) using:
- **ECS Fargate** for API service, WebSocket service, and worker services (containerized, no server management)
- **ElastiCache (Redis)** for BullMQ job queue and pub/sub
- **RDS PostgreSQL (Multi-AZ)** for the primary database
- **S3** for object storage with lifecycle rules
- **CloudFront** CDN for static frontend assets
- **AWS SES** for transactional email
- **ALB (Application Load Balancer)** in front of API and WebSocket services

Alternatively, this architecture maps cleanly to **GCP Cloud Run** or **Railway / Render** for a lower-ops early-stage deployment.

### Containerization

All services run as Docker containers. A `docker-compose.yml` supports full local development (API, workers, Redis, PostgreSQL, MinIO as local S3 substitute).

### CI/CD

- GitHub Actions pipeline: lint → test → build container → push to ECR → deploy to ECS
- Workers and API deployed independently; queue decoupling means zero-downtime worker deploys

---

## 5. Key Technical Decisions

### 5.1 Direct-to-S3 Upload (Presigned URLs)

**Decision:** The client uploads files directly to S3 using a server-generated pre-signed PUT URL rather than streaming through the API server.

**Why:** Recording files can reach 2 GB. Routing them through the API server would require high memory allocation, saturate application server bandwidth, and introduce a single point of failure for large transfers. Direct upload offloads transfer entirely to S3, which handles multipart uploads natively, provides resumability, and scales independently. The API server only handles small metadata requests.

### 5.2 Asynchronous Processing Pipeline via Job Queue

**Decision:** Transcription and AI note generation are entirely asynchronous, mediated by a persistent job queue (BullMQ / Redis).

**Why:** These operations are time-bound to third-party API latency (speech-to-text for a 60-minute recording can take 1–4 minutes; LLM inference adds additional time). Synchronous HTTP handling of this workload is impractical. A queue provides: retry logic for transient API failures, independent horizontal scaling of workers, visibility into job progress, and dead-letter handling for persistent failures. Users receive real-time feedback via WebSocket without blocking on the result.

### 5.3 Third-Party APIs for Transcription and LLM (No Proprietary ML Infrastructure)

**Decision:** Use AssemblyAI or Deepgram for speech-to-text with diarization; use OpenAI GPT-4o or Anthropic Claude for structured note generation.

**Why:** The PRD explicitly rules out building proprietary ML infrastructure. Both transcription and LLM tasks require significant model expertise and GPU infrastructure to operate at quality parity with frontier APIs. Third-party APIs allow the team to deliver a high-quality product immediately and iterate on prompt engineering rather than model training. Vendor abstraction layers in the worker code allow switching providers (e.g., if cost or accuracy shifts favor a different provider).

**Risk mitigation:** Abstract the transcription API behind an interface (`Transcription