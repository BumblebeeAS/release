# vision_pipeline

**Role:** Meta-package that orchestrates the perception stack. Bundles
launch files and provides lifecycle + composable-node managers so a single
service call can flip a whole task pipeline on or off.

## Key entrypoints

| File | Purpose |
|---|---|
| `launch/gate.launch.py`, `bin.launch.py`, `torpedo.launch.py`, `slalom.launch.py`, `trash.launch.py`, `image_matching.launch.py` | One launch per RoboSub task. |
| `vision_pipeline/lifecycle_manager_node.py` | Drives `LifecycleNode` state transitions (`configure` → `activate`). |
| `vision_pipeline/component_manager_node.py` | Loads / unloads composable nodes inside a container. |
| `config/auv4_orin/*.yaml` | Per-task model / topic configs. |

## ROS interfaces

| Type | Name |
|---|---|
| Service | `manage_nodes` (`lifecycle_msgs/srv/ChangeState`) |
| Service | `manage_components` (`std_srvs/srv/SetBool`) |

Vision nodes are namespaced `/auv4/<task>/...` (preserved verbatim from
upstream).

## Pipeline composition

This package itself owns no perception logic — it composes:

- [yolo_ros_trt](yolo_ros_trt.md) — segmentation
- [pose_estimator](pose_estimator.md) — PnP → TF
- [image_matching](image_matching.md) — XFeat correspondences
- [image_processing](image_processing.md) — brighten / restamp
- [filters](filters.md) — clustering / SORT3D
- `aruco_loco`, `depth_anything_ros2_trt`, `custom_image_republisher` — auxiliary

## Gotchas

!!! warning "Lifecycle state"
    Lifecycle nodes start **inactive**. They must be activated explicitly
    via `manage_nodes` or `ros2 lifecycle set /<node> activate` before
    they publish anything.

!!! warning "Depth Anything is `.onnx`, not `.engine`"
    Depth Anything configs must reference `.onnx` files. The runtime
    compiles `.engine` on first run.

!!! warning "QoS mismatch"
    Image topics use `sensor_data` QoS (best-effort). Subscribers expecting
    `reliable` QoS will not receive messages — this is the most common
    silent-perception-failure mode.

- All vision nodes run on the Orin in the real robot, consuming
  decompressed images that the SBC republishes
  (`orin_cam_repub.launch.py`).

## See also

- [Strategies](../strategies/index.md) — each BT mission starts its
  matching `vision_pipeline` task on entry and stops it on exit.
