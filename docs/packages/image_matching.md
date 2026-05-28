# image_matching

**Role:** XFeat-based template matching. For planar targets (torpedo
panels, bin templates) it extracts learned features from the current image
and matches them against a registered template, emitting 2D ↔ 3D point
correspondences that `points_pose_estimator_node` turns into a TF.

## Key entrypoints

| File | Purpose |
|---|---|
| `image_matching/simple_matcher_node.py` | Main ROS node. Holds the active template, runs XFeat, publishes correspondences. |
| `image_matching/detector.py` (`main`) | Standalone feature detector. |
| `src/feature_matcher/xfeat.py:XFeatMatcher` | Backend wrapping the XFeat model. |
| `share/image_matching/templates/robosub25/templates.json` | Per-template image + 3D dimensions / offsets. |

## ROS interfaces

| Direction | Name | Type |
|---|---|---|
| Sub | `input_image_topic` (default `image`) | `sensor_msgs/Image` |
| Pub | `image_matching/point_correspondences` | `bb_perception_msgs/PointCorrespondencesStamped` |
| Pub | `output_annotations_topic` | `foxglove_msgs/ImageAnnotations` |
| Service | `image_matching/toggle_template` | `bb_perception_msgs/IMPoseEstimatorToggleTemplate` |

Toggle request fields: `template_name` (e.g. `Task04_Tagging_01.png`),
`enable` (bool).

## How the BT uses it

The torpedo and bin BTs sit on a `Retry` that calls `toggle_template` to
enable a template, then waits for a point-correspondences message to
land. Both templates (rotated / unrotated for bin, panel 1 / panel 2 for
torpedo) are sampled; a selector picks the one with more correspondences.

## Gotchas

!!! warning "Templates start disabled"
    `simple_matcher_node` boots with `template_name=None`. Until a
    `toggle_template` call enables one, no correspondences are published.

- Object frame ID is derived from the template filename stem + `_optical`
  (e.g. `Task04_Tagging_01_optical`).
- XFeat favours roughly upright cameras — large pitch/roll can degrade
  matching.
- Internal homography is only used for warping template corners during
  annotation; it does not gate publishing.
- Image is decoded as **BGR8**.

## See also

- [pose_estimator](pose_estimator.md) — `points_pose_estimator_node`
  consumes the published correspondences.
- [Bin mission](../strategies/bin.md), [Torpedo mission](../strategies/torpedo.md)
  — concrete BT usage.
