# Square mission

This is the simplest mission in the workspace: drive a 2 m × 2 m
closed square in body frame, then stop. No perception, no search, no
recovery. It exists for one reason: **to verify that the whole
control pipeline is healthy**. If the square works end-to-end, every
heavier mission has a chance of working.

!!! tip "Start here if you're new"
    This is the page to read first. The control patterns introduced
    here (`Sequence(memory=True)`, `goto.FromConstant`, body-frame
    legs, depth held implicitly) reappear on every other mission page,
    so getting comfortable here pays off later.

## What the robot actually does

1. Arms the motors and switches ArduSub into `GUIDED` mode.
2. Drives `+2 m forward` (body-relative. In the direction the robot
   is currently pointing).
3. Drives `+2 m left`.
4. Drives `−2 m backward`.
5. Drives `−2 m right`.

The vehicle ends roughly where it started. A closed 2 m × 2 m box in
body frame. Depth is held at the starting depth throughout.

## Where the code lives

`bluerov_sim/scripts/bluerov_square_mission_tree.py:62–69`. The whole
tree is small enough to fit on one screen. Open it side by side with
this page.

## The behaviour tree

`Sequence(memory=True)` containing the arming preamble plus four
`goto.FromConstant` legs in order:

``` mermaid
flowchart TD
  ROOT["Sequence(memory=True)"]
  ARM[arm_and_set_mode]
  L1["leg1_forward (+x 2 m, FLU)"]
  L2["leg2_left (+y 2 m, FLU)"]
  L3["leg3_backward (-x 2 m, FLU)"]
  L4["leg4_right (-y 2 m, FLU)"]

  ROOT --> ARM --> L1 --> L2 --> L3 --> L4
```

Each `goto.FromConstant` builds a `PoseStamped(frame_id="base_link", …)`
and ships it through `/bluerov/convert_to_controls_pose` →
`/bluerov/controls`. See [Primitives](primitives.md) for what those
each do.

!!! info "Why `Sequence(memory=True)` and not a plain `Sequence`?"
    A plain `Sequence` re-runs every child from scratch on every tick.
    The BT ticks at 10 Hz; the first goto leg might take 8 s to
    finish. Without `memory=True` the tree would forget that leg 1 had
    succeeded and restart it on every tick. The vehicle never gets
    past 2 m forward. **Always use `memory=True` on top-level
    sequences and selectors.** This is the single most common BT
    mistake.

## Per-leg detail

| Leg | Behaviour | Goal | Notes |
|---|---|---|---|
| `arm_and_set_mode` | py_trees Action | Arm + switch to `GUIDED` | Standard arming preamble. Reads `/mavros/state`, calls `/mavros/cmd/arming` and `/mavros/set_mode`. |
| `leg1_forward` | `goto.FromConstant` | `PoseStamped(base_link, x=+2.0)` | Default thresholds (0.2 m / 5°), no depth override. |
| `leg2_left` | `goto.FromConstant` | `PoseStamped(base_link, y=+2.0)` | Same defaults. |
| `leg3_backward` | `goto.FromConstant` | `PoseStamped(base_link, x=−2.0)` | Same defaults. |
| `leg4_right` | `goto.FromConstant` | `PoseStamped(base_link, y=−2.0)` | Same defaults. |

All four route through the `/bluerov/controls` action. The same path
every mission uses.

## Behaviours that often surprise readers

!!! info "Depth is held implicitly"
    No leg sets `depth_override_value`, so the action server sees
    `depth_rel=False` and publishes the **current** z as the setpoint
    every tick (`locomotion_action_server.py:260–267`). The vehicle
    naturally maintains its starting depth without any explicit "hold
    depth" behaviour. If you want a specific depth, pass
    `depth_override_value=-1.0` (negative because ENU = below
    surface = negative z).

!!! info "FLU offsets are converted; goal poses are not in map frame"
    Mission code emits FLU (`+x` forward, `+y` left, `+z` up): that's
    the natural way to write "2 m forward". The map frame is ENU. The
    bridge between them is the
    [`ConvertToControlsPose`](../packages/frames.md) service, called
    by `goto`. Mission code never writes a map-frame pose directly.

!!! info "Tick at 10 Hz"
    `bluerov_square_mission_tree.py:45` ticks the tree every 100 ms so
    the action-client future polls quickly. Slower ticks would make
    success-detection lag; faster ticks waste CPU without buying
    anything because setpoints publish at 1 Hz anyway.

!!! info "The `+y` on leg 2 is left, not east"
    `+y` in `base_link` is the robot's left side, not "north" or
    "east". After turning, "left" rotates with the robot. This is
    exactly the point of body-frame goals: "go 2 m left" stays
    correct regardless of how the robot is pointing.

## Search / recovery patterns

None. If any leg times out (60 s default, `locomotion_action_server.py:107–111`)
the action returns `FAILURE` and the BT halts. The square is
deliberately fragile. If it can't drive a clean square, you want to
know immediately, not paper over the problem.

## Cluster_tf integration

None. No perception runs. The only TF required is `map → base_link`,
broadcast by `ground_truth_to_mavros.py` (Gazebo ground-truth →
MAVROS pose path). That TF is the closed-loop feedback that lets the
action server know whether the vehicle has reached the goal.

## How to run it

In the container:

```bash
# Pane 1. Sim
ros2 launch bluerov_sim bluerov_sim.launch.py \
    world_name:=empty.sdf ardusub:=true mavros:=true gui:=true

# Pane 2. The BT
ros2 launch bluerov_sim bluerov_square_bt.launch.py
```

(See [Running the sim](../overview/running.md) for the Docker setup if
you haven't done that yet.)

## How to tell whether it worked

In Foxglove:

- `/tf` panel should show `map → base_link` moving along a 2 m × 2 m
  trace.
- `/mavros/local_position/pose` should follow the same trace.
- `/bluerov/controls/_action/feedback` shows per-axis errors closing
  to zero before each leg completes.

In a terminal:

```bash
ros2 topic echo /bluerov/controls/_action/feedback --once
ros2 topic echo /mavros/local_position/pose --once
```

The position trace returning to the start point (within a few
centimetres) is the smoke-test signal: the control pipeline closes
the loop, and you can move on to the perception-driven missions.

## See also

- [Primitives](primitives.md): what `goto`, `Sequence(memory=True)`,
  and the action server actually do.
- [Architecture](../overview/architecture.md): the layered pipeline
  the square rides on top of.
- [Bin drop](bin.md) and [Torpedo](torpedo.md): the missions that
  add perception, search and recovery on top of this skeleton.
