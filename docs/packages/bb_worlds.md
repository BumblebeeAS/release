# bb_worlds

**Role:** Gazebo world definitions (SDF) and launch wrappers for competition
scenarios.

## Worlds shipped

- RoboSub 2023 and 2025 pools
- SAUVC 2024
- `dave_ocean_waves` (open-water with waves)
- `robotx_2026_sg_river`

## Key files

| File | Purpose |
|---|---|
| `worlds/*.sdf` | Gazebo world files. Each declares `spherical_coordinates` which `bluerov_sim.launch.py` parses to set ArduSub home lat/lon. |
| `launch/*.launch.py` | Per-world launchers (wrap `ros_gz_bridge` for topic bridging). |

The package installs hooks that set `GZ_SIM_RESOURCE_PATH` so worlds and
models resolve correctly.

## ROS interfaces

None directly — this package only provides SDF + launch glue.

## Gotchas

- World SDFs reference models from `dave` and others. If a model fails to
  load, check `GZ_SIM_RESOURCE_PATH`.
- The RobotX worlds reference ArUco markers and course gates; their
  positions live inside the SDF.
- World names must match `GZ_SIM_RESOURCE_PATH` lookup, or
  `bluerov_sim.launch.py` falls back to default home coordinates.

## See also

- [bluerov_sim](bluerov_sim.md) — passes `world_name` to the world launch.
- [Running the sim](../overview/running.md) — concrete launch invocations.
