# TODO — DroneSync / UDAN KHATOLA

Working document. Check items off as they land. Log anything broken in `bugs.md`.

**Status legend:** `[ ]` not started · `[~]` in progress · `[x]` done & tested · `[!]` blocked

---

## Phase 0 — Ground rules (read before starting)

These exist so the work stays cheap and correct. See §7 for the full reasoning.

- Backend map convention is fixed: `x` = column, `y` = row, indexed `occupancy[y][x]`, never
  transposed or flipped. Any new producer must match this or the grid renders mirrored.
- Cell classification thresholds live only in `grid_service.py` (`-1` unknown, `0` free,
  `>=65` wall). Do not re-derive them in the frontend.
- The 20×20 dashboard grid (resized from the originally-planned 10×10 per user request — see
  §2.0) is an **aggregation** of a larger source occupancy map. Never treat the 20×20 as the
  real map — it is a view. All detail work reads from the source grid.
- One change at a time, tested, before moving on. Phases 1→4 are ordered deliberately.
- Offline persistence is **SQLite** (WAL mode, single writer). Maps and telemetry live in the
  DB; video lives on the filesystem with only its path in the DB. Full spec in §3.5.
  Phases 1–2 keep in-memory state — do not introduce the DB early, it is Phase 3 work.

---

## 1. Temporary backend — receive the map from the RPi

Goal: get a real occupancy grid from the Pi 5 into the existing FastAPI service over plain
Wi-Fi/Ethernet, with no radio link involved yet. This is throwaway plumbing that proves the
data contract before the telemetry receiver is introduced.

> **⚠️ Transport changed 2026-09-05 — see `ssh_transport.md`.** The original design (Pi POSTs
> inbound to the laptop) never delivered in practice (BUG-012, root cause never isolated). The
> link is now **SSH, pulled by the PC**: the PC opens one outbound `ssh` session, the Pi's
> publisher writes NDJSON payloads to stdout, and `tools/ssh_map_bridge.py` feeds them to the
> backend over loopback. The payload contract (§1.1), `POST /api/map`, aggregation and the whole
> dashboard are **unchanged** — only the delivery mechanism moved.

- [x] **1.1** Freeze the map payload schema (this is the contract everything else depends on).
      Implemented in `backend/models/schemas.py` (`OccupancyMapRequest`) and documented in
      `backend/README.md`:
      ```
      {
        "timestamp": <float, unix epoch>,
        "resolution": <float, metres per cell>,
        "origin": {"x": <float>, "y": <float>},   // world coords of occupancy[0][0]
        "width": <int>, "height": <int>,
        "occupancy": [[int, ...], ...]            // row-major, occupancy[y][x], -1|0..100
      }
      ```
      Not truly "generated" on both sides (the Pi only gets `tools/map_publisher.py` copied to
      it, not the backend package) — instead `tools/map_publisher.py` carries a
      `validate_payload_shape()` that mirrors the pydantic model and is commented to point back
      here; `tools/tests/test_map_publisher.py` and `backend/tests/test_grid_pipeline.py` both
      exercise the same fixture to keep the two from drifting silently.
- [x] **1.2** Confirmed: `POST /api/map` rejects wrong row count, ragged rows, missing fields,
      non-numeric values, and non-positive width/height/resolution — see
      `test_invalid_shape_and_resolution` and `test_missing_and_non_numeric_fields_rejected` in
      `backend/tests/test_grid_pipeline.py`.
- [x] **1.3** Decided **replace**, documented in `backend/README.md` ("Replace, not merge"):
      every `POST /api/map` overwrites the full stored map; no server-side accumulation.
- [x] **1.4** `tools/map_publisher.py` written: env-var + CLI config, ~1–2 Hz rate limit (115200
      baud documented), and a pure `points_to_occupancy()` rasterizer covered by
      `tools/tests/test_map_publisher.py`. Now supports three sinks — `stdout` (default; NDJSON
      for the SSH transport), `file` (atomic tmp+rename so a reader never sees a partial file),
      and `http` (the original POST path, kept for bench tests) — plus `--source replay` for
      proving the link without the LiDAR attached. **LiDAR bindings corrected 2026-09-05**
      against the SDK's own `python/examples/tri_test.py`: four settings were wrong, three
      fatally (`os_init()` never called, `SingleChannel` False→True, `SampleRate` 9→3,
      `ScanFrequency` 8.0→10.0) — see BUG-009 for the full diff. Also added `--probe` (dumps the
      installed SDK's real API surface + live scan shape) and explicit SDK-side range clipping.
      **Confirmed on real hardware 2026-09-05**: `--probe` on the Pi reports 252 points/scan,
      164 in range, angles spanning ±3.14 rad (SDK 1.2.20). Two further defects were found and
      fixed during that bring-up — the SDK writing its logs to fd 1 and corrupting the NDJSON
      stream (BUG-014, 24 payloads lost), and the grid being too small for `--max-range` so
      distant returns vanished (BUG-015). Also: this treats every revolution as an independent grid with no
      pose tracking/accumulation across scans (full SLAM is out of scope for this phase) — see
      the docstring for the explicit limitation.
- [x] **1.4b** `tools/ssh_map_bridge.py` (PC side, new): one persistent `ssh` session reading
      NDJSON → `POST http://127.0.0.1:8000/api/map`, with reconnect backoff, remote-stderr
      forwarding, and `received/accepted/rejected/reconnects/last` link stats. `BatchMode=yes`
      so a missing key fails loudly instead of hanging on an invisible password prompt;
      `ServerAliveInterval` so a drone flying out of range surfaces as a dropped connection
      rather than a silent stall. 10 unit tests in `tools/tests/test_ssh_map_bridge.py`; verified
      end-to-end locally via `--local-command` (31/31 payloads accepted, backend-down and
      malformed-line paths exercised).
- [~] **1.5** `tools/replay_map.py` written and smoke-tested end-to-end against a live backend
      instance (stdlib-only, no extra installs needed on the dev machine). The fixture itself,
      `backend/tests/fixtures/map_sample.json`, is a **synthetic placeholder** (asymmetric
      perimeter + interior wall + unknown fringe, matching the frozen contract), not yet a real
      captured scan — swap it for a real one per the note in `backend/tests/fixtures/README.md`
      once the Pi rig is running. Frontend Phase 2 work can proceed against the replay now.
- [x] **1.6** Network sanity — **done 2026-09-05**: `ssh` from the PC reaches the Pi and a live
      LiDAR stream ran end-to-end (`received=60 accepted=60 rejected=0 noise=0`). Note the Pi's
      address moved between sessions (BUG-013) — give it a DHCP reservation. Original note kept:
      the bar has changed: the only requirement now is that **`ssh` from the PC reaches the Pi**.
      No inbound port, no firewall rule, no laptop-IP discovery. Note this does *not* dodge AP
      **client isolation** — that blocks device-to-device traffic in both directions regardless
      of protocol, so a hotspot / home router / direct Ethernet is still the answer if `ping`
      itself fails. Checklist and troubleshooting table: `ssh_transport.md` §7.
- [x] **1.7** `GET /api/path` verified end-to-end **with real LiDAR data 2026-09-05**: live scans
      travelled Pi -> ssh -> bridge -> `POST /api/map` -> `GET /api/path`, with the dashboard grid
      hash changing every sample (`55c28283dd -> 44d7b93fc3 -> b948b05089`) and correct live
      dimensions (200x200 @ 0.05 m, not the fixture's 30x20 @ 0.1 m). Map *quality* is a separate
      open problem — see BUG-016.
- [x] **1.8** Pi-side setup instructions captured in **`pi_setup.md`** — OS prep, UART
      configuration, YDLidar SDK build, udev rules, Python environment, publisher deployment,
      and link verification. Keep it updated as the rig changes; §4 in particular has blanks to
      fill in once the UART conflict is resolved.

**Done when:** a real LiDAR scan taken on the Pi appears via the backend's GET endpoint with
correct dimensions and no transposition, and `pi_setup.md` is complete enough that a teammate
can reproduce the rig from a fresh SD card without asking questions.

---

## 2. Integrate the map into the grid layout + per-tile drill-down

Goal: the 20×20 grid shows a recognisable partial outline of the scanned room, and clicking a
tile expands it into a detailed view of that tile's source-map region.

### 2a. Main grid overlay

- [x] **2.0** **Resize the dashboard grid.** Overridden by the user to **20×20** (not the
      10×10 originally suggested here). Done as a real code change on both sides via a single
      named constant, not a doc-only update:
      - Backend: `GridConfig.rows`/`GridConfig.columns` default in `backend/services/grid_service.py`
        (20/20). `state_service.py`'s `dashboard_config = GridConfig()` picks it up automatically.
      - Frontend: `GRID_ROWS`/`GRID_COLUMNS` added in `src/utils/gridUtils.js` as the single
        source of truth; `App.jsx` (3× `SearchGrid` usages, `totalGridCells`, the
        `normalizeOccupancyGrid` call), `SearchGrid.jsx`'s default props, `src/data/mapConfig.js`,
        and `src/algorithms/thetaAlgorithm.js`'s defaults all import from there instead of a
        literal `15`/`10`. Also removed a stale, already-dead `grid-template-rows: repeat(15, 1fr)`
        in `SearchGrid.css` (inline style always overrides it).
      - Resolution consequence is the **opposite** of the note this replaced: going 15→20 is
        *finer* aggregation (~0.75 m/cell over a ~15 m room, not coarser), so §2.4's
        chunky-over-walled concern is less pressing here than it would have been at 10×10 — still
        worth doing for legibility, just lower urgency.
      - Verified: backend test suite (9 tests) green with the new default; `npm run build` and
        `npm run lint` clean; dev server + backend started live, a map POSTed via
        `tools/replay_map.py`, and the rendered dashboard screenshotted showing a genuine "20 × 20
        GRID" badge with the fixture's occupancy pattern correctly aggregated onto it.
- [x] **2.1** No dedicated `getMap()` was needed — `GET /api/path` already returns
      `occupancyGrid` + `metadata` bundled (see `droneApi.js`'s existing `getPath()`). Added
      `DroneApi.getMapCell(x, y)` instead, for the new per-tile drill-down endpoint (§2.6).
- [x] **2.2** `App.jsx` previously fetched everything **once on mount and never again** — there
      was no polling loop at all despite this item's wording assuming one existed. Added one:
      `loadBackendData()` now runs on mount and every `POLL_INTERVAL_MS` (2000ms, matching the
      Pi publisher's ~1-2 Hz), with `clearInterval` on unmount. All existing fetches (status,
      mission, detections, markers, flight path, events, path/map) now refresh together through
      this single loop, not just the map.
- [x] **2.3** Confirmed already true: `SearchGrid.jsx`/`SearchGrid.css` render FREE (faint),
      WALL (dark), UNKNOWN (grey, distinct from FREE), and OUTSIDE (hatched) as visually
      distinct states, and did not need new code.
- [x] **2.4** Implemented. Backend: `grid_service.source_grid_to_density()` computes an
      occupied-ratio (0..1) per dashboard region alongside the existing WALL/FREE/UNKNOWN
      classification; exposed as `PathData.densityGrid` next to `occupancyGrid`. Frontend:
      `SearchGrid` takes a `densityGrid` prop and sets a WALL cell's opacity proportional to its
      density (floor 0.35 so a barely-occupied region is still visible as WALL, not invisible).
- [x] **2.5** Confirmed already true: `getOccupancyClass` returns `'unmapped'` when
      `occupancyGrid` is null, rendering the same as UNKNOWN rather than crashing or blanking.
- [x] **(bonus, found while testing 2.3)** Fixed two live bugs uncovered doing this: `.grid-cell`
      applied `scanned-cell`/`drone-cell` classes that no CSS rule matched (scanned cells never
      looked different from unscanned ones — BUG-010), and clicking a cell **directly** on the
      main dashboard grid set `expandedPanel` to `'gridCell'`, a value the popup's render switch
      never handled, opening an empty popup (BUG-011, this made the drill-down below
      unreachable from its main entry point until fixed). Both fixed; see `bugs.md`.

### 2b. Tile drill-down

- [x] **2.6** Added `GET /api/map/cell/{x}/{y}` (`backend/routers/mapping.py`,
      `BackendState.get_cell_detail` in `state_service.py`). Reuses a new
      `grid_service.dashboard_region_bounds()` helper — the exact reverse of the forward
      aggregation math already in `source_grid_to_dashboard`/`source_grid_to_density` — so the
      forward and reverse mappings are structurally guaranteed to agree, not just tested to
      (backed by `test_dashboard_region_bounds_matches_aggregation`). Returns the sub-block
      occupancy, world-coordinate bounds, and free/wall/unknown counts; 404s cleanly when there's
      no map yet or the cell is out of bounds.
- [x] **2.7** Wired into the existing click-to-expand popup: the drill-down overlay
      (`cell-subgrid-overlay`/`cell-subgrid-panel`, already built in the popup) now fetches real
      data via `DroneApi.getMapCell` in a `useEffect` keyed on `selectedGridCell`, instead of a
      static dummy 5×5 grid with everything hardcoded.
- [x] **2.8** Expanded view now renders: real sub-block occupancy at full source resolution
      (dynamic subdivision size, not a fixed 5×5), real world-coordinate area
      (`(maxX-minX) × (maxY-minY)` in metres, computed from `worldBounds`), and real free/wall/
      unknown counts from the backend.
- [x] **2.9** `isSurvivorSubCell` is no longer an inline `false` literal — routed through
      `isSurvivorInSubCell(subRow, subCol, cellDetail)` in `src/utils/gridUtils.js`, still
      stubbed to always return false, but now a single place to wire real detections into later.
- [x] **2.10** Loading/no-data/error states added (`cellDetailStatus`: `loading` | `ok` |
      `no-data` | `error`), each with its own message in the popup instead of spinning forever or
      showing stale/dummy content.

**Done when:** the replayed fixture produces a room outline recognisable against the actual
room, and every tile opens a correct detail view. **The tile-detail-view half is verified now**
(live browser check: clicking a cell shows real occupancy/world-bounds/counts, loading and
no-data states work). The "recognisable against the actual room" half is still open — it can
only be judged once `map_sample.json` is a real capture (§1.5/§1.6), not the synthetic
placeholder it is today.

---

## 3. Permanent backend — map over the telemetry receiver

Goal: same map data, same dashboard integration, but arriving over the radio link instead of
Wi-Fi.

> **⚠️ Open question — partially resolved 2026-09-05.** User confirmed the receiver is a
> **dedicated data radio** (its own serial link, independent of the flight-control RC/ELRS
> link) — not the ELRS/CRSF backchannel, so the "can't carry a full occupancy grid at all"
> worst case in the original note doesn't apply. Exact model, baud rate, and **actual measured**
> bandwidth are still unknown — that part of §3.1 needs the real hardware in hand, which isn't
> available in this environment. §3.2's final transport choice still can't be made until that
> measurement exists; treat "probably enough bandwidth for something better than raw CRSF-style
> telemetry, but unconfirmed" as the working assumption, not a green light to assume full-grid
> transport is fine.

- [ ] **3.1** Receiver category confirmed (dedicated data radio, see above). Still open: exact
      model, link protocol, serial port, and **actual measured** usable payload bandwidth
      (measure it; do not trust the datasheet figure) — needs the physical rig.
- [ ] **3.2** Given that bandwidth, choose the map transport strategy:
      - **(a) Delta/patch updates** — send only changed cells since last ack. Best fit for a
        slowly-changing indoor map.
      - **(b) Downsampled grid** — transmit at reduced resolution, accept a coarser dashboard.
      - **(c) Split channels** — full map over Wi-Fi when in range, structured summary
        (frontier points, detected targets, pose) over the radio as the always-available
        fallback.
      Option (c) is the most likely correct answer for a rescue system; degrade gracefully
      rather than lose the map entirely when Wi-Fi drops.
- [ ] **3.3** Serial ingestion service in the backend: read frames from the receiver, validate,
      decode into the **same internal map representation Phase 1 produced**. The frontend must
      not be able to tell which transport delivered the data — if it can, the abstraction is
      wrong and Phase 2 work will need redoing.
- [ ] **3.4** Frame integrity: length prefix + checksum, and drop malformed frames rather than
      partially applying them. A half-applied map update is worse than a stale one.
- [x] **3.5** Implemented in `backend/services/db_service.py` (`Database` class) + wired into
      `state_service.BackendState` (map/telemetry updates now also record durably). This is a
      **recording layer alongside** the existing in-memory live state, not a wholesale replacement
      of it — `BackendState`'s in-memory fields still serve live dashboard reads (fast, simple);
      the DB is the durable, timestamped history that §3.3's future serial reader needs
      somewhere safe to land data regardless of process topology. `BackendState` now takes an
      optional `db` param (`None` for plain test instances — keeps the existing test suite
      disk-free and fast; the production singleton passes the real `Database`). Verified: 7 new
      unit tests (round-trip, mission export scoping, concurrent writes) + a live end-to-end
      check (POST map + telemetry, `GET/POST /api/mission*`, downloaded and re-opened the
      exported `.db` file, confirmed correct tables/rows).

  - [x] **3.5.1** Enable WAL at connection open, on every process that touches the DB:
        ```python
        conn.execute("PRAGMA journal_mode=WAL")
        conn.execute("PRAGMA synchronous=NORMAL")
        conn.execute("PRAGMA foreign_keys=ON")
        ```
        Without WAL, the serial reader's writes will block API reads and the dashboard stalls.
  - [x] **3.5.2** Schema. Store each map snapshot as **one row with a compressed blob** —
        never one row per cell. A 300×300 grid is 90,000 cells; at 2 Hz that is ~180k
        inserts/sec and it will destroy the SD card.
        ```sql
        CREATE TABLE map_snapshot (
          id         INTEGER PRIMARY KEY,
          ts         REAL    NOT NULL,
          resolution REAL    NOT NULL,
          origin_x   REAL, origin_y REAL,
          width      INTEGER NOT NULL, height INTEGER NOT NULL,
          occupancy  BLOB    NOT NULL      -- int8 row-major, zlib/zstd compressed
        );
        CREATE INDEX idx_map_ts ON map_snapshot(ts);

        CREATE TABLE telemetry (
          id INTEGER PRIMARY KEY, ts REAL NOT NULL,
          x REAL, y REAL, heading REAL, battery REAL, link_quality REAL
        );
        CREATE INDEX idx_telem_ts ON telemetry(ts);

        CREATE TABLE detection (
          id INTEGER PRIMARY KEY, ts REAL NOT NULL,
          target_id TEXT, confidence REAL, world_x REAL, world_y REAL,
          grid_x INTEGER, grid_y INTEGER
        );

        CREATE TABLE video_segment (
          id INTEGER PRIMARY KEY,
          path TEXT NOT NULL, start_ts REAL NOT NULL, end_ts REAL,
          source TEXT      -- 'rotg02' | 'pi_csi'
        );

        CREATE TABLE mission (
          id INTEGER PRIMARY KEY, started_ts REAL NOT NULL, ended_ts REAL, notes TEXT
        );
        ```
  - [x] **3.5.3** Used stdlib `array('b', ...)` + `zlib` instead of `np.int8` — **deviation**:
        backend has no numpy dependency and didn't seem worth adding one just for this when
        `array` gives the identical fixed-width signed-byte packing from the stdlib. Codec name
        is stored **per-row** (`map_snapshot.codec`) rather than in a separate `meta` table —
        functionally equivalent for "the format can change later without orphaning old
        recordings" (each row is self-describing), just not literally the schema in the sketch.
        Round-trip byte-identity verified in `test_map_snapshot_round_trip_is_byte_identical`.
  - [x] **3.5.4** **Video bytes never go in the database** — only paths, in `video_segment`.
        See §4.8. Blobs in SQLite break normal playback and seeking, and one corrupt write
        costs the whole mission instead of one segment. Table + `record_video_segment`/
        `close_video_segment` exist; nothing calls them yet since Phase 4 (video) isn't built.
  - [x] **3.5.5** Single writer: one `Database` instance behind a `threading.Lock`, and currently
        only one process (this API server) ever writes. `test_concurrent_writes_do_not_lock_or_raise`
        exercises 3 threads hammering writes concurrently. When §3.3's serial reader becomes a
        genuinely separate **process** (not just a thread), the single-writer guarantee must be
        re-examined — a `threading.Lock` in this process doesn't protect against a second OS
        process writing the same file; see the note added to `backend/README.md`.
  - [x] **3.5.6** Migrations: `schema_version` table + an ordered `_MIGRATIONS` list in
        `db_service.py`, applied from the stored version forward on every connection open.
  - [x] **3.5.7** Retention/export: `GET /api/mission/{id}/export` implemented — scopes every
        table by matching its timestamp column against the mission's `[started_ts, ended_ts]`
        window (there's no mission-id foreign key in the given schema, so this is the only way
        to scope it), writes a fresh single-file SQLite export, and streams it back as a
        download. Verified live: downloaded, re-opened, confirmed correct tables and rows.
  - [ ] **3.5.8** **Storage medium:** if recording on the Pi rather than the ground station, put
        the DB and video on a **USB SSD**. Sustained SQLite writes will wear out a microSD card.
        Physical hardware decision — not applicable until the Pi is actually recording.
  - [ ] **3.5.9** **Clock sync.** The Pi has no RTC and boots near epoch zero until NTP settles,
        so map/video/telemetry timestamps will not line up and scrubbing video against a map
        snapshot silently breaks. Either run NTP against the laptop, or capture a Pi↔laptop
        offset at session start and record it in `mission`. Not done — needs the real Pi to
        measure the actual offset; `mission.notes` could hold it once known, no schema change
        needed.
  - [ ] **3.5.10** Overlap worth exploiting: if §3.2 lands on delta encoding for the radio link,
        the same keyframe+patch format works as the storage format. Build it once. Blocked on
        §3.2's transport decision, which is blocked on §3.1's real bandwidth measurement.
- [ ] **3.6** Link-health indicator in the dashboard: last-packet-received age, dropped-frame
      count, current transport. An operator must be able to tell a stale map from a live one.
- [ ] **3.7** **RF check:** the Pi 5's onboard 2.4 GHz Wi-Fi/BT can interfere with a 2.4 GHz
      control link. If both are active, test range with Wi-Fi enabled and disabled, and
      consider physical separation or a 915/868 MHz variant.
- [ ] **3.8** Retire the Phase 1 HTTP path or keep it behind a config flag as the bench-test
      transport. Do not leave two live ingestion paths writing to the same state unguarded.

**Done when:** the dashboard renders the map identically to Phase 2, sourced from the radio
link, with no frontend changes required.

---

## 4. Video feed — Eachine ROTG02 receiver

Goal: live video in the dashboard's camera feed panel, replacing the placeholder
(`CameraFeed` currently has no source and `fpsValue` is passed as `null`).

> **⚠️ Architectural conflict — decide first.** The ROTG02 is a **5.8 GHz analog FPV** OTG
> receiver. It requires an analog camera + VTX on the airframe. That is a completely separate
> video path from the existing Pi camera → `rpicam-vid` → GStreamer → RTP/UDP digital pipeline.
> Running both means two cameras, two transmitters, and extra payload weight.
> **Pick one as primary before writing code**, or explicitly decide the analog link is the
> low-latency pilot view and the digital stream is the ML/detection source.

- [ ] **4.1** Plug the ROTG02 into the ground-station laptop and identify how it enumerates.
      On Windows it typically presents as a **UVC video device** (i.e. a webcam). Confirm this —
      it determines the entire integration approach.
- [ ] **4.2** If UVC: capture it **directly in the browser** via `navigator.mediaDevices
      .getUserMedia()` with a `deviceId` constraint. No backend, no transcoding, no extra
      latency. This is by far the cheapest correct path — try it before anything else.
- [ ] **4.3** Device picker in the UI: enumerate video inputs, let the operator select the
      ROTG02, persist the choice. Device IDs change between sessions; do not hardcode one.
- [ ] **4.4** Only if 4.2 fails: fall back to a backend capture → re-stream approach. Note this
      adds meaningful latency and complexity — treat it as a last resort, not a default.
- [ ] **4.5** Wire `fpsValue` to a real measured framerate rather than the current `null`.
- [ ] **4.6** Handle signal loss the way analog actually fails: static/noise, not a clean
      disconnect. The UI needs a "no signal" state that a noisy frame still triggers —
      a black-frame or brightness heuristic, plus a manual operator override.
- [ ] **4.7** Decide what the ~60 thermal demo images in `public/demo/thermal/` are for. Either
      wire them as a demo/offline mode toggle for the camera panel, or delete them. Unused
      assets that look wired-in will mislead whoever reads this next.
- [ ] **4.8** **Recording to disk** (pairs with §3.5.4). Segment continuously to ~60-second
      files and insert a `video_segment` row per file.
      - Use **MKV or fragmented MP4 — not standard MP4.** A plain MP4 writes its index at the
        end of the file, so a power loss mid-recording makes the *entire* recording unplayable.
        MKV degrades to "playable up to the cut," which is what you want on a drone.
      - Filename pattern `mission_<id>/<source>_<start_ts>.mkv`, so files remain identifiable
        if the DB is ever lost.
      - Write `end_ts` on segment close; a NULL `end_ts` marks an in-progress or crashed
        segment and is useful for recovery.
      - Disk-space guard: stop recording (and surface it in the UI) before the volume fills.
        A full disk that silently stops writing is worse than a warned-about one.
- [ ] **4.9** Playback: given a map snapshot `ts`, find the covering `video_segment` and seek to
      `ts - start_ts`. This only works if §3.5.9 clock sync is done — verify it before building
      the scrub UI.

**Done when:** live video renders in the dashboard panel with a working no-signal state.

---

## 5. Testing — before anything is marked done

Nothing gets `[x]` without passing its phase gate.

- [ ] **5.1** Extend `backend/tests/test_grid_pipeline.py` with the real captured fixture from
      1.5, not only synthetic grids. (`test_map_sample_fixture_is_valid` already exercises the
      current *synthetic* fixture — swap the fixture file for a real capture and this test
      keeps working unchanged.)
- [x] **5.2** Payload validation tests: wrong dimensions, ragged rows, out-of-range values,
      empty occupancy, missing fields. All must be rejected cleanly with a useful error.
      (`test_invalid_shape_and_resolution`, `test_missing_and_non_numeric_fields_rejected`.)
- [x] **5.3** Coordinate-convention test with a **deliberately asymmetric** map (e.g. a wall
      along one edge only). A symmetric test map will pass even when x/y are swapped — this is
      the classic way transposition bugs survive a test suite.
      (`test_asymmetric_map_preserves_orientation` in the backend suite, plus
      `test_asymmetric_scan_is_not_mirrored` in `tools/tests/test_map_publisher.py`.)
- [ ] **5.4** Aggregation test: known source map → known expected 20×20 output.
- [x] **5.5** Drill-down test: cell `(x, y)` returns the correct source sub-block and bounds.
      (`test_cell_detail_bounds_and_counts`, `test_cell_detail_missing_before_map_or_out_of_bounds`,
      `test_dashboard_region_bounds_matches_aggregation` in `test_grid_pipeline.py`.)
- [ ] **5.6** Frontend offline test: backend down → dashboard still renders, shows OFFLINE,
      does not crash or spin forever. This is the existing "clean slate" behaviour and it must
      not regress.
- [ ] **5.7** Reconnection test: kill the backend mid-session, restart, confirm the dashboard
      recovers without a page reload.
- [ ] **5.8** Radio-link tests (post-Phase 3): corrupted frames, partial frames, total link
      loss and recovery.
- [ ] **5.9** End-to-end walkthrough: walk the Pi around the real ~15×15 m area and confirm the
      dashboard outline matches the physical room. **This is the only test that actually
      validates the system.** Everything above just catches regressions.
- [ ] **5.10** Resolve the **duplicate Theta\* implementations** (`backend/services/
      theta_service.py` and `src/algorithms/thetaAlgorithm.js`). They run at different
      resolutions and will produce different paths from the same input. Pick one as
      authoritative and either delete the other or add a test asserting they agree.
- [ ] **5.11** **Storage round-trip:** write a map snapshot, read it back, assert the occupancy
      array is byte-identical after compress/decompress. A silent codec bug here corrupts every
      recording made before it is caught.
- [ ] **5.12** **Concurrent access:** serial reader writing while the API reads, sustained for
      several minutes. Confirm no `database is locked` errors and no blocked reads (this is what
      WAL is for — if it fails, WAL is not actually enabled on every connection).
- [ ] **5.13** **Crash recovery:** kill the recorder mid-write, restart, confirm the DB opens
      cleanly and the last video segment is still playable up to the cut point.
- [ ] **5.14** **Timestamp alignment:** with a known event visible on camera, confirm the
      matching map snapshot and video frame resolve to the same moment (§3.5.9, §4.9).

---

## 6. bugs.md

- [x] **6.1** `bugs.md` created alongside this file.
- [ ] **6.2** Log every bug at the moment it is found, before fixing it. A bug fixed but never
      written down teaches nobody anything and will recur.
- [ ] **6.3** Pre-seeded known issues are already in the file — triage them.

---

## 7. Keeping the work efficient

Practices that reduce wasted effort and repeated context, without cutting quality:

**Work against fixtures, not hardware.** Item 1.5 exists for this reason. Iterating UI against
a captured replay is dramatically faster than re-flying to test a CSS change, and it makes
failures reproducible.

**Fix the data contract once (1.1).** Nearly all expensive rework in a project shaped like this
comes from a schema that shifted after three components already consumed it.

**One source of truth per rule.** Classification thresholds in `grid_service.py` only.
Aggregation logic in the backend only. Duplicated logic (see: two Theta\* implementations) is
the thing that silently rots.

**Make Phase 3 invisible to the frontend (3.3).** If the transport swap forces frontend changes,
Phase 2 gets rebuilt. The abstraction boundary is where the savings are.

**Try the cheap path first.** 4.2 before 4.4. A browser `getUserMedia` call may replace an
entire backend video subsystem.

**Small diffs, tested, then move on.** Long unreviewed change sets are harder to debug than the
sum of their parts, and a failure mid-way costs the whole batch.

**Keep this file and `bugs.md` current.** They are the shared context — cheaper to read than to
reconstruct the project state from the code every session.

---

## Open questions to resolve

1. **"YRRC" receiver** — exact hardware and protocol? Blocks Phase 3. (§3.1)
2. **Analog vs digital video** — ROTG02 analog link and the Pi digital stream are different
   architectures. Which is primary? Blocks Phase 4. (§4.0)
3. **Map replace vs accumulate** — affects the ingestion contract from Phase 1 onward. (§1.3)
4. **Authoritative Theta\*** — Python or JS? (§5.10)
5. **OUTSIDE classification** — currently no cell is ever classified OUTSIDE and the search-region
   mask is unimplemented. Ship the mask, or remove the dead branch before it misleads someone.