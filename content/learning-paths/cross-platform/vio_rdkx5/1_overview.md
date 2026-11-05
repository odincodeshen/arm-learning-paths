---
title: Understand stereo visual-inertial odometry and VIMS on the RDK X5
description: Learn what visual-inertial odometry computes, how the D-Robotics VIMS solution is built on the RDK X5, which of its parts run on the Arm CPU and which run on the BPU, and why this Learning Path uses a replayed dataset.
weight: 2

layout: "learningpathall"
---

<!--
DRAFT (2026-10-30). Not reviewed.
Sources for the statements on this page: evidence/run-20261024-01 (VIMS packages, launch files, linked libraries),
run-20261025-01 (VIO BPU 0%, StereoNet BPU 60-75% with sample images), run-20261026-01, D-ROBOTICS_LP2_REPLY_v1.md.
Architecture figure: images/architecture.png (rendered from images/architecture.svg, 2026-10-31). CPU/BPU marks follow
the table on this page; the camera path is drawn dashed because this Learning Path doesn't run it.
-->

## What you will build

In this Learning Path, you run a visual-inertial odometry (VIO) workload on the D-Robotics RDK X5, a robotics developer board built around eight Arm Cortex-A55 cores, and you measure what it costs on the Arm CPU.

You use VIMS, a vision navigation solution from D-Robotics for the RDK X5. You install it on a separate RDK OS image. Instead of a camera, you feed its VIO node with a public dataset, so you can repeat every step without special hardware and compare your trajectory with ground truth. Then you measure the CPU, memory, and output rate of the VIO node, use `perf` to find where its CPU time goes, and tune the workload so that it keeps up with its input on the Arm CPU.

![Architecture diagram. On the workstation, the TUM-VI room1 sequence is converted with Python into a ROS 2 bag with the left image on top, and copied to the RDK X5 with scp. On the RDK X5, the bag is replayed with ros2 bag play, or read by the node itself in offline mode, and publishes the combined stereo image and the IMU topic to drobotics_vio. drobotics_vio runs on the Arm CPU only. Its front end does feature detection and KLT tracking with OpenCV, and its back end does IMU propagation and the MSCKF update with Eigen. Its outputs go back to the workstation: pathimu and trackhist to Foxglove, and poseimu to a trajectory error comparison with the motion capture ground truth. A dashed lower row shows the camera path in VIMS that this Learning Path doesn't use: a GS130WI camera feeds mipi_cam, which publishes the same topics; StereoNet computes depth on the BPU; RTAB-Map builds a 3D map on the Arm CPU from the depth and the VIO poses.#center](images/architecture.png "The VIO workflow of this Learning Path, and the camera path of VIMS that it doesn't use")

## What visual-inertial odometry computes

A robot needs to know how it moves. Visual-inertial odometry estimates the motion of a sensor rig from two kinds of data:

- **Camera images.** The VIO finds features, such as corners, in each image and follows them from frame to frame. With a stereo camera, it also matches features between the left and right images, which gives their distance.
- **IMU samples.** An inertial measurement unit (IMU) measures acceleration and rotation rate, typically hundreds of times per second. The VIO integrates these measurements between camera frames.

The VIO combines both in a filter and estimates the position and orientation of the rig at each frame. The result is a trajectory: a sequence of poses over time.

A trajectory isn't a map. To build a map, a separate mapping system, such as RTAB-Map, combines the poses with depth measurements of the surroundings over time. In VIMS the depth comes from a stereo neural network called StereoNet. This Learning Path runs and measures the VIO. Mapping needs a camera and isn't part of the dataset workflow.

## About the RDK X5

The RDK X5 is part of the D-Robotics RDK (Robot Developer Kit) family. The board features that this Learning Path uses:

| RDK X5 feature | How this Learning Path uses it |
| --- | --- |
| 8-core Arm Cortex-A55 CPU, up to 1.5 GHz | Runs ROS 2 and the VIO node, which is the workload you measure |
| Neural network accelerator, which D-Robotics calls the BPU | Not used by the VIO node. In VIMS it runs StereoNet depth estimation, which needs a camera. |
| 8 GB memory | Holds the VIO node and the replayed data |
| microSD card slot | Holds a separate RDK OS image for VIMS |
| Gigabit Ethernet | Connects the RDK X5 to your workstation and to the internet for the installation |
| RDK OS 3.5 with TogetheROS.Bot 2.5.2 | Provides Ubuntu 22.04, ROS 2 Humble, and the TogetheROS packages that VIMS builds on |

The board also has two MIPI camera ports for stereo cameras such as the RDK Stereo Camera GS130WI, which this Learning Path doesn't need. For the full hardware description, see the [RDK X5 hardware introduction](https://developer.d-robotics.cc/rdk_x_doc/en/Quick_start/hardware_introduction/rdk_x5).

## About VIMS

VIMS is a separate software package that D-Robotics provides on top of TogetheROS.Bot. It installs into `/userdata/vims/install` and adds its own ROS 2 packages, some of which replace TogetheROS packages of the same name when you source its environment. This Learning Path uses VIMS 0.0.6. D-Robotics describes it as a validation release, and its test package limits each run to three hours. The algorithm source of `drobotics_vio` isn't public at the time of writing, so this Learning Path uses the binary package and the public documentation. See the [VIMS user manual](https://d-robotics.github.io/tros_vims_doc/) for the complete solution.

The VIMS parts that matter here:

| Part | What it does | Where it runs |
| --- | --- | --- |
| `drobotics_vio` | Stereo visual-inertial odometry. It is based on [OpenVINS](https://docs.openvins.com/), an open-source filter-based VIO. | Arm CPU. It doesn't link any BPU library, and it showed 0% BPU load in the measurements of this Learning Path. |
| `hobot_stereonet` (StereoNet) | Depth from the two camera images with a neural network | BPU for the network, Arm CPU for pre- and postprocessing |
| RTAB-Map (`rtabmap_slam`) | Mapping, loop closure, and map storage | Arm CPU in the default configuration |
| `mipi_cam` | Driver for MIPI cameras such as the GS130WI, including its IMU | Camera interface and image processing hardware, with Arm CPU work |

This Learning Path uses only `drobotics_vio`. Because it runs entirely on the CPU, it's a good example of a CPU-bound robotics workload. When you read the CPU numbers later, keep in mind that they describe Arm CPU work only. A VIO BPU load of 0% doesn't mean that the rest of VIMS doesn't use the BPU.

## Why this Learning Path uses a replayed dataset

The VIO node subscribes to a stereo image and an IMU topic. It doesn't care whether a camera or a recording publishes them. This Learning Path uses the `room1` sequence of the public TUM visual-inertial dataset (TUM-VI), for three reasons:

- **You don't need a camera.** You need only the RDK X5, a microSD card, and a workstation.
- **Every run gets the same input.** You can repeat a measurement and compare it with the numbers on these pages.
- **The dataset includes ground truth.** A motion-capture system recorded the true pose during the sequence, so you can measure the error of the VIO trajectory.

The dataset was recorded with different cameras from the ones that VIMS targets. Results with the dataset describe this dataset and this configuration. They don't describe the performance of a VIMS camera such as the GS130WI.

## What you will do

| Page | What you do | Where |
| --- | --- | --- |
| Prepare the environment | Install VIMS 0.0.6 on a separate RDK OS 3.5.0 image, and record what changes | RDK X5 |
| Prepare the replay data | Download TUM-VI `room1`, convert it into a ROS 2 bag, and build a VIMS configuration | Workstation, then RDK X5 |
| Run VIO from the dataset | Run the VIO node from the bag, record the trajectory, and compare it with ground truth | RDK X5 and workstation |
| Profile the VIO workload | Measure CPU, memory, and output rate under fixed conditions, and find the CPU hotspots with `perf` | RDK X5 |
| Tune the VIO for the CPU budget | Change the input frame rate and the number of features one at a time, and find a setting that keeps up in real time | Workstation and RDK X5 |

## Summary

- Visual-inertial odometry estimates the motion of a camera and IMU rig as a trajectory. A map needs more: depth measurements and a mapping system.
- VIMS is a D-Robotics solution for the RDK X5. Its VIO node is based on OpenVINS and runs on the Arm Cortex-A55 CPU. Its depth estimation runs on the BPU.
- This Learning Path uses VIMS 0.0.6, a validation release, with a public dataset instead of a camera. The results describe the dataset and the configuration, not a camera product.
- CPU numbers in this Learning Path describe Arm CPU work only.
