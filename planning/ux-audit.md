# NeoVLab UX Audit
**Date:** 2026-03-26
**Auditor:** Kodo-hime, Code Wizard (via Claude Code)
**Scope:** Frontend (`src/frontend/`), Electron/WPF desktop layer (`src/desktop/`), Cast/Persona readiness

---

## Executive Summary

NeoVLab has a genuinely impressive design system for a media lab tool — the GITS-inspired aesthetic is
coherent, the CSS token architecture is thoughtful, and there are clear signs of iterative care (the
design debt catalogue in memory alone shows active stewardship). The primitives library is solid. The
keyboard navigation on CatalogPage is exceptional and rare in internal tooling.

That said, there are recurring rough edges that collectively erode the experience: inline tab bars
with divergent styling appear in three places while a perfectly good `Tabs` primitive goes unused;
accessibility coverage is shallow outside a handful of files; the MediaCard hover state competes with
selection state awkwardly; and the `TranscriptViewer` is almost useful but falls one step short of
being a real editorial tool. The Cast/Persona surface is completely absent and will need deliberate
design groundwork before the feature can be built.

**Scoring summary (subjective, 1-5):**
- Design system coherence: 4/5
- Component reuse: 3/5
- Accessibility: 2/5
- Desktop/web parity: 4/5
- Interaction polish: 3/5
- Cast/Persona readiness: 1/5

---

## 1. React Frontend

### 1.1 Styling Architecture

**What's there:** Pure CSS custom properties (`aero-theme.css`) + Tailwind for layout utilities. No
MUI, no Radix, no Headless UI. Everything is bespoke.

**Verdict:** This is actually a good call for an aesthetic-first application. The design token system
(`--accent-primary`, `--glass-bg`, `--density-*`, `--radius-*`) is comprehensive and the five named
themes plus glass-surface variants give it real flexibility. The `color-scheme: dark; accent-color:
var(--accent-primary)` declaration on `:root` is a nice touch — native form controls inherit the
brand colour without JS.

**Issues:**
- `--success` and `--warning` tokens are intentionally absent. This is documented as a known gap
  (D-012). It is actively causing workarounds: `bg-emerald-500/15 text-emerald-300` appears verbatim
  in at least three files (`JobsPage.tsx`, `MediaDetailPage.tsx`, `SmokeOutputsPage.tsx`). Each site
  has drifted slightly from the others. Add the tokens or live with the maintenance pain — just pick
  one.
- `GRADIENT_PRESETS` is defined in `SettingsAppearanceSection.tsx` and presumably duplicated in
  `BackgroundManager.tsx` (known D-003). This is a sync hazard that gets worse every time a preset
  is added.
- No font-family declaration (D-005). Segoe UI on Windows 11 is an acceptable accident but it is
  still an accident. The moment a Linux dev opens this in a browser, they get a random sans-serif.
  Declare something or acknowledge the constraint explicitly.

### 1.2 Component Architecture

**Primitives layer (`src/frontend/src/components/primitives/`):**
This is the best part of the frontend. `Button`, `Modal`, `Tabs`, `Badge`, `Skeleton`, `EmptyState`,
`CustomSelect`, `ContextMenu`, `Tooltip`, `Toast` — a complete and well-typed vocabulary. The
`Button` component's variant system is clean and density-aware. `Modal` has a real focus trap and
Escape handling. `Tabs` has `role="tablist"` and `aria-selected`. Genuinely good work here.

**The problem: the primitives are underused.**

The `Tabs` primitive exists but none of the page-level tab bars use it:

- `MediaDetailPage.tsx` (lines 153-167): hand-rolled `border-b-2 border-[var(--accent-primary)]`
  inline tab bar, no ARIA roles, no keyboard nav between tabs.
- `JobsPage.tsx` (lines 788-811): hand-rolled inline tab bar with `style` overrides for active state,
  same problems.
- `AdminTabBar.tsx`: a third divergent pattern using pill buttons with glow shadows.

Three tab bars, three different visual treatments. The `Tabs` primitive uses pill-button style; the
page-level ones use underline style. That's not inherently wrong — underline is a more standard tab
metaphor — but it means the component with ARIA roles is not the one being used.

**Recommendation:** Either migrate `MediaDetailPage` and `JobsPage` to `<Tabs>` (accepting the pill
style) or add an `underline` variant to `Tabs` and migrate to that. The current state forces every
future tab bar to be invented from scratch.

**Other architecture observations:**
- `StatusBadge` and `ProcessingTypeBadge` are defined locally in `JobsPage.tsx` but a nearly
  identical `StatusBadge` exists as inline JSX inside `MediaDetailPage.tsx` (with slightly different
  colour mappings — known D-004). Neither uses the `Badge` primitive. The `Badge` primitive
  exists in `primitives/Badge.tsx` and is used correctly in `StatusIndicators.tsx`. Pick one
  pattern and delete the others.
- `JobsPage.tsx` at 830 lines is doing too much. The kanban board logic, the detail panel, the
  routing log tab, the `JobTypeCheckboxDropdown`, and the `RoutingLogTab` sub-components are all
  co-located. The sub-components are defined inline (not imported) which makes them untestable in
  isolation and bloats the module.
- `MediaCard.tsx` hardcodes `rgba(0,210,210,0.08)` for selected-background and
  `rgba(0,210,210,0.12)` for hovered, rather than using `var(--accent-glow)`. This will not
  theme-switch correctly on non-teal themes. Same card also has mixed inline `style={{}}` and
  Tailwind classes with no clear rule for which is used when — the hover state is managed via
  React `useState` instead of CSS `:hover`, which adds unnecessary JS overhead and breaks
  pointer-speed heuristics.

### 1.3 Accessibility

**Honest assessment: shallow.**

Files with meaningful ARIA coverage: 7 (`TopBar`, `PlayerControls`, `AdminTabBar`, `CustomSelect`,
`Toast`, `Modal`, `Breadcrumb`).

The `Modal` is the standout — proper focus trap, Escape handler, `role="dialog"`, `aria-modal`,
focus restoration on close. This is production-quality.

`Tabs` primitive has `role="tablist"` and `aria-selected` on each tab button. Good.

`StatusIndicators` has `aria-live="polite"` on the job count container. Good.

**Everything else is largely undiscovered territory:**

- `MediaCard` has no `role`, no `aria-label`, no keyboard handling of its own. It receives focus via
  `CatalogPage`'s keyboard handler but the element itself has no visual focus indicator styled and
  no `tabIndex`. A screen reader visiting the catalog sees a wall of unlabelled `div` elements.
- `CatalogPage`'s keyboard handler registers `ArrowRight`/`ArrowLeft`/`ArrowUp`/`ArrowDown`/`Enter`
  etc. via a `KeyboardContext` but the arrow keys for grid navigation go up and down 1 item
  regardless of grid column count. In a dense grid with 6 columns, `ArrowDown` still moves to the
  next item, not the item directly below.
- The `JobsPage` kanban board cards have `role="button"` and `tabIndex={0}` with Enter/Space
  handling — this is correct and better than `MediaCard`. But the lane headers do not announce
  collapsed state via `aria-expanded`.
- `TranscriptViewer` uses `text-blue-400` on the "Show timed segments" toggle button (hardcoded,
  off-theme). This button has no ARIA attributes at all (`type="button"` is present — one point).
- `IngestPage`'s `DropZone` almost certainly lacks `role="region"` and an announcement for
  drop events. Not audited in detail but high probability.
- No skip-navigation link. Entirely missing. Tab order starts at the top bar logo every time.
- No `prefers-reduced-motion` media query coverage in JavaScript-driven animations (the CSS
  `@keyframes bg-drift` correctly uses it, which is commendable, but JS-driven transitions on
  `VideoPlayer`, card transforms, etc. ignore it entirely).

**Priority a11y gaps, ranked:**
1. `MediaCard` needs `role="gridcell"` (or similar), `aria-label` with file name, and
   `tabIndex={0}` with a visible focus ring.
2. Grid keyboard navigation should be column-aware (`ArrowDown` = +N where N = column count).
3. All three inline tab bars need `role="tablist"` / `role="tab"` / `aria-selected`.
4. Lane collapse buttons in `JobsPage` need `aria-expanded`.
5. `TranscriptViewer` toggle needs `aria-expanded` and on-theme colour.
6. Skip-navigation landmark link.

### 1.4 Design System vs Ad-Hoc

The design system is real and mostly enforced. The places where it breaks down:

| Location | Issue |
|---|---|
| `MediaCard.tsx` lines 119, 122 | Hardcoded `rgba(0,210,210,…)` teal bypasses theme vars |
| `MediaDetailPage.tsx` lines 190-199 | Inline `StatusBadge`-alike uses `bg-blue-500/15 text-blue-300` (hardcoded blue) |
| `TranscriptViewer.tsx` line 77 | `text-blue-400` on toggle button |
| `SettingsPage.tsx` line 108 | `text-gray-500` on loading state (Tailwind grey, not `--text-muted`) |
| `SettingsPage.tsx` line 109 | `text-red-600` on error state (Tailwind red, not `--danger`) |
| Various | `p-8` and `max-w-6xl mx-auto` repeated across page-level containers — could be a `.page-container` utility class |

The error and loading state pattern in particular is completely inconsistent across pages:
- `JobsPage`: `p-8 text-[var(--text-muted)]` / `p-8 text-[var(--danger)]` (correct)
- `MediaDetailPage`: `p-8 opacity-60` / `p-8 text-red-400` (mixed)
- `SettingsPage`: `p-8 text-gray-500` / `p-8 text-red-600` (off-system)

These three pages have three different flavours of the same two states. An `<ErrorState>` and a
`<LoadingState>` primitive would solve this entirely.

### 1.5 Layout and Responsive Behaviour

The app appears to target primarily desktop (1440px+), which is honest given the Electron shell.
Responsive breakpoints exist (`hidden sm:inline`, `hidden md:inline`, `xl:flex`, etc.) and TopBar
collapses the logo on small screens.

**Issues:**
- `JobsPage` kanban board uses `hidden xl:block` to gate the swimlane view. Below xl, users see a
  flat chronological list with no lane grouping. This is acceptable as a fallback, but the threshold
  (1280px) is fairly aggressive — a 1366px laptop will see the full board but barely.
- The detail panel in `JobsPage` uses `xl:sticky` with a `calc(100vh - ...)` `maxHeight`. This is
  correct but the `--panel-pin-top` variable needs to be set correctly or the panel overflows. Not
  confirmed to be broken — flagging as a fragile calculation.
- `CatalogPage` passes `initialPageSize: 500` to `useMediaAssets`. Loading 500 items on mount with
  no pagination UI is fine for a local library, but there's no skeleton/loading state between initial
  mount and data arrival — just an empty grid. `Skeleton` primitive exists and is unused here.
- The `VideoPlayer` component leaves the "Loading video..." placeholder as a plain `div` with no
  spinner or skeleton. It visually disappears the moment `src` is available — but between route
  mount and `streamUrl` being set (an async fetch), there's a flash of nothing.

---

## 2. Electron / WPF Desktop Layer

### 2.1 Shell Structure

The desktop layer is `src/desktop/NeoVLab.Desktop/` — a WPF app embedding WebView2. Architecture:
- `MainWindow.xaml` — custom chrome (no OS DWM), rounded-corner border, custom title bar
- `MainWindow.xaml.cs` — bridge message handler, URL resolution, WPF/React theme sync

**This is clean and purposeful.** The decision to do custom chrome with `WindowStyle="None"` and
`AllowsTransparency="True"` gives full control over the border glow and rounded corners, which
matches the aesthetic intent perfectly. The `DropShadowEffect` with themed accent colour is a nice
detail most desktop apps skip.

### 2.2 Window Management

- Minimize/Maximize/Close all work correctly. Double-click on title bar toggles maximize.
- Maximize correctly collapses the outer `Margin="10"` and the `CornerRadius="8"` to fill the screen.
  Maximize icon switches between Maximize and Restore glyphs — this is the correct Windows behaviour.
- The `ResizeCorner` is a hand-drawn two-line canvas element that performs a `WM_SYSCOMMAND`
  `SC_SIZE` + `HTBOTTOMRIGHT` call. This works but it means **resize only works from the bottom-right
  corner**. Dragging any other edge does nothing.

**Missing:** Full edge resize. The `ResizMode="CanResize"` flag is set but with `WindowStyle="None"`,
OS edge-resize hit-testing is disabled. Only the WPF-managed corner handles the resize. Users
expecting to grab any window edge will be confused.

**Fix pattern:** Install a `WndProc` hook that handles `WM_NCHITTEST` to return the appropriate
`HT*` constant for all 8 resize zones (not just HTBOTTOMRIGHT). This is ~60 lines of C# and is the
standard pattern for frameless WPF windows.

### 2.3 Bridge Messages

Current bridge surface:
- `pick-directory` — opens FolderBrowserDialog, returns path
- `list-drives` — enumerates logical drives, returns JSON array
- `open-file` — calls `Process.Start` with the local file path
- `theme-changed` — syncs accent/bg colour to WPF dynamic resources
- `pick-wallpaper` — opens OpenFileDialog for image files, returns base64 data URL

This is a well-structured message-passing bridge. The `theme-changed` → `DynamicResource` pattern
(updating `TitlebarAccentBrush` / `TitlebarBgBrush` at runtime) is particularly elegant.

**UX gaps:**
- No `window-state-changed` message back to React. The frontend has no way to know if the window is
  maximized, minimized, or restored. This matters for layout decisions (e.g., the TopBar could hide
  the logo capsule when maximized since the title bar already shows the brand name).
- `pick-wallpaper` returns a base64 data URL of potentially large images, stored in `localStorage`.
  localStorage has a ~5MB limit. A 4K JPEG wallpaper will exceed this. This will silently fail or
  throw a `QuotaExceededError` that the frontend may not handle gracefully. The bridge should instead
  return the file path and serve the image via a `WebView2.CoreWebView2.AddWebResourceRequestedFilter`
  virtual URL scheme, or at minimum cap the image dimensions before base64-encoding.

### 2.4 Native Menus and System Tray

**Neither exists.** There is no system tray icon, no native application menu, no taskbar context
menu. For a desktop media management tool this is a notable gap — users cannot right-click the
taskbar to get to key actions, and there is no way to access the app from the system tray when
minimized. These are not blocking for an internal tool, but they are expected affordances for a
desktop-first application.

### 2.5 Web-to-Desktop UX Parity

**Good:** Theme sync is real-time and bidirectional in effect (React fires `theme-changed`, WPF
updates). The `Browse` button for custom wallpaper is correctly hidden in non-desktop (browser) mode.
The `Open In Default Player` button is also gated on bridge availability.

**Gaps:**
- No desktop-specific keyboard shortcut registration at the WPF level. Global hotkeys (e.g.,
  Win+Alt+N to bring the window to front, media key support) are absent.
- The title bar says "Neo V nExT" with the unusual capitalisation. This is a branding choice but it
  looks like a typo to anyone unfamiliar with the project. At minimum the tooltip on the taskbar
  button (the `Title` property) should match the intended display name.
- `ResolveFrontendUrlAsync` hardcodes `http://localhost:6100` with only one candidate in the array
  (the fallback is the same as the only candidate). The multi-candidate probing loop is dead code.
  Either add real alternates or simplify.

---

## 3. Biggest UX Gaps

Prioritised by impact:

### GAP-001: No loading/error primitive (CRITICAL for polish)
Three pages handle loading and error states differently. None use the same class names or design
tokens. Users see visual inconsistency on every page load. An `<ErrorState>` and `<LoadingState>`
component would take 30 minutes to build and fix all three pages simultaneously. The `Skeleton`
primitive already exists and just needs to be wired in.

### GAP-002: Tab bar fragmentation (HIGH)
`Tabs` primitive exists with ARIA support. Three pages implement their own. The `MediaDetailPage`
tab bar has no ARIA roles whatsoever. Building a new feature with tabs means inventing a fourth
variant. Migrate `MediaDetailPage` and `JobsPage` inline tabs to `<Tabs>` or add an `underline`
variant to the primitive.

### GAP-003: MediaCard accessibility and selection UX (HIGH)
`MediaCard` is the most-used interactive element in the application and it has no accessibility
support beyond `onClick`. The hover state is managed in JS `useState` instead of CSS `:hover`
(see D-xxx pattern — this will also hurt performance on large grids due to re-renders on every
mouse-enter/leave). The selected+hovered state combination creates a visually confusing moment where
both styles fight.

### GAP-004: Transcript Viewer is read-only (MEDIUM)
The transcript shows text and optionally timed segments. You cannot click a segment timestamp to
seek the video. For a media lab, this is a missed feature that users will feel the absence of
immediately. The `VideoPlayer` already exposes a `videoRef` — the missing piece is wiring
`TranscriptViewer` to accept an optional `onSeek(seconds: number)` callback and making the segment
timestamps clickable.

### GAP-005: Jobs page has no auto-refresh (MEDIUM)
The jobs board loads once and never updates unless the user navigates away and back. Running jobs
do not show progress, new jobs submitted elsewhere do not appear. SignalR events are already consumed
in `StatusIndicators` (and `useComfyProgress` for ComfyUI detail). A `setInterval` or SSE-driven
refresh on the job list would cost very little and dramatically improve the live-monitoring use case.
(Partially mitigated by the ComfyUI progress hook for that specific job type — but the rest of the
board is static.)

### GAP-006: No empty state in Catalog (LOW-MEDIUM)
When the catalog is empty (new install, filters applied that return nothing, server unreachable), the
user sees a blank glass panel with no text. The `EmptyState` primitive exists and is unused in
`CatalogPage`. The error state (`media.error`) renders as a raw `text-red-400` div — again, no
`EmptyState` usage.

### GAP-007: Window edge resize (DESKTOP ONLY, MEDIUM)
Only the bottom-right corner resizes the window. All other edges are non-functional despite
`ResizeMode="CanResize"`. This breaks Windows user expectations.

### GAP-008: Wallpaper base64 localStorage overflow (DESKTOP ONLY, MEDIUM)
Custom wallpaper images are stored as base64 data URLs in `localStorage`. Large images will silently
fail or corrupt the preference store. Needs a path-based or virtual-URL approach.

### GAP-009: MediaDetailPage fetches all jobs, filters client-side (LOW)
`loadJobs` calls `getJobs()` (returns all jobs system-wide) and then `filter`s by `mediaId` in
memory. As the job history grows, this will load hundreds or thousands of jobs to display 3-5 for
one media item. There should be a `GET /api/jobs?mediaId=...` endpoint, or at minimum the call
should be `getJobsByMedia(mediaId)`.

---

## 4. Cast / Persona UI Readiness

**Current state: zero groundwork.** The cast/persona feature (face + voice identification) does not
exist in the UI layer in any form. Here is what is present that will be useful, and what needs to
be built:

### What exists and is relevant:

- `ArtifactGallery.tsx` already renders `faces_json` artifacts with a "faces" icon. The data exists;
  it is just shown as a download link, not as an identity surface.
- `TranscriptViewer.tsx` already renders `speakers_json`-type artifacts (via gallery). Speaker
  diarization output (speaker labels, timestamps) is already in the data model.
- `MediaDetailPage.tsx` has a tabs structure that could absorb a "Cast" tab without structural
  changes.
- The job pipeline already includes `face_extraction` and `diarization` job types with UI submission
  in `CatalogPage`'s job modal.
- The `Badge` and `Tooltip` primitives are ready to label identified faces/speakers.

### What needs to be designed and built:

**Data surface:**
- A `Person` entity (id, name, face thumbnails, voice sample reference) needs a backend model and
  API. There is no person/identity concept in the current schema.
- The `faces_json` artifact currently contains detection bounding boxes. It does not contain identity
  assignments. The identity assignment step (clustering + labelling) is missing.

**UI components to build:**
1. **PersonCard** — thumbnail of a face crop or avatar placeholder, name (editable), role/tag.
   Should live in `src/frontend/src/components/cast/`.
2. **CastRoster** — grid or list of `PersonCard`s for a given media asset. Tabbed into
   `MediaDetailPage` as a "Cast" tab.
3. **PersonMergeModal** — for when two detected identities are the same person (cluster merging UX).
4. **SpeakerTimeline** — the `TranscriptViewer` timed segments recomposed as a horizontal
   speaker-labelled timeline, with segments colour-coded by speaker identity and clickable to seek.
   This would replace (or extend) the current flat segment list.
5. **GlobalCastPage** — a catalog-level view of all identified persons, which media they appear in,
   and how many occurrences. Route: `/cast` or `/admin/cast`.
6. **FaceReviewQueue** — a moderation surface for unconfirmed face clusters that need a human to
   confirm or merge. This is the editorial step between raw detection and labelled identity.

**Design system requirements for cast:**
- A persona colour swatch system (assign a colour to each person for timeline differentiation).
  Could use CSS custom properties scoped per-person ID.
- Face thumbnail aspect ratio convention. Detection crops are not square — decide on a fixed
  aspect ratio (1:1 with face centred) and enforce it at the processing layer.
- An avatar fallback for persons with no confirmed face crop (initials + coloured background,
  like most chat apps).

**The transcript/cast integration is the high-value short path.** The diarization output
(`speakers_json`) already has speaker-labelled segments. Building `SpeakerTimeline` first — wiring
it to the existing `TranscriptViewer` and giving segments seek-on-click — would deliver immediate
value before the full identity pipeline exists. Speakers would be anonymous ("Speaker 1", "Speaker 2")
at first, but the UX shell would be in place to accept named identities when that backend work lands.

---

## 5. Known Debt Items Not Yet Resolved

From the project memory and audit:

| ID | Status | Description |
|---|---|---|
| D-003 | Open | `GRADIENT_PRESETS` duplicated in `SettingsAppearanceSection` and `BackgroundManager` |
| D-004 | Open | `StatusBadge` duplicated in `JobsPage` and `MediaDetailPage` with divergent colour mappings |
| D-005 | Open | No explicit font-family declaration |
| D-009 | Open | `toggle-enabled` and `input:focus` in `index.css` use hardcoded blue instead of `--accent-glow` |
| D-010 | Open | Same issue, companion to D-009 |
| D-012 | Accepted | GPU progress blocks use hardcoded emerald (no `--success` token) |
| D-014 | Open | No shared Tabs component across 3 pages — this audit confirms it is 3 divergent patterns |

New items identified in this audit:

| ID | Description |
|---|---|
| D-015 | `MediaCard` uses hardcoded `rgba(0,210,210,…)` instead of `--accent-glow` / `--accent-primary` |
| D-016 | No `<ErrorState>` / `<LoadingState>` primitives — 3 pages with divergent patterns |
| D-017 | `TranscriptViewer` uses hardcoded `text-blue-400` |
| D-018 | `SettingsPage` loading/error states use Tailwind grey/red instead of design tokens |
| D-019 | `JobsPage` inline sub-components (`StatusBadge`, `ProcessingTypeBadge`, `RoutingLogTab`, `JobTypeCheckboxDropdown`) should be extracted to separate files |
| D-020 | `MediaDetailPage` fetches all jobs system-wide and filters client-side |
| D-021 | Wallpaper base64 localStorage overflow risk |
| D-022 | WPF window — only bottom-right corner resizes (no full-edge `WM_NCHITTEST` hook) |

---

## Recommended Priority Order

1. **D-004 / D-016** — Consolidate `StatusBadge` and build `ErrorState`/`LoadingState` primitives.
   High ROI, low effort, fixes 4+ files simultaneously.
2. **D-014** — Migrate `MediaDetailPage` and `JobsPage` inline tabs to `<Tabs>` primitive (add
   `underline` variant). Fixes ARIA and visual consistency simultaneously.
3. **GAP-003 / D-015** — Fix `MediaCard` hover state to CSS-only and replace hardcoded colours with
   theme tokens. Performance + theming fix.
4. **GAP-004** — Wire `TranscriptViewer` seek callback to `VideoPlayer`. High editorial value.
5. **GAP-005** — Add SSE/SignalR-driven auto-refresh to `JobsPage`. Operational quality of life.
6. **D-022** — Add full-edge resize via `WM_NCHITTEST` hook in WPF. Desktop polish.
7. **Cast/Persona** — Start with `SpeakerTimeline` component (lowest dependency on missing backend),
   then `PersonCard`, then `CastRoster` tab in `MediaDetailPage`.
