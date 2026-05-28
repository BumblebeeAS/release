# Torpedo mission

Detect two torpedo panels with the front camera, pick the better-matching
template, and fire two torpedoes — one from the left shooter, one from
the right. Blind fallback fires both if anything fails.

Mirrored upstream from `mission_planner_2` into
`bluerov_sim/bluerov_sim/torpedo/torpedo.py`.

## Goal

Approach the torpedo panels, yaw-scan with the front camera to find them,
pick a template, align each shooter to its target, fire twice from each.

## Tree overview

`bluerov_sim/bluerov_sim/torpedo/torpedo.py:473–484` —
`Sequence(memory=True)`. Pre-seeds two blackboard keys (`choice_is_fish`,
`/global/base_link`) and then enters a `Selector(memory=True)` of main + fallback.

``` mermaid
flowchart TD
  ROOT["Sequence(memory=True)"]
  ARM[arm_and_set_mode]
  SETBB1["Set choice_is_fish = True"]
  SETBB2["Set /global/base_link"]
  SEL["Selector(memory=True)"]
  MAIN[Sequence: main]
  FB[Sequence: fallback]

  ROOT --> ARM --> SETBB1 --> SETBB2 --> SEL
  SEL --> MAIN
  SEL --> FB
```

### Main sequence

1. Start vision pipeline.
2. Goto torpedo vicinity (`map`, coarse approach between panels).
3. Front-yaw search (±30°, 15° steps).
4. Goto centre (`front_cam_optical` frame).
5. Check point correspondences for both templates.
6. Selector picks the template with more correspondences.
7. `Retry` enable the chosen template.
8. Move-and-shoot: first torpedo (left shooter).
9. Return to centre.
10. Move-and-shoot: second torpedo (right shooter).
11. Stop vision.

## Per-leg detail

### Goto torpedo vicinity (`torpedo.py:205–216`)

`goto.FromConstant`:

- `pose = PoseStamped(frame_id="map", x=-3.0, y=-1.5, yaw=90°)`
- `specified_heading=True`.
- `depth_override_value=SEARCH_DEPTH=-0.5 m` (front camera level with
  panels at `world z=-2`).

### Front-yaw search (`torpedo.py:218–223`, builder `search.py:524`)

`goto.NFromConstant` with yaw-only offsets.

- `is_relative_movement=True` — offsets are body-relative.
- `specified_heading=True`.
- `depth_override_value=SEARCH_DEPTH=-0.5 m`.
- Yaw points: `_gen_yaw_points(max_left=30°, max_right=30°, step=15°)`.
- Clusters `torpedo/yolo → torpedo/yolo/clustered`.

!!! tip "The classic 'yaw without translating' pattern"
    `move_rel=True, depth_rel=False, depth=<absolute>` — vehicle yaws
    while holding an absolute map-z. The three rel-flags are resolved
    independently in the action server.

### Goto centre (`torpedo.py:225–229`)

- `pose = PoseStamped(frame_id="torpedo/centre/view")`
- `anchor_frame_name='front_cam_optical'` — places the camera, not the
  body, on the centre target.
- No depth override; current depth is held.

### Move-and-shoot (`torpedo.py:332` and `:340`, generator
`bluerov_sim/bluerov_sim/torpedo/move_and_shoot_seq.py:95–240`)

For each torpedo:

- Anchor: `torpedo_shooter_left_link` (first) / `torpedo_shooter_right_link`
  (second), set dynamically (`:146–151`).
- Goal frame: fish or shark view, chosen from `choice_is_fish`. First
  torpedo fires at fish if true, shark if false; second is the opposite.
- `distance_threshold=0.025 m` (2.5 cm — very tight).
- `yaw_threshold=1.0°`.
- Cluster torpedo pose twice: once before approach
  (`CLUSTER_DURATION=4 s`), once after (`REALIGN_CLUSTER_DURATION=2 s`).
- Up to `MAX_ALIGN_FAILURE=5` retries.
- Stabilise 2.5 s before firing.
- Fire `SHOOT_REPEATS=2` times per torpedo.

## Search / recovery patterns

| Failure | Recovery |
|---|---|
| Yaw search doesn't see anything | Proceeds to centre anyway — no retry. |
| Template selection ambiguous | Selector picks whichever has more point correspondences. |
| Alignment misses threshold | Re-cluster and retry up to 5 times. |
| Whole main sequence fails | Fallback: fire 2× left, wait 3 s, fire 2× right, wait 3 s (`:417–437`). |

## Cluster_tf integration

| Service | `/bluerov/cluster_tfs_srv` |
|---|---|
| Raw TFs | `torpedo/yolo`, `Task04_Tagging_01_optical`, `Task04_Tagging_02_optical` |
| Clustered | `torpedo/yolo/clustered`, `torpedo_1`, `torpedo_2` |
| View frames | `torpedo/centre/view`, `torpedo_{1,2}/{fish,shark}/view` |
| Shooter anchors | `torpedo_shooter_left_link`, `torpedo_shooter_right_link` |

## Subtle gotchas

!!! warning "Depth sign — ENU"
    `SEARCH_DEPTH=-0.5` is map/ENU. Upstream NED has the opposite sign;
    flip when copying.

!!! warning "Yaw search uses body-relative pose"
    `is_relative_movement=True` (`search.py:551`). Without that flag the
    waypoints would be map-frame headings.

!!! warning "choice_is_fish is hardcoded"
    `torpedo.py:451–455` sets `choice_is_fish=True` rather than reading
    from `choice_server`. First torpedo aims at the fish panel, second
    at shark (`move_and_shoot_seq.py:122–144`). To use the live choice
    server, remove the `SetBlackboardVariable`.

!!! info "Anchor toggle between shooters"
    Each torpedo aligns relative to its own shooter link. Left and right
    are mirror-image frames; their `{fish,shark}/view` children are too.

!!! info "Double-cluster for verification"
    Move-and-shoot clusters before approach **and** after movement, so an
    alignment slip is caught before firing.

!!! info "Namespace isolation"
    All BT keys live under `/bluerov/torpedo` (`:59`) — no collision
    with the bin task if both run.

## See also

- [Primitives](primitives.md) — `goto`, search builders, action server.
- [Bin mission](bin.md) — the same skeleton, different search / anchor
  pattern.
- [image_matching](../packages/image_matching.md) — template toggle
  service used by the selector.
