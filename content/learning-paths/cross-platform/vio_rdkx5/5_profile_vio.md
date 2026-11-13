---
title: Measure and profile the VIO workload on Arm
description: Measure the CPU, memory, and output rate of the VIMS visual-inertial odometry node on the Arm Cortex-A55 cores of the RDK X5, and use perf to find where its CPU time goes.
weight: 6

layout: "learningpathall"
---

## Overview

On the previous page, you ran the VIMS VIO node with a replayed dataset and saw that it can't process every frame at the camera rate. On this page, you measure what the node costs on the RDK X5 and where its CPU time goes:

1. Measure the CPU use, memory, and output rate of the VIO node, with the same conditions in every run.
2. Check that the node doesn't use the BPU.
3. Use `perf` to find the functions that use the most CPU time, and map them to the stages of the VIO pipeline.

The VIO node runs entirely on the Arm Cortex-A55 cores. This makes it a good example of a CPU-bound robotics workload. All numbers on this page were measured with the VIMS 0.0.6 test package from D-Robotics, the TUM-VI `room1` sequence, and the configuration from the earlier pages. They describe this workload under these conditions only.

## Fix the CPU frequency

CPU percentages depend on the CPU frequency. The default `schedutil` governor raises and lowers the frequency with the load, and the `performance` governor keeps it at the maximum. On the **RDK X5**, record the current governor so you can restore it later, and then fix the frequency at the maximum:

```bash
cat /sys/devices/system/cpu/cpufreq/policy0/scaling_governor | tee ~/vio_governor.txt
echo performance | sudo tee /sys/devices/system/cpu/cpufreq/policy0/scaling_governor
```

The RDK X5 has one frequency policy for all eight cores, so `policy0` sets all of them.

## Measure the VIO node

The script below runs one real-time replay and measures the VIO node. It starts the node and the bag player, waits 20 seconds for the node to initialize, and then samples the node for 100 seconds:

- CPU and RSS with `pidstat`, once per second
- Proportional set size (PSS) from `/proc/<pid>/smaps_rollup`, once per second
- The BPU load with `hrut_somstatus`, every few seconds
- The VIO output, with `ros2 bag record`

It also summarizes the processing times that the node logs for a sample of the frames.

On the **RDK X5**, save the script:

```bash
cat > ~/vio_measure.sh <<'EOF'
#!/usr/bin/env bash
# Usage: [BAG=<bag>] [CFG=<estimator_config.yaml>] vio_measure.sh <out_dir>
# Background jobs of a non-interactive shell ignore SIGINT, so each ROS process is bounded with timeout.
OUT=$1; mkdir -p "$OUT"
BAG=${BAG:-$HOME/lp2-tumvi/room1_512_vertical}
CFG=${CFG:-$HOME/lp2-tumvi/config/tumvi_512/estimator_config.yaml}
source /opt/tros/humble/local_setup.bash >/dev/null 2>&1
source /userdata/vims/install/local_setup.bash >/dev/null 2>&1
export ROS_DOMAIN_ID=42 RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
timeout -s INT 175 ros2 launch drobotics_vio subscribe.launch.py config_path:=$CFG \
  auto_gen_config:=False downsample_cameras_already:=False base_to_imu_tf_required:=False \
  use_sim_time:=True > "$OUT/vio.log" 2>&1 &
sleep 8
timeout -s INT 165 ros2 bag record -o "$OUT/vio_out" /drobotics_vio/poseimu > "$OUT/record.log" 2>&1 &
sleep 2
timeout -s INT 160 ros2 bag play "$BAG" --clock > "$OUT/play.log" 2>&1 &
sleep 20
VIO=$(pgrep -f "lib/drobotics_vio/drobotics_vio_node")
for i in $(seq 1 100); do echo "$i $(awk '/^Pss:/{print $2}' /proc/$VIO/smaps_rollup)"; sleep 1; done > "$OUT/pss.txt" &
for i in $(seq 1 20); do timeout 3 hrut_somstatus 2>/dev/null | awk '/bpu0:/{print $NF}'; sleep 2; done > "$OUT/bpu.txt" &
pidstat -h -u -r -p $VIO 1 100 > "$OUT/pidstat.txt"
wait
pkill -f "lib/drobotics_vio/drobotics_vio_node"
echo "== $(cat /sys/devices/system/cpu/cpufreq/policy0/scaling_governor), $(cat /sys/devices/system/cpu/cpufreq/policy0/scaling_cur_freq) kHz"
grep -v '^#' "$OUT/pidstat.txt" | awk 'NF>10 {c+=$9; r+=$14; n++} END {printf "VIO CPU=%.1f%% RSS=%.1f MB\n", c/n, r/n/1024}'
awk '{s+=$2; n++} END {printf "VIO PSS=%.1f MB\n", s/n/1024}' "$OUT/pss.txt"
awk '{s+=$1; n++} END {printf "BPU=%.1f%%\n", s/n}' "$OUT/bpu.txt"
grep -oE '\[TIME\]: [0-9.]+ms total' "$OUT/vio.log" | grep -oE '[0-9.]+' | tail -n +20 | sort -n |
  awk '{a[NR]=$1; s+=$1} END {printf "frame time: mean=%.1f ms p50=%.1f ms p95=%.1f ms\n", s/NR, a[int(NR*0.5)], a[int(NR*0.95)]}'
EOF
chmod +x ~/vio_measure.sh
```

The `pidstat` columns used are `%CPU` (field 9) and `RSS` in KB (field 14). 100% CPU is one full core, so the RDK X5 has 800% in total. The frame time is the `total` value of the `[TIME]` lines that the node logs about twice per second, each for its latest frame, without the first 20 lines. It is a sample of the frames, not every frame, and its mean is higher than the average over all frames, as the results below show.

Run three repetitions, and keep other work off the RDK X5 while they run. Each run takes about 3 minutes:

```bash
for r in 1 2 3; do ~/vio_measure.sh ~/vio_runs/r$r; done
```

```output
== performance, 1500000 kHz
VIO CPU=188.7% RSS=85.0 MB
VIO PSS=72.2 MB
BPU=0.0%
frame time: mean=133.8 ms p50=143.6 ms p95=183.0 ms
...
```

`hrut_somstatus` also tries to read some files that only `root` can read, so the script discards its error messages.

To get the pose output rate of the whole replay, count the recorded poses on your workstation, or run `ros2 bag info ~/vio_runs/r1/vio_out` on the RDK X5 and divide the pose count by the duration.

Each `[TIME]` line also reports the running average over all frames so far, `avg(excl zupt)`, and the number of frames, `img count`. To get the average over all frames after the first 20 lines, run this on the **RDK X5**:

```bash
grep '\[TIME\]' ~/vio_runs/r1/vio.log | tail -n +20 | \
  sed -E 's/.*avg\(excl zupt\): ([0-9.]+)ms.*img count: ([0-9]+),.*/\1 \2/' | \
  awk 'NR==1{a=$1;n=$2} END{printf "frames=%d mean=%.1f ms\n", $2-n, ($1*$2-a*n)/($2-n)}'
```

The output is similar to:

```output
frames=1112 mean=114.8 ms
```

## Results

The table shows three real-time runs with the CPU fixed at 1.5 GHz. CPU and memory are from the 100-second window after the 20-second warm-up. Frame time and pose rate cover the whole replay:

| Run | VIO CPU | VIO RSS | VIO PSS | Sampled frame time (mean / p50 / p95) | Pose rate, whole replay |
| --- | --- | --- | --- | --- | --- |
| 1 | 188.7% | 85.0 MB | 72.2 MB | 133.8 / 143.6 / 183.0 ms | 9.25 Hz |
| 2 | 188.4% | 85.3 MB | 72.8 MB | 135.1 / 144.6 / 180.5 ms | 9.26 Hz |
| 3 | 188.8% | 84.9 MB | 72.3 MB | 137.5 / 146.8 / 183.9 ms | 9.11 Hz |

The BPU load was 0% in every sample. This shows only that the VIO node doesn't use the BPU. Other VIMS modules, such as StereoNet depth estimation, do run on the BPU. The bag player, which stands in for the camera, used about 9% CPU and 107 MB RSS. That is the cost of the input, not of the VIO node.

In an earlier set of six runs with the same configuration, which alternated the `performance` and default `schedutil` governors, both governors gave the same results: 189.7 to 190.3% CPU and 9.09 to 9.27 Hz. With both governors, the frequency read 1.5 GHz at the end of each run. It wasn't sampled during the runs.

The same six runs show two things that the table hides:

- **The average over all frames is about 115 ms** (113 to 116 ms), lower than the sampled mean of 130 to 138 ms in those runs. Frames that take longer are probably more likely to be the one that the node logs, but this Learning Path doesn't verify the cause. At about 8.4 frames per second, 115 ms adds up to about one second of processing per second, which fits a node that handles one frame at a time and is always busy.
- **The pose rate isn't constant.** It's about 17 Hz in the first 12 seconds, and 8.3 to 8.4 Hz between 20 and 120 seconds. The whole-replay rate of about 9 Hz is the average of both.

The frame time is processing time. It doesn't include the time that a frame waits in the queue, so it isn't the end-to-end latency.

For comparison, in offline bag mode the node processed all 2,821 frames in about 254 seconds, about 11 frames per second. In three runs, measured over a 30-second window, it used 147.2 to 147.4% CPU and 81 to 83 MB RSS.

## Profile the VIO node with perf

`perf` samples where the CPU spends its cycles. On the RDK X5, `perf_event_paranoid` is set to 2, so as the `sunrise` user you can profile user-space code of your own processes, which is what the VIO node runs.

Start a real-time replay as on the previous page. After the node has run for about 25 seconds, record a 30-second profile of the VIO process on the **RDK X5**:

```bash
VIO=$(pgrep -f "lib/drobotics_vio/drobotics_vio_node")
perf record -F 999 -g -e cycles:u -p $VIO -o ~/vio_perf.data -- sleep 30
```

Show which libraries the cycles were spent in:

```bash
perf report -i ~/vio_perf.data --no-children --sort dso -g none --stdio | grep -E "^\s+[0-9]" | head -8
```

```output
    23.11%  libopencv_video.so.4.5.4d
    22.91%  libopencv_imgproc.so.4.5.4d
    21.68%  libopencv_core.so.4.5.4d
     9.33%  libdrobotics_vio_lib.so
     7.92%  libc.so.6
     7.67%  libopencv_calib3d.so.4.5.4d
     1.97%  libddsc.so.0.10.5
     0.95%  libm.so.6
```

Then show the VIO functions, including the time spent in the functions they call:

```bash
perf report -i ~/vio_perf.data --children --sort dso,sym -g none --stdio | \
  grep -E "ov_msckf::(VioManager|Updater|StateHelper|Propagator)|ov_core::TrackKLT" | head -8 | cut -c1-130
```

```output
    53.21%     0.01%  libdrobotics_vio_lib.so                       [.] ov_msckf::VioManager::track_image_and_update
    43.20%     0.00%  libdrobotics_vio_lib.so                       [.] std::_Function_handler<void (cv::Range const&), ov_core::T
    41.10%     0.01%  libdrobotics_vio_lib.so                       [.] ov_core::TrackKLT::feed_new_camera
    19.28%     0.03%  libdrobotics_vio_lib.so                       [.] ov_core::TrackKLT::feed_stereo
    16.11%     0.10%  libdrobotics_vio_lib.so                       [.] ov_core::TrackKLT::perform_detection_stereo
     9.30%     0.10%  libdrobotics_vio_lib.so                       [.] ov_msckf::VioManager::do_feature_propagate_update
     4.20%     0.04%  libdrobotics_vio_lib.so                       [.] ov_msckf::UpdaterMSCKF::update
     3.02%     0.20%  libdrobotics_vio_lib.so                       [.] ov_msckf::StateHelper::EKFUpdate
```

The first column includes the time in called functions, and the second column is the time in the function itself. The second line is the parallel stereo tracking work that `feed_stereo` runs with OpenCV.

Sort by both library and symbol, as here. With `--sort sym` alone, `perf report` can list the same function more than once, split across entries. In four 30-second profiles, the shares by library stayed within about one percentage point of these values.

## Interpret what you measured

The VIMS VIO node is based on OpenVINS, an open-source filter-based VIO. Its work for each frame has two parts:

- **The front end** finds features in the left and right images and tracks them from frame to frame with the Kanade-Lucas-Tomasi (KLT) method. This is the `TrackKLT` code, and most of its work happens in OpenCV image functions such as optical flow, corner refinement (`cornerSubPix`), and feature detection.
- **The back end** propagates the state with the IMU measurements and updates it with the tracked features in a multi-state constraint Kalman filter (MSCKF). This is the `do_feature_propagate_update` and `UpdaterMSCKF` code, mostly matrix operations with Eigen.

The profile shows that the front end dominates. About three quarters of the cycles are in OpenCV, and the filter update takes about a tenth (8.6 to 10.4% in four profiles). The node uses about 1.9 cores, and with this load it needs about 115 milliseconds for a frame on average, while frames arrive every 50 milliseconds.

The measurements also tell you what doesn't change the result here:

- The CPU governor made no measurable difference in this workload. The frequency read 1.5 GHz at the end of each run with both governors, which suggests that the cores ran at their maximum, but it wasn't sampled during the runs. With other workloads the governor can matter, so keep it fixed when you measure.
- The BPU doesn't take part in VIO, so it can't speed up this node without changes to the software.

To process every frame in real time, the node needs less work per frame or fewer frames, for example fewer features, a lower input frame rate, or smaller images. Those changes affect accuracy. Keep this measurement as the baseline: on the next page, you change one parameter at a time and record the processing speed, the dropped frames, and the trajectory error for each change.

## Limits of this profile

- The system OpenCV libraries don't include symbols, so `perf report` shows some OpenCV functions only as addresses. The library-level shares are still correct.
- `perf` here records user-space cycles (`cycles:u`) only. Time in the kernel, for example in the camera driver, and work on hardware accelerators such as the BPU aren't included.
- The call graphs use frame pointers, and some libraries don't keep them. The `--children` percentages are lower bounds.
- Arm Performix Code Hotspots isn't available on this RDK X5 image, because the kernel doesn't include the tracing file system that it needs. This is the same limit as on the RDK OS image without VIMS.
- All numbers are for the TUM-VI `room1` sequence at 512x512 pixels and 20 frames per second. With a different camera, resolution, or frame rate, the cost per frame and the share of the front end change. Don't compare them with resource figures in the VIMS documentation, which were measured under other conditions.

## Restore the CPU governor

When you finish, restore the governor that you recorded:

```bash
cat ~/vio_governor.txt | sudo tee /sys/devices/system/cpu/cpufreq/policy0/scaling_governor
```

## What you have learned

You have:

- Measured the CPU use, memory, frame time, and output rate of the VIO node, with the same conditions in every run.
- Confirmed that the VIO node runs on the Arm CPU and doesn't use the BPU.
- Used `perf` to find that the image-processing front end, not the filter, uses most of the CPU time.
- Stated the conditions and limits that apply to these numbers.

On the next page, you change the input frame rate and the number of features, and find a setting in which the VIO node keeps up in real time on the Arm CPU.
