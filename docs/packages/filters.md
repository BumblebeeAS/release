# filters

**Role:** Cluster a stream of TFs (or Poses) into a single stable output,
plus a handful of detection-projection / labelling nodes. Hosts the
`cluster_tf` action and service that every BT mission depends on. The
torpedo and bin trees both push their YOLO/template TFs through here to
get a stable map-frame target before approaching it.

The ROS package name is `bb_filters` (the repo is named `filters`).

## Build & layout (upstream cleanup, 2026-05)

`bb_filters` was restructured from `ament_cmake` → `ament_python` and the
default branch was renamed `master → main`:

- All runtime nodes moved from `scripts/cluster/` → `bb_filters/nodes/cluster/`.
- Cluster helpers moved from `bb_filters/utils/cluster.py` →
  `bb_filters/utils/cluster/cluster.py` and gained a `get_largest_cluster`
  helper (the old `_get_idxs_in_largest_cluster` private staticmethod is
  gone).
- Legacy detection nodes (`detected_object_3d_filter.py`, `sort_3d.py`,
  most of the RobotX detectors) are now under `bb_filters/archive/` and
  not installed.
- C++ visualisation binaries (`detected_object_2d_vis`, etc.) and the
  `CMakeLists.txt` are gone.

`setup.py` declares its entries via `scripts=[…]` (not `console_scripts`),
so the installed ROS-2 executable names are unchanged. Anything that
launches `cluster_tf_action_server.py` still works. **Build type flipped**
though, so a full clean before the first build after pulling:

```bash
rm -rf build/bb_filters install/bb_filters
colcon build --symlink-install --packages-select bb_filters
```

## Entry points (installed scripts)

| Executable | Role |
|---|---|
| `cluster_tf_action_server.py` | Async **action** handler for `ClusterTfAction` (`/<ns>/cluster_tf`). |
| `cluster_tf_service_server.py` | Sync **service** handler for `ClusterTfSrv` (`/<ns>/cluster_tfs_srv`). |
| `cluster_tf_multi_action_server.py` | Multi-source action variant. |
| `cluster_tf_multi_service_server.py` | Multi-source service variant. |
| `cluster_slalom_tfs_node.py` | RobotX-style slalom clustering. |
| `cluster_poses_action_node.py` / `_node.py` / `_service_node.py` | **New parallel Pose API** (see below). |
| `detected_object_2d_filter_projection.py` | 2D-detection projection cleanup. |
| `detected_object_3d_labelling.py` | Label 3D detections from a TF tree. |

## The two clustering APIs

There are now two parallel cluster APIs in this package. Our BT missions
use the TF-based one; the Pose-based one is new and not consumed by any
BlueROV mission yet.

### TF-based (existing. What BTs use)

- Action: `bb_perception_msgs/ClusterTfAction` on `<ns>/cluster_tf`.
- Service: `bb_perception_msgs/ClusterTfSrv` on `<ns>/cluster_tfs_srv`.
- Server polls the TF buffer at ~12 Hz for each `input_child_frame_id`,
  buffers samples for `collection_duration`, runs HDBSCAN on the positions,
  broadcasts the centroid of the largest cluster as
  `parent → out_child_frame_id`.
- BlueROV missions build goals through a small helper that defaults
  `out_parents="map"` (so clustered TFs land in map/ENU rather than NED).

### Pose-based (new, 2026-05)

- Service: `bb_perception_msgs/ClusterPosesSrv`.
- Action: `bb_perception_msgs/ClusterPosesAction` (rewritten to use
  `ClusterPosesRequest` + `ClusterPoseResultArray`).
- Consumes a stream of poses on a topic rather than TFs, with
  `max_detection_age_s` filtering.
- `ClusterSpikeStatus.msg` was **removed** in the same cleanup.

## ROS interfaces (used by our missions)

| Direction | Topic / Action / Service |
|---|---|
| Action server | `/bluerov/cluster_tf` (`ClusterTfAction`) |
| Service server | `/bluerov/cluster_tfs_srv` (`ClusterTfSrv`) |
| TF input (sub) | Whatever frames mission code passes as `input_child_frame_ids`. E.g. `torpedo/yolo`, `Task04_Tagging_01_optical`, `bin/yolo`, `Task03_DropBRUVS_optical`. |
| TF output (pub) | The clustered frames, e.g. `torpedo_1`, `bin/yolo/clustered`. |

The bin and torpedo BTs both go through these via
`bluerov_cluster.launch.py`. See the [Architecture](../overview/architecture.md)
page for where this sits in the stack.

## Notes & gotchas

!!! warning "use_sim_time is required"
    The cluster_tf servers must be started with `use_sim_time: True`
    (`bluerov_cluster.launch.py` already does this). Without it, the
    server captures wall time at goal-start while incoming TFs are
    stamped with Gazebo sim time. Every TF then looks "decades old"
    and gets dropped as stale.

!!! warning "Source frame must actually exist"
    `cluster_tf` only **reads** the TF tree, never poses. If the BT hands
    it an `input_child_frame_id` that has no broadcaster, you get a
    `tf2.LookupException` loop and `0 valid transforms`. Make sure a
    pose estimator (e.g. `points_pose_estimator_node`, `bin_pose_estimator_node`,
    `torpedo_pose_estimator_node`) is publishing the source frame before
    the BT asks for it.

!!! info "MultiThreadedExecutor removed"
    The 2026-05 cleanup ripped out `MultiThreadedExecutors` from the
    cluster nodes. Single-threaded performs better in practice.

## See also

- [pose_estimator](pose_estimator.md): broadcasts the source TFs
  consumed by cluster_tf.
- [bb_msgs](bb_msgs.md): owns `ClusterTfAction`, `ClusterTfSrv`, and
  the new `ClusterPoses*` messages.
- [Bin mission](../strategies/bin.md), [Torpedo mission](../strategies/torpedo.md):
  primary consumers.
