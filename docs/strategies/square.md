# Square mission

The legacy mission, kept around as the simplest end-to-end smoke test:
drive a 2 m × 2 m square in body frame. No perception, no search, no
recovery. If it works, the whole control pipeline is healthy.

## Goal

Drive a closed 2 m × 2 m box using body-frame waypoints, holding the
vehicle's starting depth throughout.

## Tree

`bluerov_sim/scripts/bluerov_square_mission_tree.py:62–69` —
`Sequence(memory=True)`:

``` mermaid
flowchart TD
  ROOT["Sequence(memory=True)"]
  ARM[arm_and_set_mode]
  L1["leg1_forward (+x 2 m)"]
  L2["leg2_left (+y 2 m)"]
  L3["leg3_backward (-x 2 m)"]
  L4["leg4_right (-y 2 m)"]

  ROOT --> ARM --> L1 --> L2 --> L3 --> L4
```

All four legs use `goto.FromConstant` (defined at `goto.py:83`).

## Per-leg detail

| Leg | Behaviour | Goal | Notes |
|---|---|---|---|
| `arm_and_set_mode` | py_trees | Arm + GUIDED mode | Standard arming preamble. |
| `leg1_forward` | `goto.FromConstant` | `PoseStamped(base_link, +x=2.0)` | Defaults: thresholds 0.2 m / 5°, no depth override. |
| `leg2_left` | `goto.FromConstant` | `PoseStamped(base_link, +y=2.0)` | Same defaults. |
| `leg3_backward` | `goto.FromConstant` | `PoseStamped(base_link, -x=2.0)` | Same defaults. |
| `leg4_right` | `goto.FromConstant` | `PoseStamped(base_link, -y=2.0)` | Same defaults. |

All four route through `/bluerov/controls`.

## Behaviour details that often surprise readers

!!! info "Depth is held implicitly"
    No leg sets `depth_override_value`, so `depth_rel=False` causes the
    action server to publish the current z-coordinate
    (`locomotion_action_server.py:260–267`). The vehicle stays at the
    starting depth.

!!! info "FLU offsets are converted, not the goal poses"
    Mission code emits FLU (`+x` forward, `+y` left, `+z` up). The map
    frame is ENU. The bridge is the
    [`ConvertToControlsPose`](../packages/frames.md) service — mission
    code never writes a map-frame pose directly.

!!! info "Tick at 100 ms"
    `bluerov_square_mission_tree.py:45` ticks the tree every 100 ms so the
    action-client future polls quickly.

## Search / recovery patterns

None. If any leg times out (60 s default in
`locomotion_action_server.py:107–111`) the action fails and the BT halts.

## Cluster_tf integration

None. No perception runs. The only TF required is `map → base_link`,
broadcast by `ground_truth_to_mavros.py`.

## See also

- [Primitives](primitives.md) — `goto`, `LocomotionActionServer`.
- [Architecture](../overview/architecture.md) — what each layer of the
  pipeline does.
