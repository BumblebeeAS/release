# RobotX 2026

The [Maritime RobotX Challenge](https://robotx.org/) is an international
competition in which teams develop autonomous maritime systems that cooperate
across the surface, air, and underwater domains. Unlike single-vehicle
competitions, RobotX rewards teams that can coordinate heterogeneous robots —
a surface vessel, an aerial scout, and an underwater vehicle — to complete
tasks that no single platform could accomplish alone. Building on our 1st Place
Autonomy finish at RobotX 2024, we are preparing for RobotX 2026 with a strong
emphasis on multi-vehicle simulation before on-water testing.

## Why Simulation First

On-water testing time is scarce, expensive, and weather-dependent, and a fault
in one vehicle can stall an entire multi-robot rehearsal. By reproducing the
full three-vehicle system in simulation, we can iterate on mission logic and
inter-vehicle coordination every day, reproduce failures deterministically, and
arrive at the water with software that already works as an integrated system.
The same perception, controls, and mission code we run in the simulator is what
we intend to deploy on the real platforms, so progress made in simulation
carries directly over to hardware.

## Multi-Vehicle Simulation

Our RobotX 2026 effort is built around a shared Gazebo simulation that runs an
Autonomous Surface Vehicle (ASV), an Unmanned Aerial Vehicle (UAV), and an
Autonomous Underwater Vehicle (AUV) together in one world — a model of the
Singapore river competition course. All three vehicles are spawned into the
same scene and bridged into ROS 2, so they share a common clock, coordinate
frames, and message bus exactly as they would on the water.

The simulator and a set of end-to-end demos are public:

- [`BumblebeeAS/multivehicle_sim`](https://github.com/BumblebeeAS/multivehicle_sim)
  — the shared world, vehicle models, and the Gazebo ↔ ROS 2 bridges.
- [`BumblebeeAS/examples`](https://github.com/BumblebeeAS/examples) — example
  missions and controllers that run on top of the simulator (the
  `multivehicle` project), launchable as a full stack with a single tmuxp
  session.

| Vehicle | Platform | Stack |
|---------|----------|-------|
| ASV | BlueBoat | thrust mixer + line-of-sight waypoint controller |
| UAV | PX4 x500 | PX4 SITL + uXRCE-DDS offboard flight |
| AUV | BlueROV2 | ArduSub / MAVROS missions |

### Autonomous Surface Vehicle — BlueBoat

The BlueBoat is the centerpiece of the RobotX course. Its control stack pairs a
thrust mixer — which converts desired surge and yaw commands into individual
thruster outputs — with a line-of-sight (LOS) waypoint controller that steers
the vessel smoothly along a sequence of waypoints. A demo mission drives the
boat through a course using this controller, exercising the same path-following
logic we will use for the navigation and station-keeping tasks.

### Unmanned Aerial Vehicle — PX4 x500

The x500 quadrotor runs full PX4 software-in-the-loop, communicating over the
uXRCE-DDS bridge just as the flight controller does on real hardware. An
offboard flight demo commands autonomous trajectories from ROS 2, giving us a
realistic testbed for an aerial scout that can survey the course and feed
information back to the surface vehicle.

### Autonomous Underwater Vehicle — BlueROV2

The BlueROV2 runs ArduSub with MAVROS, mirroring the underwater platform's real
autopilot interface. A scripted square-dive mission demonstrates depth-holding
and waypoint navigation underwater, the foundation for the submerged portions of
the challenge.

## Tooling and Integration

Running all three vehicles in one world lets us develop and debug the hand-offs
between them — for example, the UAV scouting ahead of the ASV — rather than
testing each platform in isolation. The same setup also validates the supporting
infrastructure as an integrated system:

- **Odometry bridges** publish each vehicle's pose into ROS 2, so missions and
  visualization tools see a consistent picture of the whole fleet.
- **An operations dashboard** tracks all three vehicles' odometry on a live web
  view, approximating the shore-side situational awareness we rely on at the
  competition.
- **Foxglove** provides deep, real-time introspection of topics and transforms
  during development and debugging.

More details, results, and on-water footage will be added here as our RobotX
2026 campaign progresses.
