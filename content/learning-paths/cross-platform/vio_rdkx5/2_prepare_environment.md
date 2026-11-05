---
title: Install VIMS on a separate RDK OS image
description: Flash RDK OS 3.5.0 to a separate microSD card for the RDK X5, install the D-Robotics VIMS 0.0.6 solution with its documented steps, and record which packages, kernel files, and settings the installation changes.
weight: 3

layout: "learningpathall"
---

<!--
DRAFT (2026-10-30). Not reviewed. 2026-10-31: the author ran this page as written on a freshly flashed card (user report);
Claude has not compared those outputs with the sample outputs on this page. Replace the sample outputs and the timing
from that run if they differ.
Basis: evidence/run-20261023-01 (image variant), run-20261024-01 (L1-1 to L1-10: the installation done step by step,
with an interrupted apt upgrade that had to be repaired), VIMS manual section 4 (snapshot in
evidence/run-20261016-01/web/vims-index.txt), D-ROBOTICS_LP2_REPLY_v1.md section 8 and 9.
Open items before publishing:
- Done by the author 2026-10-31 (experiment B): compare that run with the sample outputs and record the time.
- D-Robotics reply v2 section 4 (2026-11-04): stay on RDK OS 3.5.0, TROS 2.5.2, VIMS 0.0.6 and record the build
  numbers. With use_local_imu false, the dataset line doesn't need the camera driver update, kernel headers, or a
  whole-system apt upgrade. A minimal install without those steps has NOT been tested; the author decided on
  2026-11-04 to keep the tested path, and the page says so. Kernel commit order checked with the GitHub compare API (6e685bd ahead of c642eab by 4).
  Package versions from apt depend on the date of the run.
- D-Robotics asks not to describe the failed services as normal (reply section 8); cause still unknown (reply v2).
- The commands use sudo from the sunrise user; the VIMS manual says to log in as root. Equivalent for installation;
  confirm the wording with D-Robotics.
-->

## Overview

On this page, you prepare an RDK X5 for VIMS:

1. Flash RDK OS 3.5.0 to a separate microSD card and record the starting state.
2. Install VIMS 0.0.6 with the steps from the [VIMS user manual](https://d-robotics.github.io/tros_vims_doc/).
3. Record the state after the installation and check which packages VIMS provides.

The installation changes more than a typical ROS 2 package installation. It upgrades the system packages, replaces the kernel and device tree packages, and adds settings to the shell of the `root` user. Use a separate microSD card for it, so another project on the same RDK X5 keeps its image.

The steps follow the VIMS 0.0.6 manual, which D-Robotics describes as a validation release. Package versions that `apt` installs depend on the day you run it, so record the versions you get.

{{% notice Note %}}
This Learning Path was tested with all the steps on this page. D-Robotics states that a dataset replay, which doesn't use the camera or the IMU of the board, doesn't need the camera driver update, the kernel headers, or a system upgrade. A smaller installation without those steps wasn't tested here, and the VIMS manual includes them. A new image still needs the ROS 2, OpenCV, and rosbag2 user-space dependencies.
{{% /notice %}}

## Flash RDK OS 3.5.0 to a separate card

On your **workstation**, download the RDK OS 3.5.0 desktop image and its checksum:

```bash
curl -fLO https://archive.d-robotics.cc/downloads/os_images/rdk_x5/rdk_os_3.5.0-2026-4-9/rdk-x5-ubuntu22-preinstalled-desktop-3.5.0-arm64.img.xz
curl -fLO https://archive.d-robotics.cc/downloads/os_images/rdk_x5/rdk_os_3.5.0-2026-4-9/rdk-x5-ubuntu22-preinstalled-desktop-3.5.0-arm64.img.xz.md5sum
```

Compare the MD5 checksum of the image with the value in the `.md5sum` file. Then write the image to the new microSD card with a tool such as [balenaEtcher](https://etcher.balena.io/). Check twice that the target is the new card.

Insert the card into the RDK X5, connect Ethernet, and power it on. Log in as the `sunrise` user over SSH, and change the default password with `passwd`. For the first-boot steps, see the [RDK X5 quick start](https://developer.d-robotics.cc/rdk_x_doc/en/Quick_start/).

{{% notice Note %}}
The image has no mDNS service, so `<hostname>.local` names don't resolve. Use the IP address of the RDK X5, for example from your router, or from `hostname -I` on the board.
{{% /notice %}}

## Record the starting state

On the **RDK X5**, record the system, kernel, and package versions before you change anything:

```bash
mkdir -p ~/vims-install
{ cat /etc/version; uname -a; df -h /; cat /sys/devices/system/cpu/cpufreq/policy0/scaling_governor
  dpkg-query -W tros-humble hobot-boot hobot-dtb hobot-kernel-headers; } | tee ~/vims-install/before.txt
dpkg -l > ~/vims-install/dpkg-before.txt
```

```output
3.5.0
Linux ... 6.1.83 #1 SMP PREEMPT Wed Apr  8 21:09:59 CST 2026 aarch64 aarch64 aarch64 GNU/Linux
Filesystem      Size  Used Avail Use% Mounted on
/dev/root        29G  8.4G   20G  31% /
schedutil
tros-humble	2.5.2-jammy.20260313.064039
hobot-boot	3.1.0-20260409105641
hobot-dtb	3.0.8-20260409105754
hobot-kernel-headers	3.0.4-20260409105759
```

The output shows a 32 GB card. The CPU governor, the Linux policy that sets the CPU frequency, is `schedutil`. You need this value to restore it later.

## Upgrade the system and install the dependencies

The VIMS manual starts with a full system upgrade, followed by the ROS 2 packages that VIMS needs, such as RTAB-Map, Cyclone DDS, Nav2, RViz, and the Foxglove bridge.

On the **RDK X5**, upgrade the system:

```bash
sudo apt update
sudo apt upgrade -y
```

{{% notice Note %}}
During the upgrade, the `openssh-server` package asks whether to keep the modified `/etc/ssh/sshd_config` file. Select **keep the local version currently installed**. Run this step in a terminal that stays open, and don't pipe `apt` into `tee`: the question then can't be answered, and the upgrade stops. If the terminal closes during the upgrade, reconnect and run `sudo dpkg --configure -a`.
{{% /notice %}}

In the test for this Learning Path, the upgrade kept TogetheROS.Bot at 2.5.2 and didn't change the `hobot-boot`, `hobot-dtb`, or `hobot-kernel-headers` packages. It upgraded 188 ROS 2 Humble packages, for example `rclcpp` from 16.0.14 to 16.0.21, together with Ubuntu system packages. The ROS 2 base on this card therefore differs from a card that you didn't upgrade.

Install the packages that the VIMS manual lists:

```bash
sudo apt install ros-dev-tools 'ros-humble-cyclonedds*' 'ros-humble-rmw-cyclonedds*' 'ros-humble-rviz*' \
  'ros-humble-rtabmap*' ros-humble-octomap-rviz-plugins ros-humble-teleop-twist-keyboard 'ros-humble-nav*' \
  ros-humble-robot-localization ros-humble-foxglove-bridge -y
```

```output
0 upgraded, 319 newly installed, 0 to remove and 8 not upgraded.
```

The patterns also install debug symbol packages for many of these components. The 8 packages that aren't upgraded are GCC 11 packages that the RDK OS image holds at their version.

Install `yq`, a YAML command-line tool that the VIMS launch script uses:

```bash
sudo wget -q https://github.com/mikefarah/yq/releases/latest/download/yq_linux_arm64 -O /usr/local/bin/yq
sudo chmod +x /usr/local/bin/yq
yq --version
```

```output
yq (https://github.com/mikefarah/yq/) version v4.54.1
```

## Install VIMS

Download the VIMS 0.0.6 package and its checksum, check it, and extract it to `/userdata/vims`. The package is about 190 MB:

```bash
sudo mkdir -p /userdata/vims /userdata/rtabmap
cd /userdata/vims
sudo wget -q https://archive.d-robotics.cc/TogetheROS/files/vision_mobile_solution/install-vims-v0.0.6.tar.gz.sha256
sudo wget -q https://archive.d-robotics.cc/TogetheROS/files/vision_mobile_solution/install-vims-v0.0.6.tar.gz
sha256sum -c install-vims-v0.0.6.tar.gz.sha256
```

```output
install-vims-v0.0.6.tar.gz: OK
```

Extract the package only if the check passed:

```bash
sudo tar -zxf install-vims-v0.0.6.tar.gz -C /userdata/vims
ls /userdata/vims/install
```

The `install` folder is a ROS 2 workspace with about 50 packages, including the VIO node `drobotics_vio`, the launch package `tros_vision_nav`, and VIMS builds of `mipi_cam`, `hobot_stereonet`, and RTAB-Map.

## Update the kernel and device tree packages

The VIMS manual updates three board support packages that are built for RDK OS 3.5.0. The version numbers are the same as in the image, but the packages are different builds with a later build date. The date doesn't tell you which kernel code is newer: in the [public kernel repository](https://github.com/D-Robotics/x5-kernel/compare/c642eab...6e685bd), commit `6e685bd` of the image comes after commit `c642eab` of these packages.

```bash
mkdir -p ~/vims-install/deb && cd ~/vims-install/deb
wget -q https://archive.d-robotics.cc/TogetheROS/files/vision_mobile_solution/deb/hobot-kernel-headers_3.0.4-20260514041655_arm64.deb
wget -q https://archive.d-robotics.cc/TogetheROS/files/vision_mobile_solution/deb/hobot-boot_3.1.0-20260514041708_arm64.deb
wget -q https://archive.d-robotics.cc/TogetheROS/files/vision_mobile_solution/deb/hobot-dtb_3.0.8-20260514041655_arm64.deb
sudo dpkg -i hobot-kernel-headers_3.0.4-20260514041655_arm64.deb hobot-boot_3.1.0-20260514041708_arm64.deb hobot-dtb_3.0.8-20260514041655_arm64.deb
```

```output
Unpacking hobot-kernel-headers (3.0.4-20260514041655) over (3.0.4-20260409105759) ...
...
Unpacking hobot-boot (3.1.0-20260514041708) over (3.1.0-20260409105641) ...
...
Unpacking hobot-dtb (3.0.8-20260514041655) over (3.0.8-20260409105754) ...
Setting up hobot-dtb (3.0.8-20260514041655) ...
Setting up hobot-boot (3.1.0-20260514041708) ...
...
Setting up hobot-kernel-headers (3.0.4-20260514041655) ...
```

The new `hobot-boot` package contains only the standard 6.1.83 kernel and its modules. It replaces `/boot/Image` and removes the real-time kernel `/boot/Image-rt` and its modules, so don't use it if you need the real-time kernel. The boot script loads `/boot/Image`, so the board still starts with the standard kernel.


## Add the VIMS shell settings

The VIMS manual adds three settings to the `.bashrc` file of the `root` user: the `performance` CPU governor, ROS domain 42, and Cyclone DDS as the ROS 2 middleware. Back up the file, and then add the settings:

```bash
sudo cp -p /root/.bashrc /root/.bashrc.pre-vims
sudo bash -c 'cat >> /root/.bashrc' <<'EOF'
echo performance > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
echo performance > /sys/devices/system/cpu/cpufreq/policy0/scaling_governor
export ROS_DOMAIN_ID=42
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
EOF
```

These settings apply only to interactive shells of the `root` user. On the next pages you run VIMS as the `sunrise` user, and you set the domain and middleware in each terminal yourself. You also set the CPU governor explicitly when you measure, and restore it afterward.

The VIMS manual also shows how to turn off Wi-Fi power saving. With a wired connection, you don't need it.

Restart the RDK X5 to boot the new kernel:

```bash
sudo reboot
```

## Check the installation

After the restart, connect again as `sunrise`, and record the state after the installation:

```bash
{ cat /etc/version; uname -a; df -h /; cat /sys/devices/system/cpu/cpufreq/policy0/scaling_governor
  dpkg-query -W tros-humble hobot-boot hobot-dtb hobot-kernel-headers; } | tee ~/vims-install/after.txt
dpkg -l > ~/vims-install/dpkg-after.txt
```

```output
3.5.0
Linux ... 6.1.83 #2 SMP PREEMPT Tue May 12 11:39:42 CST 2026 aarch64 aarch64 aarch64 GNU/Linux
Filesystem      Size  Used Avail Use% Mounted on
/dev/root        29G   12G   16G  43% /
schedutil
tros-humble	2.5.2-jammy.20260313.064039
hobot-boot	3.1.0-20260514041708
hobot-dtb	3.0.8-20260514041655
hobot-kernel-headers	3.0.4-20260514041655
```

The kernel build date changed, TogetheROS.Bot is still 2.5.2, and the three board support packages have the new builds. The governor is still `schedutil`, because the `root` shell settings haven't run.

Check the VIMS version and which packages come from VIMS:

```bash
source /opt/tros/humble/local_setup.bash
source /userdata/vims/install/local_setup.bash
grep "<version>" $(ros2 pkg prefix tros_vision_nav --share)/package.xml
for p in drobotics_vio mipi_cam hobot_stereonet rtabmap_slam rmw_cyclonedds_cpp; do echo "$p: $(ros2 pkg prefix $p)"; done
```

```output
  <version>0.0.6</version>
drobotics_vio: /userdata/vims/install
mipi_cam: /userdata/vims/install
hobot_stereonet: /userdata/vims/install
rtabmap_slam: /userdata/vims/install
rmw_cyclonedds_cpp: /opt/ros/humble
```

When you source the VIMS workspace after TogetheROS.Bot, ROS 2 uses the VIMS builds of `mipi_cam`, `hobot_stereonet`, and RTAB-Map instead of the TogetheROS and Ubuntu packages of the same name. The installed files of those packages don't change. Only the selection does, and only in shells that source the VIMS workspace.

## Review what changed

| Area | Change |
| --- | --- |
| System packages | Ubuntu and ROS 2 Humble packages upgraded; about 320 packages added, none removed |
| TogetheROS.Bot | Still 2.5.2 |
| Kernel and device tree | `hobot-boot`, `hobot-dtb`, `hobot-kernel-headers` replaced with the 20260514 builds; the real-time kernel removed |
| ROS 2 packages in use | VIMS builds of `mipi_cam`, `hobot_stereonet`, `hobot_cv`, and RTAB-Map when you source `/userdata/vims/install` |
| Files | `/usr/local/bin/yq`, `/userdata/vims`, `/userdata/rtabmap` |
| Shell settings | Governor, ROS domain, and middleware settings in `/root/.bashrc` |

In the test for this Learning Path, `systemctl --failed` also listed `auto_update_miniboot.service` and `dnsmasq.service` after the installation. These are system services, and according to D-Robotics they aren't dependencies of the VIO algorithm in offline use. Their cause hasn't been confirmed, so don't treat them as normal, and record them separately from your VIO results. They didn't affect the steps in this Learning Path.

## Troubleshooting

| Symptom | Check |
| --- | --- |
| The upgrade stops at a question about `sshd_config`, or the screen shows codes when you press the arrow keys | Run `apt` directly in the terminal, not through a pipe. Select **keep the local version currently installed**. |
| The terminal closed during the upgrade | Reconnect and run `sudo dpkg --configure -a`, then run the upgrade again. |
| `sha256sum -c` reports `FAILED` | Delete the package and download it again. |
| `uname -a` still shows the April 8 kernel after the restart | Check the `dpkg -i` output for errors, and that you restarted the board. |
| `ros2 pkg prefix tros_vision_nav` reports that the package isn't found | Source `/userdata/vims/install/local_setup.bash` after `/opt/tros/humble/local_setup.bash`. |

## What you have learned

You have:

- Prepared a separate RDK OS 3.5.0 image for VIMS, so another project keeps its own image.
- Installed VIMS 0.0.6 with the documented steps.
- Recorded the system before and after the installation, and identified the changed packages, kernel files, and settings.
- Checked which ROS 2 packages come from the VIMS workspace.

On the next page, you prepare a public dataset that the VIMS VIO node can use instead of a camera.
