# UI Design — BioSyft Unsupervised Behavior Dashboard

This document defines the UX for the seven-panel dashboard described in `ui_reqs_rough.md`. It covers layout, ASCII wireframes, interaction flows, state transitions, keyboard shortcuts, color/legend conventions, and accessibility. All numbers and rendering choices respect `tech_stack.md` (deck.gl scatter, Observable Plot stats, uPlot timeline, HTML5 video) and the project constraints: 100 FPS source video, 10–20 min duration, ~500 videos/study, 3-week timeline, two personas (PhD student, study director).

## Global Layout

The dashboard is a single-page app with three top-level routes. All routes share a thin top bar (study name, breadcrumb, share button, export button, help). The PhD student and study director share the same UI; the study director uses Compare and Export more heavily.

- `/study/:id/explore` — default landing. Embedding scatter + cluster panel + frame grid (panels 1, 2, 3).
- `/study/:id/video/:videoId?frame=N&cluster=K` — single-video deep dive (panel 4) opened from a frame tile or scatter point.
- `/study/:id/compare` — condition comparison (panel 5).

Panels 6 (export) and 7 (persistence/share link) are global affordances available from the top bar on every route, not separate pages.

```
+----------------------------------------------------------------------------+
| BioSyft  Study: "Compound-A vs Vehicle"          [Share] [Export] [Help]  |
| Explore | Video | Compare                                       user@lab  |
+----------------------------------------------------------------------------+
|                                                                            |
|                          (active route content)                            |
|                                                                            |
+----------------------------------------------------------------------------+
```

## Panel 1 — Embedding Scatter (centerpiece, in /explore)

Full-width WebGL scatter via deck.gl `ScatterplotLayer`. Renders up to 500k frame points at 60 fps; for >500k we use density-based downsampling per zoom level (precomputed server-side, fetched as Arrow).

```
/explore                                                              [Share]
+--------------------------------------------+-------------------------------+
|                                            | Clusters (12)         [+New] |
|       . :::.. ..      . . :....            | --------------------------- |
|     ::cluster-3::.   ..:cluster-7:         | [#] rearing        18,402 ▌ |
|    .:::::.::: . .   .:.::::::.             |     drug ████  veh ██       |
|       .  .   .       .   .                 | [#] grooming        9,221 ▌ |
|         .  cluster-5 (selected)            | [#] cluster_4       4,888   |
|        ::::::::::: ←lasso outline          |     ✎ rename...             |
|        :::::::::::                         | [#] locomotion     22,103   |
|                                            |     drug ██    veh █████    |
|  hover tooltip ┌─────────────┐             | [#] cluster_6       2,144   |
|                | thumb 64x64 |             | ...                          |
|                | vid12 t=04:32|             |                              |
|                | cluster: 5  |             |                              |
|                └─────────────┘             | Filters                      |
|                                            |  Condition: [All ▼]          |
|  [zoom +] [zoom -] [reset] [lasso] [box]   |  Video: [All ▼]              |
+--------------------------------------------+-------------------------------+
| Frame grid — Cluster 5 representatives (k-medoids, 12 of 12)   [open all] |
| [thumb][thumb][thumb][thumb][thumb][thumb][thumb][thumb][thumb][thumb][▶] |
|  v01    v01    v04    v07    v07    v12    v12    v18    v22    v31  ... |
|  02:11  04:32  00:48  01:55  09:02  04:32  11:20  03:47  07:14  02:00     |
+----------------------------------------------------------------------------+
```

Interactions:
- Hover (debounced 80 ms) → fetch thumbnail by `(videoId, frameNumber)` from CDN sprite sheet, show 64×64 tooltip with timestamp and cluster ID. Tooltip dismisses on `mouseleave` or `Esc`.
- Click point → highlight the cluster of that point in the cluster panel and replace the frame grid with that cluster's medoids. Modifier-click (Shift) selects multiple clusters; the cluster panel shows multi-select totals.
- Lasso/box select → polygonal selection produces an ad-hoc selection chip ("Selection: 1,243 frames across 4 clusters") above the cluster panel. Selection persists in URL state.
- Double-click point → opens `/video/:videoId?frame=N`.
- Pan = drag; zoom = wheel or pinch; `R` resets view.
- Color = cluster ID via the categorical palette below.

States:
- Loading: skeleton with shimmering circle and progress text "Loading 248k frames… 41%". Progressive Arrow chunks render as they arrive (no blocking).
- Empty (no clusters yet): centered message "Analysis not finished. We'll email you when it's ready." with link back to study list.
- Error: red banner at top "Could not load embedding (HTTP 503). [Retry]". Last-known-good cached embedding still rendered behind a dimming overlay if available.
- Partial-data: if biometrics or thumbnails fail but coordinates loaded, scatter still works; tooltip shows "(thumbnail unavailable)" placeholder.

## Panel 2 — Cluster Panel (in /explore, right rail)

Right-rail vertical list, fixed 320 px wide, virtualized with TanStack Virtual.

```
Clusters (12)                         [+ New cluster] [Sort: count ▼]
-------------------------------------------------------------------
[●] rearing                  18,402 frames                       ✎
    drug ████████████░░  veh ████░░░░░░░░░       (45% / 14%)
-------------------------------------------------------------------
[●] grooming                  9,221 frames                       ✎
    drug ███░░░░░░░░░░░  veh ████████░░░░░       (8% / 21%)
-------------------------------------------------------------------
[●] cluster_4                 4,888 frames                       ✎
    [click pencil to rename]
-------------------------------------------------------------------
```

Each row:
- Color swatch (matches scatter), cluster name (or default `cluster_<id>`), frame count, condition breakdown bar, percent split.
- Pencil icon → inline rename. Enter saves; Esc cancels. Names sync via debounced PATCH (`/studies/:id/clusters/:clusterId`) with optimistic update; on failure, the row reverts and a toast appears.
- Eye toggle on hover hides/shows that cluster in the scatter (does not delete).
- Right-click → context menu: Rename, Hide, Isolate (hides others), Open in Compare, Copy cluster URL.

States:
- Default name shown italic to signal "unnamed". Once renamed, plain weight.
- Persistence indicator: a tiny green dot next to the name when the latest edit is confirmed saved; gray dot while in flight.

## Panel 3 — Frame Grid (in /explore, bottom strip)

Below the scatter, a 1-row strip of 10 representative thumbnails per selected cluster, scrollable horizontally; expands to a full grid in a modal via "open all".

```
Frame grid — Cluster "rearing" — 12 medoids                [Open all ▦]
+--------+--------+--------+--------+--------+--------+--------+--------+
| [img]  | [img]  | [img]  | [img]  | [img]  | [img]  | [img]  | [img]  |
| v01    | v01    | v04    | v07    | v07    | v12    | v12    | v18    |
| 02:11  | 04:32  | 00:48  | 01:55  | 09:02  | 04:32  | 11:20  | 03:47  |
+--------+--------+--------+--------+--------+--------+--------+--------+
```

Tile = 128×128 thumbnail, video ID + timestamp caption. Click → opens deep dive at that frame. Hover plays a 1-sec mini-loop (5 frames sampled around the medoid) without audio.

"Open all" modal: 6×N grid covering all medoids (typically 8–24 per cluster). Filterable by source video and condition.

States:
- No selection: caption reads "Select a cluster to see representatives." Empty grid with dashed border.
- Loading thumbnails: gray rectangles with a slight pulse.
- Sprite-sheet miss: single placeholder icon and a "retry" affordance per tile.

## Panel 4 — Single-Video Deep Dive (/video/:videoId)

Two-column layout: video + scatter mini-map on the left, biometric/cluster timelines on the right and below.

```
/video/v12?frame=27200&cluster=5                                 [Share]
+----------------------------------------+-------------------------------+
| Video v12  drug arm                    | Mini-map (embedding)          |
|  +----------------------------------+  | +---------------------------+ |
|  |                                  |  | |     . :::.. .             | |
|  |        [video frame]             |  | |    ::cur frame::          | |
|  |    keypoints overlay (toggle)    |  | |    .  ★ .                 | |
|  |                                  |  | +---------------------------+ |
|  +----------------------------------+  | Active cluster: rearing       |
|  ◀◀ ◀ ▶ ▶▶  04:32 / 18:00  [1x ▼]    | Frame: 27200 (t=04:32.00)     |
+----------------------------------------+-------------------------------+
| Cluster track                                                          |
|  ▓▓▓▓░░░░▓▓▓▓▓▓▓▓░░░░░░▒▒▒▒▒▒▒▒▒▓▓▓░░░░▓▓▓▓▓▓▓▓░░░░▒▒▒▒▒▒░░░░         |
|  rearing  groom rearing       walk    rearing       walk              |
| Speed (px/s)                                                           |
|   /\__/\__/^^\___/\___/\/\___/\__/^^^\__/\___/\/\___                  |
| Posture (deg)                                                          |
|   ~~~~~~~~~~^^^^^~~~~~~~~~^^^^^^^^^^^~~~~~~~~^^^^^^^^^                |
+------------------------------------------------------------------------+
```

Interactions:
- Standard video controls + speed picker (0.25x, 0.5x, 1x, 2x, 4x). Spacebar play/pause.
- Scrubbing the video updates a star marker on the mini-map embedding so the user always sees "where on the map am I right now". Marker debounced at 50 ms during scrub; exact on pause.
- Cluster track is colored bands matching cluster palette; hovering a band shows the cluster name and duration. Clicking a band scrubs the video to its start.
- Speed and posture lines are uPlot Canvas, synced cursor with cluster track and video.
- Keypoint overlay: toggle button "Show keypoints"; when on, an SVG layer draws nose/neck/paws atop the video frame at the current time. (Source = keypoints API, decimated to 10 FPS for render economy.)
- Back arrow returns to /explore preserving prior selection (URL state).

States:
- Video buffering: spinner over poster. Timelines remain interactive at last known cursor.
- Missing biometrics for video: timelines show "No speed/posture for this video — cluster track only" and remain functional.
- Frame out of range (invalid `?frame=`): toast "Frame N not in v12 (max 108,000)" and snap to frame 0.

## Panel 5 — Condition Comparison (/compare)

Designed for the study director making a go/no-go figure. Two display modes selectable by a segmented control: Bars and Small-multiples.

```
/compare      Conditions: [vehicle] vs [compound-A] vs [compound-B]   [Edit]
                                  Display: ( Bars ) Small-multiples   [Export]
+----------------------------------------------------------------------------+
| Cluster usage (% of frames per condition)                                  |
|                                                                            |
|  rearing      vehicle ████░░░░  6.2%   compoundA ████████ 12.1%  ▲ d=0.81*|
|               compoundB ███░░░░  4.9%                                      |
|  grooming     vehicle ████████ 14.0%   compoundA ███░░░░  4.4%  ▼ d=-0.92*|
|               compoundB ███████ 12.7%                                      |
|  locomotion   vehicle ██████░░  9.8%   compoundA █████░░  8.9%   ns       |
|               compoundB ██████░░ 10.1%                                     |
|  ...                                                                       |
|                                                                            |
|  * Cohen's d, BH-adjusted p<0.05.   Effect-size legend:  ▲ up  ▼ down ns  |
+----------------------------------------------------------------------------+
```

Small-multiples mode: per cluster, a tiny Observable Plot bar of condition usage. 4×3 grid of mini-charts, click any to open a larger view with per-video dots.

Interactions:
- Edit conditions → modal lets the user assign each video to a condition (vehicle / compound-A / etc). The mapping is study metadata; cached locally and persisted server-side.
- Click a cluster row → cross-links to /explore with that cluster pre-selected.
- Export button on the panel exports just this figure (PNG @ 2x, SVG, plus a CSV of the underlying numbers).

States:
- < 2 conditions defined: panel shows a setup prompt "Define at least two conditions to compare. [Set up conditions]".
- Insufficient samples (< 3 videos per condition): table renders but effect-size column shows "n too small" with a tooltip explaining why.
- Mid-export: button shows spinner; cancel returns to idle.

## Panel 6 — Export (top-bar global + per-figure)

```
Export                                                                  [×]
( ) CSV — cluster names, per-condition counts, percentages, effect sizes
( ) PNG — comparison figure, 2x DPI, slide-sized 1920x1080
( ) SVG — comparison figure, vector
( ) Bundle — CSV + PNG + SVG + filter state JSON

Include: [x] applied filters  [x] selection chips  [ ] raw coordinates

Filename: biosyft_study123_2026-05-07_compareA-B.zip
                                                  [Cancel]  [Download]
```

Behavior:
- CSV columns: `cluster_id, cluster_name, condition, video_id, frame_count, percent_of_video, percent_of_condition, effect_size, p_adj`.
- Raw coordinates intentionally OFF by default — the artifact is interpretive, not bulk data.
- Filter state JSON round-trips into a shareable URL (see Panel 7).
- All exports are pure client-side from in-memory Arrow tables; no server roundtrip required.

## Panel 7 — Persistent View State / Shareable Link

URL is the source of truth for view state. State is encoded in a compact base64-JSON `?view=` parameter plus discrete params (`?cluster=`, `?condition=`, `?selection=`).

```
[Share]                                                                 [×]
URL: https://app.biosyft.com/study/123/explore?view=eyJjbHVzdGVycyI6WzUsN10s
     ImNvbmRpdGlvbnMiOlsidmVoaWNsZSIsImNvbXBvdW5kQSJdLCJzZWxlY3Rpb24iOiJsYX
     NzbzpwMSJ9                                              [Copy] ✓ Copied

This link captures:
 • Selected clusters: rearing, locomotion
 • Renamed clusters and notes (study-scoped, applied to anyone with access)
 • Active filters (drug, vehicle)
 • Current selection chip
 • Camera position (zoom + pan)

Reset view to defaults                                            [Reset]
```

Persistence model:
- Cluster names/notes: server-stored per study (`PATCH /studies/:id/clusters/:cId`). Visible to anyone with study access.
- Camera + selection + filters: URL-only. No server write needed.
- Last-opened state per user: localStorage `biosyft.study.<id>.lastView` so a refresh restores the previous URL even if it wasn't bookmarked.

## Cross-Cutting Interaction Flows

1. Hover preview: scatter point → 80 ms debounce → CDN sprite sheet fetch → 64×64 thumbnail tooltip. Cancelled on mouseleave.
2. Lasso select: hold `L` or click lasso tool → freehand polygon → on release, points inside become a Selection. Selection chip appears, frame grid shows medoids of the selection (ad-hoc). Saving selection requires "Convert to cluster" (creates a virtual cluster, server-side flag).
3. Click-to-expand: tile click in frame grid → /video route at that frame. Back returns with selection intact (URL).
4. Video-scrub-syncs-scatter: every video timeupdate event → compute current frame → look up cluster → highlight star on mini-map and active cluster name. RAF-throttled.
5. Cluster rename: pencil → inline input → optimistic update → debounced 400 ms PATCH. Failure → revert + toast "Couldn't save 'rearing'. [Retry]".
6. Cross-route memory: navigation between Explore/Video/Compare preserves selected cluster and condition filter via URL.

## Keyboard Shortcuts

| Key            | Action                                                      |
|----------------|-------------------------------------------------------------|
| `?`            | Open shortcut cheat-sheet                                   |
| `1` / `2` / `3`| Switch to Explore / Video / Compare                         |
| `Space`        | Play/pause video (Video route)                              |
| `← / →`        | Step 1 frame back/forward (Video, paused)                   |
| `Shift+← / →`  | Step 1 second back/forward (Video)                          |
| `L`            | Lasso tool (Explore)                                        |
| `B`            | Box-select tool (Explore)                                   |
| `R`            | Reset scatter view                                          |
| `H`            | Hide/show selected cluster                                  |
| `F2`           | Rename selected cluster (inline)                            |
| `E`            | Open Export dialog                                          |
| `S`            | Open Share dialog                                           |
| `Esc`          | Cancel rename / close modal / dismiss tooltip               |
| `/`            | Focus cluster search                                        |

All shortcuts are listed in the `?` cheat-sheet modal and announced via `aria-keyshortcuts` on each control.

## Color & Legend Conventions

- Cluster palette: 12-class Tableau-derived categorical, extended with Glasbey-generated colors for clusters 13+. Maximum simultaneously visible distinct colors = 24; beyond that, less-frequent clusters collapse into a striped "Other" bucket toggleable from the legend.
- Each cluster has a stable color across the entire dashboard: scatter dots, cluster panel swatch, frame-grid border, cluster track in deep dive, comparison bars. Color is keyed by cluster ID so renames don't recolor.
- Conditions use a separate, perceptually distinct duotone (vehicle = neutral gray, primary compound = teal, additional compounds = ramp from teal → magenta). This avoids collision with cluster colors.
- Effect-size glyphs: ▲ increase, ▼ decrease, `ns` not significant. Color (red/blue) reinforces but never carries the only meaning.
- All charts ship a visible legend; exported PNG/SVG embed it.

## Accessibility Notes (PhD-student-first, but applies to all)

- All actions are reachable by keyboard. Focus rings are 2 px solid `#005FCC` and never suppressed.
- Color is never the sole channel: cluster swatches include a numeric ID badge; condition bars include text labels and patterns (solid/diagonal/dotted) when high-contrast mode is on.
- Tooltips are also surfaced as `aria-describedby` text and re-rendered into a polite live region when focus (not just hover) lands on a point.
- Scatter has an alternative "list mode" toggle: a virtualized table of (frame, video, cluster, x, y) for screen-reader users; lasso replaced by checkbox selection.
- Video player exposes native `<video>` controls plus our custom UI (custom UI proxies into the underlying element so VoiceOver/NVDA work).
- Color palette passes WCAG AA against both light and dark backgrounds; we ship light theme by default and a dark theme toggle. All text ≥ 14 px, line-height ≥ 1.4.
- Motion: hover loops on frame tiles respect `prefers-reduced-motion: reduce` and degrade to a static thumbnail.
- Errors are announced via `role="alert"`; non-blocking notifications via `role="status"`.
- Plain-language affordances: every icon has a visible label or a tooltip ≤ 250 ms; nothing relies on a discoverable right-click. The non-technical PhD student should be able to do everything (rename, filter, export) using only the visible top bar and right-rail without learning shortcuts.

## Loading / Empty / Error / Partial-Data Matrix

| Surface          | Loading                                  | Empty                                  | Error                                | Partial-data                                   |
|------------------|------------------------------------------|----------------------------------------|--------------------------------------|------------------------------------------------|
| Scatter          | Skeleton + progressive Arrow render      | "Analysis not finished" CTA            | Banner + retry, last-known-good dim  | Render coords; thumbnails fall back to icons   |
| Cluster panel    | Row skeletons                            | "No clusters returned" CTA to support  | Inline error per row + retry         | Show counts even if names PATCH fails          |
| Frame grid       | Pulsing tiles                            | "Select a cluster" prompt              | Per-tile retry icon                  | Mix of loaded + placeholder tiles              |
| Deep dive        | Buffering spinner                        | Invalid video → 404 page with back btn | Banner + retry on biometric fetch    | Cluster track only when biometrics missing     |
| Compare          | Skeleton bars                            | "Define conditions" CTA                | Banner + retry                       | "n too small" annotation per cluster           |
| Export           | Spinner on Download                      | n/a                                    | Toast "Export failed [Retry]"        | Disable PNG if comparison data partially loaded|
| Share            | n/a                                      | n/a                                    | Toast "Could not save name. [Retry]" | URL state always available even if server down |

## Open UX Decisions (resolved)

- Multi-cluster select: Shift-click in the cluster panel and lasso both produce multi-selection; the frame grid shows up to 4 medoids per cluster when more than one is selected.
- Renaming collisions: two users renaming the same cluster — last-write-wins with an "Updated by user@x at HH:MM" tooltip on hover.
- Mini-map vs full embedding in deep dive: mini-map only, to keep the deep dive focused. Users return to /explore for full interaction.
- Video keypoints: off by default; toggle persists per user in localStorage.
- Share link permissions: link inherits study ACL; we do not create unauthenticated public links.
