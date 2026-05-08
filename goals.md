# Goals & Success Metrics

This document defines what success looks like for the BioSyft unsupervised behavior dashboard MVP. Targets are measurable, persona-anchored, and bounded by the 3-week solo-developer timeline.

## Guiding Principle

The dashboard succeeds when a non-technical PhD student can name clusters and find drug effects without help, and a technical study director can produce slide-ready figures for a go/no-go meeting in a single session. Everything else is secondary.

## Per-Persona Success Metrics

### PhD student (academia, ~40 videos, drug vs. vehicle)

The PhD student is Excel-native and has never used a custom analysis dashboard. Success means interpretation, not engineering.

| # | Metric | Target |
|---|--------|--------|
| P1 | Time from landing on dashboard to first cluster renamed | <= 5 minutes, no documentation read |
| P2 | Can name all clusters in a 40-video / ~24M-frame study in one sitting | <= 60 minutes, single session |
| P3 | Can identify at least one cluster with a between-condition effect size flagged by the comparison view | Achievable without writing code or exporting raw data |
| P4 | Cluster names persist across sessions and across browser refresh | 100% persistence over a 2-week study window |
| P5 | Can answer "what does cluster 7 look like?" by clicking once on the cluster panel | <= 1 click to representative frame grid; <= 2 clicks to playing video at that frame |
| P6 | Self-reported confidence that named clusters reflect real behaviors | >= 4/5 on post-task survey (n>=3 pilot users) |

### Study director (industry, ~500 videos, 6 conditions, go/no-go)

The study director is technical (R/Python) and lives in GraphPad/PowerPoint. Success means defensible figures and exportable tables.

| # | Metric | Target |
|---|--------|--------|
| S1 | Can generate a per-condition cluster-usage figure suitable for a go/no-go slide | <= 10 minutes from dashboard load |
| S2 | PNG/SVG export of comparison figure renders at slide resolution (>=1920x1080, vector preferred) | 100% of exports |
| S3 | CSV export contains user-named clusters + per-condition aggregations + active filters | One file, opens cleanly in Excel and R |
| S4 | Shareable URL reproduces exact view (selection, filters, zoom) for program lead and biostatistician | Link works across users with study access; round-trip fidelity 100% |
| S5 | Effect-size indicators are visible per cluster x condition pair without writing stats code | Cohen's d or equivalent shown inline |
| S6 | Biostatistician can validate findings against the CSV without re-deriving cluster IDs | Cluster name column present and stable |

## Acceptance Criteria

These are pass/fail gates for the MVP. Each is independently testable.

### Embedding view (scatter)
- AC-E1: Renders 500,000 points at >= 30 FPS sustained pan/zoom on a 2020-or-newer laptop (integrated GPU acceptable).
- AC-E2: Lasso and box selection complete in <= 500 ms for a 500k-point scene.
- AC-E3: Hover-to-thumbnail latency <= 150 ms (cached) / <= 500 ms (cold).
- AC-E4: Color-by-cluster is stable across sessions; cluster IDs do not get reassigned on reload.

### Cluster panel & naming
- AC-C1: Inline rename commits on blur or Enter; visible everywhere within 200 ms.
- AC-C2: Names persist per study (server-side) and survive browser refresh, new device, and shareable-link load.
- AC-C3: Frame counts and per-condition breakdown bars are correct to the frame (validated against API totals).

### Frame grid & deep dive
- AC-F1: K-medoid representative grid loads <= 2 s for any cluster.
- AC-F2: Click-through from grid tile to single-video player at correct timestamp (+/- 1 frame at 100 FPS) works 100% of the time.
- AC-F3: Biometric timeline (speed, posture) and cluster-assignment track align to the video clock within 10 ms.
- AC-F4: Current-frame highlight on the scatter updates at >= 10 Hz during playback.

### Condition comparison
- AC-X1: Side-by-side cluster usage renders for any partition of conditions in <= 3 s.
- AC-X2: Effect-size indicator (Cohen's d or equivalent) shown per cluster, with a clear visual threshold for "noteworthy."
- AC-X3: Figure exports as PNG (>=1920x1080) and SVG; both open cleanly in PowerPoint and Illustrator.

### Export & persistence
- AC-P1: CSV export contains: cluster_id, user_cluster_name, condition, frame_count, fraction_of_condition, effect_size_vs_reference, active_filter_summary.
- AC-P2: Shareable URL encodes selection, zoom, active filters, and is <= 2 KB.
- AC-P3: View state and cluster names are stored server-side keyed by study + user; no data loss on logout.

## Performance Targets

| Surface | Cold load | Warm interaction |
|---------|-----------|------------------|
| Initial dashboard (40-video study, ~24M frames downsampled to <= 500k scatter points) | <= 5 s to first interactive scatter | n/a |
| Initial dashboard (500-video study, ~60M frames) | <= 10 s to first interactive scatter via progressive Arrow loading | n/a |
| Pan / zoom scatter | n/a | >= 30 FPS at 500k points |
| Cluster select -> frame grid | n/a | <= 2 s |
| Frame click -> video playing at timestamp | n/a | <= 3 s (byte-range MP4) |
| CSV export | <= 5 s for 500-video study | n/a |
| Comparison figure render | <= 3 s | <= 1 s on filter change |

## Scale Behavior (60M-frame upper bound)

- The scatter never attempts to render 60M raw points. Above 500k visible points the system uses density-aware downsampling or tile-based aggregation; cluster identity is preserved.
- API responses use Apache Arrow + progressive loading; the UI is interactive before the full payload arrives.
- Memory ceiling on the client: <= 1.5 GB resident for a 500-video study on a 16 GB laptop.
- Server-side cluster aggregations (frame counts, per-condition breakdowns, effect sizes) are precomputed or cached; the client never iterates 60M rows.

## Non-Goals (We Do Not Measure Success On These)

- Re-running clustering or changing clustering hyperparameters from the UI.
- Editing or correcting keypoint data; keypoint overlay is read-only and optional.
- Multi-study cross-comparison in one view; one study at a time.
- Synchronized multi-video playback; the frame grid is the explicit substitute.
- Real-time / streaming ingestion; analysis is assumed complete before the dashboard opens.
- Statistical inference beyond effect-size indicators (no p-values, no model fitting in the UI).
- Mobile or tablet layouts; desktop Chrome/Edge/Firefox at >= 1440 px width only.
- Authentication, user management, billing — assumed handled by the existing BioSyft platform.
- Offline mode.
- Annotation/labeling workflows beyond cluster naming.

## Definition of Done (3-Week MVP)

The MVP is done when all of the following are true:

1. All seven UI components from `scope.md` are implemented at the acceptance-criteria level above: embedding view, cluster panel, frame grid, single-video deep dive, condition comparison, export, persistence.
2. Every acceptance criterion (AC-E*, AC-C*, AC-F*, AC-X*, AC-P*) passes on the reference 40-video study.
3. The 500-video / 60M-frame scale tests pass for cold load and pan/zoom; degraded but usable behavior is documented for anything that does not hit target.
4. At least one PhD-student pilot user completes the P1-P5 flow unaided in a single session.
5. At least one study-director pilot user produces a go/no-go-ready PNG and a CSV that opens cleanly in Excel and R.
6. Shareable links round-trip across two browsers / two users with study access.
7. Cluster names and view state persist across logout and across devices.
8. Deployed to BioSyft's existing static-hosting + BFF environment; no new infra introduced.
9. Known limitations and non-goals are written down and linked from the dashboard's help affordance.
10. Code is in main; a 1-page runbook covers deploy, rollback, and the cache-invalidation path for cluster aggregations.

If any item 1-7 fails, the MVP is not done. Items 8-10 are blockers for handoff but not for internal demo.
