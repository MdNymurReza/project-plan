# design.md — Recipe Sharing App

---

## 1. Design Philosophy & UX Principles

### Core Principles

| Principle | Application |
|---|---|
| **Offline-transparent** | The UI never blocks the user due to connectivity state. Sync status is communicated passively, never intrusively. Offline mode feels identical to online mode for all core tasks. |
| **Low-friction capture** | Recipe creation must feel as fast as jotting a note. Reduce required fields, support progressive disclosure, and allow saving at any point. Target ≤ 4 minutes to publish. |
| **Content-first** | Recipe photos, ingredients, and steps are the hero content. UI chrome is minimal. Typography and whitespace do the heavy lifting. |
| **Optimistic by default** | All writes (create, edit, bookmark, follow, comment, rate) reflect immediately in the UI without waiting for server confirmation. Errors surface quietly and are retryable. |
| **Personal and communal** | The app should feel like a well-organized personal cookbook that also happens to be connected to a community — not a social media feed that also stores recipes. |
| **Progressive disclosure** | Advanced fields (dietary tags, cuisine, cook time) appear after the essentials (title, ingredients, steps). Don't overwhelm first-time creators. |

---

## 2. Navigation Architecture

The app uses a **bottom tab bar** as the primary navigation shell with five persistent destinations. All tabs maintain independent navigation stacks.

```
┌──────────────────────────────────────┐
│                                      │
│           Screen Content             │
│                                      │
│                                      │
├──────────────────────────────────────┤
│  🏠 Feed  │ 🔍 Discover │ ➕ Create │ 📚 Library │ 👤 Profile │
└──────────────────────────────────────┘
```

| Tab | Icon | Purpose |
|---|---|---|
| **Feed** | Home | Recipes from followed cooks + discovery content for new users |
| **Discover** | Search | Search, filter, and browse the full recipe community |
| **Create** | Plus (elevated/accent) | Entry point to recipe creation; always accessible |
| **Library** | Bookmark | Personal recipe collection + saved recipes from others |
| **Profile** | Avatar | Own profile, settings, follower/following counts |

The **Create tab** uses a visually distinct button (larger, accent-colored) to signal it as a primary action — similar to the FAB pattern but anchored to the tab bar for single-thumb reachability.

---

## 3. Key User Flows

### 3.1 Onboarding & Registration

```
App Launch
    │
    ├─► [Splash / Brand moment — 1.5s max]
    │
    ├─► New User ──► [Welcome Screen]
    │                    │
    │              ┌─────┴──────┐
    │              │            │
    │         [Sign Up]    [Log In]
    │              │
    │       ┌──────┴──────┐
    │       │             │
    │  [Email/Password] [Google / Apple]
    │       │
    │  [Profile Setup — single screen]
    │       • Display name (required)
    │       • Avatar photo (optional, skippable)
    │       • Dietary preferences (optional, skippable)
    │       │
    │  [Follow Suggestions — curated cooks to seed feed]
    │       • Show 6–10 suggested cooks
    │       • "Skip" available prominently
    │       │
    │  [Feed] ◄── Onboarding complete
    │
    └─► Returning User ──► [Feed] (token still valid)
                      └──► [Log In] (token expired)
```

**UX Notes:**
- Apple Sign-In is required for App Store compliance and must appear before other social options on iOS.
- Profile setup and follow suggestions are each single, scrollable screens — not a multi-step wizard with progress bars, which adds perceived friction.
- Dietary preferences set here power the personalized discovery feed. Label the benefit ("We'll show you more vegetarian recipes") to encourage completion.

---

### 3.2 Recipe Creation

This is the most critical flow for the "≤ 4 minutes to publish" success metric.

```
[Create Tab Tap]
    │
    ▼
[Recipe Editor — single long-form scrollable screen]
    │
    ├── Title field (autofocus on open)
    ├── Cover Photo (tap to add — camera or library)
    ├── Ingredients
    │       • One ingredient per row
    │       • "Add ingredient" taps to add new row
    │       • Quantity + Unit + Name inline
    ├── Steps
    │       • Numbered automatically
    │       • Optional step photo per step
    │       • "Add step" appends new row
    ├── [Expand for more details — collapsed by default]
    │       • Cook time
    │       • Cuisine
    │       • Dietary tags (multi-select chips)
    │       • Serving size
    │
    ├── [Save Draft] — always visible in header (top right)
    ├── [Publish] — sticky button at bottom of screen
    │
    └── On Publish:
            ├── Validates required fields (title + ≥ 1 ingredient)
            ├── If offline: Saves as pending_sync, shows "Saved — will publish when online" toast
            └── If online: Uploads photos → submits → success state
```

**UX Notes:**
- The editor is a **single scrollable canvas**, not a multi-step wizard. Users can jump between sections freely.
- Ingredient and step rows are **inline-editable** — tapping activates the field, swipe-left reveals a delete option.
- "Save Draft" is always accessible so users never fear losing work mid-session.
- The **Publish button is sticky** at the bottom of the screen and becomes active as soon as a title is entered.
- After publish, show a **success card** (recipe thumbnail + "Published!" message) with quick-action buttons: "Share," "View Recipe," "Create Another."
- If the publish fails due to a network error, show an inline error with a "Try Again" option — never a blocking modal.

#### Recipe Editor — Screen Layout

```
┌─────────────────────────────────┐
│ ← Back           [Save Draft]  │  ← Header
├─────────────────────────────────┤
│                                 │
│   [Tap to add cover photo  ]   │  ← 16:9 photo area
│                                 │
├─────────────────────────────────┤
│ Recipe Title                    │  ← Prominent, large type
│ ─────────────────────────────── │
│                                 │
│ INGREDIENTS                     │
│  ○  1  cup  flour          [×]  │
│  ○  2  tsp  baking powder  [×]  │
│  + Add ingredient               │
│                                 │
│ STEPS                           │
│  1. [Preheat oven to 350°F]     │
│  2. [Mix dry ingredients…]      │
│  + Add step                     │
│                                 │
│ ▸ More details (cook time…)    │  ← Expandable section
│                                 │
│ ┌─────────────────────────────┐ │
│ │         Publish             │ │  ← Sticky CTA
│ └─────────────────────────────┘ │
└─────────────────────────────────┘
```

---

### 3.3 Recipe Discovery & Search

```
[Discover Tab]
    │
    ▼
[Discover Screen]
    ├── Search bar (autofocuses on tab tap if user came via search intent)
    ├── Filter row (horizontal scrollable chips): All · Vegetarian · Quick (<30 min) · Italian · …
    ├── [Default state: trending / recently published grid]
    │
    └── [On Search Input]
            • Results update as user types (debounced 300ms)
            • Results show: recipe card grid
            • Filter chips remain active alongside search
            │
            └── [Filter Sheet — tap "Filters" button]
                    • Ingredient (text input)
                    • Cuisine (single-select list)
                    • Dietary tags (multi-select chips)
                    • Cook time (range slider: 0–180 min)
                    • Rating (≥ 3★, ≥ 4★, ≥ 4.5★)
                    • [Apply Filters] sticky button
                    • [Reset] text link
```

**UX Notes:**
- Filter chips above results provide one-tap filtering for the most common refinements (dietary tags, quick meals). The "Filters" button opens a bottom sheet for advanced options.
- Search results use a **2-column card grid** in browse mode, switching to a **single-column list** when showing filtered results with cook time and rating visible on each card.
- An empty state (no results) shows: illustration + "No recipes match your filters" + "Clear filters" button. Never a blank screen.
- Search is powered by Elasticsearch but the client shows **instant local results** from the cached local DB before the server responds, then merges/updates.

---

### 3.4 Feed

```
[Feed Tab]
    │
    ▼
[Feed Screen]
    │
    ├── [New User / <5 follows] → Discovery Feed (mixed trending + suggestions)
    │       └── "Follow more cooks to personalize your feed" nudge card
    │
    └── [Established User] → Personalized Feed
            • Vertical scroll of Recipe Cards from followed cooks
            • Each card: cover photo, title, author avatar + name, cook time, rating
            • Pull-to-refresh (online only; silently no-ops offline)
            • Offline banner: subtle top strip "You're offline — showing saved feed"
```

**Feed Card — Anatomy:**

```
┌───────────────────────────────┐
│                               │
│       Cover Photo (16:9)      │
│                               │
├───────────────────────────────┤
│ 🧑 Sarah Chen                 │  ← Author row (tappable → profile)
│ Lemon Ricotta Pancakes        │  ← Title
│ ⏱ 25 min  ·  ★ 4.7 (34)     │  ← Metadata row
│                        [🔖]  │  ← Bookmark (optimistic toggle)
└───────────────────────────────┘
```

---

### 3.5 Recipe Detail View

Accessed by tapping any recipe card from Feed, Discover, Library, or a user's profile.

```
[Recipe Detail]
    │
    ├── [Hero photo — full-width, 16:9]
    │       └── Swipeable gallery if multiple photos
    ├── Title (large, prominent)
    ├── Author row: avatar, name [Follow/Following toggle]
    ├── Metadata strip: cook time · servings · cuisine · dietary tags
    ├── Rating summary: ★★★★☆ 4.3 (128 ratings) + [Rate this recipe] CTA
    │
    ├── INGREDIENTS (tab or section)
    │       • Ingredient list with optional servings scaler (1×, 2×, 4×)
    │
    ├── STEPS (tab or section)
    │       • Numbered steps, full-width readable
    │       • Optional step photos inline
    │
    ├── COMMENTS
    │       • Comment count summary
    │       • Top-level comments with replies (collapsed at 2 levels)
    │       • [Add a comment…] input pinned above keyboard
    │
    └── Action Bar (sticky bottom):
            [🔖 Save]  [⭐ Rate]  [↗ Share]  [… More]
```

**UX Notes:**
- Ingredients and Steps are sections in a single continuous scroll — no tabs, as tabs hide content and interrupt cooking flow.
- The **serving scaler** (1×/2×/4×) instantly updates all ingredient quantities inline — a high-value micro-interaction.
- The **Action Bar** is always visible, even while reading steps mid-scroll.
- "Rate" opens a bottom sheet with a 5-star picker and optional short text. Submits optimistically.
- "Share" triggers the native share sheet with both a deep link (`recipeshare://recipe/{id}`) and a plain-text fallback with title and URL.
- For recipes authored by the current user, "… More" reveals "Edit" and "Delete" options. For others, it reveals "Report."

---

### 3.6 User Profile

```
[Profile Screen — own or others']
    │
    ├── [Header]
    │       • Avatar (large)
    │       • Display name
    │       • Followers · Following counts (tappable → lists)
    │       • Short bio (if set)
    │       • [Follow / Following] button (other's profile)
    │       • [Edit Profile] button (own profile)
    │
    └── [Recipe Grid — 2-column, published recipes]
            • Tap any recipe → Recipe Detail
            • Own profile: includes drafts as a separate section above published
```

---

### 3.7 Library (Personal Collection)

```
[Library Tab]
    │
    ├── Segment control: [My Recipes] | [Saved]
    │
    ├── [My Recipes]
    │       • All published recipes by the user
    │       • Drafts section (if any) shown at top
    │       • [+ New Recipe] button (shortcut to Create flow)
    │
    └── [Saved]
            • All bookmarked recipes from other cooks
            • Available offline (synced to local DB)
            • Pull-to-refresh online; static when offline
```

---

### 3.8 Offline Experience

**The app must feel intentional offline, not broken.**

| State | Visual Treatment |
|---|---|
| **Fully offline** | Subtle, non-blocking banner at top: "You're offline" in muted color. No modals, no blocking screens. |
| **Pending sync items** | Recipes, comments, or ratings created offline show a small clock icon (⏳) indicating pending sync. |
| **Sync in progress** | Small animated indicator in the header or status bar. No spinner blocking the UI. |
| **Sync complete** | Brief "All caught up ✓" toast. Auto-dismisses in 2 seconds. |
| **Sync conflict** | Edge case: show an in-app notification card in the Library: "One recipe had a conflict. We kept the most recent version." with a "View" link. |
| **Action unavailable offline** | If the user tries an action that requires connectivity (e.g., browsing Discover beyond the local cache), show an inline message: "Connect to the internet to discover more recipes." with a retry button. Never a blocking modal. |

---

## 4. Component & Interaction Patterns

### Recipe Card
- **Default state:** Cover photo, title, author, cook time, rating, bookmark toggle.
- **Bookmark toggle:** Heart or bookmark icon; optimistic fill animation on tap; reverts with shake animation on error.
- **Long-press:** Contextual menu — Save, Share, View Author.

### Ingredient Row (Editor)
- **Tap:** Opens inline keyboard with quantity/unit/name fields.
- **Swipe left:** Reveals red delete button.
- **Drag handle (right side):** Reorder ingredients via long-press drag.

### Step Row (Editor)
- **Auto-numbers** — renumbers on delete/reorder.
- **Optional inline photo:** Small camera icon in bottom-right of each step row.
- **Swipe left to delete**, drag to reorder (same as ingredients).

### Follow Button
- **States:** Follow (outlined) → Following (filled, accent color) → Unfollow (on press-hold or re-tap shows confirmation).
- Optimistic update; reverts on error.

### Rating Sheet
- Full-width bottom sheet.
- Large star row (48px tap targets for accessibility).
- Optional comment field below stars.
- [Submit] button activates when ≥ 1 star selected.
- One rating per user per recipe; reopening shows current rating pre-filled for editing.

### Sync Status Indicator
- Lives in the navigation header area.
- Invisible when online and synced.
- Small cloud icon with animation when syncing.
- Small cloud-with-line icon when offline (non-alarming).