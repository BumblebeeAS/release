# Bin

The bin task is a RoboSub staple: a square bin sits face-up on the
pool floor with a picture (a "template") inside it. The vehicle has
to find the bin, identify which of two pictures the bin is showing,
align a dropper mechanism over it, and release weights into it. Each
weight that lands in the bin scores points; landing the right weights
(matching the picture) scores more.

This mission lives in `bluerov_sim/bluerov_sim/bins/bins.py`, with
template-selection helpers in
`bluerov_sim/bluerov_sim/bins/template_selector.py`. The search
builders are shared with the rest of the BlueROV missions through
`bluerov_sim/bluerov_sim/shared_trees/`.

!!! tip "Read these first"
    - [Concepts](../overview/concepts.md): for the vocabulary
      (cluster_tf, anchor frames, sim time).
    - [Primitives](primitives.md): for `goto`, the action server,
      and the search builders this tree composes.
    - [Square mission](square.md): for the simplest possible
      `Sequence(memory=True) + goto` example.

## Goal

Navigate to a known coarse bin location (`x=-6, y=4.5` in `map`,
roughly above where the bin tends to sit in the RoboSub-25 pool), find
the bin via YOLO detections plus clustering, pick the better of two
template matches, switch the goal's anchor frame to the dropper, drop
three weights in sequence. If anything in that chain fails, fall back
to firing all three droppers blindly from the coarse approach pose.

## Tree overview

`bluerov_sim/bluerov_sim/bins/bins.py:670–679`. A top-level
`Sequence(memory=True)` whose interesting child is a
`Selector(memory=True)`. The selector tries the **main drop sequence**
first; if any leg of that fails, it falls through to the **blind
fallback**.

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

!!! info "Why a selector at the top?"
    Because the worst RoboSub failure mode is "perception didn't
    converge, mission halted, scored zero on the task". A `Selector`
    here means "try the smart plan, but if anything goes sideways,
    fire the droppers anyway". It's literally a sibling, not a
    bolted-on error handler.

### Main drop sequence. The 17 steps in plain English

1. **Bring up the vision pipeline.** Lifecycle-manager configures and
   activates the YOLO node so it starts producing detections.
2. **Drive to the bin vicinity.** Coarse `map`-frame waypoint just above
   where the bin tends to live.
3. **Layered-square search.** A 4-waypoint downward square scan, while
   YOLO is detecting and `cluster_tf` is averaging detections into a
   stable `bin/yolo/clustered` frame.
4. **Wait for the centre-view TF to appear.** `find_acute_angle`
   (helper) reorients the clustered pose so the camera approaches the
   bin straight-on, not at an oblique angle. It publishes a derived
   `bin/centre/view` TF.
5. **Drive over the bin centre.** Position the body above
   `bin/centre/view`.
6. **Collect template-1 ("unrotated") point correspondences.** Toggle
   `simple_matcher_node` to the unrotated template, wait for points,
   record how many it produced.
7. **Collect template-2 ("rotated") point correspondences.** Same, for
   the 90°-rotated template.
8. **Pick the better template.** Whichever produced more
   correspondences wins.
9. **Re-enable the winning template** so its TF keeps streaming during
   final alignment.
10. **Swap the anchor frame to `dropper_link`.** Now "drive to the
    target" means "put the dropper on the target", not the body
    centre.
11. **Goto the cluster** with tight thresholds (5 cm position) and up
    to 4 retries on cluster spread.
12. **Stabilise for 5 s** so the vehicle settles before actuating.
13. **Fire dropper #1**, wait 3.5 s.
14. **Fire dropper #2**, wait 0.5 s.
15. **Fire dropper #3.**
16. **Stop the vision pipeline.**

If anything in that chain fails, the outer selector falls through
to step 17:

17. **Blind fallback:** fire all three droppers from the vicinity
    pose, accepting reduced score over scoring zero.

## Per-leg detail

The line numbers below refer to `bins.py` unless noted otherwise.

### Goto bin vicinity (`bins.py:182–192`)

`goto.FromConstant` with:

- `pose = PoseStamped(frame_id="map", x=-6.0, y=4.5)`
- `anchor_frame_name='base_link'`. Body centre to the goal, since we
  just want to get into the rough area.
- `specified_heading=False`. Heading doesn't matter yet; we just need
  to be near the bin.
- `depth_override_value=SEARCH_DEPTH=-0.3`. ENU, 0.3 m below the
  surface. Shallow so the bottom camera has a wide footprint.

!!! info "Why `specified_heading=False`?"
    Heading-gated goals add yaw error to the success condition.
    During the coarse approach the bin might not even be in frame yet,
    there's no "right" heading to aim for. Search will sort the
    heading out next.

### Layered-square search (`bins.py:194–207`)

`goto.NFromConstant` via
`create_search_bot_layered_square_root` ([search.py:326](primitives.md#shared_treessearchpy-reusable-search-behaviours)):

- 4 base waypoints in `base_link`: 1 m forward / 1 m back, 0.5 m left /
  0.5 m right. So one "square" is a small box-pattern over the
  vicinity pose.
- `offset_coeff=1.0` for subsequent layers (only one layer in current
  config, so this only matters if you bump `NUM_SQUARES`).
- `wait_between_moves=1.0 s`.
- `depth_override_value=SEARCH_DEPTH=-0.3`.
- Search short-circuits when the cluster spread drops below
  `CLUSTER_DIST_THRESHOLD=0.5 m` *or* when all `num_squares` layers
  are exhausted (`:401`).
- Source TF for clustering: `bin/yolo` → produces clustered
  `bin/yolo/clustered`.

!!! info "What 'cluster spread' means"
    `cluster_tf` collects samples for `cluster_duration` seconds, runs
    HDBSCAN on the positions, and computes the maximum distance
    between inliers (the "spread"). Low spread → detections are
    landing on roughly the same place → we believe in the cluster.
    High spread → detections are scattered → don't trust it yet,
    re-search.

### Acute-angle correction (`bins.py:29`, `:237`)

`find_acute_angle` reads the clustered bin TF and produces a derived
`bin/centre/view` TF whose orientation is "rotated so the bin's nearest
edge is parallel to the camera's view direction". Approaching at an
oblique angle would leave the dropper at a yaw offset and weights would
land outside the bin.

### Goto bin centre (`bins.py:241–246`)

`goto.FromBlackboard` reading the acute-corrected pose:

- Frame: `BIN_CENTRE_VIEW_FRAME = "bin/centre/view"`
- `depth_override_value=BIN_DEPTH_OVERRIDE_VALUE=-1.3 m`. Descend
  closer to the bin so the bottom camera resolves the template.
- Anchor still `base_link`. We're getting the body roughly over the
  centre, not aligning the dropper yet.

### Template selection (`bins.py:259–360`)

The whole template-selector subtree is in
`bluerov_sim/bluerov_sim/bins/template_selector.py`. Two parallel
attempts (with `Retry(100)` decorators) call
`simple_matcher_node`'s `toggle_template` service, wait for
`PointCorrespondencesStamped` messages, and write the count to the
blackboard. A final behaviour compares the counts and writes the
winning template's `Task03_DropBRUVS_optical` (or `…_Rotated_optical`)
frame to the alignment-pose blackboard key.

!!! info "Why both templates?"
    The picture inside the bin can be one of two designs (and can be
    rotated). The picture you actually drop on scores more. Trying
    both lets the vehicle pick the better-matched one rather than
    committing prematurely.

### Goto align target. Dropper (`bins.py:437–442`)

`goto.FromBlackboard` reading the selected template's view frame:

- `anchor_frame_name='dropper_link'`: **this is the anchor swap**.
  The goal now puts the dropper (not the body centre) on the
  alignment target. The conversion service handles the offset.
- `depth_override_value=BIN_DEPTH_OVERRIDE_VALUE=-1.3 m`.
- Final position threshold: 0.05 m XYZ (`:463`): much tighter than
  the 0.2 m default. Necessary because a 20 cm dropper miss would
  drop the weight outside the bin.
- Wrapped in a retry/recluster decorator: if the cluster goes stale,
  recompute it and try again, up to `RETRIES=4` (`:412–434`,
  `:453–466`).

!!! tip "When to swap the anchor frame"
    Any time the body centre is not the thing you want on the target.
    Bin uses `dropper_link`; torpedo uses
    `torpedo_shooter_{left,right}_link`; the goto-centre-with-camera
    legs use `front_cam_optical`. The anchor swap is one parameter to
    `goto.FromBlackboard`. Never hand-rolled.

### Firing the droppers (`bins.py:550–600`)

Three calls to the gripper command service, each followed by a
`Wait` (3.5 s, 0.5 s, 0 s). The first wait is longer because the
vehicle needs to settle after the first drop kicks it around.

## Search / recovery patterns

| Failure                                                    | Recovery                                                                                       |
| ---------------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| Cluster spread too loose on layer 1                        | The outer search-`Selector` keeps re-clustering, or moves to layer 2 if `NUM_SQUARES > 1`.     |
| Unrotated template doesn't match                           | `Retry(100)` on the toggle + wait pair.                                                        |
| Rotated template also doesn't match                        | `Retry(100)` on the rotated pair. (Both retries can give up. Then the selector falls through.) |
| Goto-cluster fails (vehicle blew past, cluster went stale) | Re-cluster and retry up to `RETRIES=4` (`:453–466`).                                           |
| Whole main sequence fails                                  | Outer selector falls to the blind drop: fire three droppers without alignment (`:617–657`).    |

## Cluster_tf integration

| Service    | `/bluerov/cluster_tfs_srv`                                                 |
| ---------- | -------------------------------------------------------------------------- |
| TF inputs  | `bin/yolo`, `Task03_DropBRUVS_optical`, `Task03_DropBRUVS_Rotated_optical` |
| TF outputs | `bin/yolo/clustered`, `bin/centre/view`, plus the per-template view frames |

The TF-checker leg (`:220–231`) waits for `base_link → bin/centre/view`
to be broadcastable before proceeding. If it never appears. Because
no template ever matched. The leg times out and we fall to the
blind drop.

### Who broadcasts the source TFs

`cluster_tf` only reads the TF tree. It never subscribes to a pose
topic. Each input TF needs a broadcaster running and publishing the
frame before cluster_tf can collect samples. (See [Concepts:
cluster_tf](../overview/concepts.md#cluster_tf-what-does-clustering-actually-do)
for a refresher.)

| Source TF                                                         | Broadcaster (node)                                                                                       | Package                                         |
| ----------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- | ----------------------------------------------- |
| `bin/yolo`                                                        | `bin_pose_estimator_node` (`PoseEstimatorTransformPubNode`)                                              | [pose_estimator](../packages/pose_estimator.md) |
| `Task03_DropBRUVS_optical` and `Task03_DropBRUVS_Rotated_optical` | `points_pose_estimator_node`. Consumes `image_matching/point_correspondences` from `simple_matcher_node` | [pose_estimator](../packages/pose_estimator.md) |

All broadcasters are launched by `bluerov_bin_vision.launch.py`.
`simple_matcher_node` only starts emitting point correspondences
once the BT toggles a template via
`/bluerov/bin/image_matching/toggle_template`, so the
`Task03_DropBRUVS_*` TFs first appear after that service call.

## How to run it

```bash
tmuxp load bluerov_bin_mission.yaml
```

That brings up five panes (`sim`, `controls`, `cluster`, `vision`,
`bt`). Each pane runs one launch file; you can `Ctrl-C` any single
pane and restart it without affecting the others. The `cluster`
pane is noisy until the vision pipeline starts producing detections;
that's expected, not a bug.

## How to tell whether it worked

In Foxglove:

- The vehicle should descend to roughly `z = -1.3` and hover over the
  bin.
- The TF tree should show `bin/yolo/clustered`, `bin/centre/view`,
  and one of the two `Task03_DropBRUVS_*_optical` frames appearing.
- Three weights should detach from the dropper and fall into the bin.

A successful run ends with the BT root returning `SUCCESS` and the BT
pane logging `Sequence: main drop` complete. A blind-fallback run
also returns `SUCCESS` but you'll see "fallback" in the logs. That
means perception didn't converge and you should look at the vision
panes to see why.

## Subtle gotchas

!!! warning "Depth sign. ENU"
    `SEARCH_DEPTH=-0.3` and `BIN_DEPTH_OVERRIDE_VALUE=-1.3` are map/ENU
    (negative below surface).

!!! warning "Anchor-frame swap is the magic"
    Approach uses `base_link`, final alignment uses `dropper_link`.
    The same `FromBlackboard` call with a different anchor parameter
    shifts the commanded pose so the drop mechanism, not the body
    centre, lands on the target. Without the swap the weights would
    drop ~30 cm in front of the bin.

!!! warning "Tight thresholds for the final goto"
    The final alignment leg uses 0.05 m position threshold (vs 0.2 m
    default). If you loosen this you'll start missing the bin even
    when perception is fine. If you tighten it further, the alignment
    might never close and you'll always end up in the fallback.

!!! info "Acute-angle correction"
    `find_acute_angle` (`bins.py:29`, `:237`) reorients the detected
    bin pose so the camera approaches it straight-on rather than at
    an oblique angle. Without it the dropper would slide in at a yaw
    offset and weights would tumble outside the bin.

!!! info "Blackboard namespace"
    All keys are namespaced under `/bluerov/bins` (`:60`) so a parallel
    torpedo run can't clobber shared keys. Two missions can therefore
    coexist on the blackboard as long as their namespaces differ.

!!! info "Choice not hardcoded"
    Unlike the torpedo BT, the bin BT does **not** hardcode
    `/global/choice_is_fish`; it expects an upstream `choice_server` to
    populate it. If a choice-related key is missing on a clean run,
    that's the place to look.

## Where to dig deeper

- [Primitives](primitives.md). `goto`, the search builders, the
  action server.
- [pose_estimator](../packages/pose_estimator.md): what broadcasts
  the `bin/*` TFs and how `PoseEstimatorTransformPubNode` works.
- [image_matching](../packages/image_matching.md): the template
  toggle service.
- [filters](../packages/filters.md): how `cluster_tf` accumulates
  noisy TFs into a stable one.
- [Conventions](../overview/conventions.md): the FLU vs ENU vs NED
  rules that you'll hit the moment you change a goal pose.
- [Torpedo mission](torpedo.md): the same skeleton with a different
  search pattern and a per-panel anchor swap.
