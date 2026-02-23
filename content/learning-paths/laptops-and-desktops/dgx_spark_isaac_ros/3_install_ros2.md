---
title: Install ROS 2 and connect it to Isaac Sim
weight: 4

### FIXED, DO NOT MODIFY
layout: learningpathall
---

## Install ROS 2 Jazzy

Set up the locale and add the ROS 2 apt repository:

```bash
sudo apt update && sudo apt install -y locales software-properties-common curl gnupg
sudo locale-gen en_US en_US.UTF-8
sudo update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8
export LANG=en_US.UTF-8

sudo add-apt-repository universe
sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key -o /usr/share/keyrings/ros-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu $(. /etc/os-release && echo $UBUNTU_CODENAME) main" | sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null
sudo apt update
```

Install the desktop version (includes RViz2) and teleop:

```bash
sudo apt install -y \
    ros-jazzy-desktop \
    ros-jazzy-teleop-twist-keyboard
```

Quick check:

```bash
source /opt/ros/jazzy/setup.bash
ros2 topic list
```

You should see `/parameter_events` and `/rosout`.

{{% notice Note %}}
Do **not** add `source /opt/ros/jazzy/setup.bash` to your `~/.bashrc`. It can interfere with Isaac Sim's internal ROS 2 libraries. Source it manually in each terminal where you need ROS 2 tools.
{{% /notice %}}

## Configure the ROS 2 Bridge

Isaac Sim 6.0 includes a built-in ROS 2 Bridge that handles communication with the ROS 2 ecosystem. You need to tell it which ROS distro and middleware to use, and point the library paths to the bridge's internal libraries:

```bash
echo 'export ROS_DISTRO=jazzy' >> ~/.bashrc
echo 'export RMW_IMPLEMENTATION=rmw_fastrtps_cpp' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH="${ISAACSIM_PATH}/exts/isaacsim.ros2.core/jazzy/lib:${LD_LIBRARY_PATH}"' >> ~/.bashrc
source ~/.bashrc
```

## Launch Isaac Sim with the bridge

Open a terminal and launch Isaac Sim (**don't** source ROS 2 `setup.bash` in this terminal):

```bash
${ISAACSIM_PATH}/isaac-sim.sh
```

In the startup log, look for this:

```output
Attempting to load system rclpy
Could not import system rclpy: No module named 'rclpy'
Attempting to load internal rclpy for ROS Distro: jazzy
rclpy loaded
```

That `Could not import system rclpy` is expected. Isaac Sim falls back to its internal `rclpy` and loads it successfully.

## Verify the connection

Open a **separate terminal**, source ROS 2, and check the topics:

```bash
source /opt/ros/jazzy/setup.bash
ros2 topic list
```

You should see at least `/parameter_events` and `/rosout`. Once you load a scene with a robot, sensor topics will appear. The bridge is working.
