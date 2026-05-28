# yolo_ros_trt

**Role:** ROS 2 wrapper for YOLOv11 instance segmentation, running on
TensorRT engines.

## Key entrypoints

| File | Purpose |
|---|---|
| `yolo_ros_trt/yolo_node.py` (`main`) | `LifecycleNode` — loads `.engine`, subscribes to image, publishes detections + annotations. |
| `yolo_ros_trt/tracking_node.py` (`main`) | Optional Ultralytics-tracker wrapper on top of detections. |
| `yolo_ros_trt/utils/yolo_node_helper.py` | Conversion helpers to `yolo_msgs/DetectionArray`. |

## ROS interfaces — `yolo_node`

| Direction | Topic | Type |
|---|---|---|
| Sub | `input_image_topic` (default `image`) | `sensor_msgs/Image` |
| Pub | `output_detections_topic` (default `yolo/detections`) | `yolo_msgs/DetectionArray` |
| Pub | `output_annotations_topic` (default `yolo/annotations`) | `foxglove_msgs/ImageAnnotations` |

Parameters: `model_path` (required `.engine`), `conf`, `iou`,
`agnostic_nms`, image / detection / annotation topic names, `font_size`,
`display_tracker_id`.

## Models used

TensorRT engines from [ml_models](ml_models.md), e.g.

- `yolov11s_gate_20250807_0.engine`
- `yolov11s_trash_20250816_0.engine`
- `yolov11s_slalom_20250809_0.engine`

Models are stored in `.pt` form and exported to `.engine` once per host.

## Gotchas

!!! warning "Lifecycle"
    Starts in `Inactive`. Doesn't publish until `activate` is called.
    Confirm with `ros2 lifecycle list /yolo_node`.

!!! warning "Encoding is BGR8"
    Inherited from OpenCV / `cv_bridge`. Anything that consumes annotations
    should expect BGR.

!!! warning "Tunables require reload"
    `conf` / `iou` apply at NMS time. Changes need a deactivate → activate
    cycle (or `.engine` reload) to take effect.

- No tracking is built into `yolo_node`. Use `tracking_node` if you need it.
- Engine generation: `Ultralytics.YOLO.export("engine")` from a `.pt`.

## See also

- [ml_models](ml_models.md) — engine files.
- [pose_estimator](pose_estimator.md) — consumes `yolo/detections`.
- [filters](filters.md) — post-processes detections.
