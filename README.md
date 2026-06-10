# Archery Simulator

Browser-based target archery simulator. Single-file, no build step — open `index.html` directly.

---

## Usage

1. Open `index.html` in any modern browser.
2. Select a **distance** from the dropdown (10 m – 70 m).
3. Set **bow poundage** — arrow mass and speed are auto-calculated and displayed.
4. Choose **arrow spine** offset (−2 to +2). Spine 0 = optimal for your poundage.
5. Set the **sight** (tick-based, like a real sight): the elevation/windage sliders move the sight pin in 0.05° clicks. Moving the sight **down** forces you to raise the bow to re-center the pin, so the arrow hits **higher** — use the **Ref:** value as the starting elevation setting. Uncheck **Use sight** to shoot instinctive/gap style (no pin, no offsets — just a faint focus dot).
6. Aim with the **arrow keys** — the cursor starts at the bottom of the view each time, so you raise the bow onto the target for every arrow. With **Bow droop & sway** on (the default), elevation constantly droops and windage sways randomly, so you must actively hold the position. Press **Space** to shoot.
7. Optionally enable **wind** — a random wind is generated; click "New" to reroll it.
8. Press **SHOOT** (or **Space**). The arrow lands according to physics + scatter. Score appears as a popup and is added to the running total.
9. **Clear Arrows** resets the group while keeping settings.
10. **View toggles**: *Flight side view* shows a bottom panel where each shot's arc is animated from archer to target. *Path preview* shows the predicted trajectory (dashed arc in the side view) and predicted impact point (cyan dashed circle on the target) — deterministic physics only, scatter excluded.

### Reading the UI

| Element | Meaning |
|---|---|
| **Ref:** label under sight elevation | How many ticks **down** the sight needs to hit dead-center at current distance/poundage (pin centered on the gold). Use this as your starting point. |
| **Drop @0°** in the stat strip | How far the arrow falls if launched perfectly horizontal — useful to understand why elevation is needed. |
| **Arrow mass / speed / optimal spine** | Read-only info derived from poundage. |
| Green ring (sight pin) | Your aiming direction. Center it on the gold to aim for X/10. Where the arrow actually lands depends on physics + the sight offset. With *Use sight* off it becomes a faint dot (instinctive aiming) and no sight offset applies. |
| Wind banner (top-right) | Active wind speed and compass direction. Compensate with the sight windage. |
| Cyan dashed circle on target | The sight's **zero point**: where the arrow lands if the pin is held dead-center on the gold (physics + wind + spine, **without** human scatter). It depends only on the sight setting, not the cursor — dial the sight until it sits in the middle. Toggle with *Path preview*. |
| Bottom panel (side view) | Side-on view of the range: archer left, target butt right, ground line, 10 m gridlines. Dashed green arc = predicted path (preview only); orange arc = arrow in flight / last shot trace (always shown, even with preview off). Toggle with *Flight side view*. |
| Wind sock (side view, mid-range) | Appears when wind is enabled. Droops in light wind, stands horizontal at 15+ mph; points toward the target for tailwinds, toward the archer for headwinds, and foreshortens for crosswinds. |

---

## Implementation

### File structure

Everything lives in a single file: `index.html`

```
index.html
  <style>   — all CSS (dark theme, flexbox layout)
  <body>    — HTML structure (controls panel + canvas area)
  <script>  — constants, physics, rendering, state, event listeners
```

### Physics

#### Arrow speed

Linear model calibrated to typical recurve performance:

```
speed_fps = 100 + poundage * 2.2
```

Examples: 40 lb → 188 fps, 60 lb → 232 fps, 70 lb → 254 fps.

#### Arrow mass

Auto-matched at 8.5 grains per pound of draw weight (keeps AFO ≈ 8–9 gr/lb):

```
grains = poundage * 8.5
kg     = grains * 0.00006480
```

Arrow mass is displayed as info only — it does not currently feed back into the speed formula (so poundage is the sole speed driver). This is a known simplification; see **Future work**.

#### Trajectory

Standard 2D projectile motion. `th` = elevation angle in radians, `ps` = windage angle:

```
t      = D / (v * cos(th))               // flight time
y_hit  = v * sin(th) * t  -  0.5 * g * t²   // vertical offset at target (+ = up)
x_hit  = tan(ps) * D                     // horizontal offset (+ = right)
```

#### Sight model

The cursor (`S.elev` / `S.windage`) is the raw aiming direction — where the eye/pin points. The sight stores an offset in **ticks** of `TICK_DEG = 0.05°` each. Launch angles are:

```
launch_elev = aim_elev − sightElevTicks * 0.05°
launch_wind = aim_wind − sightWindTicks * 0.05°
```

Positive ticks = pin moved up/right. Moving the pin **down** (negative ticks) means the bow must come **up** to re-center the pin, so the arrow launches higher — exactly like a real sight. With *Use sight* off, launch = aim (instinctive/gap aiming).

#### Zero elevation reference

The elevation angle that puts the arrow at vertical center (small-angle approximation):

```
θ_zero ≈ atan2(g * D,  2 * v²)
```

This is shown as "Ref:" under the sight elevation slider, converted to ticks down (`round(θ_zero / 0.05°)`).

#### Wind drift

Empirical model (scaled to match typical outdoor archery observations: ~12–15 cm drift per 10 mph crosswind at 70 m):

```
wind_ms  = wind_mph * 0.44704
wx (lateral) = wind_ms * sin(dir_rad) * D * 0.00018
wy (fore/aft) = wind_ms * cos(dir_rad) * D * 0.00004
```

Wind direction convention: `dir = 0°` = tailwind (pushes arrow forward/up slightly), `dir = 90°` = right crosswind.

#### Arrow spine offset

Spine error is an integer offset (−2 = very underspined, 0 = optimal, +2 = very overspined). Underspined arrows drift left for a right-handed archer; overspined drift right. The offset scales with distance:

```
sx = spineError * 0.003 * dist_m / 18
```

#### Human scatter

Gaussian noise applied after all systematic offsets. Standard deviation scales mildly with distance and worsens with spine mismatch:

```
std = (0.008 + dist_m * 0.0001) * (1 + |spineError| * 0.45)
```

At 18 m optimal spine: std ≈ 10 mm. At 70 m: std ≈ 15 mm.

Box-Muller transform used for Gaussian sampling (`gauss()` function).

### Target rendering

Two stacked 500 × 500 canvases share the same `#canvas-wrap` container:

- **`#target-canvas`** — target face and arrow holes (transparent background — the green range gradient of `#target-area` shows through, no visible box).
- **`#peep-canvas`** — aiming cursor overlay (sight ring or instinctive dot). `pointer-events: none` so it doesn't block clicks.

#### Target scaling (perspective)

Angular size = physical size / distance. A focal constant converts this to pixels:

```
apparent_radius_px = (face_radius_m / dist_m) * FOCAL
```

`FOCAL = 8000` makes the 500 px canvas span ±~1.7° of aim — wide enough that short-range gap aiming (hold-over) stays on screen, while the 18 m / 40 cm indoor target still fills a good third of the view. The same constant is used at all distances, so the 70 m Olympic target (122 cm face) appears *smaller* than the 18 m target despite being physically larger — which matches reality. The aim cursor is clamped to this span (`aimLimDeg()`), so it is always visible.

#### Ring boundaries

Rings are defined as fractions of the face radius (World Archery proportions):

| Score | Outer fraction |
|---|---|
| X (inner gold) | 0.04 |
| 10 | 0.08 |
| 9 | 0.16 |
| 8 | 0.24 |
| 7 | 0.32 |
| 6 | 0.40 |
| 5 | 0.52 |
| 4 | 0.64 |
| 3 | 0.76 |
| 2 | 0.88 |
| 1 | 1.00 |

Scoring is computed by `dist_from_center / face_radius` and compared against these fractions.

#### Aiming cursor overlay

The green scope aperture ring (or, with the sight disabled, a faint focus dot) is positioned at:

```
rx = CX + tan(windage_rad) * dist_m * px_per_m
ry = CY - tan(elev_rad)    * dist_m * px_per_m   // inverted Y axis
```

This represents the *raw aiming direction* projected to the target plane — not the predicted landing point. The gap between the cursor and where arrows land is closed by adjusting the sight (or learned, when shooting instinctive). There is no peep vignette; the cursor is never hidden.

### Flight side view & path preview

A third canvas (`#flight-canvas`, full-width × 150 px) sits in `#flight-panel` below `<main>`. It draws a side-on view of the range: ground line, 10 m gridlines, stick archer (left), target butt with gold band (right). Launch point and target center are both at `LAUNCH_H = 1.5 m` above ground.

**Trajectory sampling** — `previewPts(elev)` samples the deterministic (no-scatter) arc at 100 points:

```
y(x) = LAUNCH_H + x·tan(θ) − g·x² / (2·v²·cos²θ)
```

**Shot animation** — on SHOOT (when enabled), the scored hit is computed immediately but committed only when the animated arrow lands. The drawn arc is the deterministic path bent linearly so it terminates at the actual hit height:

```
y_anim(x) = y_det(x) + (y_hit − y_det(D)) · x / D
```

Animation duration is `400 + t·800` ms (real flight time slowed for visibility). Shooting again mid-flight lands the previous arrow instantly. With the side view disabled, shots commit instantly as before.

**Path preview** — one toggle controls two cues: the dashed green arc in the side view, and a cyan dashed circle on the target face marking the sight's zero point (`computeHit(zeroLaunchElev(), zeroLaunchWind(), /*noise=*/false)` — the launch angles with the pin held dead-center; includes wind drift and spine offset, excludes Gaussian scatter). Both depend only on the sight setting, not on the momentary cursor position, so the sight can be dialed in without holding an aim. The *flown* path (in-flight orange arc and last-shot trace) is independent of this toggle and always draws while the side view is enabled.

**Wind sock** — drawn at mid-range on a 2 m pole when wind is active. Droop angle interpolates from hanging (~69° below horizontal at 0 mph) to fully horizontal at 15 mph. The sock points toward the target when the fore/aft wind component is a tailwind (`cos(dir) ≥ 0`) and toward the archer for headwinds; its length is foreshortened by `|cos(dir)|` (floored at 0.3) so a pure crosswind — blowing into or out of the screen — shows as a short sock, matching the side-on perspective.

The vertical scale of the side view auto-fits the tallest visible arc (preview, in-flight, or last shot), so steep lobs at 70 m never clip. Arcs that dip below ground level stop drawing at the ground line.

### Aiming

A `requestAnimationFrame` loop (`aimLoop`) runs always: arrow keys move the aim cursor (**↑/↓** 0.9 °/s, **←/→** 0.7 °/s), and both axes clamp to `aimLimDeg()` so the cursor never leaves the canvas. **Space** shoots (`e.repeat` guarded, so holding Space doesn't machine-gun).

When the arrow reaches the target (i.e. when the shot commits — instantly with the side view off, on landing with the flight animation on), `startLowering()` holds the aim steady for `AIM.followThru` = 0.7 s (follow-through), then glides the cursor down to the bottom edge of the view (`AIM.lowerRate` = 4 °/s, ease-out) with a small random windage offset — the bow visibly comes down, and you must raise it and settle the aim for each arrow, like a real shot cycle. Pressing any arrow key mid-lowering cancels the glide and resumes manual control. On init and distance change, `resetAim()` does the same instantly.

When **Bow droop & sway** is enabled (default), the loop adds hold instability:

- **Elevation droop** — drops continuously at `0.05 + poundage × 0.002` °/s (40 lb ≈ 0.13 °/s, 70 lb ≈ 0.19 °/s), simulating holding weight at full draw.
- **Windage sway** — a damped random walk on sway *velocity*: each frame `swayV += rand(±swayAccel·dt)`, damped at 0.5 /s and capped at ±0.3 °/s, then `windage += swayV·dt`.

All tuning constants live in the `AIM` object. Arrow keys and Space are `preventDefault`-ed at document level (except when a `<select>` or `<input>` has focus, so the sight sliders stay keyboard-accessible) so they never scroll the page. Because 1° of aim error maps to a constant `FOCAL × tan(1°) ≈ 140 px` on screen at any distance, droop/sway feel equally fast at 18 m and 70 m.

### State object

All mutable state lives in a single `S` object:

```js
S = {
  dist,        // number — current distance in metres
  poundage,    // number — draw weight in lbs
  spineErr,    // integer − -2..+2 — spine error offset
  elev,        // number — raw aim (cursor) elevation in degrees
  windage,     // number — raw aim (cursor) windage in degrees
  sightOn,     // boolean — sight enabled (off = instinctive/gap aiming)
  sightElev,   // integer ticks — + pin up (hits lower), − pin down (hits higher)
  sightWind,   // integer ticks — + pin right (hits left)
  windOn,      // boolean
  wind: { mph, dir },  // active wind (dir in degrees)
  arrows: [],  // array of { x, y, dist, score } — all shots ever taken
  total,       // running score total
  flightOn,    // boolean — flight side view panel + shot animation
  previewOn,   // boolean — predicted path arc + impact marker
  dynamicOn,   // boolean — bow droop + sway while aiming
}
```

Two module-level variables track the side-view animation outside `S`: `flightAnim` (the shot currently in the air) and `lastFlight` (the most recent completed animated shot, drawn faded; cleared on distance change / reset).

Arrows from a different distance are stored but not rendered on the current target view (filtered by `a.dist !== S.dist`).

### Key constants

| Constant | Value | Purpose |
|---|---|---|
| `FOCAL` | 8000 px | Visual zoom/scale of target (sets ±~1.7° aim span) |
| `TICK_DEG` | 0.05° | One sight tick (click) |
| `G` | 9.81 m/s² | Gravity |
| `SZ` | 500 px | Canvas size |
| Wind lateral coefficient | 0.00018 | Empirical drift scale |
| Wind fore/aft coefficient | 0.00004 | Empirical headwind/tailwind scale |
| Spine offset per error level | 0.003 m / 18 m | Lateral offset per spine step at 18 m |
| `LAUNCH_H` | 1.5 m | Launch / target-center height above ground in side view |
| Flight animation duration | 400 + t·800 ms | Slowed real flight time t for visibility |
| `AIM.droopBase` / `droopPerLb` | 0.05 / 0.002 °/s | Elevation droop rate (base + per lb) |
| `AIM.elevRate` / `windRate` | 0.9 / 0.7 °/s | Key correction speeds |
| `AIM.swayAccel` / `swayDamp` / `swayMax` | 0.9 °/s² / 0.5 /s / 0.3 °/s | Windage random-sway dynamics |

---

## Future work

- **Arrow mass → speed coupling**: feed the auto-calculated kg value into a kinetic-energy speed formula so heavier arrows are demonstrably slower.
- **Manual arrow mass override**: allow the user to set grains independently of poundage.
- **Bow type selector**: recurve vs compound (compound is faster/flatter, different efficiency curve).
- **Arrow length selector**: affects dynamic spine and thus the spine matching chart.
- **Archer's paradox oscillation**: add shaft flex oscillation to the flight side view (parabolic side view is done).
- **Session persistence**: save arrow groups to `localStorage` so they survive page refresh.
- **End-based scoring**: track 6-arrow ends, show per-end totals and an end history.
- **Target face variants**: 3-ring indoor face, 5-ring outdoor face, field archery faces.
- **Draw length variation**: affects stored energy; currently fixed at 28".
- **Air resistance**: simplified drag would lower effective speed at longer distances.
