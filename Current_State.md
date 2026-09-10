# Current State — DroneSync / UDAN KHATOLA Services

## What this project is

A control-room dashboard for an autonomous drone **search-and-rescue** mission, plus a backend
that will eventually ingest real telemetry and mapping data from a drone/Raspberry Pi rig.

- **Frontend** (`/src`, project name `dronesync`): a React 19 + Vite single-page dashboard
  ("UDAN KHATOLA") showing live drone telemetry, a 20×20 search grid, a Leaflet map, camera
  feed placeholder, AI detections, mission status, an event log, and analysis/coverage charts
  (Recharts). It has three pages: Dashboard, Mission Control, Detection Center, and Analysis,
  plus click-to-expand popup panels for each widget.
- **Backend** (`/backend`, "UDAN KHATOLA SERVICES API"): a FastAPI service that is the intended
  source of truth for drone telemetry, occupancy-grid mapping, mission state, detections, and
  Theta* path planning. State is in-memory only.
- **Theta\*** planning: implemented on both sides — `backend/services/theta_service.py` (Python,
  used server-side against ingested occupancy maps) and `src/algorithms/thetaAlgorithm.js`
  (JS, client-side implementation of the same idea).

## Current state (what actually works today)

- The dashboard **starts fully offline** — every telemetry field shows `--` / OFFLINE until the
  backend responds. There are no mock/fake data generators; this was a deliberate "clean slate"
  (see the single commit "Initial commit - Clean Slate").
- `src/services/droneApi.js` calls a hardcoded `http://localhost:8000` backend for: drone status,
  mission, detections, path, events, markers, flight path. There is currently **no code path that
  POSTs telemetry, map data, or a goal from the frontend** — those backend endpoints
  (`/api/drone/telemetry`, `/api/map`, `/api/goal`) exist but are unused by the UI so far.
- Backend routers implemented: `telemetry` (GET/POST), `mission` (GET), `detections` (GET),
  `events`/`markers`/`flight-path` (GET), `mapping` (`POST /api/map`, `POST /api/goal`), and
  `path` (`GET /api/path`).
- Map ingestion, grid classification (FREE/WALL/OUTSIDE/UNKNOWN), coordinate conversion
  (world → source → 20×20 dashboard grid), and Theta* planning are implemented in
  `backend/services/` and covered by `backend/tests/test_grid_pipeline.py`.
- Theta* only runs once an occupancy grid, start, and goal are all present; it rejects invalid
  endpoints, diagonal corner-cutting, and blocked line-of-sight.
- No hardware integration yet: no serial/ROS/LiDAR packet parsing, no WebSockets, no database,
  no authentication. Backend README explicitly flags in-memory state as unsafe for multi-process
  deployment.
- Some UI pieces are still stubbed/placeholder: the sensor readings panel (temperature, thermal
  target, LiDAR distance, gas level, humidity, GPS accuracy) shows static `--`/`NO DATA` text,
  and the per-cell "sub-grid" survivor overlay always evaluates `isSurvivorSubCell = false`
  (no real sub-cell detection wired in yet). `CameraFeed` doesn't have a live feed source wired
  in (`fpsValue` is passed as `null`).
- Demo assets exist (`public/demo/thermal/*.jpg`, ~60 thermal images) but aren't yet referenced
  from the camera/detection components — likely intended for a future thermal-camera demo mode.

## Notable design decisions (from backend README)

- Map convention: `x` = column, `y` = row; backend indexes `occupancy[y][x]` and never
  transposes/flips.
- Grid classification thresholds are centralized in `grid_service.py` (`-1` unknown, `0` free,
  `>=65` wall).
- The 20×20 dashboard grid is an aggregation of a larger source occupancy map; any occupied
  source cell marks the whole dashboard region as WALL, and OUTSIDE is reserved for a future
  explicit search-region mask (not yet implemented — currently no cell is ever classified
  OUTSIDE).

## Likely next steps (inferred, not yet done)

- Wire the frontend to actually POST telemetry/map/goal data (currently only GET calls exist).
- Replace in-memory backend state with something persistent/safe for real deployment.
- Connect real sensor and camera/thermal feeds to replace the static placeholders.
- Implement the OUTSIDE region mask for the occupancy grid.
- Wire real survivor sub-cell detection into the grid-cell drill-down view.
