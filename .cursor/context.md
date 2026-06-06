# Touch Grass — project context

Barley-like tall grass field: cursor trail, travelling wind gusts, grass cutting with clip particles, day/night TOD, fireflies, rain weather system, and ASMR audio. **Single-file WebGL build:** `index.html`.

## Files

| File | Role |
|------|------|
| `index.html` | Active build — Three.js r169 (esm.sh), instanced grass tiers, gust CPU, trail + mow ping-pong RTs, Web Audio, fireflies, clips. |
| `.cursor/design.md` | Colour values, TOD targets, geometry tiers, visual architecture. |
| `.cursor/context.md` | This file — behaviour, systems, performance notes. |
| `assets/` | Custom cursor SVGs (`Hand32.svg`, `Sci32.svg`, 64px variants). |

Serve locally: `npx serve .` → open `index.html`.

---

## Stack

- Three.js **0.169** via importmap (`three` from esm.sh).
- **295,000** instanced blades across three tiers (see `.cursor/design.md`).
- DPR cap: `Math.min(devicePixelRatio, 1.0)`.
- Clear colour day `#0d1a09`; ground uses TOD-aware gradient shader.

---

## Field & camera

**No intro pan** — camera starts locked at final pose.

| | Value |
|---|--------|
| Position | `(0, 18, 4)` |
| Look-at | `(0, 1, -29)` |
| FOV | 35 |

Field bounds are computed once from the locked frustum raycast to **Y=0**, using a **16:9 reference aspect** (`FIELD_REFERENCE_ASPECT`), then expanded **6% on all sides** (X, near Z, far Z):

- `fieldMinX`, `fieldMaxX`, `fieldMinZ`, `fieldMaxZ` (logged to console).
- Grass/trail/mow UVs: `(world - fieldMin) * uFieldInvW/D` (reciprocals set at init).
- Ground plane sized/positioned to match field bounds.

---

## Grass tiers

Three `InstancedMesh` layers share one vertex/fragment shader (separate materials for culling):

| Tier | Count | Segments | Side |
|------|-------|----------|------|
| `grassNear` | 145,000 | 3 | `DoubleSide` |
| `grassFar` | 100,000 | 2 | `FrontSide` |
| `grassVeryFar` | 50,000 | 1 | `FrontSide` |

Per-instance attributes: `aHeight` 0.4–1.2, `aWidth` ~0.018–0.035, `aLean` −0.25 to +0.45, `aRandom`.

Z placement (within field bounds):

- **Near:** `zT = 0.28 + pow(rand, 0.22) * 0.72` (near-biased, spans full depth).
- **Far:** `zT = pow(rand, 2.2) * 0.88`.
- **Very far:** `zT = pow(rand, 3.5) * 0.60`.

---

## Grass vertex shader — displacement order

1. Scale width/height, S-curve lean in local X.
2. **Mow height** — sample `uMowMap` at shared `fieldUV`; cut phase (>0.5) clamps blade to 50% height; regrowth phase linearly restores full height.
3. **Trail displacement** (local space): sample `uTrail` at same `fieldUV`.
   - R/G: stroke direction; B: intensity.
   - `push = trailIntensity * pow(windUvY, 2) * uTrailStrengthX * 2.5 * effectiveScale`.
4. **Gust band** (world space): early-out when `uGustBlend <= 0.001` or `abs(uGustStrength) <= 0.001`; otherwise `inGust = lead * trailEdge * uGustBlend`.
5. **Ambient wind** — fixed `uAmbientDirX/Z`, independent of gust direction.
6. **`vCursorBend`:** `pow(trailIntensity, 3.5)` for fragment bent-colour mix.

**Key grass uniforms:** `uTime`, gust/spring uniforms, ambient, `uTrail`, `uMowMap`, `uFieldMinX/Z`, `uFieldInvW/D`, `uTrailStrengthX/Z`, `uCutHeight`.

Removed dead uniforms (not referenced in shaders): `uTrailSpring`, `uWindSpeed`, `uWindStrength`, `uTurbulence`, `uWindDir`.

---

## Grass fragment shader

Procedural barley gradient with TOD-lerpable palette uniforms (`uBaseColor`, `uMidColor`, `uTipPale`, `uTipBright`, `uBentBase`–`uBentTipHi`).

- AO via `uAOMin`; directional light via **`uSunDir` pre-normalised on CPU** (init + after TOD lerp).
- Distance haze via TOD-lerpable `uHazeColor`.
- Mow cut brightening when `vMow > 0.35`.
- Cursor bend mixes toward bent palette using `vCursorBend`.

---

## Gust system (CPU)

State machine: `CALM` → `BUILDING` → `ACTIVE` → `FADING` → `CALM`.

Visual strength is **`springPos`** (underdamped spring), not raw `gust.strength`. Audio uses `gust.strength`.

| Constant | Value |
|----------|-------|
| `SPRING_STIFFNESS` | 1.8 |
| `SPRING_DAMPING` | 0.9 (×4 overdamping in CALM after 5 s) |
| `TRAIL_STIFFNESS` | 3.0 |
| `TRAIL_DAMPING` | 3.5 |

Smoothed uniforms: `currentDirX/Z`, `currentWidth`, `gustBlend` → `uGustBlend`.

---

## Mouse trail (256×256 ping-pong RT)

- Write pass: fade `prev.b *= uFade` (default **0.996**), direction neutralises as B fades, brush at cursor UV.
- Read in grass: trail displacement + `vCursorBend` for colour.
- `uBrushSize` default **0.007** (field-normalised).

---

## Mow system

| Layer | Resolution | Role |
|-------|-----------|------|
| Mow RT (`uMowMap`) | 256×256 HalfFloat ping-pong | Visual blade height (GPU) |
| `mowCells` grid | 48×48 CPU | FX gating — snip audio + clip particles |

- **Regrowth:** `safeDelta / 20.0` per frame (one full-map decay pass, then stamp sub-steps; intensity 1→0 over ~20 s).
- **GPU stamps:** `renderMowPass` cut mode; brush `step(dist, uBrushSize * 0.6)` — **no** trail `farBoost` (trail brush is wider, especially toward horizon).
- **Horizontal cut fidelity:** world-space sub-steps (`stepLenWorld = brush * 0.25 * min(fieldW, fieldD)`, up to 48); dual endpoint sampling on first sub-step (`t0` + `t1`); stamp passes use a small **scissor** around each sub-step end (~brush radius in texels).
- **`applyMowAtPoint`:** check `recentlyCut` (**3 s** per cell, `(mowElapsed - mowCells[idx]) < 3`) **before** `mowCellMark`; spawn clips + `fireCutSnip` if clear, max **3** per frame; then mark cell. Do not mark before the check (same-frame `recentlyCut` would always block FX).
- **`mowCellCut`:** audio “over cut” timbre only — cell marked within last **10 s** (`mowElapsed - mowCells[idx] < 10`); **no** `readRenderTargetPixels` on mow RT (removed for perf).
- **Input:** cut when `isMouseDown || isTouchMowing` and cursor on field; desktop = **LMB hold**; touch = **two fingers**, same direction.
- **Clips:** 1200-slot pool; **`activeClips`** index list — `updateClips()` iterates only active clips and returns early when empty.

---

## TOD, fireflies, audio

- **TOD toggle** (`#btn-tod`): 2 s quadratic ease; lerps grass palette, bent colours, `uHazeColor`, `uSunDir` (re-normalised each frame during transition), sky, ground.
- **Fireflies:** permanent + temp population (defaults 280 + 40); glow uses 12 slot instances. Sim + glow updates **skipped when `fireflyFadeT === 0 && !isNight`**. Fireflies suppressed during rain (`rainT > 0`).
- **Firefly dev panel** (`#ff-dev-panel`): collapsed by default (`ff-dev-collapsed` on init); visible in `?dev` mode context alongside `#fps-dev`.
- **Audio:** cursor rustle, wind gust layers, cut drag loop (`cutDragGain` while LMB/touch cutting on grass), snip via `fireCutSnip` (50 ms throttle, gated by `applyMowAtPoint`).

---

## Rain weather system

Toggle: `#btn-weather` → `toggleWeather()` → `isRain` bool → `rainTransitioning = true`.

### Transition

`rainT` (0 = dry, 1 = raining) transitions over `WEATHER_TRANS_DUR` (quadratic ease). Every frame while `rainT > 0`, `applyWeather()` is called. `applyWeather()` performs a **2D bilinear lerp** across four palette snapshots — `SUNNY_DAY`, `SUNNY_NIGHT`, `RAIN_DAY`, `RAIN_NIGHT` — using both `todT` and `rainT` simultaneously to resolve all grass, ground, sky, haze, and streak uniforms correctly.

### Rain streaks

- **`RAIN_COUNT = 50000`** instanced quads (`InstancedMesh`), geometry `hw=0.010 / hh=0.42`.
- Per-instance `InstancedBufferAttribute`s: `aRainPos` (vec3), `aRainSpeed`, `aRainAlpha`, `aRainLen`.
- `rainMat`: `ShaderMaterial`, `transparent: true`, `depthWrite: false`, `AdditiveBlending`, `renderOrder: 10`.
- **Fall direction:** hardcoded `FALL_DIR = vec3(0.0, 1.0, 0.08)` (slight forward tilt, no wind lean). Only Y is updated per-frame; X is static.
- **Vertex shader billboard approach:**
  1. Compute world-space `wBottom / wTop` from `aRainPos ± FALL_DIR * halfLen`.
  2. Project both through MVP to clip space (`cBot`, `cTop`).
  3. `cPos = mix(cBot, cTop, t)` for interpolation along centerline.
  4. Derive screen-space perpendicular **in pixel space** (multiply NDC axis by `aspect = projectionMatrix[1][1] / projectionMatrix[0][0]`, then divide X of result back), so width is measured in **viewport-height units** and is independent of window width.
  5. Perpendicular rotation must be **90° clockwise** (`vec2(axisP.y, -axisP.x)`); CCW produces inverted winding → backface cull → invisible.
  6. `clipHw = (0.0012 + depthT * 0.0008) * cPos.w` → ~1.3–2.2 px total width on 1080p.
- **Fragment:** `gl_FragColor = vec4(uRainColor, vAlpha * hazeFade * depthAlpha)`.
- **`updateRain(delta)`:** early-return when `rainT === 0`. Falls `fallSpeed=16` world-units/s scaled per-instance by `(0.6 + depthT*0.9)`. Respawn at `Y=28–34` when `Y < -1`. Sets `rainPosBuf.needsUpdate = true`.

### Grass response to rain

In the grass **vertex shader** (`uHazeDensity` doubles as rain intensity, `value = rainT`):

- **Haze tightening:** `hazeNear = 0.55 - uHazeDensity * 0.30`, `hazeFar = 0.45 - uHazeDensity * 0.10`.
- **Rain impact tremor:** time-quantized hash per blade; tick rate `5.0 Hz`; hit probability `10%`; attack `4.0`, decay `2.5`; displacement magnitude `0.38 * aHeight * windStrength`. Creates occasional asynchronous blade flicks suggesting raindrop impacts.
- Ambient sway and cursor trail strength scaled down by `rainT` in JS (`uTrailStrengthX/Z *= (1 - rainT)`).

### Rain rebound particles

Reuse the existing 1200-slot clip pool. `spawnRainDrop()` sets the `isRain` flag on a clip. The clip shaders branch on `vIsRain` (from `aClipIsRain` attribute) to use `uRainDropColor` instead of the grass palette. `uRainDropColor` lerps day-rain `(0.82, 0.88, 0.82)` to night-rain `(0.55, 0.62, 0.78)`. Spawned 20–40 per frame at canopy height `Y=0.4–0.9` when `rainT > 0`.

### Rain audio

- **`rain.mp3`** (`MediaElementSource`, `audio/` folder): looped manually via `ended` event listener (not `loop = true`). Volume faded with `rainT`. Music and wind audio scaled by `(1 - rainT)`.
- **`thunder.mp3`** (`BufferSource`): scheduled at random 15–35 s intervals while `isRain`; gain randomised `0.35–0.50`.
- **`rainGustGain`** (synthetic bandpass gust hiss): set to `0.0` during rain — `rain.mp3` replaces it.

---

## UI

| Control | Notes |
|---------|--------|
| `#btn-info` | Info card (morph animation) |
| `#btn-tod` | Day/night toggle |
| `#btn-weather` | Rain toggle → `toggleWeather()` |
| Custom cursor | Hand hover / scissors while cutting; hidden until loader clears (`sceneReady`) |
| `#fps-dev` | FPS overlay when `?dev` in URL |
| `#ff-dev-panel` | Firefly tuning (dev) |

No WIND / INTERACTION slider panels — trail/mow brush values are code constants.

---

## Animation loop (each frame)

1. `updateGust` → spring integration → gust uniforms.
2. TOD lerp (if transitioning) → **`uSunDir.normalize()`**.
3. Weather lerp (if `rainTransitioning`) → `applyWeather()` 2D bilinear lerp.
4. Firefly fade timer; trail + mow updates; audio (wind layers scaled by `1 - rainT`).
5. `updateRain(delta)` — streak physics + rebound particle spawning (early-out if `rainT === 0`).
6. `updateFireflies` (day skip); `updateClips` (active-list early-out); `updateFireflyGlowInstances` (day skip, reused buffers).
7. Render grass tiers + ground + fireflies + clips + glow + rain streaks.

---

## Performance optimisations (June 2026)

CPU/shader changes with **no intended visual or audio change**:

| Fix | Mechanism |
|-----|-----------|
| Gust early-out | Skip gust sin block when blend/strength ≈ 0 |
| Firefly day skip | Skip sim + glow when fully faded and day mode |
| Firefly buffer reuse | Pre-allocated `_ffOccupiedSet` + `_ffCandidates`, cleared each frame |
| Clip active-list | `activeClips[]` maintained on spawn/deactivate; no full 1200-slot scan |
| Field UV dedup | Single `fieldUV` in vertex shader; `uFieldInvW/D` reciprocals at init |
| CPU `uSunDir` | Normalised on CPU; fragment uses `uSunDir` directly |
| Dead uniforms | Removed unused wind + `uTrailSpring` declarations and uploads |
| Mow stamp scissor | Cut stamps render only a small RT region per sub-step |
| Mow CPU readback removed | No `readRenderTargetPixels` on mow RT; `mowCellCut` uses cell timer only |
| Trail audio readback removed | `updateAudioFromTrail` uses `smoothVelocity` instead of `readRenderTargetPixels`; eliminates GPU→CPU stall on every frame the cursor is on field |
| Trail RT idle skip | `updateTrail()` skips when cursor off-field for ≥ 750 frames (~12.5 s); `trailHasResidual` tracks state |
| Mow RT idle skip | `updateMow()` skips all RT passes when `!mowHasActive && !cutting`; `mowHasActive` set by `applyMowAtPoint`, cleared 22 s after last cut |
| `clipMesh.count` respects perf mode | `applyPerfModeState` sets `clipMesh.count = clipPoolMax`; GPU only processes active clip count |
| Firefly loop ceiling | `fireflyLoopCount` variable capped at `FF_PERM_PQ + 10 = 90` in perf mode (vs 320); ~75% reduction in night firefly iteration |
| Grass bounding spheres | Correct field-sized `THREE.Sphere` set on all six grass meshes; `frustumCulled = true` for driver-hint correctness |

**Not applied (visual regression):** `grassNear` `FrontSide` culling — reverted to `DoubleSide`.
**Deferred (too invasive):** dual-renderer MSAA disable in perf mode — cannot change `antialias` after renderer creation without destroying/recreating WebGL context and all RTs.

Target: ~295k blades, DPR 1.0 — profile in browser.

---

## Known bad approaches

- Solid-fill blades without gradients.
- Global sine wind only (replaced by travelling gust bands).
- Projecting ambient lean onto gust direction (direction snap).
- Clamping `springPos` to ≥ 0 (kills overshoot).
- Cursor tip colour from raw trail B (use `vCursorBend`).
- RT-only mow FX gating without sub-stepping (horizontal cuts missed snips/clips).
- `mowCellMark` before `recentlyCut` in `applyMowAtPoint` (blocks all clip/snip spawns).
- `readRenderTargetPixels` on mow RT in the cut hot path (GPU stall).
- `readRenderTargetPixels` on trail RT in `updateAudioFromTrail` every frame cursor is on field (GPU→CPU sync stall); replaced by velocity-based intensity via `smoothVelocity`.
- Per-frame `new Set()` / full clip-pool scans in hot paths.
- **Rain streak perp = `vec2(-axis.y, axis.x)`** (90° CCW → left-pointing → inverts triangle winding → CW → backface cull → all streaks invisible). Use `vec2(axis.y, -axis.x)` (CW rotation).
- **Rain streak width in raw NDC units** without aspect correction: `perp ≈ (1, 0)` for vertical streaks, so width ∝ viewport *width* — shrinking the window makes rain thinner. Normalize axis in pixel space (`axis.x * aspect`) and convert perp back; express width in NDC-*height* units.
- **`rain.mp3` at `assets/rain.mp3`** — file lives in `audio/rain.mp3`.
- **`rainAudioEl.loop = true`** — unreliable for `MediaElementSource`; use manual `ended` listener.
- **Rain lean driven by `currentDirX` / `springPos`** — was causing wind gust orientation to bleed into rain direction; replaced with fixed `FALL_DIR = vec3(0.0, 1.0, 0.08)`.

---

## Not in build

- Canvas 2D reference (`touch_grass.html`) — removed; three-angle spring physics + awn geometry not ported.
- Camera intro pan / post-intro grass trim — removed.
- WIND panel live tuning — removed.
