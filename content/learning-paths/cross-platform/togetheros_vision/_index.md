---
title: Build ROS 2 robotics vision pipelines on the RDK X5 with TogetheROS
description: Stream a USB camera from a D-Robotics RDK X5 to a Raspberry Pi with ROS 2, verify TogetheROS shared-memory transport on the RDK X5, and profile the vision pipeline on its Arm Cortex-A55 CPU.

minutes_to_complete: 120

who_is_this_for: This is an introductory topic for ROS 2 developers, robotics software engineers, and embedded Linux developers who want to build and observe a camera-based ROS 2 pipeline on the D-Robotics RDK X5, an Arm Cortex-A55 robotics developer board.

learning_objectives:
    - Identify the RDK X5 hardware, standard ROS 2 interfaces, and TogetheROS.Bot extensions used in a robotics vision workflow.
    - Verify the RDK X5 and Raspberry Pi environments, then stream a USB camera topic from the RDK X5 to a standard ROS 2 subscriber on the Raspberry Pi.
    - Run a TogetheROS shared-memory image example on the RDK X5 and describe what evidence supports its operation.
    - Collect repeatable frame-rate, CPU, and memory observations, and compare TogetheROS data sharing with standard image messages while stating which conditions limit the comparison.
    - Use Arm Performix to observe system utilization and system calls of a ROS 2 pipeline, and use perf to locate CPU hotspots.

prerequisites:
    - A D-Robotics RDK X5 running RDK OS 3.5 (Ubuntu 22.04) with [TogetheROS.Bot](https://developer.d-robotics.cc/rdk_doc/en/Robot_development/tros/) 2.5.2 for ROS 2 Humble.
    - A UVC USB camera (this Learning Path uses a Lenovo USB camera).
    - A Raspberry Pi (this Learning Path uses a Raspberry Pi 5) with Docker installed, on the same network as the RDK X5. ROS 2 Humble runs in a container.
    - A wired Ethernet connection for the RDK X5 is recommended for the camera streaming steps.
    - Basic familiarity with Linux and ROS 2 nodes, topics, and messages. If you need to install ROS 2, see the [ROS 2 install guide](/install-guides/ros2/).

author:
    - Odin Shen

generate_summary_faq: false
rerun_summary: false
rerun_faqs: false

shared_path: true
shared_between:
    - automotive
    - embedded-and-microcontrollers

### Tags
skilllevels: Introductory
subjects: Performance and Architecture
armips:
    - Cortex-A
tools_software_languages:
    - ROS 2
    - TogetheROS.Bot
    - Arm Performix
    - perf
operatingsystems:
    - Linux

further_reading:
    - resource:
        title: RDK X3/X5 documentation
        link: https://developer.d-robotics.cc/rdk_x_doc/en/
        type: documentation
    - resource:
        title: RDK X5 hardware introduction
        link: https://developer.d-robotics.cc/rdk_x_doc/en/Quick_start/hardware_introduction/rdk_x5
        type: documentation
    - resource:
        title: TogetheROS.Bot introduction
        link: https://developer.d-robotics.cc/rdk_doc/en/Robot_development/tros/
        type: documentation
    - resource:
        title: TogetheROS.Bot release notes
        link: https://developer.d-robotics.cc/tros_doc/en/quick_start/changelog
        type: documentation
    - resource:
        title: Find CPU hotspots with Arm Performix
        link: /learning-paths/servers-and-cloud-computing/cpu_hotspot_performix/
        type: website
    - resource:
        title: Install Docker
        link: /install-guides/docker/
        type: documentation
    - resource:
        title: ROS 2 Humble documentation
        link: https://docs.ros.org/en/humble/
        type: documentation

### FIXED, DO NOT MODIFY
# ================================================================================
weight: 1                       # _index.md always has weight of 1 to order correctly
layout: "learningpathall"       # All files under learning paths have this same wrapper
learning_path_main_page: "yes"  # This should be surfaced when looking for related content. Only set for _index.md of learning path content.
---
