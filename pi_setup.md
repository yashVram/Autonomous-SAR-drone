# Pi Setup — DroneSync / UDAN KHATOLA

Setup for the Raspberry Pi side of the map ingestion pipeline. Referenced from `todo.md` §1.

Target: **Raspberry Pi 5** (main compute — LiDAR + publisher).
Secondary: **Raspberry Pi 4** (camera streaming, headless).

Unless a section says otherwise, commands here run **🍓 on the Pi** — its own terminal, via
monitor or an SSH session. Sections that mix machines are labelled per block:
💻 **PC** = Git Bash on the Windows ground station, 🍓 **Pi** = the Pi's shell. If the prompt says
`pi@raspberrypi`, you're on the Pi. The PC-side SSH setup itself lives in **`ssh_transport.md`**.

---

## 0. Before you start

Two things cause most of the lost time on this setup, so handle them first:

**Network.** Test on a **personal hotspot, home router, or direct Ethernet**. Campus and
institutional Wi-Fi commonly enforce client isolation, which silently drops device-to-device
traffic regardless of firewall configuration. You will see packets leaving the Pi and nothing
arriving at the laptop, with no error anywhere.

**UART contention.** The YDLIDAR and the ELRS receiver both want a UART. The Pi's primary UART
is single-occupancy. Decide now (see §4) — retrofitting this after the publisher works is
irritating.

---

## 1. Base OS

Raspberry Pi OS (Trixie or later), 64-bit.

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y git cmake build-essential python3-dev python3-pip python3-venv
```

Set a static IP or a DHCP reservation for the Pi. The publisher and any debugging session both
assume the address doesn't move between reboots.

---

## 2. Serial ports

Enable the UART and free it from the serial console:

```bash
sudo raspi-config
# Interface Options → Serial Port
#   Login shell over serial? → NO
#   Serial port hardware enabled? → YES
sudo reboot
```

Confirm what you have:

```bash
ls -l /dev/serial* /dev/ttyAMA* /dev/ttyUSB*
```

Add the user to `dialout` so the publisher doesn't need root:

```bash
sudo usermod -aG dialout $USER
# log out and back in for this to take effect
```

---

## 3. YDLIDAR SDK

Build from source:

```bash
cd ~
git clone https://github.com/YDLIDAR/YDLidar-SDK.git
cd YDLidar-SDK
mkdir build && cd build
cmake ..
make -j4
sudo make install
```

Python bindings:

```bash
cd ~/YDLidar-SDK
pip install . --break-system-packages
```

udev rules (creates the stable `/dev/ydlidar` symlink):

```bash
cd ~/YDLidar-SDK/startup
sudo chmod +x initenv.sh
sudo ./initenv.sh
```

Replug the LiDAR, then verify:

```bash
ls -l /dev/ydlidar
```

### Critical settings

- **Baud rate: `115200`.** Not 128000 — that value appears in some YDLIDAR docs for other units
  and will produce either silence or garbage for this one.
- Port: use `/dev/ydlidar` (the udev symlink), never a raw `/dev/ttyUSB0` — the number changes
  between boots.

Smoke test with the SDK's bundled sample before writing any project code. If the sample doesn't
stream, nothing downstream will.

---

## 4. UART conflict — LiDAR + ELRS

Both devices need a serial port and there's one primary UART. Pick one:

**Option A — USB-to-UART adapter (recommended for the MVP).**
Put one device (usually the ELRS receiver) on a USB serial adapter. No config file edits, no
boot-time surprises, trivially reversible. Costs a USB port and a few grams.

**Option B — enable a secondary UART in `/boot/firmware/config.txt`.**

```
enable_uart=1
dtoverlay=uart2
```

Reboot, then confirm the new device node appears. Cleaner physically, but overlay names and
pin mappings differ across Pi models — verify against Pi 5 documentation specifically, not a
Pi 4 tutorial.

Whichever you choose, **write it down in this file** with the resulting device paths, because
the publisher config and the Phase 3 serial reader both depend on it.

```
LiDAR:  /dev/ydlidar      @ 115200
ELRS:   /dev/__________   @ ______      # fill in
```

---

## 5. Python environment

```bash
cd ~
python3 -m venv --system-site-packages dronesync-env
source dronesync-env/bin/activate
pip install requests pyserial numpy
```

`--system-site-packages` matters — it lets the venv see the system-installed YDLidar bindings.

Add to `~/.bashrc` if you want it active by default:

```bash
source ~/dronesync-env/bin/activate
```

---

## 6. Map publisher

> **Transport changed (2026-09-05).** The Pi no longer POSTs to the laptop. The laptop now pulls
> over SSH — see **`ssh_transport.md`**, which is the authoritative setup guide for the link.
> This section covers only the Pi-side deployment; do the SSH/key setup from the PC side.

Deploy `tools/map_publisher.py` (see `todo.md` §1.4) to the Pi.

### 💻 On the PC — Git Bash, from the repo root

These are typed on the PC and copy files *onto* the Pi:

```bash
ssh pi "mkdir -p ~/tools"
scp tools/map_publisher.py pi:~/tools/
```

The publisher defaults to `--sink stdout`, which is what the SSH transport reads — the PC runs
`ssh pi "python3 -u ~/tools/map_publisher.py --sink stdout"` and each line of stdout is one map
payload. **Nothing else may be printed to stdout**; all logging goes to stderr, or the stream is
corrupted. `python3 -u` is required on the remote command — without it Python buffers stdout and
the payloads arrive in stalled 8KB bursts.

### 🍓 On the Pi — only if you want persistent defaults

Tuning is via env vars (so the systemd unit in §8 doesn't need editing) or CLI flags. Set these
on the Pi, in `~/.bashrc` or the systemd unit — **not** on the PC, where they'd have no effect:

```bash
export DRONESYNC_LIDAR_PORT="/dev/ydlidar"
export DRONESYNC_LIDAR_BAUD="115200"
export DRONESYNC_PUBLISH_HZ="2"
export DRONESYNC_GRID_SIZE="200"
export DRONESYNC_RESOLUTION="0.05"
# DRONESYNC_BACKEND is only needed for the legacy --sink http mode
```

In stream mode it's usually simpler to skip the env vars entirely and pass flags in the PC-side
`--remote-command` instead (`ssh_transport.md` §6, stage B).

Behaviour requirements, restated from the todo because they're easy to skip:

- Rate-limit to **~1–2 Hz**. The dashboard can't render faster and full-grid payloads at scan
  rate will saturate the link.
- A failed send must never kill the scan loop.
- Match the payload schema in `todo.md` §1.1 exactly, including `occupancy[y][x]` row-major
  ordering. Transposition here surfaces as a mirrored dashboard, which is easy to misdiagnose
  as a frontend bug.

Run it in the foreground first and watch the output. In stream mode you do **not** need to
daemonise it at all — SSH starts it on demand (§8 applies only to the file-sink/pull mode).

---

## 7. Verifying the link

Full troubleshooting table lives in `ssh_transport.md` §7.

### 💻 On the PC — all four, each ruling out one layer

```bash
ping raspberrypi.local                                  # 1. network reaches the Pi
ssh -o BatchMode=yes pi echo ok                         # 2. ssh + key auth (no password prompt)
ssh pi "python3 ~/tools/map_publisher.py --help"        # 3. publisher deployed and runnable
curl http://127.0.0.1:8000/                             # 4. local backend alive
```

If ping works but ssh doesn't, check `sudo systemctl status ssh` on the Pi. If ping itself fails,
you're on a network with client isolation — switch to a hotspot, home router, or direct Ethernet
(§0). SSH does not work around client isolation; nothing does.

The old failure mode — packets leaving the Pi and vanishing into the laptop's firewall — no
longer applies, because the PC now initiates the connection outbound and the final hop into the
backend is loopback.

---

## 8. Run on boot — file-sink mode only

> **Not needed for the default SSH stream transport.** There, the SSH command starts the
> publisher on demand and it exits with the connection; a systemd unit would fight it for the
> LiDAR's serial port. Use this **only** for the file-sink/pull arrangement
> (`ssh_transport.md` §8), where the publisher must run independently of any SSH session.

```bash
sudo nano /etc/systemd/system/dronesync-publisher.service
```

```ini
[Unit]
Description=DroneSync map publisher (file sink)
After=network-online.target

[Service]
Type=simple
User=pi
Environment="DRONESYNC_SINK=file"
Environment="DRONESYNC_MAP_FILE=/home/pi/dronesync/latest_map.json"
Environment="DRONESYNC_PUBLISH_HZ=2"
ExecStart=/home/pi/dronesync-env/bin/python /home/pi/tools/map_publisher.py
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now dronesync-publisher
journalctl -u dronesync-publisher -f
```

---

## 9. Headless access (Pi 4 / camera Pi)

Currently unresolved — see `todo.md`. State as of last session:

- TigerVNC is the client; NoMachine removed.
- `wayvnc` conflict resolved; `raspi-config` set to **X11 mode + desktop autologin**.
- TigerVNC still reporting connection refused.

Next things to check:

```bash
# Is anything actually listening on the VNC port?
sudo ss -tlnp | grep 590

# Firewall
sudo ufw status
```

In the TigerVNC viewer, use `<pi-ip>:1` (display number) or `<pi-ip>::5901` (explicit port) —
the single-vs-double colon distinction is a common cause of "connection refused" when the server
is in fact running.

SSH is sufficient for everything in §1–§8 and doesn't depend on this being fixed.

---

## 10. RF note

The Pi 5's onboard 2.4 GHz Wi-Fi/Bluetooth can interfere with a 2.4 GHz ELRS control link. If
both are active on the airframe, range-test with Wi-Fi enabled and disabled before trusting the
link. Mitigations: physical separation of the antennas, or a 915/868 MHz ELRS variant.

This is a Phase 3 concern, but worth measuring early — it can influence the transport decision
in `todo.md` §3.2.

---

## Checklist

- [ ] OS updated, static IP or DHCP reservation set
- [ ] **SSH enabled on the Pi and reachable from the PC** (`ssh_transport.md` §1–2)
- [ ] **Passwordless key auth working** — `ssh -o BatchMode=yes pi echo ok` prints `ok`
      (`ssh_transport.md` §3; the bridge cannot prompt for a password)
- [ ] UART enabled, serial console disabled, user in `dialout`
- [ ] YDLidar SDK built and installed; `/dev/ydlidar` symlink present
- [ ] SDK sample streams at 115200
- [ ] UART conflict resolved and device paths recorded in §4
- [ ] Python venv created with `--system-site-packages`
- [ ] `map_publisher.py` deployed to `~/tools/` and `--help` runs on the Pi
- [ ] Stage A passed: bridge + replay fixture puts a grid on the dashboard
      (`ssh_transport.md` §6)
- [ ] Stage B passed: same, with the real LiDAR
- [ ] Link verified through all four steps in §7
- [ ] One real scan captured to `backend/tests/fixtures/map_sample.json` (`todo.md` §1.5)
- [ ] If the Pi will record locally (Phase 3+): USB SSD mounted for the SQLite DB and video
      segments — not the microSD card (`todo.md` §3.5.8)
- [ ] NTP or a recorded Pi↔laptop clock offset in place (`todo.md` §3.5.9)