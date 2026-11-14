# DATA_SOURCES.md

Everything here is tagged `[VERIFIED]`, `[ASSUMPTION]`, or `[TODO-VERIFY]` (see AGENT_RULES.md section 5).
Official references (save copies into `docs/reference/` if possible):

- Cutout service: https://irsa.ipac.caltech.edu/data/SPHEREx/docs/cutout_tool.html
- Cutout tutorial: https://caltech-ipac.github.io/irsa-tutorials/spherex-cutouts
- Data overview tutorial: https://caltech-ipac.github.io/irsa-tutorials/spherex-intro
- PSF tutorial: https://caltech-ipac.github.io/irsa-tutorials/spherex-psf
- SPHEREx archive guide: https://caltech-ipac.github.io/spherex-archive-documentation
- Explanatory Supplement: https://irsa.ipac.caltech.edu/data/SPHEREx/docs/SPHEREx_Expsupp.pdf

## 1. The instrument and bands

- SPHEREx covers 0.75-5 micron. The focal plane has **six 2040x2040 detector arrays** behind linear
  variable filters (LVFs). Wavelength varies across each array. [VERIFIED]
- Bands (detector N is called `SPHEREx-DN` in the archive): [VERIFIED]

| Detector | Wavelength (um) | R |
|---|---|---|
| D1 | 0.75 - 1.09 | 39 |
| D2 | 1.10 - 1.62 | 41 |
| D3 | 1.63 - 2.41 | 41 |
| D4 | 2.42 - 3.82 | 35 |
| D5 | 3.83 - 4.41 | 112 |
| D6 | 4.42 - 5.00 | 128 |

- Because wavelength varies across the array, one sky position lands at a different wavelength in
  each exposure. Every frame therefore has its own wavelength at the target (from the spectral WCS).
  Stacking within a detector mixes nearby wavelengths; label this clearly in the UI. [VERIFIED reasoning from docs]
- Pixel scale is about 6.2 arcsec per pixel (derived from the example header PC matrix). Read the true
  value from each cutout's WCS; do not hard-code it. [ASSUMPTION; TODO-VERIFY in Slice 1]
- Data releases: DR1 and QR3 are current. QR3 and DR1 supersede QR2; do not use QR2. TAP metadata
  tables include DR1 and QR3. Ingestion is weekly. [VERIFIED]

## 2. How we query: TAP (primary method) [VERIFIED]

- Endpoint: `https://irsa.ipac.caltech.edu/TAP` (tested from Codespace with pyvo).
- Use TAP rather than SIA because TAP has data immediately after weekly ingestion; SIA lags ~1 day. [VERIFIED]
- The IRSA tutorial query (adapt, do not guess other table names):

```sql
SELECT a.uri, p.time_bounds_lower
FROM spherex.artifact a
JOIN spherex.plane p ON a.planeid = p.planeid
WHERE 1 = CONTAINS(POINT('ICRS', <ra_deg>, <dec_deg>), p.poly)
  AND p.energy_bandpassname = 'SPHEREx-D<n>'
ORDER BY p.time_bounds_lower
```

  - The tutorial selects the full cutout URL in ADQL via string concatenation. **We select `a.uri`
    only and build the URL in Python** so we can validate it (see SECURITY.md).
  - `a.uri` looks like `ibe/data/spherex/dr1/level2/2025W18_1B/l2b-v27-2026-196/1/level2_..._spx_l2b-v27-2026-196.fits`.
    The full URL is `https://irsa.ipac.caltech.edu/` + `a.uri`. [VERIFIED from tutorial output]
  - `p.time_bounds_lower` is the observation time, in days (almost certainly MJD). [ASSUMPTION; confirm
    against header DATE/MJD keywords in Slice 1]
  - **Latency: the tutorial query took ~20 s.** Cache TAP results (see ARCHITECTURE.md). [VERIFIED]
  - Python entry points: `astroquery.ipac.irsa.Irsa.query_tap(query)` (used in the tutorial) or
    `pyvo.dal.TAPService(...)`. Either is fine; pick one, wrap it in `backend/app/irsa.py`.
- A second table, `spherex.obscore`, also works (columns include `obs_id, t_min, t_max, em_min, em_max,
  access_url, s_region, s_pixel_scale, energy_bandpassname, ...`). It is optional; the artifact/plane join
  above is the verified cutout workflow. [VERIFIED columns by developer test]
- Example scale: one tutorial position gave 84 images for detector D2 alone; another gave 443 across detectors.
  Expect hundreds of frames per position over time. Hence `max_frames` limits. [VERIFIED]

## 3. How we get cutouts [VERIFIED]

- A cutout is requested by appending `center` and `size` to the full image URL. `size` is in degrees.
  Two forms appear in IRSA documentation; both are accepted:
  - tutorial form: `...fits?center=210.80227,54.34895d&size=0.1`
  - documentation form: `...fits?center=156.09328159,-41.64466331&size=0.1`
  Use the documentation form (plain decimal degrees) unless it fails, then try the tutorial form.
- Whether `size` is full width or half width, and what happens near detector edges: `[TODO-VERIFY]` in
  Slice 1 by checking the returned image shape against `size / pixel_scale`.
- A cutout is a multi-extension FITS (MEF) with the same structure as the full file:

| HDU | Name | Content |
|---|---|---|
| 0 | PRIMARY | minimal header; contains `VERSION` (processing version) |
| 1 | IMAGE | calibrated flux, **MJy/sr**, no zodiacal subtraction |
| 2 | FLAGS | per-pixel bitmask (int32, compressed image HDU) |
| 3 | VARIANCE | per-pixel variance, (MJy/sr)^2 |
| 4 | ZODI | modeled zodiacal light, MJy/sr, NOT subtracted from IMAGE |
| 5 | EPSF | binary table of effective PSFs (ignore in MVP) |
| 6 | WCS-WAVE | lookup table for the spectral WCS |

  Read extensions **by name** (`hdul["IMAGE"]`), not by index.
- Useful IMAGE header keywords: `DETECTOR`, `OBSID`, `DATE`, `BUNIT`, `PSF_FWHM`, plus the standard
  spatial WCS (`CTYPE = RA---TAN-SIP / DEC--TAN-SIP`). Counts of flagged pixels are in `HIERARCH L2 N_*` keywords.
- **Spatial WCS:** `astropy.wcs.WCS(hdul["IMAGE"].header)`.
- **Spectral WCS (wavelength at a pixel):**

```python
spectral_wcs = WCS(header=hdul["IMAGE"].header, fobj=hdul, key="W")
spectral_wcs.sip = None            # REQUIRED: SIP must be turned off for the spectral WCS
wavelength, bandwidth = spectral_wcs.pixel_to_world(x, y)   # x, y are 0-based pixel coords
wl_um = wavelength.to("um").value
```

- Reading files: the tutorial opens URLs with astropy and sets `astropy.utils.data.conf.remote_timeout = 120`.
  **We download bytes with `httpx` (allowlisted host, our timeouts) and open them with
  `fits.open(io.BytesIO(data), memmap=False)`.** [ASSUMPTION that a plain GET returns the FITS bytes; TODO-VERIFY Slice 1]
- **Transient failures are normal.** In the tutorial's parallel run, a noticeable fraction of cutouts failed
  (timeouts, HTTP errors, incomplete reads) and were simply skipped. Retry with backoff (3 tries), then skip and
  count failures. Catch: `TimeoutError`, `urllib.error.HTTPError`, `http.client.IncompleteRead`,
  `http.client.RemoteDisconnected`, plus httpx equivalents. [VERIFIED]
- The tutorial downloads with 10 parallel threads without trouble; we use at most 8 globally. [VERIFIED/our choice]

## 4. FLAGS bit definitions [VERIFIED from FLAGS header]

| Bit | Name | Meaning |
|---|---|---|
| 0 | TRANSIENT | transient detected during up-the-ramp sampling |
| 1 | OVERFLOW | overflow detected |
| 2 | SUR_ERROR | error in onboard processing |
| 6 | NONFUNC | permanently unusable pixel |
| 7 | DICHROIC | low efficiency due to dichroic |
| 9 | MISSING_DATA | onboard data lost |
| 10 | HOT | hot pixel |
| 11 | COLD | anomalously low signal |
| 12 | FULLSAMPLE | full sample history available (informational) |
| 14 | PHANMISS | phantom correction not applied |
| 15 | NONLINEAR | nonlinearity correction not reliable |
| 17 | PERSIST | persistent charge above threshold |
| 19 | OUTLIER | flagged by outlier detection |
| 20 | CROSSTALK | affected by crosstalk |
| 21 | SOURCE | pixel mapped to a known source (**NOT bad**; most pixels have it) |
| 22 | GHOST | ghost from bright source in exposure |
| 23 | GHOST_FPA | ghost from bright source on another detector |
| 24 | GHOST_EXT | ghost from bright source outside FOV |
| 25 | STREAK | streak |
| 26 | BLOOM | bloom artifact |
| 27 | SNOWBALL | snowball artifact |
| 28 | HALO | transient halo |

Masking policy for this project (config constants in `config.py`):

- `mask_level="minimal"` (default): mask bits **1, 2, 6, 9, 10, 11, 15** only (clearly unusable pixels).
- `mask_level="strict"`: minimal plus bits **0, 17, 19, 20, 22, 23, 24, 25, 26, 27, 28**.
- **Never mask SOURCE (bit 21).**
- Why minimal by default: a real moving object may be flagged TRANSIENT or OUTLIER in one exposure. Masking
  those by default could erase exactly what the tool exists to find. [ASSUMPTION; keep the strict option]

## 5. Background

- IMAGE has **no zodiacal-light subtraction**, and the zodiacal background changes with time and pointing, which
  makes a naive blink flicker. Options (`background` parameter): `none`, `median` (subtract the median of valid
  pixels of each frame; default), `zodi` (subtract the ZODI extension, then the median). [ASSUMPTION; check visually in Slice 1]

## 6. Name resolution and moving targets

- Use CDS Sesame/SIMBAD via astropy/astroquery for names. `SkyCoord.from_name(name)` is the simple route.
  Better: query SIMBAD for position **and proper motion** (`astroquery.simbad.Simbad`); field names in
  astroquery 0.4.11 `[TODO-VERIFY]` in Slice 2.
- **Epoch problem (important):** catalog positions are for a catalog epoch (SIMBAD: J2000 for most stars; the
  challenge's WISE 0855 position may be from ~2010). SPHEREx observes in 2025-2026. Fast movers have shifted
  noticeably. Approximate proper motions (from memory, `[TODO-VERIFY]` in Simbad): Barnard's Star ~10.4 arcsec/yr
  (about 4.5 arcmin shift since 2000), WISE 0855 ~8 arcsec/yr. A 6 arcmin field centered on a J2000 position may
  **not contain the star any more**.
- Therefore: the resolver returns the catalog position AND, when proper motion is known, a position propagated
  to `TARGET_EPOCH = J2026.0` (astropy `SkyCoord(..., pm_ra_cosdec, pm_dec, obstime=...)` then
  `apply_space_motion(new_obstime=Time("J2026.0"))`). The UI centers on the propagated position by default and
  shows both. For raw RA/Dec typed by the user, use the values as given.
- For WISE 0855 given as raw coordinates, use FOV >= 8 arcmin for the first test, because of the epoch shift.

## 7. Overlay catalogs (Slice 4)

- **Gaia DR3** (latest release, as the challenge requires). Catalog epoch is J2016.0. A star at time t is at
  `pos + pm * (t - 2016.0)`. Access options, pick the first that works from Codespaces `[TODO-VERIFY]`:
  1. `astroquery.vizier` with catalog `I/355/gaiadr3`;
  2. ESA Gaia TAP via pyvo/astroquery.gaia.
  Return at most 500 rows with `source_id, ra, dec, pmra, pmdec, parallax, phot_g_mean_mag`.
- **Aladin Lite** (CDS) in the browser shows survey backdrops and catalogs on its own. Use the official, pinned
  CDN URL from the Aladin Lite documentation `[TODO-VERIFY]`; add Subresource Integrity if the docs provide a hash.
- WISE overlay is optional; Aladin can show the AllWISE catalog directly.

## 8. Open verification checklist (resolve in the named slice, record in DEVLOG.md)

| # | Question | Slice |
|---|---|---|
| 1 | Cutout `size` semantics and output shape; edge behaviour | 1 |
| 2 | Plain-GET download of a cutout works with httpx | 1 |
| 3 | `time_bounds_lower` is MJD | 1 |
| 4 | Real pixel scale and WCS of cutouts (read from header) | 1 |
| 5 | Default detector choices for each test target give visible results | 1-2 |
| 6 | SIMBAD field names for proper motion in astroquery 0.4.11 | 2 |
| 7 | Gaia access route works from Codespaces and host | 4 |
| 8 | Aladin Lite CDN URL / version / SRI | 4 |
| 9 | Hosting platform can reach IRSA and fits in RAM | 0 and 5 |
