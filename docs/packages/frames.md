# frames

**Role:** Compile-time C++17 frame-conversion library plus the runtime
`ConvertToControlsPose` service that every BT leg goes through.

The C++ side is header-only with template magic to make `ENUtoNED` /
`NEDtoENU` / `ENUtoFLU` / `FLUtoFRD` (and more) static. Wrong combinations
fail at compile time. Compile-time unit tests run as part of `colcon build`.

## Key entrypoints

| File | Purpose |
|---|---|
| `frames/` (headers) | C++17 templated frame conversions. |
| `scripts/convert_to_controls_pose.py` | The service node BT legs call to turn a body-frame goal into a map-frame `PoseStamped`. |
| `scripts/static_tfs_node.py` | Publishes static TFs from a YAML config. |
| `src/dest_pose_from_tf.cpp` | C++ helper that turns a TF lookup into a `PoseStamped`. |
| `launch/sim_launch.py` | Launches the converter + static TF publishers. |

## ROS interfaces

| Type | Name |
|---|---|
| Service | `<ns>/convert_to_controls_pose` (e.g. `/bluerov/convert_to_controls_pose`) |
| Service | `<ns>/dest_pose_from_tf` |

The service contract is `bb_planner_msgs/GetPoseToControlsFrame.srv`. See
[bb_msgs](bb_msgs.md).

## Conventions implemented

- ROS REP 105: world frames are ENU/NED, vehicle frames are FLU/FRD.
- Rotation order is intrinsic YPR (yaw → pitch → roll), **not** extrinsic
  RPY. Mismatch here is the source of most "looks 90° off" bugs.
- Geodetic ↔ Cartesian conversions take a reference lat/lon.

## Anchor frame logic

`recalculate_target()` in `convert_to_controls_pose.py` is where the
[anchor-frame pattern](../overview/architecture.md#anchor-frame-pattern) is
implemented: it subtracts the anchor's offset from `base_link`, so the goal
ends up relative to the anchor (camera, gripper, dropper) rather than the
robot centre.

## See also

- [Architecture](../overview/architecture.md): where `ConvertToControlsPose`
  sits in the pipeline.
- [bluerov_sim](bluerov_sim.md): the action server that consumes the
  converted pose.
