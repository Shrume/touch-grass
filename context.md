# Touch Grass — project context

Barley-like tall grass field: cursor interaction, travelling wind gusts, and ASMR audio. Two implementations share the same visual goal; **WebGL is the active build**.

## Files

| File | Role |
|------|------|
| `touch_grass.html` | Canvas 2D reference — three-angle blade physics, awns, full `P` tuning. **Do not modify** unless explicitly migrating features. |
| `touch_grass_webgl.html` | Active WebGL prototype — Three.js r169 (esm.sh), 150k instanced blades, gust state machine, directional trail, spring recovery, audio. |
| `.cursor/rules/grass.mdc` | Cursor rules for both HTML builds. |

Serve locally: `npx serve .` → open `touch_grass_webgl.html`.

---

## Canvas 2D reference (`touch_grass.html`)

- **Physics:** `Blade.update()` — three-angle springs (`baseAngle`, `tipAngle`, `awnAngle`).
- **Rendering:** `Blade.draw()` — cubic S-curve, 3-stop gradient, awn strokes at tip.
- **Population:** `mkBlades()` with bottom-biased tufts; density ~2600 blades (Canvas ceiling ~4000).
- **Defaults:** density 2600, height 1.4, width 0.7, spread 0.045, wind/mow params in `P` object.

Not yet ported to WebGL: per-blade spring physics, awn geometry, mow mode.

---

## WebGL build (`touch_grass_webgl.html`)

### Stack

- Three.js **0.169** via importmap (`three` from esm.sh).
- Single module script: scene, shaders, gust CPU, trail ping-pong RT, Web Audio.
- **150,000** instanced blades; `PlaneGeometry(1,1,1,6)` per blade, base at Y=0.
- `setPixelRatio(Math.min(devicePixelRatio, 1.5))`.
- Clear color `0x040a01`; ground `0x0a1503`.

### Field placement

On init, camera is set to **final** pose; frustum corners are raycast to **Y=0**; bounds get **25% margin**:

- `fieldMinX`, `fieldMaxX`, `fieldMinZ`, `fieldMaxZ` (logged to console).
- Blades: uniform random in that box (no Z power-bias or distance culling).
- Ground plane sized/positioned to match field bounds.
- Trail/grass UVs: `(world - fieldMin) / (fieldMax - fieldMin)`.

### Blade attributes (per instance)

- `aHeight` 0.4–1.2, `aWidth` ~0.018–0.035, `aLean` **−0.25 to +0.45**, `aRandom` (±10/255 color variation).
- Placement: `rotation.y = 0` (world-consistent wind/gust bow).

### Grass vertex shader — displacement order

1. Scale width/height, S-curve lean in local X.
2. **Trail displacement** (local space, before instance matrix): sample `uTrail` at field UV.
   - **R/G:** encoded stroke direction (0.5 = neutral).
   - **B:** intensity.
   - `push = trailIntensity * pow(uv.y,2) * uTrailStrengthX * (1.2 + uCursorSpeed * 1.8) * (1.0 + uTrailSpring * 0.8)`.
   - `transformed += trailDir * push`.
3. **Gust band** (world space): `inGust = lead * trailEdge * uGustBlend` from `uGustFrontX`, `uGustWidth`, smoothed `uGustDirX/Z`.
4. **Gust lean** (gust-aligned): `baseDisp` from sin×sin patches × `uGustStrength` (spring output); small `turbDisp` (15% of strength); `crossDisp` for subtle cross-wind.
5. **Ambient wind** (fixed direction, independent of gust): `uAmbientStrength` (0.18), `uAmbientDirX/Z` (0.92, 0.38) — patch lean + slow temporal breeze + tip sway (`ambientX`). Applied in **world space along ambient axis**, not rotated with gust direction (prevents direction snap).
6. Tip weighting: `windStrength = pow(uv.y, 2.0)` on gust and ambient magnitudes.
7. **`vCursorBend`** (for fragment color only): `length(trailDir * push) / (aHeight * 0.28)` — per-vertex, tip-weighted via `push`; wind/gust do not contribute.

**Key grass uniforms:** `uTime`, `uGustFrontX`, `uGustStrength` (from spring), `uGustWidth` (smoothed), `uGustDirX/Z` (smoothed), `uGustBlend`, `uAmbientStrength`, `uAmbientDirX/Z`, `uTrailSpring`, `uTrail`, `uCursorSpeed`, `uTrailStrengthX/Z`.

Old global sine-wave wind was **removed**; WIND panel uniforms remain bound but are not used by the grass shader.

**Anti-banding / Moiré mitigations:** irrational spatial frequencies; per-blade `rPhase` and `freqVar` from `aRandom`/`aHeight`; gust-aligned `scroll`/`cross` coordinates; sin×sin pocket patterns instead of single-axis stripes.

### Grass fragment shader — colors

Procedural barley gradient (no texture):

| Stop | RGB | Hex (approx) | Blend region (`vUv.y`) |
|------|-----|--------------|-------------------------|
| Base | (28, 58, 18) | `#1C3A12` | 0% + `randVar` from `aRandom` |
| Mid | (52, 105, 32) | `#346920` | smoothstep 0.35 → 0.50 |
| Tip pale | (195, 215, 155) | `#C3D79B` | smoothstep 0.70 → 0.85 |
| Tip bright | (225, 232, 190) | `#E1E8BE` | mixed into tip 0.70 → 1.0 |

Post-processing on `col`:

- **AO:** `col *= (0.25 + 0.75 * smoothstep(0, 0.25, vUv.y))`.
- **Directional light:** sun `(0.8, 0.6)` vs `vWorldNormal` (from `aLean`); `light = 0.6 + 0.4 * sunDot`.
- **Tip specular:** above 0.85 height; warm add `vec3(0.9, 1.0, 0.7) * specular`, strength up to 0.4.

**Cursor bend → darker tips (cursor only, not wind/gust):**

| Color | RGB | Role |
|-------|-----|------|
| `tipDarkGreen` | (38, 72, 22) | Bent tip lower |
| `tipDarkGreenHi` | (52, 98, 32) | Bent tip upper |

- `cursorBendAmt = pow(clamp(vCursorBend, 0, 1), 1.4)` — tracks **push magnitude**, not rawMin raw trail B.
- Unbent: `tipPale → tipBright`. Bent: mix toward `tipDarkGreen → tipDarkGreenHi`.
- Tip-zone only: extra 18% darken + specular suppressed when bent.
- Color recovery follows push + trail spring (same signal as bend animation); fully normal when `push → 0`.

Output: `gl_FragColor = vec4(col, 1.0)` (opaque).

### Gust system (CPU)

State machine: `CALM` → `BUILDING` → `ACTIVE` → `FADING` → `CALM`.

**Spawn params (`spawnGust`):**

| Param | Range |
|-------|--------|
| `speed` | 6–18 units/s |
| `width` | 180–400 |
| `targetStrength` | 0.3–1.0 |
| `buildDuration` | 1.5–4 s |
| `fadeDuration` | 2–5 s (stored; not used for visual fade — see FADING) |
| `calmTimer` (between gusts) | 0.5–3 s |
| Initial calm | 0.5–1.5 s |

- Spawns off-field: `frontX = fieldMinX - 20` or `fieldMaxX + 20` by travel direction.
- **ACTIVE → FADING:** directional exit when `frontX` passes field bounds ±20.
- **FADING:** `gust.strength` steps to **0** immediately (spring drives visual release with overshoot); after **0.05 s** → `CALM`.
- **ACTIVE:** constant `gust.strength = targetStrength` (no sine wobble).
- Spawn gated: `calmTimer <= 0` **and** (`springSettled` or `phaseTimer > 8 s`).

### Spring recovery (CPU → shader)

Visual gust strength is **`springPos`**, not raw `gust.strength`. Audio still uses `gust.strength`.

| Constant | Value | Role |
|----------|-------|------|
| `SPRING_STIFFNESS` | 1.8 | Slower recovery (~5 s period) |
| `SPRING_DAMPING` | 0.9 | Underdamped overshoot on release |
| `TRAIL_STIFFNESS` | 3.0 | Cursor bend spring-back |
| `TRAIL_DAMPING` | 0.8 | |

- **No clamp** on `springPos` / `trailSpring` (allows negative overshoot past upright).
- **CALM overdamping:** `SPRING_DAMPING * 4` only after `phaseTimer > 5 s` in CALM (lets ring ~5 s first).
- **Smoothed uniforms:** `currentDirX/Z` (lerp over ~⅓ build duration), `currentWidth` (lerp `delta * 1.5`), `gustBlend` (ramps in during BUILDING/ACTIVE/FADING; fades in CALM once spring settled).
- **`uGustBlend`:** gates `inGust` to 0 at spawn so band geometry cannot snap visible lean before blend ramps up.

Console on load: `Springs: 1.8 0.9 3 0.8`.

### Mouse trail (256×256 ping-pong RT)

**Write (trail fragment shader):**

- `prev.b *= uFade` (slider-controlled; default **0.996**, range **0.95–0.998**).
- `prev.rg = mix(vec2(0.5), prev.rg, prev.b)` — direction returns to neutral as intensity fades.
- Hard cutoff: `if (prev.b < 0.012)` → `(0.5, 0.5, 0.0)`.
- Brush at `uMouse` (field UV): strength × `(0.4 + uCursorSpeed * 0.6)`; paints encoded direction into RG and adds to B.

**CPU each frame (`updateCursorDirection`):**

- Smoothed world-space direction from cursor delta (`smoothDirX/Z`).
- Smoothed speed `smoothSpeed`; `uCursorSpeed = min(1, smoothSpeed * 0.8)` on trail + grass materials.

**Trail spring (`animate`):** `trailTarget = min(1, smoothVelocity * 0.8) * onGrass` → `uTrailSpring` multiplies cursor push for residual overshoot after cursor stops.

**Read (grass):** `trailDir = rg*2-1`, `trailIntensity = b`. B drives **displacement**; color uses `vCursorBend` derived from push (B × tip height × spring), not raw B alone.

**Mouse:** raycast to Y=0 → field UV → `uMouse`. `mouseleave` sets `uMouse (-10,-10)` (stops painting; does **not** clear RT).

At **60 fps**, default `uFade = 0.996` gives slower spatial fade than 0.992 (~2× longer trail persistence).

### Camera

**Locked** after intro — no CAMERA panel.

| | Start (intro) | End (rest) |
|---|----------------|------------|
| Position | `(0, 34.0, 13.5)` | `(0, 18, 4)` |
| Look at | `(0, 1, -29)` | same |
| FOV | 43 | 35 |

- Intro **2.0 s**, cubic ease-out (`INTRO_DURATION`).
- Resize updates aspect + renderer size only.

### Audio (Web Audio; first click/mousemove unlocks)

**Cursor rustle** (panner follows mouse X):

- Two noise layers: bandpass ~2.8 kHz + lowpass ~400 Hz.
- Driven by `smoothVelocity * trailIntensity` (B channel 8×8 read under cursor).
- `cursorActivity` caps rustle when still or off grass.

**Wind** (separate `windPanner`, pans with `gust.frontX`):

- **Whoosh:** low bandpass ~300–550 Hz, `gain ∝ smoothWindActivity * 0.009`.
- **Rustle:** high bandpass 1800 Hz + 2400–4000 Hz, `gain ∝ smoothWindActivity * 0.038`.
- Driven by `gust.strength` (asymmetric attack/decay), not `springPos`.

**MUTE** silences cursor + wind gains.

### UI

| Panel | Notes |
|-------|--------|
| **WIND** | Speed, strength, turbulence, direction — bound to legacy grass uniforms (visual gust uses CPU machine + springs). |
| **INTERACTION** | Brush, trail fade, displacement X/Z. |
| **MUTE** | Bottom-right. |
| **Fullscreen** | Above MUTE; `requestFullscreen` on `<html>`. |

**INTERACTION defaults:**

| Control | Default |
|---------|---------|
| Brush Size | 0.022 |
| Brush Strength | 0.85 |
| Trail Fade (`uFade`) | **0.996** |
| Displacement X (`uTrailStrengthX`) | 0.8 |
| Displacement Z (`uTrailStrengthZ`) | 0.8 (bound; push magnitude uses X scale in shader) |

**WIND defaults:** speed 0.75, strength 0.8, turbulence 0.3, direction 0.125.

### Animation loop (each frame)

1. `getDelta()` / `elapsed`; camera intro (if elapsed < 2 s).
2. `updateGust(delta)`.
3. Gust spring integration → `uGustStrength = springPos`; `gustBlend` update → `uGustBlend`.
4. Gust uniforms: `uGustFrontX`, smoothed `uGustWidth`, `uGustDirX/Z`, `uTime`.
5. `updateCursorDirection()` + `uCursorSpeed`.
6. `updateTrail()` (ping-pong).
7. `updateAudioFromTrail()`; trail spring → `uTrailSpring`.
8. Render grass + ground.

---

## Planned / not in WebGL yet

- Random wind gusts tied to WIND sliders in shader (replaced by CPU gust bands).
- Grass cutting / mow mode.
- Three-angle spring physics + awns from Canvas reference.
- Short lawn mode switcher.

---

## Known bad approaches (both builds)

- Solid-fill blades without gradients (looked like paint).
- Layered shell/dot passes (pixelated).
- Single-angle rigid physics (Canvas; do not revert in 2D).
- Cross-plane merged blade geometry (reverted; X-clumps, 2× triangles).
- Global sine wind only (replaced by travelling gust bands in WebGL).
- Projecting ambient lean onto gust direction axis (caused direction snap — ambient now uses fixed `uAmbientDirX/Z`).
- Spring reset on `spawnGust()` (killed continuity; removed).
- Clamping `springPos` to ≥ 0 (prevented visible overshoot; removed).
- Smooth FADING ramp of `gust.strength` (spring arrived at 0 with no velocity; replaced by step to 0 for spring-driven recovery).
- **Cursor tip color from raw trail B** (B lingers in RT after push recovers → ghost discoloration; use `vCursorBend` from push instead).
- **Hard `smoothstep` gate on cursor color** (unnatural recovery cliff; use `pow(bend, 1.4)` instead).
- **Double `tipWeight` on color** (color faded before visible bend recovered; `vCursorBend` is already tip-weighted per vertex).

---

## Performance

- WebGL target: **150k** instances, ~1.8M triangles, DPR cap 1.5 — profile in browser (headless FPS not reliable here).
- Canvas 2D hard limit ~4000 blades; WebGL migration was for density + GPU trail.
