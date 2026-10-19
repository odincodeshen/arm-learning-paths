---
title: Stream a USB camera from the RDK X5 to a Raspberry Pi
description: Publish a USB camera stream with the TogetheROS hobot_usb_cam node on the RDK X5, receive it on a Raspberry Pi, and measure rate, CPU, and memory.
weight: 4

layout: "learningpathall"
---

## Overview

In path A, start the TogetheROS USB camera node on the RDK X5. On the Raspberry Pi, receive its image topic with a standard ROS 2 subscriber. Then check what the pipeline delivers to the Pi and what the camera node costs on the RDK X5.

![Path A: the RDK X5 forwards USB camera images to a standard ROS 2 subscriber on the Raspberry Pi.#center](images/path-a-camera.svg)

The results on this page were measured with this network:

| Device | Network |
| --- | --- |
| RDK X5 | Wired Gigabit Ethernet to the router (Wi-Fi disconnected during the test) |
| Raspberry Pi 5 | Wi-Fi, 2.4 GHz band, receive link rate about 96 Mbit/s |

The camera stream is MJPEG at 1920x1080 and 30 frames per second, with JPEG frames of about 190 to 230 KB. Frame size depends on the scene and the camera, and it has a large effect on network behavior.

The subscriber's QoS setting has a large effect on how many frames reach the Pi over this network. An optional section at the end of this page shows the measurements.

## Find the camera device and its modes

Connect the USB camera to a USB port on the RDK X5. On the **RDK X5**, list the video devices:

```bash
v4l2-ctl --list-devices
```

```output
Lenovo Performance Camera: Leno (usb-xhci-hcd.2.auto-1.3):
	/dev/video0
	/dev/video1
	/dev/video2
	/dev/video3
	/dev/media0
	/dev/media1
```

A single USB camera can create several device nodes. This camera has a color sensor, an infrared sensor, and a metadata node for each. Check the formats of the first node:

```bash
v4l2-ctl -d /dev/video0 --list-formats-ext | grep -E "\[|Size|Interval" | head -12
```

```output
	[0]: 'MJPG' (Motion-JPEG, compressed)
		Size: Discrete 640x480
			Interval: Discrete 0.033s (30.000 fps)
			Interval: Discrete 0.042s (24.000 fps)
		...
		Size: Discrete 1280x720
			Interval: Discrete 0.033s (30.000 fps)
		...
		Size: Discrete 1920x1080
			Interval: Discrete 0.033s (30.000 fps)
```

`/dev/video0` is the color camera. It supports Motion-JPEG (MJPEG) up to 1920x1080 at 30 frames per second. Uncompressed YUYV at 1920x1080 reaches only 5 frames per second, so use MJPEG. Your device numbers and modes can differ. Use the values your camera reports in the following steps.

## Start the camera node on the RDK X5

On the **RDK X5**, in a terminal where you sourced `/opt/tros/humble/setup.bash` and set `ROS_DOMAIN_ID=42`, start the camera at 1920x1080 and 30 frames per second:

```bash
ros2 launch hobot_usb_cam hobot_usb_cam.launch.py \
  usb_video_device:=/dev/video0 \
  usb_image_width:=1920 usb_image_height:=1080 \
  usb_framerate:=30 usb_pixel_format:=mjpeg
```

The launch prints the camera's supported formats and a few messages:

```output
Hobot shm pkg enables zero-copy with fastrtps profiles file: /opt/tros/humble/lib/hobot_shm/config/shm_fastdds.xml
Hobot shm pkg sets RMW_FASTRTPS_USE_QOS_FROM_XML: 1
[INFO] [hobot_usb_cam-1]: process started with pid [19494]
[hobot_usb_cam-1] [WARN] [...] [hobot_usb_cam]: framerate: 30
[hobot_usb_cam-1] [WARN] [...] [hobot_usb_cam]: pixel_format_name: mjpeg
[hobot_usb_cam-1] [WARN] [...] [hobot_usb_cam]: Camera calibration file: [...] does not exist!
```

The calibration warning is expected. The node still publishes images. The two `Hobot shm` lines show that the launch file also loads the TogetheROS shared-memory configuration for Fast DDS. You look at that configuration on the next page.

{{% notice Note %}}
The launch file defaults to `/dev/video8` and 960x480. Always pass `usb_video_device`, `usb_image_width`, and `usb_image_height` to match your camera.
{{% /notice %}}

## Inspect the topic on the RDK X5

Open a second RDK X5 terminal, source the environment, and list the topics with their types:

```bash
ros2 topic list -t
```

```output
/camera_info [sensor_msgs/msg/CameraInfo]
/image [sensor_msgs/msg/CompressedImage]
/parameter_events [rcl_interfaces/msg/ParameterEvent]
/rosout [rcl_interfaces/msg/Log]
```

The camera publishes `sensor_msgs/msg/CompressedImage`. Each message carries the JPEG bytes that the camera produced. The RDK X5 doesn't decode the image on this path, so no decode work runs on the RDK X5 CPU or on its hardware codec.

Check the publisher's QoS:

```bash
ros2 topic info -v /image | grep -E "Type|Node name|Reliability|Durability"
```

```output
Type: sensor_msgs/msg/CompressedImage
Node name: hobot_usb_cam
  Reliability: RELIABLE
  Durability: VOLATILE
```

Measure the rate and bandwidth as seen on the RDK X5:

```bash
ros2 topic hz /image
```

```output
average rate: 29.998
	min: 0.013s max: 0.068s std dev: 0.00299s window: 494
```

```bash
ros2 topic bw /image
```

```output
6.85 MB/s from 100 messages
	Message size mean: 0.23 MB min: 0.23 MB max: 0.23 MB
```

At 1920x1080, each JPEG frame is about 200 to 230 KB and the stream is about 7 MB/s (about 55 Mbit/s). JPEG size depends on what the camera sees, so your values will vary.

{{% notice Note %}}
`ros2 topic hz` reports the rate at which messages arrive at this subscriber. It isn't a direct measurement of the camera sensor's frame rate.
{{% /notice %}}

## Receive the images on the Raspberry Pi

On the Raspberry Pi, you use a small `rclpy` subscriber that counts frames and reports the rate and message size. On the **Pi** host, not in the container, save it into the shared folder you created on the previous page. Paste the whole block into the Pi terminal:

```bash
cat > ~/ros2/image_counter.py <<'EOF'
#!/usr/bin/env python3
"""Minimal standard ROS 2 image subscriber: counts frames and reports rate, size, and encoding."""
import argparse
import time

import rclpy
from rclpy.node import Node
from rclpy.qos import QoSProfile, ReliabilityPolicy, HistoryPolicy
from sensor_msgs.msg import CompressedImage, Image


class ImageCounter(Node):
    def __init__(self, topic, msg_type, reliable):
        super().__init__('image_counter')
        qos = QoSProfile(
            depth=5,
            history=HistoryPolicy.KEEP_LAST,
            reliability=ReliabilityPolicy.RELIABLE if reliable else ReliabilityPolicy.BEST_EFFORT,
        )
        cls = CompressedImage if msg_type == 'compressed' else Image
        self.create_subscription(cls, topic, self.on_msg, qos)
        self.count = 0
        self.bytes = 0
        self.first = None
        self.last_report = time.monotonic()
        self.window_count = 0

    def on_msg(self, msg):
        now = time.monotonic()
        if self.first is None:
            self.first = now
            if isinstance(msg, Image):
                desc = f'{msg.width}x{msg.height} encoding={msg.encoding} step={msg.step}'
            else:
                desc = f'format={msg.format}'
            self.get_logger().info(f'first frame: frame_id={msg.header.frame_id} {desc} bytes={len(msg.data)}')
        self.count += 1
        self.window_count += 1
        self.bytes += len(msg.data)
        if now - self.last_report >= 5.0:
            rate = self.window_count / (now - self.last_report)
            self.get_logger().info(f'frames={self.count} rate_5s={rate:.2f} Hz avg_bytes={self.bytes // self.count}')
            self.last_report = now
            self.window_count = 0


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument('--topic', default='/image')
    parser.add_argument('--type', choices=['raw', 'compressed'], default='raw')
    parser.add_argument('--seconds', type=float, default=60.0)
    parser.add_argument('--reliable', action='store_true')
    args = parser.parse_args()

    rclpy.init()
    node = ImageCounter(args.topic, args.type, args.reliable)
    end = time.monotonic() + args.seconds
    try:
        while rclpy.ok() and time.monotonic() < end:
            rclpy.spin_once(node, timeout_sec=0.5)
    finally:
        if node.first is not None and node.count > 1:
            elapsed = time.monotonic() - node.first
            node.get_logger().info(f'summary: frames={node.count} elapsed={elapsed:.1f}s mean_rate={node.count / elapsed:.2f} Hz')
        else:
            node.get_logger().info(f'summary: frames={node.count}')
        node.destroy_node()
        rclpy.shutdown()


if __name__ == '__main__':
    main()
EOF
```

The script uses only standard ROS 2 packages (`rclpy` and `sensor_msgs`). It doesn't use any TogetheROS component.

In the **Pi** container from the previous page, with ROS 2 sourced, run the subscriber for 60 seconds:

```bash
python3 /ros2/image_counter.py --topic /image --type compressed --seconds 60
```

```output
[INFO] [...] [image_counter]: first frame: frame_id=default_usb_cam format=jpeg bytes=195512
[INFO] [...] [image_counter]: frames=150 rate_5s=29.88 Hz avg_bytes=195606
[INFO] [...] [image_counter]: frames=301 rate_5s=29.94 Hz avg_bytes=195451
...
[INFO] [...] [image_counter]: summary: frames=1800 elapsed=60.0s mean_rate=30.01 Hz
```

The Pi receives 1,800 messages in 60 seconds, about 30 per second. The count shows the delivered rate. On its own, it doesn't prove that no frame was lost or delayed.

Without the `--reliable` option, `image_counter.py` subscribes with BEST_EFFORT reliability, VOLATILE durability, and KEEP_LAST history with a depth of 5. A BEST_EFFORT subscriber can receive from the RELIABLE publisher. This QoS suits live previews and processing that should use the newest frame and can tolerate dropped frames. If your application needs every frame, test reliable delivery, back pressure, and storage separately.

This choice matters on this network. With a RELIABLE subscriber and the default TogetheROS settings, the Pi received only about 1 frame per second of the same 1920x1080 stream. That result applies to this network, this frame size, and these software versions. It isn't a general measure of RELIABLE performance. For camera streams to another device, use a BEST_EFFORT subscriber. If you need RELIABLE delivery, test it with your image size and network first. The optional section at the end of this page shows the measurements.

## Measure CPU and memory on the RDK X5

While the subscriber runs, sample the camera process on the **RDK X5** once per second for 60 seconds:

```bash
pidstat -h -u -r -p $(pgrep -f "^/opt/tros/humble/lib/hobot_usb_cam/hobot_usb_cam") 1 60
```

Results from three 60-second runs at 1920x1080, each after a 20-second warm-up. CPU is per process, where 100% is one full core:

| Measurement | Run 1 | Run 2 | Run 3 |
| --- | --- | --- | --- |
| Messages received on the Pi in 60 s | 1,800 | 1,800 | 1,800 |
| RDK X5 `hobot_usb_cam` CPU | 19.0% | 18.5% | 17.9% |
| RDK X5 `hobot_usb_cam` RSS | 45 MB | 45 MB | 45 MB |

The RDK X5 camera node forwards the camera's JPEG bytes without decoding them, at about 7 MB/s. Where its CPU time goes inside the process wasn't profiled for this page.

## Troubleshooting

| Symptom | Check |
| --- | --- |
| No `/dev/video*` device for the camera | Run `lsusb` to confirm that the camera is detected, and `id` to confirm that your user is in the `video` group |
| Launch fails or the image size isn't what you set | The device, width, height, and format must match a mode listed by `v4l2-ctl --list-formats-ext` |
| `/image` doesn't appear on the RDK X5 | Check the launch output for errors, and check that `ROS_DOMAIN_ID` matches in both terminals |
| `/image` is listed on the Pi but nothing arrives | Run the subscriber with `--type compressed`, because the topic carries `CompressedImage`, not `Image`. A subscriber with the wrong type never matches the publisher. |
| Pi receives far fewer frames than the RDK X5 publishes | Run the subscriber with BEST_EFFORT, which is the default for `image_counter.py`. Check the IP reassembly counters described in the optional section below. If the RDK X5 uses Wi-Fi, try a wired connection. |

## Optional: why subscriber reliability matters

{{% notice Note %}}
This section is optional. It shows the measurements behind the BEST_EFFORT recommendation. You can skip it if you only need a working camera stream.
{{% /notice %}}

The first version of this test used a RELIABLE subscriber to match the publisher. At 1920x1080, the Pi then received only about 1 frame per second. To find out why, the test was repeated with two changes, each tested separately:

- **Subscriber reliability:** RELIABLE or BEST_EFFORT.
- **Fast DDS message size on the RDK X5:** the TogetheROS default, which sends UDP datagrams of up to 64 KB, or a profile that limits each datagram to 1,400 bytes. A 64 KB datagram is split into about 45 IP fragments on a standard 1,500-byte network. A 1,400-byte datagram fits in one packet without IP fragmentation. The 1,400-byte profile is a diagnostic comparison, not a fix. It splits each image into more RTPS fragments and adds packet-processing work.

The 1,400-byte results below are the author's supplementary measurements. The exact Fast DDS XML used for that arm isn't included in this Learning Path, so this arm isn't a reader exercise and its numerical results aren't independently reproducible from the steps here. The default-profile BEST_EFFORT stream above is the tested reader workflow.

You can watch IP fragmentation on the Pi with the kernel's reassembly counters. Read them before and after a run, and compare the differences:

```bash
awk '/^Ip:/{getline; print "ReasmReqds="$15, "ReasmOKs="$16, "ReasmFails="$17, "ReasmTimeout="$14}' /proc/net/snmp
```

`ReasmReqds` counts IP fragments that needed reassembly. `ReasmOKs` counts datagrams that were reassembled. `ReasmFails` counts failures that the reassembly code detected. The counters measure different things, so compare each counter between runs, not one counter against another.

Messages received on the Pi in three 60-second runs at 1920x1080, with the same network as above:

| RDK X5 Fast DDS message size | RELIABLE subscriber | BEST_EFFORT subscriber |
| --- | --- | --- |
| Default, up to 64 KB (IP fragmentation) | 61, 61, 84 (1.0 to 1.4 Hz) | 1,800 in each run (30.0 Hz) |
| Limited to 1,400 bytes (no IP fragmentation) | 215, 288, 232 (3.6 to 4.8 Hz) | 1,790, 1,800, 1,800 (29.9 to 30.0 Hz) |

The counter differences on the Pi for each 60-second run:

| Configuration | `ReasmReqds` | `ReasmOKs` | `ReasmFails` | `ReasmTimeout` |
| --- | --- | --- | --- | --- |
| Default size, RELIABLE | 262,000 to 302,000 | 320 to 399 | 244,000 to 283,000 | 110 to 361 |
| Default size, BEST_EFFORT | about 240,000 | 5,400 to 5,560 | 0 | 0 |
| 1,400 bytes, either reliability | 0 | 0 | 0 | 0 |

What these measurements show:

- The network can carry this 1080p30 stream. With a BEST_EFFORT subscriber, about 30 messages per second arrive, with or without IP fragmentation.
- IP fragmentation alone doesn't cause the loss. With BEST_EFFORT, about 240,000 fragments per run were reassembled with no failures.
- The drop happens only with a RELIABLE subscriber. With the default message size, the Pi recorded many IP reassembly failures. The 1,400-byte arm avoided IP reassembly, but the Pi still received fewer than five messages per second with a RELIABLE subscriber. In that arm, the Pi received about eight times more UDP datagrams per delivered message than with BEST_EFFORT. On the RDK X5, the camera process also used more CPU: about 25% with the default size and about 70% with 1,400-byte messages, compared with about 19% with the default size and BEST_EFFORT. With 1,400-byte messages and BEST_EFFORT, it used 34% to 42%, which shows the extra packet-processing cost of smaller messages.

What these measurements don't show is the exact mechanism inside Fast DDS reliable delivery that causes this behavior for large fragmented samples over this link. RTPS retransmissions weren't recorded. Results also depend on frame size: in a separate test with smaller JPEG frames over 5 GHz Wi-Fi, D-Robotics didn't see this drop with RELIABLE subscribers.

As a baseline, the test was repeated with both devices on wired Gigabit Ethernet and Wi-Fi disconnected on both. The camera view had changed, so JPEG frames were larger, about 310 to 320 KB. In three 60-second runs for each setting, both RELIABLE and BEST_EFFORT subscribers received 1,800 or 1,801 messages, about 30 per second. The Pi reassembled about 390,000 to 440,000 IP fragments per run with no `ReasmFails`. The drop with RELIABLE subscribers is therefore specific to the path that includes the Pi's 2.4 GHz Wi-Fi link.

In three earlier runs each on this network, RELIABLE subscribers received the full 30 frames per second at 1280x720 (about 80 KB per frame) and at 640x480 (about 30 KB per frame). In an earlier single run with the RDK X5 also on Wi-Fi, a BEST_EFFORT subscriber received about 5 frames per second at 1920x1080. That suggests the RDK X5's own Wi-Fi link was also a limit in that setup.

## What you have learned

You have:

- Started the TogetheROS `hobot_usb_cam` node with an explicit device, resolution, and format.
- Received its `CompressedImage` topic on a Raspberry Pi with a standard `rclpy` subscriber.
- Measured the delivered rate on the Raspberry Pi and the camera node's CPU and memory on the RDK X5, and saw how subscriber reliability affects delivery of large images over the network.

On the next page, you run the TogetheROS shared-memory example locally on the RDK X5.
