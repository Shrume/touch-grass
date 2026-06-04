# Touch Grass — project context

Barley-like tall grass field: cursor trail, travelling wind gusts, grass cutting with clip particles, day/night TOD, fireflies, and ASMR audio. **Single-file WebGL build:** `index.html`.

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
- **Fireflies:** permanent + temp population (defaults 280 + 40); glow uses 12 slot instances. Sim + glow updates **skipped when `fireflyFadeT === 0 && !isNight`**.
- **Firefly dev panel** (`#ff-dev-panel`): collapsed by default (`ff-dev-collapsed` on init); visible in `?dev` mode context alongside `#fps-dev`.
- **Audio:** cursor rustle, wind gust layers, cut drag loop (`cutDragGain` while LMB/touch cutting on grass), snip via `fireCutSnip` (50 ms throttle, gated by `applyMowAtPoint`).

---

## UI

| Control | Notes |
|---------|--------|
| `#btn-info` | Info card (morph animation) |
| `#btn-tod` | Day/night toggle |
| Custom cursor | Hand hover / scissors while cutting; hidden until loader clears (`sceneReady`) |
| `#fps-dev` | FPS overlay when `?dev` in URL |
| `#ff-dev-panel` | Firefly tuning (dev) |

No WIND / INTERACTION slider panels — trail/mow brush values are code constants.

---

## Animation loop (each frame)

1. `updateGust` → spring integration → gust uniforms.
2. TOD lerp (if transitioning) → **`uSunDir.normalize()`**.
3. Firefly fade timer; trail + mow updates; audio.
4. `updateFireflies` (day skip); `updateClips` (active-list early-out); `updateFireflyGlowInstances` (day skip, reused buffers).
5. Render grass tiers + ground + fireflies + clips + glow.

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

**Not applied (visual regression):** `grassNear` `FrontSide` culling — reverted to `DoubleSide`.

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
- Per-frame `new Set()` / full clip-pool scans in hot paths.

---

## Not in build

- Canvas 2D reference (`touch_grass.html`) — removed; three-angle spring physics + awn geometry not ported.
- Camera intro pan / post-intro grass trim — removed.
- WIND panel live tuning — removed.
