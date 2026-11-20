# Guide 03: First SITL flight (+ QGroundControl on the Mac)

How I flew ArduCopter in SITL for the first time: takeoff, Loiter, landing, all from the terminal. Then how to watch it from QGroundControl on the Mac. Includes the "command not found" problem and its fix.

Machine: Ubuntu 26.04 arm64 VM (UTM on Apple Silicon). Follows [guide 02](02-ardupilot-sitl-build.md).

---

## What runs when SITL starts

```
 arducopter (the real flight code)  ◀──▶  built-in physics model
   (black "ArduCopter" window)            (simulated quad + GPS, IMU, baro, compass)
         │ MAVLink
         ▼
 MAVProxy (text ground station, in my terminal)  ──--out──▶  QGroundControl on the Mac
```

`sim_vehicle.py` is a helper script that starts all of these and connects them. MAVProxy talks to the "flight controller" over **MAVLink**, the same protocol a telemetry radio and Mission Planner use.

---

## ⚠️ Problem first: `sim_vehicle.py: command not found`

### Why it happens
- `sim_vehicle.py` lives in `~/ardupilot/Tools/autotest/`. Linux only finds a command by name if its folder is on the **PATH**, the list of folders it searches.
- Linux **doesn't search the current folder**, even if the file is right there. To run something in the current folder you write `./sim_vehicle.py`. This is a safety rule, so a stray file can't impersonate a system command.
- The prereqs script added the PATH line and the venv activation to **`~/.profile`**, but on this Ubuntu, terminal windows didn't pick those up, even after logging out. The prompt never showed `(venv-ardupilot)`.
- `venv-ardupilot` is **a folder** (`~/venv-ardupilot`), not a command. Typing its name gives "command not found". It's switched on with `source ~/venv-ardupilot/bin/activate`.

### Fix: put the settings in `~/.bashrc`
`~/.bashrc` is read by **every** terminal window you open. Run this once:
```bash
cat >> ~/.bashrc <<'EOF'

# ArduPilot: tools on PATH + Python venv in every terminal
export PATH="$HOME/ardupilot/Tools/autotest:$PATH"
source "$HOME/venv-ardupilot/bin/activate"
EOF
source ~/.bashrc
```
- `cat >> file <<'EOF' … EOF` **adds** the lines to the end of the file. `>>` appends; a single `>` would overwrite the whole file.
- `source ~/.bashrc` applies it to the current terminal straight away.

Check: the prompt now starts with `(venv-ardupilot)`, and
```bash
which sim_vehicle.py     # → /home/<user>/ardupilot/Tools/autotest/sim_vehicle.py
```

---

## Part 1: Fly from the terminal

### 1. Start SITL
```bash
cd ~/ardupilot
sim_vehicle.py -v ArduCopter -w
```
- `-v ArduCopter`: the vehicle type.
- `-w`: **wipe** parameters to defaults. Use it only the first time; afterwards leave it out so parameter changes are kept, like on a real FC.

Two windows appear:
- **Black "ArduCopter" window**: the flight controller itself (the `arducopter` program built in guide 02). Leave it open; closing it is like unplugging the FC.
- **Terminal**: MAVProxy, the ground station. Commands go here.

### 2. Reading the boot (same sequence as a real FC)

| Message | Meaning |
|---|---|
| `Init Gyro`, `Barometer calibration complete` | Sensors initialised and zeroed |
| `EKF3 IMU0/IMU1 initialised`, `tilt alignment complete` | Both EKF instances running; attitude known |
| `MAG0 initial yaw alignment complete` | Heading known from the compass |
| `GPS 1: detected u-blox` | Simulated GPS found |
| `EKF3 IMU0 origin set`, `EKF3 IMU0 is using GPS` | Position known. **Ready to fly.** |
| `Failed to download /SRTM3/…` | **Harmless**: terrain-height data couldn't be downloaded; only needed for terrain following |

The prompt may show `MAV>` until MAVProxy catches up. Press Enter and it becomes `STABILIZE>`. Commands work at either prompt.

### 3. Fly
```
mode guided
arm throttle
takeoff 10
```
| Command | What it does |
|---|---|
| `mode guided` | GUIDED: the FC follows commands from a ground station or companion computer. The flagship VIO project will use this mode. |
| `arm throttle` | Arms the motors. If refused, a `PreArm:` line says why; usually wait 20 s more for GPS/EKF. |
| `takeoff 10` | Climb to 10 m |

Then switch to Loiter and fly with the virtual sticks:
```
rc 3 1500
mode loiter
rc 2 1400
rc 2 1500
mode land
```
| Command | What it does |
|---|---|
| `rc 3 1500` | **Centre the throttle stick before Loiter.** SITL's simulated transmitter starts with throttle at 1000 (bottom). In Loiter the throttle controls climb/descent, so a bottom stick would descend into the ground. |
| `mode loiter` | Position hold, the same mode I tuned on the Micro and Nano, but on simulated GPS instead of optical flow |
| `rc 2 1400` | Pitch stick forward: flies forward |
| `rc 2 1500` | Stick centred: stops and holds position |
| `mode land` | Descends, lands, disarms by itself |

RC channels: **1 = roll, 2 = pitch, 3 = throttle, 4 = yaw**. Values are PWM, 1000–2000, with 1500 centre.

### 4. What a good flight looks like
```
Got COMMAND_ACK: NAV_TAKEOFF: ACCEPTED
AP: EKF3 IMU0 MAG0 in-flight yaw alignment complete     ← heading refined once moving, as on a real quad
Got COMMAND_ACK: DO_SET_MODE: ACCEPTED
LOITER> Mode LOITER
LAND> Mode LAND
AP: SIM Hit ground at 0.501246 m/s                      ← gentle touchdown
AP: Disarming motors
DISARMED
```
Two messages printed at the same moment can run together on one line (`mode landFlight battery 80 percent`). That's only display, not an error.

### 5. Stop SITL
**Ctrl+C** in the MAVProxy terminal. (`exit` is not a MAVProxy command: "Unknown command".)

### 6. The flight log
Every SITL flight writes a real dataflash log:
```bash
ls -lt ~/ardupilot/logs/
```
My first flight: `00000001.BIN` (10.4 MB). `LASTLOG.TXT` holds the number of the newest log. It's the same `.BIN` format as real flights, so the same tools work on it (Mission Planner, my `log_report.py`). Later, when I change ArduPilot code, these logs are how I prove what the change did.

---

## Part 2: QGroundControl on the Mac

MAVProxy can send a copy of the telemetry to another ground station.

### 1. Find the Mac's address as the VM sees it
```bash
ip route | grep default
# default via 192.168.64.1 dev enp0s1 proto dhcp src 192.168.64.2 metric 100
```
- `via 192.168.64.1` = **the Mac** (the VM's gateway, UTM's shared network).
- `src 192.168.64.2` = the VM itself.

### 2. Install and open QGroundControl on the Mac
From qgroundcontrol.com (macOS version). It listens on UDP port 14550 automatically.

### 3. Start SITL with an extra output
```bash
cd ~/ardupilot
sim_vehicle.py -v ArduCopter --out=udp:192.168.64.1:14550
```
`--out=udp:<address>:<port>` = "also send the telemetry stream there". No `-w`, so parameters are kept.

### ⚠️ Pitfall: angle brackets
In instructions, `<that address>` is a **placeholder**: replace the whole thing, brackets included. Typing `--out=udp:<192.168.64.1>:14550` fails with `bash: 192.168.64.1: No such file or directory`, because in the shell `<` means "read input from a file".

### If QGC doesn't connect
- macOS firewall: allow QGroundControl when asked, or check System Settings → Network → Firewall.
- Check that the address after `via` still matches. It can change if UTM's network settings change.

Status 2026-11-20: QGC installed, SITL started with `--out`; connection still to be confirmed.

---

## Troubleshooting summary

| Symptom | Cause | Fix |
|---|---|---|
| `sim_vehicle.py: command not found`, no `(venv-ardupilot)` in prompt | Tools folder not on PATH, venv not active in terminal windows | Add both lines to `~/.bashrc` (above) |
| `venv-ardupilot: command not found` | It's a folder, not a command | `source ~/venv-ardupilot/bin/activate` |
| `PreArm: …` when arming | GPS/EKF not settled yet, or a real pre-arm check | Wait 20–30 s; read the message |
| Quad sinks after `mode loiter` | Throttle stick at bottom (1000) | `rc 3 1500` before switching |
| `Unknown command 'exit'` | MAVProxy has no exit command | Ctrl+C |
| `bash: <address>: No such file or directory` | Typed placeholder brackets | Remove `<` and `>` |

---

## Next: guide 04
Run SITL under **gdb** and stop it inside `AP_OpticalFlow`, the first look inside the running flight code.
