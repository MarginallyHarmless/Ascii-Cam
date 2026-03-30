# Three.js Instanced Particle System — ASCII Cam

## Summary

Replace the 2D canvas rendering with a Three.js `InstancedMesh` particle system. Each ASCII character becomes a textured quad floating in 3D space. Motion detected in the webcam feed drives scatter and turbulence forces on nearby characters. Characters drift in an ambient floating state at rest and spring back after disturbance. A subtle auto-orbiting camera reveals the 3D depth. The old glitch system is removed entirely.

## Architecture

Two layers:

### 1. Video Sampling Layer (retained, modified)

Same pipeline as today with the glitch system stripped out:

- `sampleFrame()` downscales the webcam feed to a `numCols x numRows` grid via the hidden `#src-canvas`.
- Per cell: compute brightness, map to character index, store RGB color.
- Output: flat arrays `targetIdx[]` (Int32Array) and `charColors[]` (Uint8Array).
- `applyGlitch()`, `glitchTTL`, `glitchRate`, `glitchRange`, `displayIdx` are all removed.

Additionally, `sampleFrame()` now computes a **motion delta** array:

- A `prevBrightness` Float32Array stores last frame's per-cell brightness.
- `motionDelta[i] = abs(currentBrightness[i] - prevBrightness[i])` — a value 0..1 per cell.
- `prevBrightness` is updated to current values at the end of each frame.

### 2. Three.js Rendering Layer (new, replaces 2D canvas)

**Scene setup:**
- `THREE.WebGLRenderer` with `alpha: false`, background `#000`, `antialias: true`.
- Renderer canvas replaces `#out-canvas`. Renderer is appended to `<body>`, sized to fill viewport via `renderer.setSize(window.innerWidth, window.innerHeight)`. On resize, update renderer size and camera aspect.
- `THREE.PerspectiveCamera`, fov 50, positioned to frame the full character grid. The camera auto-orbits: `camera.position.x += sin(time * 0.15) * 0.3` and `camera.position.y += cos(time * 0.1) * 0.2`, always looking at the grid center.

**Texture atlas:**
- On startup (and when charset changes), render all unique characters in the active charset onto a hidden canvas in a single row. Each cell is 32px wide x 64px tall. Upload as `THREE.CanvasTexture` with `minFilter: LinearFilter`.
- For charset "dense" (`@#S%?*+;:,. ` — 12 chars), atlas is 384x64px. For "blocks" (5 chars), 160x64. For "minimal" (4 chars), 128x64.
- Store the atlas canvas reference to regenerate on charset change.

**InstancedMesh:**
- Geometry: `THREE.PlaneGeometry(charWidth, charHeight)` — one quad.
- Material: `THREE.MeshBasicMaterial` with the atlas texture, `transparent: true`, `alphaTest: 0.1`.
- Instance count: `numCols * numRows` (recomputed when cols or viewport changes).
- Per-instance attributes (all `THREE.InstancedBufferAttribute`):
  - `instanceUvOffset` (Float32, 1) — horizontal UV offset into the atlas to select the character.
  - `instanceColor` (Float32, 3) — RGB color for the instance (using mesh's `.instanceColor`).

**Per-instance state (JS-side arrays, not GPU attributes):**
- `restPosition[i]` — vec3, the grid slot position.
- `position[i]` — vec3, current position.
- `velocity[i]` — vec3, current velocity.
- `rotation[i]` — float, current Z rotation.
- `angularVelocity[i]` — float, current spin speed.

These are applied to the instance matrix each frame via `dummy.position.set(...)`, `dummy.rotation.z = ...`, `mesh.setMatrixAt(i, dummy.matrix)`.

## Motion Detection to Forces

Each frame, after `sampleFrame()`:

1. For each cell `i`, read `motionDelta[i]`.
2. If `motionDelta[i] > sensitivityThreshold` (user-configurable, default 0.08):
   - Compute force magnitude: `forceMag = motionDelta[i] * forceMultiplier` (forceMultiplier = 15.0).
   - Apply scatter impulse: `velocity[i] += randomDirection3D() * forceMag`.
   - Apply angular impulse: `angularVelocity[i] += (Math.random() - 0.5) * forceMag * 2.0`.
3. `randomDirection3D()` returns a normalized vec3 with random x, y, z components in [-1, 1].

## Physics Step (per frame)

For each instance `i`:

```
springForce = -springK * (position[i] - restPosition[i])   // springK = 2.0
velocity[i] += springForce * deltaTime
velocity[i] *= dampingFactor                                 // dampingFactor = 0.95
position[i] += velocity[i] * deltaTime

angularVelocity[i] *= angularDamping                         // angularDamping = 0.92
rotation[i] += angularVelocity[i] * deltaTime
```

`deltaTime` is clamped to max 0.05 (20fps floor) to prevent explosion on tab-switch.

## Ambient Floating Drift

When no motion is detected, characters still have a gentle ambient drift:

- Each instance has a phase offset `phi[i] = i * 0.1`.
- Each frame: `restPosition[i].z = sin(time * 0.5 + phi[i]) * 0.15` — a very subtle sine wave in Z.
- This means "at rest" is not perfectly flat, but gently undulating, giving a suspended-in-liquid feel.

## Grid Sizing

- `numCols` is controlled by the columns slider (30-140, default 72).
- `numRows` is derived from viewport dimensions and character cell aspect ratio, same as current logic (accounting for bar height).
- Character quad size in world space: `charWidth = viewportWorldWidth / numCols`, `charHeight = charWidth * 2.0` (to match monospace character proportions).
- Grid is centered at world origin.

When `numCols` changes or viewport resizes:
- Recreate the `InstancedMesh` with new instance count.
- Recompute `restPosition` for all instances.
- Reset all velocities and rotations to zero.

## Color Mode

- **Mono mode (default):** `instanceColor` is set to a grayscale value derived from brightness. Brighter cells = lighter gray.
- **Color mode:** `instanceColor` is set to the actual RGB from the video frame.
- Same toggle button as current app.

## Bottom Bar Changes

Remove:
- Glitch rate slider (`#rate-r`, `#rate-v`)
- Glitch range slider (`#range-r`, `#range-v`)

Add:
- Sensitivity slider: `id="sens-r"`, min 1, max 20, default 8, step 1, `title="sensitivity"`. Maps to `sensitivityThreshold = (21 - sliderValue) / 100` — higher slider value = more sensitive (lower threshold).

Keep:
- Charset select
- Columns slider
- Flip, color, invert, save buttons
- Logo

## Removed Code

- `applyGlitch()` function
- `glitchRate`, `glitchRange`, `glitchTTL`, `displayIdx` variables
- `drawFrame()` function (replaced by Three.js render loop)
- `outCtx` (2D context no longer needed)
- `#out-canvas` CSS fixed positioning (Three.js renderer creates its own canvas)
- Glitch rate and glitch range slider event listeners
- Glitch rate and glitch range HTML elements

## Dependencies

- Three.js r170 via CDN: `<script src="https://cdn.jsdelivr.net/npm/three@0.170.0/build/three.min.js"></script>`
- No build system. Single `index.html` file.

## File Structure

Remains a single `index.html` file. Three.js is loaded via CDN script tag in `<head>`.

## Save Button

The save button currently calls `outCv.toDataURL()`. With Three.js, it changes to `renderer.domElement.toDataURL('image/png')`. The renderer must call `preserveDrawingBuffer: true` on creation to support this.
