# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

ASCII Cam is a single-file browser application (`ascii-cam.html`) that converts a live webcam feed into real-time ASCII art rendered on a canvas. No build system, no dependencies, no framework — just one self-contained HTML file with inline CSS and JavaScript.

## Development

Open `ascii-cam.html` directly in a browser. No build step or server required (though a local server is needed if testing camera permissions on some browsers).

## Architecture

The entire app lives in `ascii-cam.html` with three inline sections:

- **CSS** (`<style>`): Dark theme with CSS custom properties (`--bg`, `--accent`, etc.). Layout uses flexbox with a `.shell` container, header, canvas area, and controls panel.
- **HTML**: Header with status indicator, a display area (`#out-canvas`), and a controls panel with buttons/sliders/select for configuration.
- **JavaScript** (`<script>`): All logic is in the global scope with no modules.

### Rendering Pipeline

1. **`startCam()`** — Requests webcam via `getUserMedia`, starts the `requestAnimationFrame` loop.
2. **`sampleFrame()`** — Downscales video frame to a grid (`numCols × numRows`) using a hidden `#src-canvas`, computes per-cell brightness → character index, stores RGB for color mode.
3. **`applyGlitch()`** — Randomly offsets character indices with configurable rate/range and TTL-based decay.
4. **`drawFrame()`** — Renders ASCII characters onto `#out-canvas` with per-character color (mono brightness or RGB color mode).
5. **`loop()`** — `requestAnimationFrame` driver calling sample → glitch → draw each frame.

### Key State Variables

- `cols` — ASCII grid width (30–140, default 72)
- `charset` — Active character ramp (dense/blocks/minimal via `CHARSETS` object)
- `colorMode` / `inverted` — Toggle color vs mono, invert brightness
- `glitchRate` / `glitchRange` — Control random character displacement
- `facingMode` — Front/rear camera toggle
- Typed arrays (`targetIdx`, `displayIdx`, `glitchTTL`, `charColors`) — Per-cell rendering state, resized when grid dimensions change

### Front Camera Mirroring

The source canvas horizontally flips the image when `facingMode === 'user'` so the preview feels natural.
