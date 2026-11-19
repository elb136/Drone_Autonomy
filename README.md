# Drone Autonomy Log

My work log as I move from flight-controller integration into **autonomy**: ArduPilot C++, visual-inertial odometry (VIO) and SLAM, and GPS-denied flight.

**Goal:** ArduPilot holding position on VIO alone, with no GPS and no optical-flow sensor, in Gazebo simulation. Accuracy is measured against ground truth, and failure cases are documented.

**Background:** I've built and tuned ArduPilot quads with optical-flow position hold, from 4" cinewhoops down to a 2S micro whoop on custom firmware: [Project Micro](https://github.com/elb136/Project-Micro), [Project Nano](https://github.com/elb136/Project-Nano), [Project Pico](https://github.com/elb136/Project-Pico). I also wrote the [ArduPilot Log Field Guide](https://elb136.github.io/ardupilot-log-guide/). This repo is the next step: replacing the flow sensor with a camera and an estimator.

## Progress

| Phase | Status | What it delivers |
|---|---|---|
| 0. Dev environment | In progress | Ubuntu, C++ toolchain, ArduPilot SITL, ROS 2 Humble, Gazebo Harmonic |
| 1. Foundations | Not started | ArduPilot driver internals; first VIO run on the EuRoC dataset |
| 2. Contribute and measure | Not started | First upstream ArduPilot PR; VIO benchmark scored with `evo` |
| 3. Flagship | Not started | VIO → ArduPilot ExternalNav in Gazebo, GPS-free Loiter, stress tests |
| 4. Publish | Not started | Write-up, video, results |

Detailed checklist: [ROADMAP.md](ROADMAP.md)

## Repo layout

```
log/       dated entries: what I did, what broke, what I learned
guides/    step-by-step guides I wrote so I can redo or explain each setup
ROADMAP.md the full milestone checklist
```

## Latest entries

- [2026-11-19: Dev environment, part 1 (toolchain, Git/SSH, VS Code)](log/2026-11-19-dev-environment.md)
