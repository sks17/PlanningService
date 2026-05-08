# Timeline — 3-Week Solo Build

Solo developer, 3 calendar weeks, 5 working days each (15 days total). Assumes BioSyft's REST API and CDN are live; Day 1 begins with API contract clarification, not infra setup. Stack is fixed (see `tech_stack.md`): React + TS + Vite, deck.gl, Observable Plot, uPlot, Apache Arrow, TanStack Query, Zustand.

## Panel-to-Priority Map

| # | Panel | Priority | Persona Driver |
|---|-------|----------|----------------|
| 1 | Embedding view (scatter) | Critical (MVP) | Both |
| 2 | Cluster panel + rename | Critical (MVP) | PhD student (interpretation) |
| 3 | Frame grid | Critical (MVP) | PhD student (interpretation) |
| 4 | Single-video deep dive | Important | PhD student (validation) |
| 5 | Condition comparison | Critical (MVP) | Study director (go/no-go figure) |
| 6 | Export (CSV + PNG/SVG) | Critical (MVP) | Study director (slides) |
| 7 | Persistence + share link | Stretch | Study director (handoff) |

MVP = panels 1, 2, 3, 5, 6. Panel 4 ships if Week 2 runs clean. Panel 7 is the first cut.

## Dependency Graph

```
API contract  ──►  Arrow loader  ──►  Scatter (1)  ──►  Lasso/box select  ──►  Cluster panel (2)
                                          │                                         │
                                          ▼                                         ▼
                                      Hover thumb                              Frame grid (3)
                                          │                                         │
                                          ▼                                         ▼
                                   Video deep dive (4) ◄───── timestamp/frame ──────┘
                                          │
                                          ▼
                              Condition comparison (5)  ──►  Export (6)
                                                                  │
                                                                  ▼
                                                          Persistence (7)
```

Hard dependencies: scatter must render and support selection before the cluster panel is meaningful; cluster naming state must exist before export can encode it; condition comparison reads from the same cluster-assignment store as the panel, so it cannot start before Week 2.

## Week 1 — Foundation & the Scatter (Days 1–5)

Goal by Friday demo: a study loaded end-to-end, 500k points rendering smoothly, clusters colored, hover thumbnails working. Stakeholder sees "the map."

| Day | Focus | Deliverable |
|-----|-------|-------------|
| 1 (Mon) | API contract clarification with BioSyft; Arrow schema sketch; Vite + TS scaffold; TanStack Query + Zustand wiring | Running app shell, API client stub, agreed Arrow column layout |
| 2 (Tue) | Arrow ingestion + progressive load for unsupervised coords; cluster color palette (categorical, colorblind-safe) | `/study/:id` route loads coords, logs frame count |
| 3 (Wed) | deck.gl ScatterplotLayer; pan/zoom; LOD culling for 500k+ points | 60 fps scatter on a 60M-frame study sample |
| 4 (Thu) | Hover → CDN thumbnail preview (sprite sheet lookup); cluster ID tooltip | Hover any point, see frame thumb in <150 ms |
| 5 (Fri) | Lasso + box select; selection state in Zustand; **Week 1 demo** | Stakeholder demo: load study, explore the map, lasso a region |

Risk buffer this week: 0.5 day baked into Day 5 (lasso is well-trodden in deck.gl). If API contract slips on Day 1, Day 2 absorbs it.

## Week 2 — Interpretation Loop (Days 6–10)

Goal by Friday demo: the PhD-student workflow is complete. Lasso → see cluster → see representative frames → click into a video → name the cluster.

| Day | Focus | Deliverable |
|-----|-------|-------------|
| 6 (Mon) | Cluster panel: list, frame counts, condition breakdown bar | Panel reflects current selection; counts update on lasso |
| 7 (Tue) | Inline rename field; persisted to local state (server persistence is Week 3) | Rename "cluster 7" → "rearing"; survives reload via localStorage |
| 8 (Wed) | Frame grid: k-medoid representatives per cluster, sprite-sheet tiles, click-to-expand | Grid of 16–32 reps per selected cluster |
| 9 (Thu) | Single-video deep dive: HTML5 video + byte-range, frame-accurate seek by timestamp | Click a tile → land on that frame in player |
| 10 (Fri) | uPlot biometric timeline (speed, posture) + cluster-assignment track under video; current frame ↔ scatter highlight; **Week 2 demo** | Full interpretation loop demoable |

Risk buffer this week: Day 9 is the highest-risk day — frame-accurate seek on 100 FPS MP4 with byte-range is finicky. If it slips, fall back to second-accurate seek and ship; precision-seek becomes a Week 3 polish item. Keypoint overlay is explicitly deferred (it was already optional in the spec).

## Week 3 — The Study-Director Deliverable & Polish (Days 11–15)

Goal by Friday demo: a study director can produce the go/no-go figure and a CSV in under five minutes, end-to-end, on a real ~500-video study.

| Day | Focus | Deliverable |
|-----|-------|-------------|
| 11 (Mon) | Condition comparison: small-multiples bar charts via Observable Plot; effect-size indicators (Cliff's delta or Cohen's h) | Side-by-side cluster usage across conditions |
| 12 (Tue) | CSV export: named clusters + per-condition aggregations + active filters; PNG/SVG export of comparison figure sized for 16:9 slides | Working export buttons; CSV opens cleanly in Excel |
| 13 (Wed) | Persistence: PUT cluster names + view state to BFF; shareable URL encodes selection/filters | Reload + share link both restore state |
| 14 (Thu) | Performance pass on 500-video study; perceived-load tuning; empty/error states; PhD-persona usability fixes from Week 2 demo | Cold load <8 s, interaction <100 ms |
| 15 (Fri) | Bug bash, copy pass, Loom walkthrough, **final stakeholder demo** | Shippable build |

Risk buffer this week: Day 14 is the dedicated slack day. If Week 2 ran over by a day, persistence (Day 13) compresses into half a day and Day 14 absorbs the rest. If everything is on track, Day 14 polishes the export figures (study director's most-used artifact).

## Demoable Checkpoints

| End of | What stakeholder sees |
|--------|----------------------|
| Week 1 | "The map" — load a real study, fly around 500k+ points, hover for thumbnails, lasso a region |
| Week 2 | Full PhD-student loop — lasso, name a cluster, browse representative frames, drop into a video at the right frame, see biometrics |
| Week 3 | Study-director artifact — comparison figure exported as PNG, CSV opens in Excel with named clusters, share link works |

## Risk Buffer Summary

- **Built-in slack:** Day 5 PM (0.5d), Day 14 (1d). Total ~1.5 days across 15.
- **Where slip is most likely:** Day 1 (API contract ambiguity), Day 9 (frame-accurate video seek), Day 11 (effect-size choice + small-multiples layout).
- **Cut order if behind:** (1) drop persistence/share link, (2) drop keypoint overlay (already optional), (3) drop biometric timeline under video, (4) ship video deep-dive as "open in new tab at timestamp" instead of embedded player. The condition comparison and CSV export are never cut — they are the study director's reason for opening the dashboard.

## MVP vs Stretch

**MVP (must ship, Week 3 Friday):** scatter + lasso, cluster panel + rename, frame grid, condition comparison, CSV + PNG export.

**Important (ship if Week 2 clean):** single-video deep dive with biometric timeline, frame ↔ scatter highlight.

**Stretch (cut first):** server-side persistence and share links (degrade to localStorage), keypoint overlay toggle, SVG (in addition to PNG) export, dark mode.

## Key Assumptions

- BioSyft's REST API and CDN are live and responsive; thumbnail sprite sheets per cluster are either pre-built or cheap to build server-side.
- A representative ~500-video study is available by Day 3 for realistic perf testing — not just the 40-video academic sample.
- BioSyft hosts the static bundle (no infra work on our side).
- The k-medoid representative frames for each cluster are computed upstream and exposed via the API; we do not compute them client-side.
