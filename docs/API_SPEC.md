# API_SPEC.md

Base path: `/api/v1`. JSON in, JSON out, UTF-8. Public, read-only, no authentication, no cookies.
Static frontend is served from `/`. All limits referenced here live in `backend/app/config.py`
(values in the SECURITY.md limits table).

## 1. Conventions

- Coordinates: ICRS decimal degrees. `ra` in [0, 360), `dec` in [-90, 90]. Sexagesimal parsing happens in the
  frontend; the API accepts only numbers.
- Validation is strict (pydantic, `extra="forbid"`, `allow_inf_nan=False`). Unknown fields are rejected.
- Every response carries `X-Request-ID`. Errors use the standard envelope (section 8).
- Time: `mjd` (float days) and `iso` (UTC ISO-8601 string).
- Frame image orientation: **north up, east left**, row 0 = north.

## 2. Parameter table (validated for `POST /sequences`)

| Field | Type | Default | Allowed | Notes |
|---|---|---|---|---|
| `ra` | float | required | 0 <= ra < 360 | |
| `dec` | float | required | -90 <= dec <= 90 | |
| `fov_arcmin` | float | 6 | 1 - 30 | field of view (full width) |
| `detector` | int | 2 | 1 - 6 | maps to `SPHEREx-D<n>` via a dict, never string-built from input |
| `mode` | enum | `frames` | `frames`, `stack` | `stack` co-adds per time bin |
| `bin_days` | float | 30 | 1 - 180 | used only when `mode=stack` |
| `background` | enum | `median` | `none`, `median`, `zodi` | |
| `mask_level` | enum | `minimal` | `minimal`, `strict` | see DATA_SOURCES.md |
| `stretch` | enum | `global` | `global`, `frame` | |
| `upsample` | int | 2 | 1 - 4 | auto-reduced to respect `MAX_GRID_PX` |
| `max_frames` | int | 60 | 2 - 150 | frames (or bins) after subsampling |

## 3. Endpoints

### 3.1 `GET /health`
`200 {"status":"ok","version":"<app version>","pipeline_version":"<n>","uptime_s":123.4}`
`?deep=1` additionally pings IRSA TAP with a trivial query and returns `"irsa":"ok"|"unavailable"`
(rate-limited to 2/min per client).

### 3.2 `GET /meta`
`200 {"pipeline_version":"1","detectors":[{"id":1,"name":"SPHEREx-D1","wavelength_um":[0.75,1.09]}, ...],
"limits":{"fov_arcmin":[1,30],"max_frames":150}, "latest_observation_mjd": 61234.5 | null, "latest_observation_iso": "..." | null,
"attribution":"..."}`
`latest_observation_*` comes from a cached (1 h) query of the max `time_bounds_lower`; `null` if unavailable.

### 3.3 `GET /resolve?name=<string>`
Resolves an object name. `name`: 1-64 chars matching `^[A-Za-z0-9 .+\-_\[\]'*/:]{1,64}$` else `INVALID_PARAMETER`.
`200`:
```json
{
  "query": "Barnard's Star",
  "ra": 269.452, "dec": 4.6934,                 // catalog position (ICRS)
  "catalog_epoch": "J2000.0",                    // null if unknown
  "pm_ra_mas_yr": -798.58, "pm_dec_mas_yr": 10328.12,   // null if unknown (values illustrative)
  "ra_now": 269.4, "dec_now": 4.77,              // propagated to TARGET_EPOCH, null if no proper motion
  "target_epoch": "J2026.0",
  "source": "SIMBAD"
}
```
Errors: `NOT_FOUND` (404), `UPSTREAM_UNAVAILABLE`, `INVALID_PARAMETER`.
The numeric example values are illustrative only; never hard-code them.

### 3.4 `POST /sequences`  (create or fetch a sequence)
Body: JSON with the fields in section 2 (only `ra` and `dec` required).
- `200` if a fresh result already exists (body is the Status document with `status:"done"`).
- `202` if a job was created or already running (body is the Status document; also `Location` header).
- Errors: `INVALID_PARAMETER` (422), `RATE_LIMITED` (429, `Retry-After`), `BUSY` (503, `Retry-After`).

### 3.5 `GET /sequences/{key}`  (poll status/result)
`key` must match `^[0-9a-f]{32}$` else `NOT_FOUND`. Returns the Status document:

```json
{
  "key": "0123456789abcdef0123456789abcdef",
  "status": "queued | running | done | failed",
  "stage": "query | select | download | process | render | null",
  "progress": {"done": 12, "total": 40},
  "request": { "ra": 133.795125, "dec": -7.245147, "fov_arcmin": 8, "detector": 5, "mode": "frames", "...": "normalized echo" },
  "result": null,
  "error": null,
  "created_at": "2026-11-14T04:12:33Z",
  "finished_at": null
}
```

When `status = "done"`, `result` is:

```json
{
  "n_frames": 54,
  "n_available": 143,
  "truncated": true,
  "n_failed_downloads": 6,
  "grid": {
    "width": 160, "height": 160,
    "pixscale_arcsec": 3.08, "native_pixscale_arcsec": 6.16, "upsample": 2,
    "orientation": "north-up, east-left, row 0 = north",
    "wcs": {"ctype": ["RA---TAN","DEC--TAN"], "crval": [133.795125, -7.245147],
            "crpix": [80.5, 80.5], "cdelt_deg": [-0.000856, 0.000856]}
  },
  "stretch": {"mode": "global", "function": "asinh", "vmin": -0.8, "vmax": 12.4, "unit": "MJy/sr"},
  "frames": [
    {
      "index": 0,
      "url": "/api/v1/sequences/<key>/frames/0.png",
      "mjd": 60800.1, "iso": "2025-05-14T02:41:00Z",
      "wavelength_um": 4.62, "detector": 5,
      "n_exposures": 1, "obs_id": "2025W20_2D_0176_4",
      "version": "l2b-v27-2026-198",
      "flag_fraction": 0.031,
      "center_value": 1.84
    }
  ],
  "warnings": ["12 frames skipped: download failed"]
}
```

Notes: example numbers are illustrative. `center_value` = mean of the central 3x3 grid pixels in MJy/sr
(or `null`), provided for the optional sparkline (Slice 3); it mixes wavelengths across frames, so the UI must label it as such.
`obs_id` for a stack is a list or `null`; use `obs_ids` (array) for stack frames. Frames are ordered by `mjd` ascending.
When `status = "failed"`, `error` is `{"code","message"}` and `result` is `null`.

### 3.6 `GET /sequences/{key}/frames/{index}.png`
`index` integer 0..n_frames-1. Returns `image/png`, `Cache-Control: public, max-age=86400, immutable`,
`X-Content-Type-Options: nosniff`. `NOT_FOUND` if the key/index is unknown or the sequence expired.

### 3.7 `GET /catalog/gaia`  (Slice 4)
Query: `ra`, `dec`, `radius_arcmin` (0.5-20). Returns up to 500 rows:
`{"epoch":"J2016.0","count":N,"truncated":false,"sources":[{"source_id":"...","ra":..,"dec":..,"pmra_mas_yr":..,"pmdec_mas_yr":..,"parallax_mas":..,"g_mag":..}]}`
`source_id` is returned as a **string** (64-bit ids exceed JavaScript safe integers). Cached 24 h per rounded query.
The frontend propagates positions to each frame's epoch.

### 3.8 `GET /spectrum`  (STRETCH, only if time remains)
Query: `key`, `x`, `y` (grid pixel coordinates in a done sequence), `radius_px` (1-5). Returns
`{"points":[{"wavelength_um":..,"flux_mjy_sr":..,"sigma":..,"mjd":..,"detector":..}]}` using each frame's
wavelength at that sky position. Requires the sequence to be cached with per-frame arrays retained or recomputable;
if not implemented, the endpoint must not exist (return 404 via normal routing).

## 4. Status codes

`200` ok, `202` accepted, `400` malformed JSON, `404` not found, `422` validation, `429` rate limited,
`500` internal, `502` upstream unavailable, `503` busy, `504` upstream timeout.

## 5. Rate limits (per client IP; see SECURITY.md)

General 60 req/min; `POST /sequences` 6/min; `GET /resolve` 20/min; frames are not counted against the general
limit but share a 600/min ceiling. Exceeding returns 429 with `Retry-After` seconds.

## 6. Caching headers

- `GET /meta`, `/health`: `Cache-Control: no-store` (health) / `max-age=60` (meta).
- Status: `no-store` while `queued|running`; `max-age=300` when `done`.
- Frames: `public, max-age=86400, immutable`.
- Static assets: `max-age=3600` (or `no-cache` during development).

## 7. CORS and embedding

- API: `Access-Control-Allow-Origin: *` for `GET` and `POST` only, **no credentials**. Safe because the API is public
  and has no cookies or per-user state.
- The frontend page must be embeddable: do **not** send `X-Frame-Options: DENY`. Use CSP `frame-ancestors *`
  (intentional, documented).

## 8. Error envelope and codes

```json
{"error": {"code": "INVALID_PARAMETER", "message": "fov_arcmin must be between 1 and 30", "request_id": "b3f1..."}}
```

| Code | HTTP | When |
|---|---|---|
| `INVALID_PARAMETER` | 422 | validation failure (message names the field, never echoes raw long input) |
| `NOT_FOUND` | 404 | unknown key/index/name |
| `NO_COVERAGE` | 404 | no SPHEREx images cover the position for this detector |
| `RATE_LIMITED` | 429 | rate limit exceeded; `Retry-After` set |
| `BUSY` | 503 | job queue full; `Retry-After` set |
| `UPSTREAM_UNAVAILABLE` | 502 | IRSA/Sesame/Gaia failed or too few cutouts downloaded |
| `UPSTREAM_TIMEOUT` | 504 | upstream or job timeout |
| `INTERNAL` | 500 | anything unexpected; generic message only |

Messages are written for end users (plain language) and never contain stack traces, file paths, or upstream URLs.

## 9. Versioning

URL prefix `/api/v1`. `pipeline_version` (string) is part of every cache key. Additive changes only within v1.

## 10. OpenAPI

FastAPI's auto docs (`/docs`, `/openapi.json`) may stay enabled in development. In production keep `/openapi.json`
(useful for judges) but you may disable `/docs` and `/redoc` via an env flag `ENABLE_DOCS=0`.
