# Overview

Four pages explain how the workspace fits together. **Read them in this
order**: each builds on the previous.

- **[Concepts](concepts.md)**: *start here if anything above looks
  unfamiliar*: ROS 2, TF, behaviour trees, py_trees, MAVROS, ArduSub,
  Gazebo, cluster_tf, anchor frames, sim time. A survival glossary for
  beginners.
- **[Architecture](architecture.md)**: the mission → vehicle pipeline,
  layer by layer. What each layer is responsible for, and what
  contract it exposes to the next.
- **[Conventions](conventions.md)**: frame conventions (FLU vs ENU),
  depth-sign rules, anchor frames, the 1 Hz setpoint invariant, and
  other things that silently break code if violated.
- **[Running the sim](running.md)**: Docker entry, tmuxp missions,
  manual two-terminal launch, and the diagnostic commands you'll
  actually use.

Once you've finished these four, the per-mission
**[Strategies](../strategies/index.md)** and per-package
**[Packages](../packages/index.md)** pages will make a lot more sense.
