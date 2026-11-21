# Guide 04: Debugging SITL (gdb, then VS Code)

How I stopped the running flight code inside the optical-flow driver and looked around. I did it first with gdb in a terminal, then with the same debugger through VS Code's graphical view, which is much easier to read.

Machine: Ubuntu 26.04 arm64 VM (UTM on Apple Silicon). Follows [guide 03](03-first-sitl-flight.md).

---

## What a debugger is, and why bother

**gdb** (the GNU Debugger) runs another program under its control, here `arducopter`, the flight code. It can:
- **Pause** the program at a chosen function or line (a **breakpoint**)
- **Show** every variable's value at that moment
- **Show** the chain of function calls that led there (the **backtrace** / **call stack**)
- **Step** forward one line at a time

It's a freeze-frame for code: the quad is frozen mid-air, and I can inspect every "wire".

**What it's for:**
- **Learning the codebase.** The fastest way to understand a feature in thousands of files is to stop inside it and see who calls it and with what data.
- **Finding causes, not guessing.** Logs say *what* happened; the debugger shows *how*. For example, stop where motor commands go out at arming (the Pico AM32 stall), or where RC input is read (my RCIN-switch feature).

---

## Step 1: Enable simulated optical flow (once)

SITL has no flow sensor by default (`FLOW_TYPE = 0`), so the flow code never runs and a breakpoint there would never trigger.
```bash
cd ~/ardupilot
sim_vehicle.py -v ArduCopter
```
In MAVProxy:
```
param set FLOW_TYPE 10
```
`10` = the SITL flow sensor. Ctrl+C to stop; the parameter is saved (like a real FC's EEPROM).

---

## Step 2: Debug build

```bash
sim_vehicle.py -v ArduCopter -D -j 2
```
**`-D` = debug build**: compiled with debug information and without optimisation, so the debugger can map machine code back to source lines and show every variable. In a normal (optimised) build the compiler reorders and removes code, and the debugger shows `<optimized out>`.

The first `-D` run is a full rebuild (several minutes); so is switching back to a normal build. `-j 2` limits parallel compiles to fit the VM's RAM.

---

## Option A: gdb in the terminal

```bash
sim_vehicle.py -v ArduCopter -D -G -B AP_OpticalFlow::update -j 2
```
| Option | Meaning |
|---|---|
| `-G` | Run the flight code inside gdb, in its own window ("ArduCopter (gdb)", an xterm) |
| `-B AP_OpticalFlow::update` | Breakpoint at the start of the flow update |

### ⚠️ Pitfall: typing in the wrong window
gdb runs in **its own window**: the red/blue "X" (xterm) icon in the dock. The purple terminal is **MAVProxy**, which answers `Unknown command 'bt'`.

While the code is paused, MAVProxy prints **`no link`** over and over. That's expected: a frozen FC sends no heartbeats.

### gdb commands
| Command | Does |
|---|---|
| `bt` | Backtrace: who called this function |
| `list` | Source around the current line |
| `p *this` | Print the whole current object |
| `info locals` | Local variables |
| `n` | Next line (doesn't enter function calls) |
| `s` | Step into the function call on this line |
| `finish` | Run until the current function returns |
| `c` | Continue until the next breakpoint |
| `delete` | Remove all breakpoints |
| `quit` | Exit gdb (and the flight code) |

Bigger font in the xterm: hold **Ctrl + right-click** → VT Fonts → Large/Huge.

---

## Reading the backtrace: how ArduPilot is organised

What `bt` printed when stopped in the flow update (bottom = where the program started, top = where it's paused):

| # | Function | Meaning |
|---|---|---|
| #7 | `main` (Copter.cpp:971) | Program start |
| #6 | `HAL_SITL::run` | **Hardware layer** for SITL. On a real board it's `HAL_ChibiOS::run`; this is the only layer that differs between SITL and hardware. |
| #5 | `AP_Vehicle::loop` | The main loop, which runs forever |
| #4 | `AP_Scheduler::loop` | One pass of the main loop |
| #3 | `AP_Scheduler::run (time_available=2500)` | Runs the task table within a **2500 µs** budget → 1 s / 2500 µs = **400 Hz**, Copter's loop rate |
| #2, #1 | `Functor…` (`Copter::scheduler_tasks+544`) | Wrapper that calls a task from the **scheduler table**; this is the flow sensor's entry |
| #0 | `AP_OpticalFlow::update` (AP_OpticalFlow.cpp:196) | **Paused here**: the same driver that ran the H-Flow (Micro/Nano) and MTF-02 (Pico) |

**Key insight:** ArduPilot isn't one big loop calling everything. It's a **scheduler with a table of tasks**, each with its own rate and time budget. The table is `scheduler_tasks` in `ArduCopter/Copter.cpp`. `this=0x… <copter+42544>` means the flow object lives inside the one big `copter` object that holds the whole vehicle.

**Driver pattern:** `AP_OpticalFlow` is the **front end**, the same for every sensor. The **back end** differs: `AP_OpticalFlow_SITL` here, DroneCAN on Micro/Nano, MAVLink on Pico. Front end/back end is ArduPilot's standard driver design (roadmap milestone A1).

---

## Option B: VS Code (same debugger, readable)

| gdb | VS Code |
|---|---|
| `b function` | Click left of a line number (red dot), or **Run → New Breakpoint → Function Breakpoint…** |
| `bt` | **Call Stack** panel; click a frame to jump to its code |
| `p var` | **Variables** panel, or hover over a variable |
| `n` / `s` / `finish` / `c` | F10 / F11 / Shift+F11 / F5, or the toolbar |
| `list` | The file opens with the current line in yellow |

### One-time setup
**1. Allow attaching to a running program** (Ubuntu blocks it by default; this lasts until the next reboot):
```bash
echo 0 | sudo tee /proc/sys/kernel/yama/ptrace_scope
```
**2. Create the debug configuration** `~/ardupilot/.vscode/launch.json`:
```bash
mkdir -p ~/ardupilot/.vscode
cat > ~/ardupilot/.vscode/launch.json <<'EOF'
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Attach to SITL",
      "type": "cppdbg",
      "request": "attach",
      "program": "${workspaceFolder}/build/sitl/bin/arducopter",
      "processId": "${command:pickProcess}",
      "MIMode": "gdb",
      "setupCommands": [
        { "text": "-enable-pretty-printing", "ignoreFailures": true }
      ]
    }
  ]
}
EOF
```
"attach" = connect to flight code that's already running, instead of starting it.

### Each session
1. Start SITL with the debug build, **without `-G`**: `sim_vehicle.py -v ArduCopter -D`
2. Open the **folder**: `code ~/ardupilot` (with the `/`; `code ~ardupilot` creates an empty file named `~ardupilot`). Trust the folder when asked.
3. **Run and Debug** icon (▷ with a bug) → **Attach to SITL** → green ▶ (or F5).
4. A list of **running programs** appears (each line has a process ID). Pick the one ending in `build/sitl/bin/arducopter`, not python or mavproxy. Enter the password if asked.
5. Breakpoint: **Run → New Breakpoint → Function Breakpoint…** → `AP_OpticalFlow::update`. Or open the file and use **Go → Go to Line/Column…** → 196, then click left of the line number.
6. Explore: **Call Stack** (= `bt`), **Variables** → expand `this` (= `p *this`), F10 to step.
7. Done: remove the breakpoint (click the red dot), then F5 to continue. MAVProxy reconnects.

### ⚠️ Pitfalls hit
| Symptom | Cause | Fix |
|---|---|---|
| Run and Debug says "open a folder and create a launch.json" | VS Code opened a file, not the folder | File → Open Folder → `ardupilot` |
| Typing `arducopter` lists files, not processes | That's the file search (Quick Open), not the debugger | Esc; start the debugger first (▶ / F5), then the process list appears |
| Ctrl+G opens something on the Mac | UTM passes some Ctrl combinations to macOS (a Mac automation grabbed it) | Use the menus: Go → Go to Line/Column…, or Run → New Breakpoint |
| `no link` spam in MAVProxy | FC paused at a breakpoint | Expected; continue (F5 / `c`) |

---

## Next
Milestone A1: use the debugger to read `AP_OpticalFlow` and `AP_RangeFinder` front end → back end, and draw the class diagram.
