# bring-up

**Role:** Unified provisioning scripts for the AUV4.5 / UAV2 / ASV
platforms. Both real hardware (SBC + Orin) and the various simulation
environments.

For the BlueROV2 workspace, the relevant pieces are the simulation
configurations under `etc/sim/`.

## Layout

| Path | Purpose |
|---|---|
| `etc/sim/` | Tmuxp configs and per-environment settings for simulation. |
| `etc/auv4_sbc/` | Real-AUV4 SBC: systemd user services, network config, udev rules. |
| `etc/auv4_orin/` | Orin compute module setup. |
| `*.repos` | vcstool repo lists per environment (some are private bumblebeeAS GitHub repos). |

## ROS interfaces

None. This is a pure provisioning package.

## Gotchas

- Needs `vcstool` to import dependencies, and SSH access to the private
  bumblebeeAS GitHub for those repos.
- Real-hardware SBC bring-up uses systemd **user** services (non-root
  startup): different from a typical robot system service install.
- SSH key setup between SBC and Orin is a prerequisite for inter-board
  comms.
- Orin setup runs **outside** Docker before first build; that affects
  cache ownership inside the container.

## See also

- [Running the sim](../overview/running.md): the BlueROV sim does not
  use this package directly; we run our own tmuxp mission files.
