# Tech Stack

The stack is chosen to deliver the dashboard described in `scope.md` and `ui_reqs_rough.md` within the 3-week solo-developer budget, while handling up to ~500 videos at 100 FPS (~60M frames/study) without re-engineering BioSyft's existing pipeline. Every choice below favors boring, well-documented tooling over novelty — the risk budget is spent on the embedding view (deck.gl) and the wire format (Arrow), where the scale demands it.

## Summary Table

| Layer | Choice | Why |
|---|---|---|
| Frontend bundle | React + TypeScript + Vite | Standard SPA toolchain, fast HMR, large hiring/AI assist surface |
| Static hosting | Whatever BioSyft already uses (CDN/S3+CloudFront/Vercel) | Don't introduce infra in a 3-week build |
| Scatter plot | deck.gl (`ScatterplotLayer`) | WebGL, scales to 500k+ points with zoom/pan/lasso |
| Statistical charts | Observable Plot | Clean SVG export for PowerPoint/GraphPad-bound study directors |
| Timeline (biometrics, cluster track) | uPlot | Canvas-based, sub-ms redraws on 100 FPS time series |
| Wire format | Apache Arrow + progressive loading | JSON serialization breaks at multi-million-row embedding payloads |
| Video playback | HTML5 `<video>` + byte-range MP4 | BioSyft already serves video; don't rebuild it |
| Thumbnails / frame grid | CDN-served sprite sheets per cluster | One HTTP request per cluster; CSS `background-position` per tile |
| BFF | FastAPI (or Node) on Cloud Run / Fly.io / BioSyft's K8s | Persistent process for Arrow assembly + cache; not Lambda |
| Server state | TanStack Query | Caching/dedup/invalidation is load-bearing for this UI |
| Client state | Zustand | Cluster names, selection, filters, view state — no Redux ceremony |

---

## Frontend Bundle: React + TypeScript + Vite

React is the default for dashboards of this complexity; deck.gl, TanStack Query, and Zustand all ship first-class React bindings, so picking anything else (Svelte, SolidJS) costs integration time we don't have. TypeScript catches the class of bug that wrecks data dashboards: shape mismatches between API payloads and components. Vite over CRA/Webpack for cold-start speed (sub-second HMR matters when iterating wireframes daily) and native ESM.

**Alternatives rejected:**
- **Next.js** — server rendering buys nothing for an authenticated, interactive analytics tool, and adds a deploy surface (Node runtime) we don't need.
- **Svelte/SolidJS** — smaller ecosystem for the WebGL/Arrow libraries below; not worth the perf wins on a 3-week timeline.

**Risks:** None material. React 18 concurrent features are stable; Vite is mature.

---

## Scatter Plot: deck.gl

The embedding view must render every frame in a study — potentially hundreds of thousands of points after sub-sampling, millions if we don't sub-sample — with zoom, pan, lasso, and hover-to-preview. SVG-based libraries (D3, Recharts, Plotly's SVG mode) collapse above ~10k points. deck.gl's `ScatterplotLayer` is GPU-accelerated, handles 1M+ points at 60fps, and supports the picking/lasso interactions we need via `@deck.gl/extensions`.

**Alternatives rejected:**
- **Plotly** — its WebGL `scattergl` works up to ~100k points, but customization and lasso behavior are clunky, and the bundle is heavy.
- **regl-scatterplot** — lighter, but smaller community and we'd reimplement camera/picking utilities deck.gl gives for free.
- **Three.js direct** — too low-level for the timeline.

**Risks:** deck.gl has a real learning curve (layers, coordinate systems, view state). Mitigation: stick to the documented `ScatterplotLayer` + `OrthographicView` recipe; don't write custom shaders.

---

## Statistical Charts: Observable Plot

The condition comparison view (cluster usage, effect sizes) is the figure the study director exports to PowerPoint. Observable Plot produces clean SVG by default, has a grammar-of-graphics API that makes small-multiples trivial, and is maintained by the same team behind D3.

**Alternatives rejected:**
- **Recharts** — React-native but limited grammar; small-multiples and faceting are painful.
- **Vega-Lite** — more expressive but heavier and slower to author.
- **D3 directly** — too much code for a 3-week build.

**Risks:** Plot is younger than D3; some edge cases in legends/axes need hand-tuning. Acceptable.

---

## Timeline: uPlot

The single-video deep dive shows biometrics (speed, posture) and cluster assignment as time series under the video. At 100 FPS over 20 minutes that's 120k samples per track. uPlot is canvas-based and benchmarked at the top of every charting library shootout; it redraws in <5ms at this scale.

**Alternatives rejected:**
- **Chart.js** — fine for small data, slow at 100k+ points.
- **Observable Plot** — already chosen for stats; using it here too would couple unrelated views and lose the perf headroom we need for video-synced playback.

**Risks:** uPlot's API is terse and imperative. We wrap it in a thin React component once and move on.

---

## Wire Format: Apache Arrow + Progressive Loading

Embedding payloads (frame_id, x, y, cluster_id, video_id) for a 500-video study are tens to hundreds of MB. JSON parses at ~50–100 MB/s and bloats numeric data ~3x; the browser hangs. Arrow IPC streams parse near-zero-copy into typed arrays that deck.gl can consume directly — no per-point object allocation. Progressive loading (stream chunks per video, render as they arrive) keeps first-paint under a few seconds.

**Alternatives rejected:**
- **JSON** — fails at scale, period.
- **Protobuf** — efficient but requires schema codegen and doesn't give us the columnar typed-array layout deck.gl wants.
- **CSV** — text-based, same parse-cost problem as JSON.

**Risks:** Arrow tooling on the BFF side (`pyarrow` for FastAPI) is solid; if BioSyft's existing API is JSON-only, the BFF must transcode JSON→Arrow once and cache. Browser-side `apache-arrow` JS package adds ~150 KB gzipped — acceptable.

---

## Video: HTML5 `<video>` + Byte-Range MP4

BioSyft already stores and serves videos. The dashboard requests the MP4 by URL and seeks via the HTML5 video element, which uses HTTP byte-range requests against any S3/CDN origin that supports them (the default). No HLS, no DASH, no transcoding pipeline owned by us.

**Alternatives rejected:**
- **HLS.js / video.js** — needed only if we were transcoding to chunked formats, which we aren't.
- **Custom WebGL frame decoder** — overkill; the browser does this.

**Risks:** Frame-accurate seeking at 100 FPS depends on the MP4 having close-spaced keyframes. **Dependency on BioSyft:** their MP4 encode must use a keyframe interval ≤ 1s, or seek granularity will be coarse. We confirm this in week 1.

---

## Thumbnails: CDN-Served Sprite Sheets per Cluster

The frame grid shows ~25–50 representative frames per cluster. Per-frame requests = thundering herd. Per-cluster sprite sheet (a single JPEG with all medoids tiled) = one request, rendered in the grid via CSS `background-position`. Sprite generation runs once at study-analysis time on BioSyft's existing video pipeline (ffmpeg), uploaded to their CDN.

**Alternatives rejected:**
- **Per-frame JPEG URLs** — works but blows out the connection pool and CDN cache on a 500-video study.
- **Client-side frame extraction from video** — slow, requires seeking the full MP4 just for a thumbnail.

**Risks:** Sprite generation is a new pipeline step. **Dependency on BioSyft:** we need a hook in their analysis pipeline to emit thumbnails, or a one-shot worker we run post-analysis. Estimated half-day of work.

---

## BFF (Backend-for-Frontend): FastAPI on a Persistent Host

The dashboard needs: (1) Arrow assembly from BioSyft's REST API, (2) per-study cache of embedding + cluster-name persistence, (3) auth pass-through. FastAPI gives us `pyarrow` natively, async I/O for fan-out to BioSyft's API, and Pydantic for the wire schema. Deployed on Cloud Run, Fly.io, or BioSyft's existing K8s — wherever has the least ops overhead.

**Alternatives rejected:**
- **AWS Lambda / serverless** — cold starts plus 6 MB response cap kill Arrow streaming.
- **Node + Fastify** — viable, but `pyarrow` is more mature than `apache-arrow` JS for server-side assembly, and FastAPI's typing story is excellent.
- **No BFF, talk to BioSyft API directly** — only viable if their API already returns Arrow; otherwise transcoding in the browser is wasted work.

**Risks:** One more service to operate. **Dependency on BioSyft:** an internal service token or pass-through auth into their REST API.

---

## Server State: TanStack Query

Embedding payloads, cluster lists, biometrics — all cached, deduped, invalidated on cluster rename, refetched on study change. Hand-rolling this with `useEffect` is the #1 way to ship a slow, buggy dashboard. TanStack Query also gives us request-in-flight indicators, retries, and stale-while-revalidate for free.

**Alternatives rejected:**
- **SWR** — equivalent, smaller. TanStack Query's mutation/invalidation API is more ergonomic for the rename workflow.
- **Redux Toolkit Query** — pulls in Redux, which we don't need.

**Risks:** None.

---

## Client State: Zustand

Cluster names (until persisted), current selection, lasso state, active filters, view state for shareable URLs. Zustand is ~1 KB, no provider tree, no boilerplate. Selectors prevent re-renders.

**Alternatives rejected:**
- **Redux Toolkit** — heavier, more boilerplate; we don't need time-travel debugging or middleware.
- **Context + useReducer** — fine for small trees but causes re-render storms on the embedding view.
- **Jotai/Recoil** — atom model is overkill for the handful of stores we need.

**Risks:** None.

---

## Build & Deploy Assumptions

- BioSyft has an existing CDN (CloudFront, Cloudflare, or equivalent) we can serve the SPA bundle and sprite sheets from.
- BioSyft's REST API is reachable from a server-side BFF with a service credential.
- BioSyft's video storage supports HTTP byte-range requests (S3 default; standard CDN behavior).
- BioSyft's MP4s are encoded with keyframe interval ≤ 1 second (verify week 1; if not, request a re-encode setting for new studies).
- We deploy the SPA as static assets and the BFF as a single container — no Kubernetes mastery required.
- CI: GitHub Actions running `vite build`, `pytest`, and a container build/push. No custom infra-as-code; use whatever BioSyft already runs.
- Auth: pass-through to BioSyft's existing session/token; we do not stand up our own identity.

## Cross-Cutting Risks

| Risk | Likelihood | Mitigation |
|---|---|---|
| deck.gl learning curve eats week-1 buffer | Medium | Time-box to documented recipes; ship sub-sampled 50k-point view first, scale up later |
| BioSyft REST API too slow / no Arrow | Medium | BFF caches first study fetch to local Parquet/Arrow; subsequent loads are instant |
| MP4 keyframe spacing too sparse for frame-accurate seek | Low–Medium | Confirm with BioSyft week 1; fall back to nearest-keyframe + thumbnail overlay |
| Sprite-sheet pipeline not ready | Low | Solo dev writes the ffmpeg one-shot in week 1; runs as a post-analysis step |
| Browser memory at 500-video scale | Medium | Sub-sample embedding to ≤500k points client-side; full data lives in BFF |
