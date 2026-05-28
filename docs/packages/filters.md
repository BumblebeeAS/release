# filters

**Role:** Post-processing for detections — 3D pose filtering, 2D
projection cleanup, and SORT-based tracking. Sits between the
[pose_estimator](pose_estimator.md) output and the BT.

## Key entrypoints

| File | Purpose |
|---|---|
| `scripts/detected_object_3d_filter.py` | 3D detection filter / clusterer. |
| `scripts/detected_object_2d_filter_projection.py` | 2D projection filtering. |
| `bb_filters/sort_3d.py` | SORT in 3D. |
| `launch/asv4_filters.launch.py` | Wires the filter pipeline up. |

## ROS interfaces

| Direction | Topics |
|---|---|
| Sub | Detection topics from the perception pipeline (`DetectedObject*` msgs). |
| Pub | Filtered / tracked detections with consistent IDs. |

Parameters cover clustering tolerances and the time-alignment window for
multi-detection fusion.

## Notes

- **Pose clustering, not TF clustering.** This package clusters
  `geometry_msgs/Pose` for efficiency; the separate `cluster_tf` (used by
  the BTs) does the same idea but on TFs.
- `MultiThreadedExecutor` is **not** recommended here — single-threaded
  performs better in practice.
- The subscriber-time-alignment path has a small TOCTOU window; if you
  see flaky cross-topic syncs, that's the place to look.
