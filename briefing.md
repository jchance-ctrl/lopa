# BRONCHOSCOPY 3D SIMULATOR — PROJECT BRIEFING
## Paste this entire document at the start of a new conversation

---

## WHAT WE ARE BUILDING

An interactive 3D bronchoscopy flight simulator delivered as a single self-contained HTML file.
Target users: ICU nurses, respiratory therapists, and physicians learning bedside bronchoscopy.
The experience simulates the endoscopic point-of-view flying through the airway in real time.
No installation, no login — open the HTML file and it works immediately in any browser.

---

## REPOSITORY & DEPLOYMENT MODEL

This project lives in a GitHub repository as **source-of-truth and version history only**.
The repo is **not** the deploy target. Hosting happens on a separate host that provides the public URL embedded into Articulate Rise 360 as a Web Object.

Repo layout:

```
.
├── bronchsim.html              # The entire app — single self-contained file
├── README.md
├── CHANGELOG.md
├── docs/
│   ├── briefing.md             # This document
│   └── anatomy-reference.md    # (optional) clinical notes
└── .gitignore
```

Workflow:

1. Edit `bronchsim.html` locally, test by opening in a browser (no build step).
2. Commit + push.
3. Upload the same file to the external host that serves the Rise 360 URL.
4. Update the Rise 360 Web Object URL if the host path changed.

Implication for code: **the file must stay single-file and self-contained**. Do not split into separate JS/CSS files, do not introduce a build step, do not add npm dependencies. The repo move changes nothing about the code itself.

---

## TECHNOLOGY STACK (locked — do not change)

- **Three.js r128** from cdnjs: `https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js`
- Single HTML file, no build tools, no npm, no external fonts
- Must work offline after first load (except Three.js CDN)
- Target browsers: Chrome, Edge, Safari, Firefox (desktop + mobile)

---

## THREE.JS r128 CRITICAL GOTCHAS (will cause runtime errors if violated)

```javascript
// ❌ BROKEN — position attribute is READ-ONLY after TubeGeometry/CylinderGeometry:
geo.attributes.position.getX(i)   // TypeError
geo.attributes.position.getZ(i)   // TypeError
geo.translate(0, 0.25, 0)         // TypeError on frozen geometry

// ✅ CORRECT — read raw Float32Array directly:
var arr = geo.attributes.position.array;
var x = arr[i*3], y = arr[i*3+1], z = arr[i*3+2];

// ❌ BROKEN — lerpColors added in r129, not in r128:
new THREE.Color().lerpColors(c0, c1, t)  // TypeError

// ✅ CORRECT — manual lerp:
var r = c0r + (c1r - c0r) * t;

// ❌ BROKEN — geometry.translate() on BufferGeometry after creation:
new THREE.ConeGeometry(0.1, 0.5).translate(0, 0.2, 0)  // TypeError

// ✅ CORRECT — move the mesh, not the geometry:
mesh.position.y += 0.2;
```

---

## ARCHITECTURE THAT WORKS (learned through iteration)

### The only approach that avoids wall collisions:

Use **CylinderGeometry segments between waypoints**, NOT TubeGeometry with CatmullRomCurve3 branches.

```javascript
// WAYPOINTS define camera stops and tube radii:
var PATH_R = [
  {key:'ett',      pos:[0.00,  0.00,  0.00], r:1.30},
  {key:'trachea',  pos:[0.02,  0.00, -5.50], r:1.28},
  {key:'carina',   pos:[0.05, -0.05,-11.00], r:1.20},
  {key:'rmain',    pos:[1.10, -0.10,-14.00], r:1.02},
  // etc.
];

// BUILD: one CylinderGeometry per consecutive waypoint pair:
function addSegment(wp0, wp1){
  var p0 = new THREE.Vector3(...wp0.pos);
  var p1 = new THREE.Vector3(...wp1.pos);
  var dir = new THREE.Vector3().subVectors(p1, p0);
  var len = dir.length();
  var geo = new THREE.CylinderGeometry(wp1.r, wp0.r, len, 24, 12, true);
  // ... vertex colours ...
  var mesh = new THREE.Mesh(geo, mat);
  mesh.position.addVectors(p0, p1).multiplyScalar(0.5);
  mesh.quaternion.setFromUnitVectors(new THREE.Vector3(0,1,0), dir.normalize());
  scene.add(mesh);
}
```

**Why this works:** Each segment's start = previous segment's end. Mathematically impossible to have gaps or overlaps between connected segments.

### Collision prevention — always verify with this Python check:

```python
import math
trachea_r = 1.30
# All branch waypoints must be at z < -11.0 (trachea ends at z=-11)
# OR their lateral distance minus tube radius must exceed trachea_r
for wp in branch_waypoints:
    x, y, z = wp['pos']
    lat = math.sqrt(x*x + y*y)
    inner_edge = lat - wp['r']
    assert z < -11.0 or inner_edge > trachea_r, f"COLLISION: {wp['key']}"
```

---

## WHAT DOES NOT WORK (do not try these again)

| Approach | Problem |
|----------|---------|
| TubeGeometry + CatmullRomCurve3 for branches | Branches all fan out from a shared start point, walls clip through each other |
| Separate torus ring geometry on tube walls | Z-fighting with BackSide tube → bright orange glowing rings |
| ConeGeometry for carina ridge | Appears as floating dark hexagon blob blocking view |
| Warm fill PointLight on camera | Creates orange cast on all geometry, makes rings glow |
| emissiveIntensity > 0.05 on any mesh | Self-illuminates geometry independently of scope light |
| 2D canvas overlay replacing 3D view | Breaks immersion, becomes a slideshow |
| Multiple intro/end/connect screens | User wants ONE seamless 3D experience, no scene transitions |
| Branch selection UI | Learner should just press advance and the scope goes forward |
| geometry.translate() after creation | Crashes in r128 |
| lerpColors() | Doesn't exist in r128 |
| posArr.getX/getZ | Read-only in r128, use .array directly |
| Floating carina ridge mesh | Always looks like a blob or collides with tube walls |
| Splitting into multiple JS/CSS files | Breaks the single-file constraint; the deploy target is one .html |
| Adding a build step (Vite, webpack, etc.) | Same — the deployable must be hand-editable HTML |

---

## LIGHTING (locked values — do not exceed)

```javascript
// Single spotlight — scope tip. NO fill light, NO warm light.
var spot = new THREE.SpotLight(
  0xffffff,      // pure white — real endoscope LED
  4.0,           // intensity — MAX 4.5 to avoid ring glow
  18,            // distance
  Math.PI*0.19,  // angle — narrow cone
  0.60,          // penumbra — must be >= 0.50
  1.5            // decay
);
// Attach to camera so it always points where scope is looking:
camera.add(spot);
var tgt = new THREE.Object3D(); tgt.position.set(0,0,-9);
camera.add(tgt); spot.target = tgt;
scene.add(camera); // camera must be in scene for children to work

// Tiny ambient only — just lifts pure-black shadows:
scene.add(new THREE.AmbientLight(0x080606, 0.10)); // max 0.15

// Fog — realistic endoscope depth falloff:
scene.fog = new THREE.FogExp2(0x000000, 0.016);

// LED flicker in render loop (never exceeds 4.5):
spot.intensity = 4.0 + Math.sin(t*2.0)*0.14 + Math.sin(t*7.1)*0.05;
```

---

## MUCOSA COLOURS (anatomically correct white-light endoscopy)

```javascript
function mucosaColor(t){ // t=0 proximal, t=1 distal
  return [
    0.84 - t*0.28,  // R: pale salmon → deeper pink-red
    0.58 - t*0.28,  // G
    0.50 - t*0.24   // B
  ];
}
// Bake ring bands into vertex colours (trachea segments only):
var bandPos = (localT * 12) % 1.0;
var dark = bandPos < 0.08 ? 0.62 : 1.0;
// No separate torus ring geometry — ever.
```

---

## ANATOMICALLY CORRECT WAYPOINTS

```javascript
// RIGHT LUNG PATH (standard bronchoscopy order):
var PATH_R = [
  {key:'ett',       name:'ETT Tip',             pos:[ 0.00,  0.00,  0.00], r:1.30},
  {key:'trachea',   name:'Trachea',             pos:[ 0.02,  0.00, -5.50], r:1.28},
  {key:'carina',    name:'Main Carina',          pos:[ 0.05, -0.05,-11.00], r:1.20},
  {key:'rmain',     name:'R. Main Bronchus',     pos:[ 1.10, -0.10,-14.00], r:1.02},
  {key:'rul',       name:'R. Upper Lobe',        pos:[ 1.40,  0.70,-16.50], r:0.68},
  {key:'bronchint', name:'Bronchus Intermedius', pos:[ 1.55, -0.30,-18.00], r:0.88},
  {key:'rmiddle',   name:'R. Middle Lobe',       pos:[ 1.10, -0.55,-20.00], r:0.58},
  {key:'rlower',    name:'R. Lower Lobe',        pos:[ 1.30, -1.20,-22.50], r:0.76},
];
// LEFT LUNG PATH (branches from carina):
var PATH_L = [
  {key:'carina',  name:'Main Carina',            pos:[ 0.05, -0.05,-11.00], r:1.20},
  {key:'lmain',   name:'L. Main Bronchus',        pos:[-1.20, -0.10,-15.00], r:0.90},
  {key:'lul',     name:'L. Upper / Lingula',      pos:[-1.50,  0.65,-18.50], r:0.66},
  {key:'llower',  name:'L. Lower Lobe',           pos:[-1.40, -1.10,-22.00], r:0.74},
];
```

---

## ANATOMY REFERENCE (clinical accuracy)

- **Trachea**: 10–14cm, 18–24 C-shaped cartilaginous rings, posterior membranous wall, diameter ~2.5cm
- **Carina**: Sharp keel-shaped ridge at T5. Sharp = normal. Splayed/widened = subcarinal mass/lymphadenopathy
- **Right main bronchus**: 25–30° from midline, ~2cm to RUL orifice, WIDER than left
- **Left main bronchus**: 45° from midline, ~5cm before bifurcation, LONGER and narrower
- **RUL orifice**: visible at ~2cm from carina on right, branches into B1(apical), B2(posterior), B3(anterior)
- **Bronchus intermedius**: right continuation past RUL, RML orifice at 3 o'clock (medial wall)
- **RLL superior basal (B6)**: 3–6 o'clock when scope in anterior/posterior position
- **LLL superior basal (B6)**: 6–9 o'clock
- **Aortic pulsation**: visible on anterolateral wall of left main bronchus at 9–12 o'clock
- **Endoscope light**: white LED, warm, circular beam. Mucosa looks PINK-SALMON under it, NOT orange/brown

---

## NAVIGATION DESIGN (what the user wants)

```
W / ↑        Advance to next location
S / ↓        Retract to previous location  
L            Switch to left lung path (when at carina)
Space        Suction
Mobile       On-screen d-pad buttons (↑ ↓ ← →)
```

- **No branch selection UI** — W always advances forward
- **No intro screen** — 3D starts immediately on page load
- **No end screen** — experience loops or just stops at final segment
- **No scene transitions** — one continuous 3D render loop always running
- Camera smoothly lerps between waypoints (lerp factor ~0.06–0.08)
- Breathing sway: `Math.sin(t*0.22)*0.012` on X, `Math.sin(t*0.17)*0.008` on Y

---

## MOBILE REQUIREMENTS

```html
<!-- D-pad overlay — shows only on mobile via CSS media query -->
<div id="dpad">
  <button onclick="NAV.fwd()">↑</button>
  <button onclick="NAV.back()">↓</button>
  <button onclick="NAV.suction()">○ Suction</button>
</div>
<style>
#dpad { display:none; }
@media(max-width:680px){ #dpad { display:grid; } }
</style>
```

- Touch events: use `touchstart` with `{passive:false}` and `e.preventDefault()`
- Viewport: `<meta name="viewport" content="width=device-width, initial-scale=1.0">`
- Renderer pixel ratio: `Math.min(window.devicePixelRatio, 2)` — cap at 2 for performance
- FOV: 72° works on both desktop and mobile
- HUD: bottom strip only, large tap targets (min 44px height)

---

## HUD DESIGN (minimal — content is the 3D view)

Keep UI to absolute minimum:
- **Top-left**: current location name badge only
- **Top-right**: small airway map (150×114px) showing position
- **Bottom strip**: SpO₂, HR, location name, Advance/Retract/Suction buttons
- **No panels, no checklists, no info boxes, no intro/end screens**
- All backgrounds: `rgba(0,0,0,0.5)` with `backdrop-filter:blur(6px)`
- Text colours: `#d0e4f0` (readable), `#4a6070` (dim labels)

---

## FILE DELIVERY FORMAT

Single self-contained `bronchsim.html` file at the repo root:
- All CSS in `<style>` block
- All JS in `<script>` blocks
- Only external dependency: Three.js r128 from cdnjs
- Target file size: 15–25KB (not bloated with scaffolding)
- Compatible with Articulate Rise 360 Web Object (hosted URL required for Rise)

The repo stores the source. The deployed copy lives on an external host and is what Rise 360 points to. They must stay in sync — update both when shipping a change, and note meaningful changes in `CHANGELOG.md`.

---

## SOURCE DOCUMENTS AVAILABLE

The following documents were uploaded in the original session and inform the clinical content:

1. **Ambu aView User Guide** — aView monitor operation, button layout, file management
2. **Ambu aScope 3 Quick Guide** — scope setup, ET tube sizing (≥6.0 regular, ≥7.5 large), suction, BAL procedure
3. **Applied Anatomy of the Airways (Kavuru & Mehta)** — TBNA clock positions, anatomical relationships, lymph node locations, Figures 5-1 through 5-8
4. **LOPA Diagnostic/Therapeutic Bronchoscopy Protocol** — step-by-step procedure protocol, scope inspection checklist, video file transfer instructions
5. **LOPA Bronch/Line Cart Supply Checklist** — supplies by drawer, PPE requirements

---

## WHAT GOOD LOOKS LIKE

When working correctly:
- Trachea view: circular pink-salmon tunnel, subtle ring bands visible on walls, lumen dark ahead
- Carina view: camera at z≈-11, two dark circular openings visible ahead (right larger, left smaller and more oblique), pale ridge visible between them
- Right main bronchus: camera curves right, RUL orifice visible as smaller opening superiorly
- Bronchus intermedius: RML orifice visible medially at ~3 o'clock
- All segments feel like ONE continuous connected tube, not separate objects

When broken:
- Floating disconnected tubes visible (fix: waypoint-based CylinderGeometry, not TubeGeometry branches)
- Orange glowing rings (fix: remove ring geometry, bake into vertex colours, remove fill light)
- Dark polygon/blob at carina (fix: no solid mesh at junctions)
- Wall collision geometry poking through (fix: Python collision check, branches start at z < -11)
- Scene transition / slideshow (fix: remove overlays, one continuous render loop)

---

## QUICK START PROMPT FOR NEW SESSION

"I am working on a 3D bronchoscopy simulator. The project lives in a GitHub repository as source-of-truth only; the deployable is a single self-contained `bronchsim.html` at the repo root, served from a separate host that provides the URL for Articulate Rise 360. Read this full briefing before changing any code. Three.js r128, no build step, no multi-file split. Use the waypoint + CylinderGeometry architecture described and run the Python collision check before finalising coordinates."
