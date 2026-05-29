# Architecture

This page is the **mental model**: how mission code in this workspace
reaches the thrusters, layer by layer. If you understand this stack,
every other page in the docs is easier to navigate.

If terms like "py_trees", "MAVROS", "ArduSub", or "TF" are unfamiliar,
read [Concepts](concepts.md) first.

## The 30-second version

The BlueROV2 simulation runs the **exact same firmware** (ArduSub) and
**exact same MAVROS bridge** that the physical vehicle does. Your
mission code talks to MAVROS; everything below MAVROS is identical
between sim and hardware. That portability is the whole point.

## Mission → vehicle pipeline

``` mermaid
flowchart TD
  BT["py_trees BT<br/>(e.g. bluerov_torpedo_mission_tree.py)"]
  CVT["ConvertToControlsPose service<br/>(frames/convert_to_controls_pose.py)"]
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

| Layer | What it does | Who provides it |
|---|---|---|
| **BT (py_trees)** | Mission-level decision logic. Decides "drive there, search, fire". Sends body-frame goals. | `bluerov_sim/bluerov_sim/{bins,torpedo}/*.py`, `bluerov_sim/scripts/bluerov_square_mission_tree.py`. |
| **`ConvertToControlsPose`** | Resolves a body- or anchor-frame goal against the current `base_link` and converts it to a map-frame ENU pose ready for the action server. | [`frames`](../packages/frames.md). |
| **`/bluerov/controls` action** | The single transport contract from mission code to control. | Schema in [`bb_msgs`](../packages/bb_msgs.md). |
| **`LocomotionActionServer`** | Holds the goal, publishes `/mavros/setpoint_position/local` at 1 Hz, polls feedback, declares success / failure. | [`bluerov_sim`](../packages/bluerov_sim.md). |
| **MAVROS → ArduSub** | Standard MAVLink bridge. Same setup as the physical vehicle. | Upstream ROS package. |
| **ArduSub → Gazebo** | The [ArduPilot Gazebo plugin](../packages/ardupilot_gazebo.md) bridges SITL ↔ Gazebo over UDP. | Upstream + a pinned commit in `bluerov_ws.repos`. |

!!! tip "Read each layer's contract, not the implementation"
    The contracts between layers are stable. The implementations
    aren't (and you might rewrite one). When debugging, suspect the
    contracts first. Is the BT sending the action a sensible goal?
    Is the action server publishing setpoints? Is MAVROS connected?

## What closes the loop

The feedback path is just as important as the command path.

``` mermaid
flowchart LR
  GZ[Gazebo physics]
  GT["ground_truth_to_mavros.py<br/>(or dvl_to_mavros.py)"]
  TFT[TF tree:<br/>map → base_link]
  ODOM[/mavros/odometry/out]
  POSE[/mavros/local_position/pose]
  AS[LocomotionActionServer]

  GZ -- ground truth pose --> GT
  GT --> TFT
  GT --> ODOM
  ODOM -- EKF3 external nav --> POSE
  POSE -- "current pose" --> AS
```

- **`ground_truth_to_mavros.py`** is the default. Gazebo's true model
  pose becomes both a TF (`map → base_link`) and an odometry message
  that ArduSub's EKF3 consumes as an external nav source.
- **`dvl_to_mavros.py`** is the alternative (`use_dvl:=true`): uses
  Gazebo DVL twist for velocity but keeps the ground-truth pose. More
  realistic for vehicle-noise testing.

Without one of these running, MAVROS has no idea where the vehicle is,
so the action server's feedback never tightens up and goals never
complete.

## Layered launch (the 5-pane mission)

Each mission `tmuxp` profile (`bluerov_bin_mission.yaml`,
`bluerov_torpedo_mission.yaml`) splits the system across five panes,
one per launch file:

| Pane | Launch file | What it brings up |
|---|---|---|
| `sim` | `bluerov_<task>_sim.launch.py` | Gazebo + ArduSub SITL + MAVROS + Foxglove bridge + URDF |
| `controls` | `bluerov_<task>_controls.launch.py` | `LocomotionActionServer` + `frames`'s convert-pose service |
| `cluster` | `bluerov_<task>_cluster.launch.py` | `cluster_tf` action and service servers |
| `vision` | `bluerov_<task>_vision.launch.py` | YOLO + pose estimators + image matching |
| `bt` | `bluerov_<task>_bt.launch.py` | The mission py_trees tree |

The split lets you `Ctrl-C` any single layer and restart it without
bouncing the others. Useful when iterating on perception or the BT
specifically.

## Key invariants

Changing these silently breaks the stack. They look like preferences
but they're load-bearing.

!!! warning "1 Hz setpoint rate"
    Setpoints to `/mavros/setpoint_position/local` are published at
    **1 Hz** (`PUBLISH_RATE_HZ = 1.0` in the action server). Higher
    rates make ArduSub keep replanning. The vehicle never executes a
    setpoint before it's overwritten, so it stalls in place. **Do not
    raise this.**

!!! warning "10 Hz BT tick + multi-threaded executor"
    The BT ticks at **10 Hz** so action futures resolve responsively
    when goals complete. The action server uses a
    `MultiThreadedExecutor` with a `ReentrantCallbackGroup`.
    Single-threaded executors deadlock on `goal.get_result()` because
    the result callback can't run while the action server thread is
    blocked waiting for it.

!!! warning "`Sequence(memory=True)`"
    The BT root is a `Sequence(memory=True)`. Completed legs are
    remembered, not re-ticked when a subsequent leg fails. Forgetting
    `memory=True` makes the tree restart from leg 0 every tick. **Use
    `memory=True` on every Sequence and Selector** in this
    workspace.

!!! warning "Three independent rel-flags"
    `Locomotion.action` carries **three** independent boolean flags:
    `move_rel` (fwd / sidemove), `depth_rel` (depth), `heading_rel`
    (heading). The action server resolves each axis on its own flag;
    they don't travel together. The torpedo's front-yaw search relies
    on this: `move_rel=True, depth_rel=False, depth=<absolute>` so
    the vehicle yaws while holding an absolute map-z.

!!! warning "`use_sim_time=True` on cluster_tf"
    `bluerov_cluster.launch.py` sets `use_sim_time: true` on the
    cluster_tf servers. Without it, the server captures wall-time
    when a goal starts, then drops every incoming TF as "older than
    start_time" because the TFs are stamped in Gazebo time. Symptom:
    "0 valid transforms collected" with no obvious cause. See
    [Concepts: sim time vs wall time](concepts.md#sim-time-vs-wall-time).

## Anchor frame pattern

Goals carry an `anchor_frame_name` (default `base_link`). When the
anchor is `base_link` the conversion is a no-op. For any other
vehicle-mounted frame (camera, gripper, shooter) `recalculate_target()`
in `frames/convert_to_controls_pose.py` subtracts the anchor's offset
from `base_link`, so the commanded pose lands the **anchor**, not the
body centre, on the target.

This is what lets the bin BT say "put the dropper on the target" or
the torpedo BT say "put the left shooter on the hole" without
hand-rolling the geometry every time.

Mirror this pattern any time you add a new action client that needs
to align something other than the body centre.

## Odometry sources

| Source | Node | What it gives | When to use |
|---|---|---|---|
| Ground-truth (default) | `ground_truth_to_mavros.py` | Publishes Gazebo odom to `/mavros/odometry/out` and broadcasts `map → base_link` TF. Required for EKF3 external nav and for `ConvertToControlsPose`. | Default. Clean closed loop. |
| DVL (`use_dvl:=true`) | `dvl_to_mavros.py` | Substitutes DVL twist for velocity; keeps ground-truth pose for now. | Testing what happens when only velocity (not absolute pose) is available. |

## The closed loop, one more time

Putting the command path and the feedback path together:

``` mermaid
flowchart LR
  BT[py_trees BT]
  AS[LocomotionActionServer]
  MAVROS
  ARD[ArduSub SITL]
  GZ[Gazebo]
  GT[ground_truth_to_mavros]
  POSE[/mavros/local_position/pose]
  TFT[TF tree]

  BT -- Locomotion goal --> AS
  AS -- "setpoint @ 1 Hz" --> MAVROS
  MAVROS -- MAVLink --> ARD
  ARD -- thruster cmd --> GZ
  GZ -- ground-truth pose --> GT
  GT -- odom --> MAVROS
  GT -- TF --> TFT
  MAVROS --> POSE
  POSE -- feedback --> AS
  AS -- "RUNNING / SUCCESS / FAILURE" --> BT
```

The whole loop runs at **1 Hz on the setpoint side** and **10 Hz on
the BT side**. Slow enough that you can watch it tick and reason
about what's happening at each layer.

## See also

- [Concepts](concepts.md): the foundational glossary.
- [Conventions](conventions.md). FLU vs ENU, depth sign, anchor
  frames, the three rel-flags.
- [Running the sim](running.md): how to actually launch this stack.
- [BT strategies](../strategies/index.md): concrete mission
  walkthroughs that exercise this pipeline.
- [bb_msgs](../packages/bb_msgs.md): the `Locomotion.action` schema.
- [frames](../packages/frames.md): the conversion service the BT
  goes through.
- [bluerov_sim](../packages/bluerov_sim.md): the action server +
  ground-truth bridge.
