# BT primitives

A behaviour-tree mission in this workspace is built out of a small
vocabulary of reusable pieces. This page is the dictionary. Every
mission page below references these names.

If words like "action server", "blackboard", "TF" or "behaviour tree"
are unfamiliar, read [Concepts](../overview/concepts.md) first.

## The three pieces every mission rests on

1. **`goto.py`**: the action-client template. Every "move the robot"
   leg in every mission goes through this module.
2. **`LocomotionActionServer`**: the action server that actually
   drives the vehicle. Listens on `/bluerov/controls`.
3. **`shared_trees/search.py`**: reusable search behaviours
   (single-layer square, layered square, front-camera yaw scan) that
   wrap `goto` with cluster_tf bookkeeping.

The rest of this page goes through each in detail.

---

## `goto.py`. Action-client template

`bluerov_sim/scripts/goto.py` is a thin py_trees action-client layer
between behaviour code and the locomotion action server. Every "drive
to a pose" leg is one of four classes from this module.

### What `goto` does on each tick

When a `goto` leaf is ticked for the first time:

1. **Builds (or reads from the blackboard) a `PoseStamped`** in some
   frame. `base_link` for body-relative, `map` for world-relative,
   `front_cam_optical` or another vehicle frame to "place the camera
   on" a target.
2. **Calls `/bluerov/convert_to_controls_pose`** (provided by
   [frames](../packages/frames.md)) to resolve the pose against the
   chosen *anchor frame* and convert it to a map-frame ENU pose ready
   for the action server.
3. **Sends the converted pose as a `Locomotion` goal** to
   `/bluerov/controls`.

On subsequent ticks, it polls the action future. The behaviour returns
`RUNNING` until the action server says the goal succeeded
(`SUCCESS`) or failed (`FAILURE`).

!!! info "Why a separate `convert_to_controls_pose` service?"
    Because the BT writes goals in whichever frame makes the mission
    code natural (`+2 m forward in base_link`, `the camera on
    bin/centre/view`), but the action server only knows how to drive
    to map-frame ENU poses. Centralising the conversion in one service
    means anchor-frame logic isn't duplicated in every behaviour.

### The four `goto` classes

| Class | When to use | Goal source |
|---|---|---|
| `FromConstant` (`goto.py:83`) | The pose is known at compile time. Most common case. | Hardcoded `PoseStamped` passed at construction. |
| `FromBlackboard` (`goto.py:29`) | The pose was produced by a previous behaviour (pose estimation, template selection, …). | Read from a blackboard key at tick time. |
| `NFromConstant` (`goto.py:217`) | A *sequence* of hardcoded poses (search patterns). | List of hardcoded `PoseStamped`s. |
| `NFromBlackboard` (`goto.py:151`) | A sequence of poses produced earlier. | List read from a blackboard key. |

### Common `goto` parameters

These appear on every `goto` call. Get comfortable with them. Most
"why did the vehicle do *that*?" debugging starts with checking these.

| Param | Default | Meaning |
|---|---|---|
| `anchor_frame_name` | `'base_link'` | What part of the robot is supposed to end up at the goal. See [anchor frames](../overview/concepts.md#anchor-frames). Common values in this workspace: `'base_link'`, `'dropper_link'`, `'front_cam_optical'`, `'torpedo_shooter_left_link'`, `'torpedo_shooter_right_link'`. |
| `specified_heading` | `True` | If `True`, the goal isn't "done" until the heading error is within `yaw_threshold`. Set `False` for legs where you only care about position. |
| `is_relative_movement` | `False` | If `True`, the pose offsets are interpreted as body-relative. Used by the yaw-scan in the torpedo BT. Yaw-only deltas in `base_link`. |
| `depth_override_value` | `None` | If set, the action server holds the vehicle at this absolute map-z during the move. Negative numbers go down (ENU). Almost every search leg uses this so the depth doesn't drift while scanning. |
| `x_threshold` / `y_threshold` / `z_threshold` | `0.2 m` | How close (per axis) the action server has to get before declaring success. Tighten these for high-precision alignment (the torpedo BT uses 2.5 cm). |
| `yaw_threshold` | `5.0°` | Heading tolerance. Only enforced when `specified_heading=True`. |
| `stabilize_duration` | `60 s` | How long the goal hold can persist before timing out. |

### Adding a new BT leg. Recipe

1. Build a `PoseStamped` in the right frame:
    - `frame_id="base_link"` for body-relative.
    - `frame_id="map"` for world-absolute.
    - `frame_id="<some_target>/view"` for "land the anchor on
      `<some_target>`".
2. Wrap it in `goto.FromConstant(name=..., pose=..., anchor_frame_name=...)`.
3. Add `depth_override_value=Z` (negative in ENU) if you want depth
   clamped during the move.
4. Set `is_relative_movement=True` *only* for body-frame deltas
   (yaw-only scans).
5. Drop the leaf into your `Sequence(memory=True)` and tick the tree.

!!! tip "Resist putting control logic in behaviours"
    Behaviours assemble goals and read feedback. Anything that looks
    like "if the error is X then send Y" belongs in the action server,
    not in a BT leaf. Keeping behaviours declarative is what makes the
    tree readable.

---

## `LocomotionActionServer`. What actually drives the vehicle

`bluerov_sim/scripts/locomotion_action_server.py` implements the
`/bluerov/controls` action. This is the only component that talks to
MAVROS in the *output* direction (everyone else just subscribes to
MAVROS topics). Its `_build_target` method (`:215–297`) is where each
goal turns into a MAVROS setpoint.

### How `_build_target` resolves a goal

Each axis is resolved on **its own boolean flag**. The three flags do
not interact.

| Flag | What happens when `True` | What happens when `False` |
|---|---|---|
| `move_rel` | `pose.position.{x,y}` interpreted in `base_link`. Action server uses TF to transform to `map`. | `pose.position.{x,y}` interpreted in `map` directly. |
| `depth_rel` | The given `depth` is added to the current map-z. | The given `depth` is the absolute map-z setpoint. |
| `heading_rel` | The given heading is added to the current yaw. | The given heading is the absolute map-yaw setpoint. |

!!! warning "These flags do NOT travel together"
    Earlier versions of this code conflated `depth_rel` with
    `move_rel`. The fix was making each axis independent. The
    torpedo's front-yaw search leans on this: it sends
    `move_rel=True` (body-relative yaw deltas), `depth_rel=False`,
    `depth=<absolute>` (hold an absolute map-z while yawing).

### Why 1 Hz?

Setpoints publish at `PUBLISH_RATE_HZ = 1.0`. Higher rates make
ArduSub keep replanning, and the vehicle stalls in place because it
never gets time to execute the previous setpoint. **Do not change
this.**

### Success condition

A goal succeeds when:

```
distance < dist_threshold
AND (not specified_heading OR yaw_error < yaw_threshold)
```

Default thresholds: 0.2 m position, 5.0° yaw, 60 s timeout. The
feedback published on every tick contains per-axis errors, total
distance error, and yaw error. Handy in Foxglove when debugging an
alignment that won't close.

### Topics & services the action server uses

| Direction | Topic / Service | Purpose |
|---|---|---|
| Subscribes | `/mavros/local_position/pose` | The vehicle's current map-frame pose. Used for feedback and success check. |
| Subscribes | `/mavros/state` | Arming + flight mode. |
| Publishes | `/mavros/setpoint_position/local` | The setpoint stream MAVROS forwards to ArduSub. |
| Provides | `/bluerov/controls` (action) | The mission contract. |

---

## `shared_trees/search.py`. Reusable search behaviours

When perception can't see the target yet, you need a search pattern.
`bluerov_sim/bluerov_sim/shared_trees/search.py` ships several
plug-and-play wrappers. Each one is a small subtree
(`start_cluster → goto_pattern → end_cluster`) that the mission tree
just drops in.

### Available search builders

| Function (`search.py:line`) | Pattern | Used by |
|---|---|---|
| `create_search_bot_constant_root` (`:147`) | Single-layer downward square (4 waypoints, looking down). | Bin coarse approach. |
| `create_search_bot_layered_square_root` (`:326`) | Multi-layer expanding square with cluster-spread validation between layers. | Bin (the main bin search). |
| `create_homing_search_bot_layered_square_root` (`:197`) | Layered square + early-stop topic check (homing beacon). | Not currently wired up. |
| `create_search_front_root` (`:524`) | Front-camera yaw scan. `±N°` in `step°` increments. | Torpedo. |
| `_create_search_bot_root` (`:114`) | Internal helper: wraps a pose list in `Sequence(start_cluster → goto → end_cluster)`. | Used by the two square builders. |
| `_generate_layered_square_search_bot_pattern` (`:79`) | Generates N concentric squares with growing offset. | Internal to the layered builder. |
| `_gen_yaw_points` (`:504`) | Yaw-sweep waypoint generator (left arc + right arc). | Internal to the front-yaw builder. |

### Anatomy of a search subtree

Every search builder produces a subtree of roughly the same shape:

``` mermaid
flowchart TD
  ROOT["Selector(memory=True)"]
  TF["TF-checker:<br/>has the clustered TF appeared yet?"]
  RETRY["Retry(N) on Sequence(memory=True)"]
  START_CL["Start clustering<br/>(call cluster_tf action)"]
  GOTO["Goto pattern<br/>(NFromConstant or NFromBlackboard)"]
  END_CL["End clustering<br/>(wait for cluster_tf to finish)"]

  ROOT --> TF
  ROOT --> RETRY
  RETRY --> START_CL
  START_CL --> GOTO
  GOTO --> END_CL
```

- The **`Selector`** at the root short-circuits the moment the
  clustered TF appears in the TF tree. You don't need to finish the
  pattern if you've already got a good fix.
- The **`Retry`** lets the pattern run again if the cluster didn't
  hold (spread too wide).
- **Start / end clustering** sandwich the goto pattern: the goal is to
  keep the cluster_tf action running while the vehicle is moving so
  detections from across the whole scan are pooled.

### Key parameters across the search builders

| Param | What it does |
|---|---|
| `cluster_srv_name` (default `/bluerov/cluster_tfs_srv`) | The clustering service to call. |
| `out_parents` (default `'map'`) | Parent frame for the clustered TFs. Map/ENU. Bins and torpedo trees rely on this default. |
| `search_depth` | Absolute map-z to hold during the search. Negative in ENU. |
| `object_frame` / `object_frame_clustered` | The raw and clustered TF names. E.g. `bin/yolo` → `bin/yolo/clustered`, `torpedo/yolo` → `torpedo/yolo/clustered`. |
| `cluster_dist_threshold` | Maximum spread (m) of inlier positions before a cluster is accepted. Looser = accept noisier clusters; tighter = reject more, search longer. |
| `min_cluster_size` | Minimum number of inlier points before the cluster is accepted. Default `2` for the front-yaw scan, `4` for the layered square. |
| `cluster_duration` | Seconds the cluster_tf server collects samples before HDBSCAN runs. |

!!! tip "If a search 'always succeeds but the cluster is bad'…"
    …then your `cluster_dist_threshold` is too loose. If a search
    *never* succeeds even though detections are streaming, it's too
    tight. Check both knobs before assuming perception is broken.

---

## Putting it together. Example flow for a perception-driven leg

A typical "find a target and approach it" leg in the bin or torpedo
tree looks like this (in pseudo-Python):

```
Sequence(memory=True, children=[
    # 1. Start in a known coarse pose.
    goto.FromConstant(
        name="goto_vicinity",
        pose=PoseStamped(frame_id="map", x=-6.0, y=4.5),
        depth_override_value=SEARCH_DEPTH,
    ),

    # 2. Run a search pattern. Internally clusters detections into
    #    bin/yolo/clustered. Short-circuits as soon as the cluster
    #    converges.
    create_search_bot_layered_square_root(
        object_frame="bin/yolo",
        object_frame_clustered="bin/yolo/clustered",
        search_depth=SEARCH_DEPTH,
        ...
    ),

    # 3. Switch anchor frame to dropper_link and approach the clustered
    #    target. Now "land on it" means "put the dropper on it".
    goto.FromBlackboard(
        name="goto_align",
        key="...",  # contains a pose stamped on bin/yolo/clustered
        anchor_frame_name="dropper_link",
        depth_override_value=BIN_DEPTH_OVERRIDE_VALUE,
        x_threshold=0.05, y_threshold=0.05, z_threshold=0.05,
    ),

    # 4. Stabilise + actuate.
    Wait(5.0),
    fire_dropper_1,
])
```

You'll see this pattern in slightly different forms on the
[Bin](bin.md) and [Torpedo](torpedo.md) mission pages.

## See also

- [Architecture](../overview/architecture.md): the layered pipeline
  this BT vocabulary sits on top of.
- [Conventions](../overview/conventions.md): frame, sign and naming
  rules.
- [bb_msgs](../packages/bb_msgs.md): the actual `Locomotion.action`
  schema.
- [filters](../packages/filters.md): what `cluster_tf` does under the
  hood.
