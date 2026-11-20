# Guide 02: ArduPilot source + SITL build

How I downloaded ArduPilot, installed what it needs, and built **SITL**, plus the two problems I hit and how I fixed them.

Machine: Ubuntu 26.04 arm64 VM (UTM on Apple Silicon). Follows [guide 01](01-dev-environment.md).

---

## What SITL is, and why it comes first

**SITL = Software In The Loop.** It's the real ArduCopter firmware, the same code that runs on my Matek and Flywoo boards, compiled as a normal Linux program instead of for an ARM flight controller. It runs on the PC against a simulated vehicle and sensors.

That means I can fly, debug and change the firmware with **no hardware at all**. Everything later in the plan (debugging drivers, ROS 2, Gazebo, VIO) runs on top of this.

```
Real quad:   ArduCopter code ──compile for STM32──▶ .apj/.hex ──flash──▶ flight controller ── real sensors
SITL:        ArduCopter code ──compile for Linux──▶ arducopter program ──run──▶ my PC ── simulated sensors
```

---

## 0. Check the VM has enough resources

```bash
nproc        # CPU cores
free -h      # RAM (look at Mem: total / available, and whether Swap is in use)
df -h ~      # disk (look at Avail)
```

| | My VM | Needed |
|---|---|---|
| CPU cores | 4 | 2+ |
| Disk free | 46 GB | 15 GB+ |
| RAM | 3.3 GB, with 1.6 GB of swap already in use | 4 GB+ |

**Swap** is disk space Linux uses as overflow when RAM is full. It's much slower than RAM. Swap in use with only the desktop open means RAM is tight. Two ways to handle it:
- **Give the VM more RAM:** shut the VM down (Power Off, not pause), then in UTM: select the VM → Edit → System → Memory. Leave the Mac at least half of its RAM.
- **Or build with fewer parallel jobs**, `-j2` (step 3), and close VS Code and Firefox while building.

---

## 1. Download the source code

```bash
cd ~
git clone --recurse-submodules https://github.com/ArduPilot/ardupilot.git
cd ardupilot
```
- `git clone` downloads the repository and its full history into `~/ardupilot`.
- `--recurse-submodules` also downloads the **submodules**, other projects ArduPilot depends on that live in their own repos (ChibiOS, MAVLink definitions, the waf build tool itself). Without it, those folders are empty and the build fails.

This is the **official** ArduPilot repo, not my `ardupilotELB` fork. I start from the official code and will add my fork as a second remote when I get to the RCIN work (milestone A3).

---

## 2. Install ArduPilot's prerequisites

```bash
Tools/environment_install/install-prereqs-ubuntu.sh -y
. ~/.profile
```
Then **close the terminal and open a new one.**

- The script installs what ArduPilot needs beyond the guide 01 tools: Python packages (pymavlink; MAVProxy, the text ground station SITL is flown with), the ARM cross-compiler for real-board firmware, and libraries. `-y` answers yes to every question.
- It creates a **Python virtual environment** at `~/venv-ardupilot`: a separate Python setup just for ArduPilot's tools, so they can't conflict with Ubuntu's own Python. When it's active, the prompt starts with `(venv-ardupilot)`.
- `. ~/.profile` reloads shell settings, because the script added ArduPilot's tools to the **PATH** (the list of folders Linux searches for commands). A new terminal does the same thing.

### ⚠️ Problem 1: `ERROR: Failed building wheel for wxpython`

**What it is:** wxPython is a window toolkit. MAVProxy uses it only for its optional pop-up **console** and **map** windows (`--console --map`). It has nothing to do with compiling ArduPilot.

**Why it failed:** Python packages normally arrive as a **wheel**, a ready-built package that installs in seconds. wxPython has no wheel for Linux on ARM, so pip tried to compile it from source: a huge C++ build (about 8 minutes) that failed. Low RAM and the very new Python on 26.04 were probably both factors. The script stops at its first error, so later steps didn't run.

**Fix:** re-run the script and tell it to skip the graphical extras:
```bash
cd ~/ardupilot
SKIP_AP_GRAPHIC_ENV=1 Tools/environment_install/install-prereqs-ubuntu.sh -y
```
`SKIP_AP_GRAPHIC_ENV=1` before a command sets a variable for that one run; the script checks it and skips the GUI packages. Steps that already finished run quickly the second time.

**What I lose:** only MAVProxy's console and map windows. SITL can still be flown from the MAVProxy text prompt, or from QGroundControl / Mission Planner on the Mac over the network (guide 03).

### ⚠️ Problem 2: `Release file … is not valid yet (invalid for another 17h…)`

**What it is:** the VM's **clock was about 19.5 hours behind**. The VM showed Nov 19 22:54 while the Mac showed Nov 20 18:21. Pausing a UTM VM, or the Mac sleeping, freezes the VM's clock, and it doesn't always catch up.

**Why it breaks apt:** `apt` checks the date on every package list it downloads. To a VM that thinks it's still yesterday, today's lists look like they come from the future, so it rejects them. That's a security check, not a real fault.

**Fix:** make the VM sync its clock from the internet, then run the script again:
```bash
sudo timedatectl set-ntp true
sudo systemctl restart systemd-timesyncd
sleep 5; date          # should now match the Mac
```
If `date` is still wrong, restart the VM.

**Prevention:** shut the VM down (Power Off) instead of pausing it.

### Check the tools are installed
```bash
mavproxy.py --version
which sim_vehicle.py
```
Both should print something. "command not found" means the script didn't finish: re-run it and read the last lines for the error.

---

## 3. Build SITL

```bash
cd ~/ardupilot
./waf configure --board sitl
./waf copter            # or: ./waf copter -j2   if RAM is tight
```
- **`./waf configure --board sitl`** chooses what to build for. `sitl` = a Linux program on this PC. For the Pico it would be `--board FlywooF745`. Configure checks the compiler and tools, and saves the settings in `build/sitl/`.
- **`./waf copter`** builds ArduCopter: about 1,500 `.cpp` files compiled and linked into one program (the compiler, linker and ccache from guide 01 doing their jobs).
- **`-j2`** sets how many files compile at the same time. The default is one per CPU core, and each one uses memory. If the build dies with `Killed` or `c++: fatal error`, the VM ran out of RAM; use `-j2`.

Success looks like:
```
Build commands will be stored in build/sitl/compile_commands.json
'copter' finished successfully (2m25.341s)
```
The program is `build/sitl/bin/arducopter`.

### Reading the BUILD SUMMARY

```
Target          Text (B)  Data (B)  BSS (B)  Total Flash Used (B)  Free Flash (B)
bin/arducopter  4849409   217559    246352   5066968               Not Applicable
```
| Column | Meaning |
|---|---|
| **Text** | The program's machine code (about 4.8 MB) |
| **Data** | Memory for variables that have a starting value |
| **BSS** | Memory for variables that start at zero |
| **Total Flash Used** | How much flash the firmware needs |
| **Free Flash** | "Not Applicable": SITL runs on a PC, which has no flash limit |

On a real board this table matters. It's the reason the standard FlywooF745 build for the Pico left out optical flow and rangefinder support: they didn't fit in the F745's flash, so I made a custom build. When building for real boards, this shows how close I am to the limit.

### Second build: 1.5 seconds
Running `./waf copter` again with no code changes finished in **1.5 s**. waf compares each source file with what it built last time and skips anything unchanged. That's the "only rebuild what changed" job of a build system. After editing one file, only that file recompiles.

---

## 4. Make VS Code understand ArduPilot (clangd)

waf writes `compile_commands.json` **by itself** (the "Build commands will be stored in…" line above), so `bear` isn't needed. clangd looks for the file in the repo's top folder, so make a shortcut there:
```bash
cd ~/ardupilot
ln -s build/sitl/compile_commands.json .
code .
```
**Test:** open `libraries/AP_OpticalFlow/AP_OpticalFlow.cpp` and **Ctrl+click** a function name. It should jump to its definition. The first time, clangd indexes the whole codebase in the background, which takes several minutes (see the status bar). Close VS Code before big builds to save RAM.

If the symlink ever breaks (for example after deleting `build/`), re-run `./waf configure --board sitl && ./waf copter` and it's restored.

---

## Troubleshooting summary

| Symptom | Cause | Fix |
|---|---|---|
| `Failed building wheel for wxpython` | No ARM wheel; compiling from source fails | `SKIP_AP_GRAPHIC_ENV=1` when running the prereqs script |
| `Release file … is not valid yet` | VM clock behind (after a pause or sleep) | `sudo timedatectl set-ntp true` + restart `systemd-timesyncd`, or reboot |
| `Killed` / `c++: fatal error` during build | Out of RAM | `./waf copter -j2`, close apps, or give the VM more RAM |
| `mavproxy.py: command not found` | Prereqs script didn't finish, or old terminal | Re-run the script, open a new terminal |
| Red squiggles everywhere in VS Code | clangd can't find `compile_commands.json` | Symlink it into `~/ardupilot` (section 4) |

---

## Next: guide 03

First SITL flight with `sim_vehicle.py`: arm, take off, Loiter. Connect QGroundControl on the Mac to SITL in the VM. Then run SITL under gdb and stop it inside `AP_OpticalFlow`.
