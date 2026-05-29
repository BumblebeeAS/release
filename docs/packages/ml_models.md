# ml_models

**Role:** Asset repository for pre-trained models. No code. Just weights
and exported engines, installed by `colcon` into `share/ml_models/`.

## Contents

| Directory | What's there |
|---|---|
| `yolov11_segment/` | YOLOv11-small instance-segmentation `.pt` weights (gate, trash, slalom, floor, bins, torpedoes). Timestamped filenames (`yolov11s_<task>_<YYYYMMDD>_<rev>.pt`). |
| `depth_anything/` | Depth Anything v2 ViT-B ONNX models (`depth_anything_v2_vitb_518.onnx`, `_770.onnx`; ~390 MB each). |

## How they're consumed

- [yolo_ros_trt](yolo_ros_trt.md) loads `.engine` files (compiled from
  `.pt` once per host via `Ultralytics.YOLO.export("engine")`).
- The Depth Anything ROS node loads the `.onnx` and JIT-compiles a
  `.engine` on first run.

## Gotchas

!!! warning "Hardcoded paths in configs"
    Vision configs reference these models by absolute path
    (`/workspaces/isaac_ros-dev/src/ml_models/...`). If the workspace is
    not mounted at that location, configs must be updated to match.

!!! warning "Engine vs onnx vs pt"
    The runtime loads `.engine` for YOLO; the engine isn't checked in.
    First-time setup must export `.pt → .engine`. For Depth Anything,
    configs point to `.onnx`.

- No versioning beyond timestamps in filenames; no compression.
