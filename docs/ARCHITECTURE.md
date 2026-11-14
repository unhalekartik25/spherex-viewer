# ARCHITECTURE.md

## 1. Overview

One Python process (FastAPI) serves both the JSON API and the static frontend. The browser never talks
to IRSA; the server does all astronomy work and returns small PNG frames plus JSON metadata.

```
Browser (HTML/JS)  --HTTP-->  FastAPI app  --TAP/HTTPS-->  IRSA (TAP + cutout service)
   |  player, URL params         |  jobs, cache, processing        (only host we fetch FITS from)
   |  Aladin Lite (Slice 4)      +--> Sesame/SIMBAD (name resolve, via astropy/astroquery)
   +--(Aladin loads its own catalogs/surveys directly from CDS)
```

## 2. Key decisions (final, do not revisit)

| Decision | Choice | Reason |
|---|---|---|
| Database | **None.** | No accounts or user data. Everything is derivable from IRSA. A DB adds setup, security and deploy cost for no benefit. |
| Persistence | Disk cache (files) + in-memory job registry | Cache is rebuildable; losing it only costs time. |
| Processes | **Exactly one uvicorn worker** | Job registry and rate limiter are in memory. Scale vertically; do not add workers. |
| Heavy work | Background thread pool (`JOB_WORKERS=2`) | Requests return immediately with a job key; the UI polls. TAP alone can take ~20 s. |
| Image delivery | 8-bit grayscale+alpha **PNG** per frame | Small, universal, animatable on a canvas. No FITS handling in the browser. |
| Data updates | Live TAP query, TTL cache | IRSA ingests weekly. New data appears automatically after cache TTL; no re-ingest step. |
| Frontend | Plain HTML/CSS/ES modules | No build step; fastest to ship solo. |
| Config | Environment variables, defaults in `config.py` | 12-factor; works locally and in Docker. |

## 3. Components (files and responsibilities)

```
backend/app/
  main.py        create_app(); mounts /api/v1 router and StaticFiles("frontend") at "/"; middleware
  config.py      Settings dataclass from env; ALL limits and constants (see SECURITY.md table)
  schemas.py     pydantic models: SequenceRequest (validated), SequenceStatus, FrameInfo, errors
  errors.py      AppError(code, http_status, message); exception handlers -> standard error JSON
  security.py    rate limiter, concurrency guard, security-headers middleware, request-id middleware,
                 allowlisted HTTP client factory
  resolver.py    name -> coordinates (+ proper motion, propagation to TARGET_EPOCH)
  irsa.py        build_adql(); query_images(ra, dec, detector) -> list[ImageRef]; URL validation/building
  cutouts.py     download_cutout(url) with retries -> bytes; parse_cutout(bytes) -> Cutout dataclass
  processing.py  mask_flags(), subtract_background(), make_grid(), reproject_frame(), stack_bins(), stretch()
  render.py      to_png(array_8bit, alpha) -> bytes
  cache.py       TTLCache wrappers; DiskCache (atomic writes, size-capped LRU eviction)
  jobs.py        JobRegistry, submit(), status(), cancellation, TTL cleanup, ThreadPoolExecutor
  pipeline.py    run_sequence(params, progress_cb) -> SequenceResult  (orchestrates all steps)
  catalogs.py    Gaia query (Slice 4)
  api.py         FastAPI routes (thin: validate -> call jobs/pipeline -> serialize)
backend/tests/   unit + integration tests
frontend/        index.html, style.css, app.js (+ small modules: api.js, player.js, url.js, ui.js, aladin.js)
scripts/         run_targets.py, warm_cache.py, smoke_deploy.py
```

Keep `api.py` thin. All astronomy logic lives in `irsa.py`, `cutouts.py`, `processing.py`, `pipeline.py`
so it can be tested without HTTP.

## 4. Data model (in memory / on disk; no database)

```python
@dataclass(frozen=True)
class ImageRef:            # one SPHEREx exposure that covers the target
    uri: str               # validated relative path "ibe/data/spherex/..."
    mjd: float             # time_bounds_lower
    url: str               # full cutout URL (built + validated server-side)

@dataclass
class Cutout:              # parsed from a downloaded MEF
    ref: ImageRef
    image: np.ndarray      # float32, MJy/sr
    flags: np.ndarray      # int32
    variance: np.ndarray   # float32
    zodi: np.ndarray       # float32
    wcs: astropy.wcs.WCS   # spatial, from IMAGE header
    detector: int
    obs_id: str
    version: str           # PRIMARY VERSION
    wavelength_um: float | None   # at the target position, from spectral WCS

@dataclass
class Frame:               # one output frame (single exposure or time-bin stack)
    data: np.ndarray       # float32 on the common grid, NaN where invalid
    mjd: float             # mean time
    wavelength_um: float | None
    n_exposures: int
    obs_ids: list[str]
    flag_fraction: float
```

Disk layout under `CACHE_DIR` (default `./cache`, gitignored):

```
cache/cutouts/<sha256(url)>.fits        raw downloaded cutout bytes (immutable, LRU-evicted)
cache/sequences/<key>/meta.json         final status/result JSON
cache/sequences/<key>/<index>.png       rendered frames
```

`<key>` is `sha256(canonical_json(normalized_params) + PIPELINE_VERSION)` truncated to 32 hex chars.
Bump `PIPELINE_VERSION` whenever processing changes so stale renders are not reused.
Round `ra/dec` to 6 decimals and `fov` to 3 decimals before hashing.

## 5. The pipeline (`pipeline.run_sequence`)

Input: validated `SequenceRequest`. Stages are reported through `progress_cb(stage, done, total)`.

1. **query** (TTL-cached by `(ra6, dec6, detector)`, TTL `TAP_CACHE_TTL_S`)
   Run the ADQL from DATA_SOURCES.md. Validate every returned `uri` (prefix `ibe/data/spherex/`, `.fits`
   suffix, no `..`, no scheme). Build the cutout URL in Python: `BASE + uri + "?center=<ra>,<dec>&size=<deg>"`.
   If zero rows: fail with `NO_COVERAGE`.
2. **select**: if more than `max_frames` images, subsample evenly in time (keep first and last); record
   `n_available` and `truncated=true`. In `stack` mode, first group by time bin, then cap total bins.
3. **download** (shared thread pool, global concurrency cap, per-file retry 3x with 1/2/4 s backoff + jitter)
   Use the disk cache by `sha256(url)`. Skip failures, count them. If fewer than 2 frames survive: `UPSTREAM_UNAVAILABLE`.
4. **parse**: open FITS from bytes (`memmap=False`); read `IMAGE, FLAGS, VARIANCE, ZODI`; reject files whose
   image dimensions exceed `CUTOUT_MAX_PIXELS`; compute spatial WCS and `wavelength_um` at the target pixel
   (wrap in try/except; on failure `None`).
5. **clean**: set flagged pixels to NaN per `mask_level`; apply `background`.
6. **grid**: build one tangent-plane WCS centered on the **target** (north up, east left), pixel scale =
   native scale / `upsample` (native scale taken from the first cutout WCS via
   `astropy.wcs.utils.proj_plane_pixel_scales`), size = `ceil(fov_arcmin*60 / grid_scale)`, reduced
   automatically so neither side exceeds `MAX_GRID_PX` (then report the effective upsample).
7. **reproject** each frame onto the grid with `reproject.reproject_interp((data, wcs), grid_wcs, shape_out,
   order="bilinear")` (also variance, same footprint). Pixels outside coverage become NaN. Process frames one
   at a time and release the input arrays.
8. **stack** (only if `mode="stack"`): group frames into bins of `bin_days` by MJD; per bin compute the
   inverse-variance weighted mean ignoring NaNs; frame time = mean MJD; `n_exposures` = bin size. Frames in
   `mode="frames"` pass through unchanged.
9. **stretch**: `stretch="global"` (default): compute `vmin, vmax` once as the 1st and 99.5th percentiles of all
   finite values across all output frames, then apply `asinh` scaling to 0-255. `stretch="frame"` does it per frame.
   Same `vmin/vmax` for every frame in global mode so brightness changes are real, not rescaling artifacts.
10. **render**: PNG mode `LA` (gray + alpha), alpha 0 where NaN. **Row 0 of the PNG is NORTH (top).** FITS arrays
    are bottom-up; flip vertically before encoding. Write atomically (temp file + rename).
11. **finish**: write `meta.json` (the `result` object in API_SPEC.md), mark job `done`.

Memory budget: at most `MAX_FRAMES_HARD x MAX_GRID_PX^2 x 4 bytes` of float32 in RAM (150 x 512^2 x 4 = ~157 MB)
plus variance if stacking. Do not exceed; free input arrays as soon as reprojected.

## 6. Job model

- `POST /sequences` validates, computes `key`. If `meta.json` exists and is fresh: return it (`status=done`).
  If a job with that key is active: return its status (**deduplicate**). Otherwise enqueue (reject with
  `RATE_LIMITED`/`BUSY` if limits are hit) and return `202`.
- States: `queued -> running(stage) -> done | failed`. Jobs are removed from memory `JOB_TTL_S` after finishing;
  results remain on disk until the rendered-cache TTL expires.
- Hard timeout `JOB_TIMEOUT_S` per job; cooperative cancellation checked between files/frames.
- The registry is a dict guarded by a lock. One worker process only (see section 2).

## 7. Caching summary

| Layer | Key | TTL / bound |
|---|---|---|
| TAP result list | `(ra6, dec6, detector)` | `TAP_CACHE_TTL_S` = 6 h, in memory (`cachetools.TTLCache`, max 256 entries) |
| Raw cutout bytes | `sha256(url)` | immutable; LRU eviction when `CACHE_MAX_BYTES` exceeded |
| Rendered sequence | `key` | `SEQ_CACHE_TTL_S` = 24 h, on disk |
| Name resolution | normalized name | 24 h, in memory |
| HTTP | frames: `Cache-Control: public, max-age=86400, immutable` (key is content-addressed) | |

Cache-miss stampede protection: job deduplication by key plus a per-URL lock around downloads.

## 8. Updating with new SPHEREx data

Nothing to re-ingest. After `TAP_CACHE_TTL_S` the next request sees new exposures, which changes `n_available`;
the sequence key also changes when `max_frames`/params change. Provide `GET /api/v1/meta` that returns the latest
observation time seen (cached 1 h) so the UI can print "data through <date>". Document this in the README
(it answers the challenge's "easy to update" consideration).

## 9. Error handling principles

- Every exception is mapped to `AppError` with a code from API_SPEC.md; unknown exceptions become `INTERNAL`
  with a generic message and a `request_id`; the stack trace goes to the log only.
- Partial success is normal: report `n_failed_downloads` and `warnings` instead of failing the job.
- Upstream (IRSA) failures are `UPSTREAM_UNAVAILABLE` (502) or `UPSTREAM_TIMEOUT` (504); never leak internal URLs or paths.

## 10. Logging and observability

JSON-lines logs to stdout: timestamp, level, `request_id`, route, status, duration, job key, stage timings.
No IP addresses or user-supplied free text beyond validated numeric parameters. `GET /api/v1/health` for the
platform health check (cheap, no upstream call; `?deep=1` pings IRSA, rate-limited).

## 11. Performance targets

- Cached sequence: first frame visible in < 1 s after the status request.
- Uncached, 6 arcmin, 40 frames: under ~60 s on the deployed host (dominated by IRSA TAP ~20 s and downloads).
  Show stage + counts so waiting feels purposeful.
- Frontend preloads all frame images before auto-play.
