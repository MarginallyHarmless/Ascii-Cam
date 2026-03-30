# Fullscreen Redesign Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Redesign ASCII Cam so the visualization fills the entire viewport with a subtle frosted-glass control bar at the bottom, auto-starting the camera on load.

**Architecture:** Single-file rewrite of `index.html`. The HTML body becomes just a fullscreen canvas, a bottom bar overlay, hidden helper elements, and an error overlay. CSS is replaced entirely. JS rendering logic stays intact; layout, startup, and DOM references change.

**Tech Stack:** Vanilla HTML/CSS/JS (unchanged), CSS `backdrop-filter` for frosted glass.

---

### Task 1: Replace HTML Structure

**Files:**
- Modify: `index.html:219-293` (HTML body)

- [ ] **Step 1: Replace the entire `<body>` content**

Remove the `.shell` wrapper, `.header`, `.canvas-wrap`, `.controls`, `#idle-msg`, and `#btn-cam`. Replace with this flat structure:

```html
<body>

<canvas id="out-canvas"></canvas>

<div class="error-overlay" id="error-overlay">camera access required</div>

<div class="bar">
  <div class="bar-left">
    <div class="logo">ascii<span>cam</span></div>
  </div>
  <div class="bar-center">
    <select id="charset-sel" title="charset">
      <option value="dense">dense</option>
      <option value="blocks">blocks</option>
      <option value="minimal">minimal</option>
    </select>
    <input type="range" id="cols-r" min="30" max="140" value="72" step="2" title="columns" />
    <span class="val" id="cols-v">72</span>
    <input type="range" id="rate-r" min="0" max="100" value="18" step="1" title="glitch rate" />
    <span class="val" id="rate-v">18</span>
    <input type="range" id="range-r" min="1" max="8" value="2" step="1" title="glitch range" />
    <span class="val" id="range-v">2</span>
  </div>
  <div class="bar-right">
    <button class="btn" id="btn-flip" style="display:none;">flip</button>
    <button class="btn" id="btn-color">mono</button>
    <button class="btn" id="btn-inv">inv</button>
    <button class="btn" id="btn-save">save</button>
  </div>
</div>

<canvas id="src-canvas"></canvas>
<video id="feed" playsinline muted autoplay></video>

</body>
```

- [ ] **Step 2: Commit**

```bash
git add index.html
git commit -m "Replace HTML structure with fullscreen layout"
```

---

### Task 2: Replace CSS

**Files:**
- Modify: `index.html:7-217` (entire `<style>` block)

- [ ] **Step 1: Replace the entire `<style>` block**

```css
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

:root {
  --bg: #0a0a0a;
  --border: #2a2a2a;
  --text: #c8c8c8;
  --muted: #555;
  --accent: #e8734a;
  --font: 'IBM Plex Mono', 'Courier New', monospace;
}

html, body {
  background: #000;
  color: var(--text);
  font-family: var(--font);
  font-size: 13px;
  height: 100%;
  overflow: hidden;
}

/* ---------- fullscreen canvas ---------- */
#out-canvas {
  position: fixed;
  inset: 0;
  width: 100vw;
  height: 100vh;
  display: block;
}

/* ---------- error overlay ---------- */
.error-overlay {
  display: none;
  position: fixed;
  inset: 0;
  z-index: 10;
  background: rgba(0,0,0,0.85);
  color: var(--muted);
  font-size: 13px;
  letter-spacing: 0.08em;
  justify-content: center;
  align-items: center;
  text-align: center;
}

.error-overlay.visible {
  display: flex;
}

/* ---------- bottom bar ---------- */
.bar {
  position: fixed;
  bottom: 0;
  left: 0;
  right: 0;
  z-index: 20;
  height: 40px;
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 0 14px;
  background: rgba(10, 10, 10, 0.6);
  backdrop-filter: blur(12px);
  -webkit-backdrop-filter: blur(12px);
}

.bar-left {
  flex-shrink: 0;
}

.logo {
  font-size: 10px;
  letter-spacing: 0.12em;
  color: var(--muted);
  text-transform: uppercase;
  white-space: nowrap;
}

.logo span { color: var(--accent); }

.bar-center {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 10px;
  min-width: 0;
}

.bar-right {
  flex-shrink: 0;
  display: flex;
  gap: 6px;
}

/* ---------- controls ---------- */
select {
  background: rgba(10, 10, 10, 0.5);
  border: 1px solid var(--border);
  border-radius: 4px;
  color: var(--text);
  font-family: var(--font);
  font-size: 10px;
  padding: 4px 6px;
  -webkit-appearance: none;
  outline: none;
  max-width: 80px;
}

input[type=range] {
  -webkit-appearance: none;
  width: 60px;
  height: 2px;
  background: var(--border);
  border-radius: 1px;
  outline: none;
  flex-shrink: 1;
  min-width: 30px;
}

input[type=range]::-webkit-slider-thumb {
  -webkit-appearance: none;
  width: 14px;
  height: 14px;
  border-radius: 50%;
  background: var(--text);
  cursor: pointer;
}

input[type=range]::-webkit-slider-thumb:active {
  background: var(--accent);
}

.val {
  font-size: 10px;
  color: var(--muted);
  min-width: 16px;
  text-align: right;
}

/* ---------- buttons ---------- */
.btn {
  padding: 5px 10px;
  border: 1px solid var(--border);
  border-radius: 4px;
  background: rgba(10, 10, 10, 0.4);
  color: var(--text);
  font-family: var(--font);
  font-size: 10px;
  letter-spacing: 0.04em;
  cursor: pointer;
  transition: border-color 0.1s, color 0.1s;
  -webkit-tap-highlight-color: transparent;
  touch-action: manipulation;
  white-space: nowrap;
  height: 28px;
}

.btn:active { background: var(--border); }

.btn.active {
  border-color: var(--accent);
  color: var(--accent);
}

/* hidden helpers */
#src-canvas, #feed { display: none; }

/* ---------- mobile ---------- */
@media (max-width: 600px) {
  .bar {
    height: auto;
    min-height: 40px;
    flex-wrap: wrap;
    padding: 8px 10px;
    gap: 8px;
  }

  .bar-left {
    order: 1;
  }

  .bar-center {
    order: 3;
    flex-basis: 100%;
    justify-content: flex-start;
  }

  .bar-right {
    order: 2;
    margin-left: auto;
  }

  input[type=range] {
    width: 50px;
    min-width: 30px;
  }
}
```

- [ ] **Step 2: Commit**

```bash
git add index.html
git commit -m "Replace CSS with fullscreen layout and frosted-glass bar"
```

---

### Task 3: Update JavaScript — DOM References and Auto-Start

**Files:**
- Modify: `index.html` (the `<script>` block)

- [ ] **Step 1: Remove stale DOM references and add new ones**

Remove these lines:
```js
const statusEl = document.getElementById('status');
const idleMsg  = document.getElementById('idle-msg');
const btnCam   = document.getElementById('btn-cam');
```

Add this line after the `outCtx` declaration:
```js
const errorOverlay = document.getElementById('error-overlay');
```

Remove the `setStatus` function entirely.

- [ ] **Step 2: Rewrite `startCam()` for auto-start**

Replace the entire `startCam` function:

```js
async function startCam() {
  if (stream) { stream.getTracks().forEach(t => t.stop()); stream = null; }
  try {
    stream = await navigator.mediaDevices.getUserMedia({
      video: { facingMode: { ideal: facingMode }, width: { ideal: 640 } },
      audio: false
    });
    feed.srcObject = stream;
    await feed.play();
    running = true;
    const devices = await navigator.mediaDevices.enumerateDevices();
    const camCount = devices.filter(d => d.kind === 'videoinput').length;
    btnFlip.style.display = camCount > 1 ? '' : 'none';
    if (!animId) loop();
  } catch(e) {
    errorOverlay.classList.add('visible');
    console.error(e);
  }
}
```

- [ ] **Step 3: Remove `stopCam()` and start/stop button listener**

Delete the entire `stopCam` function.

Delete this listener:
```js
btnCam.addEventListener('click', () => {
  if (running) stopCam(); else startCam();
});
```

- [ ] **Step 4: Update `drawFrame()` to fill viewport**

Replace the top of `drawFrame` (the sizing calculations) so the canvas always fills the window. Replace the entire function:

```js
function drawFrame() {
  const W = window.innerWidth;
  const H = window.innerHeight;

  // derive font size and cell dimensions from cols to fill width
  const cw = W / numCols;
  const fs = cw / 0.62;
  const ch = fs * 1.25;

  if (outCv.width !== W || outCv.height !== H) {
    outCv.width = W; outCv.height = H;
  }

  outCtx.fillStyle = '#000';
  outCtx.fillRect(0, 0, W, H);
  outCtx.font = `${fs}px monospace`;
  outCtx.textBaseline = 'top';

  const cs = charset;
  for (let row = 0; row < numRows; row++) {
    for (let col = 0; col < numCols; col++) {
      const i = row * numCols + col;
      const ci = displayIdx[i];
      const ch_char = cs[ci];
      if (!ch_char || ch_char === ' ') continue;

      const glitching = glitchTTL[i] > 0;
      if (colorMode) {
        const r = charColors[i*3], g = charColors[i*3+1], b = charColors[i*3+2];
        outCtx.fillStyle = glitching
          ? `rgb(${Math.min(255,r+50)},${g},${b})`
          : `rgb(${r},${g},${b})`;
      } else {
        const br = 1 - ci / (cs.length - 1);
        const lv = Math.round(br * 255);
        outCtx.fillStyle = glitching
          ? `rgb(${Math.min(255,lv+70)},${lv},${lv})`
          : `rgb(${lv},${lv},${lv})`;
      }
      outCtx.fillText(ch_char, col * cw, row * ch);
    }
  }
}
```

- [ ] **Step 5: Update `sampleFrame()` to derive rows from viewport**

Replace the row calculation in `sampleFrame`. Change:
```js
  const aspect = sh / sw;
  numCols = cols;
  numRows = Math.max(1, Math.round(cols * aspect / 2.1));
```

To:
```js
  numCols = cols;
  const cw = window.innerWidth / numCols;
  const fs = cw / 0.62;
  const ch = fs * 1.25;
  numRows = Math.max(1, Math.floor(window.innerHeight / ch));
```

This ensures the character grid exactly fills the viewport vertically regardless of video aspect ratio.

- [ ] **Step 6: Add auto-start on DOMContentLoaded**

At the very end of the `<script>` block, add:

```js
document.addEventListener('DOMContentLoaded', () => { startCam(); });
```

- [ ] **Step 7: Update flip button listener**

Replace the flip listener. Change:
```js
btnFlip.addEventListener('click', () => {
  facingMode = facingMode === 'user' ? 'environment' : 'user';
  btnFlip.textContent = facingMode === 'user' ? 'flip' : 'rear';
  if (running) startCam();
});
```

To:
```js
btnFlip.addEventListener('click', () => {
  facingMode = facingMode === 'user' ? 'environment' : 'user';
  btnFlip.textContent = facingMode === 'user' ? 'flip' : 'rear';
  startCam();
});
```

- [ ] **Step 8: Update invert button text**

Change the invert button listener so the default text matches the new HTML (`inv` instead of `off`):

Replace:
```js
document.getElementById('btn-inv').addEventListener('click', function() {
  inverted = !inverted;
  this.textContent = inverted ? 'on' : 'off';
  this.classList.toggle('active', inverted);
});
```

With:
```js
document.getElementById('btn-inv').addEventListener('click', function() {
  inverted = !inverted;
  this.textContent = inverted ? 'inv!' : 'inv';
  this.classList.toggle('active', inverted);
});
```

- [ ] **Step 9: Commit**

```bash
git add index.html
git commit -m "Update JS for auto-start, fullscreen canvas, remove start/stop"
```

---

### Task 4: Manual Verification

- [ ] **Step 1: Open in browser and verify**

Open `index.html` in a browser. Verify:
1. Camera auto-starts, ASCII art fills entire viewport
2. Bottom bar is visible, frosted glass effect works
3. All controls function: charset select, column slider, glitch rate, glitch range, color, invert, save
4. Flip button only appears with multiple cameras
5. On narrow viewport (<600px), bar wraps to two rows
6. If camera is denied, error overlay appears

- [ ] **Step 2: Push to remote**

```bash
git push origin main
```
