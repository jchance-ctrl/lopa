# bronchsim

Interactive 3D bronchoscopy flight simulator — single self-contained HTML file, Three.js r128, no build step.

Target users: ICU nurses, respiratory therapists, and physicians learning bedside bronchoscopy.
Delivery format: one `.html` file, opened directly in any modern browser, embedded into Articulate Rise 360 via Web Object (hosted URL).

## Repo purpose

This repository is **source-of-truth and version history only**. It is not a deploy target.
The deployable artifact is `bronchsim.html` — a single self-contained file. Hosting happens on a separate host (Articulate Rise 360 needs a public URL pointing to the file).

## Deploy workflow

1. Edit `bronchsim.html` locally, test in a browser.
2. Commit and push to this repo.
3. Upload `bronchsim.html` to the chosen host (wherever generates the Rise 360 URL).
4. Update the Web Object URL in Rise 360 if the host path changed.

The host is intentionally separate from the repo — GitHub Pages is not used here.

## Structure

```
.
├── bronchsim.html              # The entire app — single file, self-contained
├── README.md                   # This file
├── CHANGELOG.md                # Human-readable record of meaningful changes
├── docs/
│   ├── briefing.md             # Full project briefing — paste at start of every new AI session
│   └── anatomy-reference.md    # Clinical anatomy notes (optional, for content edits)
└── .gitignore
```

## Constraints (locked)

- Three.js r128 from cdnjs — no other dependencies
- Single HTML file, no build tools, no npm
- Target file size 15–25 KB
- Works offline after first load (except Three.js CDN fetch)
- Chrome / Edge / Safari / Firefox, desktop + mobile

See `docs/briefing.md` for the full architectural rules, including the r128 gotchas, waypoint geometry approach, and lighting values. Any AI session working on this code must read that briefing first.

## Quick start for a new AI session

> I am working on a 3D bronchoscopy simulator. The full project briefing is at `docs/briefing.md` — read it before changing any code. The current build is `bronchsim.html` (single self-contained file, Three.js r128). The repo is source-only; deploys go to a separate host.
