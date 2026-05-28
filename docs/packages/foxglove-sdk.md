# foxglove-sdk

**Role:** Foxglove bridge + MCAP logging. Streams ROS topics over a
WebSocket so Foxglove Studio can visualise them, and records MCAP files for
post-mission analysis.

## Why this needs special handling

`ros-humble-foxglove-bridge` was removed from the Humble package index on
2026-03-20. To get the bridge, you must clone the SDK manually:

```bash
git clone https://github.com/foxglove/foxglove-sdk src/foxglove-sdk
```

`vcs import` doesn't pull it. This is a one-off setup step.

## ROS interfaces

| Component | Behaviour |
|---|---|
| `foxglove_bridge` node | Subscribes to all topics, relays over WebSocket on `localhost:8765`. |
| `foxglove_msgs` | Custom visualization message types (`SceneUpdate`, `PointCloud`, `ImageAnnotations`, …). |

## Connecting

In Foxglove Studio: **Open Connection** → **Rosbridge** →
`ws://localhost:8765`.

Useful panels: `/tf`, `/mavros/local_position/pose`, `/bluerov/odom`,
`/world_objects/markers`, the image topics, and `yolo/*/annotations` for
detections.

## See also

- [Running the sim](../overview/running.md) — Foxglove launch and connection.
- [yolo_ros_trt](yolo_ros_trt.md) — publishes `foxglove_msgs/ImageAnnotations`.
