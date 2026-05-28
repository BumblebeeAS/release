# Running the sim

Everything runs inside the Docker container — the host doesn't have ROS or
Gazebo installed. The workspace mounts at `/root/HOST/bluerov_ws`.

## First-time setup

```bash
# Host
./build.bash                    # tags bluerov_ws:humble
./run.bash bluerov_ws:humble    # rocker wrapper: --nvidia --x11 --network=host
```

`--network=host` is required: MAVROS broadcasts MAVLink on UDP 14550 so
QGroundControl on the host picks it up automatically; the Foxglove bridge
listens on 8765.

```bash
# Container — first time only
vcs import src < bluerov_ws.repos

# Container — build (always with --symlink-install so Python edits take
# effect without rebuild)
source /opt/ros/humble/setup.bash
colcon build --symlink-install
source install/setup.bash

# After editing a single package
colcon build --packages-select bluerov_sim && source install/setup.bash
```

There is no separate lint or test step. `frames/` has compile-time unit
tests that run as part of `colcon build`.

## Launching a mission

Three tmuxp configurations, one per mission:

```bash
tmuxp load bluerov_mission.yaml            # legacy square / sm waypoint
tmuxp load bluerov_bin_mission.yaml        # bin drop (RoboSub-2025 pool)
tmuxp load bluerov_torpedo_mission.yaml    # torpedo launch (RoboSub-2025 pool)
```

The bin and torpedo missions follow a 5-pane split:

| Pane | Launch file | What it runs |
|---|---|---|
| `sim` | `bluerov_<task>_sim.launch.py` | Gazebo + ArduSub + MAVROS + Foxglove |
| `controls` | `bluerov_<task>_controls.launch.py` | `LocomotionActionServer` + frames service |
| `cluster` | `bluerov_<task>_cluster.launch.py` | `cluster_tf` aggregators |
| `vision` | `bluerov_<task>_vision.launch.py` | Detector + pose estimator |
| `bt` | `bluerov_<task>_bt.launch.py` | The mission behaviour tree |

Each layer is its own launch file, so any single pane can be `Ctrl-C`'d and
restarted without bouncing the rest. The `cluster` pane is noisy — `cluster_tf`
polls TFs at ~12 Hz and warns until a target frame is broadcast.

## Manual two-terminal launch (legacy)

```bash
# T1: sim + ArduSub SITL + MAVROS + Foxglove + world markers
ros2 launch bluerov_sim bluerov_sim.launch.py \
    world_name:=empty.sdf ardusub:=true mavros:=true gui:=true

# T2a: state-machine waypoint mission
ros2 launch bluerov_sim bluerov_mission.launch.py use_dvl:=false

# T2b: BT square mission (preferred)
ros2 launch bluerov_sim bluerov_square_bt.launch.py
```

`bluerov_sim.launch.py` parses `spherical_coordinates` from the world SDF to
set ArduSub home. If the SDF is not on `GZ_SIM_RESOURCE_PATH`, it falls back
to default lat/lon.

## Diagnostics

```bash
ros2 topic echo /mavros/state                       # FCU / MAVROS connection
ros2 topic echo /mavros/local_position/pose         # vehicle pose (feedback path)
ros2 topic echo /bluerov/odom                       # raw Gazebo odom
ros2 action list | grep controls                    # /bluerov/controls should exist
ros2 service list | grep convert_to_controls_pose   # /bluerov/convert_to_controls_pose should exist
```

Manual arming and mode switch (useful for bypassing the BT during debugging):

```bash
ros2 service call /mavros/cmd/arming \
    mavros_msgs/srv/CommandBool "{value: true}"
ros2 service call /mavros/set_mode \
    mavros_msgs/srv/SetMode "{custom_mode: 'GUIDED'}"
```

## Foxglove

Open Foxglove Studio → **Open Connection** → **Rosbridge** →
`ws://localhost:8765`.

Useful panels: `/tf`, `/mavros/local_position/pose`, `/bluerov/odom`,
`/world_objects/markers`.
