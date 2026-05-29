# Running the sim

This page covers everything from "I just cloned the repo" to "the
robot is moving in Gazebo". If a step's purpose isn't obvious, the
[Concepts](concepts.md) and [Architecture](architecture.md) pages
explain *why*.

!!! info "Everything runs inside Docker"
    The host does **not** need ROS or Gazebo installed. The workspace
    mounts at `/root/HOST/bluerov_ws` inside the container, so file
    edits on the host show up immediately inside.

## First-time setup (on the host)

```bash
# Build the docker image. Tags it as `bluerov_ws:humble`.
./build.bash

# Launch a containerised shell. Uses rocker to set up:
#   --nvidia    GPU access for Gazebo + YOLO
#   --x11       so Gazebo can pop up a window on your screen
#   --network=host
./run.bash bluerov_ws:humble
```

!!! info "Why `--network=host`?"
    Two services need it:
    
    - **MAVROS** broadcasts MAVLink on UDP 14550 so a copy of
      **QGroundControl** running on the host picks it up
      automatically.
    - The **Foxglove bridge** listens on TCP 8765 so Foxglove Studio
      on the host can connect.
    
    Without host networking you'd have to bridge each port by hand and
    QGC wouldn't auto-discover the vehicle.

## First-time setup (inside the container)

```bash
# Import external packages pinned in bluerov_ws.repos.
# --recursive is required because image_matching vendors a git submodule
# (BumblebeeAS/accelerated_features for the XFeat model).
vcs import --recursive src < bluerov_ws.repos

# Build the workspace.
source /opt/ros/humble/setup.bash
colcon build --symlink-install
source install/setup.bash
```

!!! tip "Always `--symlink-install`"
    `--symlink-install` symlinks Python files instead of copying them
    into `install/`. After a Python edit, you can re-`source
    install/setup.bash` (or just re-launch the node): no rebuild
    needed. C++ changes still require a rebuild.

### Incremental rebuilds

After editing a single package, build only that one:

```bash
colcon build --packages-select bluerov_sim
source install/setup.bash
```

For a build-type change (e.g. a package switching from `ament_cmake`
to `ament_python`), wipe the per-package build / install dirs first
or `colcon` will leave stale artefacts:

```bash
rm -rf build/<pkg> install/<pkg>
colcon build --symlink-install --packages-select <pkg>
```

There is **no separate lint or test step** configured. `colcon
build` already runs the compile-time unit tests in `frames/`. If you
want to run any package-specific pytest suites, drop into
`build/<pkg>/` and run `pytest` directly.

## Launching a mission

Three `tmuxp` configurations, one per mission. Pick one:

```bash
tmuxp load bluerov_mission.yaml            # legacy square / sm waypoint
tmuxp load bluerov_bin_mission.yaml        # bin drop (RoboSub-2025 pool)
tmuxp load bluerov_torpedo_mission.yaml    # torpedo launch (RoboSub-2025 pool)
```

The bin and torpedo profiles follow a 5-pane split. One launch file
per layer of the stack:

| Pane | Launch file | What it runs |
|---|---|---|
| `sim` | `bluerov_<task>_sim.launch.py` | Gazebo + ArduSub SITL + MAVROS + Foxglove |
| `controls` | `bluerov_<task>_controls.launch.py` | `LocomotionActionServer` + the frames `ConvertToControlsPose` service |
| `cluster` | `bluerov_<task>_cluster.launch.py` | `cluster_tf` action + service servers |
| `vision` | `bluerov_<task>_vision.launch.py` | YOLO detectors + pose estimators + image matching |
| `bt` | `bluerov_<task>_bt.launch.py` | The mission py_trees tree |

Each layer is its own launch file, so any single pane can be `Ctrl-C`'d
and restarted without bouncing the rest. That's the iteration loop
when developing. Restart the BT pane to try a different mission
behaviour, restart the vision pane to swap a perception node, etc.

!!! info "The cluster pane is noisy on purpose"
    `cluster_tf` polls TFs at ~12 Hz and emits a warning every time it
    can't find one of its source frames. Until perception starts
    publishing, that's every tick. You'll see hundreds of lines.
    That noise is not the bug; it's a status indicator.

## Manual two-terminal launch (legacy / debugging)

If you want to bypass `tmuxp` and the per-task launch splits:

```bash
# Terminal 1: sim + ArduSub SITL + MAVROS + Foxglove + world markers
ros2 launch bluerov_sim bluerov_sim.launch.py \
    world_name:=empty.sdf ardusub:=true mavros:=true gui:=true

# Terminal 2a: legacy state-machine waypoint mission
ros2 launch bluerov_sim bluerov_mission.launch.py use_dvl:=false

# Terminal 2b: BT square mission (preferred legacy smoke test)
ros2 launch bluerov_sim bluerov_square_bt.launch.py
```

`bluerov_sim.launch.py` parses `spherical_coordinates` from the world
SDF to set ArduSub's home GPS position. If the SDF isn't on
`GZ_SIM_RESOURCE_PATH`, it falls back to default lat / lon.

## Diagnostic commands

The "is anything alive?" command toolkit. Memorise these. They solve
most "why isn't it working?" questions before you have to read code.

```bash
ros2 topic echo /mavros/state                       # FCU / MAVROS connection state
ros2 topic echo /mavros/local_position/pose         # vehicle pose (feedback path)
ros2 topic echo /bluerov/odom                       # raw Gazebo odom
ros2 action list | grep controls                    # /bluerov/controls should exist
ros2 service list | grep convert_to_controls_pose   # /bluerov/convert_to_controls_pose should exist
```

What each one tells you:

- `/mavros/state` shows `connected: true` and `mode`. If it isn't
  connected, the SITL ↔ MAVROS link is broken (ArduSub didn't start, or
  network is wrong).
- `/mavros/local_position/pose` is the closed-loop feedback the action
  server uses. If this isn't updating, the EKF isn't fusing odometry.
- `ros2 action list | grep controls` tells you the action server is
  alive and namespaced correctly.

### Manual arming and mode switch (bypass the BT)

When debugging without the BT, you can arm and switch to GUIDED by
hand:

```bash
ros2 service call /mavros/cmd/arming \
    mavros_msgs/srv/CommandBool "{value: true}"

ros2 service call /mavros/set_mode \
    mavros_msgs/srv/SetMode "{custom_mode: 'GUIDED'}"
```

Then publish a single setpoint by hand to drive the vehicle:

```bash
ros2 topic pub --once /mavros/setpoint_position/local geometry_msgs/PoseStamped \
    "{header: {frame_id: 'map'}, pose: {position: {x: 2.0, y: 0.0, z: -1.0}, orientation: {w: 1.0}}}"
```

Useful for confirming the control side works while perception is being
debugged.

## Foxglove

Open Foxglove Studio → **Open Connection** → **Rosbridge** →
`ws://localhost:8765`.

Recommended panels for the simulation:

| Panel | Topic | Why |
|---|---|---|
| 3D | `/tf` + your model URDF | See the robot moving in context. |
| Plot | `/mavros/local_position/pose` | Confirm closed-loop feedback updates. |
| Plot | `/bluerov/odom` | Raw Gazebo odom. |
| Topic List |. | Quick "is anything publishing?" view. |
| Markers | `/world_objects/markers` | Bin / torpedo panels in the pool world. |

## QGroundControl (optional)

QGC on the host will auto-discover the SITL because of `--network=host`.
Open it, click the vehicle in the connections list, and you can:

- Arm / disarm manually.
- Switch flight modes (GUIDED, STABILIZE, …).
- Send manual waypoints (bypassing the BT entirely).

Useful for confirming the sim is running independent of any ROS code.

## Common first-run failures and fixes

!!! warning "`vcs import` fails on submodules"
    You forgot `--recursive`. `image_matching` vendors XFeat as a
    submodule and `simple_matcher_node` crashes on import without it.
    Re-run `vcs import --recursive src < bluerov_ws.repos` or
    `cd src/image_matching && git submodule update --init --recursive`.

!!! warning "Gazebo can't find the world SDF"
    Check `echo $GZ_SIM_RESOURCE_PATH`. Worlds in
    [bb_worlds](../packages/bb_worlds.md) install a hook that adds the
    package's share dir to that variable. If you sourced
    `install/setup.bash` from a different terminal, the path is empty
    there.

!!! warning "MAVROS shows `connected: false`"
    Sequence: did ArduSub SITL actually launch? Check the `sim` pane
    for ArduPilot startup logs. If SITL didn't start, the JSON-UDP
    bridge isn't there for MAVROS to connect to.

!!! warning "`cluster_tf` logs `LookupException` forever"
    The source TF the BT asked for has no broadcaster. Check that
    the right `*_pose_estimator_node` is running in the vision pane
    and that it's actually receiving its input (camera image,
    point correspondences, …). Examples in the
    [Bin](../strategies/bin.md) and [Torpedo](../strategies/torpedo.md)
    mission pages.

## See also

- [Concepts](concepts.md): vocabulary for everything above.
- [Architecture](architecture.md): what each layer does.
- [Strategies](../strategies/index.md): what each mission's BT
  actually does.
- [Conventions](conventions.md): the rules every change to the BT
  or control side has to respect.
