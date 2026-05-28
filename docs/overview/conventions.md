# Conventions

Quirks of frames, units and signs that silently break code if violated.

## Body-frame offsets are FLU

Offsets in mission code are in the `base_link` frame, **FLU**:

- `+x` forward
- `+y` left
- `+z` up

Anything fed to `/mavros/setpoint_position/local` is map-frame **ENU**. The
conversion happens inside `GetPoseToControlsFrame` — mission code never
writes map-frame poses directly.

## Depth sign — read carefully

| Stack | Convention | Below surface is… |
|---|---|---|
| map/ENU (this workspace) | ENU | **negative** z |
| AUV4 (upstream) | NED | **positive** z |

When mirroring a tree from `mission_planner_2`, flip the sign on every
`SEARCH_DEPTH` / `DEPTH_OVERRIDE_VALUE`.

## URDF and SDF must change together

- `urdf_path` drives Foxglove visualisation + ROS TF.
- `model_sdf_path` drives Gazebo physics, collisions, sensors, thrusters.

Editing only one yields a silent mismatch — Foxglove shows one robot, physics
simulates another. Mirror joint and TF frame names across both files.

## Adding a new BT leg / action client

Use `bluerov_sim/scripts/goto.py` as the template — it is a thin `py_trees`
action client that calls `GetPoseToControlsFrame` first, then sends a
`Locomotion` goal.

!!! tip
    Don't put control logic in behaviours; it belongs in the action server.
    Behaviours assemble goals and read feedback.

## Mirroring a `mission_planner_2` tree

Recipe (see `bluerov_sim/bins/` and `bluerov_sim/torpedo/`):

1. **Hardcode the namespace.** `NAMESPACE = "/bluerov/<task>"`. The upstream
   `generate_namespace()` walks the caller path looking for `/trees/`; our
   mirror files aren't under `/trees/` so it fails.
2. **Import the local goto.** Use the ament-prefix `sys.path` hack to import
   `bluerov_sim/scripts/goto.py`, so Locomotion goals land on
   `/bluerov/controls` not `/auv4/controls`.
3. **Patch cluster requests.** Pass `out_parents="map"` to every
   `create_clustering_request` / `create_clustering_goal`. Upstream default
   is `world_ned`.
4. **Swap shared-action enums.** `AUVSharedAction.CLUSTER` →
   `BlueROVSharedAction.CLUSTER`.
5. **Pre-seed blackboard keys.** Set `/global/base_link = "base_link"` and
   any `/global/choice_*` keys at the top of the root sequence. Upstream
   lazily populates them from bootstrap nodes we don't run, so the BT
   `KeyError`s on first read otherwise.

## Vision → cluster_tf integration

`cluster_tf` only consumes **TFs**, never pose topics. A pose estimator must
inherit `PoseEstimatorTransformPubNode` (or `PoseEstimatorDataPubNode`) so it
broadcasts an object frame; `PoseEstimatorPosePubNode` alone causes
`cluster_tf` to collect zero samples and the BT stalls in search.

!!! warning "Sim time"
    Also set `use_sim_time=True` on the `cluster_tf` nodes so their
    `start_time` lives on Gazebo's clock — otherwise every TF stamp lands
    "before" the wall-clock `start_time` and is dropped as "old".

## DAVE underwater camera plugin

If switching to the DAVE `UnderwaterCamera`:

- Consuming ROS nodes must decode **BGR8**, not RGB8.
- There is no `CameraInfo` topic.
- `dave_gz_sensor_plugins`' install lib must be on
  `GZ_SIM_SYSTEM_PLUGIN_PATH`.
