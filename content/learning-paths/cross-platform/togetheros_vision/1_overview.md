---
title: Understand the RDK X5 and TogetheROS for robotics vision
description: Learn what the D-Robotics RDK X5 provides for robotics vision, how TogetheROS.Bot extends ROS 2 on it, and which work runs on the Arm CPU and which runs on dedicated hardware.
weight: 2

layout: "learningpathall"
---

## What you will run

You will use the D-Robotics RDK X5 in two ROS 2 vision experiments. In path A, a USB camera feeds `hobot_usb_cam` from [TogetheROS.Bot](https://developer.d-robotics.cc/rdk_doc/en/Robot_development/tros/). The node publishes JPEG images over the network to a standard ROS 2 subscriber on a Raspberry Pi.

![Path A: a USB camera streams images from the RDK X5 to a standard ROS 2 subscriber on a Raspberry Pi.#center](images/path-a-camera.svg)

In path B, `hobot_image_pub` sends a stored image to `dnn_node_example` on the X5. You inspect their Fast DDS data-sharing path.

![Path B: a stored image passes between two nodes on the RDK X5 through Fast DDS data sharing, then reaches image processing and inference stages.#center](images/path-b-data-sharing.svg)

Fast DDS data sharing is verified between the local publisher and subscriber. The nodes still copy image data while preparing and processing it. Later, you run the same X5 nodes with a standard Image message for a local comparison, measure resource use, and profile where the inference node spends CPU time.

## About the RDK X5

The RDK X5 is part of the D-Robotics RDK (Robot Developer Kit) family. It's a developer board built on the D-Robotics Sunrise 5 chip, with an 8-core Arm Cortex-A55 processor and dedicated hardware for tasks such as neural network inference and image processing. Together with the TogetheROS.Bot robot software stack, the RDK boards are intended for building and evaluating robot prototypes.

The board features that this Learning Path uses:

| RDK X5 feature | How this Learning Path uses it |
| --- | --- |
| 8-core Arm Cortex-A55 CPU | Runs ROS 2, the camera node, Fast DDS, and the CPU parts of image processing and inference |
| Neural network accelerator | Runs the object detection model that serves as a downstream workload in the shared-memory example |
| Image processing hardware | Resizes images for the model, through `hobot_cv` |
| 4 USB 3.0 ports | Connect the USB camera |
| Gigabit Ethernet | Carries the camera stream to the Raspberry Pi |
| RDK OS 3.5 with TogetheROS.Bot 2.5.2 | Provides Ubuntu 22.04, ROS 2 Humble, and the TogetheROS packages |

The board has other interfaces that this Learning Path doesn't use, including two MIPI camera ports that also support stereo cameras, Wi-Fi, HDMI, CAN FD, and a 40-pin GPIO header. For the full hardware description, see the [RDK X5 hardware introduction](https://developer.d-robotics.cc/rdk_x_doc/en/Quick_start/hardware_introduction/rdk_x5) and the [RDK documentation](https://developer.d-robotics.cc/rdk_x_doc/en/).

## ROS 2 building blocks used here

If you have used ROS 2 before, the concepts are familiar:

- A **node** is a process that does one job, such as reading a camera.
- A **topic** is a named channel. Nodes publish messages to it and other nodes subscribe to it.
- A **message type** defines the data on a topic. In this Learning Path the camera publishes `sensor_msgs/msg/CompressedImage`, which carries the JPEG bytes produced by the camera.
- The **middleware (RMW)** moves messages between nodes. On both devices in this Learning Path it is eProsima Fast DDS (`rmw_fastrtps_cpp`).

If you need a refresher, see the [ROS 2 install guide](/install-guides/ros2/) and the [ROS 2 Humble documentation](https://docs.ros.org/en/humble/).

## What TogetheROS.Bot adds

TogetheROS.Bot (tros.b) is D-Robotics' robot software stack for its RDK boards. It is built on ROS 2 Humble and keeps the standard ROS 2 APIs, so standard tools such as `ros2 topic` and ordinary `rclpy` subscribers work with it. On top of ROS 2 it adds:

| Package | What it provides | Role in this Learning Path |
| --- | --- | --- |
| `hobot_usb_cam` | ROS 2 node for USB cameras | Publishes the USB camera stream |
| `hobot_shm` | Runtime environment and Fast DDS profile for shared-memory communication | Enables data sharing for the TogetheROS nodes |
| `hbm_img_msgs` | Image messages with a fixed maximum size | Carries images in the shared-memory example |
| `hobot_codec` | Image and video encoding and decoding | Encodes images for the web view in the shared-memory example. Not used on the camera path. |
| `hobot_cv` | Image processing, with platform acceleration paths. Whether hardware is used depends on the interface and the input. | Resizes images for the model in the shared-memory example |
| `dnn_node`, `dnn_node_example` | Model inference on the board's accelerator. Pre- and postprocessing can still use the CPU. | The subscriber in the shared-memory example |
| `hobot_image_publisher`, `websocket` | Publishing images from files, and browser visualization | The publisher in the shared-memory example |

## Know what runs where

The RDK X5 combines Arm CPU cores with dedicated hardware: a neural network accelerator, which D-Robotics calls the BPU, and media blocks for video codec and image processing, such as the VSE image scaling block that `hobot_cv` can use. This Learning Path focuses on the Arm CPU and the ROS 2 software. It uses the dedicated hardware only through the TogetheROS examples, and it doesn't measure the hardware's performance. Work done by those blocks is not Arm CPU work. When you read CPU numbers later in this Learning Path, keep three things apart:

- **Arm CPU work:** ROS 2 nodes, DDS serialization and transport, and any software image handling.
- **Accelerator work:** neural network inference in `dnn_node_example`.
- **Media hardware work:** hardware codec or image processing, when a pipeline uses it.

TogetheROS packages are built for D-Robotics RDK boards such as the RDK X5. Their API compatibility with ROS 2 does not mean that every package runs on every Arm system. In this Learning Path the Raspberry Pi runs only standard ROS 2 Humble packages.

## What the experiments measure

Path A measures delivery across devices, including the network. Path B examines communication between processes on the X5. The local comparison in path B uses the same image source, payload, and subscriber workload in both runs, but the message type and QoS differ. Read its CPU and delay results as observations of these two configurations, not as a transport-only speedup.

## Summary

- The RDK X5 is a developer board with an 8-core Arm Cortex-A55 processor.
- TogetheROS.Bot builds on ROS 2 Humble and adds RDK-specific packages, such as the USB camera node and shared-memory support.
- The first experiment streams a USB camera from the RDK X5 to a standard ROS 2 subscriber on a Raspberry Pi over the network.
- The second experiment runs on the RDK X5 only and shares images between processes through Fast DDS data sharing.
- CPU numbers in this Learning Path describe Arm CPU work only. Work on dedicated hardware isn't included.
