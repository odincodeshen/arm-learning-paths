---
title: Prepare and verify the RDK X5 and Raspberry Pi environments
description: Record the RDK X5 software baseline, run ROS 2 Humble in a container on a Raspberry Pi, and verify ROS 2 discovery between the two devices.
weight: 3

layout: "learningpathall"
---

## Before you begin

You need two devices on the same network:

| Role | Device used in this Learning Path |
| --- | --- |
| Publisher | D-Robotics RDK X5 V1.0 (6.9 GiB memory visible to Linux), RDK OS 3.5.0 (Ubuntu 22.04.5), kernel 6.1.83 |
| Standard subscriber | Raspberry Pi 5 Model B, 8 GB, Raspberry Pi OS based on Debian 13 (trixie) |

Each command block in this Learning Path names the device it runs on. Open one terminal on each device.

## Record the RDK X5 baseline

Every later result depends on the software baseline, so record it first. On the **RDK X5**, run:

```bash
uname -m; uname -r
cat /etc/version
grep PRETTY_NAME /etc/os-release
lscpu | grep -E "Model name|^CPU\(s\)|CPU max MHz"
```

The output is similar to:

```output
aarch64
6.1.83
3.5.0
PRETTY_NAME="Ubuntu 22.04.5 LTS"
CPU(s):                               8
Model name:                           Cortex-A55
CPU max MHz:                          1500.0000
```

Check the installed TogetheROS.Bot release, the packages used in this Learning Path, and the ROS 2 middleware packages. Record the full Debian versions, because they identify the exact builds:

```bash
dpkg-query -W -f='${Package} ${Version}\n' tros-humble tros-humble-hobot-usb-cam \
  tros-humble-hobot-shm tros-humble-hbm-img-msgs tros-humble-hobot-image-publisher \
  tros-humble-dnn-node tros-humble-dnn-node-example tros-humble-hobot-cv tros-humble-hobot-codec \
  ros-humble-rmw-fastrtps-cpp ros-humble-fastrtps ros-humble-fastcdr
```

```output
ros-humble-fastcdr 1.0.24-2jammy.20250701.013706
ros-humble-fastrtps 2.6.10-1jammy.20250719.003736
ros-humble-rmw-fastrtps-cpp 6.2.8-1jammy.20250719.091658
tros-humble 2.5.2-jammy.20260313.064039
tros-humble-dnn-node 2.6.1-jammy.20260312.094112
tros-humble-dnn-node-example 2.6.1-jammy.20260312.112442
tros-humble-hbm-img-msgs 2.0.0-jammy.20241225.031409
tros-humble-hobot-codec 2.3.5-jammy.20260312.111803
tros-humble-hobot-cv 2.3.7-jammy.20260401.124358
tros-humble-hobot-image-publisher 2.2.1-jammy.20250425.114211
tros-humble-hobot-shm 0.0.1-jammy.20241225.030528
tros-humble-hobot-usb-cam 2.3.0-jammy.20250425.110259
```

The `tros-humble` 2.5.2 meta package is the TogetheROS.Bot release for RDK OS 3.5. Its individual packages have their own version numbers, so `dnn_node` 2.6.1 is part of the 2.5.2 release. The middleware is Fast DDS 2.6.10 with `rmw_fastrtps_cpp` 6.2.8.

{{% notice Note %}}
This Learning Path was tested with the TogetheROS.Bot 2.5.2 release for RDK OS 3.5. Don't run a general `apt upgrade` to "get the latest" before you follow it. Newer packages can change launch arguments and default behavior.
{{% /notice %}}

## Source the TogetheROS environment on the RDK X5

The default login shell on the RDK X5 doesn't load ROS 2. In every new RDK X5 terminal, run:

```bash
source /opt/tros/humble/setup.bash
export ROS_DOMAIN_ID=42
```

The TogetheROS setup script also loads ROS 2 Humble from `/opt/ros/humble`. With no `RMW_IMPLEMENTATION` set, ROS 2 uses Fast DDS:

```bash
ros2 doctor --report | grep "middleware name"
```

```output
middleware name    : rmw_fastrtps_cpp
```

`ROS_DOMAIN_ID` isolates your nodes from other ROS 2 systems on the same network. The default is 0, which other ROS 2 devices on a shared network are likely to use as well. This Learning Path uses 42 only because it's a non-default value. You can choose any value from 0 to 101, but it must be the same on both devices.

## Prepare ROS 2 on the Raspberry Pi

ROS 2 Humble packages are built for Ubuntu 22.04, while the Raspberry Pi OS image used here is based on Debian 13. Instead of building ROS 2 from source, run the official `ros:humble-ros-base` container on the Pi. Install Docker first if needed; see the [Docker install guide](/install-guides/docker/).

On the **Raspberry Pi**, pull the image:

```bash
docker pull ros:humble-ros-base
```

Record the image digest with your results. The tag can change over time; the digest identifies the image you pulled:

```bash
docker image inspect ros:humble-ros-base --format '{{json .RepoDigests}}'
```

Create a working folder that you share with the container. Create it as your own user before you start the container. If the folder doesn't exist, Docker creates it as `root`, and you can't save files in it later:

```bash
mkdir -p ~/ros2
```

Start a shell in the container:

```bash
docker run -it --rm --net=host --ipc=host -e ROS_DOMAIN_ID=42 -v ~/ros2:/ros2 ros:humble-ros-base
```

Inside the container, source ROS 2:

```bash
source /opt/ros/humble/setup.bash
```

The `--net=host` option is required. It puts the container on the Pi's own network interfaces so that DDS discovery multicast reaches the RDK X5. The `-v ~/ros2:/ros2` option shares a folder for the subscriber script you use on the next page.

## Verify discovery between the devices

Before you send any images, confirm that the two devices can find each other.

On the **RDK X5**, publish a small test message twice per second:

```bash
ros2 topic pub -r 2 /lp1_ping std_msgs/msg/String "{data: hello-from-x5}"
```

In the **Raspberry Pi** container, list the topics and receive one message:

```bash
ros2 topic list
ros2 topic echo --once /lp1_ping std_msgs/msg/String
```

```output
/lp1_ping
/parameter_events
/rosout
data: hello-from-x5
---
```

Stop the publisher with Ctrl+C. Then repeat the test in the other direction: publish from the Pi container and run `ros2 topic echo --once` on the RDK X5. Discovery must work in both directions before you continue.

## Troubleshooting discovery

If the topic doesn't appear, check these items in order:

| Symptom | Check |
| --- | --- |
| No topics from the other device | Both devices use the same `ROS_DOMAIN_ID` |
| Topic visible on RDK X5 only | The Pi container was started with `--net=host` |
| Nothing works across devices | Both devices are on the same subnet. Some guest Wi-Fi networks isolate clients or block multicast. |
| `ros2: command not found` | You sourced `/opt/tros/humble/setup.bash` (RDK X5) or `/opt/ros/humble/setup.bash` (Pi) in this shell |
| Firewall enabled | UDP ports 7400 and above are allowed between the devices |

## Plan the network

Discovery works over Wi-Fi, but a 1080p camera stream is about 55 Mbit/s, so the network matters for the next page. This Learning Path was measured with the following network:

| Device | Connection |
| --- | --- |
| RDK X5 | Gigabit Ethernet to the router |
| Raspberry Pi 5 | Wi-Fi, 2.4 GHz band |

If the RDK X5 has both Ethernet and Wi-Fi connected to the same network, check which interface it uses to reach the Pi. Replace `<pi-address>` with the IP address of your Pi:

```bash
ip route get <pi-address>
```

```output
192.168.1.20 dev wlan0 src 192.168.1.10 uid 1000
```

Your addresses will differ. On the RDK X5 used for this Learning Path, NetworkManager gives the Wi-Fi route a lower metric (600) than the Ethernet route (700). Traffic to the Pi therefore leaves through Wi-Fi even when the cable is connected. In a 15-second check in this state, the Pi received about 16 frames per second of the 1080p stream instead of 30. To make sure image data uses the wired link, disconnect Wi-Fi on the RDK X5 while you measure.

If you work on the RDK X5 over SSH, first make sure that your session uses the wired address. Otherwise the next command disconnects you. Print the address of your current session and compare it with the `eth0` address:

```bash
echo $SSH_CONNECTION | cut -d' ' -f3
ip -4 -o addr show eth0
```

The first command can print an IPv6 address. An address that ends in `%wlan0`, or that matches the `wlan0` address, means that your session uses Wi-Fi. In that case, reconnect over SSH to the IPv4 address that `eth0` shows, for example `ssh sunrise@192.168.1.11`. Then disconnect Wi-Fi:

```bash
sudo nmcli dev disconnect wlan0
ip route get <pi-address>
```

The second command now shows `dev eth0`.

Reconnect it when you finish:

```bash
sudo nmcli dev connect wlan0
```

The next page shows how the network and the subscriber's QoS setting affect how many frames reach the Pi.
