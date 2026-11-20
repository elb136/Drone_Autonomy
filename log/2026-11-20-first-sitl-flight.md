# 2026-11-20: First SITL flight

**Phase:** 0, Dev environment · **Time:** about 1 hour

## Done
- Flew ArduCopter SITL from MAVProxy: GUIDED takeoff to 10 m → Loiter → forward flight on virtual sticks → position hold → LAND (touchdown 0.5 m/s, auto-disarm)
- First SITL dataflash log recorded: `00000001.BIN`, 10.4 MB, the same format as my real-flight logs
- Found the Mac's address from the VM (`192.168.64.1`) and connected QGroundControl on the Mac to SITL in the VM via MAVProxy `--out`: live map and telemetry

## What broke, and the fix
- **`sim_vehicle.py: command not found`.** The prereqs script put the PATH line and venv activation in `~/.profile`, which terminal windows on this Ubuntu didn't apply, even after logging out. Moved both into `~/.bashrc`, which every terminal reads.
- **`--out=udp:<192.168.64.1>:14550` failed.** I typed the placeholder's angle brackets; the shell read `<` as "input from file". Removed them.

## Learned
- SITL boots exactly like a real FC (gyro init → EKF alignment → GPS → origin set), so the same reading skills apply.
- Loiter in SITL needs the virtual throttle stick centred first (`rc 3 1500`), the same logic as on a real radio.
- Every SITL flight leaves a `.BIN` log. That's the evidence trail for future code changes.

## Next
SITL under gdb with a breakpoint in `AP_OpticalFlow`. ([guide 03](../guides/03-first-sitl-flight.md))
