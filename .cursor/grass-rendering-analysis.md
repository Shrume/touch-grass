# Grass Rendering Analysis

`touch_grass / index.html` — Three.js r169, 295k instanced blades

---

## Summary

Field bounds are computed **once at init** using a hardcoded 16:9 reference aspect. All 295k grass blade positions are baked to that fixed rectangle. The camera aspect updates on resize, but the field never does — creating a mismatch that wastes GPU work and exposes the clear-color background (`#0d1a09`) when the window shape deviates from 16:9.

---

## Initialization & resize pipeline

```
FIELD_REFERENCE_ASPECT   →   computeFieldBounds()   →   fieldMin/MaxX/Z    →   populateGrassInstances()   →   onResize()
      = 16/9                     @ init only              frozen in JS             295k blades                  camera.aspect only
  (hardcoded constant)        frustum ray → Y=0          + 6% padding          world positions baked           field never updated
                                                                                                                    ↑ window resize
```

The field bounding box is computed exactly once on page load. Grass blade world positions are baked into GPU instance buffers at that moment and never rebuilt. A window resize only updates the camera's aspect ratio projection matrix.

---

## Camera parameters

| Parameter | Value |
|-----------|-------|
| Position | X: 0 · Y: 18 · Z: 4 |
| Look-at | X: 0 · Y: 1 · Z: −29 |
| FOV | 35° vertical |
| Near / Far | 0.1 / 200 |

Three.js `PerspectiveCamera` uses **vertical** FOV. Changing the window aspect preserves vertical world-space coverage at all depths. Only horizontal coverage widens or narrows. Because the field was sized for 16:9 horizontal coverage, any wider aspect exposes the bare background beyond the left/right field edges.

---

## Field bounds (fixed at init)

`computeFieldBounds()` creates a temporary `boundsCam` with `FIELD_REFERENCE_ASPECT = 16/9` and the locked camera pose. Its four frustum corner rays are intersected with the Y=0 ground plane. The resulting rectangle is expanded 6% on all sides. These four values drive everything else and are **never recomputed**.

| Variable | Approx value | Used for |
|----------|-------------|----------|
| `fieldMinX` | ≈ −22 | X west edge |
| `fieldMaxX` | ≈ +22 | X east edge |
| `fieldMinZ` | ≈ −58 | Z far edge (horizon) |
| `fieldMaxZ` | ≈ +4.5 | Z near edge (behind cam) |
| `fieldW` | ≈ 44 wu | Trail/mow UV scale |
| `fieldD` | ≈ 62.5 wu | Trail/mow UV scale |

### Top-down view (schematic)

```
                     ← X world →
         ┌──────────────────────────────────┐  fieldMaxX ≈ +22
         │                                  │
         │       grass field (16:9 + 6%)    │
         │                                  │
         │                                  │
         │          lookAt (0,1,−29)        │
         │                                  │
         │                                  │
         └──────────────────────────────────┘  fieldMinX ≈ −22
  Z far (−58)                              Z near (+4.5)
                        ▲
                    cam (0,18,4)
                   [frustum cone]

  ← 16:9 reference frustum matches field exactly
  ← wider aspect frustum extends beyond left/right edges → exposes clear color
```

### Side view (schematic)

```
  Y
  18 ● cam (0, 18, 4)
     |\
  35°| \ upper ray (toward horizon / fieldMinZ)
  vFOV\ \
       \ \____ lower ray (toward fieldMaxZ / near)
  0 ════════════════════════════════════════════════ ground Y=0
       Z +4.5              Z ≈ −29         Z −58
       (near)          (lookAt)           (far)
       |←───────── grass field (≈ 62.5 wu deep) ─────────→|
       ||||  |||||||||  ||||||||||||||||||  ||||  ||  |
       (schematic blade positions across field depth)
```

---

## Grass tiers

| Mesh | HQ blades | PQ blades | Segments | Side | Z placement bias |
|------|-----------|-----------|----------|------|-----------------|
| `grassNear` | 145,000 | 90,000 | 3 (HQ) / 2 (PQ) | DoubleSide | near-biased, full depth (`zT = 0.28 + rand^0.22 × 0.72`) |
| `grassFar` | 100,000 | 65,000 | 2 | FrontSide | mid-concentrated (`zT = rand^2.2 × 0.88`) |
| `grassVeryFar` | 50,000 | 25,000 | 1 | FrontSide | front-concentrated (`zT = rand^3.5 × 0.60`) |
| **Total HQ** | **295,000** | | | | |
| **Total PQ** | **180,000** | | | | |

All tiers share a single `populateGrassInstances()` call. X positions use a striped column grid across the full field width. Z positions use tier-specific power-curve biases so denser blades appear in mid/near-field. The **bounding sphere** on each mesh covers the entire field, so Three.js frustum culling is all-or-nothing per mesh — it cannot cull individual blades outside the actual viewport.

---

## Identified issues

### 1. Field bounds are frozen to a 16:9 reference aspect at init · `root cause`

`computeFieldBounds()` runs once using `FIELD_REFERENCE_ASPECT = 16/9`. `fieldMinX/Z` and `fieldMaxX/Z` never change on resize. All 295k blade positions, trail/mow UVs, and the ground plane are built against these frozen coords.

### 2. Camera aspect updates on resize — field does not · `root cause`

`onResize()` → `updateRendererSize()` updates `camera.aspect` and calls `updateProjectionMatrix()`. Since Three.js `PerspectiveCamera` uses vertical FOV (35°), vertical world-space coverage stays constant across all aspect ratios. Horizontal coverage widens/narrows. The grass field was sized for horizontal 16:9 coverage only.

### 3. All 295k blades are submitted to the GPU regardless of visible area · `waste`

At window aspect > 16:9 (wide): camera's horizontal FOV exceeds field → background visible on sides. At window aspect < 16:9 (tall/narrow): camera's horizontal FOV is narrower than field → blades in X margins are drawn but clipped by the viewport. In both cases the GPU processes all 295k instances via instanced draw calls.

### 4. Reduced window height shows distant background · `visual bug`

When the window is made shorter while keeping the same width, aspect ratio increases (wider). The horizontal FOV expands, but the field was only padded 6% beyond the 16:9 frustum sides. The camera now sees world space left/right of the grass patch, where only the clear color (`#0d1a09`) is painted — appearing as a flat dark band at the scene edges.

### 5. No per-blade GPU culling · `waste`

`populateGrassInstances()` leaves each mesh with `frustumCulled = false` during build. Bounding spheres are set later in a separate block and `frustumCulled` is re-enabled — but the bounding spheres span the entire field, so Three.js frustum culling can only cull all-or-nothing per mesh, not individual blades. There is no per-blade GPU culling.

---

## What the code does today vs consequences

### What the code does today

1. At page load, cast frustum corner rays (16:9 virtual camera) onto Y=0. Expand 6%. Freeze as field bounds.
2. Scatter 295,000 blade instance matrices randomly within those bounds. Upload to GPU once; never rebuilt.
3. On resize: update `camera.aspect` + projection matrix only. Field bounds, blade positions, ground plane unchanged.
4. Each frame: draw all 295k instances unconditionally (bounding sphere per tier, not per blade).

### Consequences

- At any non-16:9 aspect, the camera sees world space that contains no grass (bare `#0d1a09` clear color). On very wide or narrow windows, wide strips of background are exposed on the left/right edges.
- GPU always processes the full 295k-blade draw call regardless of how much of the field is in view. If the window is small (e.g. 400×300), the field covers far more screen area than the viewport but all blades still run through the vertex shader.
- Mow/trail UV mapping, gust front position, and the ground plane are all keyed to the frozen field bounds — resizing the window does not cause UV drift, but if the field were ever resized these would all need updating too.
