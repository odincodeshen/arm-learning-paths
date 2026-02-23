---
title: Run Isaac Sim 6.0 and ROS 2 robotics simulation on Arm-based DGX Spark

minutes_to_complete: 90

who_is_this_for: This is an introductory topic for robotics developers who want to run NVIDIA Isaac Sim 6.0 with ROS 2 on the Arm-based DGX Spark, simulate a robot in a warehouse, drive it with keyboard teleoperation, and write a reactive obstacle-avoidance script that drives the robot autonomously.

learning_objectives:
    - Build NVIDIA Isaac Sim 6.0 from source on DGX Spark (Grace-Blackwell GB10, aarch64)
    - Install ROS 2 Jazzy and connect it to Isaac Sim through the ROS 2 Bridge
    - Teleoperate a simulated Nova Carter robot with keyboard commands
    - Visualize 3D LiDAR point clouds and camera feeds in RViz2
    - Write a minimal reactive obstacle-avoidance script that drives the robot autonomously using LiDAR data (a simple proof-of-concept, AI-based navigation will be covered in a future Learning Path)

prerequisites:
    - Access to an NVIDIA DGX Spark with at least 50 GB of free disk space
    - Familiarity with command-line interfaces and basic Linux operations
    - Basic knowledge of robotics concepts (sensors, coordinate frames)

lastmod: 2026-02-18

author: Asier Arranz

### Tags
skilllevels: Introductory
subjects: ML
armips:
    - Cortex-A
    - Cortex-X
operatingsystems:
    - Linux

### Cross-platform metadata only
shared_path: true
shared_between:
    - embedded-and-microcontrollers

tools_software_languages:
    - Python
    - ROS 2
    - Isaac Sim
    - Bash

further_reading:
    - resource:
        title: Isaac Sim 6.0 Early Developer Release notes
        link: https://docs.isaacsim.omniverse.nvidia.com/6.0.0/overview/release_notes.html
        type: documentation
    - resource:
        title: NVIDIA DGX Spark Playbooks, Isaac Sim and Isaac Lab
        link: https://github.com/NVIDIA/dgx-spark-playbooks/tree/main/nvidia/isaac
        type: documentation
    - resource:
        title: NVIDIA Isaac Sim documentation
        link: https://docs.isaacsim.omniverse.nvidia.com/latest/index.html
        type: documentation
    - resource:
        title: NVIDIA DGX Spark website
        link: https://www.nvidia.com/en-gb/products/workstations/dgx-spark/
        type: website

### FIXED, DO NOT MODIFY
# ================================================================================
weight: 1                       # _index.md always has weight of 1 to order correctly
layout: "learningpathall"       # All files under learning paths have this same wrapper
learning_path_main_page: "yes"  # This should be surfaced when looking for related content. Only set for _index.md of learning path content.
---
