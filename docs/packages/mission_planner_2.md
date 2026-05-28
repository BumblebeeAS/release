# mission_planner_2

**Role:** BumblebeeAS shared py_trees mission stack (originally for AUV4).
**Read-only upstream** for this workspace — we mirror trees and configs
into `bluerov_sim/` rather than editing here.

## Why we mirror instead of edit

`mission_planner_2` assumes AUV4 conventions:

- Namespace `/auv4/...` (not `/bluerov/...`).
- World frame is `world_ned` (we use `map`/ENU).
- `generate_namespace()` walks for `/trees/` in the caller path.
- Tree files live under `mission_planner_2/trees/<vehicle>/<mission>`.
- Choice / bootstrap nodes lazily populate `/global/*` blackboard keys.

Our BlueROV mirrors live outside `/trees/`, use a different namespace, and
don't run the upstream bootstrap, so we have to patch a handful of things.
See [Conventions: Mirroring a mission_planner_2 tree](../overview/conventions.md#mirroring-a-mission_planner_2-tree).

## Key entrypoints

| File | Purpose |
|---|---|
| `launch/main.launch.py` | Static TF publisher + choice server. |
| `scripts/main.py` | Mission executable (py_trees BT). |
| `scripts/choice_server_node.py` | Service server for runtime choice selections (e.g. fish vs shark). |
| `cfg/cfg.yaml` | Static TF configuration (inverse transforms, top-left object origins). |
| `trees/<vehicle>/<mission>/` | Per-mission BTs we mirror. |

## ROS interfaces

| Type | Name |
|---|---|
| Service | Choice / decision server (task-specific). |
| TF | Publishes static transforms for camera and object frames. |
| Action | Calls `bb_controls_msgs/Locomotion` for movement goals. |
| Subscribes | Odometry, detection topics from the vision pipeline. |

## Gotchas

- **Static TFs are inverse:** the config stores `B → A` not `A → B`. Query
  parent → child.
- **Camera frames need explicit RPY:** the front camera is
  `roll=-90, pitch=-90, yaw=0` to align with body frame.
- **Object origins are top-left**, not centre. Pose-estimator code assumes
  this.

## See also

- [bluerov_sim](bluerov_sim.md) → `bins/`, `torpedo/`, `shared_trees/` —
  where the BlueROV mirrors live.
- [Bin mission](../strategies/bin.md), [Torpedo mission](../strategies/torpedo.md)
  — concrete uses of the mirrored upstream behaviours.
