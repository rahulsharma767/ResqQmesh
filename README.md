# ResQMesh — Integrated (Frontend + Person 1 LLM layer + Person 2 real-data backend)

Three previously-separate pieces wired into one app **without redesigning the
existing UI**:

- `frontend/` — the existing React/Vite dispatcher UI (unchanged layout/theme;
  only additive wiring: a Person 1 API call that already existed, plus a new
  "Live Backend Verification" panel that shows what Person 2's real backend
  independently decided).
- `person1_backend/` — Person 1's LLM incident-extraction pipeline
  (`raw text -> IncidentState`), exposed over FastAPI at `/api/analyze`.
  Logic untouched.
- `person2_backend/` — the real Mumbai-data dispatch engine (475,832-node
  road graph, 921 routable hospitals, 150 simulated ambulances, A* routing,
  road closures). Logic untouched; only `backend/app/main.py` was extended
  (CORS + a few read/state endpoints — see "What changed" below).

```
USER REPORT (raw text)
        |
frontend (existing UI, unchanged)
        |
        v
Person 1  /api/analyze          -> IncidentState (location text, severity cues, etc.)
        |
        v
frontend maps IncidentState -> local demo pipeline (existing, unchanged)
AND in parallel:
frontend -> Person 2 /api/v1/emergency/dispatch (NEW wiring)
        |
        v
Person 2 (authoritative): ambulance selection, A* routing, hospital
selection, hospital routing -> real DispatchResponse
        |
        v
frontend "Live Backend Verification" panel (NEW, additive) shows the real
result next to the existing demo pipeline's result — never replacing it.
```

## Why two backends run on different ports

Both `person1_backend` and `person2_backend` previously defaulted to port
**8000**. That's the concrete bug that made "HTTP not working": whichever
process you started second either failed to bind the port, or the frontend
silently talked to the wrong one. Fixed:

| Service          | Port |
|-------------------|------|
| Person 1 (LLM)     | 8001 |
| Person 2 (dispatch)| 8000 |
| Frontend (Vite)    | 5173 |

Person 2's `main.py` also had **no CORS middleware**, so even with ports
fixed, the browser blocks every request from `http://localhost:5173` before
the frontend ever sees a response — that alone looks exactly like "the API
isn't working" even though a direct `curl`/Python call succeeds. Both are
fixed in this package.

## What changed, file by file

- `person2_backend/backend/app/main.py`
  - **Added** `CORSMiddleware` (env-configurable via `CORS_ALLOWED_ORIGINS`,
    same pattern Person 1 already used).
  - **Added** `GET /api/v1/ambulances`, `GET /api/v1/hospitals`,
    `GET /api/v1/hospitals/{id}`, `PATCH /api/v1/hospitals/{id}/state`.
    All additive — they call your existing `load_ambulances()` /
    `load_hospital_data()`, no changes to those files.
  - `POST /api/v1/emergency/dispatch` now loads hospital state itself and
    merges any live `PATCH` overrides on top before calling your existing
    `dispatch()` — so a hospital-state update actually affects the next
    dispatch decision. `dispatch()` itself (ambulance selection, A*, hospital
    selection) is **untouched**.
  - Nothing else in `person2_backend` was modified. `router.py`,
    `ambulance_selection.py`, `hospital_selection.py`, `road_closures.py`,
    `emergency.py` are byte-for-byte what you had.
- `frontend/ResQMesh.jsx`
  - `PERSON1_API_URL` default changed `:8000` → `:8001`.
  - **Added** `PERSON2_API_URL`, a real Mumbai lat/lon table for the demo
    locality names, `dispatchViaPerson2Backend()`, and a new
    `BackendVerificationPanel` shown alongside the existing panels.
  - Nothing else changed: same components, same layout, same styling, same
    N1–N22 demo graph/route visualization, same mock ambulance/hospital
    lists driving the existing panels.
- `person1_backend/` — **no code changes**. Only the recommended run port
  (8001 instead of 8000) is new, documented below.

## What I could and couldn't verify here

I don't have network access or `npm`/`fastapi` installed in this sandbox, so
I could not literally boot `uvicorn`/`vite` end-to-end. What I did verify:
- Ran your real `dispatch()` function directly (bypassing HTTP) against your
  actual 475,832-node graph and hospital/ambulance CSVs — it returned a real
  ambulance, hospital, full route geometry, and reasoning in ~6s, including
  with a simulated hospital-state override applied.
- Confirmed the new `/api/v1/hospitals` filter matches exactly 921 routable
  hospitals (your documented number).
- Both `main.py` and `ResQMesh.jsx` pass a syntax/bundle check (`py_compile`
  and `esbuild`, respectively).

**You should still run the full local test list below** before considering
this done — that part needs your machine.

## Setup (Windows)

### 1. Person 2 (real-data dispatch backend) — port 8000
Your existing `data/` folder (with `mumbai_road_graph.pkl` etc.) is **not**
included in this package (too large / regenerable — see `.gitignore`). Copy
`person2_backend/backend`, `person2_backend/scripts`, and
`person2_backend/tests` over your existing local `resqmesh_backend/` folder
so they sit next to your existing `data/`.

```
cd resqmesh_backend
py -3.11 -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt
py -3.11 -m uvicorn app.main:app --app-dir backend --host 127.0.0.1 --port 8000
```
Health check: http://127.0.0.1:8000/health
Swagger UI: http://127.0.0.1:8000/docs

### 2. Person 1 (LLM incident extraction) — port 8001
```
cd person1_backend
python -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt
copy .env.example .env      REM defaults to PERSON1_MODE=mock, no API key needed
uvicorn api:app --reload --port 8001
```
Health check: http://127.0.0.1:8001/api/health
Swagger UI: http://127.0.0.1:8001/docs

### 3. Frontend — port 5173
```
cd frontend
copy .env.example .env
npm install
npm run dev
```
Open http://localhost:5173

## Environment variables

| File | Variable | Default | Purpose |
|---|---|---|---|
| `frontend/.env` | `VITE_PERSON1_API_URL` | `http://localhost:8001` | Person 1 API |
| `frontend/.env` | `VITE_PERSON2_API_URL` | `http://localhost:8000` | Person 2 API |
| `person1_backend/.env` | `PERSON1_MODE` | `mock` | `mock` = offline heuristics, `real` = calls `LLM_PROVIDER` |
| `person1_backend/.env` | `LLM_PROVIDER`, `LLM_API_KEY`, `LLM_MODEL` | — | only needed if `PERSON1_MODE=real` |
| `person1_backend/.env` | `CORS_ALLOWED_ORIGINS` | `http://localhost:5173,http://127.0.0.1:5173` | |
| `person2_backend/.env` | `CORS_ALLOWED_ORIGINS` | `http://localhost:5173,http://127.0.0.1:5173` | |

## API endpoints (Person 2, port 8000)

```
GET    /health
POST   /api/v1/emergency/dispatch
GET    /api/v1/ambulances                     (new)
GET    /api/v1/hospitals                      (new)
GET    /api/v1/hospitals/{hospital_id}        (new)
PATCH  /api/v1/hospitals/{hospital_id}/state  (new)
GET    /api/v1/roads/closures
POST   /api/v1/roads/closures
POST   /api/v1/roads/reopen
DELETE /api/v1/roads/closures
POST   /api/v1/roads/reroute
```

Person 1 (port 8001): `GET /api/health`, `POST /api/analyze`.

## Test checklist (run on your machine, in order)

1. `person2_backend`: `pytest tests/` — your existing dispatch demo test.
2. `person2_backend` running → open `/docs`, try `POST /api/v1/emergency/dispatch`
   with the `EXAMPLE_EMERGENCY_INPUT` from `models/emergency.py` → expect 200
   with a real ambulance/hospital/route.
3. `curl http://127.0.0.1:8000/api/v1/ambulances` → 150 ambulances.
4. `curl http://127.0.0.1:8000/api/v1/hospitals` → 921 hospitals.
5. `person1_backend` running → `person1_backend`: `python run_tests.py` (existing tests).
6. `POST http://127.0.0.1:8001/api/analyze {"raw_text": "..."}` → 200 with
   `incident_state` + `ambiguity_result`.
7. Frontend running, both backends running → submit or pick a demo scenario
   in the UI → confirm the existing pipeline still animates through exactly
   as before, **and** the new "Live Backend Verification" panel shows
   `REAL DATA` with a real `AMB-0xx` / real hospital name / real ETA.
8. Stop `person2_backend` only → resubmit an incident → panel should show
   `UNREACHABLE` with the restart command, and the rest of the UI should
   continue working exactly as it did before this integration (no crash).
9. `POST /api/v1/roads/closures {"u": <node>, "v": <node>}` (use a `node_path`
   value from step 2's response) → `POST /api/v1/roads/reroute` with the same
   origin/destination → confirm a different route or `found: false` if none
   exists → `DELETE /api/v1/roads/closures` to clear.
10. `PATCH /api/v1/hospitals/{id}/state {"status": "FULL"}` → dispatch an
    incident whose only good hospital match was that one → confirm the
    backend picks a different hospital.

## Known limitations / not yet done

- The existing N1–N22 demo route visualization is **not** re-driven by
  Person 2's real polyline/node-path data — per your "don't redesign the
  map" instruction, mapping a 475k-node real graph onto a 22-node stylized
  demo map would require inventing a new visual, not adapting the existing
  one. The real route/ETA/reasoning is surfaced as data (Live Backend
  Verification panel + `routes` field in the raw API response) rather than
  redrawn on the existing stylized graph.
- Ambulance-failure / hospital-capacity-drop simulated events in the
  existing UI still only mutate local demo state, not Person 2's fleet —
  wiring those is a natural next step using the new
  `PATCH /hospitals/{id}/state` endpoint but wasn't in scope of "fix the
  HTTP problem and integrate."
- I could not run `npm install` / `uvicorn` myself (no network in this
  environment) — steps 1–10 above are load-bearing and untested by me
  beyond direct-function-call-level verification.

## Git — ready to commit

```
cd ResQMesh_integrated
git init                          # only if not already a repo
git add .
git status                        # sanity check — data/, node_modules/, .env, __pycache__ should NOT appear
git commit -m "Integrate frontend + Person 1 LLM layer + Person 2 real-data backend; fix port collision and missing CORS"
git remote add origin <your-repo-url>
git push -u origin main
```

Large files intentionally excluded (see `.gitignore`), expected locally at:
- `person2_backend/data/data/raw/western-zone-260822.osm.pbf`
- `person2_backend/data/processed/mumbai_road_graph.pkl`
- `person2_backend/data/*` generally (regenerate via `scripts/` or keep your
  existing local copy — never committed)
