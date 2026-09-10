# BUGS — DroneSync / UDAN KHATOLA

Log bugs **when found**, before fixing. Cross-reference the `todo.md` item where relevant.

**Severity:** `S1` blocks the mission/demo · `S2` major, workaround exists · `S3` minor ·
`S4` cosmetic
**Status:** `OPEN` · `INVESTIGATING` · `FIXED` · `WONTFIX` · `CANNOT REPRODUCE`

---

## Entry template

```
### BUG-000 — <one-line summary>
- Severity: S?
- Status: OPEN
- Found: YYYY-MM-DD · <who>
- Component: frontend / backend / pi / radio / video
- Related: todo §x.x

**Steps to reproduce**
1.
2.

**Expected**

**Actual**

**Notes / logs**

**Fix**
```

---

## Known issues (pre-seeded — triage these)

### BUG-001 — Two Theta* implementations can disagree
- Severity: S2
- Status: OPEN
- Component: backend + frontend
- Related: todo §5.10

`backend/services/theta_service.py` plans against the full source occupancy map;
`src/algorithms/thetaAlgorithm.js` plans client-side against the 20×20 aggregate (note: as of
2026-09-05 this JS function isn't actually called from anywhere in the frontend, so this
particular disagreement is currently latent, not live — still worth resolving before it's wired
up). Different
resolutions mean the same scenario can yield different paths, and the aggregate view marks a
whole region WALL if any source cell is occupied — so the JS planner will refuse routes the
Python planner accepts.

**Impact:** dashboard may display a path the drone will not fly, or show no route where one
exists.

**Fix:** pick one as authoritative. If both must exist, add a test asserting agreement on a
shared fixture.

---

### BUG-002 — OUTSIDE classification is unreachable dead code
- Severity: S3
- Status: OPEN
- Component: backend
- Related: todo §7 open questions

`grid_service.py` defines an OUTSIDE state reserved for a search-region mask that is not
implemented. No cell is ever classified OUTSIDE.

**Impact:** misleading to anyone reading the classifier; any frontend styling for OUTSIDE is
untested and unreachable.

**Fix:** implement the mask, or remove the branch until it is needed.

---

### BUG-003 — Sensor readings panel is entirely static placeholder text
- Severity: S3
- Status: OPEN
- Component: frontend

Temperature, thermal target, LiDAR distance, gas level, humidity and GPS accuracy render fixed
`--` / `NO DATA` strings. There are no backing fields anywhere in the telemetry schema, so the
panel cannot currently show data even if a sensor were connected.

**Impact:** reads as "sensors offline" when it is actually "sensors were never wired".

**Fix:** either add the fields to the telemetry contract, or mark the panel visibly as
not-implemented.

---

### BUG-004 — `isSurvivorSubCell` hardcoded false
- Severity: S3
- Status: OPEN
- Component: frontend
- Related: todo §2.9

Per-cell survivor overlay always evaluates false, so the overlay can never appear.

**Impact:** a survivor detection would not be visible in the grid drill-down.

**Fix:** route through a single detection function so real data can be attached without
touching render code.

---

### BUG-005 — Backend URL hardcoded to `http://localhost:8000`
- Severity: S2
- Status: OPEN
- Component: frontend

`src/services/droneApi.js` hardcodes the base URL. The dashboard cannot reach a backend running
on the Pi or on another machine without a code edit and rebuild.

**Impact:** blocks any deployment where the ground station and backend are not the same host.

**Fix:** move to a Vite env var with `localhost:8000` as the default.

---

### BUG-006 — In-memory backend state unsafe once a serial reader is added
- Severity: S2
- Status: INVESTIGATING (partially fixed 2026-09-05)
- Component: backend
- Related: todo §3.5

The backend README already flags in-memory state as unsafe for multi-process deployment. Phase 3
introduces exactly that: a serial ingestion process alongside the API server.

**Impact:** map data written by the serial reader may be invisible to the API server, or lost.

**Fix so far:** map snapshots and telemetry now durably record to a SQLite DB
(`backend/services/db_service.py`, WAL mode) alongside the existing in-memory live state, so the
durable half of this concern (data being silently lost) is addressed. **Not yet fixed:** the
actual multi-process topology this bug describes doesn't exist yet — there is still only one
process (this API server). When §3.3's serial reader becomes a real separate OS process, it must
either become the sole DB writer (with this process reading only) or route writes through this
same process; the in-process `threading.Lock` added now does not protect against two separate
OS processes writing the same SQLite file. Re-open / re-verify once that process exists.

---

### BUG-007 — Thermal demo assets present but unreferenced
- Severity: S4
- Status: OPEN
- Component: frontend
- Related: todo §4.7

~60 images in `public/demo/thermal/` are not referenced by any camera or detection component.

**Impact:** ships dead weight and implies a demo mode that does not exist.

**Fix:** wire as an explicit demo toggle, or delete.

---

## Active bugs

### BUG-016 — LiDAR is enclosed: 99% of every scan is UNKNOWN
- Severity: S2 (transport works; the map is unusable)
- Status: OPEN — physical setup, not code
- Found: 2026-09-05 · Claude, during live hardware bring-up
- Component: pi / physical

With the link fully working, the produced map is ~92-99% UNKNOWN. Measured from a live payload:

```
source 200x200 @ 0.05m = 40000 cells
  free    =    235 ( 0.6%)
  wall    =     41 ( 0.1%)
  unknown =  39724 (99.3%)
41 wall cells; distance from sensor: min=0.11m median=0.40m max=2.86m
wall returns closer than 0.5 m: 26/41
```

Returns arrive from all bearings (angles span the full ±3.14 rad) but almost all terminate within
half a metre. The sensor is enclosed or sitting among clutter, so every ray stops immediately and
nothing further out is ever observed.

**Impact:** the dashboard grid shows a few wall blobs and no room outline — indistinguishable at a
glance from "the link is broken", which is how this was first reported.

**Fix:** put the LiDAR in clear space — open floor, ideally the middle of the room, nothing within
~1 m of the scan plane. Re-run and expect a recognisable outline.

**Second, independent contributor:** the unit returns only ~252 points per revolution, so rays are
sparse spokes with unknown gaps between them that widen with distance. Single-scan rasterization
cannot fill those; only accumulation across scans (pose tracking / SLAM) can, which todo.md §1
explicitly puts out of Phase 1 scope. Expect a spoke-like map even in open space.

---

### BUG-015 — Grid too small for --max-range; distant returns silently discarded
- Severity: S2
- Status: FIXED 2026-09-05
- Found: 2026-09-05 · Claude, during live hardware bring-up
- Component: pi

The grid spans `grid_size * resolution` metres in total, so it reaches only **half** that from the
centred sensor. The defaults (200 cells x 0.05 m) reach 5.0 m, while `--max-range` defaulted to
8.0 m. Returns between 5 m and 8 m were rasterized toward a target outside the array:
`points_to_occupancy` traced the ray to the boundary but never marked the hit, so the wall
vanished with no error anywhere.

**Fix:** `check_grid_extent()` now warns at startup with both numbers and the two ways out:

```
WARNING: --max-range 8.0m exceeds the grid's 5.00m reach (200 cells x 0.05m = 10.0m across).
Returns beyond 5.00m are being discarded. Either lower --max-range to 5.00, or raise
--grid-size to 320 to cover the full range.
```

Verified firing on the Pi. Covered by `GridExtentTests`.

---

### BUG-014 — YDLidar SDK writes to stdout, corrupting the NDJSON payload stream
- Severity: S1 (silent data loss on the live link)
- Status: FIXED 2026-09-05
- Found: 2026-09-05 · Claude, during live hardware bring-up
- Component: pi · Related: `ssh_transport.md`

The SSH transport reserves the publisher's stdout for one JSON payload per line. The SDK's C++
layer writes its own logs (`[2026-09-05][info] SDK initializing`, checksum warnings, ANSI colour
codes) straight to **file descriptor 1**, landing mid-stream. First live run: **24 of 78 payloads
lost** (`received=78 accepted=54 rejected=24`).

Both observed parse errors reproduce exactly from SDK log lines — `[2026-...` parses as a JSON
array then fails at column 6; a line starting with the ANSI escape fails at column 0.

Python-level discipline (`file=sys.stderr`) cannot prevent this: the writes never pass through
Python.

**Fix, two layers:**
1. `claim_stdout_for_payloads()` in the publisher dups fd 1 to a private stream for payloads, then
   points fd 1 at stderr — the SDK (or any C library) is now physically unable to reach the
   payload channel. Called before the SDK is imported.
2. The bridge classifies any line not starting with `{` as remote *noise* (SSH motd, shell
   warnings) rather than a rejected payload, so link-health stats stay meaningful. Tracked
   separately as `noise=`.

**Verified on hardware:** `received=60 accepted=60 rejected=0 noise=0` — down from 24 losses.

---

### BUG-013 — Pi unreachable by hostname; drops off the network mid-session
- Severity: S1 (blocks all on-device work)
- Status: RESOLVED for now — recurrence likely
- Found: 2026-09-05 · Claude
- Component: pi / network
- Related: BUG-012 · `ssh_transport.md` §0, §7

The SSH bridge was streaming from `rpi` at 2 Hz, then stopped. ~109 s later the last map payload
was still the newest one the backend had.

**Observed from the PC at that moment:**
- `ping raspberrypi` → "could not find host"; `ping raspberrypi.local` → same.
- No Raspberry Pi OUI (`b8-27-eb`, `dc-a6-32`, `e4-5f-01`, `28-cd-c1`, `d8-3a-dd`, `2c-cf-67`)
  anywhere in the ARP cache.
- PC is on Wi-Fi `172.26.130.75`. A full ping sweep of `172.26.130.0/24` populated only **4** ARP
  entries for the entire subnet.

**Reading:** 4 reachable hosts on a whole /24 is the signature of either AP client isolation or a
routed campus network where the Pi sits on a different L2 segment. Note this is the same class of
failure that BUG-012 was worked around rather than diagnosed — and SSH does not solve it, because
it blocks traffic in both directions regardless of protocol.

**Impact:** blocks deploying/verifying the BUG-009 LiDAR fix, and blocks Stage B entirely.

**2026-09-05 update:** the Pi came back on a different subnet (`10.241.187.72`, was reachable
as `raspberrypi`), and the full live bring-up succeeded from there. The address moved between
sessions, which is the recurrence risk below.

**Fix:** put both machines on a **phone hotspot** or a direct Ethernet cable and re-test. If it
works there and not on the campus network, BUG-010's root cause is confirmed as client isolation
and should be closed out as such. Give the Pi a static IP / DHCP reservation so its address stops
moving (`pi_setup.md` §1).

---

### BUG-012 — Pi→PC HTTP POST transport never delivered; root cause never confirmed
- Severity: S1 (blocked the whole Phase 1 data path)
- Status: WORKED AROUND — root cause still undiagnosed
- Found: 2026-09-05 · user
- Component: pi / backend / network
- Related: todo §1.4, §1.6, §3.8 · `ssh_transport.md`

The Phase 1 design had the Pi POST map payloads to `http://<laptop-ip>:8000/api/map`. In practice
"sending and receiving messages" did not work as intended. Reported by the user; the specific
failing layer was **not isolated** before the decision to change transports.

**Impact:** no map data reached the dashboard over the intended path.

**Workaround (not a diagnosis):** transport inverted — the PC now initiates an outbound SSH
connection and pulls payloads from the Pi (`tools/ssh_map_bridge.py`), delivering them to the
backend over loopback. This structurally eliminates the three most likely causes:
inbound Windows Firewall blocking, uvicorn bound to `127.0.0.1`, and the Pi needing to track the
laptop's changing IP.

**Why this still matters:** if the real cause was **AP client isolation**, SSH will fail exactly
the same way, because that blocks device-to-device traffic at the access point regardless of
direction or protocol. The symptom to watch for is `ssh` itself timing out while `ping` also
fails. If that happens, the answer is a phone hotspot / home router / direct Ethernet — not
another transport change. Worth 10 minutes to confirm the original cause rather than carrying an
unknown forward.

---

### BUG-009 — `tools/map_publisher.py` YDLIDAR bindings were wrong in four places
- Severity: S2 (blocks todo §1.4/§1.6 "done & tested")
- Status: FIXED — **confirmed on real hardware 2026-09-05**
- Found: 2026-09-05 · Claude (working todo.md §1)
- Fixed: 2026-09-05 · Claude, by diffing against the SDK's own reference examples
- Component: pi
- Related: todo §1.4 · `ssh_transport.md` §6

Originally logged as "unverified". Verified on 2026-09-05 against the YDLidar-SDK's published
`python/examples/tri_test.py` (the triangle-lidar reference, matching this rig's 115200 baud) and
`plot_tof_test.py`. The bindings were not merely unverified — **four settings were wrong**, three
of them fatal:

| Setting | Was | SDK reference | Consequence of the old value |
|---|---|---|---|
| `ydlidar.os_init()` | **never called** | called before device use | `os_isOk()` false → scan loop exits immediately; publisher appears to start, emits nothing |
| `LidarPropSingleChannel` | `False` | `True` | single-channel unit: `initialize()` succeeds, then zero points forever |
| `LidarPropSampleRate` | `9` | `3` | wrong family (9/20 belong to TOF/dual-channel units) |
| `LidarPropScanFrequency` | `8.0` | `10.0` | off-spec spin rate |

Confirmed **correct** as originally written: `CYdLidar`, `setlidaropt`, `TYPE_TRIANGLE`,
`YDLIDAR_TYPE_SERIAL`, `LaserScan`, `doProcessSimple`, `os_isOk`, `turnOff`/`disconnecting`,
plain iteration over `scan.points`, and `point.angle` / `point.range` with **angle in radians** —
which is what `points_to_occupancy` assumes.

**Also added:**
- `LidarPropMin/MaxAngle` and `Min/MaxRange` are now set explicitly, so the SDK clips at the same
  distance as the Python-side `--max-range` filter instead of streaming points that get discarded.
- `--probe` mode: prints the installed SDK's actual API surface, detected ports, the exact options
  being applied, and the shape of five live scans. Turns "unverified bindings" into a five-second
  on-device check.
- `_sdk_attr()` guards every constant lookup, so a version mismatch names the missing attribute
  and points at `--probe` instead of raising a bare `AttributeError` mid-setup.
- `--single-channel` / `--no-single-channel` and `--sample-rate` are overridable, because the
  correct values are model-dependent.
- Regression tests (`LidarOptionsTests`) pin the SDK-reference values so they can't be quietly
  reverted.

**Confirmed on hardware 2026-09-05** via `--probe` on the Pi (SDK 1.2.20, `/dev/ydlidar` ->
ttyUSB0):

```
Lidar successfully connected [/dev/ydlidar:115200]
Lidar running correctly! The health status good
Sample Rate: 3.50K                      <- matches the corrected SampleRate=3
scan 0: 252 points, 164 within range; angle -3.120..3.140 rad, range 0.13..5.47 m
scan 4: 252 points, 166 within range; angle -3.136..3.123 rad, range 0.13..5.51 m
```

All 6 `LidarProp*` constants used are present in the 21 this SDK build exposes, and
`TYPE_TRIANGLE`/`YDLIDAR_TYPE_SERIAL` resolve. Closed.

**Benign startup noise to expect:** "Fail to get baseplate device information!" and a handful of
"Checksum error" lines while the SDK auto-negotiates intensity down 16->8->0 bit. It settles and
streams normally; these are not failures.

---

## Fixed

### BUG-011 — Clicking a cell directly on the main dashboard grid opened an empty popup
- Severity: S2
- Status: FIXED
- Found: 2026-09-05 · Claude (working todo.md §2)
- Component: frontend
- Related: todo §2.6-2.10

`App.jsx`'s main dashboard `SearchGrid`'s `onCellClick` set `expandedPanel` to `'gridCell'`
(distinct from `'grid'`, which is what the panel-wide click handler sets), but the popup's
render switch only had a branch for `expandedPanel === 'grid'` guarding the cell-detail overlay.
Clicking a cell directly (not the panel background) opened the popup shell with no content
inside it.

**Impact:** the tile drill-down (the entire point of todo §2.6-2.10) was unreachable from its
most obvious entry point.

**Fix:** the cell-detail overlay condition now renders for both `expandedPanel === 'grid'` and
`'gridCell'`. Confirmed live via a headless-browser click on a random dashboard grid cell —
now shows the real drill-down content instead of an empty box.

### BUG-010 — `scanned-cell` / `drone-cell` CSS classes never matched any rule
- Severity: S3
- Status: FIXED
- Found: 2026-09-05 · Claude (working todo.md §2)
- Component: frontend

`SearchGrid.jsx` applies `scanned-cell` and `drone-cell` classes, but `SearchGrid.css` only
defined `.grid-cell.scanned` (no `-cell` suffix) and nothing at all for `.drone-cell`. Scanned
cells therefore never looked different from unscanned FREE cells, contradicting the grid legend
("FREE / SCANNED" implies a visual distinction that didn't exist), and the drone's current
position had no distinguishing highlight beyond the 🚁 emoji itself.

**Fix:** renamed the CSS selector to `.grid-cell.scanned-cell` and added a `.grid-cell.drone-cell`
rule (a green inset ring).

---

### BUG-008 — Backend requires Python 3.10+, but `python` on PATH resolves to 3.9
- Severity: S3
- Status: FIXED
- Found: 2026-09-05 · Claude (working todo.md §1)
- Component: backend

`models/schemas.py` (and other backend modules) use `X | Y` union-type syntax at runtime (e.g.
`list[StrictInt | StrictFloat]`), which raises `TypeError: unsupported operand type(s) for |`
on Python 3.9. On this dev machine, `python`/`python3` on PATH is 3.9; only `py -3.12` has a
working interpreter with the project's dependencies.

**Steps to reproduce**
1. `cd backend && python -m unittest discover -s tests` with `python` resolving to 3.9.

**Expected:** tests run.

**Actual:** `ImportError` at module import time before any test runs.

**Fix:** documented the requirement in `backend/README.md` ("Requires Python 3.10+ ... use
`py -3.12` on Windows"). Did not change the union syntax since it's used consistently throughout
and 3.10+ is cheap to require — no code change needed once the version is right.