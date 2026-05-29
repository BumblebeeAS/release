# Conventions

The quirks of frames, units, and signs that silently break code if
violated. These aren't preferences. They're load-bearing. Skim them
before reading any mission code or you'll spend hours wondering why
the robot is driving the wrong way.

If "frame", "FLU", "ENU", or "anchor frame" are unfamiliar, read
[Concepts](concepts.md) first.

## Body-frame offsets are FLU

Offsets written in mission code are in the `base_link` frame, **FLU**:

| Axis | Direction |
|---|---|
| `+x` | forward |
| `+y` | left |
| `+z` | up |

Anything fed to `/mavros/setpoint_position/local` is map-frame **ENU**.
The conversion happens inside `ConvertToControlsPose`, so **mission
code never writes a map-frame pose directly**.

!!! info "Why FLU for body-relative?"
    Because mission code is naturally body-relative: "go forward 2 m",
    "turn left 30°", "lift the camera up". FLU lets you write those
    intuitively without thinking about the world frame. The
    conversion service handles the world-frame maths for you.

!!! tip "Sign check before you tick"
    The single most common "the robot did the wrong thing" bug is a
    sign flip on a body-frame offset. Re-read your offset out loud
    before pushing: "I'm sending positive x, that's forward in FLU,
    so the robot should drive forward." Two seconds saves an hour.

## Depth sign. ENU is negative below the surface

| Stack | Convention | Below the surface is… |
|---|---|---|
| map/ENU (this workspace) | ENU | **negative** z |

`SEARCH_DEPTH`, `BIN_DEPTH_OVERRIDE_VALUE`, every depth constant in
the BT modules is map/ENU. "1 m below the surface" is `z = -1.0`. If
you cross-reference NED-based code anywhere else, remember to flip the
sign.

!!! warning "If you see a positive depth, suspect a NED leak"
    A `SEARCH_DEPTH = 1.0` somewhere in this workspace would mean "1 m
    above the surface". Almost certainly a bug. If you spot one,
    it's probably a copy-paste from NED reference code that didn't
    get the sign flip.

## URDF and SDF must change together

The BlueROV2 model is defined in **two** files:

| File | Drives | Used by |
|---|---|---|
| `urdf_path` (`urdf/bluerov2.urdf`) | Foxglove visualisation + ROS TF tree | `robot_state_publisher` |
| `model_sdf_path` (`sdf/bluerov2/model.sdf`) | Gazebo physics, collisions, sensors, thrusters | Gazebo |

Editing one without the other yields a silent mismatch. Foxglove
shows one robot, physics simulates a different one, and the TF tree
disagrees with where the robot really is. Mirror joint and TF frame
names across both files.

!!! warning "Common silent mismatch"
    Adding a new sensor frame to the URDF without adding the
    corresponding link/joint to the SDF means the sensor pose looks
    right in Foxglove but the actual sensor data (camera image, IMU,
    DVL) comes from a slightly different pose. Pose estimators built
    against the wrong intrinsics drift, and you'll chase the drift in
    perception code that's actually fine.

## The three independent rel-flags

`Locomotion.action` carries three boolean flags. The action server
evaluates **each axis independently**:

| Flag | `True` semantics | `False` semantics |
|---|---|---|
| `move_rel` | `pose.position.{x,y}` is in `base_link`; converted to map via TF. | `pose.position.{x,y}` is in `map` directly. |
| `depth_rel` | `depth` is added to the current map-z. | `depth` is the absolute map-z setpoint. |
| `heading_rel` | `heading` is added to the current yaw. | `heading` is the absolute map-yaw setpoint. |

**Do not assume they travel together.** The torpedo's front-yaw search
sends `move_rel=True, depth_rel=False, depth=<absolute>` so the
vehicle yaws (body-relative) while holding an absolute depth.
Conflating the flags is a common bug. Earlier versions of this code
treated `depth_rel` as following `move_rel` and the depth axis drifted
during searches.

## Setpoint rate is 1 Hz

`/mavros/setpoint_position/local` is published at **1 Hz**
(`PUBLISH_RATE_HZ = 1.0` in `locomotion_action_server.py`). Higher
rates cause ArduSub to keep replanning and the vehicle stalls in
place. **Don't raise it.** If you need finer control, change the
target pose, not the rate.

## BT tick is 10 Hz, executor is multi-threaded

BT ticks every 100 ms so action-client futures resolve responsively.
The action server uses a `MultiThreadedExecutor` with a
`ReentrantCallbackGroup`. Single-threaded executors deadlock on
`goal.get_result()` because the result callback can't run while the
action thread is blocked.

## `Sequence(memory=True)` and `Selector(memory=True)`

py_trees `Sequence` and `Selector` re-run their children from child 0
on every tick unless `memory=True`. Mission legs take seconds; ticks
fire every 100 ms. Without `memory=True`, the tree restarts the first
leg forever.

**Always use `memory=True`** for top-level (and most nested)
sequences and selectors in this workspace. The square mission's tree
is `Sequence(memory=True)`; the bin and torpedo trees' top-level is
`Sequence(memory=True)` wrapping a `Selector(memory=True)`.

## Adding a new BT leg / action client

Use `bluerov_sim/scripts/goto.py` as the template. It's a thin
py_trees action client that calls `ConvertToControlsPose` first, then
sends a `Locomotion` goal. See
[Primitives](../strategies/primitives.md) for the full walkthrough.

!!! tip "Behaviours assemble goals; control logic lives in the server"
    Anything that looks like "if the error is X then publish Y"
    belongs in the action server, not in a BT leaf. BT leaves are
    declarative: "drive to this pose with this anchor". Resist the
    urge to put PID-ish logic in a behaviour.

## Tree layout

Behaviour-tree mission code lives in three subdirectories under
`bluerov_sim/bluerov_sim/`:

| Directory | Contents |
|---|---|
| `bins/` | `bins.py` (root tree), `template_selector.py` (rotated vs unrotated template picker). |
| `torpedo/` | `torpedo.py` (root tree), `move_and_shoot_seq.py` (per-torpedo align + fire generator), `point_correspondences_check.py` (template selection helper). |
| `shared_trees/` | `search.py` (square / layered / front-yaw search builders), plus pose / blackboard / TF-checker helpers shared between bin and torpedo. |

Conventions any new tree should follow:

1. **Hardcoded namespace.** Each task module sets
   `NAMESPACE = "/bluerov/<task>"` at the top of the file and uses
   `fk()` ("full-key") wrappers when reading/writing the blackboard
   so keys can't collide across parallel missions.
2. **Use `goto.py`.** Build legs by wrapping `goto.FromConstant`,
   `goto.FromBlackboard`, `goto.NFromConstant`, or
   `goto.NFromBlackboard`. Never call the action server directly
   from a behaviour. See [Primitives](../strategies/primitives.md).
3. **Cluster in `map`.** Pass `out_parents="map"` to every clustering
   helper so the clustered TFs end up in map/ENU.
4. **Pre-seed shared blackboard keys.** Set `/global/base_link =
   "base_link"` and any `/global/choice_*` keys at the top of the
   root sequence before any behaviour reads them.

## Vision → cluster_tf integration

`cluster_tf` only consumes **TFs**, never pose topics. A pose
estimator must inherit `PoseEstimatorTransformPubNode` (or
`PoseEstimatorDataPubNode`) so it broadcasts an object frame.
`PoseEstimatorPosePubNode` alone causes `cluster_tf` to collect zero
samples and the BT stalls in the search leg.

| Base class | Broadcasts TF? | Publishes pose topic? | Use for |
|---|---|---|---|
| `PoseEstimatorTransformPubNode` | ✅ | ❌ | cluster_tf-compatible publishers (the default for everything in this workspace). |
| `PoseEstimatorDataPubNode` | ✅ | ✅ | When you also need the pose topic for visualisation or downstream subscribers. |
| `PoseEstimatorPosePubNode` | ❌ | ✅ | Standalone pose publishers (**incompatible with cluster_tf**). |

!!! warning "Sim time on cluster_tf"
    Set `use_sim_time=True` on every `cluster_tf` node so its
    `start_time` lives on Gazebo's clock. Otherwise every TF stamp
    lands "before" the wall-clock `start_time` and is dropped as
    "old". `bluerov_cluster.launch.py` already does this; double-check
    it on any new launch file you write.

## DAVE underwater camera plugin

If you swap the BlueROV2's standard Gazebo camera for the DAVE
`UnderwaterCamera`:

- Consuming ROS nodes must decode **BGR8**, not RGB8.
- There is **no** `CameraInfo` topic. Pose-estimator code needs the
  intrinsics provided some other way (param file, hardcoded, …).
- `dave_gz_sensor_plugins`' install lib must be on
  `GZ_SIM_SYSTEM_PLUGIN_PATH`, or Gazebo silently won't load the
  plugin.

See [packages/dave](../packages/dave.md) for the full caveats.

## See also

- [Concepts](concepts.md): vocabulary for everything on this page.
- [Architecture](architecture.md): where each of these conventions
  fits into the layered pipeline.
- [Primitives](../strategies/primitives.md). `goto.py`, the action
  server, the search builders.
