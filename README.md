# UAV MOTION STACK

This repository contains tools and configurations for PX4 SITL (Software In The Loop) simulation and hardware-specific deployment.

## Architecture Overview

The system consists of several modular ROS2 packages, each with a specific responsibility:

```
uav_motion_stack_template/
├── ros2_ws-src/                    # ROS2 workspace with modular packages
│   ├── drone_odometry2/            # Vehicle odometry publisher (submodule)
│   ├── babyk_drone_manager/        # Drone state management and safety (submodule)
├── docker/               # Docker configurations
├── models/               # Custom Gazebo models
├── worlds/               # Gazebo worlds for simulation
└── PX4_neabotics/        # PX4 custom firmware 
```

## System Requirements

- **Docker**: For isolated development environment
- **ROS2 Humble**: Robotics framework
- **PX4 v1.14+**: Autopilot firmware
- **Gazebo Garden**: 3D simulator
- **Eigen3**: Mathematical library for matrix operations

## Installation and Setup

A step by step series of examples that tell you how to get a development environment running:

### 1. Repository Clone
```bash
git clone --recursive https://github.com/Prisma-Drone-Team/uav_motion_stack_template.git -b paper_stable
cd uav_motion_stack_template
```

### 2. Clone PX4 Firmware (Optional)
> **Note:** This step is completely optional given the current state of the repository. The custom firmware in step 3 is sufficient for the full stack.

```bash
git clone --single-branch -b release/1.14 git@github.com:PX4/PX4-Autopilot.git --recursive
```

### 3. Clone PX4 Neabotics (Required for Plug-and-Play)
> **Important:** This custom firmware is **required** for the plug-and-play UAV motion stack functionality.

```bash
git clone --single-branch -b feature/diffgains_fix_servo_k https://github.com/Prisma-Drone-Team/Px4_hcore_autopilot.git PX4_neabotics --recursive
```

### 4. Build Docker Image
```bash
cd docker
docker build -t leo-img -f px4_humble_dockerfile.txt .
```
> **Note:** The container uses Gazebo Garden simulator. A Dockerfile for Gazebo Classic is also available but its integration into the stack is deprecated.

### 5. Run Container
```bash
./run_cnt.sh
```

## Development Configuration

### ROS2 Workspace Build
```bash
cd ros2_ws
source install/setup.bash
colcon build --cmake-args -DCMAKE_BUILD_TYPE=Release
source install/setup.bash
```

### Main Dependencies
```xml
<!-- Common package.xml -->
<depend>rclcpp</depend>
<depend>px4_msgs</depend>
<depend>nav_msgs</depend>
<depend>geometry_msgs</depend>
<depend>trajectory_msgs</depend>
<depend>tf2</depend>
<depend>tf2_ros</depend>
<depend>eigen3_cmake_module</depend>
```

## Usage in simulation with TMUX
```bash
cd ros2_ws
tmuxp load src/pkg/babyk_drone_manager/utils/simulation.yml
```
### Terminate the simulation
**Note:** kill the PX4 firmware and in the same terminal type: 
```bash
tmux kill-server
```
## Package Documentation

Each ROS2 package used in this system is documented in its own specific README:

- **drone_odometry2**: Odometry message conversion specifications
- **babyk_drone_manager**: Safety system and state monitoring

Refer to the README.md file in each package folder for technical details.

## Important Notes

**PX4 Firmware**: The PX4-Autopilot and PX4_neabotics firmwares must be downloaded separately and are used exclusively for SITL simulation. They are not required for deployment on real hardware.

**PX4_neabotics**: This firmware is specialized for tiltrotor drones and optimized for the Leonardo Drone Contest field, with specific improvements for tiltrotor flight dynamics.

## ROS2 Packages

### 📡 drone_odometry2
**Vehicle odometry publisher**

Converts PX4 status messages to standard ROS2 odometry.

**Main Topics:**
- **Subscriber:** `/fmu/out/vehicle_odometry` (px4_msgs/VehicleOdometry)
- **Publisher:** `/px4/odometry/out` (nav_msgs/Odometry)
- **Subscriber:** `/px4/trajectory_setpoint_enu` (enu set-points)
- **Publisher:** `/fmu/in/trajectory_setpoint` (px4 compatible ned set-points)

### 🛡️ babyk_drone_manager
**State management and safety**

Monitors drone status and implements safety functions. Implements the communication layer with the GCS.

**Main Topics:**
- **Subscriber:** `/seed_pdt_drone/command` (std_msgs/String) - new task primitive received
- **Publisher:** `/seed_pdt_drone/status` (std_msgs/String) - task status to GCS

