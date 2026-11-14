# SECURITY.md

Threat model in one paragraph: this is a public, unauthenticated web service that makes outbound requests to a
shared scientific archive (IRSA) on behalf of anonymous visitors, does CPU/memory-heavy image processing, and
serves a browser UI. The realistic risks are **abuse/denial of service** (against us and against IRSA), **server-side
request forgery**, **injection** (ADQL, paths, HTML), **resource exhaustion**, and **information leakage**.
There are no accounts, passwords, payments, or personal data, which removes most other risks. Keep it that way.

Every item marked **MUST** is part of the definition of done for the slice that introduces it.

## 1. Limits table (all in `backend/app/config.py`, overridable by env var)

| Constant | Default | Purpose |
|---|---|---|
| `MIN_FOV_ARCMIN` / `MAX_FOV_ARCMIN` / `DEFAULT_FOV_ARCMIN` | 1 / 30 / 6 | field of view bounds |
| `MAX_FRAMES_HARD` / `DEFAULT_MAX_FRAMES` | 150 / 60 | frames per sequence |
| `MAX_GRID_PX` | 512 | max output width/height in pixels |
| `MAX_UPSAMPLE` | 4 | |
| `CUTOUT_MAX_BYTES` | 30,000,000 | refuse larger downloads (stream and abort) |
| `CUTOUT_MAX_PIXELS` | 4,000,000 | refuse FITS images with more pixels (e.g. > 2000x2000) |
| `TAP_TIMEOUT_S` | 90 | |
| `CUTOUT_TIMEOUT_S` | 90 | per download attempt |
| `JOB_TIMEOUT_S` | 600 | whole job |
| `DOWNLOAD_RETRIES` | 3 | backoff 1/2/4 s + jitter |
| `DOWNLOAD_CONCURRENCY` | 8 | global, across all jobs |
| `JOB_WORKERS` | 2 | concurrent jobs |
| `MAX_QUEUED_JOBS` | 10 | beyond this: `BUSY` |
| `MAX_ACTIVE_JOBS_PER_IP` | 2 | |
| `RATE_GENERAL_PER_MIN` | 60 | per IP |
| `RATE_CREATE_PER_MIN` | 6 | `POST /sequences`, per IP |
| `RATE_RESOLVE_PER_MIN` | 20 | per IP |
| `RATE_FRAMES_PER_MIN` | 600 | per IP |
| `TAP_CACHE_TTL_S` / `SEQ_CACHE_TTL_S` / `JOB_TTL_S` | 21600 / 86400 / 3600 | |
| `CACHE_MAX_BYTES` | 1,000,000,000 | disk cache cap (LRU eviction) |
| `MAX_REQUEST_BODY_BYTES` | 4096 | JSON body cap |
| `TARGET_EPOCH` | `J2026.0` | proper-motion propagation epoch |
| `ALLOWED_FETCH_HOSTS` | `{"irsa.ipac.caltech.edu"}` | outbound FITS/HTTP allowlist |
| `USER_AGENT` | `spherex-blink-viewer/<version> (NASA Space Apps 2026; contact: <set via env CONTACT_EMAIL>)` | identify ourselves to IRSA |

## 2. Input validation (MUST)

1. All request bodies/queries go through pydantic models with `extra="forbid"`, `allow_inf_nan=False`, and the
   ranges in API_SPEC.md. Reject, never "clamp silently".
2. `detector` is an int 1-6 mapped through a fixed dict to the band name. Never format raw input into a band string.
3. **ADQL injection:** the only user-derived values in ADQL are validated floats formatted with `f"{v:.8f}"` and the
   dict-mapped band name. No other user string ever reaches ADQL. Add a unit test that feeds adversarial strings
   (quotes, semicolons, `--`) and proves they are rejected before any query is built.
4. `name` for `/resolve`: regex `^[A-Za-z0-9 .+\-_\[\]'*/:]{1,64}$`. Pass only to the astropy/astroquery resolver.
   Never use it in a file name, shell command, log format string, or URL built by hand.
5. `key` path parameter: regex `^[0-9a-f]{32}$`; `index`: integer in range. Cache file paths are built **only** from
   `sha256` hex digests and integer indices, never from user strings. Add a path-traversal test (`../`, `%2e%2e`, backslashes).
6. Request body size capped (`MAX_REQUEST_BODY_BYTES`); reject others with 413/422.

## 3. Outbound requests / SSRF (MUST)

1. The browser **never** supplies a URL that the server fetches. Cutout URLs are built server-side from TAP results.
2. Validate every `a.uri` from TAP: starts with `ibe/data/spherex/`, ends with `.fits`, no `..`, no `://`, no whitespace,
   length <= 300. Reject and log otherwise.
3. The HTTP client (`httpx`) used for cutouts: HTTPS only; host must be in `ALLOWED_FETCH_HOSTS`; `follow_redirects=False`
   (if IRSA redirects, validate the `Location` host against the allowlist and allow at most 3 hops manually);
   timeouts from the limits table; stream the body and abort at `CUTOUT_MAX_BYTES`; send `USER_AGENT`.
4. Library-managed requests (astropy Sesame name resolution, astroquery SIMBAD/Gaia/IRSA TAP) cannot be routed through
   our client. Mitigation: their inputs are validated as above and their hosts are fixed by the libraries. Document this in the README.
5. No endpoint may proxy arbitrary URLs. No `file://`, no user-provided hostnames.

## 4. Abuse and resource exhaustion (MUST)

1. **Rate limiting** per client IP with a token bucket in memory (own small implementation; no new dependency unless approved).
   Client IP = socket peer unless env `TRUST_PROXY=1`, in which case take the **last** hop appended by the platform
   proxy from `X-Forwarded-For` (never the first value, which clients can forge). Return 429 + `Retry-After`.
2. **Concurrency guards:** `JOB_WORKERS`, `MAX_QUEUED_JOBS`, `MAX_ACTIVE_JOBS_PER_IP`, global `DOWNLOAD_CONCURRENCY` semaphore.
3. **Deduplicate** identical requests by sequence key so ten identical clicks cause one IRSA load.
4. **Bounded memory:** respect `MAX_FRAMES_HARD x MAX_GRID_PX^2`; process frame-by-frame; free arrays; refuse oversized FITS.
5. **Bounded disk:** `CACHE_MAX_BYTES` with LRU eviction; atomic writes; clean temp files at startup.
6. **Timeouts everywhere:** no network call or job may run without a timeout; jobs are cancelled at `JOB_TIMEOUT_S`.
7. **Be polite to IRSA:** cache TAP results and cutouts; cap concurrency; identifying User-Agent; back off on 429/503 from IRSA
   (respect `Retry-After` if present). Never loop-retry indefinitely.
8. Do not expose any "refresh/purge cache" endpoint publicly. (If needed for admin, protect with a secret from env and
   add it only with the developer's approval.)

## 5. Safe handling of FITS data (MUST)

IRSA is a trusted source, but parse defensively:
- `fits.open(io.BytesIO(data), memmap=False)`; read extensions by name; verify presence of `IMAGE`, `FLAGS`, `VARIANCE`, `ZODI`.
- Verify `image.ndim == 2`, shapes match across extensions, `size <= CUTOUT_MAX_PIXELS`, dtypes numeric.
- Wrap parsing in try/except; a bad file is skipped and counted, not fatal.
- Never `eval`/`exec`/pickle anything from files or headers. Never use header text in file paths, logs as format strings, or HTML.
- Header strings that reach the API response (`obs_id`, `version`) must match `^[A-Za-z0-9._\-]{1,64}$` or be replaced by `null`.

## 6. Web security headers and browser protections (MUST)

Add middleware sending on every response:
- `Content-Security-Policy`: `default-src 'self'; script-src 'self' <pinned Aladin CDN origin>; style-src 'self' 'unsafe-inline';
  img-src 'self' data: blob: <survey/tile origins used by Aladin>; connect-src 'self' <Aladin/CDS origins if required>; frame-ancestors *; base-uri 'none'; form-action 'none'; object-src 'none'`.
  Start strict (`self` only) in Slices 0-3 and widen minimally in Slice 4 after checking the browser console for blocked requests.
- `X-Content-Type-Options: nosniff`
- `Referrer-Policy: no-referrer`
- `Permissions-Policy: camera=(), microphone=(), geolocation=(), payment=()`
- `Strict-Transport-Security: max-age=31536000` only when served over HTTPS (set via env `ENABLE_HSTS=1` in production).
- **Embedding is a requirement**, so `frame-ancestors *` is intentional and X-Frame-Options is NOT sent. Document it.
- CORS: public read API, `*`, no credentials (API_SPEC.md section 7). No cookies are ever set.

## 7. Frontend security (MUST)

1. Never assign API data, URL parameters, or user input to `innerHTML`/`outerHTML`/`insertAdjacentHTML`/`document.write`. Use
   `textContent`, `setAttribute` for known-safe attributes, and DOM construction.
2. Parse and validate URL parameters client-side (numbers within the same ranges, enums from fixed sets); ignore anything else.
3. Image URLs for frames come only from the API response and are verified to start with `/api/v1/sequences/`.
4. Third-party scripts: only Aladin Lite, **pinned to an exact version**, with SRI (`integrity`, `crossorigin="anonymous"`) if available. No other third-party JS, fonts, or analytics.
5. Do not include secrets or internal paths in the JS bundle.

## 8. Error handling and information disclosure (MUST)

- Generic error messages to clients; details only in server logs with `request_id`.
- No stack traces, file paths, library versions, or upstream URLs in responses. Disable debug mode and
  `--reload` in production.
- Do not echo raw user input in error messages beyond the field name and the allowed range.
- `Server` header version disclosure disabled where possible (`--no-server-header` for uvicorn).

## 9. Logging and privacy (MUST)

- No cookies, no analytics, no accounts, no fingerprinting.
- Logs contain `request_id`, route, status, duration, numeric parameters (ra/dec/fov/detector), and job key.
  Do **not** log IP addresses (the rate limiter holds them in memory only) or free-text input.
- Add a short privacy note in the footer/README: "No personal data is collected or stored."

## 10. Dependencies and supply chain

- `requirements.txt` stays pinned (`pip freeze`). Add new packages only with approval and pin them.
- Optionally run `python -m pip install pip-audit` then `python -m pip_audit` before deploying and note results in DEVLOG.md.
- Frontend has no package manager and no build step, hence no npm supply chain.

## 11. Container and deployment hardening (see DEPLOYMENT.md)

- Run as a non-root user; read-only filesystem except the cache directory; no shell tools needed at runtime.
- Expose one port; HTTPS terminates at the platform; health check on `/api/v1/health`.
- Environment variables hold configuration only; none are secrets in this project. `.env` is gitignored; `.env.example` is committed.

## 12. Security tests to write (pytest)

| Test | Proves |
|---|---|
| out-of-range / NaN / inf / string `ra,dec,fov` rejected | validation |
| ADQL injection strings never reach the query builder | no injection |
| malicious `uri` values (`../`, `http://`, wrong prefix) rejected | SSRF/path safety |
| host not in allowlist -> download refused | SSRF |
| response larger than `CUTOUT_MAX_BYTES` aborted | resource cap |
| `key` and `index` traversal strings -> 404, no file access | path safety |
| 7th `POST /sequences` in a minute -> 429 with `Retry-After` | rate limiting |
| identical requests share one job | dedupe |
| error responses contain no stack trace / path | disclosure |
| security headers present on `/` and `/api/v1/health` | headers |
| frontend grep test: no `innerHTML` in `frontend/*.js` | XSS hygiene |

## 13. Known accepted risks (documented, not fixed)

- In-memory rate limiting resets on restart and does not work across multiple instances (single instance by design).
- Library-managed name-resolution requests are not constrained by our allowlist (inputs validated; hosts fixed by libraries).
- A determined attacker with many IPs can still consume the free-tier quota; mitigation is the global concurrency caps and caching.
