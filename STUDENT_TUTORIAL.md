# Student Tutorial for UAV Motion Stack

This document explains how to use the provided PX4 SITL environment and how to complete the student-facing part of the project.

The stack is designed so that students do not need to start from a blank simulator. The low-level integration is already provided, and the student work is focused on high-level autonomy:

- planning
- navigation
- mission sequencing
- interpreting high-level task commands

## 1. What Is Already Provided

The repository already contains the infrastructure needed to run a realistic simulation:

- Docker files for a reproducible development environment
- Gazebo Garden models and worlds
- PX4 ROS 2 offboard control example supported through `px4_ros_com`
- frame conversion and odometry support through `drone_odometry`
- a high-level command manager through `babyk_drone_manager`

The most important idea is this:

- `px4_ros_com` is the minimal offboard bridge provided just as an example
- `drone_odometry` handles ENU/NED and PX4-compatible odometry wiring
- `babyk_drone_manager` receives high-level string commands and converts them into goals for the rest of the stack

Students should build above these layers, not replace them.

## 2. What Students Must Build

The student deliverable is the autonomy layer that sits on top of the provided middleware. In practice, this means implementing or extending the components that decide where the drone should go and how it should move there safely.

Typical responsibilities are:

- deciding the next mission goal
- generating a path or trajectory between goals
- handling transitions such as takeoff, navigation, loiter, and landing
- reacting to the command interface exposed by the manager package
- validating behavior in simulation before any real-world use

A good final result is a stack where a student can send a high-level command and see the drone execute a complete mission in simulation.

## 3. Repository Layout

The most relevant paths are:

- [README.md](README.md)
- [ros2_ws-src/pkg/babyk_drone_manager/README.md](ros2_ws-src/pkg/babyk_drone_manager/README.md)
- [ros2_ws-src/pkg/drone_odometry/README.md](ros2_ws-src/pkg/drone_odometry/README.md)
- [ros2_ws-src/px4_ros_com/README.md](ros2_ws-src/px4_ros_com/README.md)
- [ros2_ws-src/pkg/babyk_drone_manager/utils/simulation.yml](ros2_ws-src/pkg/babyk_drone_manager/utils/simulation.yml)

The simulation session is started from the manager package because it already knows how the pieces fit together.

## 4. Setup

### 4.1 Clone the repository

```bash
git clone --recursive https://github.com/Prisma-Drone-Team/uav_motion_stack.git
cd uav_motion_stack
```

### 4.2 Clone the custom PX4 firmware

The container startup script expects the custom firmware checkout to be present as `PX4_neabotics` in the repository root.

```bash
git clone --single-branch -b feature/diffgains_fix_servo_k https://github.com/Prisma-Drone-Team/Px4_hcore_autopilot.git PX4_neabotics --recursive
```

### 4.3 Build the Docker image

```bash
cd docker
docker build -t leo-img -f px4_humble_dockerfile.txt .
```

### 4.4 Start the container

```bash
cd ..
./run_cnt.sh
```

Inside the container, `init_drone.sh` will build the ROS 2 workspace and prepare the environment.

## 5. First Build Inside The Container

Once the container is running:

```bash
cd ~/ros2_ws
source /opt/ros/humble/setup.bash
colcon build --cmake-args -DCMAKE_BUILD_TYPE=Release
source install/setup.bash
```

If you are iterating on a single package, rebuild only that package when possible to keep the feedback loop short.

## 6. Start The Full Simulation

The provided TMUX session launches the main simulation components.

```bash
cd ~/ros2_ws
tmuxp load src/pkg/babyk_drone_manager/utils/simulation.yml
```

To stop everything:

```bash
tmux kill-server
```

## 7. How The Command Flow Works

The stack is designed around a simple contract.

### Input side

A high-level command is sent to the manager as a string on:

- `/seed_pdt_drone/command`

Examples of commands that the manager already understands include:

- `takeoff`
- `land`
- `stop`
- `go(x,y,z)`
- `flyto(frame_name)`

### Output side

The manager publishes status information on:

- `/seed_pdt_drone/status`

That status channel is what students should use to understand whether the system is ready, busy, or blocked.

### Low-level bridge

The PX4 bridge and odometry layer handle the technical communication details:

- `/fmu/out/vehicle_odometry`
- `/px4/odometry/out`
- `/px4/trajectory_setpoint_enu`
- `/fmu/in/trajectory_setpoint`

Students should not spend time re-implementing these lower layers unless they are explicitly extending them.

## 8. Recommended Implementation Path

The safest way to complete the project is to follow this order.

### Step 1. Understand the provided manager

Read the package documentation for `babyk_drone_manager` and understand:

- which commands it accepts
- which status messages it publishes
- how it maps commands to goals and trajectories

### Step 2. Verify offboard communication

Use `px4_ros_com` as the simplest bridge to confirm that offboard messages can reach PX4 correctly.

### Step 3. Confirm odometry consistency

Check that `drone_odometry` is publishing coherent frame transforms and odometry in the frame your planner expects.

### Step 4. Implement the student autonomy layer

Add the actual planner and navigation logic that decides where the drone should go next. This may include:

- waypoint selection
- obstacle-aware routing
- path smoothing
- trajectory interpolation
- safety checks before command execution

### Step 5. Wire the planner to the manager

Make sure the planner can consume the high-level commands and produce executable goals for the rest of the stack.

### Step 6. Validate in simulation

Start with simple missions:

- takeoff and land
- move to one waypoint
- move through a short sequence of waypoints
- recover cleanly after a stop command

Only after these work should you test more complex mission logic.

## 9. Minimal Testing Checklist

Before considering the project complete, verify the following:

- the container starts without manual fixes
- the ROS 2 workspace builds cleanly
- the simulation launches through the TMUX session
- `takeoff` works
- `land` works
- a direct movement command reaches the drone
- a multi-goal mission can be executed
- the status topic reflects the current state correctly
- the system can be stopped cleanly

## 10. Practical Development Tips

- Keep your planning layer independent from PX4 details whenever possible.
- Use the manager package as the interface contract with the rest of the stack.
- Validate each layer separately before combining them.
- Prefer deterministic behavior during testing, then add randomness only if it is useful for robustness checks.
- If a command is not understood, the manager should fail safely rather than guess.

## 11. Suggested Deliverable Structure

A clean student solution usually ends up with:

- a planning node
- a navigation or mission manager node
- a trajectory or waypoint interpolation node if needed
- configuration files for the simulation and the real test arena
- launch files that start the new student stack together with the provided infrastructure

## 12. Final Goal

The final goal is a simulation where a student can send one high-level command and let the stack handle the rest:

1. interpret the request
2. compute a path or mission plan
3. convert the result into a valid flight goal
4. execute the motion safely in PX4 SITL
5. report the current state back through the manager

That is the point of the repository: the lower layers are already in place, so students can focus on autonomy instead of infrastructure.
