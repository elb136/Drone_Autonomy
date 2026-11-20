# 2026-11-20: ArduPilot source + SITL build

**Phase:** 0, Dev environment · **Time:** two evenings

## Done
- Cloned ArduPilot (official repo, with submodules) and installed its prerequisites, including the `venv-ardupilot` Python environment with MAVProxy and pymavlink
- Built ArduCopter SITL on arm64: 2 min 25 s for a full build, 1.5 s for a no-change rebuild
- clangd navigation working across the ArduPilot tree, using the `compile_commands.json` that waf generates
- Found the VM runs **Ubuntu 26.04**, not 22.04 as I'd assumed; corrected the earlier docs

## What broke, and the fix
- **`Failed building wheel for wxpython`.** No prebuilt wxPython for Linux arm64, so pip compiled it from source and failed. It's only needed for MAVProxy's console/map windows. Re-ran the prereqs script with `SKIP_AP_GRAPHIC_ENV=1`.
- **`apt`: "Release file is not valid yet".** The VM clock was about 19.5 h behind after the VM had been paused, so apt rejected package lists dated "in the future". Re-enabled NTP time sync (`timedatectl set-ntp true`, restarted `systemd-timesyncd`).

## Learned
- SITL is the real flight code built for Linux instead of an STM32, which is why the whole autonomy plan can start without hardware.
- The build summary's flash columns are what limit real-board firmware. It's the same limit that forced a custom FlywooF745 build to fit optical flow on the Pico.
- A build system's main job is rebuilding only what changed (1.5 s instead of 2.5 min).

## Risk noted
ArduPilot's ROS 2 guides target Ubuntu 22.04 + ROS 2 Humble. On 26.04 the matching versions are ROS 2 Lyrical Luth + Gazebo Jetty. To check at the ROS 2 step; fallback is a separate 22.04 VM.

## Next
First SITL flight (arm, takeoff, Loiter), QGroundControl on the Mac connected to SITL, then SITL under gdb. ([guide 02](../guides/02-ardupilot-sitl-build.md))
