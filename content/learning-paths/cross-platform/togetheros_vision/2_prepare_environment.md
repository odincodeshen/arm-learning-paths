---
title: Prepare the RDK X5 and the Raspberry Pi
description: Check the RDK X5 software versions, prepare a Raspberry Pi to receive the camera topic, and verify ROS 2 discovery before streaming images.
weight: 3

layout: "learningpathall"
---

## Before you begin

Ensure that both devices are on the same network. This Learning Path uses the following hardware:

- D-Robotics RDK X5
- Raspberry Pi 5

TogetheROS.Bot comes preinstalled in the official RDK OS 3.5 images for the RDK X5. This Learning Path was tested with RDK OS 3.5.0 and TogetheROS.Bot 2.5.2.

On the **RDK X5**, check your installed versions:

```bash
cat /etc/version
dpkg-query -W -f='${Package} ${Version}\n' tros-humble
```

The output on the tested setup was:

```output
3.5.0
tros-humble 2.5.2-jammy.20260313.064039
```

Different software versions may change launch arguments or default behavior.

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

The Pi needs ROS 2 only to receive the camera topic. ROS 2 Humble packages are built for Ubuntu 22.04, while the Raspberry Pi OS image used here is based on Debian 13. In the following steps, you will pull and run the official `ros:humble-ros-base` container on the Pi. Install Docker first if needed; see the [Docker install guide](/install-guides/docker/).

On the **Raspberry Pi**, pull the image:

```bash
docker pull ros:humble-ros-base
```

{{% notice Note %}}
This Learning Path was tested with the arm64 image pulled on October 15, 2026, with digest `sha256:860e53793ccf575e5369b5e9a19939de2bfcfb46a299954d06f54f2af54fed19`. If your results differ and you want to use the same image, replace `ros:humble-ros-base` with `ros:humble-ros-base@sha256:860e53793ccf575e5369b5e9a19939de2bfcfb46a299954d06f54f2af54fed19` in the `docker pull` and `docker run` commands.
{{% /notice %}}

```bash
docker image inspect ros:humble-ros-base --format '{{json .RepoDigests}}'
```

Start a shell in the container:

```bash
docker run -it --rm --net=host --ipc=host -e ROS_DOMAIN_ID=42 ros:humble-ros-base
```
The `--net=host` option gives ROS 2 access to the Pi's network interfaces. It also lets the Pi host reach the browser viewer that you start inside the container on the next page.

### Install the live-view packages

Inside the **Pi container**, install the viewer and its compressed-image transport plugin for the camera example:

```bash
apt-get update
apt-get install -y --no-install-recommends ros-humble-web-video-server ros-humble-compressed-image-transport
```

`apt-get update` refreshes the package list. The install command downloads the viewer and its required dependencies. Run this installation once per container, then keep the container open for the next page. With `--rm`, exiting removes the container and its installed packages; a new container needs this installation again.

Source ROS 2:

```bash
source /opt/ros/humble/setup.bash
```

## Test multicast between the devices

Fast DDS uses multicast announcements for discovery by default. However, depending on your network configuration, this may be blocked.

In the **Raspberry Pi container**, start the receiver and leave it waiting:

```bash
ros2 multicast receive
```

While the receiver is waiting, run on the **RDK X5**:

```bash
ros2 multicast send
```

The Pi should print a message similar to:

```output
Received from <x5-address>:<port>: 'Hello World!'
```

If this works, skip the fallback section below and continue to [Verify discovery between the devices](#verify-discovery-between-the-devices). If the Pi receives nothing, stop the receiver with Ctrl+C and use the unicast fallback.

### Fallback: Use unicast discovery when multicast is unavailable

Configure [Fast DDS unicast discovery](https://fast-dds.docs.eprosima.com/en/v2.6.10/fastdds/transport/disabling_multicast.html) with explicit peer addresses. This can work across routed subnets, provided the network allows UDP traffic between the devices.

On **each device's host**, find its IPv4 address. For the Pi, open another SSH session to run this command and leave the container running:

```bash
ip -br -4 addr
```

In the block below, replace `RDK_X5_IP` and `PI_IP` with those addresses. Use the connected Ethernet or Wi-Fi address, not `lo` or `docker0`. Run the same block **natively on the X5 and inside the Pi container**, after sourcing the respective ROS environment:

```bash
export FASTRTPS_DEFAULT_PROFILES_FILE="$(mktemp /tmp/lp1-fastdds-XXXXXX.xml)"
export FASTDDS_DEFAULT_PROFILES_FILE="$FASTRTPS_DEFAULT_PROFILES_FILE"

cat > "$FASTRTPS_DEFAULT_PROFILES_FILE" <<'XML'
<?xml version="1.0" encoding="UTF-8"?>
<profiles xmlns="http://www.eprosima.com/XMLSchemas/fastRTPS_Profiles">
  <transport_descriptors>
    <transport_descriptor>
      <transport_id>lp1_udp</transport_id>
      <type>UDPv4</type>
      <maxInitialPeersRange>32</maxInitialPeersRange>
    </transport_descriptor>
  </transport_descriptors>
  <participant profile_name="lp1_unicast" is_default_profile="true">
    <rtps>
      <builtin>
        <metatrafficUnicastLocatorList>
          <locator/>
        </metatrafficUnicastLocatorList>
        <initialPeersList>
          <locator><udpv4><address>RDK_X5_IP</address></udpv4></locator>
          <locator><udpv4><address>PI_IP</address></udpv4></locator>
        </initialPeersList>
      </builtin>
      <userTransports>
        <transport_id>lp1_udp</transport_id>
      </userTransports>
      <useBuiltinTransports>false</useBuiltinTransports>
    </rtps>
  </participant>
</profiles>
XML

export ROS_DOMAIN_ID=42
export ROS_LOCALHOST_ONLY=0
export RMW_IMPLEMENTATION=rmw_fastrtps_cpp
export RMW_FASTRTPS_USE_QOS_FROM_XML=0
unset ROS_DISCOVERY_SERVER SKIP_DEFAULT_XML
ros2 daemon stop
```

Setting `RMW_FASTRTPS_USE_QOS_FROM_XML=0` keeps ROS 2's history memory policy when the camera launch runs; the XML still configures discovery and transport.

Continue with the text-message test below in these configured shells. For any new ROS terminal or Pi container used for the camera example, source ROS and repeat this fallback setup. Validate camera delivery separately on the next page.

## Verify discovery between the devices

Before you send images from `hobot_usb_cam`, confirm that its future receiver can discover ROS 2 topics from the RDK X5. Use a small text message for this check.

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

Stop the publisher with Ctrl+C. Then repeat the test in the other direction: publish from the Pi container and run `ros2 topic echo --once` on the RDK X5. Once discovery works in both directions, you're ready to test the camera stream on the next page.

## Troubleshooting discovery

If the topic doesn't appear, check these items in order:

| Symptom | Check |
| --- | --- |
| No topics from the other device | Both devices use the same `ROS_DOMAIN_ID` |
| Topic visible on RDK X5 only | The Pi container was started with `--net=host` |
| Nothing works across devices | Both devices are on the same subnet. Some guest Wi-Fi networks isolate clients or block multicast. |
| `ros2: command not found` | You sourced `/opt/tros/humble/setup.bash` (RDK X5) or `/opt/ros/humble/setup.bash` (Pi) in this shell |
| Firewall enabled | UDP ports 7400 and above are allowed between the devices |

The environment is ready. On the next page, publish the camera stream from the RDK X5, receive it on the Pi, and view it in your computer's browser.
