# image_processing

**Role:** Lightweight image transforms — brightness/sharpening, depth
visualisation, header restamping. Small utility nodes wired in front of
perception when the raw camera stream needs adjusting.

## Nodes

| Node | Behaviour |
|---|---|
| `image_brighten_node` | Scales brightness, then applies a fixed `[[0,-2,0],[-2,9,-2],[0,-2,0]]` sharpening kernel. |
| `depth_to_rgb_node` | Colourmap visualisation of a `32FC1` depth map. |
| `restamp_camera_node` | Rewrites `Image` / `CameraInfo` header timestamps (useful for sim-time replay). |

## ROS interfaces — `image_brighten_node`

| Direction | Topic | Type |
|---|---|---|
| Sub | `input_image_topic` (default `image`) | `sensor_msgs/Image` |
| Pub | `output_image_topic` (default `brighten/image`) | `sensor_msgs/Image` |

Parameters: `brightness_factor` (default `1.2`).

## Utilities

| Module | Purpose |
|---|---|
| `image_processing/utils/image_annotations.py` | `get_image_annotations()` builds `foxglove_msgs/ImageAnnotations` from polygons / points. |
| `image_processing/utils/ros_np_multiarray.py` | Bidirectional conversion between `std_msgs/MultiArrayLayout` and numpy. |

## Gotchas

- **Output is BGR8** regardless of input encoding.
- The sharpening kernel is hardcoded; no parameter exposes it.
- Headers are passed through unchanged in `image_brighten_node`.
  Restamping needs `restamp_camera_node`.
