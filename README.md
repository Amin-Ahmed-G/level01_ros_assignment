# Level 1: ROS2 Navigation Assignment - Amin Ahmed G

In this assignment I fixed the bugs I found in the starter code and wrote my own `testbed_navigation` package. My launch files bring up map_server, AMCL and the Nav2 servers one by one, without using nav2_bringup.

## Environment

- ROS 2 Humble, Gazebo 11 (classic), RViz2
- My laptop runs Ubuntu 24.04 with Jazzy, so I did all my work inside an `osrf/ros:humble-desktop` Docker container
- I used CycloneDDS because the FAQ suggested it, but I think the default FastDDS should work too

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

Then give a goal with 2D Goal Pose in RViz. The initial pose is already set to the spawn point (0, 5, 0), if the particles look off just use 2D Pose Estimate. If you restart the sim, restart all four (see Challenges).

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

## Approach

I decided to split my setup into three launch files, each with its own lifecycle manager. My thinking was that I could test one stage at a time: bring up the map and check it in RViz, then add AMCL, then Nav2. It helped a lot, because whenever something broke I knew which part to look at.

### Map loader

I used `nav2_map_server` with `lifecycle_manager_map`. I load the map from the installed testbed_bringup share folder instead of a source path, so my launch file works from any workspace. I also added a `map` argument in case someone wants to try a different map (`map:=<path>`).

### Localization

For localization I went with AMCL using `nav2_amcl::DifferentialMotionModel`, since my robot is diff drive and can't move sideways. For the laser model I kept `likelihood_field`, which is the Nav2 default and I thought it was fine for a small indoor map. My base frame is `base_footprint`.

- I set `laser_max_range` to 12 m so it matches the lidar after my range fix
- I kept `max_beams` at 60 (the lidar gives 300). I felt that was enough for AMCL and it saves CPU
- I used 500 to 2000 particles
- I set the initial pose to the spawn point (0, 5, 0) so the robot is localized from the start

To check my localization I compared AMCL's pose with the robot's pose in Gazebo. My error was about 7 cm.

### Navigation

- Planner: I picked NavFn (Dijkstra). The map is small and I only needed point-to-point goals, so I didn't think Smac or anything fancier was needed.
- Controller: I chose Regulated Pure Pursuit because it's made for diff drive and slows down by itself in tight turns and near walls. I set my `desired_linear_vel` to 0.3 m/s. I considered DWB, but it has a lot more params to tune.
- velocity_smoother: I put it between my controller and the robot so the velocity doesn't jump. My controller publishes on `cmd_vel_nav`, and the smoother takes that and publishes to `cmd_vel`.
- behavior_server: I added spin, backup and wait so my robot has something to try when it gets stuck.
- bt_navigator: I used the default Nav2 tree. Humble needs the BT plugins listed in `plugin_lib_names`, so I listed only the 13 that the default trees use instead of copying the full list.

### Costmaps

- I set `robot_radius` to 0.25 m. I measured my robot from the STL meshes, it's about 0.35 x 0.41 m.
- I used 0.45 m inflation so my planned paths stay away from the walls.
- I set obstacle / raytrace range to 11.5 / 12 m to match the lidar. I kept raytrace a bit longer so old obstacles get cleared.
- My local costmap is a 4x4 m rolling window in `odom`. I chose `odom` because it's smooth, while `map` can jump when AMCL corrects.
- My global costmap uses the static layer from the map, in the `map` frame.

## Bugs found

I found 6 bugs plus 5 smaller things that I thought were wrong. I made each fix its own commit, and in [BUGS.txt](BUGS.txt) I wrote down the file, problem, fix and how I checked each one.

1. `ament_package` was missing `()` in testbed_description's CMakeLists.txt, so nothing built. I added the brackets.
2. The LiDAR max range was 1.5 m. The nearest wall is ~3.4 m away, so my scan was basically empty. I changed it to 12 m.
3. The map yaml pointed to `wrong_path_testbed_world.pgm`. I changed it to the real file, `testbed_world.pgm`.
4. testbed_bringup didn't install `maps/`, so my launch file couldn't find the map in the share folder. I added it to `install()`.
5. `base_footprint` was about 6 cm away from the real rotation centre. I moved it to the middle of the wheel axle.
6. Some `exec_depend`s were missing (xacro, gazebo_ros, joint_state_publisher etc), so rosdep wouldn't install them on a fresh machine. I added them to the package.xml files.

I also removed the old ROS1 `libgazebo_ros_control.so` plugin and a stray `>` in the IMU block, turned on `use_sim_time` for robot_state_publisher, installed the gazebo `models/` folder, and changed `Gazebo/Silver` to `Gazebo/Grey` because Silver doesn't exist in Gazebo 11.

## Challenges

### Particle cloud not showing in RViz

My AMCL was running, but no particles showed up. I found out it was a QoS problem: AMCL publishes `/particle_cloud` as best effort and my RViz display was set to reliable. I switched the display to Best Effort and it showed up.

### bt_navigator wouldn't activate

I only needed navigate_to_pose, but bt_navigator loads the navigate_through_poses tree as well, and that one needs `compute_path_through_poses` and `remove_passed_goals`. It stayed inactive until I added those two to my `plugin_lib_names`.

### base_footprint in the wrong place

When I turned the robot in place with teleop, I saw base_footprint moving in a circle of about 6 cm in RViz instead of staying on the spot. That told me it wasn't on the rotation axis. I moved the joint to the midpoint of the two wheels and now it stays put (drift went from ~5 cm to ~0.1 cm).

### Restarting only Gazebo breaks Nav2

When I restarted just the sim, the clock jumped back, AMCL still had the old pose and my controller failed with "patience exceeded". After that I always restarted all four launches in order.

### Docker

Since I had to run Humble in Docker, I mixed up host and container paths a few times at the start. I also tried passing my GPU into the container. It rendered, but the windows were black under Wayland, so I went back to software rendering and mostly ran Gazebo with `gui:=false`.

## Contact Info
 - Name: Amin Ahmed G
 - Contact number: +91 8122241705
 - Email Address: aminahmedg2005@gmail.com
