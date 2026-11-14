# DEPLOYMENT.md

Goal: one public URL, one container, deployed **early (Slice 0)** and redeployed after every slice. A deploy surprise at
the last hour is the biggest solo-hackathon risk.

## 1. Targets

Pick ONE host; both below can build from a Dockerfile in a GitHub repo. The developer has a GitHub account and no host account yet
(create one before the hackathon starts; verify the free-tier limits on the host's current pricing page, they change):

- **Render** (Web Service from a Dockerfile) or **Hugging Face Spaces** (Docker Space).
- Platform facts to confirm on day one `[TODO-VERIFY]`: free RAM limit (plan for ~512 MB), whether the disk is ephemeral
  (assume yes), sleep-on-idle behaviour, the port the platform expects (Render injects `PORT`; Hugging Face Docker Spaces
  expect the port declared as `app_port` in the Space README, commonly 7860), and **outbound access to
  `irsa.ipac.caltech.edu`** (test in Slice 0 with `scripts/smoke_deploy.py`).

## 2. Dockerfile requirements (create in Slice 0)

- Base image: official `python:3.x-slim` matching the development minor version (3.14) **if that image exists on the
  registry**; otherwise use the newest available 3.13/3.12 and run the full test-suite inside the image. Record the choice in DEVLOG.
- Install from `requirements.txt` with `--no-cache-dir`. If a wheel is missing for the chosen Python version, fall back to
  the next lower minor version and re-pin.
- Create a non-root user; copy only `backend/`, `frontend/`, `requirements.txt`, `scripts/`; set `WORKDIR /app`;
  `ENV PYTHONUNBUFFERED=1 CACHE_DIR=/tmp/cache`.
- `CMD` uses the platform port: `python -m uvicorn backend.app.main:app --host 0.0.0.0 --port ${PORT:-8000} --workers 1 --no-server-header`
  (shell form or an entrypoint script so `${PORT}` expands). **Never more than one worker.**
- Add `HEALTHCHECK` (or platform health check path) -> `GET /api/v1/health`.
- `.dockerignore`: `.venv`, `cache`, `.git`, `__pycache__`, `docs/reference`, tests optional.

## 3. Environment variables (document all in `.env.example`; none are secrets)

| Variable | Default | Meaning |
|---|---|---|
| `PORT` | 8000 | listen port |
| `CACHE_DIR` | `./cache` | disk cache root (use `/tmp/cache` in containers) |
| `CACHE_MAX_BYTES` | 1000000000 | lower to ~300 MB on small free disks |
| `CONTACT_EMAIL` | (empty) | put the developer's email into the User-Agent so IRSA can contact us |
| `TRUST_PROXY` | 0 | set 1 behind the platform proxy (rate-limit IP detection, see SECURITY.md) |
| `ENABLE_HSTS` | 0 | set 1 in production over HTTPS |
| `ENABLE_DOCS` | 1 | set 0 to hide `/docs` in production |
| `MAX_FRAMES_HARD`, `MAX_GRID_PX`, `JOB_WORKERS`, `DOWNLOAD_CONCURRENCY` | see SECURITY.md | lower these if the host runs out of RAM |
| `LOG_LEVEL` | INFO | |

If the free tier has ~512 MB RAM, start with `MAX_FRAMES_HARD=100`, `MAX_GRID_PX=400`, `JOB_WORKERS=1` and raise only after
measuring (log peak RSS once per job).

## 4. Deployment procedure

1. Slice 0: Dockerfile + minimal app (health + static page) -> push to GitHub -> connect repo on the host -> confirm the public URL
   serves the page, `/api/v1/health` is `ok`, and `scripts/smoke_deploy.py <URL> --deep` shows `irsa: ok`.
2. After each slice: push to `main`; host auto-deploys; run the smoke script; record the result in DEVLOG.
3. Before submission: run `scripts/run_targets.py` against the **deployed** URL (supports `--base-url`) and confirm all five targets.
4. Keep the previous working deploy reachable (or note the commit) so a bad deploy can be rolled back quickly.

## 5. Free-tier behaviour to plan for

- **Cold start:** the first request after idle can take tens of seconds. Before judging/demo, open the site and run a request
  (or use `scripts/warm_cache.py`), then record the demo video from a warm instance.
- **Ephemeral disk:** the cache vanishes on restart/redeploy. Mitigation (Slice 5 stretch): pre-render the five test targets into
  `demo_data/` (committed, small), seed the cache from it at startup so the headline demos work instantly and even if IRSA is down.
- **Memory pressure:** if the process is killed, lower limits per section 3.
- **IRSA outage or slowness:** the UI must show a clear message and the seeded demos remain usable.

## 6. Smoke test script (`scripts/smoke_deploy.py`)

Arguments: `base_url`, optional `--deep`. Checks: `GET /` returns HTML with expected title; `GET /api/v1/health` ok; `GET /api/v1/meta`
ok; `--deep` calls `/health?deep=1`; creates a small sequence (WISE 0855, FOV 8, detector 5, `max_frames=10`), polls until done or 5 minutes,
fetches frame 0 and verifies it is a PNG. Exit code non-zero on any failure. Use only `httpx` and the standard library.

## 7. Release checklist (before submitting)

- [ ] Public URL works from a clean browser (no cache) and from a phone.
- [ ] All five targets verified on the deployed instance (TEST_TARGETS.md).
- [ ] README: what it is, live URL, screenshots/GIF, how to run locally, how updates work, data credits, limitations, license.
- [ ] 2-3 minute demo video recorded from a warm instance.
- [ ] `.env.example` and `requirements.txt` committed; no secrets; `cache/` not committed.
- [ ] SECURITY.md MUST items checked off in DEVLOG.
- [ ] Re-check the official Space Apps submission requirements (links, formats, deadline) and satisfy them.
