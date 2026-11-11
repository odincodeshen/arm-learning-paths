---
title: Tune the VIO workload for the CPU budget of the RDK X5
description: Change the input frame rate and the number of tracked features of the VIMS visual-inertial odometry node, one at a time, and measure how each change affects the processing time per frame, the CPU use, the dropped frames, and the trajectory error on the RDK X5.
weight: 7

layout: "learningpathall"
---

<!--
DRAFT (2026-10-31). Not reviewed. Run as written 2026-10-31 (evidence/run-20261031-01/P6-verify) for the commands
shown (6.7 Hz with 100 features): the conversion gave bags with the same md5 as the experiment bags (and frame-step 1
the same as page 3); the np* configs differ from tumvi_512 only in num_pts; offline real 70.6 s, 934 poses, ATE
0.304 m; real-time 93.2% CPU, frame time mean 74.8 ms, 0 drops, 935 poses, ATE 0.304 m. Not done as written: the scp
of the bags (identical bags already on the X5, md5 checked). The table numbers come from the experiment scripts
(scripts/x5_vio_replay_measure.sh, scripts/x5_vio_offline_bag.sh, scripts/tumvi_to_ros2bag_step.py), which do the
same steps. Evidence: evidence/run-20261031-01 (T1 track_frequency, T1b frame rate,
T3 num_pts), one run per setting; offline mode is deterministic (run-20261025-01, run-20261030-01).
- D-Robotics guidance quoted in substance: reply section 6 (fix a baseline, then change one parameter at a time;
  record speed, dropped frames and trajectory error). Reply v2 section 3 corrects the tracking frequency part;
  VIMS params.md: mipi_image_framerate 6.0, "X5 建议 6.0 平衡算力；提高需降分辨率". Link only to the manual root
  until the params page URL is confirmed.
- track_frequency: no effect with the combined stereo callback (T1); cause from reply v2 section 3, not checked by us.
- 5 Hz: "Detect Image lost, ts diff: 0.20" on every frame. Reply v2: warning only, gap > 0.19 s, hard-coded; it
  doesn't drop frames or stop initialization. The cause of the failed initialization is unknown.
- Inference, labeled on the page: cost per frame rises at lower frame rates because of larger motion between frames.
- Not measured: image resolution (downsample_cameras), 10 Hz with 100 features, num_opencv_threads, CPU affinity.
- Offline per-frame time = wall time of the launch (including a few seconds of startup) / frames in the bag.
-->

## Overview

On the previous page, you found that the VIO node needs about 115 milliseconds of processing for each frame on average, while the dataset delivers a new frame every 50 milliseconds. The node uses about 1.9 of the 8 Cortex-A55 cores, drops about half of the frames, and gives a different trajectory error in every real-time run. Most of its CPU time goes to the image-processing front end.

On this page, you change the workload so that it fits the CPU, without changes to the VIMS software. D-Robotics recommends a method for this: fix a baseline, then change one parameter at a time, and record the processing speed, the dropped frames, and the trajectory error for each change. You do this for two settings:

1. **The input frame rate.** You convert the dataset again with fewer frames per second, as a camera configured for a lower frame rate would deliver them. The IMU data stays at 200 samples per second.
2. **The number of features.** You change `num_pts`, the number of features that the front end extracts and tracks in each camera image.

For each setting, you run the node in offline bag mode, which gives the cost per frame and an error that you can reproduce, and in real-time replay, which shows whether the node keeps up.

All numbers on this page are from one run per setting, with the VIMS 0.0.6 test package, the TUM-VI `room1` sequence, and the configuration from the earlier pages. They describe this dataset and this configuration, not the GS130WI camera.

## Why the frame rate comes from the input

The configuration file has a `track_frequency` parameter, set to 20 for TUM-VI. With the input in this Learning Path, changing it doesn't change how many frames the node processes. With values of 20, 10, 8, and 6, the node printed the new value at startup, but the number of poses, the CPU use, and the time per frame stayed the same in real-time replay. In offline bag mode with a value of 6, the node produced exactly the same 2,799 poses as with 20. According to D-Robotics, only the monocular input and the input with separate left and right topics apply this limit. The combined stereo image doesn't, and offline bag mode uses the same code path.

The input frequency of the VIO node depends on the effective frequency of the image stream that it subscribes to. With a camera, you set it with `mipi_image_framerate`, or with a separate image stream for VIO. If depth estimation and VIO use different image streams, configure each one separately. The parameter reference in the [VIMS user manual](https://d-robotics.github.io/tros_vims_doc/) recommends a camera frame rate of 6 frames per second on the RDK X5 to balance the compute load, and says that a higher frame rate needs a lower resolution. To emulate a camera at a lower frame rate with the dataset, you keep only every second, third, or fourth stereo pair when you convert it.

## Convert the dataset at lower frame rates

On your **workstation**, make a copy of the conversion script from the earlier page:

```bash
cd ~/tumvi
cp tumvi_to_ros2bag.py tumvi_to_ros2bag_step.py
```

Open `tumvi_to_ros2bag_step.py` in an editor and make two changes. After the line that adds the `out` argument, add a `--frame-step` argument:

```python
    ap.add_argument('out', type=Path)
    ap.add_argument('--frame-step', type=int, default=1, help='keep every Nth image pair (default 1 = all)')
```

Then keep only every Nth stereo pair. Change the line that builds the list of image time stamps to:

```python
    stamps = sorted(set(cam0) & set(cam1))[::args.frame_step]
```

The IMU samples stay unchanged. With `--frame-step 1`, the script writes exactly the same bag as before.

Convert the sequence at 10, 6.7, and 5 frames per second:

```bash
for n in 2 3 4; do
  venv/bin/python tumvi_to_ros2bag_step.py dataset-room1_512_16/mav0 room1_512_vertical_s$n --frame-step $n
done
```

```output
images: 1411 (512x1024 mono8), IMU samples: 28122, span: 141.0 s
images: 941 (512x1024 mono8), IMU samples: 28122, span: 141.0 s
images: 706 (512x1024 mono8), IMU samples: 28122, span: 141.0 s
```

Copy the three bags to the RDK X5. Together they are about 1.5 GB:

```bash
scp -r ~/tumvi/room1_512_vertical_s2 ~/tumvi/room1_512_vertical_s3 ~/tumvi/room1_512_vertical_s4 sunrise@<x5-address>:lp2-tumvi/
```

## Create configurations with fewer or more features

The TUM-VI configuration sets `num_pts` to 200. On the **RDK X5**, make one copy of the configuration folder for each value you want to test, and change only that line:

```bash
cd ~/lp2-tumvi/config
for n in 100 150 250 300; do
  cp -r tumvi_512 np$n
  sed -i "s/^num_pts: 200 /num_pts: $n /" np$n/estimator_config.yaml
done
grep -H "^num_pts" */estimator_config.yaml
```

```output
np100/estimator_config.yaml:num_pts: 100 # number of points (per camera) we will extract and try to track
np150/estimator_config.yaml:num_pts: 150 # number of points (per camera) we will extract and try to track
np250/estimator_config.yaml:num_pts: 250 # number of points (per camera) we will extract and try to track
np300/estimator_config.yaml:num_pts: 300 # number of points (per camera) we will extract and try to track
tumvi_512/estimator_config.yaml:num_pts: 200 # number of points (per camera) we will extract and try to track
```

The copied folders keep the camera chain and IMU files, which the estimator settings refer to by name.

## Measure each setting

Use the same environment as on the earlier pages in every terminal on the **RDK X5**.

### Offline bag mode

Offline bag mode processes every frame, so the wall-clock time divided by the number of frames gives the average processing time per frame. Record the output as on the VIO page, and run the node with `time`. For example, for the 6.7 Hz bag with 100 features:

```bash
ros2 bag record -o ~/lp2-vio/off_s3_np100 /drobotics_vio/poseimu
```

```bash
time ros2 launch drobotics_vio subscribe.launch.py \
  config_path:=$HOME/lp2-tumvi/config/np100/estimator_config.yaml \
  auto_gen_config:=False downsample_cameras_already:=False \
  base_to_imu_tf_required:=False use_sim_time:=True \
  offline_bag_path:=$HOME/lp2-tumvi/room1_512_vertical_s3
```

When the node exits, stop the recorder with **Ctrl+C**. The replay summary shows that all 941 frames reached the estimator, and `time` prints the wall-clock time:

```output
real	1m10.568s
```

Divide the `real` time by the number of frames in the bag, for example 71 seconds for 941 frames is about 75 milliseconds per frame. Then copy the recorded bag to your workstation and compute the error with the comparison script from the VIO page:

```bash
scp -r sunrise@<x5-address>:lp2-vio/off_s3_np100 ~/tumvi/
~/tumvi/venv/bin/python ~/tumvi/eval_vio_vs_mocap.py ~/tumvi/off_s3_np100 ~/tumvi/dataset-room1_512_16/mav0
```

```output
poses: 934, matched to ground truth: 913
ATE: rmse=0.304 m  median=0.215 m  max=0.903 m
path length: estimate=146.9 m  ground truth=145.3 m
```

### Real-time replay

Use the measurement script from the profiling page, with the CPU governor fixed at `performance` as on that page. The script reads the bag and the configuration from `BAG` and `CFG`, so set them for each run:

```bash
BAG=~/lp2-tumvi/room1_512_vertical_s3 CFG=~/lp2-tumvi/config/np100/estimator_config.yaml \
  ~/vio_measure.sh ~/vio_runs/rt_s3_np100
grep -c "drop image" ~/vio_runs/rt_s3_np100/vio.log
```

```output
== performance, 1500000 kHz
VIO CPU=93.2% RSS=80.2 MB
VIO PSS=67.6 MB
BPU=0.0%
frame time: mean=74.8 ms p50=87.0 ms p95=134.9 ms
0
```

The last command counts the frames that the node dropped from its image queue. Each dropped frame logs a line such as `drop image: ..., que size: 3`. Compute the error of the recorded `vio_out` bag on your workstation as before.

The `frame time` line is the sampled value. To get the average over all frames, which the tables below use, run the command from the profiling page on `~/vio_runs/rt_s3_np100/vio.log`. For this run, it gives 68.8 ms.

When you finish all runs, restore the governor as described on the profiling page.

## Results: input frame rate

The table shows the TUM-VI configuration with 200 features at four input frame rates. The 20 Hz row is the baseline from the earlier pages:

| Input frame rate (frame interval) | Offline: time per frame | Offline: ATE | Real-time: dropped frames | Real-time: VIO CPU | Real-time: ATE |
| --- | --- | --- | --- | --- | --- |
| 20 Hz (50 ms) | about 90 ms | 0.274 m | about 1,530 of 2,821 | 190% | 0.35 to 0.57 m |
| 10 Hz (100 ms) | about 120 ms | 0.301 m | 256 of 1,411 | 185% | 0.446 m |
| 6.7 Hz (150 ms) | about 138 ms | 0.736 m | 3 of 941 | 178% | 0.732 m |
| 5 Hz (200 ms) | not initialized | none | not initialized | 51% | none |

At 6.7 Hz, the processing time per frame is shorter than the frame interval, so the node keeps up. It dropped only 3 frames, and its real-time error matches the offline error. At 10 and 20 Hz, it can't keep up and drops frames.

The time per frame grows as the frame rate falls, from about 90 to about 138 milliseconds. A likely reason, which this Learning Path doesn't verify, is that the camera moves further between two frames, so the front end must track features over a longer distance and detect more new ones. As a result, the node still uses about 1.8 cores at 6.7 Hz. Lowering the frame rate made the node keep up, but it didn't free much CPU.

{{% notice Note %}}
At 5 Hz, the node never initializes on this sequence, in either mode. It also logs `Detect Image lost, ts diff: 0.20` for every frame. According to D-Robotics, this warning appears when the gap between two images is more than 0.19 seconds, and it doesn't drop frames or stop the initialization. So the warning doesn't explain the failure, and this Learning Path didn't find the cause.
{{% /notice %}}

The result agrees with the VIMS parameter reference, which uses 6 frames per second on the RDK X5 at a resolution of 640x352 pixels per camera. TUM-VI has 512x512 pixels per camera, a similar number of pixels. The conditions aren't the same, so treat this as agreement, not as a measurement of the VIMS default.

## Results: number of features

The table shows the 6.7 Hz bag, the only frame rate at which the baseline keeps up, with five values of `num_pts`. The row for 200 is from the previous table:

| `num_pts` | Offline: time per frame | Offline: ATE | Real-time: time per frame (all-frames mean) | Real-time: VIO CPU | Real-time: dropped frames | Real-time: ATE |
| --- | --- | --- | --- | --- | --- | --- |
| 100 | about 75 ms | 0.304 m | 69 ms | 94% | 0 | 0.304 m |
| 150 | about 111 ms | 0.436 m | 106 ms | 140% | 0 | 0.437 m |
| 200 | about 138 ms | 0.736 m | 136 ms | 178% | 3 | 0.732 m |
| 250 | about 167 ms | 0.828 m | 166 ms | 187% | 100 | 0.637 m |
| 300 | about 190 ms | 0.903 m | 192 ms | 186% | 195 | 1.080 m |

The processing time per frame grows almost in proportion to the number of features. This matches the profile on the previous page, where the feature tracking front end used most of the CPU time.

With 100 or 150 features, each frame takes less than the 150-millisecond interval. The node drops no frames, and its real-time error is the same as its offline error. With 250 or 300 features, it can't keep up, and the real-time error depends on which frames it drops.

With 100 features at 6.7 Hz, the VIO node uses less than one of the eight Cortex-A55 cores. That is about half the CPU of the 200-feature setting, and it leaves the other cores for other work. This Learning Path doesn't measure how much of that the rest of VIMS needs. For this sequence, images at 6.7 Hz, the IMU at 200 Hz, and 100 features are the verified lightweight configuration.

## Interpret the results

Two rules explain the measurements:

- **The node keeps up when its time per frame is shorter than the frame interval.** Otherwise frames pile up in its queue and it drops them, and the result changes from run to run.
- **Its CPU use is roughly its time per frame multiplied by the frame rate.** You can reduce the time per frame with fewer features, and the frame interval sets how much time each frame may take.

The error needs more care. On this sequence, fewer features and a higher frame rate both gave a lower error in offline bag mode. That doesn't mean fewer features are better in general. The estimator settings come from the VIMS configuration for its SC132GS stereo camera, including the feature grid and the minimum distance between features, and weren't tuned for the TUM-VI images. On another sequence or with another camera, the best number can be different. Each setting was run once, and offline bag mode gives the same result in every run.

To choose an operating point for your own application, use the same method: start from a baseline that you have measured, change one parameter at a time, and record the time per frame, the CPU use, the dropped frames, and the error. Other parameters that the VIMS documentation names, such as the image resolution, weren't measured here.

## What you have learned

You have:

- Found that with a combined stereo image, the frequency of the input stream, not `track_frequency`, sets how many frames the VIMS 0.0.6 VIO node processes.
- Converted the dataset at lower frame rates to emulate a camera configuration, and found the frame rate at which the node keeps up on the RDK X5.
- Changed the number of features and seen that the processing time per frame, and so the CPU use, follows it almost in proportion.
- Measured a setting in which the VIO node keeps up in real time with less than one Cortex-A55 core, and stated the limits of that result.
