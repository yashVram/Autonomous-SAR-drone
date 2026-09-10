# SSH Transport — Pi → PC map delivery

Replaces the Phase 1 arrangement where the Pi POSTed directly to the PC's backend
(`todo.md` §1.4/§1.6). Everything downstream of `POST /api/map` is unchanged: same frozen
payload contract, same grid aggregation, same 20×20 dashboard, same Theta* planning.

---

## Which machine am I on?

Every command block below is labelled with the machine it runs on. Getting this wrong is the
single easiest way to waste an hour here.

| Label | Machine | Where you actually type it |
|---|---|---|
| 💻 **PC** | Windows ground station | **Git Bash**, in the repo root (`clean-slate/`) |
| 🍓 **Pi** | Raspberry Pi 5 | The Pi's own shell — either a monitor+keyboard plugged into it, or inside an `ssh` session opened from the PC |

**The part that trips people up:** from §2 onward you'll often be typing into an SSH session. Your
PC's terminal window is then showing the *Pi's* shell, and everything you type runs on the Pi
until you type `exit`.

> **Rule of thumb: look at the prompt.** If it says `cloud@raspberrypi:~ $` you are on the Pi.
> If it says something like `sahup@DESKTOP MINGW64 ~/...` you are on the PC.

A third pattern appears in §5–§6: commands you type **on the PC** that carry a quoted string
executed **on the Pi**, e.g. `ssh rpi "python3 ~/tools/map_publisher.py --help"`. You are on the
PC; the text inside the quotes runs on the Pi. Those are labelled 💻 **PC**, because that's where
your fingers are.

> **This guide uses your actual login: `cloud@raspberrypi`.** Once you set up the `rpi` alias in
> §4, commands use `rpi` instead of typing that out every time — it is not a different machine or
> a different user, just a shortcut for `cloud@raspberrypi`.

---

## 0. Why this changes anything

The old path needed the Pi to open an **inbound** connection to the PC on port 8000. That is the
fragile direction, and it fails for reasons that produce no useful error on either end:

- **Windows Firewall** silently drops inbound connections to a Python process. This is the most
  common cause by far — packets leave the Pi and simply vanish.
- The Pi has to **know the PC's IP**, which changes with every network you join.
- Uvicorn bound to `127.0.0.1` is invisible to the network no matter what the firewall says.

The SSH transport inverts the direction:

```
[Pi]                                        [Windows PC]
YDLIDAR
  │
map_publisher.py --sink stdout
  │  one JSON payload per line
  │
  └──── ssh (outbound, initiated BY the PC) ────▶ ssh_map_bridge.py
                                                       │  POST to 127.0.0.1:8000
                                                       ▼
                                                  FastAPI backend → dashboard
```

- The PC connects **outbound**. No inbound firewall rule needed.
- The Pi never needs to know the PC's address.
- The last hop is **loopback**, which no firewall filters.
- One long-lived SSH connection, not a TCP handshake per scan.
- SSH compression (`-C`) is on by default; occupancy grids are long runs of `-1`/`0` and shrink a lot.

**What this does not fix:** a network that blocks device-to-device traffic outright (AP *client
isolation*, common on campus/guest Wi-Fi). If `ssh` itself can't reach the Pi, no transport will —
use a phone hotspot, a home router, or a direct Ethernet cable. See §7.

---

## 1. Enable SSH on the Pi

### 🍓 On the Pi — needs a monitor + keyboard attached to the Pi

```bash
sudo raspi-config     # Interface Options → SSH → Yes
# or directly:
sudo systemctl enable --now ssh
sudo systemctl status ssh      # should say "active (running)"
```

Then get its address — you'll need this on the PC in §2:

```bash
hostname -I          # e.g. 192.168.1.42
hostname             # e.g. raspberrypi  → usually reachable as raspberrypi or raspberrypi.local
```

### 💻 No monitor for the Pi? Enable it from the PC instead

Power the Pi off, pull its microSD card, and put it in the PC. Create an empty file named exactly
`ssh` (no extension) in the small `boot` / `bootfs` partition that Windows mounts. Eject, put the
card back, power the Pi on — Raspberry Pi OS enables SSH on boot when it sees that file.

You'll then need the Pi's IP without a screen: try `raspberrypi.local` first, or check your
router's client list.

---

## 2. First connection from the PC

### 💻 On the PC — Git Bash

Use **Git Bash**, not PowerShell: it ships the full OpenSSH suite including `ssh-copy-id`, which
the Windows built-in client does not have.

```bash
ssh cloud@raspberrypi
# or, if mDNS doesn't resolve:
ssh cloud@192.168.1.42
```

Accept the host key, enter the Pi's password.

> **You are now on the Pi.** The prompt changed to `cloud@raspberrypi:~ $`. Anything you type goes
> to the Pi. Type `exit` to come back to the PC — and do come back, because §3 runs on the PC.

If this step fails, stop here; nothing else will work until it succeeds. Jump to §7.

---

## 3. Passwordless key authentication (required, not optional)

The bridge runs `ssh` with `BatchMode=yes`, which **refuses to prompt for a password**. That is
deliberate: an automated tool blocked on an invisible password prompt would hang forever with no
error. So key auth has to work first.

> ### ⚠️ All three commands in this section run on the **PC**.
> Not on the Pi. `ssh-copy-id` is the confusing one — you *run* it on the PC, and it *writes* the
> key onto the Pi for you. If your prompt says `cloud@raspberrypi`, type `exit` first.

### 💻 On the PC — Git Bash

You currently have no key. Create one:

```bash
ssh-keygen -t ed25519 -C "dronesync-ground-station"
# press Enter for the default path; leave the passphrase EMPTY for unattended use
```

> An empty passphrase is the right call for a field rig that must reconnect unattended. The key
> only grants access to the Pi. If you'd rather protect it, use a passphrase plus `ssh-agent` —
> but then the agent must be running before the bridge starts.

Push it to the Pi (still on the PC — this asks for the Pi's password one last time):

```bash
ssh-copy-id cloud@raspberrypi
```

Verify. This must print `ok` with **no password prompt**:

```bash
ssh -o BatchMode=yes cloud@raspberrypi echo ok
```

If that prints `ok`, the transport is ready. If it says `Permission denied (publickey)`, the key
didn't land — rerun `ssh-copy-id` and watch its output for errors.

---

## 4. Optional: a short host alias

### 💻 On the PC — create/edit `~/.ssh/config` (that's `C:\Users\sahup\.ssh\config`)

```
Host rpi
    HostName raspberrypi
    User cloud
    Compression yes
    ServerAliveInterval 5
    ServerAliveCountMax 3
```

Now `ssh rpi` works from the PC, and you can pass `--target rpi` to the bridge. **The rest of this
guide assumes you've done this.** If you skipped it, replace `rpi` with `cloud@raspberrypi`
everywhere below.

---

## 5. Deploy the publisher to the Pi

### 💻 On the PC — Git Bash, from the repo root (`clean-slate/`)

These run on the PC and push files *to* the Pi:

```bash
ssh rpi "mkdir -p ~/tools"
scp tools/map_publisher.py rpi:~/tools/
scp backend/tests/fixtures/map_sample.json rpi:~/tools/     # for the link test in §6
```

Confirm it runs over there (should print usage, not a traceback). Still typed on the PC — the
quoted part executes on the Pi:

```bash
ssh rpi "python3 ~/tools/map_publisher.py --help"
```

> **No systemd unit is needed for stream mode.** The SSH command starts the publisher on demand
> and it exits when the connection drops. The `dronesync-publisher.service` in `pi_setup.md` §8
> is only relevant to the file-sink/pull arrangement (§8 below).

---

## 6. Bring-up in two stages

Debug one thing at a time. Stage A proves the SSH pipeline with a known-good payload; stage B
swaps in the real sensor. If you skip stage A and it doesn't work, you won't know whether the
problem is the link or the LiDAR.

**Everything in this section is typed on the 💻 PC, in two separate Git Bash windows.** Nothing
here is typed on the Pi — the bridge reaches over and starts the publisher for you.

### Stage A — link only (no LiDAR)

**💻 PC — terminal 1** (repo root): start the backend, and leave it running.

```bash
cd backend
py -3.12 -m uvicorn main:app --port 8000
```

**💻 PC — terminal 2** (repo root): start the bridge. The long quoted string is the command the
bridge will run *on the Pi*:

```bash
py -3.12 tools/ssh_map_bridge.py --target rpi \
  --remote-command "python3 -u ~/tools/map_publisher.py --sink stdout --source replay --replay-file ~/tools/map_sample.json --hz 2"
```

Expected output in terminal 2:

```
[ssh_bridge] feeding http://127.0.0.1:8000 (mode=stream)
[ssh_bridge] connecting: ssh -C -o BatchMode=yes ... rpi 'python3 -u ~/tools/...'
  [pi] [map_publisher] source=replay sink=stdout rate=~2.0Hz
  [pi] [map_publisher] replaying ~/tools/map_sample.json (30x20) -- NOT live sensor data
[ssh_bridge] received=20 accepted=20 rejected=0 reconnects=0 last=0.0s ago
```

Lines prefixed `[pi]` are the **Pi's** stderr, forwarded to you — that's how you see remote
crashes without opening a second SSH session.

`accepted` climbing with `rejected=0` means the whole chain works. Then in a **💻 PC — terminal 3**,
start the dashboard with `npm run dev` and open http://localhost:5173 — the search grid should
show the fixture's room outline.

> The fixture is **synthetic placeholder data**, clearly labelled as such by the publisher. It
> proves plumbing, not mapping quality.

### Stage B — real LiDAR

First redeploy the publisher (the LiDAR bindings were fixed — BUG-009), then confirm the sensor
actually streams before involving the bridge.

**💻 On the PC** — push the current publisher and probe the device in one go:

```bash
scp tools/map_publisher.py rpi:~/tools/
ssh rpi "python3 ~/tools/map_publisher.py --probe"
```

`--probe` prints the installed SDK's real API surface, the detected ports, every option being
applied, and the shape of five live scans. What you want to see:

```
  scan 0: 812 points, 640 within range; angle -3.141..3.140 rad, range 0.12..7.84 m
```

Non-zero point counts and angles spanning roughly ±3.14 rad means the bindings are good.
**If point counts are zero**, the device is dual-channel — add `--no-single-channel` (and pass it
through in the `--remote-command` below). If a `LidarProp*` constant is reported missing, your SDK
build differs; the probe output lists what it does expose.

**💻 PC — terminal 2**: drop the replay flags. The defaults are already the live-LiDAR path:

```bash
py -3.12 tools/ssh_map_bridge.py --target rpi
```

That runs `python3 -u ~/tools/map_publisher.py --sink stdout` on the Pi. To tune it, pass a full
`--remote-command`, e.g.:

```bash
py -3.12 tools/ssh_map_bridge.py --target rpi \
  --remote-command "python3 -u ~/tools/map_publisher.py --sink stdout --hz 2 --grid-size 200 --resolution 0.05"
```

---

## 7. Troubleshooting

| Symptom | Where | Cause | Fix |
|---|---|---|---|
| `ssh: connect to host ... Connection timed out` | 💻 PC | Wrong IP, Pi off, or client isolation | `ping` the Pi from the PC. If ping works but ssh doesn't, check `sudo systemctl status ssh` on the Pi. If ping fails too, switch networks (§0). |
| `Permission denied (publickey)` | 💻 PC | Key auth not set up | Redo §3 **on the PC**. Verify with `ssh -o BatchMode=yes rpi echo ok`. |
| Bridge hangs with no output | 💻 PC | A password prompt you can't see | `BatchMode=yes` should prevent this. If you edited it out, put it back. |
| `received` climbs, `accepted` stays 0 | 💻 PC | Backend not running or wrong port | `curl http://127.0.0.1:8000/` on the PC. Start uvicorn (§6 terminal 1). |
| `backend rejected the payload (422)` | 💻 PC | Payload shape violates the contract | The message includes the reason. Contract is in `backend/README.md`. |
| `dropped a malformed line` repeatedly | 🍓 Pi | Something on the Pi prints to **stdout** | All Pi-side logging must go to stderr; stdout carries only payloads. Check edits to `map_publisher.py`. |
| Data arrives in bursts, then stalls | 💻 PC | Missing `python3 -u` in `--remote-command` | `-u` is required — without it Python buffers stdout into 8KB chunks. |
| `ssh` not found | 💻 PC | Using PowerShell without OpenSSH | Use Git Bash, or install Settings → Optional Features → OpenSSH Client. |
| Reconnect loop every few seconds | 🍓 Pi | Publisher crashing on the Pi | Read the `[pi]` lines in terminal 2 — they're the remote stderr, verbatim. |

### Isolate the layer, in order

**💻 All five run on the PC:**

```bash
ping raspberrypi                                          # 1. network reaches the Pi
ssh -o BatchMode=yes rpi echo ok                          # 2. ssh + key auth
ssh rpi "python3 ~/tools/map_publisher.py --help"         # 3. publisher deployed
ssh rpi "python3 -u ~/tools/map_publisher.py --sink stdout --source replay --replay-file ~/tools/map_sample.json --hz 1" | head -c 300   # 4. payloads flow
curl http://127.0.0.1:8000/                              # 5. backend alive
```

The first one that fails is your problem; everything after it is noise.

---

## 8. Alternative: file + pull mode

Brute-force fallback. The Pi writes the latest map to a file; the PC polls it with `ssh cat`.
Slower (one SSH handshake per poll) but trivially inspectable by hand.

**🍓 On the Pi** (via `ssh rpi`, or a monitor) — leave this running:

```bash
python3 ~/tools/map_publisher.py --sink file --map-file ~/dronesync/latest_map.json --hz 2
```

**💻 On the PC** — in its own terminal:

```bash
py -3.12 tools/ssh_map_bridge.py --target rpi --mode pull --remote-file '~/dronesync/latest_map.json' --hz 1
```

The file sink writes to a temp file and renames it, and rename is atomic on POSIX — so a reader
can never catch a half-written file, no matter when it looks. This mode is also the one worth
running under systemd on the Pi (`pi_setup.md` §8), since the publisher must survive independently
of any SSH session.

---

## 9. What this leaves open

- **Phase 3's radio link** (`todo.md` §3) is still a separate concern. SSH runs over IP, so it
  needs Wi-Fi/Ethernet. The dedicated data radio remains the answer for flights outside Wi-Fi range.
- **Link health in the dashboard** (§3.6) — the bridge reports `received/accepted/rejected/
  reconnects/last` to stderr, but nothing surfaces it in the UI yet.
- There is still only **one ingestion endpoint** (`POST /api/map`) — the bridge is a second
  possible *producer* into it, alongside a Pi still configured with `--sink http`. Running both
  at once would interleave two independent maps into the same state, since each POST replaces the
  previous one wholesale ("Replace, not merge", `backend/README.md`). Pick one producer.
  `todo.md` §3.8 tracks retiring or flag-gating the direct-POST path.
