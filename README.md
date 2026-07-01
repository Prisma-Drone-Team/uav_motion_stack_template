# UAV Motion Stack

This repository provides a Gazebo Garden and PX4 SITL development environment for a drone motion stack focused on student-level high-level autonomy. The goal is to give a ready-to-run simulation where students work on planning and navigation, while the low-level PX4 bridge, frame conversions, and command management are already available.

For the student workflow and the expected implementation path, see [STUDENT_TUTORIAL.md](STUDENT_TUTORIAL.md).

## Architecture Overview

The repository is organized around a small set of ROS 2 packages and support files:

```
uav_motion_stack/
├── docker/                     # Docker image and container startup files
├── models/                     # Custom Gazebo models for the arena
├── worlds/                     # Gazebo world definitions
├── ros2_ws-src/
│   ├── px4_ros_com/            # Minimal PX4 ROS 2 bridge and offboard example
│   └── pkg/
│       ├── drone_odometry/     # Odometry and frame conversion utilities
│       └── babyk_drone_manager/# High-level command manager and safety layer
├── init_drone.sh               # Container bootstrap script
└── run_cnt.sh                  # Host-side container launcher
```

## What Is Included

- A Docker-based ROS 2 Humble environment for PX4 SITL.
- Gazebo Garden simulation assets for the Leonardo Drone Contest-style arena.
- `px4_ros_com` as the simplest offboard reference implementation.
- `drone_odometry` for ENU/NED conversions and PX4-compatible odometry wiring.
- `babyk_drone_manager` as the high-level command layer that receives strings and turns them into goals for the rest of the stack.

## What Students Are Expected To Build

The repository is intentionally incomplete at the autonomy layer. Students are expected to implement the parts that sit above the provided infrastructure:

- Planning logic.
- Navigation logic.
- Goal generation and mission sequencing.
- Any student-specific behavior that interprets the high-level commands exposed by `babyk_drone_manager`.

The provided packages are the support layer that makes that work possible.

## System Requirements

- Docker
- ROS 2 Humble
- PX4 v1.14 or newer
- Gazebo Garden
- Eigen3

## Installation and Setup

### 1. Clone the repository

```bash
git clone --recursive https://github.com/Prisma-Drone-Team/uav_motion_stack_template.git -b paper_stable
cd uav_motion_stack_template
```

### 2. Clone the PX4 firmware used for plug-and-play simulation

This repository expects the custom PX4 firmware checkout to be available as `PX4_neabotics` next to the repo root when using the provided container scripts.

```bash
git clone --single-branch -b feature/diffgains_fix_servo_k https://github.com/Prisma-Drone-Team/Px4_hcore_autopilot.git PX4_neabotics --recursive
```

### 3. Build the Docker image

```bash
cd docker
docker build -t leo-img -f px4_humble_dockerfile.txt .
```

The container is configured for Gazebo Garden. 

### 4. Run the container

From the repository root:

```bash
./run_cnt.sh
```

The container mounts the ROS 2 workspace, the custom PX4 firmware, and the provided launch assets.

## Build The ROS 2 Workspace

Inside the container:

```bash
cd ~/ros2_ws
source /opt/ros/humble/setup.bash
colcon build --cmake-args -DCMAKE_BUILD_TYPE=Release
source install/setup.bash
```

For iterative development, rebuilding with `--symlink-install` is also supported by the packages in this workspace.

## Simulation Workflow

The quickest way to start the full simulation is to load the TMUX session prepared by `babyk_drone_manager`:

```bash
cd ~/ros2_ws
tmuxp load src/pkg/babyk_drone_manager/utils/simulation.yml
```

To stop everything, terminate the TMUX server:

```bash
tmux kill-server
```

## Package Roles

### px4_ros_com

This is the minimal ROS 2 side of the PX4 DDS bridge. In this stack it is used as the simplest offboard reference and as the foundation for sending commands to PX4. The package also includes an offboard example node and launch files.

Useful reference: [ros2_ws-src/px4_ros_com/README.md](ros2_ws-src/px4_ros_com/README.md)

### drone_odometry

This package handles frame conversions and odometry wiring for the stack. It converts PX4 data into ROS 2-friendly messages and publishes the transforms needed for navigation.

Useful reference: [ros2_ws-src/pkg/drone_odometry/README.md](ros2_ws-src/pkg/drone_odometry/README.md)

### babyk_drone_manager

This package is the high-level control and safety entry point. It receives string commands, manages state, and translates those commands into goals that the student stack can consume.

Useful reference: [ros2_ws-src/pkg/babyk_drone_manager/README.md](ros2_ws-src/pkg/babyk_drone_manager/README.md)

## Main Interfaces

### High-level command flow

- Input command topic: `/seed_pdt_drone/command` (`std_msgs/String`)
- Status topic: `/seed_pdt_drone/status` (`std_msgs/String`)

### PX4 and odometry integration

- PX4 odometry input: `/fmu/out/vehicle_odometry`
- ROS odometry output: `/px4/odometry/out`
- ENU setpoint input: `/px4/trajectory_setpoint_enu`
- PX4 setpoint output: `/fmu/in/trajectory_setpoint`

## Main Dependencies

The common ROS 2 dependencies used by the stack are:

```xml
<depend>rclcpp</depend>
<depend>px4_msgs</depend>
<depend>nav_msgs</depend>
<depend>geometry_msgs</depend>
<depend>trajectory_msgs</depend>
<depend>tf2</depend>
<depend>tf2_ros</depend>
<depend>eigen3_cmake_module</depend>
```

## Notes On The Intended Workflow

- `px4_ros_com` is the smallest possible bridge to verify offboard communication.
- `drone_odometry` keeps the frame conversions and odometry alignment consistent.
- `babyk_drone_manager` is the contract between the provided infrastructure and the student-developed autonomy layer.
- Students should extend the stack above these interfaces, not replace them.

## Documentation

- [STUDENT_TUTORIAL.md](STUDENT_TUTORIAL.md)
- [ros2_ws-src/px4_ros_com/README.md](ros2_ws-src/px4_ros_com/README.md)
- [ros2_ws-src/pkg/drone_odometry/README.md](ros2_ws-src/pkg/drone_odometry/README.md)
- [ros2_ws-src/pkg/babyk_drone_manager/README.md](ros2_ws-src/pkg/babyk_drone_manager/README.md)

## Important Notes

The PX4 firmware trees are used for simulation and are not required on the target student machine if the containerized stack is used as intended. The custom `PX4_neabotics` checkout is the one expected by the container scripts and the simulation flow in this repository.

