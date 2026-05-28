# BT primitives

The substrate every mission rests on: how BT legs talk to the action server
(`goto`), what the action server actually does, and which reusable
search behaviours live in `shared_trees/`.

## `goto.py` — action-client template

`bluerov_sim/scripts/goto.py` wraps the upstream
`mission_planner_2.vehicles.auv.trees.goto`, substituting BlueROV
action / service registries so `Locomotion` goals land on `/bluerov/controls`
and pose conversion goes through `/bluerov/convert_to_controls_pose`.

Four classes:

| Class | When to use |
|---|---|
| `FromConstant` (`goto.py:83`) | Single hardcoded `PoseStamped`. Most common. |
| `FromBlackboard` (`goto.py:29`) | Goal pose read from a blackboard key. Used after pose estimation. |
| `NFromConstant` (`goto.py:217`) | Sequence of hardcoded poses (search patterns). |
| `NFromBlackboard` (`goto.py:151`) | Sequence of poses from blackboard. |

### Common parameters

| Param | Default | Meaning |
|---|---|---|
| `anchor_frame_name` | `'base_link'` | What the pose is measured from. Other frames (e.g. `dropper_link`, `front_cam_optical`, `torpedo_shooter_left_link`) shift the goal to put that anchor on the target rather than the body centre. |
| `specified_heading` | `True` | If `False`, heading errors don't gate success. |
| `is_relative_movement` | `False` | If `True`, pose offsets are body-relative (yaw-only scans). |
| `depth_override_value` | `None` | Absolute map-z to force during the move. Used by every search leg. |
| `x_threshold` / `y_threshold` / `z_threshold` | `0.2 m` | Position tolerances. |
| `yaw_threshold` | `5.0°` | Heading tolerance. |
| `stabilize_duration` | `60 s` | Time to hold the goal after reaching threshold. |

### Adding a new leg

1. Build a `PoseStamped` in the right frame (`base_link` for body-relative,
   `map` for world).
2. Wrap in `goto.FromConstant(name=..., pose=..., anchor_frame_name=...)`.
3. Set `depth_override_value=Z` to clamp depth during the move.
4. Set `is_relative_movement=True` only for body-frame deltas like yaw-only
   scans.

## `LocomotionActionServer` — what actually drives the vehicle

`bluerov_sim/scripts/locomotion_action_server.py` implements
`/bluerov/controls`. The interesting logic lives in `_build_target`
(approximately `:215–297`):

- `move_rel=True` → transform `base_link` → `map` via TF. Used for
  body-relative moves.
- `depth_rel=True` → add offset to current z.
- `heading_rel=True` → add offset to current yaw.
- Each axis resolves **independently**. (Earlier bug: depth_rel was
  conflated with move_rel.)

Setpoints publish at **1 Hz** (`PUBLISH_RATE_HZ = 1.0`). The goal succeeds
when:

```
distance < dist_threshold
AND (not specified_heading OR yaw_error < yaw_threshold)
```

Default thresholds: 0.2 m, 5.0°, 60 s timeout.

Feedback is published every tick: per-axis errors, yaw error, overall
progress.

## `shared_trees/search.py` — reusable search behaviours

`bluerov_sim/bluerov_sim/shared_trees/search.py` ships several
plug-and-play search wrappers.

| Function (`search.py:line`) | Pattern | Used by |
|---|---|---|
| `create_search_bot_constant_root` (`:147`) | Single-layer downward square (4 waypoints). | Bin coarse approach. |
| `create_search_bot_layered_square_root` (`:326`) | Multi-layer expanding square with cluster-spread validation. | Bin search. |
| `create_homing_search_bot_layered_square_root` (`:197`) | Layered square + early-stop topic check. | Not currently used. |
| `create_search_front_root` (`:524`) | Front-camera yaw scan (`±N° in steps`). | Torpedo. |
| `_create_search_bot_root` (`:114`) | Internal: wraps poses in `Sequence(start_cluster → goto → end_cluster)`. | Helper. |
| `_generate_layered_square_search_bot_pattern` (`:79`) | Generate N concentric squares with growing offset. | Bin. |
| `_gen_yaw_points` (`:504`) | Yaw-sweep waypoints (left arc + right arc). | Torpedo. |

### Key parameters across the search builders

| Param | What it does |
|---|---|
| `cluster_srv_name` (default `/bluerov/cluster_tfs_srv`) | The clustering service to call. |
| `out_parents` (default `'map'`) | Output frame for clustered TFs. **Patched from upstream** `world_ned`. |
| `search_depth` | Depth override during the search. Negative in ENU. |
| `object_frame` / `object_frame_clustered` | The raw and clustered TF names (e.g. `bin/yolo` → `bin/yolo/clustered`). |
| `cluster_dist_threshold` | Maximum spread (m) before a cluster is accepted. |
| `min_cluster_size` | Minimum points required to cluster (default 2 or 4 depending on pattern). |

## See also

- [Architecture](../overview/architecture.md) — pipeline overview.
- [Conventions](../overview/conventions.md) — frame, sign and mirroring rules.
- [bb_msgs](../packages/bb_msgs.md) — `Locomotion.action` schema.
