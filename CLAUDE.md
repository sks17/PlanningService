# PlanningService

This repo contains the plan for BioSyft's unsupervised behavioral analysis dashboard.

## Context
BioSyft provides automated behavioral analysis for animal research. Their pipeline clusters video frames by behavioral profile and projects them into 2D. We are planning (not building) the dashboard that lets users explore these clusters.

## Constraints
- 3-week delivery timeline, solo developer
- Users: PhD students (non-technical, Excel-native) and biotech study directors (technical, R/Python, PowerPoint/GraphPad)
- Data: REST API providing unsupervised coordinates, biometrics, keypoints, videos (10-20 min @ 100 FPS)
- Scale: up to ~500 videos per study → potentially 60M+ frames

## Plan Components
Each component lives in its own markdown file and gets its own PR branch:
- `scope.md` — What's in/out
- `goals.md` — Success metrics
- `tech_stack.md` — Justified technology choices
- `ui_design.md` — Wireframes and UX flow descriptions
- `timeline.md` — 3-week breakdown
- `data_architecture.md` — API schema, caching, data flow
- `user_workflows.md` — Persona-specific workflows

## Workflow
Each plan layer goes through a 3-agent adversarial process:
1. **Proposer** drafts the section on a branch
2. **Adversary** reviews and challenges it
3. **Supervisor** resolves disputes and finalizes
