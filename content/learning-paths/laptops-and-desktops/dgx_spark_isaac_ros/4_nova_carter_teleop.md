---
title: Drive a simulated robot and visualize its sensors
weight: 5

### FIXED, DO NOT MODIFY
layout: learningpathall
---

Time to put it all together. You will load a Nova Carter robot in Isaac Sim, drive it with your keyboard, and watch its LiDAR and cameras live in RViz2.

## Load the scene

In Isaac Sim:

1. Go to **Window > Examples > Robotics Examples**
2. Navigate to **ROS2 > Isaac ROS > Sample Scene**
3. Click **Load Sample Scene**
4. Press **Play** (▶)

You should see a Nova Carter robot sitting in a warehouse.

## Check the topics

In a new terminal:

```bash
source /opt/ros/jazzy/setup.bash
ros2 topic list
```

You should see something like:

```output
/chassis/imu
/chassis/odom
/clock
/cmd_vel
/front_3d_lidar/lidar_points
/front_stereo_camera/left/camera_info
/front_stereo_camera/left/image_rect_color
/front_stereo_camera/right/camera_info
/front_stereo_camera/right/image_rect_color
/tf
```

The robot's 3D LiDAR, stereo cameras, IMU, and odometry are all streaming over ROS 2.

## Teleoperate with the keyboard

Open a terminal and run:

```bash
source /opt/ros/jazzy/setup.bash
ros2 run teleop_twist_keyboard teleop_twist_keyboard
```

Controls:
- **i** forward
- **,** backward
- **j** / **l** turn left / right
- **k** stop
- **z** / **q** decrease / increase speed

Drive the robot around the warehouse. You will see it moving in the Isaac Sim viewport in real time.

## Visualize in RViz2

Open another terminal:

```bash
source /opt/ros/jazzy/setup.bash
rviz2
```

RViz2 opens with an empty 3D viewport and a "Global Status: Error" in the **Displays** panel on the left. That error appears because the default Fixed Frame is set to `map`, which doesn't exist in this scene. Here is how to fix it and start seeing sensor data:

### Fix the coordinate frame

In the **Displays** panel (left side of the window), expand **Global Options** at the top. Click the **Fixed Frame** dropdown, it currently says `map`, and change it to `odom`. The error disappears.

Why `odom`? RViz2 needs a fixed reference point to draw everything in 3D space. The `odom` frame is the spot where the robot started moving, think of it as the origin of its world. The default `map` frame doesn't exist here because this scene doesn't use SLAM or Nav2 (which are the systems that normally publish a `map` frame). Since this is a simple demo, `odom` is the only fixed reference available.

### Add the LiDAR point cloud

1. Click the **Add** button at the bottom of the Displays panel.
2. In the dialog that appears, select the **By topic** tab.
3. Expand `/front_3d_lidar` → `/lidar_points` and select **PointCloud2**. Click **OK**.
4. The point cloud appears in the viewport, but the points may be hard to see. In the Displays panel, expand the new **PointCloud2** entry and change **Size (m)** from `0.01` to `0.05` so the dots are visible.

### Add the front camera feed (optional)

1. Click **Add** → **By topic** tab.
2. Expand `/front_stereo_camera` → `/left` → `/image_rect_color` and select **Camera**. Click **OK**.

A small camera preview window appears inside the viewport showing what the robot's front camera sees.

### Drive and watch

Switch to the teleop terminal and drive the robot around. In RViz2, you will see the 3D LiDAR point cloud update in real time. Walls, shelves, and obstacles appear as clusters of dots around the robot as it moves through the warehouse.

In the next section you will write a simple Python script that uses this LiDAR data to drive the robot autonomously through the warehouse.
