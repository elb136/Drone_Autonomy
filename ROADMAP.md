# Roadmap

Tick items as they're done (`[x]`). Each finished milestone gets a dated entry in `log/`.

## Phase 0: Dev environment

Ubuntu 26.04 (arm64 VM on Apple Silicon). ArduPilot's ROS 2 guides are written for Ubuntu 22.04 + ROS 2 Humble + Gazebo Harmonic; on 26.04 the matching versions are **ROS 2 Lyrical Luth + Gazebo Jetty**. Decide at the ROS 2 step whether ArduPilot's ROS 2 packages work on Lyrical, or whether a second 22.04 VM is needed.

- [x] Ubuntu 26.04 installed (UTM/QEMU VM, arm64)
- [x] Build tools: git, cmake, ninja, gcc/clang, gdb, clangd, ccache, bear ([guide](guides/01-dev-environment.md))
- [x] Git identity + SSH key connected to GitHub
- [x] VS Code (arm64, from Microsoft's apt repo) with clangd, C/C++, CMake Tools, Python
- [x] Hello-world check: CMake build, breakpoint in debugger, clangd hover
- [x] This log repo created
- [x] ArduPilot cloned, prerequisites installed, `./waf configure --board sitl && ./waf copter` builds ([guide](guides/02-ardupilot-sitl-build.md))
- [x] `compile_commands.json` for ArduPilot, so clangd navigation works in the ArduPilot tree
- [x] `sim_vehicle.py -v ArduCopter` flies takeoff, Loiter and landing from MAVProxy ([guide](guides/03-first-sitl-flight.md))
- [x] QGroundControl on the Mac connected to SITL (`--out=udp:192.168.64.1:14550`)
- [ ] SITL under gdb, breakpoint hit inside `AP_OpticalFlow`
- [ ] ROS 2 installed (Lyrical on 26.04, or Humble in a 22.04 VM); ArduPilot "ROS 2 with SITL" guide working (`ros2 node list` shows `/ap`)
- [ ] Gazebo (Jetty on 26.04, or Harmonic on 22.04) + `ardupilot_gazebo` plugin, iris model flying from SITL (check 3D performance in the VM first)
- [ ] Python tools: pymavlink and MAVProxy ✅ (from ArduPilot's prereqs); still to add: evo, numpy, matplotlib

## Phase 1–2, Track A: C++

- [ ] A1. Read `AP_OpticalFlow` and `AP_RangeFinder` end to end (front end → back ends); class diagram in notes
- [ ] A2. First change in SITL: a debug parameter or log field in the flow driver
- [ ] A3. RCIN-switch feature at upstream quality (rebased, parameterised, ArduPilot style, SITL autotest)
- [ ] A4. RCIN PR opened and reviewed
- [ ] A5. Two small upstream PRs (param docs, small fixes, newcomer issues)
- [ ] A6. CMake + Eigen frames library (quaternions, NED↔ENU, camera↔body) with GoogleTest tests

## Phase 1–2, Track B: SLAM/VIO without hardware

- [ ] B1. Theory pass 1: *Introduction to Visual SLAM* (rigid-body motion, cameras, features, VO) + Stachniss lectures
- [ ] B2. OpenVINS on EuRoC (`V1_01_easy`, `MH_01_easy`), trajectory in RViz
- [ ] B3. ATE/RPE with `evo` on 4+ EuRoC sequences
- [ ] B4. Second algorithm (ORB-SLAM3 or VINS-Fusion), compared on the same sequences
- [ ] B5. OpenVINS on UZH-FPV: find where it breaks, tune it
- [ ] B6. Theory pass 2: IMU preintegration, MSCKF
- [ ] B7. Public benchmark repo with scripts, configs, results

## Phase 3: Flagship (GPS-free ArduPilot on VIO in SITL)

- [ ] F1. Baseline: ArduPilot "Cartographer SLAM with ROS 2 in SITL" guide end to end
- [ ] F2. Camera + IMU in the Gazebo model (rates, intrinsics, extrinsics checked)
- [ ] F3. OpenVINS on simulated sensors, scored against Gazebo ground truth
- [ ] F4. C++ bridge node: VIO → NED/body frame → MAVLink `ODOMETRY`
- [ ] F5. ArduPilot config: `VISO_TYPE` MAVLink, EKF3 ExternalNav, GPS off
- [ ] F6. Fly Loiter and a Guided square; hold error vs ground truth
- [ ] F7. Stress tests: low texture, dim light, fast motion, latency, time offset
- [ ] F8. Ship: README case study, results table, 60 s video, diagram

## Phase 4: Publish

- [ ] V1. Project repos public, profile README live
- [ ] V2. RCIN PR → Discourse + LinkedIn post
- [ ] V3. VIO benchmark → Discourse + LinkedIn post
- [ ] V4. Flagship → Discourse write-up, LinkedIn video, YouTube, Pages case study
- [ ] V5. Talk at an ArduPilot dev call or a local meetup
