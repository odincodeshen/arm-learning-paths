---
title: Publish and verify a USB camera stream with TogetheROS
description: Configure TogetheROS `hobot_usb_cam` on the RDK X5, inspect the JPEG image topic, verify delivery to a ROS 2 receiver, and measure camera-node resource use.
weight: 4

layout: "learningpathall"
---

## Overview

With TogetheROS and ROS 2 discovery ready, start `hobot_usb_cam` on the RDK X5. First confirm the camera mode and inspect the image topic locally. Then use the Raspberry Pi as a receiver to check delivered frames and measure the camera node's CPU and memory use on the RDK X5.

![Path A: the RDK X5 forwards USB camera images to a standard ROS 2 subscriber on the Raspberry Pi.#center](images/path-a-camera.svg)

The results on this page were measured with this network:

| Device | Network |
| --- | --- |
| RDK X5 | Wired Gigabit Ethernet to the router (Wi-Fi disconnected during the test) |
| Raspberry Pi 5 | Wi-Fi, 2.4 GHz band, receive link rate about 96 Mbit/s |

The camera stream is MJPEG at 1920x1080 and 30 frames per second, with JPEG frames of about 190 to 230 KB. Frame size depends on the scene and the camera, and it has a large effect on network behavior.

The subscriber's QoS setting affects delivery over this network. The main steps use BEST_EFFORT to receive the live camera stream; an optional troubleshooting section at the end of the page shows the observed difference.

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

The calibration warning is expected. The node still publishes images. The two `Hobot shm` lines show that the launch file loads a TogetheROS Fast DDS profile. On the next page, you inspect that profile and verify data sharing in a local image-processing example. These log lines alone don't establish which transport this camera topic uses.

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

The receiver uses `rclpy` and `sensor_msgs`, so it can subscribe to the camera topic without installing TogetheROS on the Pi.

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

On the tested network, a RELIABLE subscriber received far fewer 1920x1080 frames than the BEST_EFFORT subscriber. This is an observation of the camera stream, receiver settings, and network used here. For a live stream that can tolerate dropped frames, start with BEST_EFFORT. If your application needs RELIABLE delivery, test it with your image size and network. The optional section below gives the measured counts.

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
| Receiver gets far fewer frames than the RDK X5 publishes | Run the subscriber with BEST_EFFORT, which is the default for `image_counter.py`. Check the network route described on the preparation page. If the RDK X5 uses Wi-Fi, try a wired connection. |

## Optional: troubleshoot a low receive rate

{{% notice Note %}}
This section shows the measurements behind the BEST_EFFORT setting used above. You can skip it when the camera stream is working at the rate your application needs.
{{% /notice %}}

With the default TogetheROS Fast DDS profile, the tested 1920x1080 camera stream gave these receiver counts over the RDK X5 Ethernet to Pi 2.4 GHz Wi-Fi link. Each value is a 60-second run:

| Subscriber reliability | Messages received in three runs |
| --- | --- |
| BEST_EFFORT | 1,800 / 1,800 / 1,800 (about 30 per second) |
| RELIABLE | 61 / 61 / 84 (about 1.0 to 1.4 per second) |

If your receiver sees a low rate, check its message type and QoS first, then check the network route described on the preparation page. The reason for the RELIABLE result on this link wasn't established. It doesn't predict RELIABLE delivery on other networks or with smaller images.

## What you have learned

You have:

- Started the TogetheROS `hobot_usb_cam` node with an explicit device, resolution, and format.
- Received its `CompressedImage` topic on a Raspberry Pi with a standard `rclpy` subscriber.
- Checked delivered image rate with a ROS 2 receiver and measured the camera node's CPU and memory on the RDK X5.

You have now captured and published live camera images with TogetheROS. Next, use a separate stored-image example to inspect local image sharing and object detection on the RDK X5.
