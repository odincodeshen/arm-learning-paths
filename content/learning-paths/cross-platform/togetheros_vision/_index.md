---
title: Run ROS 2 vision pipelines on the RDK X5 with TogetheROS

description: Use TogetheROS.Bot on the D-Robotics RDK X5 to publish USB camera images, run a local image-processing example, verify data sharing, and profile CPU work.

minutes_to_complete: 120

who_is_this_for: This is an introductory topic for ROS 2 developers, robotics software engineers, and embedded Linux developers who want to run and inspect TogetheROS.Bot vision workflows on the D-Robotics RDK X5.

learning_objectives:
    - Identify the TogetheROS.Bot packages and RDK X5 components used to capture, publish, share, and process images.
    - Configure `hobot_usb_cam` to publish a USB camera topic and verify that a ROS 2 receiver on a Raspberry Pi receives the images.
    - Run a local TogetheROS image-processing example and verify the Fast DDS data-sharing step between its publisher and subscribers.
    - Measure frame rate, CPU, and memory for the local example in two message configurations, and explain the limits of that comparison.
    - Use Arm Performix to observe system utilization and system calls, and use perf to locate CPU hotspots in the vision workload.

prerequisites:
    - A D-Robotics RDK X5 running RDK OS 3.5 (Ubuntu 22.04) with [TogetheROS.Bot](https://developer.d-robotics.cc/rdk_doc/en/Robot_development/tros/) 2.5.2 for ROS 2 Humble.
    - A UVC USB camera (this Learning Path uses a Lenovo USB camera).
    - A Raspberry Pi (this Learning Path uses a Raspberry Pi 5) with Docker installed, on the same network as the RDK X5. It runs a ROS 2 Humble container to receive the camera stream.
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
        title: Install Docker
        link: /install-guides/ros2/
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
