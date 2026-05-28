# Architecture

Mission code reaches the thrusters through a layered pipeline. Understanding
this stack is the prerequisite for almost any change in the workspace.

## Mission → vehicle pipeline

``` mermaid
flowchart TD
  BT["py_trees BT<br/>(bluerov_square_mission_tree.py)"]
  CVT["GetPoseToControlsFrame service<br/>(frames/convert_to_controls_pose.py)"]
  ACT["/bluerov/controls action<br/>(bb_controls_msgs/Locomotion)"]
  AS["LocomotionActionServer<br/>(bluerov_sim/scripts/locomotion_action_server.py)"]
  MR[MAVROS]
  SUB[ArduSub SITL]
  GZ[Gazebo thrusters]

  BT -- "body-frame setpoint (FLU)" --> CVT
  CVT -- "map-frame PoseStamped" --> ACT
  ACT --> AS
  AS -- "/mavros/setpoint_position/local @ 1 Hz" --> MR
  MR -- MAVLink --> SUB
  SUB --> GZ
```

| Layer | What it does |
|---|---|
| **BT (py_trees)** | Mission-level decision logic. Sends body-frame goals. |
| **`ConvertToControlsPose`** | Resolves the body-frame goal against the current `base_link` (and any anchor frame) to produce a map-frame pose. |
| **`/bluerov/controls` action** | The single transport contract from mission code to control. Defined by `bb_controls_msgs/Locomotion.action`. |
| **`LocomotionActionServer`** | Holds the goal, publishes setpoints to MAVROS, polls feedback, reports done. |
| **MAVROS → ArduSub** | Standard MAVLink bridge — same setup as the physical vehicle. |
| **ArduSub → Gazebo** | The ArduPilot Gazebo plugin bridges SITL to thruster joints over UDP. |

## Key invariants

Changing these breaks the stack silently:

!!! warning "1 Hz setpoint rate"
    Setpoints to MAVROS are published at **1 Hz**. Higher rates cause
    ArduSub to keep replanning and the vehicle stalls in place.

!!! warning "10 Hz BT tick + multi-threaded executor"
    The BT ticks at **10 Hz** so action futures resolve responsively. The
    action server uses a `MultiThreadedExecutor` with a
    `ReentrantCallbackGroup`. Single-threaded executors deadlock on
    `goal.get_result()`.

!!! warning "Sequence(memory=True)"
    The BT root is a `Sequence(memory=True)` — completed legs aren't
    re-ticked when a subsequent leg fails. Forgetting `memory=True` makes
    the tree restart from leg 0.

!!! warning "Three independent rel-flags"
    `Locomotion.action` carries **three** independent boolean flags —
    `move_rel` (fwd/sidemove), `depth_rel` (depth), `heading_rel`
    (heading). The action server resolves each axis on its own flag; do
    **not** conflate them. The front-yaw search sends
    `move_rel=True, depth_rel=False, depth=<absolute>` so the vehicle yaws
    while holding an absolute map-z setpoint.

## Anchor frame pattern

Goals carry an `anchor_frame_name` (default `base_link`). When the anchor is
`base_link` the conversion is a no-op. For any other vehicle-mounted frame
(camera, gripper, …) `recalculate_target()` in
`frames/convert_to_controls_pose.py` subtracts the anchor's offset from
`base_link`, so the commanded pose is relative to the anchor rather than the
body centre.

Mirror this pattern when adding new action clients.

## Odometry sources

| Source | Node | What it gives |
|---|---|---|
| Ground-truth (default) | `ground_truth_to_mavros.py` | Publishes Gazebo odom to `/mavros/odometry/out` and broadcasts `map → base_link` TF. Required for EKF3 external nav and for `ConvertToControlsPose`. |
| DVL (`use_dvl:=true`) | `dvl_to_mavros.py` | Substitutes DVL twist for velocity; keeps ground-truth pose. |

## See also

- [Conventions](conventions.md) — frame, unit and sign rules.
- [Running the sim](running.md) — how to actually launch this stack.
- [BT strategies](../strategies/index.md) — concrete mission walkthroughs.
