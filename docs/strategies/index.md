# Mission strategies

This workspace ships **three** behaviour-tree missions. They share the
same control substrate (`goto`, `cluster_tf`, the search builders in
`shared_trees/`) but they exist at very different levels of
sophistication.

## Read these before the per-mission pages

- **[Concepts](../overview/concepts.md)**: if "behaviour tree", "TF",
  "anchor frame", or "cluster_tf" are not yet vocabulary, start here.
- **[Architecture](../overview/architecture.md)**: the layered
  pipeline every mission rides on top of.
- **[Conventions](../overview/conventions.md)**: frame, sign and
  naming rules. Mission code is much less cryptic once you've internalised
  these.
- **[Primitives](primitives.md)**: `goto`, `LocomotionActionServer`,
  the search builders. Every mission page below uses these as
  vocabulary.

## The three missions at a glance

| Mission | What the robot is trying to do | Search behaviour | Fallback if everything fails |
|---|---|---|---|
| **[Square](square.md)** | Drive a 2 m × 2 m closed square in body frame. No perception. | None (open-loop). | None. Fail-stop on action timeout. |
| **[Bin drop](bin.md)** | Find a bin on the pool floor, identify which of two templates it matches, line up the dropper, release three weights into it. | Layered downward "square scan" until enough YOLO detections cluster tightly. | Fire all three droppers blindly from the coarse approach pose. |
| **[Torpedo](torpedo.md)** | Find two torpedo panels on the pool wall, pick the correct hole on each (fish vs shark), align each shooter to its hole, fire two torpedoes. | Front-camera yaw scan (±30°, 15° steps). | Fire both torpedoes blindly. |

## Choosing what to read first

- **Brand-new to robotics or BTs?** Start with [Square](square.md). It's
  intentionally the simplest mission. No perception, no recovery, no
  cluster_tf, no anchor-frame trickery. If you can read square.py
  end-to-end and follow what each leg does, you can read every other
  mission too.
- **Comfortable with ROS but new to mission BTs?** Read
  [Primitives](primitives.md) next, then any mission page. The
  primitives are the vocabulary the missions are written in.
- **Already shipped a RoboSub mission once?** Skim Square, then jump to
  [Bin](bin.md) or [Torpedo](torpedo.md) for the actually-interesting
  patterns (cluster_tf integration, anchor-frame swaps, template
  selection, blind fallbacks).

## What all three missions have in common

```
Arm + GUIDED mode
    ↓
Drive to a "vicinity" pose. Coarse approach in the map frame
    ↓
Search if perception isn't seeing the target yet
    ↓
Cluster the noisy detections into a stable TF
    ↓
Switch anchor frame to whichever part of the robot needs to land on the target
    ↓
Final approach. Tight thresholds, retries on failure
    ↓
Actuate (drop / fire / nothing)
    ↓
(if main sequence fails anywhere). Blind fallback so we still score points
```

The bin and torpedo pages walk this structure in detail. The square
mission is a smoke test that runs only the first two and last two of
those phases (no perception, no actuation), giving you a clean way to
verify the whole control pipeline before adding vision on top.

!!! tip "Every mission ends with a fallback"
    The competition rule is unforgiving: a perception failure that
    drops your score to zero is worse than a half-blind drop that
    still scores a few points. Both bin and torpedo wrap their main
    sequence in a `Selector(memory=True)` whose right child is a
    "fire-blindly" sequence. When you read the trees, the
    fallback is a literal sibling of the main plan, not an
    error-handler bolted on.
