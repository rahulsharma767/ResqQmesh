# ResQMesh — Backend / Data / Routing Layer (Person 2)

Mumbai emergency-response ambulance dispatch backend: real Mumbai road
network + real government hospital directory + simulated ambulance fleet
and hospital bed state, wired into a working ambulance-selection,
hospital-selection, and A* routing pipeline.

## Status: Parts 1–11 of the spec complete and tested. Parts 12–21
(HTTP dispatch endpoint, dynamic re-routing events, full test suite,
Person 1/3 integration docs) are NOT yet built — see "What's not done yet"
below.

## Environment limitation (read this first)

This was built in a sandbox with **no network access** and **none of
pyrosm / osmium / geopandas / shapely / pydantic / fastapi installed**, and
no way to `pip install` them. Every OSM PBF parsing, polygon geometry, and
API-contract-validation step below is implemented in **pure Python stdlib**
(+ `numpy`/`scipy`/`networkx`, which were available) instead. This is
documented inline in every script that had to work around a missing
library. If you run this in an environment with real network access,
swapping in `pyrosm`/`osmium` for `scripts/pbf_reader.py` and
`pydantic`/`fastapi` for `backend/app/models/emergency.py` would be a
straightforward upgrade — the surrounding code doesn't depend on *how*
those steps are implemented, only on their output shape.

## Setup

1. Unzip this package anywhere, e.g. `~/resqmesh/`.
2. Create `resqmesh/data/raw/` and place your **3 original source files**
   there, with these exact names (paths.py expects them):
   ```
   resqmesh/data/raw/hospital_directory.csv
   resqmesh/data/raw/mumbai_ward_boundary_geojson.json
   resqmesh/data/raw/western-zone-260822_osm.pbf
   ```
   (These 3 files aren't included in the zip — the PBF alone is ~220MB.)
3. Requirements: Python 3.9+ with `numpy`, `scipy`, `networkx` installed
   (`pip install numpy scipy networkx`). Nothing else — no pyrosm/osmium/
   geopandas/shapely/pydantic/fastapi needed, see limitation note above.

All paths are resolved relative to the project root via `scripts/paths.py`
— you don't need to edit any script, wherever you unzip it.

## Pipeline — run in this order

```bash
cd scripts

# Part 1 data is already inspected (see conversation history / report).
# Part 2: clean the Mumbai ward boundary (Area=="Mumbai" only)
python3 clean_boundary.py

# Part 3: clean the hospital directory down to a validated Mumbai-only set
python3 clean_hospitals.py

# Part 4: extract the Mumbai road network from the OSM PBF and build the graph
python3 extract_nodes.py            # ~40s -- nodes inside Mumbai bbox
python3 extract_ways.py             # ~35s -- highway ways referencing those nodes
python3 extract_missing_nodes.py    # ~40s -- resolve boundary-crossing node refs
python3 build_graph.py              # ~4s  -- build directed, oneway-aware edge list
python3 cache_graph.py              # ~15s -- pickle the graph for fast reload

# Part 5: connect hospitals to their nearest road-graph node
python3 connect_hospitals.py

# Part 6/7: generate SIMULATED ambulance fleet + hospital live-state
python3 seed_ambulances.py
python3 seed_hospital_state.py
```

Total pipeline runtime: **~2.5 minutes** on the full 219MB Western Zone PBF.

## Try it

```bash
cd ../tests
python3 test_dispatch_demo.py
```

This runs 3 realistic Mumbai incidents through the full pipeline: capability-
aware ambulance selection → real road routing to the incident → capability/
availability-aware hospital selection → real road routing to the hospital.
Each dispatch takes well under 5 seconds end to end (most of the code being
pure Python instead of an installed extension).

## Data: REAL vs SIMULATED

**REAL** (from your 3 uploaded files, cleaned but not fabricated):
- `data/processed/mumbai_wards.geojson`, `mumbai_boundary.geojson` — from the ward boundary GeoJSON
- `data/processed/mumbai_hospitals.csv` — from the government hospital directory
- `data/processed/mumbai_road_nodes.csv`, `mumbai_road_edges.csv` — from the OSM PBF

**SIMULATED** (clearly labeled `is_simulated=True` in every row, never presented as live data):
- `data/simulation/ambulances.csv` — no real ambulance GPS feed was provided
- `data/simulation/hospital_state.csv` — the directory has no live bed-count field

## Known limitations

1. **No true polygon dissolve** for the unified Mumbai boundary — it's a
   MultiPolygon of the 24 ward polygons, not one dissolved ring. Fine for
   point-in-polygon containment (used throughout); not fine if you need a
   single clean outline for display.
2. **Hospital directory data quality**: many rows are small clinics/nursing
   homes, not major trauma centers. The Part 10 scoring will pick whatever
   scores best among *available* options — for severe trauma cases in
   sparse areas, that can be a small clinic if nothing better routes
   quickly. This is a reflection of the real input data, not a selection
   bug — worth knowing before treating results as clinically meaningful.
3. **No pydantic/FastAPI** — `EmergencyInput` is a stdlib dataclass with
   manual validation, not a real HTTP-exposed API yet.
4. **maxspeed / road-class defaults are placeholders** documented in
   `build_graph.py` — no real traffic data source exists yet.
5. Implicit roundabout one-way-ness (junction=roundabout without an
   explicit oneway tag) is not inferred.

## What's not done yet (Parts 12–21 of the original spec)

- Part 12: actual HTTP `POST /api/v1/emergency/dispatch` endpoint (blocked
  on FastAPI not being installable here — the dispatch logic itself exists
  and is tested in `tests/test_dispatch_demo.py`, it just isn't behind an
  HTTP route yet)
- Part 13/14: road-closure/hospital-update events + re-routing endpoint
  (`find_route_excluding_edges()` in `router.py` is the building block for
  this, not yet wired to an event store)
- Part 18: the full 8-case test matrix from the spec (only 3 ad hoc
  incidents tested so far)
- Part 19/20: formal Person 1 / Person 3 integration README sections
