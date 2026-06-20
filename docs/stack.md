# BumblebeeAS Software Stack Overview

This page contains an overview of our vehicle software architectures and maps out the individual Git repositories.
See the README of each repository for further information.

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

## Shared Interfaces and Mission Planning

| Repository                                                                          | What it does                                                                  |
| ----------------------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| [`bb_msgs`](https://github.com/BumblebeeAS/bb_msgs)                                 | Shared ROS messages, services, and actions.                                   |
| [`frames`](https://github.com/BumblebeeAS/frames)                                   | Compile-time frame types to make coordinate conversions safer.                |
| [`mission_planner_release`](https://github.com/BumblebeeAS/mission_planner_release) | py_trees-based mission-planning stack and reusable behaviour-tree primitives. |

## Perception

```mermaid
flowchart LR
  subgraph VISION["Vision Pipeline"]
    direction LR

    CAM["Camera"]
    IMG["Image Processing"]
    MATCH["Image Matching"]
    YOLO["YOLO"]
    DEPTH["Depth Estimation"]
    POSE["Pose Estimator"]

    CAM --> IMG --> MATCH --> POSE
    CAM --> YOLO --> POSE
    CAM --> DEPTH --> POSE
  end

  POSE --> CLUSTER["Pose Clustering (Filters)"]
  CLUSTER --> BT["Mission Planner"]
```

| Repository                                                                          | What it does                                                  |
| ----------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| [`accelerated_features`](https://github.com/BumblebeeAS/accelerated_features)       | XFeat implementation for fast local feature extraction.       |
| [`depth-anything-tensorrt`](https://github.com/BumblebeeAS/depth-anything-tensorrt) | TensorRT inference for Depth Anything V1 and V2.              |
| [`filters`](https://github.com/BumblebeeAS/filters)                                 | Detection filtering and pose clustering for stable targets.   |
| [`image_matching`](https://github.com/BumblebeeAS/image_matching)                   | Local-feature matching against known targets.                 |
| [`image_processing`](https://github.com/BumblebeeAS/image_processing)               | ROS 2 utilities for image enhancement and timestamp handling. |
| [`pose_estimator`](https://github.com/BumblebeeAS/pose_estimator)                   | Vision-based 6-DoF target-pose estimation and TF publication. |
| [`vision_pipeline`](https://github.com/BumblebeeAS/vision_pipeline)                 | Launch files and utilities for orchestrating vision nodes.    |
| [`yolo_ros_trt`](https://github.com/BumblebeeAS/yolo_ros_trt)                       | ROS 2 object-detection wrapper for YOLO and TensorRT.         |

## Simulation

| Repository                                                  | What it does                                                 |
| ----------------------------------------------------------- | ------------------------------------------------------------ |
| [`ardusub_sim`](https://github.com/BumblebeeAS/ardusub_sim) | ArduSub and Gazebo simulation, built on Project DAVE.        |
| [`bb_worlds`](https://github.com/BumblebeeAS/bb_worlds)     | Gazebo worlds and assets for vehicles and competition tasks. |

## Developer Platforms and Tooling

| Repository                                                                                        | What it does                                                   |
| ------------------------------------------------------------------------------------------------- | -------------------------------------------------------------- |
| [`controlkitv3`](https://github.com/BumblebeeAS/controlkitv3)                                     | Foxglove-based interface for vehicle control and telemetry.    |
| [`isaac-ros-docker`](https://github.com/BumblebeeAS/isaac-ros-docker)                             | Docker tooling for Isaac ROS on legacy JetPack 6 Orin systems. |
| [`release`](https://github.com/BumblebeeAS/release)                                               | Source for this documentation site and public release notes.   |
| [`ros-humble-noetic-bridge`](https://github.com/BumblebeeAS/ros-humble-noetic-bridge)             | Ready-to-use bridge between ROS Noetic and ROS 2 Humble.       |
| [`ros-humble-ros1-bridge-builder`](https://github.com/BumblebeeAS/ros-humble-ros1-bridge-builder) | Builder for producing a Humble–ROS 1 bridge package.           |
