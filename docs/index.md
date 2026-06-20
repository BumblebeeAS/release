# BumblebeeAS Open Source

We build software for autonomous underwater, surface, and aerial robots. Our stack spans mission planning, perception, controls, simulation, sensor integration, and developer tooling.

## AUV architecture

```mermaid
%%{init: {"themeVariables": {"fontSize": "18px"}}}%%
flowchart LR
  subgraph SENSING[" "]
    direction TB

    subgraph PERCEPTION["<b>Perception</b>"]
      direction TB
      HYDROPHONES["Hydrophones"] --> ACOUSTIC_DETECTION_AUV["Acoustic Detection"]
      CAMERAS["Cameras"] --> VISION["Vision Pipeline"]
      FLS["FLS Sonar"] --> SONAR["Sonar Processing"]
    end

    subgraph LOCALIZATION["<b>Localization</b>"]
      direction LR
      DVL["Doppler<br/>Velocity Log"] --> FUSION["UKF + Fusion"]
      IMU["Inertial<br/>Measurement Unit"] --> FUSION
      FOG["Fiber Optic<br/>Gyroscope"] --> FUSION
    end
  end

  subgraph MISSION["<b>Mission Architecture</b>"]
    PLANNER["Mission Planner"]
  end

  subgraph CONTROL["<b>Navigation &amp; Control</b>"]
    direction TB
    TRAJECTORY["Trajectory Planner"] --> CONTROLLER["Controller"] --> ALLOCATOR["Thrust Allocator"]
  end

  subgraph NETWORK["<b>Network &amp; Telemetry</b>"]
    direction LR
    IVC["IVC Module"] <--> ACOUSTIC["Hydrophones"]
    CONVERSION["Message Conversion"] --> TELEMETRY["Telemetry"]
  end

  CONVERSION <--> THRUSTERS["Thrusters"]
  POWER["Power"] --> CONVERSION
  ACTUATOR_CONTROL_AUV["Actuator Control"] --> CONVERSION

  ACOUSTIC_DETECTION_AUV --> PLANNER
  VISION --> PLANNER
  SONAR --> PLANNER
  FUSION --> CONTROLLER

  PLANNER --> TRAJECTORY
  PLANNER <--> CONVERSION
  PLANNER <--> IVC
  ALLOCATOR --> CONVERSION

  style SENSING fill:transparent,stroke:transparent
```

## ASV architecture

```mermaid
%%{init: {"themeVariables": {"fontSize": "18px"}}}%%
flowchart LR
  subgraph SENSING_ASV[" "]
    direction TB

    subgraph PERCEPTION_ASV["<b>Perception</b>"]
      direction LR
      HYDROPHONES_ASV["Hydrophones"] --> ACOUSTIC_DETECTION["Acoustic<br/>Detection"]
      CAMERAS_ASV["Cameras"] --> IMAGE_PIPELINE["Vision<br/>Pipeline"]
      LIDARS["LiDARs"] --> LIDAR_PIPELINE["LiDAR<br/>Pipeline"]

      ACOUSTIC_DETECTION --> TARGET_FUSION["Multi-target<br/>Tracking / Fusion"]
      IMAGE_PIPELINE --> TARGET_FUSION
      LIDAR_PIPELINE --> TARGET_FUSION
    end

    subgraph LOCALIZATION_ASV["<b>Localization</b>"]
      direction LR
      GNSS["Dual-antenna<br/>GNSS/INS"] --> UKF["UKF"]
    end
  end

  subgraph MISSION_ASV["<b>Mission Architecture</b>"]
    MISSION_PLANNER_ASV["Mission Planner"]
  end

  subgraph NAVIGATION["<b>Navigation &amp; Control</b>"]
    direction LR
    BT_NAVIGATOR["BT Navigator"] --> NAV_PLANNER["Trajectory Planner"] --> CONTROLLER_ASV["Controller"] --> THRUST_ALLOCATOR_ASV["Thrust Allocator"]
  end

  subgraph COMMS[" "]
    direction TB
    TELEMETRY_ASV["Telemetry"] --> TD_CLIENT["TD Client"]
    subgraph AUXILIARY_CONTROL[" "]
      direction LR
      SHOOTER["Shooter"]
      HYDROPHONES_CONTROL["Hydrophones"]
    end
  end

  THRUST_ALLOCATOR_ASV --> THRUSTERS_ASV["Thrusters"]
  ACTUATOR_CONTROL["Actuator Control"]

  LIDAR_PIPELINE --> UKF
  LIDAR_PIPELINE --> MAPPER["Mapper"]
  TARGET_FUSION --> MISSION_PLANNER_ASV
  UKF --> MAPPER
  UKF --> NAV_PLANNER

  MISSION_PLANNER_ASV --> BT_NAVIGATOR
  MISSION_PLANNER_ASV <--> TELEMETRY_ASV
  MISSION_PLANNER_ASV --> SHOOTER
  MISSION_PLANNER_ASV --> HYDROPHONES_CONTROL
  MISSION_PLANNER_ASV --> ACTUATOR_CONTROL

  style SENSING_ASV fill:transparent,stroke:transparent
  style COMMS fill:transparent,stroke:transparent
  style AUXILIARY_CONTROL fill:transparent,stroke:transparent
```

## UAV architecture

```mermaid
%%{init: {"themeVariables": {"fontSize": "18px"}}}%%
flowchart LR
  subgraph PERCEPTION_UAV["<b>Perception</b>"]
    direction LR
    CAMERA_UAV["Camera"]
    ML["ML"]
    IMAGE_MATCHING_UAV["Image Matching"]
    FILTERS_UAV["Filters"]

    CAMERA_UAV --> ML
    CAMERA_UAV --> IMAGE_MATCHING_UAV
    ML --> FILTERS_UAV
    IMAGE_MATCHING_UAV --> FILTERS_UAV
  end

  subgraph MISSION_UAV["<b>Mission Architecture</b>"]
    MISSION_PLANNER_UAV["Mission Planner"]
  end

  subgraph FLIGHT_CONTROL["<b>Navigation &amp; Control</b>"]
    direction LR
    RTK_GPS["RTK GPS"] --> PX4
    IMU_UAV["IMU"] --> PX4
    TRAJECTORY_UAV["Trajectory Planner"] --> PX4["PX4 Flight<br/>Controller"] --> THRUST_ALLOCATOR_UAV["Thrust Allocator"]
  end

  subgraph COMMUNICATION_UAV["<b>Communication</b>"]
    RADIO_UAV["Radio"]
  end

  THRUST_ALLOCATOR_UAV --> THRUSTERS_UAV["Thrusters"]
  ACTUATOR_CONTROL_UAV["Actuator Control"]

  FILTERS_UAV --> MISSION_PLANNER_UAV
  MISSION_PLANNER_UAV --> TRAJECTORY_UAV
  PX4 --> MISSION_PLANNER_UAV
  MISSION_PLANNER_UAV <--> RADIO_UAV
  MISSION_PLANNER_UAV --> ACTUATOR_CONTROL_UAV

```

## Getting Started

To view our software in action, see the [`BumblebeeAS/examples`](https://github.com/BumblebeeAS/examples) repository which contains end-to-end demos and quick-start instructions on how to set up your **ROS2 Humble** workspace.

The [software stack overview](stack.md#bumblebeeas-software-stack-overview) provides a reference of the individual Git repositories in our stack.

## Competitions

To see our approach in applying our software stack in competitions settings, see:

- [RoboSub25](robosub25.md#robosub-2025)
- *more to come ...*

## Contributing

Contributions are welcome!

Feel free to open an issue or pull request in the respective repository. We are also open to discussion on the RoboNation Community discord.
