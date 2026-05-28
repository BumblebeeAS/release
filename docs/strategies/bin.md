# Bin drop mission

Detect a bin, decide whether the rotated or unrotated template is the
better match, align the dropper, and release three weights. With a blind
backup if everything else fails.

Mirrored upstream from `mission_planner_2` into
`bluerov_sim/bluerov_sim/bins/bins.py`.

## Goal

Navigate to a known coarse bin location (`x=-6, y=4.5` in map frame), find
the bin via YOLO detections + clustering, pick the best template via image
matching, align the dropper, drop weights one by one.

## Tree overview

`bluerov_sim/bluerov_sim/bins/bins.py:670–679` —
`Sequence(memory=True)`. The body is a `Selector(memory=True)` that tries
the main drop sequence first, then a blind fallback.

``` mermaid
flowchart TD
  ROOT["Sequence(memory=True)"]
  ARM[arm_and_set_mode]
  SEL["Selector(memory=True)"]
  MAIN["Sequence: main drop"]
  FB["Sequence: fallback (3× blind drop)"]

  ROOT --> ARM --> SEL
  SEL --> MAIN
  SEL --> FB
```

### Main drop sequence

1. Start vision pipeline.
2. Goto bin vicinity (coarse approach).
3. Layered-square search (4-waypoint sweep, up to `NUM_SQUARES=1` layer).
4. TF-checker for `bin/centre/view`.
5. `find_acute_angle` — reorient detected pose to face the camera head-on.
6. Goto bin centre.
7. `Retry` enable YOLO + collect **unrotated** template points.
8. `Retry` enable rotated detections + collect **rotated** template points.
9. Template selector — pick whichever template has more points.
10. `Retry` enable the chosen template.
11. Switch anchor frame to `dropper_link`.
12. Goto cluster (with threshold checks + up to 4 retries).
13. Stabilise 5 s.
14. Fire dropper #1, wait 3.5 s.
15. Fire dropper #2, wait 0.5 s.
16. Fire dropper #3.
17. Stop vision.

## Per-leg detail

### Goto bin vicinity (`bins.py:182–192`)

`goto.FromConstant` with:

- `pose = PoseStamped(frame_id="map", x=-6.0, y=4.5)`
- `anchor_frame_name='base_link'`
- `specified_heading=False` — heading doesn't matter yet.
- `depth_override_value=SEARCH_DEPTH=-0.3` — ENU; 0.3 m below surface.

### Layered-square search (`bins.py:194–207`)

`goto.NFromConstant` via
`create_search_bot_layered_square_root` (`shared_trees/search.py:326`).

- 4 base waypoints in `base_link`: 1 m fwd/back, 0.5 m left/right.
- `offset_coeff=1.0` for subsequent layers (only one in current config).
- `wait_between_moves=1.0 s`.
- `depth_override_value=SEARCH_DEPTH=-0.3`.
- Search stops when cluster spread `< CLUSTER_DIST_THRESHOLD=0.5 m` **or**
  when `num_squares==1` (auto-pass, `:401`).
- Clusters `bin/yolo → bin/yolo/clustered`.

### Goto bin centre (`bins.py:241–246`)

`goto.FromBlackboard` reading the acute-corrected pose:

- Frame: `BIN_CENTRE_VIEW_FRAME = "bin/centre/view"`.
- `depth_override_value=BIN_DEPTH_OVERRIDE_VALUE=-1.3 m`.

### Goto align target — dropper (`bins.py:437–442`)

`goto.FromBlackboard` reading the selected template's view frame:

- `anchor_frame_name='dropper_link'` — the goal puts the dropper, **not**
  the body centre, on the target.
- `depth_override_value=BIN_DEPTH_OVERRIDE_VALUE=-1.3 m`.
- Clustered 4 times with retries (`:412–434`).
- Final threshold: 0.05 m XYZ (`:463`).

## Search / recovery patterns

| Failure | Recovery |
|---|---|
| Cluster too loose on layer 1 | Outer `Selector` keeps retrying layer 1 (or moves to layer 2 if configured). |
| Unrotated template doesn't match | `Retry`(100) on the toggle / wait pair. |
| Rotated template also fails | `Retry`(100) on the rotated toggle / wait. |
| Goto cluster fails | Re-cluster and retry up to `RETRIES=4` (`:453–466`). |
| Whole main sequence fails | Selector falls through to blind drop: fire three droppers without alignment (`:617–657`). |

## Cluster_tf integration

| Service | `/bluerov/cluster_tfs_srv` |
|---|---|
| TF inputs | `bin/yolo`, `Task03_DropBRUVS_optical`, `Task03_DropBRUVS_Rotated_optical` |
| TF outputs | `bin/yolo/clustered`, `bin/centre/view`, plus fish/shark view frames |

The TF-checker leg waits for `base_link → bin/centre/view` to be
broadcastable before proceeding (`:220–231`).

## Subtle gotchas

!!! warning "Depth sign — ENU"
    `SEARCH_DEPTH=-0.3` and `BIN_DEPTH_OVERRIDE_VALUE=-1.3` are
    map/ENU (negative below surface). When mirroring from AUV4's NED,
    flip the sign.

!!! warning "Anchor frame swap"
    Approach uses `base_link`, final alignment uses `dropper_link`. The
    same `FromBlackboard` machinery with a different anchor shifts the
    commanded pose to put the drop mechanism on the target.

!!! info "Acute-angle correction"
    `find_acute_angle` (imported from upstream `mission_planner_2`,
    `bins.py:29`) reorients the detected bin pose so the camera approaches
    it straight-on rather than at an oblique angle (`:237`).

!!! info "Blackboard namespace"
    All keys are namespaced under `/bluerov/bins` (`:60`) to keep parallel
    tasks (like torpedo) from clobbering shared keys.

!!! info "Choice not hardcoded"
    Unlike the torpedo BT, the bin BT does **not** hardcode
    `/global/choice_is_fish`; it expects upstream `choice_server` to
    populate it. If a choice-related key is missing, that's where to look.

## See also

- [Primitives](primitives.md) — `goto`, search builders, action server.
- [pose_estimator](../packages/pose_estimator.md) — what broadcasts the
  `bin/*` TFs.
- [image_matching](../packages/image_matching.md) — template toggle service.
