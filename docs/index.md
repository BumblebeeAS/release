# BlueROV Workspace

A ROS 2 Humble + Gazebo Harmonic simulation stack for the **BlueROV2**.
The design constraint shaping the whole repo: **mission code is
identical between simulation and the real vehicle**.

The full pipeline:

``` mermaid
flowchart LR
  GZ[Gazebo Harmonic] <-- JSON/UDP --> AP[ArduPilot Gazebo plugin]
  AP <--> SITL[ArduSub SITL]
  SITL <-- MAVLink --> MR[MAVROS]
  MR <-- ROS 2 --> MISSION[Mission BT / action server]
```

ArduSub SITL runs the **same firmware** as the physical BlueROV2, so
the MAVROS mission code on top is portable to hardware without
changes.

## New here? Read in this order

1. **[Overview → Concepts](overview/concepts.md)**: survival glossary
   for newcomers. ROS 2 primitives, TF, behaviour trees, MAVROS,
   ArduSub, cluster_tf, anchor frames. Read first if any of those
   look unfamiliar.
2. **[Overview → Architecture](overview/architecture.md)**: the
   layered pipeline above, expanded layer by layer. Closed-loop
   feedback, key invariants, anchor-frame pattern.
3. **[Overview → Running the sim](overview/running.md)**: Docker
   setup, tmuxp missions, diagnostic commands.
4. **[Strategies → Square](strategies/square.md)**: the simplest
   end-to-end mission. Read the code alongside; it's the smallest
   thing in the repo that exercises the whole pipeline.
5. **[Strategies → Primitives](strategies/primitives.md)**: `goto`,
   the action server, the search builders. Vocabulary every mission
   uses.
6. **[Strategies → Bin](strategies/bin.md)** and
   **[Torpedo](strategies/torpedo.md)**: the perception-driven
   missions. Read after Square + Primitives.
7. **[Packages](packages/index.md)**: one page per `src/` package.
   Reach for these when you want to know what a specific dependency
   does or how to extend it.

## Where to start by background

<div class="grid cards" markdown>

-   :material-school:{ .lg .middle } **Brand-new to robotics**

    ---

    Start with [Concepts](overview/concepts.md). Then read
    [Architecture](overview/architecture.md) and
    [Square mission](strategies/square.md) side by side with the
    source.

-   :material-rocket:{ .lg .middle } **ROS-savvy but new to RoboSub**

    ---

    Skim [Architecture](overview/architecture.md), then jump straight
    to [Strategies](strategies/index.md) for what each competition
    mission actually does.

-   :material-wrench:{ .lg .middle } **Extending a specific package**

    ---

    [Packages](packages/index.md) has one focused page per `src/`
    package. Conventions for new BT legs are in
    [Conventions](overview/conventions.md).

</div>

## Workspace layout at a glance

```
bluerov_ws/
├── build.bash / run.bash       # docker image + rocker wrapper
├── bluerov_*.yaml              # tmuxp mission launchers (one per task)
├── bluerov_ws.repos            # vcs-imported external packages (pinned)
└── src/
    ├── bluerov_sim/            # orchestrator: SDF + URDF + launch + action server + BT
    ├── frames/                 # frame conversion library + ConvertToControlsPose
    ├── bb_msgs/                # Locomotion.action, GetPoseToControlsFrame.srv, perception msgs
    ├── bb_worlds/              # competition worlds (RoboSub 2023/2025, SAUVC, …)
    ├── ardupilot_gazebo/       # JSON-UDP bridge plugin (Gazebo ↔ ArduSub SITL)
    ├── dave/                   # DAVE underwater sim ecosystem (sensor plugins, DVL msg types)
    ├── vision_pipeline/        # perception lifecycle orchestration
    ├── pose_estimator/         # vision → TF broadcasters
    ├── image_matching/         # XFeat template matching
    ├── image_processing/       # brighten / restamp / utils
    ├── yolo_ros_trt/           # YOLOv11 + TensorRT (or PyTorch fallback)
    ├── ml_models/              # YOLO weights + Depth-Anything ONNX assets
    ├── filters/                # cluster_tf (TFs + Poses)
    ├── bring-up/               # robot bring-up entrypoints
    └── foxglove-sdk/           # foxglove bridge (manually cloned)
```

## What this is *not*

- Not a generic ROS 2 starter. Assumes Humble, Gazebo Harmonic, and
  the BlueROV2 model specifically.
- Not a tutorial for ArduSub or QGroundControl. Those are upstream
  tools used as-is. We link to their docs where relevant.
- Not an SDK for "your own AUV". Re-targeting to a different vehicle
  is a real project (new SDF, new URDF, new thruster layout, new
  ArduPilot params).

## Anything that surprised you?

If a concept on this site reads like jargon, that's a doc bug. The
[Concepts](overview/concepts.md) page is supposed to translate every
piece of vocabulary the other pages use. PRs / issues welcome on the
repo linked at the top right.
