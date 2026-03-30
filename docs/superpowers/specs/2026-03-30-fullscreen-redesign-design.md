# Fullscreen Redesign — ASCII Cam

## Summary

Redesign the ASCII Cam UI so the ASCII visualization fills the entire browser window and all controls collapse into a single thin, frosted-glass bar pinned to the bottom of the viewport. The camera auto-starts on page load.

## Layout

- The output canvas (`#out-canvas`) becomes `position: fixed; inset: 0` filling the full viewport.
- Remove the `.shell` flex container, `.header`, `.canvas-wrap`, and `.controls` wrapper. Replace with a flat structure: canvas + bottom bar.
- The bottom bar is `position: fixed; bottom: 0; left: 0; right: 0`, approximately 40px tall.
- Bottom bar styling: `background: rgba(10, 10, 10, 0.6); backdrop-filter: blur(12px);` — no top border, no hard edge.

## Bottom Bar Contents

Single horizontal row using flexbox, vertically centered, with horizontal padding ~14px and gap ~12px:

| Section | Contents |
|---------|----------|
| **Left** | Logo: "ascii**cam**" — same accent-colored "cam", font-size ~10px, `letter-spacing: 0.12em`, uppercase |
| **Center** | Charset `<select>`, columns slider + value, glitch rate slider + value, glitch range slider + value — all compact inline. Sliders ~60–80px wide. Labels hidden (sliders are self-explanatory at this size; tooltip or `title` attribute provides the name). |
| **Right** | Buttons: flip (conditional), color, invert, save — small pill-style buttons, abbreviated text, ~28px tall |

Center section uses `flex: 1` and `justify-content: center` with `gap: 10px`. Right section uses `flex-shrink: 0`.

## Auto-Start Behavior

- On `DOMContentLoaded`, call `startCam()` immediately.
- Remove the idle message (`#idle-msg`) and the start/stop button entirely.
- If `getUserMedia` is denied, display a centered overlay message on the canvas: "camera access required" in muted text, no retry button (user refreshes to retry).
- The flip button appears only when `enumerateDevices` reports more than one `videoinput` (unchanged).

## Canvas Sizing

- `drawFrame()` must size the output canvas to fill the viewport: compute character grid dimensions from `window.innerWidth` and `window.innerHeight` rather than from the current `cols` slider alone.
- The `cols` slider now controls column count as before (30–140), and rows are derived from viewport aspect ratio divided by character cell aspect ratio (~2.1).
- On `window.resize`, recalculate grid dimensions on the next frame.

## Mobile Responsive

- On viewports narrower than 600px, the bottom bar wraps to two rows (~70px total): top row for sliders/select, bottom row for logo + buttons.
- All touch targets remain at least 36px.
- Sliders shrink to ~50px wide on mobile but remain functional.

## What Does Not Change

- `CHARSETS` object and character ramps.
- `sampleFrame()` video-to-grid sampling logic (downscale, mirror front camera).
- `applyGlitch()` glitch system (rate, range, TTL decay).
- `drawFrame()` per-character rendering loop (color mode, mono mode, glitch highlight).
- `loop()` requestAnimationFrame driver.
- Save button behavior (canvas `toDataURL` → download link).
- Color/invert toggle logic.

## Removed Elements

- `.shell` container, `.header`, `#idle-msg`, `#btn-cam` (start/stop), `.canvas-wrap`, `.controls` wrapper.
- Status text (`#status`) — no longer needed since camera auto-starts.
- All `ctrl-label` elements (slider labels) — replaced by `title` attributes.

## File Structure

Remains a single `index.html` file with inline CSS and JS. No new files.
