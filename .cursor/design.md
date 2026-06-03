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

---

### Grass Distance Haze (fragment shader — fixed, not TOD-aware)

| Value | vec3 | Hex |
|-------|------|-----|
| `hazeColor` | `(0.13, 0.22, 0.09)` | `#213817` |
| Mix formula | `mix(hazeColor, col, 0.55 + depthT * 0.45)` | — |

⚠️ Currently stays green in night mode — not blue-shifted with TOD.

---

### UI / Chrome

| Element | Day | Night |
|---------|-----|-------|
| `html, body` background | `#040a01` | same |
| Loading overlay | `#0d1a09` | N/A |
| Icon buttons | light glass | `rgba(28, 42, 68, …)` blue |
| Info card | light glass | blue-shifted |

---

## TOD Transition System

**Duration:** 2s, quadratic ease-in-out (`TOD_TRANS_DUR`)
**Mechanism:** CPU snapshot → frame-by-frame lerp while `todTransitioning`

### What changes on TOD switch

| Element | Mechanism | Works? |
|---------|-----------|--------|
| Grass palette (all uniforms above) | Uniform lerp | ✅ |
| Sky / clear colour | `renderer.setClearColor` lerp | ✅ |
| Ground gradient (`uGroundNear` / `uGroundFar`) | Uniform `.lerpColors()` | ✅ |
| Grass distance haze | Not implemented | ❌ stays green at night |

### Night targets (`transitionTo` when `isNight`)

```js
sky:        new THREE.Color(3/255,  5/255,  12/255)  // #03050c
groundNear: new THREE.Color(0x03050c)                 // matches sky
groundFar:  new THREE.Color().setRGB(0.078, 0.135, 0.207)  // #142335 — day-far luminosity, blue hue
```

---

## Known Mismatches (open issues)

1. **Grass distance haze stays green at night** — `hazeColor` is a fixed constant in the fragment shader; should shift toward a blue-grey at night.

---

## Grass Geometry Tiers

| Tier | Instances | Segments | Tris/blade | Total tris | Z placement |
|------|-----------|----------|------------|------------|-------------|
| `grassNear` | 260,000 | 3 | 6 | 1,560,000 | `0.28 + pow(r, 0.22) * 0.62` (near-biased) |
| `grassFar` | 180,000 | 2 | 4 | 720,000 | `pow(r, 2.2) * 0.88` (far-biased) |
| `grassVeryFar` | 90,000 | 1 | 2 | 180,000 | `pow(r, 3.5) * 0.60` (very-far-biased) |
| **Total** | **530,000** | — | — | **~2.46M** | — |

`grassVeryFar` uses `THREE.FrontSide` only; all others `DoubleSide`.

---

## Mow System

| Layer | Resolution | Role |
|-------|-----------|------|
| Mow RT (`uMowMap`) | 256×256 HalfFloat ping-pong | Visual blade height |
| `mowCells` grid | 48×48 CPU | FX gating (snip audio + clip particles) |

- Regrowth: linear decay `safeDelta / 20.0` per frame (one decay pass per frame, decoupled from stamp sub-steps)
- Cell suppression: 20s cooldown, gated by RT intensity > 0.5

---

## Camera (locked)

| | Intro start | Final (locked) |
|--|------------|----------------|
| Position | `(0, 34, 13.5)` | `(0, 18, 4)` |
| Look-at | `(0, 1, -29)` | same |
| FOV | 43 | 35 |

Intro: 2s cubic ease-out. Field bounds fixed at canonical 16:9 aspect from locked pose.
