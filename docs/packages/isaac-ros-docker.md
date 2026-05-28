# isaac-ros-docker

**Role:** Provisions Docker images for NVIDIA's Isaac ROS GPU-accelerated
vision stack (Orin / x86).

**Not the base image for this workspace.** We build on
`ros:humble-ros-base-jammy` + rocker. Isaac ROS images are vendored here so
ML deps can be cherry-picked into our image rather than us switching to
the heavier Isaac ROS base.

## Layout

| Path | Purpose |
|---|---|
| `dockerfiles/environments/Dockerfile.*` | Per-environment images (`auv4_orin`, `asv`, `uav2`, `auv_sim`, `uav2_sim`). |
| `environments/` | Per-environment config and dependency lists. |
| `scripts/setup_env_paths.sh` | Symlinks config files into the Isaac ROS common workspace. |

## ROS interfaces

None — pure container provisioning.

## Gotchas

- Needs `isaac_ros_common` (cloned separately from NVIDIA upstream, branch
  `release-3.2`).
- `BUILD_IMAGE_FLAG=0` is recommended for multi-container tmuxp launches.
- GPU / Jetson VPI must be set up on the host **before** image build.
- rosdep apt/pip lists are generated per environment and baked into the
  Dockerfile.

## See also

- The BlueROV workspace uses [build.bash](../overview/running.md#first-time-setup)
  + rocker. Do not migrate to the Isaac ROS base.
