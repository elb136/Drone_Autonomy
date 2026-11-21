# 2026-11-21: Debugging the flight code in SITL

**Phase:** 0, Dev environment · **Time:** about 1 hour

## Done
- Enabled the SITL optical-flow sensor (`FLOW_TYPE 10`) and built SITL in debug mode (`-D`)
- Stopped the running flight code at `AP_OpticalFlow::update` with gdb (`-G -B`), and read the full call stack from `main` to the flow driver
- Set up VS Code to attach its graphical debugger to running SITL (`launch.json`, ptrace permission), with breakpoints, call stack and variables in the editor

## What broke, and the fix
- **`Unknown command 'bt'`.** I typed gdb commands into MAVProxy; gdb runs in its own xterm window.
- **VS Code couldn't find the debug config.** It had opened `~ardupilot` as a new file instead of the `~/ardupilot` folder. Reopened the folder.
- **Ctrl+G intercepted by macOS.** Used the Go / Run menus instead; function breakpoints don't need a line number at all.

## Learned
- How ArduPilot runs: `main` → hardware layer (`HAL_SITL`, or `HAL_ChibiOS` on real boards) → main loop → **scheduler** running a **task table** within a 2500 µs budget (400 Hz) → each task, such as the flow update.
- Drivers are split into a common **front end** (`AP_OpticalFlow`) and sensor-specific **back ends** (SITL, DroneCAN, MAVLink). The same front end ran my real H-Flow and MTF-02 builds.
- Debug builds (`-D`) are needed to see variables. Optimised builds show `<optimized out>`.

## Next
Milestone A1: trace `AP_OpticalFlow` and `AP_RangeFinder` front end → back end in the debugger, and draw the class diagram. ([guide 04](../guides/04-sitl-debugging.md))
