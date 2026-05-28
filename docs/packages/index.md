# Packages

One page per package in `src/`. Each page lists the package's purpose, key
entrypoints, ROS interfaces it owns, and gotchas worth knowing before editing
it.

## Core simulation & control

<div class="grid cards" markdown>

-   **[bluerov_sim](bluerov_sim.md)** — orchestrator. SDF + URDF + launch +
    action server + behaviour trees.
-   **[frames](frames.md)** — compile-time frame library + the
    `ConvertToControlsPose` service.
-   **[bb_msgs](bb_msgs.md)** — `Locomotion.action`,
    `GetPoseToControlsFrame.srv`, perception msgs.
-   **[bb_worlds](bb_worlds.md)** — competition worlds (SDF + launch).
-   **[mission_planner_2](mission_planner_2.md)** — upstream BumblebeeAS BT
    stack (read-only; trees are mirrored into `bluerov_sim`).

</div>

## Physics & infrastructure

<div class="grid cards" markdown>

-   **[ardupilot_gazebo](ardupilot_gazebo.md)** — JSON/UDP bridge plugin
    (ArduSub ↔ Gazebo).
-   **[dave](dave.md)** — vendored underwater sim ecosystem.
-   **[foxglove-sdk](foxglove-sdk.md)** — Foxglove bridge (manually
    cloned).
-   **[bring-up](bring-up.md)** — robot bring-up / provisioning scripts.
-   **[isaac-ros-docker](isaac-ros-docker.md)** — Isaac ROS docker image
    (vendored, not the base for this workspace).

</div>

## Perception

<div class="grid cards" markdown>

-   **[vision_pipeline](vision_pipeline.md)** — lifecycle / composition
    orchestration for perception nodes.
-   **[yolo_ros_trt](yolo_ros_trt.md)** — YOLOv11 segmentation via
    TensorRT.
-   **[pose_estimator](pose_estimator.md)** — 2D detections → PnP →
    TF broadcast.
-   **[image_matching](image_matching.md)** — XFeat template matching.
-   **[image_processing](image_processing.md)** — brighten, depth→RGB,
    restamp.
-   **[filters](filters.md)** — detection post-processing
    (clustering, SORT3D).
-   **[ml_models](ml_models.md)** — YOLO weights + Depth Anything ONNX
    assets.

</div>
