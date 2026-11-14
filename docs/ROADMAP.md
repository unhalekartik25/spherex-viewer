# ROADMAP.md

Work **strictly in this order**. Tick boxes only when the acceptance criteria pass (run them, do not assume).
Time budget is ~24-36 working hours total; the hour marks are guides, not deadlines. If a slice runs over by 50%,
cut the lowest-priority items inside it and move on. The cut-list at the bottom says what to drop first.

---

## Slice 0: Skeleton, tests, and a live URL  (target: hours 0-3)

Build:
- `backend/__init__.py`, `backend/app/__init__.py`, `main.py`, `config.py`, `errors.py`, `security.py` (security headers + request-id
  middleware only for now), `api.py` with `GET /api/v1/health`, `GET /api/v1/meta` (static part), static mount of `frontend/`.
- `frontend/index.html` placeholder page with title and attribution footer.
- `pytest.ini` (marker `network`), a first test (health returns ok, headers present).
- `.env.example`, `Dockerfile`, `.dockerignore`, `scripts/smoke_deploy.py`, README stub.
- Deploy to the chosen host.

Acceptance:
- [ ] `python -m pytest -q` passes; server runs locally in Codespaces and `/api/v1/health` returns ok.
- [ ] Security headers from SECURITY.md section 6 present (CSP initially `self` only).
- [ ] Public URL serves the page and health check; `smoke_deploy.py <URL> --deep` reports IRSA reachable from the host (DATA_SOURCES check #9).
- [ ] DEVLOG entry: host chosen, Python version in the image, RAM limit, port.

## Slice 1: Core loop, one target end to end  (hours 3-12)  **MVP CORE**

Build (backend first, no UI polish):
- `schemas.py` (`SequenceRequest` with all parameters and limits), `irsa.py` (ADQL builder, validated URI handling, TAP cache),
  `cutouts.py` (download with allowlist/timeouts/retries/size cap, FITS parsing, spectral WCS wavelength), `processing.py`
  (mask, background, grid, reproject, stretch), `render.py`, `cache.py`, `jobs.py`, `pipeline.py`, routes
  `POST /sequences`, `GET /sequences/{key}`, `GET /sequences/{key}/frames/{i}.png`.
- A scratch script `scripts/run_targets.py` that runs the pipeline directly (no HTTP) for WISE 0855 and prints counts and timings, and saves a
  contact sheet PNG (`cache/contact_<target>.png`) for eyeballing.
- Minimal frontend: coordinate fields, detector select, FOV, "Show sky", progress text, a canvas that plays frames with Play/Pause.
- Resolve DATA_SOURCES checks #1-#5 and record the answers in DEVLOG (fix DATA_SOURCES.md where reality differs).

Acceptance:
- [ ] WISE 0855 (`133.795125, -7.245147`, FOV 8, detector 5 or 6) returns >= 10 frames in the deployed app.
- [ ] Frames are aligned: a fixed star stays within ~1 grid pixel across frames (check on the contact sheet).
- [ ] Row 0 of PNGs is north; N up / E left verified against a known pair of stars or the WCS (write a unit test with a synthetic WCS).
- [ ] Failed downloads are retried, then skipped and counted; one bad file never fails the job.
- [ ] Re-running the same request is served from cache in < 2 s.
- [ ] Limits enforced (FOV, frames, timeouts, size caps); validation and SSRF/path tests from SECURITY.md section 12 pass.
- [ ] Rate limiting and job dedupe working (tests).
- [ ] DATA_SOURCES checks #1-#5 answered and documented.

## Slice 2: Usability and sharing  (hours 12-18)

Build:
- Full player (speed, scrub, step, loop, bounce, keyboard, info line), overlays (crosshair, scale bar, compass, hover coords).
- Options panel per FRONTEND_SPEC.md, plain-language labels and tooltips; progress stages; error/retry states.
- `GET /resolve` (SIMBAD with proper motion and propagation to `TARGET_EPOCH`; fallback to `SkyCoord.from_name`), name input,
  five quick-pick buttons, sexagesimal parsing.
- URL parameters + `embed=1` + "Copy share link" + "Embed" snippet.
- `GET /meta` latest observation time; footer "data through".
- Tests: parser unit tests (frontend logic may be tested via a small Node-free approach: keep parsing in a plain module and test
  with `python`-side equivalents if no JS runner; otherwise document manual test steps in DEVLOG).

Acceptance:
- [ ] All five targets can be loaded by name or quick-pick in under 3 clicks (TEST_TARGETS.md procedure).
- [ ] Barnard's Star by name is centered on its **propagated** position and the star is inside the field (DATA_SOURCES section 6).
- [ ] `/?ra=133.795125&dec=-7.245147&fov=8&det=5` auto-runs; the same URL with `embed=1` works inside an iframe test page.
- [ ] No `innerHTML` in `frontend/` (grep test); invalid URL params ignored safely.
- [ ] Keyboard control works; reduced-motion respected.

## Slice 3: Stacking and metadata  (hours 18-24)

Build:
- `mode=stack` with time bins and inverse-variance weighted co-add (`processing.stack_bins`), per-detector (D1-D6) selector with
  wavelength hints.
- Frame table with flagged %, exposures, processing version; truncation and warning notices.
- Optional: `center_value` sparkline for variable stars (RW Aur, R Dra) with the "mixes wavelengths" caveat.
- Optional: `frames/{i}/flags.png` mask overlay.

Acceptance:
- [ ] Unit test: stacking synthetic frames with known noise improves S/N by about sqrt(N) (tolerance documented).
- [ ] Stack mode on WISE 0855 (detector 5 or 6) shows a visibly cleaner image than single exposures; motion between bins is still visible (record screenshots).
- [ ] Metadata table matches API values; truncation message correct when `n_available > max_frames`.

## Slice 4: The one extra feature, Gaia overlay + Aladin context  (hours 24-30)

Build:
- `catalogs.py` + `GET /catalog/gaia` (DATA_SOURCES section 7; resolve checks #7, #8).
- Canvas overlay of Gaia sources propagated to each frame's epoch, optional proper-motion arrows, toggle; cap on drawn sources.
- Aladin Lite panel (lazy-loaded, pinned, SRI), CSP widened minimally.

Acceptance:
- [ ] For Barnard's Star the overlay marker follows the star's motion between frames (visual check, screenshot in DEVLOG).
- [ ] CSP has no blocked resources in the browser console; only the pinned Aladin origin(s) added.
- [ ] If Gaia/Aladin fails, the rest of the app still works and shows a friendly message.

If Slice 4 is at risk, switch to the backup: image add/subtract (difference of two selected frames/stacks), which needs no new external service.

## Slice 5: Harden, deploy, document  (hours 30-36; reserve at least 4 hours; do not spend this time elsewhere)

Do:
- Run `scripts/run_targets.py --base-url <deployed>`; fix failures. Check all items in `TEST_TARGETS.md`.
- Seed/pre-warm: `scripts/warm_cache.py`; optional `demo_data/` seeding (DEPLOYMENT.md section 5).
- Walk through the SECURITY.md MUST list and tick each item in DEVLOG.
- README with screenshots/GIF, usage, architecture diagram (copy the one from ARCHITECTURE.md), "how updates work", limitations, credits.
- Record the demo video; verify the public link from another device; re-read the official submission requirements.

Acceptance:
- [ ] Everything in DEPLOYMENT.md section 7 release checklist ticked.
- [ ] Final commit tagged `v1.0-submission`.

---

## Priority / cut-list (drop from the top first if time is short)

1. Aladin Lite panel (keep the canvas Gaia overlay or the add/subtract backup)
2. Sparkline, flags overlay, A/B blink mode, bounce mode
3. Spectrum endpoint, pre-seeded demo data
4. Stack mode polish (keep a basic stack)
5. Never cut: the core loop, rate limiting/limits, validation, deploy, README, demo video, all five targets verified.

## Parking lot (do NOT do)
Accounts, saving views, object detection/ML, database, mosaics, mobile app, multi-detector joint stacks, custom colormaps beyond grayscale + optional invert.
