# bb_msgs

**Role:** Message-definitions umbrella for the BumblebeeAS robotics stack.
Several sub-packages, each owning a topic / service / action contract used
across the workspace.

This package is the *interface boundary* between layers. When you change a
field here you change everything that imports it.

## Sub-packages

| Sub-package | Owns |
|---|---|
| `bb_controls_msgs` | `Locomotion.action` (BT → action server), `Thrusters`, `ThrusterForces`, `Status`. |
| `bb_planner_msgs` | `SetPoint`, `SetPoints` (waypoint sequence), `PlannerFeedbackMsg`, `GetPoseToControlsFrame.srv`. |
| `bb_perception_msgs` | Detection + segmentation msgs, point-correspondence msgs, image-matching toggle service, **cluster APIs** (`ClusterTfAction`/`Srv` + new `ClusterPosesAction`/`Srv`). |
| `bb_auv_msgs` / `bb_asv_msgs` / `bb_uav_msgs` | Vehicle-specific sensor/control interfaces. |
| `bb_robosub_msgs` | RoboSub competition task msgs. |
| `yolo_msgs` | Detection array (used by [yolo_ros_trt](yolo_ros_trt.md)). |
| `intercomm_msgs` | Inter-vehicle comms. |

## The two contracts you'll touch most often

### `bb_controls_msgs/Locomotion.action`

The BT → action server contract.

**Independent rel-flags**: `move_rel` (fwd/sidemove), `depth_rel` (depth),
`heading_rel` (heading). Each axis is resolved on its own flag inside
`locomotion_action_server.py`. Don't conflate them.

Other notable fields:

- `planner`. Empty string for direct setpoints, or a planner name to
  synthesise a trajectory.
- `max_correction_time`. Caps the feedback window (default 30 s) before
  the goal returns success.
- `specified_heading`. When false, heading errors don't gate success.
- `*_tolerance` fields. Completion thresholds (default 0.2 m position,
  5.0° yaw).

### `bb_planner_msgs/GetPoseToControlsFrame.srv`

The BT → frames service. Takes a body- or anchor-frame `PoseStamped` plus an
`anchor_frame_name`; returns a map-frame `PoseStamped` ready for the action
server.

### `bb_perception_msgs/ClusterTfAction` (and friends)

The BT → `cluster_tf` contract. Goal carries `input_child_frame_ids`,
`out_child_frame_ids`, `out_parent_frame_ids`, plus collection /
HDBSCAN tuning. Bin and torpedo trees build it through a small helper
that defaults `out_parents="map"` and sets per-task collection
durations.

The 2026-05 "New Cluster API" added a parallel **pose-based** path:
`ClusterPosesAction`, `ClusterPosesSrv`, `ClusterPosesRequest`,
`ClusterPoseResult`/`ClusterPoseResultArray`. No BlueROV mission consumes
the pose-based variant yet. Bin/torpedo still use the TF action. But
it's available if you want to skip the TF-broadcast hop. The same
cleanup **removed** `ClusterSpikeStatus.msg`; if you see import errors
referencing it, rebase past it.

## Gotchas

- This is a monorepo of independent `package.xml`s. Each sub-package builds
  on its own. `colcon build --packages-select bb_controls_msgs` is valid.
- Changes here cascade through every dependent package. Rebuild everything
  after edits.

## See also

- [bluerov_sim](bluerov_sim.md): provides the `Locomotion` action server.
- [frames](frames.md): provides the `ConvertToControlsPose` service.
