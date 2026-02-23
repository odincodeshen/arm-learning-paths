---
title: Scale Robotics Reinforcement Learning Workflows to Manipulation and Multi-Agent Tasks with IsaacLab

minutes_to_complete: 90

who_is_this_for: This advanced topic is intended for robotics developers, simulation engineers, and AI researchers who want to run high-fidelity robotic simulations and reinforcement learning (RL) pipelines using Isaac Sim and Isaac Lab on Arm-based NVIDIA DGX Spark systems powered by the Grace–Blackwell (GB10) architecture.

learning_objectives:
    - Describe the roles of Isaac Sim and IsaacLab, and explain how DGX Spark accelerates robotic simulation and learning workloads
    - Train a reinforcement learning policy for the Unitree H1 humanoid robot using Isaac Lab and the RSL-RL interface
    - Scale reinforcement learning workloads across multiple tasks and multi-agent environments using Isaac Lab

prerequisites:
    - Access to an NVIDIA DGX Spark system with at least 50 GB of free disk space
    - Familiarity with command-line interfaces and basic Linux operations
    - Experience with Python scripting and virtual environments
    - Basic understanding of reinforcement learning concepts (rewards, policies, episodes)
    - Experience building software from source using CMake and make

author:
    - Johnny Nunez
    - Odin Shen

### Tags
skilllevels: Advanced
subjects: ML
armips:
    - Cortex-X
    - Cortex-A
tools_software_languages:
    - Python
    - Bash
    - IsaacSim
    - IsaacLab
operatingsystems:
    - Linux

further_reading:
    - resource:
        title: Isaac Sim Documentation
        link: https://docs.isaacsim.omniverse.nvidia.com/latest/index.html
        type: documentation
    - resource:
        title: Isaac Lab Documentation
        link: https://isaac-sim.github.io/IsaacLab/main/index.html
        type: documentation
    - resource:
        title: NVIDIA DGX Spark Playbooks
        link: https://github.com/NVIDIA/dgx-spark-playbooks
        type: documentation
    - resource:
        title: Isaac Lab Available Environments
        link: https://isaac-sim.github.io/IsaacLab/main/source/overview/environments.html
        type: website
    - resource:
        title: DGX Spark Isaac Sim and Isaac Lab Playbook
        link: https://build.nvidia.com/spark/isaac/overview
        type: website

### FIXED, DO NOT MODIFY
# ================================================================================
weight: 1                       # _index.md always has weight of 1 to order correctly
layout: "learningpathall"       # All files under learning paths have this same wrapper
learning_path_main_page: "yes"  # This should be surfaced when looking for related content. Only set for _index.md of learning path content.
---