---
title: Overview
weight: 2

### FIXED, DO NOT MODIFY
layout: learningpathall
---

The NVIDIA DGX Spark is the first compact AI workstation that can run Isaac Sim natively on Arm. In this Learning Path you will build Isaac Sim 6.0 (Early Developer Release) from source, install ROS 2 Jazzy, simulate a [Nova Carter](https://robotics.segway.com/nova-carter/) robot in a warehouse, and drive it around while watching its LiDAR and cameras live in RViz2.

![Nova Carter autonomous navigation in Isaac Sim#center](carter.gif "Nova Carter navigating autonomously in a simulated warehouse on DGX Spark.")

## What you will need

| Requirement | Details |
|-------------|---------|
| Hardware | NVIDIA DGX Spark (Grace-Blackwell GB10) |
| Storage | At least 50 GB free |
| OS | DGX OS (Ubuntu 24.04) |
| GPU driver | 580.x or later |
| CUDA | 13.0 or later |
| Network | Internet access to clone repos and install packages |

Verify your system is ready:

```bash
nvidia-smi
```

You should see the GB10 GPU with CUDA 13.0+.

## What you will build

Isaac Sim handles all the physics simulation and rendering on the Blackwell GPU. A built-in ROS 2 Bridge connects Isaac Sim to the ROS 2 ecosystem, publishing sensor data (3D LiDAR point clouds, stereo camera feeds, IMU, odometry) and accepting velocity commands, the same topics you would get from a real Nova Carter robot. On the ROS 2 side, you will use standard tools like `teleop_twist_keyboard` to drive the robot and RViz2 to visualize what it sees. By the end, you will also write a simple Python script that reads the LiDAR data and drives the robot autonomously through a warehouse.
