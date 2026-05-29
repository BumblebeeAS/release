# dave

**Role:** Underwater simulation ecosystem (DAVE = "Dynamic Aquatic Vehicle
Environment"). Vendored from the field-robotics-lab/dave upstream and
provides protobuf↔ROS bridges plus a handful of Gazebo plugins we depend
on.

## Sub-packages used here

| Sub-package | What we use it for |
|---|---|
| `dave_interfaces` | DVL message types (consumed by `dvl_to_mavros.py`). |
| `dave_ros_gz_plugins` | Protobuf ↔ ROS bridges for sensor plugins. |
| `dave_gz_*` | Underwater camera, sonar, environmental sensors. The `UnderwaterCamera` sensor is the most relevant for us. |

## Gotchas

!!! warning "UnderwaterCamera plugin caveats"
    If you switch the BlueROV2 to DAVE's `UnderwaterCamera` (alternative to
    the standard Gazebo RGB camera):

    - The plugin needs `gz-sim-rgbd-camera-system` loaded in the world SDF.
    - Output is **BGR8**, not RGB8. Consuming nodes must decode accordingly.
    - There is **no** `CameraInfo` topic. Pose-estimator code needs to be
      told the intrinsics another way.
    - `dave_gz_sensor_plugins`' install lib must be on
      `GZ_SIM_SYSTEM_PLUGIN_PATH`.

- Requires `protoc` for plugin builds.
- Gazebo-version-coupled (Harmonic in this workspace).

## See also

- [bb_worlds](bb_worlds.md). `dave_ocean_waves` is a DAVE-provided world.
- [Conventions: DAVE underwater camera plugin](../overview/conventions.md#dave-underwater-camera-plugin)
  for the BGR/CameraInfo caveats in one place.
