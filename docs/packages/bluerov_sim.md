# bluerov_sim

**Role:** Orchestrator. Everything that makes the BlueROV2 actually simulate
and execute missions lives here — model files, launch entrypoints, the
locomotion action server, and the behaviour-tree mission code.

## Key entrypoints

| File | Purpose |
|---|---|
| `launch/bluerov_sim.launch.py` | Top-level world launch: Gazebo + ArduSub + MAVROS + Foxglove + URDF. |
| `launch/bluerov_square_bt.launch.py` | BT square mission launcher. |
| `scripts/locomotion_action_server.py` | `/bluerov/controls` action server — the mission→vehicle contract. |
| `scripts/ground_truth_to_mavros.py` | Bridges Gazebo odom → `/mavros/odometry/out` + `map→base_link` TF. |
| `scripts/dvl_to_mavros.py` | Same, but with DVL twist for velocity (`use_dvl:=true`). |
| `scripts/bluerov_square_mission_tree.py` | Legacy square BT. |
| `scripts/goto.py` | py_trees action-client template for `Locomotion` goals. |
| `bluerov_sim/bins/` | Bin-drop BT (mirrored from `mission_planner_2`). |
| `bluerov_sim/torpedo/` | Torpedo BT (mirrored from `mission_planner_2`). |
| `bluerov_sim/shared_trees/` | Reusable search behaviours (single / layered / front-yaw). |
| `models/bluerov2/` | SDF model — Gazebo physics, sensors, thrusters. |
| `urdf/` | URDF — Foxglove visualisation + ROS TF tree. |

## ROS interfaces

| Type | Name | Notes |
|---|---|---|
| Action server | `/bluerov/controls` | `bb_controls_msgs/Locomotion`. Body-frame moves: fwd / sidemove / depth / heading. |
| Service | `/bluerov/convert_to_controls_pose` | Provided by `frames`, namespaced under bluerov. |
| Publishes | `/mavros/odometry/out` | EKF3 external nav (ground-truth path). |
| Publishes | `/bluerov/odom` | Raw Gazebo odom. |
| Broadcasts TF | `map → base_link` | From ground_truth_to_mavros. |

## Gotchas

!!! warning "Setpoint rate"
    MAVROS setpoints are published at **1 Hz**. ArduSub replans every time
    a setpoint lands — higher rates make the vehicle stall in place.

!!! warning "URDF and SDF must change together"
    URDF drives Foxglove/TF; SDF drives Gazebo physics. Editing one without
    the other gives a silent mismatch. See
    [Conventions](../overview/conventions.md#urdf-and-sdf-must-change-together).

!!! info "All editable Python lives here"
    Mission code, action server, ground-truth bridge — all in `scripts/` or
    `bluerov_sim/`. Build with `--symlink-install` and edits take effect
    without rebuilding.

## See also

- [Architecture](../overview/architecture.md) — the full pipeline.
- [Strategies](../strategies/index.md) — what each BT actually does.
- [frames](frames.md) — the conversion service used by every BT leg.
- [bb_msgs](bb_msgs.md) — the `Locomotion.action` schema.
