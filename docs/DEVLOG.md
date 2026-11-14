# DEVLOG.md

Running notes. Append after every slice (newest at the bottom). This becomes the README and the demo script, so write
for a reader who was not there. Record **facts you verified**, not guesses.

## Setup facts (pre-filled)

- 2026-11-14 (pre-hackathon): Development in GitHub Codespaces, Python 3.14.2, venv `.venv`.
- Installed (pip freeze): astropy 8.0.2, astroquery 0.4.11, pyvo 1.10.0, reproject 0.21.0, scipy 1.18.1, numpy 2.5.4,
  pillow 13.0.0, fastapi 0.162.7, uvicorn 0.54.0, httpx 0.28.1, cachetools 8.0.0, pytest 9.2.0.
- Verified: IRSA TAP reachable from the Codespace; table `spherex.obscore` returns rows (columns include obs_id, t_min, t_max,
  em_min, em_max, access_url, s_region, s_pixel_scale, energy_bandpassname).
- The developer's Windows laptop cannot run the scientific stack (Application Control policy); all code runs in Codespaces.

## Verification checklist (copy results here; also fix DATA_SOURCES.md)

| # | Question | Answer | Slice | Date |
|---|---|---|---|---|
| 1 | Cutout `size` semantics, output shape, edge behaviour | | 1 | |
| 2 | Plain-GET download of a cutout works with httpx | | 1 | |
| 3 | `time_bounds_lower` is MJD | | 1 | |
| 4 | Real pixel scale and WCS of cutouts | | 1 | |
| 5 | Best detector settings per target | | 1-2 | |
| 6 | SIMBAD proper-motion field names (astroquery 0.4.11) | | 2 | |
| 7 | Gaia access route that works | | 4 | |
| 8 | Aladin Lite CDN URL/version/SRI | | 4 | |
| 9 | Host reaches IRSA; RAM/port/disk limits | | 0 | |

## Target results table (fill from TEST_TARGETS.md procedure)

| Target | Detector | FOV | Mode | Frames | Failed DLs | Time uncached (s) | Phenomenon visible? | Screenshot / URL |
|---|---|---|---|---|---|---|---|---|
| WISE 0855 | | | | | | | | |
| Barnard's Star | | | | | | | | |
| Epsilon Indi B | | | | | | | | |
| RW Aurigae | | | | | | | | |
| R Draconis | | | | | | | | |

## SECURITY.md MUST checklist (tick as implemented and tested)

- [ ] 2.1-2.6 input validation and tests
- [ ] 3.1-3.5 outbound allowlist / SSRF protections and tests
- [ ] 4.1-4.8 rate limits, concurrency, dedupe, memory/disk caps, timeouts
- [ ] 5 safe FITS handling
- [ ] 6 security headers (CSP, nosniff, referrer, permissions, HSTS in prod)
- [ ] 7 frontend: no innerHTML, validated URL params, pinned third-party script
- [ ] 8 error handling: no leaks, debug off
- [ ] 9 logging/privacy: no IP or free text in logs, no cookies

## Entries

### Template (copy for each slice)
**Slice N: <name>  |  <date/time>  |  commit <hash>**
- Built:
- Verified (commands and results):
- Decisions made where the docs were silent:
- Surprises / doc corrections:
- Performance numbers:
- Blocked / risks:
- Next:
