# ARC Emulator — Phase 2 Handoff Brief
## Performer Path Through Cymbal Rings

**Repo:** `github.com/kanta/200-cymbals` — single file: `index.html`  
**Context:** Large-scale outdoor performance artwork. ~200 cymbal players arranged in concentric rings around a central stage. A performer needs a physical walking path from outside all rings to the stage area. This document specifies Phase 2 of the emulator: visualising and computing that path.

---

## 1. The Geometry

### Why it can't be radial
The performer carries a Sacone device (an LED-lit modified traffic cone on a stick) and faces the direction of travel. A radial path would point them directly at the stage center at every step — meaning the Sacone faces the gap and that direction has no sound. The path must be **non-radial** so the Sacone always faces somewhere with cymbal players.

### The solution: tangent path with outward tilt
The path is a **straight line** that:
1. Starts at a point on a **tangent circle** of radius `T` (where `T` is between `stageRadius` and `radii[0]`)
2. Travels **outward** at angle `α` from the tangent (i.e. α = 0° is pure tangent, α < 0° tilts outward/away from stage, making the path shorter)

At every point along this line, the walker faces **sideways or outward**, never toward stage center. The gap in each cymbal ring falls at the intersection of this line with that ring — automatically distributed, no per-row offset needed.

### Coordinate system (local, before `pathAngle` rotation)
- Stage center = origin `(0, 0)`
- Tangent touch point = `(-T, 0)` on the tangent circle
- Path direction vector = `(sin α, -cos α)` (upward = away from stage when α=0)
- For α < 0 (outward tilt): path goes upper-left, reaching outer rings sooner

So the path line in parametric form is:
```
x(t) = -T + t·sin(α)
y(t) = 0  - t·cos(α)     (note: negative because "up" in plan view is -y)
```

At `t = 0`, we're at the tangent touch point `(-T, 0)` inside ring 1.  
As `t` increases, we move outward through the rings.

### Gap computation per ring
For ring `ri` at radius `r`:

Find the intersection of the path line with the circle of radius `r`:
```
(x(t))² + (y(t))² = r²
(-T + t·sinα)² + (-t·cosα)² = r²
t²(sin²α + cos²α) - 2T·t·sinα + T² - r² = 0
t² - 2T·sinα·t + (T² - r²) = 0
```

Solving (quadratic formula):
```
discriminant = (T·sinα)² - (T² - r²) = r² - T²·cos²α
t = T·sinα ± √discriminant
```

Take the **larger** `t` (the intersection on the far side, outward along the path).

Gap center position:
```
x_gap = -T + t·sinα
y_gap =    - t·cosα
gap_angle_rad = atan2(y_gap, x_gap)   ← this is the angle in the ring coordinate system
```

Gap half-angle (how wide the gap is at this ring's radius):
```
halfAngle = arcsin(pathWidth / (2 · r))
```

A Cymbaler at index `i` in ring `ri` is **excluded** from drawing if:
```
angleDiff = normalise_to_minus_pi_pi(cymbalerAngle - (gap_angle_rad + pathAngle_rad))
abs(angleDiff) < halfAngle
```

(where `pathAngle_rad` is the global rotation of the whole path arrangement)

### Safe range for α
- `α = 0°`: pure tangent, path is longest, all gaps at 90° from radial — safest acoustically
- `α < 0°` (outward): path shorter, gap angles spread to >90° — also safe, even better
- `α > 0°` (inward, toward stage): gap angles shrink toward 0° — becomes problematic
- Hard cap: `α` must stay within **−75° to +45°** to keep gap angles >45° from radial
- Add an amber warning in the UI when `α > 20°`, red warning at `α > 40°`

### Special case: ring 1 (innermost)
When `T > stageRadius`, the path starts *inside* ring 1 but `T < radii[0]`. The intersection for ring 1 still uses the same formula. The path segment from the tangent touch point `(-T, 0)` to the ring 1 intersection is the "inner corridor" connecting to the stage area. This segment doesn't need to exclude any performers (there are none inside ring 1), but it should still be drawn as part of the corridor overlay.

---

## 2. New Controls (Sidebar Section "Path")

Add a new sidebar section **below the Heights section**, before the closing `</div>` of the sidebar. Section header key: `secPath`.

### Controls:

| Slider ID | Label key | Range | Default | Step |
|---|---|---|---|---|
| `c-path-angle` | `cPathAngle` | 0–360° | 90° | 1 |
| `c-path-T` | `cPathT` | `stageRadius` → `radii[0]` | midpoint | dynamic |
| `c-path-alpha` | `cPathAlpha` | −75° → +45° | 0° | 1 |
| `c-path-width` | `cPathWidth` | 0.5–2.0 m | 0.8 m | 0.05 |

**Note on `c-path-T`:** Its `min`/`max` depend on current layout sliders (stageR and radii[0]). Update its `min`, `max`, and clamp its `value` inside `upd()` each time those change — same pattern used elsewhere.

**Path enable toggle:** Add a checkbox or segmented toggle (`c-path-enable`) at the top of the section. When disabled, no gaps are cut and no overlay is drawn. This lets the user compare with/without path easily. Default: enabled.

### Display values:
- `pathAngle`: show in degrees (integer)
- `T`: show in current units via `fmtS()`
- `α`: show as `+N°` / `−N°` / `0°` in degrees
- `pathWidth`: show via `fmtCm()`

### Warning display:
Add a new `<div class="warn warn-yellow" id="warn-path-alpha"></div>` below the existing warn blocks. Show it when `α > 20°` with text key `warnPathAlpha`. Turn it `.warn-red` when `α > 40°`.

---

## 3. Changes to `upd()`

Read the new path parameters:
```js
const pathEnabled = document.getElementById('c-path-enable').checked;
const pathAngleDeg = val('c-path-angle');
const pathT       = val('c-path-T');
const pathAlpha   = val('c-path-alpha') * Math.PI / 180;
const pathWidth   = val('c-path-width');
```

Update `c-path-T` bounds dynamically:
```js
const pathTSlider = document.getElementById('c-path-T');
pathTSlider.min  = stageR.toFixed(2);
pathTSlider.max  = radii[0].toFixed(2);
pathTSlider.value = Math.min(Math.max(parseFloat(pathTSlider.value), stageR), radii[0]);
```

Compute `pathGapAngles[]` — one gap center angle per ring (in world coords, after pathAngle rotation):
```js
const pathGapAngles = pathEnabled ? radii.map(r => {
  const disc = r*r - pathT*pathT * Math.cos(pathAlpha)*Math.cos(pathAlpha);
  if (disc < 0) return null; // path doesn't reach this ring (shouldn't happen within valid α range)
  const t = pathT*Math.sin(pathAlpha) + Math.sqrt(disc);
  const xGap = -pathT + t*Math.sin(pathAlpha);
  const yGap =        - t*Math.cos(pathAlpha);
  const localAngle = Math.atan2(yGap, xGap);
  return localAngle + pathAngleDeg * Math.PI / 180;
}) : radii.map(()=>null);
```

Pass to `drawPlan`:
```js
drawPlan(radii, counts, stageR, cymD, pw, seated, sightClear,
         pathEnabled, pathAngleDeg, pathT, pathAlpha, pathWidth, pathGapAngles);
```

Also pass to `drawElev` if you want to show the path gap in elevation view (optional, lower priority).

---

## 4. Changes to `drawPlan()`

### Signature
```js
function drawPlan(radii, counts, stageR, cymD, pw, seated, sightClear,
                  pathEnabled, pathAngleDeg, pathT, pathAlpha, pathWidth, pathGapAngles)
```

### Cymbal/performer exclusion
Inside the inner loop `for(let i=0;i<n;i++)`, after computing `angle`:
```js
const angle = (2*Math.PI*i)/n - Math.PI/2;

// Path gap exclusion
if (pathEnabled && pathGapAngles[ri] !== null) {
  const halfAngle = Math.asin(Math.min(1, (pathWidth/2) / radii[ri]));
  let diff = angle - pathGapAngles[ri];
  // Normalise to [-π, π]
  diff = diff - Math.round(diff / (2*Math.PI)) * 2*Math.PI;
  if (Math.abs(diff) < halfAngle) continue; // skip this performer
}
```

### Path corridor overlay (draw after rings/stage, before performers)
Draw a translucent filled corridor and dashed edge lines.

The corridor in **local coordinates** (before pathAngle rotation) is:
- A straight band of width `pathWidth` centred on the path line `x = -T`... wait, no — the path line passes through `(-T, 0)` at angle `α` from vertical.
- The two edge lines are at perpendicular offset `±pathWidth/2` from the path centre line.

The path corridor goes from `t = 0` (inner end, at the tangent touch point `(-T, 0)`) to `t = t_outer` where `t_outer` is the intersection with the outermost ring plus a small margin (e.g. `outerR + cymD`).

Perpendicular direction to the path: `(cos α, sin α)` (rotate path direction 90°).

Four corners of the corridor rectangle (in local coords, then rotate by `pathAngleDeg` and map to canvas):
```
p0 = (-T - (pathWidth/2)·cosα,  0 + (pathWidth/2)·sinα)   // inner left
p1 = (-T + (pathWidth/2)·cosα,  0 - (pathWidth/2)·sinα)   // inner right
p2 = outer_right  // = p1 + t_outer·(sinα, -cosα)
p3 = outer_left   // = p0 + t_outer·(sinα, -cosα)
```

Apply the `pathAngleDeg` rotation to all four points around the origin, then convert to canvas coords via `cx + x·scale`, `cy + y·scale`.

Draw:
1. Filled corridor: `rgba(239,159,39, 0.12)` (orange tint, matches existing UI accent)
2. Dashed edge lines: `stroke: rgba(239,159,39, 0.5)`, `lineWidth: 1`, `setLineDash([5,4])`
3. At the outer end, draw a small perpendicular line with the width label in current units

### Draw the tangent circle T
Draw a faint dashed circle of radius `T` centred on stage center:
- `strokeStyle: rgba(239,159,39, 0.35)`, `lineWidth: 0.8`, `setLineDash([3,3])`
- Label it `T = {fmtS(pathT)}` near the 3 o'clock position

---

## 5. Translation Keys to Add

### English
```js
secPath: 'Path',
cPathAngle: 'Path direction',
cPathT: 'Tangent radius T',
cPathAlpha: 'Path tilt α',
cPathWidth: 'Path width',
cPathEnable: 'Show path',
warnPathAlpha: 'Warning: positive α tilts path toward stage — gap angles narrowing.',
warnPathAlphaRed: 'Danger: α too high — path gaps may face stage center (silence).',
```

### Japanese
```js
secPath: '動線',
cPathAngle: '動線方向',
cPathT: '接線半径 T',
cPathAlpha: '傾き α',
cPathWidth: '通路幅',
cPathEnable: '動線を表示',
warnPathAlpha: '注意: αが正のため、動線がステージ方向に傾いています。',
warnPathAlphaRed: '危険: αが大きすぎます — ギャップがステージ中心を向く可能性があります。',
```

---

## 6. Code Conventions to Follow

- **No library dependencies** — single `index.html`, vanilla JS, 2D canvas only
- **Persistent state pattern:** if path params need memory across layout changes, mirror the `stoolHeights[]` / `countOffsets[]` pattern (persistent array declared at top level, DOM rebuilt in `upd()`)
- **Scale clamping:** both `drawPlan` and `drawElev` already clamp `availW` / scale to a positive minimum — don't remove that
- **Dark mode:** all colors check `isDark=window.matchMedia('(prefers-color-scheme:dark)').matches` — add dark variants for any new path colors
- **Unit display:** use existing `fmt()`, `fmtS()`, `fmtCm()` helpers — they handle m↔ft switching automatically
- **`val(id)`** helper reads `parseFloat(document.getElementById(id).value)`
- **`t(key)`** helper returns `T[lang][key]` — use for all user-visible strings
- **Fuzz test** after implementing: the existing `jsdom` + fake canvas harness pattern (in prior conversation) catches negative-radius `arc()`/`ellipse()` calls that the browser throws on. Run at least 5,000 randomised `upd()` calls across all slider combinations to confirm no crashes.

---

## 7. What NOT to Change

- Do not reorganise the sidebar sections (the artist's team is sensitive to layout changes)
- Do not alter the elevation view (`drawElev`) unless specifically adding path gap display
- Do not change `drawPlan`'s existing cymbal/performer rendering logic except for the exclusion `continue` inside the loop
- The `countOffsets[]` array and `nudgeTo200()` function from Phase 1 must remain intact and working — the path exclusion reduces total count, which the user compensates with the Phase 1 controls

---

## 8. Suggested Implementation Order

1. Add HTML sidebar section with controls (static, hardcoded defaults for T range first)
2. Add translation keys
3. Read values in `upd()`, add dynamic T slider bounds update
4. Add gap angle computation in `upd()`
5. Add exclusion `continue` in `drawPlan()` loop — verify gaps appear correctly in canvas
6. Add corridor overlay drawing — verify visually
7. Add T circle drawing
8. Add α warning display
9. Fuzz test

---

## 9. Quick Geometry Sanity Checks

Given default values (stageR=2m, gap1=2m, rowSp=1.5m → radii=[4,5.5,7,8.5,10,...]):

With `T=3m, α=0°, pathWidth=0.8m, pathAngle=90°`:
- Ring 1 (r=4): `disc=4²−3²=7`, `t=0+√7≈2.65`, gap at `(-3+0, -2.65)` → angle≈−(90°+41°)` from +x` → after +90° rotation lands at the right side. Half-angle = `arcsin(0.4/4)≈5.7°`
- Ring 4 (r=8.5): `disc=8.5²−3²=63.25`, `t≈7.95`, gap at `(-3, -7.95)`, angle≈−(90°+21°)` → half-angle≈`arcsin(0.4/8.5)≈2.7°`

With `T=3m, α=−30°, pathWidth=0.8m`:
- The path tilts outward; outer rings are reached at smaller `t`, path is shorter
- Gap angles at outer rings are >90° from radial (120°+) — acoustically safer
