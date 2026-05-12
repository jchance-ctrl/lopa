# Changelog

All notable changes to `bronchsim.html` are recorded here. Newest at the top.

Format: each entry has a date, a short label, and a plain-language description of what changed and why. Reference commit SHAs where helpful.

---

## Unreleased

### 2026-05-12 — Model inspector added; pivot to GLB-based geometry
- New `inspector.html`: dev-only viewer for `assets/tracheobronchial_tree.glb`. Free-fly camera (WASD + mouse-look), scope-light/overview toggle, wireframe toggle, waypoint marking (number keys 1–9), and JS export of marked positions ready to paste into `bronchsim.html`.
- Committed `assets/tracheobronchial_tree.glb` (Sketchfab export, 122k verts / 175k tris, single white material).
- README updated with local-serve instructions (`python3 -m http.server`) since browsers block `file://` GLB loading.
- `bronchsim.html` not yet updated to use the GLB; waypoints in the model's coordinate space need to be recorded via the inspector first.

### 2026-05-12 — Initial buildable version
- First `bronchsim.html`. Trachea + carina + right lung path only (8 waypoints).
- Waypoint-based `CylinderGeometry` segments between consecutive waypoints (no `TubeGeometry`, no Catmull splines).
- Single white spotlight on camera, intensity 4.0 base with mild flicker, narrow cone, penumbra 0.60. No fill light, no warm light.
- Vertex-coloured mucosa gradient (pale salmon → pink-red). Ring banding baked into trachea segment vertex colours — no separate ring geometry.
- Fog `FogExp2` density 0.016, ambient `0x080606` at 0.10.
- Navigation: W/↑ advance, S/↓ retract, Space suction. On-screen Advance / Retract / Suction buttons. No branch selection UI; left-lung path not wired up yet.
- HUD: top-left location badge, top-right airway schematic with position dot, bottom strip with SpO₂ / HR / nav buttons.
- Suction triggers a brief SpO₂ dip as placeholder feedback.
- Python collision check passes for both PATH_R and PATH_L waypoints.

### Repo
- Repo initialised. Source-of-truth setup; deploy host kept separate.
