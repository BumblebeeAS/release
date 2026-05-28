# Mission strategies

Three behaviour-tree missions live in this workspace. They share a common
substrate (`goto`, `cluster_tf`, `shared_trees`) but their strategies differ
significantly. This section walks through each one — what the vehicle
actually does, how it searches when it can't see the target, and what
recovers it when something fails.

| Mission | Goal | Search | Fallback |
|---|---|---|---|
| **[Square](square.md)** | Drive a 2 m × 2 m square in body frame. | None (open-loop). | None — fail-stop on timeout. |
| **[Bin drop](bin.md)** | Detect bin → match template → release three droppers. | Layered downward square. | Fire all three droppers blindly. |
| **[Torpedo](torpedo.md)** | Detect 2 panels → align dropper → fire two torpedoes. | Front-camera yaw scan (±30°). | Fire both torpedoes blindly. |

All three share the same building blocks — `goto`, the `Locomotion` action,
the `cluster_tf` service, and the search primitives. See
**[Primitives](primitives.md)** for the substrate.

!!! info "Read the conventions page first"
    Body-frame setpoints, ENU depth signs, anchor-frame logic, the three
    independent rel-flags — these are detailed in
    [Conventions](../overview/conventions.md). Without them, the BT code
    looks more cryptic than it really is.
