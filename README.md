# Level 1: ROS2 Navigation Assignment - Amin Ahmed G

Fixed the bugs in the starter code and added a `testbed_navigation` package that brings up map_server, AMCL and the Nav2 servers from my own launch files, without nav2_bringup.

## Environment

- ROS 2 Humble, Gazebo 11 (classic), RViz2
- My laptop is on Ubuntu 24.04 with Jazzy, so I ran everything in an `osrf/ros:humble-desktop` Docker container
- Used CycloneDDS since the FAQ suggested it, the default FastDDS should work too

## Setup

```bash
mkdir -p ~/assignment_ws/src && cd ~/assignment_ws/src
git clone -b testbed-navigation https://github.com/Amin-Ahmed-G/level01_ros_assignment.git
cd ~/assignment_ws
sudo apt update
rosdep install --from-paths src --ignore-src -y
colcon build --symlink-install
source install/setup.bash
```

## How to run

Separate terminal for each, in this order:

```bash
# terminal 1 - simulation (gui:=false runs gazebo headless)
ros2 launch testbed_bringup testbed_full_bringup.launch.py \
  rvizconfig:=$(ros2 pkg prefix testbed_navigation)/share/testbed_navigation/rviz/nav.rviz
# terminal 2 - map
ros2 launch testbed_navigation map_loader.launch.py
# terminal 3 - localization
ros2 launch testbed_navigation localization.launch.py
# terminal 4 - navigation
ros2 launch testbed_navigation navigation.launch.py
```

Then give a goal with **2D Goal Pose** in RViz. The initial pose is already set to the spawn point (0, 5, 0), if the particles look off just use **2D Pose Estimate**. If you restart the sim, restart all four (see Challenges).

## Package layout

```
testbed_navigation/
├── config/
│   ├── amcl_params.yaml
│   └── nav2_params.yaml
├── launch/
│   ├── map_loader.launch.py
│   ├── localization.launch.py
│   └── navigation.launch.py
└── rviz/
    └── nav.rviz
```

## Contact Info
 - Name: Amin Ahmed G
 - Contact number: +91 8122241705
 - Email Address: aminahmedg2005@gmail.com
