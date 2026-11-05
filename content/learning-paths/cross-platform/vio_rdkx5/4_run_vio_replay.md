---
title: Run visual-inertial odometry from a replayed dataset
description: Run the VIMS visual-inertial odometry node on the RDK X5 with a replayed TUM-VI sequence, record its trajectory, and compare the trajectory with the dataset ground truth.
weight: 5

layout: "learningpathall"
---

<!--
DRAFT (2026-10-30). Not reviewed. Commands and outputs come from evidence/run-20261024-03 (real-time replay),
evidence/run-20261025-01 (offline_bag_path E-offline-full1/2; real-time A-rep*; Foxglove D-foxglove) and
evidence/run-20261030-01 (Foxglove section, real-time replay, 2026-10-31),
evidence/run-20261026-01 (every command on this page run as written; <x5-address> = X5 IP; ~/tumvi/venv and
~/tumvi/dataset-room1_512_16 were symlinks standing in for page 3; Foxglove GUI step not run).
D-Robotics reply (D-ROBOTICS_LP2_REPLY_v1.md, 2026-10-30) settles: dataset replay main line, left-on-top layout,
mono8 for VIO, offline_bag_path, publishing commands/config/output with source, version and conditions.
Open items before publishing:
- D-Robotics reply v2 (2026-11-04): pin VIMS 0.0.6, no release date, keep the 10800 s note, no promise to remove
  it; drobotics_vio source isn't public, don't label the docs repo as source.
- Offline mode publishes no /clock (0 messages in E-offline-full1). use_sim_time:=False tested 2026-10-31
  (run-20261030-01 F-offline-simfalse): no poses at all. Cause from D-Robotics reply v2 section 2 (IMU age filter),
  not checked by us; the page states it as a limit of the current implementation.
- Page 3 must create ~/tumvi/ (workstation) and ~/lp2-tumvi/room1_512_vertical, ~/lp2-tumvi/config/tumvi_512/ (X5).
- Foxglove screenshot taken 2026-10-31 (run-20261030-01 G-foxglove) in real-time replay. In offline mode, pathimu
  carries stamp 0 (header and every pose); Foxglove showing it in offline mode is not verified, so the page uses
  real-time replay. images/foxglove-trackhist.png (offline mode, trackhist only) is kept for page 1 or 5.
- images/vio-vs-ground-truth.png: scripts/plot_vio_vs_mocap.py on evidence/run-20261030-01/F-offline-full3/vio_out
  (same pairing and alignment as the page script; 2742 matched, RMSE 0.274 m, max 0.586 m). Rendered from the SVG
  with headless Chrome. The "drift" sentence is a general VIO statement, not measured here.
-->

## Overview

On the previous page, you converted the TUM-VI `room1` sequence into a ROS 2 bag and a VIMS configuration, and copied both to the RDK X5. On this page, you run the VIMS visual-inertial odometry (VIO) node with that bag as its input. You record the estimated trajectory, view it on your workstation, and compare it with the ground truth that comes with the dataset.

You run the VIO node in two ways:

1. **Offline bag mode.** The VIO node reads the bag itself and processes every frame. The result is the same every time you run it, so you can compare your numbers with the ones on this page.
2. **Real-time replay.** `ros2 bag play` publishes the bag at its recorded rate, as a camera would. The VIO node must keep up with the input, and you can see what happens when it can't.

All results on this page are results for the TUM-VI dataset, measured with the VIMS 0.0.6 test package from D-Robotics and the configuration from the previous page. They don't describe the performance of the GS130WI camera or of any other product. VIMS 0.0.6 is a validation release, so its parameters can change in later releases.

## Set up the environment on the RDK X5

Run all commands on this page as the `sunrise` user. You don't need `sudo`.

In each new terminal on the **RDK X5**, source TogetheROS.Bot and VIMS, and set the ROS 2 domain and middleware that VIMS uses:

```bash
source /opt/tros/humble/local_setup.bash
source /userdata/vims/install/local_setup.bash
export ROS_DOMAIN_ID=42
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
```

The VIMS installation adds the last two settings to the `.bashrc` file of the `root` user. Set them yourself here, because the `sunrise` shell doesn't read that file.

Check that the bag and the configuration are in place:

```bash
ls ~/lp2-tumvi/room1_512_vertical ~/lp2-tumvi/config/tumvi_512
```

```output
/home/sunrise/lp2-tumvi/config/tumvi_512:
estimator_config.yaml  tumvi_512-camchain-imucam.yaml  tumvi_512-imu.yaml

/home/sunrise/lp2-tumvi/room1_512_vertical:
metadata.yaml  metadata.v8.yaml  room1_512_vertical.db3
```

## Run the VIO node in offline bag mode

The VIO node subscribes to one combined stereo image, `/sub_image_combine_raw`, and one IMU topic. In offline bag mode, it reads these messages from the bag instead of subscribing, and it waits for each frame to finish before it reads the next one.

In a first terminal on the **RDK X5**, start recording the VIO output:

```bash
mkdir -p ~/lp2-vio
ros2 bag record -o ~/lp2-vio/offline /drobotics_vio/poseimu /drobotics_vio/odomimu
```

In a second terminal on the **RDK X5**, start the VIO node in offline bag mode:

```bash
ros2 launch drobotics_vio subscribe.launch.py \
  config_path:=$HOME/lp2-tumvi/config/tumvi_512/estimator_config.yaml \
  auto_gen_config:=False downsample_cameras_already:=False \
  base_to_imu_tf_required:=False use_sim_time:=True \
  offline_bag_path:=$HOME/lp2-tumvi/room1_512_vertical
```

The arguments do the following:

- `config_path` points to the configuration you created from the TUM-VI calibration. With `auto_gen_config:=False`, the node uses that file instead of building a configuration from camera information messages.
- `downsample_cameras_already:=False` tells the node that the input images are at the resolution in the calibration file. The launch file default overrides the value in the configuration file, so you need to set it here.
- `base_to_imu_tf_required:=False` lets the node run without a robot base transform, because there is no robot here.
- `use_sim_time:=True` is required in offline bag mode. This is a limit of the current implementation. The bag was recorded in 2018. According to D-Robotics, with the system clock the node treats the first IMU messages as expired and discards them, and image processing waits for the first IMU message. In a test with `use_sim_time:=False`, the node read the whole bag in a few seconds and published no poses. Offline bag mode doesn't publish a `/clock` topic, so with `use_sim_time:=True` the ROS clock of the node stays at zero, and the log shows `now ts: 0.0000`. According to D-Robotics, the filter advances on the measurement times of the images and the IMU, so `/drobotics_vio/poseimu` carries the dataset time stamps. The clock still affects other outputs: `/drobotics_vio/pathimu` carries the node time, so its time stamps are zero in this mode. Use `poseimu` for the evaluation.

At startup the node prints its configuration. Look for these lines:

```output
[drobotics_vio_node-1] [WARN] [...] [trial_guard]: Trial guard: validating against platform X5 (time_limit_sec=10800)
...
[drobotics_vio_node-1] [...][WARN] cam_0_fisheye:1
[drobotics_vio_node-1] [...][WARN] cam_1_fisheye:1
```

The `trial_guard` line shows that VIMS checks that it runs on an RDK X5. The 0.0.6 test package also limits each run to 10,800 seconds (three hours), counted from the start of the node. When the limit is reached, the VIO and RTAB-Map nodes shut down. The limit applies to each run separately, so you can start the node again. A run on this page takes less than five minutes. D-Robotics has no release date for a later version, so this Learning Path stays on 0.0.6.

The node processes the whole 141-second sequence. In the run for this Learning Path, this took about 254 seconds on the RDK X5. When the node reaches the end of the bag, it prints a summary and exits:

```output
[drobotics_vio_node-1] [WARN] [...] [drobotics_vio.VioNode]: ================ replay_bag summary ================
[drobotics_vio_node-1]   bag_path        : /home/sunrise/lp2-tumvi/room1_512_vertical
[drobotics_vio_node-1]   imu messages    : 28122
[drobotics_vio_node-1]   image messages  : 2821
[drobotics_vio_node-1]   skipped         : 0
[drobotics_vio_node-1]   total routed    : 30943
[drobotics_vio_node-1] =====================================================
...
[INFO] [drobotics_vio_node-1]: process has finished cleanly [pid ...]
```

Every one of the 2,821 stereo frames and 28,122 IMU samples reached the estimator.

Stop the recorder with **Ctrl+C**, and check what it recorded:

```bash
ros2 bag info ~/lp2-vio/offline
```

```output
...
Topic information: Topic: /drobotics_vio/poseimu | Type: geometry_msgs/msg/PoseWithCovarianceStamped | Count: 2799 | Serialization Format: cdr
                   Topic: /drobotics_vio/odomimu | Type: nav_msgs/msg/Odometry | Count: 6895 | Serialization Format: cdr
```

The node published one pose for each frame after it initialized, 2,799 poses in total. The poses cover 140 seconds of dataset time, so the trajectory has 20 poses per second of dataset time, the same rate as the camera in the dataset. The node took about 250 seconds of wall-clock time for them, so it processed about 11 frames per second. The number of `odomimu` messages can differ by one or two between runs, because the node publishes them at a fixed wall-clock rate. The 2,799 poses are the same in every run.

## Compare the trajectory with the ground truth

TUM-VI includes motion-capture poses of the IMU for the whole `room1` sequence. Compare the VIO trajectory with them on your workstation.

On your **workstation**, copy the recorded bag:

```bash
scp -r sunrise@<x5-address>:lp2-vio/offline ~/tumvi/vio-offline
```

Save the comparison script. It reads the `/drobotics_vio/poseimu` messages, pairs each one with the nearest ground-truth pose, aligns the two trajectories with a rotation and a translation, and prints the absolute trajectory error (ATE) of the positions. The ATE is the distance between each aligned estimated position and its ground-truth position. The script prints the root mean square error (RMSE), the median, and the maximum of these distances:

```bash
cat > ~/tumvi/eval_vio_vs_mocap.py <<'EOF'
#!/usr/bin/env python3
"""Compare /drobotics_vio/poseimu with TUM-VI mocap ground truth: SE(3) alignment, position ATE."""
import argparse
import csv
from pathlib import Path

import numpy as np
from rosbags.rosbag2 import Reader
from rosbags.typesys import Stores, get_typestore


def umeyama_se3(src, dst):
    mu_s, mu_d = src.mean(axis=0), dst.mean(axis=0)
    u, _, vt = np.linalg.svd((dst - mu_d).T @ (src - mu_s) / len(src))
    s = np.eye(3)
    if np.linalg.det(u) * np.linalg.det(vt) < 0:
        s[2, 2] = -1
    r = u @ s @ vt
    return r, mu_d - r @ mu_s


def main():
    ap = argparse.ArgumentParser()
    ap.add_argument('bag', type=Path)
    ap.add_argument('mav0', type=Path)
    args = ap.parse_args()

    ts = get_typestore(Stores.ROS2_HUMBLE)
    est_t, est_p = [], []
    with Reader(args.bag) as r:
        conns = [c for c in r.connections if c.topic == '/drobotics_vio/poseimu']
        for c, _, raw in r.messages(connections=conns):
            m = ts.deserialize_cdr(raw, c.msgtype)
            est_t.append(m.header.stamp.sec + m.header.stamp.nanosec * 1e-9)
            p = m.pose.pose.position
            est_p.append([p.x, p.y, p.z])
    est_t, est_p = np.array(est_t), np.array(est_p)

    with open(args.mav0 / 'mocap0' / 'data.csv', newline='') as f:
        rows = [row for row in csv.reader(f) if row and not row[0].startswith('#')]
    gt_t = np.array([int(row[0]) * 1e-9 for row in rows])
    gt_p = np.array([[float(x) for x in row[1:4]] for row in rows])

    idx = np.clip(np.searchsorted(gt_t, est_t), 1, len(gt_t) - 1)
    idx = np.where(np.abs(gt_t[idx - 1] - est_t) < np.abs(gt_t[idx] - est_t), idx - 1, idx)
    ok = np.abs(gt_t[idx] - est_t) <= 0.01
    e, g = est_p[ok], gt_p[idx[ok]]
    r, t = umeyama_se3(e, g)
    err = np.linalg.norm(e @ r.T + t - g, axis=1)
    length = lambda p: np.linalg.norm(np.diff(p, axis=0), axis=1).sum()
    print(f'poses: {len(est_t)}, matched to ground truth: {int(ok.sum())}')
    print(f'ATE: rmse={np.sqrt((err ** 2).mean()):.3f} m  median={np.median(err):.3f} m  max={err.max():.3f} m')
    print(f'path length: estimate={length(e):.1f} m  ground truth={length(g):.1f} m')


if __name__ == '__main__':
    main()
EOF
```

Run it with the Python environment from the previous page:

```bash
~/tumvi/venv/bin/python ~/tumvi/eval_vio_vs_mocap.py ~/tumvi/vio-offline ~/tumvi/dataset-room1_512_16/mav0
```

```output
poses: 2799, matched to ground truth: 2742
ATE: rmse=0.274 m  median=0.181 m  max=0.586 m
path length: estimate=144.3 m  ground truth=146.4 m
```

In four runs for this Learning Path, on three different days, the offline bag mode produced exactly the same 2,799 positions, so every run gave these numbers. Over the poses that match a ground-truth pose, the estimated path is 144 meters long, close to the 146 meters of the ground truth. The person who recorded the sequence walked around one room and returned to the starting point.

The figure shows the same result. On the left, the aligned VIO trajectory lies over the ground truth from above. On the right, the position error stays between about 0.05 and 0.25 meters for most of the first 80 seconds, and grows to between 0.3 and 0.6 meters in the last part of the sequence. The VIO node has no loop closure and no map, so an error that builds up isn't corrected later.

![Two charts for the offline bag mode run on TUM-VI room1. On the left, a top view of the x-y positions in meters: the gray ground truth and the blue VIO estimate after alignment form overlapping loops of about four by three meters, starting and ending near x 0.85, y minus 0.25. On the right, the position error in meters against dataset time in seconds: it stays mostly between 0.05 and 0.25 meters until about 85 seconds, then rises to between 0.3 and 0.59 meters, with a dashed line at the RMSE of 0.274 meters.#center](images/vio-vs-ground-truth.png "The VIO trajectory against the motion capture ground truth, offline bag mode")

The error tells you how this configuration, with this calibration and these image conversions, tracks this sequence. The VIO parameters were tuned by D-Robotics for its own cameras, not for TUM-VI, so don't compare the number with results published for other VIO systems.

## Replay the bag in real time

In offline bag mode the node never misses a frame, because it reads the next frame only when it is ready, and it doesn't drop frames from its queue. A camera doesn't wait. To see how the node behaves when frames arrive at a fixed rate, replay the bag in real time.

The two replay methods are separate, so don't use `offline_bag_path` and `ros2 bag play` together. On the **RDK X5**, start the recorder and the VIO node as before, but without `offline_bag_path`:

```bash
ros2 bag record -o ~/lp2-vio/realtime /drobotics_vio/poseimu /drobotics_vio/odomimu
```

```bash
ros2 launch drobotics_vio subscribe.launch.py \
  config_path:=$HOME/lp2-tumvi/config/tumvi_512/estimator_config.yaml \
  auto_gen_config:=False downsample_cameras_already:=False \
  base_to_imu_tf_required:=False use_sim_time:=True
```

In a third terminal, play the bag. The `--clock` option publishes the bag time on `/clock`, which the node uses because of `use_sim_time:=True`:

```bash
ros2 bag play ~/lp2-tumvi/room1_512_vertical --clock
```

While the bag plays, the node logs a `[TIME]` line about twice per second. Each line shows the processing time of the latest frame, running statistics, and the current pose estimate, for example:

```output
[drobotics_vio_node-1] [WARN] [...] [drobotics_vio.VioNode]: [TIME]: 27.2ms total, min: 10.2ms, max: 126.1ms, avg(excl zupt): 25.4ms. viz: 0.1ms, img ts: 1520530309.450, rpy: [3.0, -2.6, 3.4], ...
```

When the bag ends after 141 seconds, stop the node and the recorder with **Ctrl+C**, and compare the result on your workstation as before.

In seven real-time runs for this Learning Path, the results were:

| Measurement | Offline bag mode | Real-time replay |
| --- | --- | --- |
| Frames that reached the estimator | 2,821 of 2,821 | about 1,300 of 2,821 |
| Poses per second of dataset time | 20 | 9.1 to 9.3 |
| Frames processed per second of wall-clock time | about 11 | about 9 |
| ATE (RMSE) | 0.274 m in every run | 0.35 to 0.57 m |
| Wall-clock time for the sequence | about 254 s | 141 s |

The node needs about 130 milliseconds for a frame in real-time replay, while a new frame arrives every 50 milliseconds. When frames pile up in its image queue, the node drops the older ones, so it processes about every second frame. Which frames it drops depends on timing, so the error is different in every run. The next page measures where this time goes.

{{% notice Note %}}
If you combine the two images side by side instead of one above the other, the node still runs and publishes poses, but the error is much larger. In one run with a side-by-side image, the ATE was 1.23 m. The node expects the left image on top.
{{% /notice %}}

{{% notice Note %}}
This replay doesn't build a 3D map. Mapping in VIMS uses depth from StereoNet, which runs on the BPU, and StereoNet doesn't accept the `mono8` images in this bag. Mapping with the replayed dataset hasn't been verified.
{{% /notice %}}

## View the trajectory on your workstation

You can watch the trajectory and the tracked features while the node runs. Foxglove connects to the RDK X5 over a WebSocket bridge. Use real-time replay for this: `ros2 bag play --clock` publishes the bag time, so `/drobotics_vio/pathimu` has valid time stamps, which it doesn't have in offline bag mode.

In another terminal on the **RDK X5**, after you source the environment, start the bridge:

```bash
ros2 run foxglove_bridge foxglove_bridge --ros-args -p port:=8765
```

On your **workstation**, open [Foxglove](https://foxglove.dev/) in a browser or in the desktop application, and open a connection to `ws://<x5-address>:8765`. Start the VIO node first, so that the bridge advertises its topics, including `/drobotics_vio/pathimu`, `/drobotics_vio/poseimu`, and `/drobotics_vio/trackhist`. Then set up the panels:

- Add a **3D** panel. In its settings, set **Display frame** to `odom`, and turn on `/drobotics_vio/pathimu` in the **Topics** list.
- Add an **Image** panel, and set its topic to `/drobotics_vio/trackhist`. This image shows the left camera on top and the right camera below, with the features that the node tracks.

VIMS also provides a Foxglove layout file at `$(ros2 pkg prefix tros_vision_nav --share)/params/vims-foxglove-layout.json`.

When the panels are ready, start `ros2 bag play` as in the previous section. Turn on the topics before the bag starts: the node publishes `/drobotics_vio/pathimu` only when it processes a frame, so a panel that subscribes after the bag ends receives nothing. When the node stops, its topics disappear and Foxglove clears the panels.

![Foxglove connected to the RDK X5 during real-time replay of TUM-VI room1. The Image panel on the left shows the trackhist image, with the left camera on top and the right camera below, and the tracked features drawn on both. The 3D panel on the right shows the VIO trajectory in the odom frame as a dense set of loops a few meters across, with the current pose drawn as axes.#center](images/foxglove-trajectory-trackhist.png "The trajectory and the tracked features in Foxglove during real-time replay")

The trajectory in the 3D panel is the path of the handheld sensor rig in the room, estimated by the VIO node. It isn't a map of the room.

## Troubleshooting

| Symptom | Check |
| --- | --- |
| `ros2 bag play` or offline bag mode can't open the bag | The bag metadata must be version 5 for ROS 2 Humble. Repeat the conversion on the previous page with the Humble metadata option. |
| The node prints `VIO not initialized` for every frame | The IMU messages aren't arriving. Check that the bag contains `/drobotics_imu/bmi08x_imu`, and that you set `use_sim_time:=True`. |
| You don't see VIO topics on the workstation | Check `ROS_DOMAIN_ID` and `RMW_IMPLEMENTATION` in every terminal, and that Foxglove connects to port 8765 of the RDK X5. |
| The node exits right after it starts | You set `auto_gen_config:=True`, or the `config_path` is wrong. With `auto_gen_config:=True` the node waits for camera information messages that the bag doesn't contain. |
| The error is several meters | Check the image layout: left image on top, and both images in `mono8`. |

## What you have learned

You have:

- Run the VIMS VIO node on the RDK X5 with a public dataset as its input, without a camera.
- Used the offline bag mode to get a result that you can reproduce exactly, and recorded the estimated trajectory.
- Compared the trajectory with the dataset ground truth, and stated what the error number does and doesn't describe.
- Replayed the dataset in real time and seen that the node can't process every frame at the camera rate.

On the next page, you measure how much CPU and memory the VIO node uses on the Arm Cortex-A55 cores, and where its CPU time goes.
