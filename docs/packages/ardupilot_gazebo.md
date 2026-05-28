# ardupilot_gazebo

**Role:** The official ArduPilot Gazebo plugin
(`libArduPilotPlugin.so`). Bridges ArduPilot SITL (ArduSub here) to Gazebo
over a JSON-encoded UDP socket.

This is vendored as a pinned commit via `bluerov_ws.repos`. We don't edit
it — we configure it.

## What it provides

- The plugin binary, loaded inside the BlueROV2 SDF model.
- JSON message format for flexible sensor / actuator exchange between SITL
  and Gazebo.
- Camera models with optional GStreamer streaming.

## ROS interfaces

None — communication with ArduPilot SITL is UDP/JSON, not ROS.

## Gotchas

!!! warning "Required environment"
    - `GZ_VERSION=harmonic` (Garden / Ionic / Jetty also supported, but the
      workspace pins Harmonic).
    - `GZ_SIM_SYSTEM_PLUGIN_PATH` must contain the plugin's install lib.
    - `GZ_SIM_RESOURCE_PATH` must contain the model + world directories.

- SITL frame names need a `gazebo-` prefix (`gazebo-iris`, etc.). The
  BlueROV model has its own definition in `bluerov_sim/models/bluerov2/`.

## See also

- [bluerov_sim](bluerov_sim.md) — loads the plugin inside the BlueROV2 SDF.
- [Architecture](../overview/architecture.md) — where this plugin sits in
  the pipeline.
