# FRONTEND_SPEC.md

Plain HTML + CSS + JavaScript ES modules, no framework, no build step. Files in `frontend/`:
`index.html`, `style.css`, `app.js` (entry), and small modules `api.js`, `url.js`, `player.js`, `ui.js`, `aladin.js` (Slice 4).
Served by FastAPI at `/`. Follow SECURITY.md section 7 (no `innerHTML`, validated URL params, pinned third-party script).

## 1. Layout

Single page, two regions:
1. **Control panel** (left on wide screens, top on narrow): target input, options, "Show sky" button, status/progress.
2. **Viewer** (main): the blink canvas with overlays, playback bar, frame info line. In Slice 4 an Aladin Lite panel can be
   toggled next to or below it.

Header: title + one-line explanation ("See how the sky changes: SPHEREx infrared images played like a video").
Footer: attribution (PROJECT_BRIEF.md section 7), privacy note, link to the GitHub repo, "data through <date>" from `/meta`.
Responsive down to ~360 px wide. Dark theme by default (sky images look better); respect `prefers-color-scheme` if cheap.

## 2. Target input

- One text field accepting: an object name ("Barnard's Star"), or coordinates as `RA Dec` in decimal degrees
  (`133.795125 -7.245147`) or sexagesimal (`08h55m10.831s -07d14m42.53s`, `08:55:10.831 -07:14:42.53`).
- Parsing happens in the browser (`url.js`/`ui.js`): numbers -> `{ra, dec}`; otherwise treat as a name and call `/resolve`.
- Separate numeric RA and Dec fields are shown/filled after parsing (so users can see and edit exact values).
- Quick-pick buttons for the five challenge targets (WISE 0855, Barnard's Star, Epsilon Indi B, RW Aur, R Dra) with their
  recommended detector/FOV from TEST_TARGETS.md. These make the demo one-click.
- If `/resolve` returns `ra_now/dec_now`, show a small note "Proper motion applied to 2026" with a toggle to use the catalog position instead.

## 3. Options (all have tooltips in plain language)

| Control | Values | Default | Maps to |
|---|---|---|---|
| Field of view | slider 1-30 arcmin (+ numeric) | 6 | `fov_arcmin` |
| Detector / band | six buttons D1-D6 showing wavelength range | D2 | `detector` |
| Mode | "Each exposure" / "Stacked by time" | each exposure | `mode` |
| Stack window (days) | shown only for stack | 30 | `bin_days` |
| Background | none / median / zodi model | median | `background` |
| Bad-pixel masking | minimal / strict | minimal | `mask_level` |
| Brightness scale | same for all frames / per frame | same for all | `stretch` |
| Zoom (upsample) | 1x-4x | 2x | `upsample` |
| Max frames | 10-150 | 60 | `max_frames` |

Put the rarely-used ones under an "Advanced" disclosure. Show the detector's wavelength range and a one-line hint
("Cool brown dwarfs are brightest in D4-D6").

## 4. Request flow

1. Validate inputs (same ranges as API). Show inline errors; do not call the API with invalid input.
2. `POST /api/v1/sequences`. If `200` and `status=done`, go to step 4. If `202`, poll.
3. Poll `GET /api/v1/sequences/{key}` every 1.5 s (back off to 3 s after 30 s). Show stage names in plain language
   (query -> "Finding images covering this spot", download -> "Downloading cutouts 12/40", process -> "Aligning images",
   render -> "Making frames") and a progress bar. Provide a Cancel button (stops polling; server job keeps running/caches).
4. On done: preload every frame URL into `Image`/`ImageBitmap` objects (show "Loading frames n/N"), then show the viewer paused on frame 0
   (autoplay only if the user has not enabled reduced motion).
5. Errors: show the `error.message`, a "Try again" button, and for `NO_COVERAGE` suggest another detector/larger FOV.
   For `RATE_LIMITED`/`BUSY`: show a countdown from `Retry-After`.

## 5. Player

- Canvas sized to the grid aspect ratio; `image-rendering: pixelated` (nearest-neighbour) by default with a "Smooth" toggle.
- Controls: Play/Pause, Step back/forward, speed (1-15 fps, default 4), scrub slider, Loop on/off, "Bounce" (ping-pong) toggle.
- **Keyboard** (when the viewer has focus): Space = play/pause, Left/Right = step, `[` / `]` = slower/faster, `L` = loop, `F` = fullscreen.
- **Info line** for the current frame: date (ISO, UTC), wavelength at target (um) or "n/a", detector, exposures in frame, flagged-pixel %,
  processing version, frame i / N. All inserted with `textContent`.
- **Overlays** (toggle buttons): center crosshair marking the requested target; scale bar (arcmin) computed from `grid.pixscale_arcsec`;
  compass (N up, E left); grid/coordinate readout on mouse hover (RA/Dec computed from `grid.wcs` with the TAN formula, tolerance acceptable at small FOV).
- Click on the canvas pins a position and shows its RA/Dec (copy button). (Hook for the stretch spectrum feature.)
- Fixed field: because the grid is fixed on the sky, stars stay put and movers visibly shift. Do not re-center between frames.
- Optional **blink A/B**: select two frames and alternate between them (the classic technique for spotting movers).

## 6. URL parameters (shareable + embeddable)

Read on load via `URLSearchParams`; update with `history.replaceState` after a successful request. If `ra` and `dec` (or `name`) are present, run automatically.

| Param | Meaning | Example |
|---|---|---|
| `ra`, `dec` | decimal degrees | `ra=133.795125&dec=-7.245147` |
| `name` | object name (resolved) | `name=Barnard%27s%20Star` |
| `fov` | arcmin | `fov=8` |
| `det` | detector 1-6 | `det=5` |
| `mode` | `frames` / `stack` | `mode=stack` |
| `bin` | stack days | `bin=30` |
| `bg`, `mask`, `stretch`, `up`, `max` | advanced options | `bg=median` |
| `fps` | playback speed | `fps=6` |
| `frame` | initial frame index | `frame=0` |
| `play` | `1` to autoplay (ignored if reduced motion) | `play=1` |
| `embed` | `1` hides header/footer/options for iframes (keep a small "Open full tool" link and the attribution line) | `embed=1` |

Invalid values are ignored (fall back to defaults) and never rendered as HTML. Provide a "Copy share link" and an
"Embed" button that shows an `<iframe src="...&embed=1" width="640" height="640">` snippet (rendered as text).

## 7. Metadata / quality display

- Under the viewer: a compact table or list of frames (time, wavelength, exposures, flagged %), click to jump to a frame.
- Warnings from `result.warnings` shown in a dismissible notice (e.g. "6 downloads failed; showing 54 of 60 frames").
- Truncation notice when `result.truncated` ("Showing 60 of 143 exposures, evenly spaced in time").
- "Data through <date>" in the footer from `/meta.latest_observation_iso`.
- Optional (Slice 3): sparkline of `center_value` vs time with the caveat "values mix wavelengths".

## 8. Aladin Lite panel and Gaia overlay (Slice 4)

- Toggle "Sky context". Loads Aladin Lite (pinned URL, SRI) lazily on first open. Center on the target; FOV = request FOV.
- Backdrop survey: an all-sky optical or infrared HiPS from CDS (pick one that Aladin lists by default; record the id in DEVLOG).
- Catalog layer: Gaia DR3 via Aladin's own VizieR support, **or** draw our own overlay on the player canvas from `/catalog/gaia`:
  for each source, compute its position at the current frame's epoch `pos + pm*(t - 2016.0)`, draw a small circle, and
  optionally an arrow showing 1-year motion scaled up. Toggle on/off; cap to ~300 drawn sources.
  The canvas overlay (our data) is the primary feature; the Aladin catalog is a bonus.
- Keep Aladin and the player in sync only on the center position; do not try to animate Aladin.

## 9. Accessibility and polish

- All controls are real `<button>`/`<input>`/`<select>` elements with labels; ARIA labels on icon buttons; visible focus outline.
- `prefers-reduced-motion`: never autoplay; keep manual stepping.
- Color contrast AA; do not rely on color alone.
- Loading, empty, and error states for every async action. No console errors in normal use.
- `<title>` and meta description; Open Graph tags optional.

## 10. Frontend acceptance checklist

- Works with only `ra`/`dec` in the URL; works with `embed=1` inside an iframe.
- No `innerHTML` anywhere; all URL params validated.
- Frame preloading prevents flicker; playback stays smooth at 10 fps for 60 frames of 320x320.
- Every error path shows a human-readable message and a way to retry.
- Lighthouse/DevTools shows no mixed content, no blocked CSP resources, no uncaught errors.
