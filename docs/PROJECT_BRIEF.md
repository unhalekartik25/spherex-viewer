# PROJECT_BRIEF.md

## 1. The challenge (source: NASA Space Apps Challenge 2026, "Planet X and SPHEREx", Advanced)

NASA's SPHEREx mission has mapped the whole sky every six months since 2025 in 102 near-infrared
bands. Moving or changing objects (comets, asteroids, nearby stars, brown dwarfs, new planets)
reveal themselves by shifting position or brightness between images. Citizen-science projects
(e.g. Backyard Worlds, WiseView) have discovered many objects this way. There is too much data
for any person or computer alone, so the challenge asks for:

> A **public-facing web tool** that displays images of the sky from SPHEREx, making it easy for
> anyone to quickly see how they change over time. Users enter the coordinates (RA, Dec) of a
> patch of sky. The tool displays a series of images of that patch like a little video, making
> it easy to spot moving objects.

Required test targets: **Barnard's Star, Epsilon Indi B, WISE 0855, RW Aurigae, R Draconis.**
WISE 0855 is given as RA 08h55m10.831s, Dec -7d14m42.53s (= 133.795125, -7.245147 deg).

Optional ideas from the challenge (we pick a subset, see section 4):
easy data updates; use the six detector arrays (stacking per array) and spectra; add/subtract
images; selectable field of view; overlay Gaia/WISE data; coordinates in the URL for embedding;
show metadata such as contamination flags; reuse astropy and Aladin tools; WiseView as inspiration.

## 2. Product goal (one sentence)

A fast, embeddable web page where anyone types a name or RA/Dec and within seconds-to-a-minute sees
an aligned, time-ordered blink animation of SPHEREx images of that spot, with simple controls that
make motion and brightness changes obvious.

## 3. Target users

- A curious member of the public (no astronomy training): needs sensible defaults, plain-language labels.
- A citizen-science volunteer: needs speed, keyboard control, FOV and detector choices.
- A website author: needs a URL with coordinates that can be embedded in an iframe.

## 4. Scope

### IN scope (MVP = Slices 0-2, then 3-5 as time allows; see ROADMAP.md)
- Input by RA/Dec (decimal degrees or sexagesimal) or by object name.
- Query IRSA for SPHEREx spectral images covering the position, per detector array (D1-D6).
- Download only **cutouts**, never full images.
- Align all frames onto a common sky grid (north up, east left), fixed on the sky (not tracking the target).
- Blink player: play/pause, speed, scrub, step, loop, keyboard shortcuts.
- Selectable field of view (arcminutes) and detector array.
- Per-detector **stack mode**: co-add exposures in time bins to improve signal-to-noise.
- URL parameters so a view can be shared and embedded.
- Metadata per frame: time, wavelength at the target, detector, processing version, flagged-pixel fraction.
- ONE extra feature: Gaia proper-motion overlay with an Aladin Lite sky context panel (Slice 4).
- Public deployment, README, demo video, all five test targets verified.

### OUT of scope (do not build, do not suggest)
- Automatic detection of moving objects or machine learning.
- User accounts, saved sessions, comments, databases, analytics, cookies.
- Full-sky browsing, tile servers, mosaics of the whole survey.
- Writing our own FITS viewer in the browser (all FITS handling is server-side).
- Mobile apps. (Responsive layout is enough.)

### STRETCH (only after Slice 5 is done and deployed)
- Spectrum extraction at a clicked position; image add/subtract; light-curve sparkline for variables;
  flags overlay image; pre-seeded demo data for offline resilience.

## 5. Success criteria

1. The deployed URL works for all five test targets (see TEST_TARGETS.md) and shows a smooth blink.
2. A first-time visitor can get a result in under 3 clicks without reading instructions.
3. A URL like `/?ra=133.795125&dec=-7.245147&fov=8&det=5` opens directly into the result (embeddable).
4. The service does not crash or hammer IRSA when abused (limits in SECURITY.md).
5. README + 2-3 minute demo video explain the tool, the data, and how it handles updates.

## 6. Constraints

- Solo developer, ~24-36 hours, AI-assisted coding.
- Free-tier hosting (limited RAM/CPU, ephemeral disk). Design for ~512 MB RAM. [ASSUMPTION; see DEPLOYMENT.md]
- IRSA TAP queries can take ~20 s and some cutout downloads fail transiently. UX must show progress and
  tolerate failures. [VERIFIED in IRSA tutorial]
- SPHEREx data is public; no authentication needed. [VERIFIED]

## 7. Attribution (must appear in the UI footer and README)

SPHEREx data: NASA/JPL-Caltech, served by NASA/IPAC Infrared Science Archive (IRSA). Gaia data: ESA/Gaia/DPAC.
WISE data (if shown): NASA/JPL-Caltech/UCLA. Inspired by WiseView and Backyard Worlds.
Built with Astropy, reproject, FastAPI, Aladin Lite (CDS).
Check the official Space Apps submission page for any additional required wording or deliverables.
Do not invent submission requirements.
