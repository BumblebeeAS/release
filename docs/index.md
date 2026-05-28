# BlueROV Workspace

A ROS 2 Humble + Gazebo Harmonic simulation stack for the BlueROV2, designed so
mission code is identical between simulation and the real vehicle.

The full pipeline is:

``` mermaid
flowchart LR
  GZ[Gazebo Harmonic] <-- JSON/UDP --> AP[ArduPilot Gazebo plugin]
  AP <--> SITL[ArduSub SITL]
  SITL <-- MAVLink --> MR[MAVROS]
  MR <-- ROS 2 --> MISSION[Mission BT / action server]
```

ArduSub SITL runs the same firmware as the physical BlueROV2, so the MAVROS
mission code on top is portable to hardware without changes.

## Where to start

<div class="grid cards" markdown>

-   :material-map:{ .lg .middle } **Overview**

    ---

    The end-to-end pipeline, frame and unit conventions, how to launch the
    sim.

    [:octicons-arrow-right-24: Architecture](overview/architecture.md)

-   :material-package-variant:{ .lg .middle } **Packages**

    ---

    One page per package in `src/` — purpose, key files, ROS interfaces,
    gotchas.

    [:octicons-arrow-right-24: Browse packages](packages/index.md)

-   :material-tree:{ .lg .middle } **Strategies**

    ---

    Deep dives on the behaviour-tree missions: square, bin drop, torpedo
    launch.

    [:octicons-arrow-right-24: Mission strategies](strategies/index.md)

</div>

## Project layout

```
bluerov_ws/
├── build.bash / run.bash       # docker image + rocker wrapper
├── bluerov_*.yaml              # tmuxp mission launchers
├── bluerov_ws.repos            # vcs-imported external packages
└── src/
    ├── bluerov_sim/            # orchestrator: SDF, URDF, launch, action server, BT
    ├── frames/                 # frame conversion library + ConvertToControlsPose
    ├── bb_msgs/                # Locomotion.action, GetPoseToControlsFrame.srv, …
    ├── mission_planner_2/      # BumblebeeAS shared BT stack (read-only upstream)
    ├── bb_worlds/              # competition worlds (RoboSub 2023/2025, SAUVC, …)
    ├── ardupilot_gazebo/       # JSON-UDP bridge plugin
    ├── dave/                   # DAVE underwater sim ecosystem
    ├── vision_pipeline/        # perception orchestration
    ├── pose_estimator/         # vision → TF broadcasters
    ├── image_matching/         # ORB / feature matching
    ├── image_processing/       # rectification, colour correction
    ├── yolo_ros_trt/           # TensorRT YOLO inference
    ├── ml_models/              # weight files / engines
    ├── filters/                # state filters (compile-time tested)
    ├── bring-up/               # robot bring-up entrypoints
    ├── foxglove-sdk/           # foxglove bridge (manually cloned)
    └── isaac-ros-docker/       # Isaac ROS image (vendored, not used as base)
```
