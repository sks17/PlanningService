1. Embedding view (the centerpiece)
2D scatter of every frame, colored by cluster ID. Zoom, pan, lasso/box select. Hover a point → preview that frame as a thumbnail. WebGL-rendered so it survives 500k+ points.
2. Cluster panel
List of clusters alongside the scatter. Per cluster: frame count, condition breakdown bar, and an inline rename field. Naming is the primary interpretive action — this is where "cluster 7" becomes "rearing."
3. Frame grid
For a selected cluster, show representative frames (k-medoids or a sampled spread). Each tile is a static thumbnail with timestamp + source video. Click expands into the single-video deep dive. This is the substitute for synchronized multi-video playback.
4. Single-video deep dive
One video player with standard controls. Below it: a timeline of biometrics (speed, posture) and a track showing cluster assignment over time. Current frame is highlighted on the scatter while the video plays. Keypoint overlay on the video itself is optional — keep it as a toggle.
5. Condition comparison view
Side-by-side cluster usage across conditions (drug vs. vehicle, or compound 1 vs. compound 2 vs. vehicle). Bar charts or small-multiples with effect-size indicators. This is the figure the study director exports for the go/no-go meeting.
6. Export
CSV containing the user's named clusters, per-condition aggregations, and any active filters — encoding the interpretive state, not raw frame coordinates. Plus PNG/SVG export of the comparison figures, sized for slides.
7. Persistent view state
Cluster names and active filters save per study. Shareable URL encodes the current view so the study director can send a link to the program lead and they land on the same screen.