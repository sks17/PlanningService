# Scope

Defines what the BioSyft unsupervised behavior dashboard will and will not do in its 3-week v1. The contract the rest of the plan (`goals.md`, `tech_stack.md`, `ui_design.md`, `timeline.md`, `data_architecture.md`, `user_workflows.md`) builds against.

## Context Recap

BioSyft already runs the ML: frames are clustered and projected into 2D. The dashboard's job is interpretation, not computation. Two personas drive every decision:

- **PhD student** — non-technical, Excel-native, ~40 videos across two conditions. Goal: interpret clusters, find drug effects.
- **Study director** — technical (R/Python), GraphPad/PowerPoint native, ~500 videos across 4 compounds + vehicle + positive control. Goal: slide-ready figures for a go/no-go meeting.

Constraints: 3-week timeline, solo developer, REST API supplying coordinates, biometrics, keypoints, and videos.

## In Scope

The seven pillars below are the committed feature set. Each expands the original rough sketch.

### 1. Embedding view (centerpiece)

- 2D scatter of every frame, colored by cluster ID.
- Zoom, pan, box select, lasso select.
- Hover shows a frame thumbnail plus video ID and timestamp.
- WebGL rendering, sized for 500k+ visible points.
- Filter chips above the canvas: by condition, by video, by cluster visibility.
- Density (heatmap) toggle for studies where points blur together.
- Selection drives the cluster panel and frame grid — selecting in one place updates the others.

### 2. Cluster panel

- Sortable list: cluster ID, name, frame count, % of total, condition breakdown bar.
- Inline rename — the primary interpretive action, where "cluster 7" becomes "rearing." Names persist (see Persistence).
- Condition-breakdown bar per cluster (e.g., 60% vehicle / 40% drug) so signal-rich clusters surface at a glance.
- Merge action: combine cluster IDs into one named group. Label-only; original IDs stay in the data, no re-clustering.
- Click a row to isolate it on the scatter and populate the frame grid.

### 3. Frame grid

- For the selected cluster (or lasso selection), show representative frames: k-medoids if the API supplies them, otherwise a stratified sample across videos and timestamps.
- Each tile: static thumbnail, cluster-color border, timestamp, source video ID, condition label.
- Capped at ~64 tiles per page with pagination.
- Click a tile to open the single-video deep dive at that timestamp.
- Substitutes for synchronized multi-video playback (out of scope).

### 4. Single-video deep dive

- HTML5 player with play, pause, scrub, frame-step, playback rate.
- Timeline strip below: biometric tracks (speed, posture) plus a cluster-assignment track colored by cluster.
- Current playback frame highlighted as a moving dot on the embedding view.
- Keypoint overlay (nose, neck, paws) is a toggle, off by default.
- Click the timeline to jump the video; click a scatter point in this video to jump to that frame.

### 5. Condition comparison

- Side-by-side cluster-usage charts: per-cluster frame share by condition (drug vs. vehicle, or compound 1..N vs. vehicle vs. positive control).
- Per-cluster effect size (e.g., Cohen's h or log-ratio) with a visual flag when |effect| crosses threshold.
- Small multiples — one mini-chart per cluster — for studies with more than 2 conditions.
- The figure the study director exports for the go/no-go meeting.

### 6. Export

- **CSV** of interpretive state: per-cluster name, frame count, per-condition aggregations, effect sizes, active filters. Encodes decisions, not raw coordinates (raw data stays in the API).
- **PNG / SVG** of comparison figures and the embedding view, sized for slides (1920×1080 default, configurable).
- **Shareable URL** (see Persistence) is the third export channel.

### 7. Persistence

- Cluster names, merges, and filters save per study, scoped to the user via BioSyft's existing auth.
- Shareable URL encodes the current view (selected cluster, filters, video, timestamp) so a recipient lands on the same screen.
- Auto-save on rename; no explicit save action.

## Out of Scope

| Excluded | Why |
|---|---|
| Re-running clustering or changing clustering parameters | Compute is BioSyft's pipeline; dashboard is read-only over results. |
| Supervised model retraining or labeling for training | This product is explicitly the unsupervised complement. |
| Multi-study cross-comparison | One study at a time; cross-study is a v2 question about cluster identity stability. |
| Real-time / live ingestion | Studies are batch; pipeline finishes before users open the dashboard. |
| Mobile / tablet UI | Desktop-only. Personas live on laptops with PowerPoint and Excel. |
| User authentication, ACLs, org admin | Inherited from the existing BioSyft platform; not rebuilt. |
| Synchronized multi-video playback | Replaced by the frame grid. Costly, marginal value at 500-video scale. |
| Statistical hypothesis testing beyond effect sizes | Study director runs proper stats in R; dashboard surfaces signal, doesn't certify it. |
| Editing biometrics, keypoints, or video metadata | Read-only over the REST API. |
| Custom dimensionality reduction (UMAP/t-SNE knobs) | Coordinates come from the API as-is. |
| Annotation of individual frames (notes, tags per frame) | Cluster-level naming covers the interpretive need; frame-level annotation is v2. |
| Offline / on-device export of raw video | Videos stream from the API; bulk download is a platform concern. |

## Assumptions

- **API shape** — REST endpoints for: unsupervised coordinates (frame → x, y, cluster_id, video_id, frame_number); per-frame biometrics; per-frame keypoints; video streams (byte-range MP4). Where ambiguous, `data_architecture.md` picks the easiest form to consume.
- **Scale ceiling** — Up to ~500 videos × 10–20 min × 100 FPS ≈ 6–60M frames per study. The embedding view subsamples or aggregates above ~500k visible points.
- **Cluster count** — Tens per study, not hundreds. Cluster panel is a flat list.
- **Conditions** — Video metadata (e.g., `condition: "vehicle"`). Assignment is upstream; the dashboard only reads.
- **Representative frames** — k-medoid frame IDs per cluster from the API, or deterministic sampling as fallback.
- **Auth** — Inherited from BioSyft's platform; the dashboard runs inside that session.
- **Hosting** — Static frontend on BioSyft's existing infra; optional BFF only if caching demands it (see `tech_stack.md`).
- **Browser** — Modern Chromium / Firefox / Safari on desktop. WebGL2 required.

## Constraints Recap

- **Timeline** — 3 weeks, end to end. Breakdown in `timeline.md`.
- **Team** — One developer. No design partner, no separate backend.
- **Personas** — Both must succeed without a tutorial; the study director must get a slide-ready PNG without leaving the app.
- **Read-only over the API** — Dashboard writes only interpretive state (names, merges, filters, view state).

## Success Criteria Pointer

Concrete metrics — time-to-first-named-cluster, export-to-slide latency, scatter render time at 500k points — live in `goals.md`. Scope commits to surface area; goals commit to the bar.

## Decision Log

- **No multi-video sync playback.** Frame grid + deep dive cover the need at far lower cost.
- **Merging is label-only.** Keeps the dashboard read-only over the ML pipeline.
- **CSV export encodes decisions, not raw frames.** Raw frames live in the API; users want their interpretation.
- **Density mode in, 3D scatter out.** Density solves scale; 3D adds rotation UX without insight.
- **Frame-level annotation out.** Cluster names are the v1 unit of interpretation.
