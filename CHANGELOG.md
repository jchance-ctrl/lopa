# Changelog

All notable changes to `bronchsim.html` are recorded here. Newest at the top.

Format: each entry has a date, a short label, and a plain-language description of what changed and why. Reference commit SHAs where helpful.

---

## Unreleased

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
