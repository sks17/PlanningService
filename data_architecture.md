# Data Architecture

How bytes get from BioSyft's REST API into the PhD student's scatter plot and the study director's PowerPoint figure, without melting the browser.

## Scaling Math (Why Architecture Matters Here)

A worst-case industry study:

```
500 videos × 20 min × 60 s × 100 FPS = 60,000,000 frames
```

Academia best case:

```
40 videos × 10 min × 60 s × 100 FPS = 24,000,000 frames
```

Even the academic case is 24M points. A naive JSON payload of `{x, y, cluster, video_id, frame}` per frame at ~80 bytes JSON-encoded:

```
24M × 80 B  ≈ 1.9 GB    (academic)
60M × 80 B  ≈ 4.8 GB    (industry)
```

That's a non-starter over HTTP, in browser memory, and on the GPU. We do not render every frame. We render a **downsampled, level-of-detail (LOD) embedding** plus on-demand detail when the user zooms or selects.

## Frame Addressing

The canonical primary key everywhere in the app is the tuple:

```
(video_id: string, frame_number: uint32)
```

Time-in-video is derived, never stored:

```
t_seconds = frame_number / 100        // 100 FPS is fixed
t_mmss    = format(t_seconds)
```

A 64-bit packed key is used for hashing and Arrow columns where convenient:

```
frame_uid = (hash(video_id) << 32) | frame_number
```

Cluster IDs are integers from the upstream pipeline (`cluster_id: int16`, -1 reserved for noise). User-given **cluster names** are a *separate* layer (see Persistence) — the pipeline cluster IDs are immutable, the names are not.

## Assumed REST API

BioSyft's API is underspecified in the brief. We assume the following endpoints exist or can be added behind a thin BFF. **All shapes below are assumptions.**

| Endpoint | Method | Returns | Format |
|---|---|---|---|
| `/studies/{study_id}` | GET | study metadata, video list, condition labels | JSON |
| `/studies/{study_id}/embedding?lod={0..3}` | GET | downsampled coords + cluster labels | Arrow IPC |
| `/studies/{study_id}/embedding/cluster/{cluster_id}` | GET | full-resolution points for one cluster | Arrow IPC |
| `/studies/{study_id}/embedding/video/{video_id}` | GET | full-resolution points for one video | Arrow IPC |
| `/studies/{study_id}/clusters` | GET | per-cluster aggregates (count, condition breakdown, medoids) | JSON |
| `/videos/{video_id}/biometrics?from={f0}&to={f1}` | GET | speed/posture/etc. per frame, range | Arrow IPC |
| `/videos/{video_id}/keypoints?frame={n}` | GET | keypoints for one frame (lazy) | JSON |
| `/videos/{video_id}/mp4` | GET (Range) | MP4 byte range | video/mp4 |
| `/videos/{video_id}/sprites/{cluster_id}.webp` | GET | sprite sheet of medoid thumbnails | image/webp (CDN) |
| `/studies/{study_id}/state` | GET/PUT | named clusters, view state | JSON |

### Embedding payload (Arrow schema)

```
schema {
  video_id:     dict<utf8>     // dictionary-encoded; ~500 unique strings
  frame_number: uint32
  x:            float32
  y:            float32
  cluster_id:   int16
  condition:    dict<utf8>     // optional, joined server-side
}
```

Arrow IPC compresses this to roughly **18 bytes/row on the wire** (LZ4) vs. 80+ for JSON, and parses in a single zero-copy pass into typed arrays that deck.gl consumes directly. JSON.parse for 5M rows blocks the main thread for seconds; Arrow does not.

### Biometrics payload

```
schema {
  frame_number: uint32
  speed:        float32
  posture:      float32        // pipeline-derived scalar
  ... (additional scalars assumed extensible)
}
```

Fetched per-video, range-limited, on entry to the single-video deep-dive view.

## Wire Format Choices

| Data | Format | Why |
|---|---|---|
| Embedding coords + labels | **Arrow IPC over HTTP** (LZ4) | Columnar, typed, zero-copy to TypedArray, ~4× smaller than JSON, parses off-main-thread |
| Per-frame biometrics | Arrow IPC | Same reasons; uPlot consumes Float32Array natively |
| Video bytes | **Byte-range MP4** via `<video>` | Browsers already do this well; no re-encoding; seek-on-demand |
| Thumbnails / medoid sprites | WebP sprite sheets via CDN | One HTTP request per cluster covers ~64 medoids |
| Small metadata, keypoints, study state | JSON | Tiny payloads; readability wins over efficiency |

## Progressive / LOD Loading Strategy

The embedding loads in **three passes**, viewport-prioritized:

1. **LOD 0 — overview (always loaded first).** Server-side stratified sample of ~200k points, balanced across clusters. ~3.6 MB Arrow over the wire. Renders the full scatter immediately. Sufficient for the study director's overview.
2. **LOD 1 — by-cluster detail (lazy).** When a user clicks a cluster or lassoes a region, fetch full-resolution points for the relevant `cluster_id`s. Cached in TanStack Query under `['embedding', studyId, 'cluster', clusterId]`.
3. **LOD 2 — by-video detail (lazy).** When the user enters single-video deep-dive, fetch all points for that `video_id` to drive the "current frame highlighted on scatter" feature.

Server-side downsampling for LOD 0 is a **stratified random sample** keyed deterministically on `frame_uid` so the sample is stable across reloads (important for shareable URLs).

Viewport culling is handled by deck.gl in the GPU; we don't try to do it on the CPU.

## Caching Layers

```
Browser ──► CDN ─────────────► Sprite sheets, MP4 segments, immutable Arrow LOD0
   │                           (Cache-Control: public, max-age=31536000, immutable)
   │
   ├──► Browser HTTP cache ──► Arrow batches keyed by ETag/URL
   │                           (max-age=3600 + stale-while-revalidate)
   │
   ├──► TanStack Query ──────► In-memory normalized server state
   │                           (staleTime: 5 min for embeddings, ∞ for medoids)
   │
   └──► Zustand ─────────────► Client-only state:
                               - selection (cluster ids, lassoed frame_uids)
                               - cluster names (optimistic, synced to /state)
                               - view (zoom, pan, active video, current frame)
                               - filters (condition, video subset)
```

URLs for immutable resources (sprite sheets, LOD 0 embedding for a finalized study) include a content hash so they can be cached forever. Mutable resources (`/state`) bypass HTTP cache.

## Persistence

**Assumption:** BioSyft has, or will expose, a per-study key-value store accessible via `/studies/{study_id}/state` (or equivalent). If it does not, the BFF maintains a small Postgres/SQLite table keyed on `(study_id, user_id)`.

Persisted state shape (JSON):

```json
{
  "version": 1,
  "cluster_names": { "7": "rearing", "12": "grooming" },
  "view": {
    "camera": { "x": 0.4, "y": -1.2, "zoom": 3.1 },
    "selection": { "clusters": [7, 12], "frame_uids": [] },
    "filters":   { "conditions": ["drug"], "videos": null },
    "active_video": "vid_0042",
    "active_frame": 18420
  },
  "updated_at": "2026-05-07T10:21:00Z"
}
```

Shareable URL encodes a compressed form of `view` (plus `study_id`) in the query string so the study director's link drops the program lead into the same screen without a round-trip to `/state`.

## Export Pipeline

All export is **client-side**. The user's interpretive state lives in the browser; export is just serialization.

| Export | Source | Library |
|---|---|---|
| `clusters.csv` (named clusters + per-condition counts) | Zustand state + cached cluster aggregates | hand-rolled CSV (Papaparse if needed) |
| `frames.csv` (current selection: video_id, frame, time, cluster, name) | Zustand selection + Arrow batches | hand-rolled |
| Comparison figure PNG/SVG | Observable Plot's `.plot()` SVG output | Observable Plot |
| Embedding screenshot PNG | deck.gl canvas `.toDataURL()` | deck.gl |

CSV exports encode the **interpretive state** (the human-given names and the active filters), not raw 60M-frame coordinate dumps. The PhD student dropping the CSV into Excel sees `rearing, 1240 frames in drug, 312 in vehicle`, not floats.

## Persona Notes

- **PhD student (Excel-native).** Sees Arrow only as fast-loading scatter; the export they care about is `clusters.csv`. The cache hit on a refresh ("did I lose my names?") is the moment that earns trust — that's why `/state` is mandatory and why optimistic updates in Zustand sync through TanStack mutations.
- **Study director (R/Python, GraphPad).** Cares about LOD-0 latency for the program-lead demo, and SVG fidelity of comparison figures for slides. Both are wire-format decisions made above.

## What We Are Explicitly Not Doing

- Not streaming all 60M frames to the client. Ever.
- Not running cluster recomputation client-side — the upstream pipeline owns clustering.
- Not building a custom video player or re-encoding video — `<video>` + byte-range Range requests is enough.
- Not persisting raw selections of millions of frames to `/state` — selections by cluster id are tiny; ad-hoc lasso selections are session-local.
