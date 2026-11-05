---
title: Run and profile stereo visual-inertial odometry with VIMS on the D-Robotics RDK X5

description: Install the D-Robotics VIMS solution on an RDK X5, run its stereo visual-inertial odometry with a replayed public dataset, evaluate the trajectory against ground truth, and profile and tune the workload on the Arm Cortex-A55 CPU.

minutes_to_complete: 180

who_is_this_for: This is an advanced topic for robotics software engineers and embedded Linux developers who know ROS 2 and want to run and measure a visual-inertial odometry workload on an Arm-based robotics developer board, the D-Robotics RDK X5.

learning_objectives:
    - Explain how stereo images and IMU data flow through visual-inertial odometry (VIO), and which parts of the D-Robotics VIMS solution run on the Arm CPU and which run on the BPU.
    - Install VIMS on a separate RDK OS image and identify which system settings it changes.
    - Convert a public stereo-IMU dataset into a ROS 2 bag and a VIMS configuration, and run VIO from it without a camera.
    - Evaluate the VIO trajectory against the dataset ground truth, and state what the result does and doesn't describe.
    - Measure the CPU, memory, and output rate of the VIO node under fixed conditions, and use perf to find where its CPU time goes.
    - Change the input frame rate and the number of tracked features one at a time, and find a setting in which the VIO node keeps up in real time on the Arm CPU.

prerequisites:
    - A D-Robotics RDK X5 with 8 GB of memory.
    - A microSD card of at least 32 GB for a separate RDK OS 3.5.0 image. The VIMS installation changes system packages, the kernel, and the device tree, so don't use the card of another project.
    - A wired network connection with internet access for the RDK X5.
    - A workstation with macOS or Linux, Python 3.9 or later, SSH access to the RDK X5, and about 6 GB of free disk space.
    - Basic familiarity with Linux and ROS 2 nodes, topics, and bags. If you need to install ROS 2 on another system, see the [ROS 2 install guide](/install-guides/ros2/).

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
skilllevels: Advanced
subjects: Performance and Architecture
armips:
    - Cortex-A
tools_software_languages:
    - ROS 2
    - TogetheROS.Bot
    - VIMS
    - OpenVINS
    - perf
    - Python
operatingsystems:
    - Linux

further_reading:
    - resource:
        title: VIMS user manual
        link: https://d-robotics.github.io/tros_vims_doc/
        type: documentation
    - resource:
        title: VIMS documentation source
        link: https://github.com/D-Robotics/tros_vims_doc
        type: website
    - resource:
        title: TogetheROS.Bot introduction
        link: https://developer.d-robotics.cc/rdk_doc/en/Robot_development/tros/
        type: documentation
    - resource:
        title: TUM visual-inertial dataset
        link: https://cvg.cit.tum.de/data/datasets/visual-inertial-dataset
        type: website
    - resource:
        title: OpenVINS documentation
        link: https://docs.openvins.com/
        type: documentation
    - resource:
        title: Build ROS 2 robotics vision pipelines on the D-Robotics RDK X5 with TogetheROS
        link: /learning-paths/cross-platform/ros2-vision-togetheros-x5/
        type: website

### FIXED, DO NOT MODIFY
# ================================================================================
weight: 1                       # _index.md always has weight of 1 to order correctly
layout: "learningpathall"       # All files under learning paths have this same wrapper
learning_path_main_page: "yes"  # This should be surfaced when looking for related content. Only set for _index.md of learning path content.
---

<!--
DRAFT (2026-10-30). Not reviewed.
- Title and objectives cover the dataset main line (pages 1-6) only. When the GS130WI pages (7-8) are verified,
  add an objective such as "Run VIO with an RDK Stereo Camera GS130WI and build a 3D map", and use the title rule in
  LP2_HYBRID_OUTLINE.md section 1. The mapping title requires the mapping gate.
- 2026-10-31: page 6 (tuning) added: +30 min, objective 6, description "profile and tune".
- minutes_to_complete: estimate (install about 60 min, data about 20 min, VIO about 20 min, profiling about 30 min,
  overview and reading). Re-measure when page 2 is run as written.
- skilllevels Advanced: installs kernel packages and profiles a workload; confirm with the Learning Path reviewers.
- VIMS 0.0.6 is a validation release. D-Robotics reply v2 (2026-11-04): pin 0.0.6, no release date, keep the trial
  limit note; drobotics_vio source isn't public, so don't label tros_vims_doc as source. Update if that changes.
-->
