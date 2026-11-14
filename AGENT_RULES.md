# AGENT_RULES.md  (READ THIS FIRST, EVERY SESSION)

## 0. What you are building

**SPHEREx Blink Viewer**: a public web tool for the NASA Space Apps Challenge 2026
("Planet X and SPHEREx"). A visitor enters sky coordinates (or an object name). The tool
fetches small cutouts of NASA SPHEREx infrared images of that patch from IRSA, aligns them
onto one common sky grid, and plays them in time order as a blink animation so moving or
changing objects (brown dwarfs, asteroids, high-proper-motion stars, variable stars) are easy
to spot.

- One developer, working solo, ~24-36 hours, hackathon starts Sat 14 Nov 2026 at 09:00.
- The developer relies on you to write the code. **All architecture, API, database, and
  security decisions are already made in `docs/`. Do not ask the developer to re-decide them.**
  If the docs are silent, choose the simplest option that satisfies `docs/SECURITY.md`, and
  write the decision into `docs/DEVLOG.md`.
- A working, deployed demo beats a complete but broken tool. Follow `docs/ROADMAP.md` slice by slice.

## 1. Reading order (do this before writing code)

1. `docs/PROJECT_BRIEF.md`   goal, scope, what is OUT of scope
2. `docs/DATA_SOURCES.md`    verified facts about IRSA/SPHEREx data (do not guess around these)
3. `docs/ARCHITECTURE.md`    components, pipeline, caching, jobs, no-database decision
4. `docs/API_SPEC.md`        exact endpoints, parameters, limits, JSON shapes, error codes
5. `docs/SECURITY.md`        mandatory protections and the limits table
6. `docs/FRONTEND_SPEC.md`   UI behaviour, URL parameters, player
7. `docs/DEPLOYMENT.md`      container, env vars, hosting, smoke test
8. `docs/ROADMAP.md`        the slices, with acceptance criteria. Work in this order.
9. `docs/TEST_TARGETS.md`    the five required targets and the testing strategy
10. `docs/DEVLOG.md`         append notes here after every slice

## 2. Hard rules

1. **One slice at a time.** Do not start slice N+1 until slice N's acceptance criteria pass.
2. **Never invent IRSA/SPHEREx API details.** Use only what is in `docs/DATA_SOURCES.md`.
   Anything tagged `[ASSUMPTION]` or `[TODO-VERIFY]` must be tested against the live service
   the first time you use it; record the result in `docs/DEVLOG.md` and fix the doc.
3. **Security is not optional.** Every item marked MUST in `docs/SECURITY.md` is part of the
   definition of done. Never put user input into ADQL, file paths, URLs, shell commands, or
   HTML without the validation described there.
4. **Ask before adding dependencies.** Allowed already: see section 3. Do not add a database,
   an ORM, a JS framework, a bundler, or a task queue. (Reasons in `docs/ARCHITECTURE.md`.)
5. **Write tests with each feature** (`pytest`). Network tests are marked `@pytest.mark.network`
   and are skipped by default. See `docs/TEST_TARGETS.md`.
6. **Keep the code simple and readable.** Small functions, type hints, docstrings on public
   functions, constants and limits only in `backend/app/config.py`.
7. **No secrets in the repo.** There are no API keys in this project. Config via environment
   variables with safe defaults (`.env.example` documents them).
8. **Commit after each slice** with a clear message, then append to `docs/DEVLOG.md`.
9. **Do not claim something works unless you ran it.** Show the command and the output summary.
10. **Respect the data provider.** Cache aggressively, cap concurrency, identify the app in the
    User-Agent. IRSA is a shared public archive.

## 3. Environment (facts, verified)

- Development happens in **GitHub Codespaces** (Linux, bash). The developer's laptop cannot run
  the scientific stack (Windows Application Control policy), so never assume local Windows execution.
- Repo root: `/workspaces/spherex-viewer`. Virtualenv: `.venv` (`source .venv/bin/activate`).
- **Python 3.14.2.**
- Already installed (from `requirements.txt`): `astropy 8.0.2`, `astroquery 0.4.11`, `pyvo 1.10.0`,
  `reproject 0.21.0`, `scipy 1.18.1`, `numpy 2.5.4`, `pillow 13.0.0`, `fastapi 0.162.7`,
  `uvicorn 0.54.0`, `httpx 0.28.1`, `cachetools 8.0.0`, `pytest 9.2.0` (plus their dependencies).
- Verified: from the Codespace, `pyvo` reaches `https://irsa.ipac.caltech.edu/TAP` and the table
  `spherex.obscore` returns rows.
- Existing folders: `backend/app`, `backend/tests`, `frontend`, `scripts`, `docs/reference`, `cache`.
- Frontend: plain HTML + CSS + JavaScript (ES modules). Aladin Lite from a pinned CDN URL in Slice 4 only.

## 4. Commands (always use the `python -m` form)

```bash
source .venv/bin/activate
python -m uvicorn backend.app.main:app --host 0.0.0.0 --port 8000 --reload   # dev server
python -m pytest -q                                                          # offline tests
python -m pytest -q -m network                                               # live IRSA tests
python scripts/run_targets.py                                                # the 5 required targets
```

Codespaces forwards port 8000 automatically (see the "Ports" tab). Package layout requires
`backend/__init__.py` and `backend/app/__init__.py`; run everything from the repo root.

## 5. Evidence labels used in the docs

- `[VERIFIED]` seen in official IRSA docs or tested by the developer.
- `[ASSUMPTION]` reasonable but unchecked. Test it before relying on it.
- `[TODO-VERIFY]` explicitly needs a live check in the named slice.

## 6. Definition of done (per slice)

- All acceptance criteria in `docs/ROADMAP.md` for the slice pass.
- `python -m pytest -q` is green.
- Server starts cleanly; no unhandled exceptions in logs during the manual check.
- The SECURITY.md MUST items relevant to the slice are implemented and tested.
- `docs/DEVLOG.md` has an entry (what was built, what was verified, surprises, next step).
- Changes are committed.

## 7. How to report to the developer after each slice (keep it to ~6 lines)

1. What now works (one sentence).
2. How to run/see it (exact commands or URL).
3. What you verified and how.
4. Anything blocked or uncertain.
5. What the next slice is.

## 8. Things you must NOT do

- Do not let the browser supply URLs that the server fetches.
- Do not build ADQL, file paths, or shell commands from raw user strings.
- Do not use `innerHTML` with API data, URL parameters, or user input.
- Do not store anything about visitors (no cookies, accounts, analytics).
- Do not run more than one uvicorn worker (in-memory job registry; see ARCHITECTURE.md).
- Do not hold more frames in memory than the limits in `docs/SECURITY.md` allow.
- Do not silently swallow errors; map them to the error codes in `docs/API_SPEC.md`.
