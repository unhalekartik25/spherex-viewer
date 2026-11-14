# TEST_TARGETS.md

## 1. The five required targets

Only WISE 0855's coordinates are given by the challenge. For the others, **resolve by name** (SIMBAD) and record the
coordinates and proper motions you actually got in `docs/DEVLOG.md`. Never invent coordinates.
Proper-motion values below are approximate recollections to guide expectations, `[TODO-VERIFY]` against SIMBAD/Gaia.

| Target | How to enter | What it is | What a successful result looks like | Suggested start (verify) |
|---|---|---|---|---|
| **WISE 0855** | RA 133.795125, Dec -7.245147 (= 08h55m10.831s, -07d14m42.53s) | Very cold, fast-moving brown dwarf (proper motion ~8"/yr, `[TODO-VERIFY]`) | A faint source that shifts position relative to fixed stars across epochs; use FOV >= 8' because the quoted position may be from an earlier epoch | detector 5 or 6 (coldest objects are brightest at longer wavelengths), FOV 8-10', stack mode helps |
| **Barnard's Star** | name: "Barnard's Star" | Nearby red dwarf, very high proper motion (~10"/yr, `[TODO-VERIFY]`) | A bright star visibly shifting against background stars; must be centered on the **propagated 2026 position** | detector 1-2 (bright in near-IR), FOV 8-10' |
| **Epsilon Indi B** | name: "eps Ind B" or "eps Ind Ba" (try "eps Ind" if it does not resolve) | Pair of brown dwarfs orbiting the star Epsilon Indi A; large proper motion | The brown dwarf pair, and Epsilon Indi A elsewhere in the field, both shifting; may need a larger FOV to see A and B together | try detectors 2-5, FOV 10-15' |
| **RW Aurigae** | name: "RW Aur" | Young variable star (irregular brightness changes) | Little positional motion; **brightness changes between epochs** are visible with the global stretch | detector 1-3, FOV 3-6' |
| **R Draconis** | name: "R Dra" | Mira-type variable star (large periodic brightness changes) | Strong brightness variations between epochs; consider the center-value sparkline | detector 1-3 (bright), FOV 3-6' |

For each target, pass criteria (all must hold on the **deployed** app):
1. Request completes (status `done`) with at least 8 frames, or a clear `NO_COVERAGE` message with a working alternative suggestion.
2. First cached reload takes < 2 s; uncached completes in < ~90 s on the deployed host.
3. Frames are aligned (fixed stars do not jitter by more than ~1 grid pixel).
4. The expected phenomenon is visible (motion for movers, brightness change for variables) in at least one recommended detector setting.
   Record the settings that work in the DEVLOG table (target, detector, FOV, mode, frame count, time to complete, screenshot).
5. The shareable URL for that target reproduces the view.
6. No console errors; no unhandled server exceptions.

## 2. Procedure (manual, run on the deployed URL at the end of every slice from Slice 2 onward)

1. Open the site, click each quick-pick target, wait for the result, press play, take a screenshot.
2. Change detector (D1-D6) and FOV; confirm results or friendly `NO_COVERAGE`.
3. Toggle stack mode for WISE 0855.
4. Copy the share link, open it in a private window; open an embed test page containing the iframe.
5. Repeat once with a bad input (RA=500) and with 10 rapid clicks to see validation and rate-limit messages.

## 3. Automated testing strategy

`pytest.ini` defines the marker `network`; default runs are offline and fast (< 30 s).

### Unit tests (offline) in `backend/tests/`
- `test_schemas.py`: ranges, NaN/inf, enums, extra fields rejected, defaults.
- `test_irsa.py`: ADQL builder output for known inputs; adversarial strings never reach it; URI validation accepts good and rejects bad URIs; cutout URL builder.
- `test_processing.py`: with **synthetic** data: flag masking per level (SOURCE never masked); background subtraction; grid construction
  (size, orientation, pixel scale, `MAX_GRID_PX` cap); reprojection of a synthetic star with a shifted WCS lands on the same grid pixel;
  stacking S/N improves ~sqrt(N); time binning edges; global vs per-frame stretch; NaN -> alpha 0; PNG row 0 is north.
- `test_cache.py`: TTL expiry, key stability, LRU eviction under cap, atomic write, path safety.
- `test_jobs.py`: dedupe, queue limits, timeouts, cancellation, TTL cleanup.
- `test_security.py`: everything in SECURITY.md section 12 (use FastAPI `TestClient`).
- `test_api.py`: routes, status codes, error envelope, headers, rate limits (with a small injectable clock).
- `test_resolver.py`: proper-motion propagation math using a fixed synthetic star (compare against astropy directly).

### Fixtures
- Generate synthetic MEF FITS files in a pytest fixture (IMAGE/FLAGS/VARIANCE/ZODI/WCS-WAVE with a simple TAN WCS). No real data needed.
- Commit **one** tiny real cutout (< 300 KB) captured in Slice 1 as `backend/tests/data/sample_cutout.fits` to validate the parser on real structure.
- Mock `httpx` and TAP with recorded small responses (`respx`-style mocking is NOT installed; write a tiny fake transport or monkeypatch).

### Network tests (`@pytest.mark.network`, run manually)
- TAP query for WISE 0855 detector 5 returns rows and valid URIs.
- One real cutout download parses and yields a plausible wavelength within the detector's band.
- End-to-end pipeline for WISE 0855 with `max_frames=6`.

### Target runner `scripts/run_targets.py`
Runs the five targets via the pipeline (default) or against `--base-url`; prints a table: target, detector, frames, failures, seconds, cache hit;
writes contact sheets to `cache/`; exits non-zero if a target fails criteria 1-2.

### Frontend checks
Keep logic that can be unit-tested (coordinate parsing, URL-param validation) in plain modules. If no JS test runner is available, document the manual
test cases (inputs and expected outputs) in DEVLOG and keep a `frontend/tests.html` page that runs assertions in the browser and prints PASS/FAIL.
Include a grep-based test (pytest) that fails if `innerHTML`, `outerHTML`, `insertAdjacentHTML`, or `document.write` appear in `frontend/*.js`.

## 4. Performance checks (record numbers in DEVLOG)
- Time for TAP, for N cutout downloads, for reproject+stack, for render, for the whole job.
- Peak memory during a 60-frame, 6-arcmin job and a 150-frame, 30-arcmin job (log RSS).
- Fraction of failed downloads and whether retries help.
