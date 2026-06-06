# Touch Grass — Design Reference

Colour values, TOD system, and visual architecture for `index.html`.
Sourced from Cursor investigation June 2026.

---

## Scene Colours

### Ground Plane (ShaderMaterial gradient — world Z)

TOD-aware via uniforms `uGroundNear` and `uGroundFar` (lerped each frame during the 2s transition, same ease as sky/grass). Fragment shader: `mix(uGroundFar, uGroundNear, depthT)`.

Gradient direction: darker at front (`depthT = 1`), lighter toward horizon (`depthT = 0`).

**Day** (init values — use exact vec3 literals, not hex; `#0a1406` / `#15260e` differ slightly in the blue channel):

| Role | vec3 (0–1) | RGB (approx) | Hex (approx) |
|------|-----------|--------------|--------------|
| Near / front (`uGroundNear`) | `(0.039, 0.078, 0.024)` | (10, 20, 6) | `#0a1406` |
| Far / back (`uGroundFar`) | `(0.082, 0.149, 0.055)` | (21, 38, 14) | `#15260e` |

**Night** (current `transitionTo` targets):

| Role | vec3 (0–1) | RGB | Hex |
|------|-----------|-----|-----|
| Near / front (`uGroundNear`) | `(0.012, 0.020, 0.047)` | (3, 5, 12) | `#03050c` |
| Far / back (`uGroundFar`) | `(0.078, 0.135, 0.207)` | (20, 35, 53) | `#142335` |

Night near matches night sky clear colour exactly — intentional front-of-frame anchor.

Night far is tuned to **~100% of day far WCAG relative luminance** (was `#0a1220` at ~38%), shifted to the blue hue of the night grass palette (`#16263a` family) so the horizon ground reads at similar brightness to day, not as a dark void.

`DAY_SNAPSHOT` captures `groundNear` / `groundFar` from uniforms **after** `groundMaterial` init.

---

### Sky / Clear Colour

| Mode | Hex | RGB |
|------|-----|-----|
| Day | `#0d1a09` | (13, 26, 9) |
| Night | `#03050c` | (3, 5, 12) |

Lerped as a flat colour over 2s TOD transition via `renderer.setClearColor`.

---

### Grass Palette

| Uniform | Day Hex | Day vec3 | Night Hex | Night vec3 |
|---------|---------|----------|-----------|------------|
| `uBaseColor` | `#3a6428` | (58,100,40)/255 | `#16263a` | (22,38,58)/255 |
| `uMidColor` | `#4e8230` | (78,130,48)/255 | `#203654` | (32,54,84)/255 |
| `uTipPale` | `#a8d860` | (168,216,96)/255 | `#3a6294` | (58,98,148)/255 |
| `uTipBright` | `#c2e870` | (194,232,112)/255 | `#4878ac` | (72,120,172)/255 |
| `uSpecColor` | `(0.9, 1.0, 0.7)` warm | — | `(0.18, 0.22, 0.45)` blue | — |
| `uLightMin` | `0.62` | — | `0.35` | — |
| `uAOMin` | `0.32` | — | `0.08` | — |
| `uSunDir` | `(0.8, 0.6)` normalised on CPU | — | `(0.3, 0.15)` normalised on CPU | — |

### Bent grass palette (cursor trail mix)

TOD-lerpable via `uBentBase`, `uBentMid`, `uBentTipLo`, `uBentTipHi`. Day init:

| Uniform | RGB (approx) |
|---------|--------------|
| `uBentBase` | (32, 62, 18) |
| `uBentMid` | (42, 78, 25) |
| `uBentTipLo` | (65, 118, 40) |
| `uBentTipHi` | (90, 152, 58) |

Night targets set in `applyTOD()` — blue-shifted counterparts matching night grass hue.

---

### Grass Distance Haze (TOD-aware)

| Mode | `uHazeColor` vec3 | Hex (approx) |
|------|-------------------|--------------|
| Day | `(0.13, 0.22, 0.09)` | `#213817` |
| Night | `(0.08, 0.10, 0.18)` | `#141a2e` |

Mix formula (fragment): `mix(uHazeColor, col, 0.55 + depthT * 0.45)`.

Lerped during TOD transition alongside grass palette (`transitionFrom.hazeColor` / `transitionTo.hazeColor`).

---

### UI / Chrome

| Element | Day | Night |
|---------|-----|-------|
| `html, body` background | `#040a01` | same |
| Loading overlay | `#0d1a09` | N/A |
| Icon buttons | light glass | `rgba(28, 42, 68, …)` blue |
| Info card | light glass | blue-shifted |

---

## Weather Palette Constants

Four palette snapshots drive `applyWeather()`. `SUNNY_*` mirrors the TOD palette; `RAIN_*` are the wet-weather overrides. Both `todT` and `rainT` are applied as a 2D bilinear lerp each frame.

### RAIN_DAY

| Key | Value |
|-----|-------|
| `sky` | `#111a0f` |
| `groundNear` | `#0a1209` |
| `groundFar` | `#1a2416` |
| `hazeColor` | `(0.12, 0.15, 0.10)` |
| `baseColor` | `(42, 72, 28)/255` |
| `midColor` | `(56, 88, 38)/255` |
| `tipPale` | `(110, 148, 72)/255` |
| `tipBright` | `(128, 165, 84)/255` |
| `specColor` | `(0.55, 0.62, 0.45)` |
| `lightMin` | `0.44` |
| `aoMin` | `0.22` |
| `sunDir` | `(0.5, 0.35)` normalised |
| `streakColor` | `(0.55, 0.62, 0.52)` |

### RAIN_NIGHT

| Key | Value |
|-----|-------|
| `sky` | `#020408` |
| `groundNear` | `#020408` |
| `groundFar` | `#0d1820` |
| `hazeColor` | `(0.07, 0.09, 0.12)` |
| `baseColor` | `(14, 24, 32)/255` |
| `midColor` | `(20, 34, 46)/255` |
| `tipPale` | `(30, 52, 72)/255` |
| `tipBright` | `(38, 62, 86)/255` |
| `specColor` | `(0.10, 0.13, 0.22)` |
| `lightMin` | `0.22` |
| `aoMin` | `0.06` |
| `sunDir` | `(0.2, 0.10)` normalised |
| `streakColor` | `(0.18, 0.24, 0.32)` |

### Rebound drop color (`uRainDropColor`)

Lerped by `todT` inside `applyWeather()`:

| TOD | vec3 |
|-----|------|
| Day rain | `(0.82, 0.88, 0.82)` — light grey-green |
| Night rain | `(0.55, 0.62, 0.78)` — dim blue-silver |

### Rain fog (`uRainFogColor`, `uRainFogStrength`)

Applied in the grass and ground shaders on top of normal haze. Strength `= (0.72 – 0.07 * todT) * rainT`. Day fog colour interpolates from `hazeColor` toward a cool grey; night fog stays near `hazeColor`.

---

## TOD Transition System

**Duration:** 2s, quadratic ease-in-out (`TOD_TRANS_DUR`)
**Mechanism:** CPU snapshot → frame-by-frame lerp while `todTransitioning`

### What changes on TOD switch

| Element | Mechanism | Works? |
|---------|-----------|--------|
| Grass palette (all uniforms above) | Uniform lerp | ✅ |
| Bent palette | `.lerpColors()` | ✅ |
| Sky / clear colour | `renderer.setClearColor` lerp | ✅ |
| Ground gradient (`uGroundNear` / `uGroundFar`) | Uniform `.lerpColors()` | ✅ |
| Distance haze (`uHazeColor`) | Uniform lerp | ✅ |
| Sun direction (`uSunDir`) | Vector lerp + **CPU normalise** each frame | ✅ |

### What changes on weather toggle

`applyWeather()` replaces the single TOD lerp with a 2D bilinear lerp across all four snapshots. Every lerped element above is re-resolved including rain fog, streak colour, and rebound drop colour.

| Element | Mechanism |
|---------|-----------|
| All grass / ground / sky uniforms | `applyWeather()` 2D lerp (todT × rainT) |
| Streak colour (`uRainColor`) | Lerped in `applyWeather()` |
| Haze density (`uHazeDensity`) | `= rainT` directly |
| Rain fog (`uRainFogColor`, `uRainFogStrength`) | Lerped in `applyWeather()` |
| Rebound drop colour (`uRainDropColor`) | Lerped by `todT` in `applyWeather()` |
| `rain.mp3` volume | Faded with `rainT` in audio loop |
| Wind / rustle audio | Scaled by `1 - rainT` |
| Fireflies | Suppressed when `rainT > 0` |

### Night targets (`transitionTo` when `isNight`)

```js
sky:        new THREE.Color(3/255,  5/255,  12/255)  // #03050c
groundNear: new THREE.Color(0x03050c)                 // matches sky
groundFar:  new THREE.Color().setRGB(0.078, 0.135, 0.207)  // #142335 — day-far luminosity, blue hue
hazeColor:  new THREE.Vector3(0.08, 0.10, 0.18)
sunDir:     new THREE.Vector2(0.3, 0.15)  // normalised on CPU after lerp
```

---

## Grass Geometry Tiers

| Tier | Instances | Segments | Tris/blade | Total tris | Z placement | Culling |
|------|-----------|----------|------------|------------|-------------|---------|
| `grassNear` | 145,000 | 3 | 6 | 870,000 | `0.28 + pow(r, 0.22) * 0.72` | `DoubleSide` |
| `grassFar` | 100,000 | 2 | 4 | 400,000 | `pow(r, 2.2) * 0.88` | `FrontSide` |
| `grassVeryFar` | 50,000 | 1 | 2 | 100,000 | `pow(r, 3.5) * 0.60` | `FrontSide` |
| **Total** | **295,000** | — | — | **~1.37M** | — | — |

DPR capped at **1.0** (not 1.5).

---

## Field Bounds

Computed from locked camera frustum at **16:9 reference aspect**, then **6% margin** on all sides. Bounds are fixed for the session (no intro union or post-intro trim).

Grass/mow/trail UV reciprocals: `uFieldInvW = 1 / (fieldMaxX - fieldMinX)`, `uFieldInvD = 1 / (fieldMaxZ - fieldMinZ)`.

---

## Camera (locked)

| | Value |
|---|--------|
| Position | `(0, 18, 4)` |
| Look-at | `(0, 1, -29)` |
| FOV | 35 |

**No intro animation** — camera starts at final pose. Constants `CAM_INTRO_END_*` name the locked pose only.

---

## Mow System

| Layer | Resolution | Role |
|-------|-----------|------|
| Mow RT (`uMowMap`) | 256×256 HalfFloat ping-pong | Visual blade height |
| `mowCells` grid | 48×48 CPU | FX gating (snip audio + clip particles) |

- **Regrowth:** linear decay `safeDelta / 20.0` per frame (one full-map decay pass, then stamp sub-steps).
- **Visual vs trail:** mow stamp uses fixed `uBrushSize * 0.6` (`step`); trail brush uses `smoothstep` + depth `farBoost` — cuts read narrower than hover trail.
- **Horizontal cuts:** world-space sub-steps (`stepLenWorld = brush * 0.25 * min(fieldW, fieldD)`, max 48); samples segment start (`t0`) and each step end (`t1`); stamp passes use RT **scissor** around sub-step end.
- **FX (`applyMowAtPoint`):** if cell not cut in last **3 s**, spawn clips + snip (≤**3**/frame); **then** `mowCellMark`. Check before mark.
- **Audio timbre (`mowCellCut`):** “over cut” rustle shift if cell marked within **10 s**; CPU timer only (no mow RT readback).
- **Input:** LMB hold (desktop); two-finger same-direction drag (touch).
- **Clip particles:** 1200 pool; CPU maintains `activeClips[]` — physics loop skips inactive slots entirely.

---

## Performance Optimisations (June 2026)

Applied in `index.html` without intended visual/audio change:

| Area | Change |
|------|--------|
| Gust vertex block | Early-out when `uGustBlend` and `uGustStrength` ≈ 0 |
| Fireflies | Skip sim + glow when `fireflyFadeT === 0 && !isNight` |
| Firefly glow slots | Reuse `_ffOccupiedSet` / `_ffCandidates` buffers |
| Clips | `activeClips` index list; early return when empty |
| Grass vertex UV | Single `fieldUV` + CPU reciprocals `uFieldInvW/D` |
| Lighting | `uSunDir` normalised on CPU; no `normalize()` in fragment |
| Uniform cleanup | Removed dead `uTrailSpring` + legacy wind uniforms |
| Mow stamp scissor | Small RT write region per cut sub-step |
| Mow readback removed | `mowCellCut` no longer samples mow RT on CPU |

**Reverted:** `grassNear` `FrontSide` (visible culling artefact) — stays `DoubleSide`.

---

## Rain Streak Visual Spec

| Parameter | Value |
|-----------|-------|
| Count | 50,000 instanced quads |
| Geometry (`hw` × `hh`) | `0.010` × `0.42` (local space) |
| Fall direction (`FALL_DIR`) | `vec3(0.0, 1.0, 0.08)` — slight forward tilt, no wind lean |
| Fall speed base | 16 world-units/s × per-instance `0.6–1.5` depth scale |
| Spawn height | `Y = 28–34`; respawn at `Y < -1` |
| Width (near/far) | `~2.2 px / ~1.3 px` total on 1080p |
| Width formula | `ndcHalfH = 0.0012 + depthT * 0.0008` in NDC-height units |
| Depth fade | `depthAlpha = 0.3 + depthT * 0.7` |
| Haze fade | `hazeFade = clamp(0.05 + vDepthT * 0.95, 0.0, 1.0)` |
| Blending | `AdditiveBlending`, `depthWrite: false` |
| Perspective scale | `perspScale = 0.35 + depthT * 0.65` applied to `halfLen` |

---

## Dev UI

| Element | Behaviour |
|---------|-----------|
| `#fps-dev` | Shown when URL contains `?dev` |
| `#ff-dev-panel` | Firefly count/size tuning; **collapsed by default** |
