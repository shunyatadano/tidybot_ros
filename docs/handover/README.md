# TidyBot ROS 2 Simulation Environment - Handover Document

## Overview

This document describes the TidyBot simulation environment for ROS 2, designed for smooth handover between simulation and hardware teams.

## Changelog

### 2026-03-15: LiDAR Support Added

**Changes:**
- Added GPU LiDAR sensor to simulation (RPLiDAR A1M8-R6 equivalent)
- Position: Base front center (xyz=0.3, 0, 0.15 relative to base_link)
- Topics: `/scan` (LaserScan), `/scan/points` (PointCloud2)

**Modified Files:**
- `urdf/tidybot.xacro` - Added lidar argument
- `urdf/bot.xacro` - Updated load_bot macro
- `urdf/base/tidybot++/base_kinova_macro.xacro` - Added lidar_link and lidar_joint
- `urdf/tidybot.gazebo.xacro` - Added GPU LiDAR plugin
- `launch/launch_sim_robot.launch.py` - Added lidar argument and topic bridges

**Branch:** `sim/lidar`

## Environment

| Component | Version |
|-----------|---------|
| OS | Ubuntu 24.04 (WSL2) |
| ROS | ROS 2 Jazzy Jalisco |
| Simulator | Gazebo Harmonic |
| Docker | Docker Engine CE 29.3.0 + NVIDIA Container Toolkit |

## Quick Start

### Option 1: Use Pre-built Docker Image (Recommended)

1. Download the Docker image tar file:
   - **File:** `tidybot_platform_sim-lidar.tar.gz` (3.4GB)
   - **Shared via:** Google Drive / OneDrive (contact maintainer)

2. Load the image:
   ```bash
   docker load < tidybot_platform_sim-lidar.tar.gz
   docker tag tidybot_platform:latest shunyatadano/tidybot_platform:sim-lidar
   ```

3. Clone the repository:
   ```bash
   git clone -b sim/lidar https://github.com/shunyatadano/tidybot_ros.git
   cd tidybot_ros
   ```

4. Start container:
   ```bash
   ./docker/tidybot/run.sh
   ```

### Option 2: Build from Source

```bash
git clone -b sim/lidar https://github.com/shunyatadano/tidybot_ros.git
cd tidybot_ros
./docker/tidybot/build.sh
./docker/tidybot/run.sh
```

### Simulation Launch

```bash
# Inside Docker container
cd ~/Documents/tidybot_platform
source /opt/ros/jazzy/setup.bash
source install/setup.bash

# Launch simulation with LiDAR
ros2 launch tidybot_description launch_sim_robot.launch.py base_mode:=velocity

# Launch without LiDAR
ros2 launch tidybot_description launch_sim_robot.launch.py base_mode:=velocity lidar:=false
```

**Note:** If .bashrc is already configured (after first setup), the `source` commands are automatic.

### Launch Arguments

| Argument | Default | Description |
|----------|---------|-------------|
| `base_mode` | `position` | Base control mode (`position` or `velocity`) |
| `lidar` | `true` | Enable LiDAR sensor |
| `vision` | `true` | Enable RGB-D cameras |
| `use_rviz` | `true` | Launch RViz for visualization |
| `world` | `empty` | Gazebo world (`empty`, `office`, `warehouse`, `fetch_coke`, `fetch_cube`) |

## Sensor Configuration

### RGB-D Cameras

| Camera | Frame ID | Topics |
|--------|----------|--------|
| Base Camera | `base_camera_link` | `/tidybot/camera_base/color/raw`, `/tidybot/camera_base/depth/raw` |
| Arm Camera | `arm_camera_link` | `/tidybot/camera_wrist/color/raw`, `/tidybot/camera_wrist/depth/raw` |

### LiDAR (RPLiDAR A1M8-R6 equivalent)

| Property | Value |
|----------|-------|
| Frame ID | `lidar_link` |
| Type | GPU LiDAR |
| Update Rate | 10 Hz |
| Range | 0.15m - 12.0m |
| Horizontal Samples | 360 |
| Field of View | 360 degrees |

**Topics:**
- `/scan` - `sensor_msgs/msg/LaserScan`
- `/scan/points` - `sensor_msgs/msg/PointCloud2`

**Position:** Base front center (xyz=0.3, 0, 0.15 relative to base_link)

## Topic Reference

### Sensor Topics

```
/scan                              # LaserScan from LiDAR
/scan/points                       # PointCloud2 from LiDAR
/tidybot/camera_base/color/raw     # Base camera RGB
/tidybot/camera_base/depth/raw     # Base camera Depth
/tidybot/camera_wrist/color/raw    # Arm camera RGB
/tidybot/camera_wrist/depth/raw    # Arm camera Depth
```

### Control Topics

```
/joint_states                      # All joint states
/tidybot_base_pos_controller/...   # Base position controller (base_mode:=position)
/tidybot_base_vel_controller/...   # Base velocity controller (base_mode:=velocity)
/gen3_7dof_controller/...          # Arm controller
/robotiq_2f_85_controller/...      # Gripper controller
```

### TF Frames

Key frames in the robot:
- `world` - World origin
- `base` - Mobile base
- `base_camera_link` - Base RGB-D camera
- `arm_camera_link` - Wrist RGB-D camera
- `lidar_link` - LiDAR sensor
- `tool_frame` - End effector

Generate TF tree:
```bash
ros2 run tf2_tools view_frames
```

## File Structure

```
src/tidybot_description/
├── urdf/
│   ├── tidybot.xacro              # Main robot description
│   ├── bot.xacro                  # Robot assembly macro
│   ├── tidybot.gazebo.xacro       # Gazebo plugins (cameras, LiDAR)
│   ├── base/tidybot++/
│   │   └── base_kinova_macro.xacro # Base + LiDAR links
│   └── arms/gen3_7dof/            # Kinova Gen3 7-DOF arm
├── launch/
│   └── launch_sim_robot.launch.py # Main simulation launch
└── config/
    └── tidybot_controllers.yaml   # Controller configuration
```

## Known Limitations

1. **CANivore Support**: WSL2 kernel module limitations prevent canivore-usb installation. Real hardware CAN communication requires native Linux or alternative setup.

2. **LiDAR Update Rate**: In simulation, the LiDAR may run slower than configured 10Hz depending on system load.

3. **GPU Requirements**: GPU LiDAR requires NVIDIA GPU with proper driver installation. Falls back to CPU LiDAR if GPU unavailable.

## Branch Strategy

```
main                    # Stable release (sync with upstream/main)
├── sim/develop        # Simulation development
├── sim/lidar          # LiDAR feature (current)
└── hardware/develop   # Hardware-specific changes
```

## Distribution Links

| Resource | URL |
|----------|-----|
| **Repository (sim/lidar branch)** | https://github.com/shunyatadano/tidybot_ros/tree/sim/lidar |
| **Upstream Repository** | https://github.com/roahmlab/tidybot_platform |
| **Docker Image (tar)** | `tidybot_platform_sim-lidar.tar.gz` (3.4GB) - Contact maintainer for link |

### Creating Docker Image tar File

If you need to share the Docker image:

```bash
# Create tar file
docker save tidybot_platform:latest | gzip > tidybot_platform_sim-lidar.tar.gz

# Load tar file (on receiving side)
docker load < tidybot_platform_sim-lidar.tar.gz
```

## Troubleshooting

### Container won't start with GUI

```bash
# Ensure X11 forwarding is working
xhost +local:docker

# Check DISPLAY variable
echo $DISPLAY
```

### Package not found error

```bash
# Inside container, ensure workspace is sourced
cd ~/Documents/tidybot_platform
source /opt/ros/jazzy/setup.bash
source install/setup.bash

# If still not found, rebuild
colcon build --packages-select tidybot_description
source install/setup.bash
```

### LiDAR topics not publishing

1. Check if `lidar:=true` (default)
2. Verify GPU support: `nvidia-smi`
3. Check Gazebo logs for sensor errors

### Controller not spawning

```bash
# Check controller manager
ros2 control list_controllers

# Manually spawn controller
ros2 run controller_manager spawner <controller_name>
```

## Contact

- **Maintainer:** Shunya Tadano
- **Repository:** https://github.com/shunyatadano/tidybot_ros
- **Upstream:** https://github.com/roahmlab/tidybot_platform
