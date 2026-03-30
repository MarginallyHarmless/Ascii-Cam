# Three.js Instanced Particle System — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace 2D canvas ASCII rendering with a Three.js InstancedMesh particle system where webcam motion drives scatter/turbulence forces on floating character quads.

**Architecture:** Single-file `index.html` rewrite. Video sampling layer is retained (minus glitch system) with added motion detection. Three.js InstancedMesh renders character quads with per-instance UV offsets, colors, positions, and rotations. Physics step applies spring-back, damping, and motion-driven forces each frame.

**Tech Stack:** Three.js r170 (CDN), vanilla JS, single HTML file.

---

### Task 1: Add Three.js CDN and Remove Glitch HTML Controls

**Files:**
- Modify: `index.html:1-6` (head), `index.html:225-228` (bar-center glitch sliders)

- [ ] **Step 1: Add Three.js script tag in `<head>`**

Add this line just before the closing `</head>` tag (before `</style></head>`, after the `</style>` tag):

```html
<script src="https://cdn.jsdelivr.net/npm/three@0.170.0/build/three.min.js"></script>
```

- [ ] **Step 2: Replace glitch sliders with sensitivity slider in bar-center**

Replace these lines in the bar-center div:

```html
    <input type="range" id="rate-r" min="0" max="100" value="18" step="1" title="glitch rate" />
    <span class="val" id="rate-v">18</span>
    <input type="range" id="range-r" min="1" max="8" value="2" step="1" title="glitch range" />
    <span class="val" id="range-v">2</span>
```

With:

```html
    <input type="range" id="sens-r" min="1" max="20" value="8" step="1" title="sensitivity" />
    <span class="val" id="sens-v">8</span>
```

- [ ] **Step 3: Remove `#out-canvas` from HTML body**

Delete this line from the body:

```html
<canvas id="out-canvas"></canvas>
```

The Three.js renderer will create and append its own canvas.

- [ ] **Step 4: Remove `#out-canvas` CSS rule**

Delete the entire `#out-canvas` CSS block:

```css
/* ---------- fullscreen canvas ---------- */
#out-canvas {
  position: fixed;
  inset: 0;
  width: 100vw;
  height: 100vh;
  display: block;
}
```

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Add Three.js CDN, replace glitch controls with sensitivity slider"
```

---

### Task 2: Remove Glitch System from JavaScript

**Files:**
- Modify: `index.html` (script block)

- [ ] **Step 1: Remove glitch state variables**

Delete these variable declarations:

```js
let glitchRate = 18;
let glitchRange = 2;
```

And remove `displayIdx` and `glitchTTL` from this line:

```js
let targetIdx = [], displayIdx = [], glitchTTL = [], charColors = [];
```

Change it to:

```js
let targetIdx = [], charColors = [];
```

- [ ] **Step 2: Remove glitch array allocation in `sampleFrame()`**

In `sampleFrame()`, replace:

```js
  if (targetIdx.length !== n) {
    targetIdx  = new Int32Array(n);
    displayIdx = new Int32Array(n);
    glitchTTL  = new Uint8Array(n);
    charColors = new Uint8Array(n * 3);
  }
```

With:

```js
  if (targetIdx.length !== n) {
    targetIdx  = new Int32Array(n);
    charColors = new Uint8Array(n * 3);
  }
```

- [ ] **Step 3: Delete `applyGlitch()` function entirely**

Remove the entire `applyGlitch` function (lines 321-338 approximately).

- [ ] **Step 4: Delete `drawFrame()` function entirely**

Remove the entire `drawFrame` function.

- [ ] **Step 5: Remove 2D canvas references**

Delete these lines:

```js
const outCv    = document.getElementById('out-canvas');
const outCtx   = outCv.getContext('2d');
```

- [ ] **Step 6: Update the `loop()` function temporarily**

Replace:

```js
function loop() {
  animId = requestAnimationFrame(loop);
  if (!running) return;
  if (sampleFrame()) { applyGlitch(); drawFrame(); }
}
```

With:

```js
function loop() {
  animId = requestAnimationFrame(loop);
  if (!running) return;
  sampleFrame();
}
```

- [ ] **Step 7: Remove glitch slider event listeners**

Delete these two listeners:

```js
document.getElementById('rate-r').addEventListener('input', e => {
  glitchRate = parseInt(e.target.value);
  document.getElementById('rate-v').textContent = glitchRate;
});

document.getElementById('range-r').addEventListener('input', e => {
  glitchRange = parseInt(e.target.value);
  document.getElementById('range-v').textContent = glitchRange;
});
```

- [ ] **Step 8: Update save button to be a no-op temporarily**

Replace:

```js
document.getElementById('btn-save').addEventListener('click', () => {
  if (!outCv.width) return;
  const a = document.createElement('a');
  a.href = outCv.toDataURL('image/png');
  a.download = 'ascii-cam.png';
  a.click();
});
```

With:

```js
document.getElementById('btn-save').addEventListener('click', () => {
  // updated in Task 5 to use Three.js renderer
});
```

- [ ] **Step 9: Commit**

```bash
git add index.html
git commit -m "Remove glitch system, 2D canvas rendering, and drawFrame"
```

---

### Task 3: Add Motion Detection to sampleFrame()

**Files:**
- Modify: `index.html` (script block)

- [ ] **Step 1: Add motion detection state variables**

After the existing `let targetIdx = [], charColors = [];` line, add:

```js
let prevBrightness = null;
let motionDelta = null;
let sensitivity = 8;
```

- [ ] **Step 2: Add motion detection to `sampleFrame()`**

At the end of `sampleFrame()`, after the existing `for` loop that computes `targetIdx` and `charColors`, and just before `return true;`, add:

```js
  // motion detection
  if (!prevBrightness || prevBrightness.length !== n) {
    prevBrightness = new Float32Array(n);
    motionDelta = new Float32Array(n);
    for (let i = 0; i < n; i++) {
      prevBrightness[i] = brightness(data[i*4], data[i*4+1], data[i*4+2]) / 255;
    }
    return true;
  }

  for (let i = 0; i < n; i++) {
    const d = i * 4;
    const br = brightness(data[d], data[d+1], data[d+2]) / 255;
    motionDelta[i] = Math.abs(br - prevBrightness[i]);
    prevBrightness[i] = br;
  }
```

- [ ] **Step 3: Add sensitivity slider listener**

Add this listener (replacing the removed glitch listeners):

```js
document.getElementById('sens-r').addEventListener('input', e => {
  sensitivity = parseInt(e.target.value);
  document.getElementById('sens-v').textContent = sensitivity;
});
```

- [ ] **Step 4: Reset motion state on charset/cols change**

Update the charset change listener to also reset motion state:

```js
document.getElementById('charset-sel').addEventListener('change', e => {
  charset = CHARSETS[e.target.value];
  targetIdx = [];
  prevBrightness = null;
});
```

Update the cols listener similarly:

```js
document.getElementById('cols-r').addEventListener('input', e => {
  cols = parseInt(e.target.value);
  document.getElementById('cols-v').textContent = cols;
  targetIdx = [];
  prevBrightness = null;
});
```

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Add per-cell motion detection to sampleFrame"
```

---

### Task 4: Three.js Scene Setup and Texture Atlas

**Files:**
- Modify: `index.html` (script block)

- [ ] **Step 1: Add Three.js scene variables**

After the existing DOM reference declarations (after `const btnFlip = ...`), add:

```js
// Three.js state
let renderer, scene, camera, charMesh, dummy;
let atlasTexture, atlasCanvas, atlasCtx;
let charCellW = 32, charCellH = 64;
let charCount = 0;
let instancePositions, instanceVelocities, instanceRotations, instanceAngularVel, instancePhases;
const SPRING_K = 2.0;
const DAMPING = 0.95;
const ANGULAR_DAMPING = 0.92;
const FORCE_MULT = 15.0;
let lastTime = 0;
```

- [ ] **Step 2: Add `buildAtlas()` function**

Add this function after the `brightness` function:

```js
function buildAtlas(chars) {
  const count = chars.length;
  if (!atlasCanvas) {
    atlasCanvas = document.createElement('canvas');
    atlasCtx = atlasCanvas.getContext('2d');
  }
  atlasCanvas.width = charCellW * count;
  atlasCanvas.height = charCellH;
  atlasCtx.clearRect(0, 0, atlasCanvas.width, atlasCanvas.height);
  atlasCtx.fillStyle = '#fff';
  atlasCtx.font = `${charCellH * 0.75}px monospace`;
  atlasCtx.textBaseline = 'middle';
  atlasCtx.textAlign = 'center';
  for (let i = 0; i < count; i++) {
    atlasCtx.fillText(chars[i], charCellW * i + charCellW / 2, charCellH / 2);
  }
  if (atlasTexture) {
    atlasTexture.needsUpdate = true;
  } else {
    atlasTexture = new THREE.CanvasTexture(atlasCanvas);
    atlasTexture.minFilter = THREE.LinearFilter;
    atlasTexture.magFilter = THREE.LinearFilter;
  }
}
```

- [ ] **Step 3: Add `initThree()` function**

Add this function after `buildAtlas`:

```js
function initThree() {
  renderer = new THREE.WebGLRenderer({ antialias: true, preserveDrawingBuffer: true });
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.setClearColor(0x000000);
  renderer.domElement.style.cssText = 'position:fixed;inset:0;z-index:0;display:block;';
  document.body.prepend(renderer.domElement);

  scene = new THREE.Scene();
  camera = new THREE.PerspectiveCamera(50, window.innerWidth / window.innerHeight, 0.1, 1000);
  dummy = new THREE.Object3D();

  buildAtlas(charset);

  window.addEventListener('resize', () => {
    renderer.setSize(window.innerWidth, window.innerHeight);
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    prevBrightness = null;
  });
}
```

- [ ] **Step 4: Call `initThree()` at startup**

In the `DOMContentLoaded` listener, add `initThree()` before `startCam()`:

```js
document.addEventListener('DOMContentLoaded', () => { initThree(); startCam(); });
```

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Add Three.js scene setup and character texture atlas"
```

---

### Task 5: InstancedMesh Creation and Per-Frame Rendering

**Files:**
- Modify: `index.html` (script block)

- [ ] **Step 1: Add `rebuildMesh()` function**

Add after `initThree`:

```js
function rebuildMesh() {
  if (charMesh) { scene.remove(charMesh); charMesh.geometry.dispose(); charMesh.material.dispose(); }
  if (!numCols || !numRows) return;

  const barH = document.querySelector('.bar').offsetHeight;
  const viewH = window.innerHeight - barH;
  const viewW = window.innerWidth;

  // compute world-space character size
  const dist = camera.position.z || 10;
  const vFov = camera.fov * Math.PI / 180;
  const worldH = 2 * Math.tan(vFov / 2) * dist;
  const worldW = worldH * (viewW / viewH);
  const cw = worldW / numCols;
  const ch = cw * 2.0;

  // position camera to fit grid
  const gridH = numRows * ch;
  const gridW = numCols * cw;
  const neededDist = (gridH / 2) / Math.tan(vFov / 2) * 1.05;
  camera.position.set(0, 0, neededDist);
  camera.lookAt(0, 0, 0);

  const n = numCols * numRows;
  charCount = n;

  const geom = new THREE.PlaneGeometry(cw, ch);
  const mat = new THREE.MeshBasicMaterial({
    map: atlasTexture,
    transparent: true,
    alphaTest: 0.1,
    side: THREE.DoubleSide
  });
  charMesh = new THREE.InstancedMesh(geom, mat, n);
  charMesh.instanceMatrix.setUsage(THREE.DynamicDrawUsage);

  // UV offset attribute
  const uvOffsets = new Float32Array(n);
  charMesh.geometry.setAttribute('instanceUvOffset', new THREE.InstancedBufferAttribute(uvOffsets, 1));

  // custom shader injection for UV offset
  mat.onBeforeCompile = (shader) => {
    shader.vertexShader = shader.vertexShader.replace(
      '#include <uv_vertex>',
      `#include <uv_vertex>
       vMapUv.x = vMapUv.x / ${charset.length.toFixed(1)} + instanceUvOffset;`
    );
    shader.vertexShader = shader.vertexShader.replace(
      'void main() {',
      `attribute float instanceUvOffset;\nvoid main() {`
    );
  };

  // per-instance physics arrays
  instancePositions = new Float32Array(n * 3);
  instanceVelocities = new Float32Array(n * 3);
  instanceRotations = new Float32Array(n);
  instanceAngularVel = new Float32Array(n);
  instancePhases = new Float32Array(n);

  const startX = -(gridW - cw) / 2;
  const startY = (gridH - ch) / 2;

  for (let row = 0; row < numRows; row++) {
    for (let col = 0; col < numCols; col++) {
      const i = row * numCols + col;
      const x = startX + col * cw;
      const y = startY - row * ch;
      instancePositions[i * 3] = x;
      instancePositions[i * 3 + 1] = y;
      instancePositions[i * 3 + 2] = 0;
      instancePhases[i] = i * 0.1;

      dummy.position.set(x, y, 0);
      dummy.rotation.set(0, 0, 0);
      dummy.updateMatrix();
      charMesh.setMatrixAt(i, dummy.matrix);
    }
  }

  // store rest positions as a copy
  charMesh.userData.restPositions = new Float32Array(instancePositions);
  charMesh.instanceMatrix.needsUpdate = true;

  // color
  charMesh.instanceColor = new THREE.InstancedBufferAttribute(new Float32Array(n * 3), 3);
  charMesh.instanceColor.setUsage(THREE.DynamicDrawUsage);

  scene.add(charMesh);
}
```

- [ ] **Step 2: Add `updateInstances()` function**

Add after `rebuildMesh`:

```js
function updateInstances(time, dt) {
  if (!charMesh || charCount === 0) return;

  const rest = charMesh.userData.restPositions;
  const uvAttr = charMesh.geometry.getAttribute('instanceUvOffset');
  const charLen = charset.length;
  const sensThreshold = (21 - sensitivity) / 100;

  for (let i = 0; i < charCount; i++) {
    // update UV offset from video sampling
    if (targetIdx.length > i) {
      uvAttr.array[i] = targetIdx[i] / charLen;
    }

    // update color
    if (charColors.length > i * 3) {
      if (colorMode) {
        charMesh.instanceColor.array[i * 3] = charColors[i * 3] / 255;
        charMesh.instanceColor.array[i * 3 + 1] = charColors[i * 3 + 1] / 255;
        charMesh.instanceColor.array[i * 3 + 2] = charColors[i * 3 + 2] / 255;
      } else {
        const ci = targetIdx[i] || 0;
        const br = 1 - ci / Math.max(1, charLen - 1);
        charMesh.instanceColor.array[i * 3] = br;
        charMesh.instanceColor.array[i * 3 + 1] = br;
        charMesh.instanceColor.array[i * 3 + 2] = br;
      }
    }

    // motion forces
    if (motionDelta && motionDelta.length > i && motionDelta[i] > sensThreshold) {
      const force = motionDelta[i] * FORCE_MULT;
      instanceVelocities[i * 3] += (Math.random() - 0.5) * 2 * force;
      instanceVelocities[i * 3 + 1] += (Math.random() - 0.5) * 2 * force;
      instanceVelocities[i * 3 + 2] += (Math.random() - 0.5) * 2 * force;
      instanceAngularVel[i] += (Math.random() - 0.5) * force * 2.0;
    }

    // ambient drift on rest z
    const restZ = Math.sin(time * 0.5 + instancePhases[i]) * 0.15;

    // spring physics
    const rx = rest[i * 3], ry = rest[i * 3 + 1], rz = rest[i * 3 + 2] + restZ;
    const dx = instancePositions[i * 3] - rx;
    const dy = instancePositions[i * 3 + 1] - ry;
    const dz = instancePositions[i * 3 + 2] - rz;

    instanceVelocities[i * 3] += -SPRING_K * dx * dt;
    instanceVelocities[i * 3 + 1] += -SPRING_K * dy * dt;
    instanceVelocities[i * 3 + 2] += -SPRING_K * dz * dt;

    instanceVelocities[i * 3] *= DAMPING;
    instanceVelocities[i * 3 + 1] *= DAMPING;
    instanceVelocities[i * 3 + 2] *= DAMPING;

    instancePositions[i * 3] += instanceVelocities[i * 3] * dt;
    instancePositions[i * 3 + 1] += instanceVelocities[i * 3 + 1] * dt;
    instancePositions[i * 3 + 2] += instanceVelocities[i * 3 + 2] * dt;

    instanceAngularVel[i] *= ANGULAR_DAMPING;
    instanceRotations[i] += instanceAngularVel[i] * dt;

    // update matrix
    dummy.position.set(
      instancePositions[i * 3],
      instancePositions[i * 3 + 1],
      instancePositions[i * 3 + 2]
    );
    dummy.rotation.set(0, 0, instanceRotations[i]);
    dummy.updateMatrix();
    charMesh.setMatrixAt(i, dummy.matrix);
  }

  uvAttr.needsUpdate = true;
  charMesh.instanceColor.needsUpdate = true;
  charMesh.instanceMatrix.needsUpdate = true;
}
```

- [ ] **Step 3: Replace `loop()` with Three.js render loop**

Replace the existing `loop` function:

```js
function loop(timestamp) {
  animId = requestAnimationFrame(loop);
  if (!running) return;

  const time = timestamp * 0.001;
  const dt = Math.min(time - lastTime, 0.05);
  lastTime = time;

  const needsRebuild = sampleFrame();
  if (needsRebuild && (!charMesh || charCount !== numCols * numRows)) {
    rebuildMesh();
  }

  // auto-orbit camera
  if (camera && charMesh) {
    const basePos = charMesh.userData.restPositions ? camera.position.z : 10;
    camera.position.x = Math.sin(time * 0.15) * 0.3;
    camera.position.y = Math.cos(time * 0.1) * 0.2;
    camera.lookAt(0, 0, 0);
  }

  updateInstances(time, dt);
  renderer.render(scene, camera);
}
```

- [ ] **Step 4: Update save button to use renderer**

Replace the save button listener:

```js
document.getElementById('btn-save').addEventListener('click', () => {
  if (!renderer) return;
  const a = document.createElement('a');
  a.href = renderer.domElement.toDataURL('image/png');
  a.download = 'ascii-cam.png';
  a.click();
});
```

- [ ] **Step 5: Update charset change to rebuild atlas and mesh**

Replace the charset listener:

```js
document.getElementById('charset-sel').addEventListener('change', e => {
  charset = CHARSETS[e.target.value];
  targetIdx = [];
  prevBrightness = null;
  buildAtlas(charset);
  if (charMesh) {
    charMesh.material.needsUpdate = true;
    charCount = 0; // force rebuild
  }
});
```

- [ ] **Step 6: Update cols change to trigger mesh rebuild**

Replace the cols listener:

```js
document.getElementById('cols-r').addEventListener('input', e => {
  cols = parseInt(e.target.value);
  document.getElementById('cols-v').textContent = cols;
  targetIdx = [];
  prevBrightness = null;
  charCount = 0; // force rebuild on next frame
});
```

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "Add InstancedMesh rendering with physics and motion-driven forces"
```

---

### Task 6: Final Cleanup and Polish

**Files:**
- Modify: `index.html` (script block, CSS)

- [ ] **Step 1: Add `#atlas-canvas` to hidden helpers CSS**

Change:

```css
#src-canvas, #feed { display: none; }
```

To:

```css
#src-canvas, #feed, #atlas-canvas { display: none; }
```

(This is precautionary — the atlas canvas is created dynamically and not appended to the DOM, but if it ever is, it should be hidden.)

- [ ] **Step 2: Ensure renderer canvas z-index is below bar**

Verify that in `initThree()`, the renderer canvas has `z-index: 0`. The bar has `z-index: 20` and error overlay has `z-index: 10`, so this is already correct.

No code change needed — just verify.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Final cleanup: ensure proper z-index layering"
```

---

### Task 7: Manual Verification and Push

- [ ] **Step 1: Open in browser and verify**

Open `index.html` in a browser (serve with a local server if needed for camera access). Verify:

1. Camera auto-starts, ASCII characters render as floating 3D quads
2. Characters gently drift/undulate when idle (ambient floating)
3. Moving in front of the camera causes characters in motion areas to scatter and tumble
4. Characters spring back to rest after motion stops
5. Camera subtly auto-orbits, creating parallax depth
6. Bottom bar works: charset select, columns slider, sensitivity slider, color, invert, save
7. Sensitivity slider controls how easily motion triggers scatter (higher = more sensitive)
8. Color mode toggles between grayscale and RGB character coloring
9. Invert toggles brightness mapping
10. Save button downloads a PNG
11. Flip button appears only with multiple cameras
12. Error overlay appears if camera is denied
13. Mobile: bar wraps to two rows with safe-area padding

- [ ] **Step 2: Push to remote**

```bash
git push origin main
```
