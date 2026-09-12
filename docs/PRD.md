# AI Meeting Notes — Product Requirements Document

---

## Problem Statement

Remote product teams frequently conduct meetings across video conferencing platforms but struggle to capture, organize, and retain actionable information from those sessions. Manual note-taking is inconsistent, distracts participants from active discussion, and often results in incomplete records. Follow-up tasks, decisions, and context are regularly lost, leading to misalignment, duplicated effort, and slower execution. Teams need a reliable, low-friction way to convert meeting recordings into structured, searchable, and shareable notes without leaving the browser.

---

## Goals

- Enable users to upload or link meeting recordings and receive structured, human-readable notes automatically.
- Reduce the time spent on manual note-taking and post-meeting summarization by at least 80%.
- Surface key decisions, action items, and discussion topics in a consistent, predictable format.
- Make meeting notes immediately shareable and accessible to the broader team without additional tooling.
- Deliver a fully browser-based experience requiring no native application installation.

---

## User Stories

**As a product manager,**
I want to upload a meeting recording and receive an automatic summary with action items,
so that I can distribute next steps to the team within minutes of the meeting ending.

**As a remote team member,**
I want to read structured notes from a meeting I missed,
so that I can get up to speed quickly without watching the full recording.

**As a product manager,**
I want action items automatically extracted and attributed to specific participants,
so that accountability is clear without me manually reviewing the transcript.

**As a team lead,**
I want to edit and annotate AI-generated notes before sharing,
so that I can correct errors or add context before the team sees them.

**As a remote team member,**
I want to search across past meeting notes by keyword or topic,
so that I can locate specific decisions or discussions without scrolling through documents.

**As a product manager,**
I want to export notes as a formatted document or copy them to tools like Notion or Confluence,
so that meeting records live where my team already works.

**As a user,**
I want speaker labels attached to transcript segments,
so that I can understand who said what during the meeting.

**As a team lead,**
I want to share a meeting notes link with view-only access,
so that stakeholders can read the notes without needing an account.

---

## Scope

### Phase 1 — Core Experience

- **Recording ingestion:** Browser-based upload of audio and video files (MP4, MOV, M4A, MP3, WAV) up to 2 GB.
- **Transcription:** Automated speech-to-text transcription with speaker diarization (speaker labeling).
- **AI-generated structured notes:** Automatic generation of the following sections from each meeting:
  - Meeting summary (2–5 sentence overview)
  - Key decisions made
  - Action items with owner attribution and optional due dates
  - Discussion topics and themes
- **Note editor:** In-browser rich text editor allowing users to modify, annotate, and format AI-generated notes.
- **Shareable links:** Generate view-only and editable share links for each set of notes.
- **Export:** Export notes as Markdown, PDF, or plain text.
- **Authentication:** Email/password and Google OAuth sign-in.
- **Notes dashboard:** List view of all past meetings with title, date, duration, and participant count.

### Phase 2 — Enhanced Functionality

- **Direct integration with Zoom, Google Meet, and Microsoft Teams** via link or API to pull recordings automatically.
- **Notion and Confluence one-click export.**
- **Keyword and semantic search** across all meeting notes.
- **Custom note templates** so teams can define their own output structure.
- **Team workspaces** with role-based access control (admin, editor, viewer).
- **Slack notifications** when notes are ready or action items are assigned.

---

## Non-Goals

- Building a native desktop or mobile application in any phase.
- Conducting live, real-time transcription during active meetings (recordings only, Phase 1).
- Providing video playback or editing capabilities within the tool.
- Replacing full-featured project management tools for action item tracking.
- Supporting languages other than English in Phase 1.
- Storing raw video recordings long-term; source files are processed and discarded after transcription.
- Building proprietary speech-to-text or language model infrastructure; third-party APIs will be used.

---

## Success Metrics

| Metric | Target |
|---|---|
| Time from upload to structured notes delivered | ≤ 5 minutes for a 60-minute recording |
| Note accuracy rating (user-submitted thumbs up/down) | ≥ 80% positive ratings within 60 days of launch |
| Action item extraction precision | ≥ 85% of extracted items confirmed relevant by users |
| Weekly active users (WAU) at 90 days post-launch | 500 WAU |
| Average notes shared per user per week | ≥ 2 |
| User retention (return within 7 days of first use) | ≥ 40% |
| Export or share action taken per meeting processed | ≥ 60% of sessions |
| Support tickets related to transcription errors | ≤ 5% of processed meetings |