---
title: Run ROS 2 vision pipelines on the RDK X5 with TogetheROS

description: Use TogetheROS.Bot on the D-Robotics RDK X5 to publish USB camera images, run a local image-processing example, verify data sharing, and observe resource use.

minutes_to_complete: 120

who_is_this_for: This is an introductory topic for ROS 2 developers, robotics software engineers, and embedded Linux developers who want to run and inspect TogetheROS.Bot vision workflows on the D-Robotics RDK X5.

learning_objectives:
    - Identify the TogetheROS.Bot packages and RDK X5 components used to capture, publish, share, and process images.
    - Publish a USB camera stream from the RDK X5, receive it on a Raspberry Pi, and view the live video in a browser.
    - Run a local TogetheROS image-processing example and verify the Fast DDS data-sharing step between its publisher and subscribers.
    - Use Arm Performix to observe CPU, memory, disk, and network use while the local vision pipeline runs.

prerequisites:
    - A D-Robotics RDK X5 running RDK OS 3.5 (Ubuntu 22.04) with [TogetheROS.Bot](https://developer.d-robotics.cc/rdk_doc/en/Robot_development/tros/) 2.5.2 for ROS 2 Humble.
    - A USB camera.
    - A Raspberry Pi with Docker installed, on the same network as the RDK X5.
    - A computer with an SSH client and a web browser to view the stream served by the Pi.
    - Basic familiarity with Linux and ROS 2 nodes, topics, and messages.

author:
    - Odin Shen
    - Kwashie Andoh
    - Habib Ogunbanwo

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
        title: Install ROS 2
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
