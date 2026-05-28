# pose_estimator

**Role:** Turns 2D detections (YOLO masks, image-matching keypoints) into
3D poses via PnP + RANSAC, and **broadcasts the result as a TF**. The TF
broadcast is what `cluster_tf` consumes — not pose topics.

## Per-task estimator nodes

Each RoboSub task gets its own estimator:

| Node | Source 2D data | Output frame example |
|---|---|---|
| `gate_pose_estimator_node` | YOLO polyline keypoints | `gate/front` |
| `points_pose_estimator_node` | Image-matching point correspondences | template-specific |
| `slalom_pose_estimator_node` | YOLO + depth | per-pole |
| `bin_pose_estimator_node` | YOLO | `bin/yolo`, `bin/centre/view`, … |
| `torpedo_pose_estimator_node` | YOLO + correspondences | `torpedo/yolo`, `torpedo_1/{fish,shark}/view`, … |
| `trash_pose_estimator_*.py` | YOLO | per-object |

## Base class

All inherit from `utils/pose_estimator_node.py:PoseEstimatorTransformPubNode`,
which provides the `tf2_ros.TransformBroadcaster` plumbing.

!!! danger "Don't use PoseEstimatorPosePubNode for tasks that feed cluster_tf"
    `cluster_tf` only consumes **TFs**. If you inherit from
    `PoseEstimatorPosePubNode` (pose-topic publisher) instead of
    `PoseEstimatorTransformPubNode`, `cluster_tf` collects zero samples
    and the BT stalls in the search leg.

## ROS interfaces (typical)

| Direction | Topic / TF | Type |
|---|---|---|
| Sub | `input_detections_topic` (default `yolo/detections`) | `yolo_msgs/DetectionArray` |
| Sub | `camera_info_topic` | `sensor_msgs/CameraInfo` (required at startup) |
| Broadcast | TF: `object_frame_id` under camera optical frame | `geometry_msgs/TransformStamped` |
| Pub (optional) | `{object_frame_id}/pose` | `PoseWithCovarianceStamped` |

Parameters: `object_frame_id`, `input_detections_topic`, `camera_info_topic`,
`from_front`, `is_image_rectified`.

## Algorithm

1. Match detection keypoints to 3D object points (from
   `config/object_points.py`).
2. `cv2.solvePnPRansac` for an initial pose.
3. Refine on RANSAC inliers.
4. Optional homography filter (planar targets like gates).
5. Broadcast TF; optionally publish pose with covariance.

## Gotchas

- **Frame IDs:** the broadcast TF's parent is inferred from `camera_info`;
  child is the `object_frame_id` parameter. Both must match what
  `cluster_tf` is configured to lookup.
- **At least 3 keypoints** are required per detection. The gate estimator
  drops detections with fewer.
- **Image encoding:** BGR8 by inheritance from the YOLO chain.

## See also

- [yolo_ros_trt](yolo_ros_trt.md) — upstream detection source.
- [image_matching](image_matching.md) — keypoint source for templated
  tasks.
- [Conventions: vision → cluster_tf integration](../overview/conventions.md#vision-cluster_tf-integration).
