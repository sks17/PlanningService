# User Workflows

Persona-specific walkthroughs for the BioSyft unsupervised behavior dashboard. Two primary personas (PhD student, study director) plus two downstream roles (program lead, biostatistician). Numbers assumed: 100 FPS, 10–20 min videos, ~40 videos for academic studies, ~500 videos for industry studies, 3-week build.

## Entry Points

- **Assumption:** The dashboard is a single-page app reached from BioSyft's existing platform via a "View unsupervised analysis" link on the study page. No separate login; SSO from the parent platform.
- The link carries the `studyId` as a path or query param: `/unsupervised/<studyId>`.
- A shareable URL also encodes view state (selected cluster, active filters, zoom box, condition split). See exit points below.
- **Assumption:** When a study is opened for the first time, the dashboard initiates progressive Arrow loading (per `tech_stack.md`) and shows a loading skeleton with cluster panel populated first, scatter second.

## Workflow 1 — PhD Student: Drug vs. Vehicle Pharmacology Study (~40 videos)

**Time budget:** ~2 hr first session (cluster naming dominates), ~30 min for follow-up sessions. **Persona:** PhD student, non-developer, Excel-native, never used a custom analysis dashboard.

1. **Land on dashboard.** PhD student receives an email/notification: "Analysis finished for Study DRG-042." Click-through opens the dashboard in a new tab. Cluster panel shows ~15–25 unnamed clusters ("cluster 1" … "cluster N") with frame counts; embedding scatter renders all ~40 videos × 60–120k frames colored by cluster.
2. **Orient.** Read the inline help tooltip ("Each dot is one video frame. Same color = similar behavior."). Hover over a few points to see thumbnail previews. Pan/zoom to verify the data looks plausible.
3. **Open the largest cluster first.** Click the top row in the cluster panel (sorted by frame count). Frame grid populates with k-medoid representatives. The student watches the thumbnails and recognizes "grooming."
4. **Confirm with deep dive.** Click one representative tile. Single-video deep dive opens at that timestamp. Play ±5 seconds. Biometric timeline below shows low speed, stable posture — consistent with grooming. Close deep dive.
5. **Name the cluster.** Inline rename field in the cluster panel: type `grooming`. Save is automatic (debounced, persisted per `studyId`).
6. **Repeat for each cluster.** Iterate through the next ~10–20 clusters: rearing, locomotion-fast, locomotion-slow, freezing, sniffing-wall, etc. For ambiguous clusters, lasso a sub-region in the scatter to inspect a tighter sample.
7. **Split by condition.** Open the condition comparison view. Toggle the split to `condition` (drug vs. vehicle — assumed metadata field on each video). Side-by-side cluster-usage bar chart appears with named clusters on the x-axis and effect-size indicators (e.g., Cohen's d or log fold-change) marked.
8. **Identify the headline finding.** Notice "freezing" usage is 3× higher under drug than vehicle, with a large effect size. Click the freezing bar to filter the embedding scatter to just freezing frames; visually confirm the drug-condition points cluster tightly.
9. **Export figure.** Click "Export PNG" on the comparison plot, sized for advisor's slide deck. Click "Export CSV" — gets a file with named clusters as rows and per-condition frame counts/percentages as columns.
10. **Save shareable URL.** Copy the URL from the share button (encodes named clusters, condition split, freezing-cluster filter). Paste into the message to the advisor.

**Failure / edge cases for Workflow 1:**
- *One cluster is enormous and uninterpretable* (e.g., 40% of frames in one blob). Mitigation: cluster panel shows a warning chip "large cluster — consider sub-sampling"; user can lasso-select a sub-region and rename it `unsorted-low-activity`. The export CSV preserves the partial naming. No re-clustering on the dashboard (out of scope per `scope.md`).
- *One video failed to process.* The cluster panel and condition comparison flag the video as missing in a "Data quality" callout. Frame counts are computed over successful videos only; the export CSV includes a `missing_videos` column. PhD student is told to re-trigger that video upstream.
- *Biometrics partially missing for some frames.* The deep-dive biometric timeline shows greyed-out gaps; cluster naming is unaffected (it relies on thumbnails + video playback, not biometrics).
- *Browser tab crashes mid-session.* All cluster names are persisted server-side per debounce save; user reopens the URL and resumes. View-state (zoom, filter) is restored from the URL fragment.

## Workflow 2 — Study Director: 4 Compounds + Vehicle + Positive Control (~500 videos)

**Time budget:** ~30 min for the go/no-go review session (assumes clusters were named in a prior ~1.5 hr session, possibly by a junior team member). **Persona:** technical, R/Python-comfortable, GraphPad/PowerPoint-native, has used commercial behavioral tools.

1. **Load study.** Open `/unsupervised/STUDY-PROG-17`. Progressive Arrow load streams in; cluster panel renders within ~2s, scatter renders within ~10s for ~500 videos × ~90k frames. Cluster names already exist from the prior naming session.
2. **Skim the embedding.** Quick zoom-to-fit; visually check that 6 conditions (vehicle, positive control, compounds A/B/C/D) are represented across the scatter. No obvious data-loading anomalies.
3. **Open condition comparison.** Switch split to `compound` (6 levels). Small-multiples view: one bar chart per named cluster showing the 6 conditions, ordered with vehicle leftmost and positive control rightmost. Effect-size indicators highlight differences vs. vehicle.
4. **Triage compounds.** Scan for clusters where compound A/B/C/D bars resemble the positive control more than vehicle. Compound B looks like a hit on "freezing" and "rearing-suppressed." Compound D looks vehicle-like across the board.
5. **Drill into Compound B.** Click compound B in the legend to filter scatter to B's frames only. Lasso a dense region in the freezing cluster, click a representative tile, deep-dive video to spot-check a few seconds of behavior. Confirm it is genuinely freezing, not artifact.
6. **Compose the go/no-go figure.** From the comparison view, select the 3–5 clusters with the largest effect sizes. Click "Export SVG" — slide-sized vector. Click "Export CSV" — wide table: rows = named clusters, columns = per-condition mean usage %, n, effect size vs. vehicle.
7. **Capture supplementary.** Take a separate PNG of the embedding scatter with all 6 conditions colored to include as a backup slide.
8. **Generate shareable URL.** Click share → URL encodes: split=compound, selected clusters, scatter zoom box, current filter. Copy.
9. **Send hand-off.** Email program lead and biostatistician with: SVG, CSV, and the URL. (The dashboard does not send emails; user pastes into their existing email client.)

**Failure / edge cases for Workflow 2:**
- *One cluster is enormous and not interpretable* (common at 500-video scale). The director marks it `mixed-low-signal` and excludes it from the export figure via a per-cluster "exclude from comparison" toggle. CSV records the exclusion flag.
- *One or more videos failed to process.* The "Data quality" callout lists missing videos by ID and condition. If a condition lost >10% of its videos the comparison view shows an amber warning recommending the biostatistician review n's before publishing.
- *Biometrics partially missing.* Does not block the go/no-go figure (cluster usage % is independent of biometrics). Deep-dive timeline shows gaps; an `n_frames_with_biometrics` column is added to the CSV for downstream stats.
- *Scatter rendering stutters at 500k+ points.* deck.gl handles this per `tech_stack.md`; if the user's GPU is weak, a "Reduce density" toggle subsamples to 100k points for navigation while keeping all points in the underlying CSV/stats.

## Workflow 3 — Program Lead Hand-off (downstream of study director)

**Time budget:** 5–10 min. **Persona:** decision-maker, non-analyst, opens the URL on a laptop between meetings.

1. Click the URL the study director sent. Dashboard opens directly to the condition-comparison view with split=compound and the director's selected clusters pre-highlighted (state from URL).
2. Sees the same SVG-equivalent on screen. Hovers a bar to see exact mean ± SD and n.
3. Optional: clicks one cluster to drill into the frame grid and spot-check a representative tile. Does *not* rename clusters or export anything.
4. Replies to the director's email with go/no-go decision.
- **Edge case:** Program lead opens the URL after the director has renamed a cluster — names update live (read-after-write). If the director has deleted a cluster name (cleared the field), the cluster falls back to `cluster N` and the URL still resolves.

## Workflow 4 — Biostatistician Validation Hand-off

**Time budget:** ~45 min. **Persona:** statistical reviewer, R/Python user.

1. Opens the same URL the director sent. Lands on the comparison view.
2. Clicks "Export CSV" to get the underlying per-cluster, per-condition aggregations including n, mean, SD, and effect size. Also exports a second CSV (per-frame: video_id, frame, cluster_id, cluster_name, condition) from a "Raw export" button — needed to recompute stats independently in R.
3. Reads the CSV into R, recomputes effect sizes and confidence intervals, runs their own multiple-comparison correction. Compares to the dashboard's numbers.
4. Reviews the "Data quality" callout for missing videos / missing biometrics counts; flags any condition with degraded n.
5. Sends a Slack/email reply to the director: "Stats reproduce; recommend reporting CI; n for compound C is borderline."
- **Edge case:** Cluster names changed between the director's send-time and the biostatistician's open-time. The URL state captures cluster *IDs*, not names, so analysis remains stable; the CSV exports with whatever the latest names are and includes a `cluster_id` column for joins.

## Exit / Export Points (summary)

- **CSV (named, aggregated):** rows = named clusters, columns = per-condition counts/percent/effect size; includes data-quality columns (missing videos, missing-biometric frames). Used for slide tables and for joining in R/Python.
- **CSV (raw frame-level):** video_id, frame, cluster_id, cluster_name, condition. Gated behind a "Raw export" affordance; biostatistician's primary artifact.
- **PNG of comparison figure:** raster, slide-sized (1920×1080 default), for PowerPoint.
- **SVG of comparison figure:** vector, for publication-quality slides and posters.
- **PNG of embedding scatter:** for supplementary/backup slide.
- **Shareable URL:** encodes studyId, split dimension, selected clusters, scatter zoom box, active filters. Cluster names live server-side and are read on load.

## Cross-cutting Assumptions

- Single-page nav from BioSyft's existing platform; no auth UI built here.
- Condition metadata (drug/vehicle, compound label) is provided per video by the upstream platform — no manual condition-tagging UI in this scope.
- Cluster naming is the only persisted user state besides view-state; no comments/annotations on individual frames in v1 (out of scope).
- All exports happen client-side from the loaded data; no server-side export job.
- The biostatistician's "raw export" is the largest payload; for ~500 videos × ~90k frames this is ~45M rows. **Assumption:** raw export is streamed as gzipped CSV from the BFF to avoid blowing up the browser; if this proves expensive in week 2 of the build, fall back to a "request raw export" link that emails a download URL.
