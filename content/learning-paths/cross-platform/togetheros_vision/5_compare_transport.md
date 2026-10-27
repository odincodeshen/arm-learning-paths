---
title: Measure the TogetheROS local vision pipeline
description: Run the local image-processing example with TogetheROS data sharing and standard ROS 2 image messages, then interpret its CPU, memory, and delivery measurements.
weight: 6

layout: "learningpathall"
---

## Overview

You have a working stored-image publisher and inference node, and you verified their shared image buffer. Now run that local TogetheROS vision workflow at 5 and 30 frames per second. Use the nodes' two message configurations to observe delivery, CPU, memory, and delay under a known workload.

Both configurations use shared memory in some form. The data-sharing arm lets subscribers read the publisher's buffer directly. The Image-message arm uses the Fast DDS shared-memory transport, which serializes each message and copies it through shared segments. These are the configurations exposed by the TogetheROS example.

Keep the image, rate, subscriber work, and CPU frequency aligned so the observations are useful. The nodes also change message type and QoS when you switch configurations, which limits what the measurements can attribute to transport.

## Design the two arms

Both arms use the same two nodes. You change only the two TogetheROS shared-memory parameters, but that also changes the message type and the transport path:

| | Data-sharing arm | Image-message arm |
| --- | --- | --- |
| Publisher | `hobot_image_pub`, `is_shared_mem:=True` | `hobot_image_pub`, `is_shared_mem:=False` |
| Topic and type | `/hbmem_img`, `hbm_img_msgs/msg/HbmMsg1080P` | `/image_nv12`, `sensor_msgs/msg/Image` |
| Subscriber | `dnn_node_example`, `is_shared_mem_sub:=1` | `dnn_node_example`, `is_shared_mem_sub:=0` |
| Image | `config/target.jpg` as NV12, 640x640 | Same |
| Subscriber work | Object detection on the board's accelerator, plus CPU pre- and postprocessing | Same |
| Fast DDS environment | TogetheROS `shm_fastdds.xml` profile | Same |
| Transport path | Fast DDS data sharing: subscribers map the publisher's pool | Fast DDS shared-memory transport: serialize, fragment, copy, reassemble, deserialize |

The inference node performs the same downstream vision work after it receives a frame in either configuration.

## Prepare the RDK X5

Run all commands on this page on the **RDK X5**.

CPU frequency scaling changes CPU percentages a lot. In a test run with the default `schedutil` governor, the subscriber used about 15% CPU at 5 frames per second. With the CPU fixed at 1.5 GHz, it used about 4.5%. Fix the frequency for the measurement. Record the current governor first, so you can restore it later:

```bash
cat /sys/devices/system/cpu/cpufreq/policy0/affected_cpus
cat /sys/devices/system/cpu/cpufreq/policy0/scaling_governor | tee ~/lp1_governor.txt
echo performance | sudo tee /sys/devices/system/cpu/cpufreq/policy0/scaling_governor
cat /sys/devices/system/cpu/cpufreq/policy0/scaling_cur_freq
```

```output
0 1 2 3 4 5 6 7
schedutil
performance
1500000
```

On the RDK X5, a single frequency policy covers all eight cores.

Prepare a working directory and the environment in each terminal you use:

```bash
mkdir -p ~/lp1_shm && cd ~/lp1_shm
cp -r /opt/tros/humble/lib/dnn_node_example/config .
source /opt/tros/humble/setup.bash
export ROS_DOMAIN_ID=42
export RMW_IMPLEMENTATION=rmw_fastrtps_cpp
export FASTRTPS_DEFAULT_PROFILES_FILE=/opt/tros/humble/lib/hobot_shm/config/shm_fastdds.xml
export RMW_FASTRTPS_USE_QOS_FROM_XML=1
export ROS_DISABLE_LOANED_MESSAGES=0
```

The last four variables are what `hobot_shm.launch.py` sets inside TogetheROS launch files. You set them yourself here because you start the nodes with `ros2 run`. Use the same values in both arms.

## Run the data-sharing arm

Start the subscriber first:

```bash
ros2 run dnn_node_example example --ros-args --log-level warn \
  -p config_file:=config/fcosworkconfig.json -p feed_type:=1 \
  -p is_shared_mem_sub:=1 -p sharedmem_img_topic_name:=/hbmem_img
```

In a second terminal, with the same environment, start the publisher at 30 frames per second:

```bash
ros2 run hobot_image_publisher hobot_image_pub --ros-args --log-level warn \
  -p image_source:=config/target.jpg -p image_format:=jpg -p pub_encoding:=nv12 \
  -p msg_pub_topic_name:=/hbmem_img -p fps:=30 -p is_loop:=True -p is_shared_mem:=True
```

After a 10-second warm-up, sample both processes for 60 seconds in a third terminal:

```bash
PUB=$(pgrep -f "^/opt/tros/humble/lib/hobot_image_publisher/hobot_image_pub")
SUB=$(pgrep -f "^/opt/tros/humble/lib/dnn_node_example/example")
pidstat -h -u -r -p $PUB,$SUB 1 60
```

The subscriber log reports the received rate and processing times about once per second:

```output
[WARN] [...] [dnn_example_node]: Sub img fps: 31.50, Smart fps: 31.53, recved img time delay ms: 0, pre process time ms: 2, infer time ms: 3, post process time ms: 3, pipeline time ms: 10
```

Stop both nodes with Ctrl+C.

## Run the Image-message arm

Repeat the same steps with the shared-memory parameters set to off in both nodes. The subscriber reads `sensor_msgs/msg/Image` from `ros_img_topic_name`:

```bash
ros2 run dnn_node_example example --ros-args --log-level warn \
  -p config_file:=config/fcosworkconfig.json -p feed_type:=1 \
  -p is_shared_mem_sub:=0 -p ros_img_topic_name:=/image_nv12
```

```bash
ros2 run hobot_image_publisher hobot_image_pub --ros-args --log-level warn \
  -p image_source:=config/target.jpg -p image_format:=jpg -p pub_encoding:=nv12 \
  -p msg_pub_topic_name:=/image_nv12 -p fps:=30 -p is_loop:=True -p is_shared_mem:=False
```

Confirm that this arm sends the same image. The `--no-arr` option prints the array length instead of the 614,400 values:

```bash
ros2 topic echo --once --no-arr /image_nv12
```

```output
header:
  stamp:
    ...
  frame_id: config/target.jpg
height: 640
width: 640
encoding: nv12
is_bigendian: 0
step: 0
data: '<sequence type: uint8, length: 614400>'
---
```

This matches the data-sharing arm, where the subscriber log reports `nv12, h: 640, w: 640` and `data size: 614400`.

## Check what didn't match

Compare the topic details of the two arms:

```bash
ros2 topic info -v /image_nv12 | grep -E "Type|Node name|Reliability"
```

| Condition | Data-sharing arm | Image-message arm | Matched? |
| --- | --- | --- | --- |
| Image, size, encoding | 640x640 NV12, 614,400 bytes | Same | Yes |
| Publish and receive rate | 5 or 30 fps | Same | Yes |
| Subscriber work | Same node, model, and config | Same | Yes |
| CPU frequency | 1.5 GHz fixed | Same | Yes |
| Message type | Fixed-size `HbmMsg1080P` | Variable-size `sensor_msgs/msg/Image` | No. This difference is how the two transport modes work. |
| QoS reliability | BEST_EFFORT | RELIABLE | No |
| QoS history depth | 5 (`SensorDataQoS`) | 10 | No |

Both nodes set these QoS policies in code. In data-sharing mode they use `rclcpp::SensorDataQoS()`, and in Image-message mode they use a depth of 10 with default reliability. `rmw_fastrtps` always applies QoS that a node sets in code unless the node uses `SYSTEM_DEFAULT`. `RMW_FASTRTPS_USE_QOS_FROM_XML=1` only lets the XML profile set the history memory policy and the publication mode. A shared Fast DDS profile that set reliability to BEST_EFFORT therefore didn't change either arm. To align the QoS, you would need to change the code of both nodes, for example to use the same `SensorDataQoS()` in both arms, and rebuild them. That is outside the scope of this Learning Path. The results below compare the two modes as TogetheROS ships them. Even with aligned QoS, the message types and the work to build each message would still differ, so the difference couldn't all be attributed to transport.

You can also check how the Image-message arm moves data. During an Image-message run, the loopback interface carried about 1 KB/s, while the image stream was about 3 MB/s at 5 frames per second. This arm therefore doesn't use UDP over loopback. The profile on the next page confirms that it uses the built-in Fast DDS shared-memory transport. That transport serializes each image, splits it into fragments, copies the fragments through shared segments, and then reassembles and deserializes them in the subscriber. The profile of the data-sharing arm shows none of these copies.

## Repeat the measurement

Running each arm once by hand is enough to see that both work. For numbers you can compare, run each configuration several times, in alternating order, with the same warm-up and measurement window. The script below runs one arm. It starts both nodes and waits 10 seconds to warm up. During the measurement window it samples CPU and RSS with `pidstat`, and proportional set size (PSS) from `/proc/<pid>/smaps_rollup`, once per second. It then stops the nodes and summarizes the results.

On the **RDK X5**, save the script:

```bash
cat > ~/run_arm.sh <<'EOF'
#!/usr/bin/env bash
# Usage: run_arm.sh <shm|std> <fps> <seconds> <out_dir>
ARM=$1; FPS=$2; SECS=$3; OUT=$4
mkdir -p "$OUT"; cd ~/lp1_shm
source /opt/tros/humble/setup.bash >/dev/null 2>&1
export ROS_DOMAIN_ID=42 RMW_IMPLEMENTATION=rmw_fastrtps_cpp
export FASTRTPS_DEFAULT_PROFILES_FILE=/opt/tros/humble/lib/hobot_shm/config/shm_fastdds.xml
export RMW_FASTRTPS_USE_QOS_FROM_XML=1 ROS_DISABLE_LOANED_MESSAGES=0
if [ "$ARM" = shm ]; then PS=True; SS=1; T=/hbmem_img; else PS=False; SS=0; T=/image_nv12; fi
ros2 run dnn_node_example example --ros-args --log-level warn \
  -p config_file:=config/fcosworkconfig.json -p feed_type:=1 -p is_shared_mem_sub:=$SS \
  -p sharedmem_img_topic_name:=/hbmem_img -p ros_img_topic_name:=/image_nv12 > "$OUT/sub.log" 2>&1 &
sleep 3
ros2 run hobot_image_publisher hobot_image_pub --ros-args --log-level warn \
  -p image_source:=config/target.jpg -p image_format:=jpg -p pub_encoding:=nv12 \
  -p msg_pub_topic_name:=$T -p fps:=$FPS -p is_loop:=True -p is_shared_mem:=$PS > "$OUT/pub.log" 2>&1 &
sleep 10
PUB=$(pgrep -f "^/opt/tros/humble/lib/hobot_image_publisher/hobot_image_pub")
SUB=$(pgrep -f "^/opt/tros/humble/lib/dnn_node_example/example")
T0=$(date +%s.%N)
for i in $(seq 1 $SECS); do
  echo "$i $(awk '/^Pss:/{print $2}' /proc/$PUB/smaps_rollup) $(awk '/^Pss:/{print $2}' /proc/$SUB/smaps_rollup)"
  sleep 1
done > "$OUT/pss.txt" &
PSSPID=$!
pidstat -h -u -r -p $PUB,$SUB 1 $SECS > "$OUT/pidstat.txt"
T1=$(date +%s.%N)
wait $PSSPID
kill $PUB $SUB; sleep 2
echo "== $ARM ${FPS} fps, window $SECS s"
grep -v '^#' "$OUT/pidstat.txt" | awk 'NF>10 {c[$NF]+=$9; r[$NF]+=$14; n[$NF]++}
  END {for (k in c) printf "%s CPU=%.1f%% RSS=%.1f MB\n", k, c[k]/n[k], r[k]/n[k]/1024}'
awk '{p+=$2; s+=$3; n++} END {printf "PSS pub=%.1f MB sub=%.1f MB\n", p/n/1024, s/n/1024}' "$OUT/pss.txt"
# Use only log lines whose ROS timestamp falls inside the measurement window
grep "Sub img fps" "$OUT/sub.log" | sed -E 's/^\[WARN\] \[([0-9.]+)\].*Sub img fps: ([0-9.]+).*recved img time delay ms: ([0-9]+).*pipeline time ms: ([0-9]+).*/\1 \2 \3 \4/' \
  | awk -v t0=$T0 -v t1=$T1 '$1>=t0 && $1<=t1 {f+=$2; d+=$3; p+=$4; n++}
    END {printf "sub_fps=%.2f recv_delay_ms=%.2f pipeline_ms=%.2f (log lines in window: %d)\n", f/n, d/n, p/n, n}'
grep "comm delay" "$OUT/sub.log" | sed -E 's/^\[WARN\] \[([0-9.]+)\].*comm delay \[([0-9.]+)\]ms.*/\1 \2/' \
  | awk -v t0=$T0 -v t1=$T1 '$1>=t0 && $1<=t1 {s+=$2; n++; if ($2>m) m=$2}
    END {if (n) printf "comm_delay_ms mean=%.3f max=%.3f (samples in window: %d)\n", s/n, m, n}'
EOF
chmod +x ~/run_arm.sh
```

The `pidstat` columns used are `%CPU` (field 9) and `RSS` in KB (field 14). The `-h` option prints the time as two fields. PSS divides shared pages among the processes that map them, so unlike RSS it doesn't count the shared pool twice. The subscriber log prints a summary line about once per second. The script averages only the log lines whose timestamps fall inside the measurement window, so CPU, memory, and log values all exclude the warm-up.

Run three repetitions. Each repetition runs all four configurations in turn:

```bash
for r in 1 2 3; do
  for cfg in "shm 5" "std 5" "shm 30" "std 30"; do
    set -- $cfg
    ~/run_arm.sh $1 $2 60 ~/lp1_eq/$1-$2fps-r$r
  done
done
```

Each configuration takes about 75 seconds, so the full set takes about 15 minutes. Keep other work off the RDK X5 while it runs. Each output folder keeps the raw `pidstat` samples and node logs.

## Results

Each cell shows three 60-second measurement windows, each after a 10-second warm-up. The four configurations ran interleaved in each repetition, with the CPU fixed at 1.5 GHz. CPU is per process, where 100% is one full core. RSS counts every page a process maps. PSS divides shared pages among the processes that map them.

| Arm | Rate | Received per second | Publisher CPU | Subscriber CPU | Reported receive delay | Pipeline time |
| --- | --- | --- | --- | --- | --- | --- |
| Data sharing | 5 fps | 5.00 / 5.00 / 5.00 | 0.3 / 0.3 / 0.3% | 4.4 / 4.4 / 4.4% | 0.04 / 0.17 / 0.24 ms | 11.0 / 10.4 / 10.3 ms |
| Image message | 5 fps | 5.00 / 5.00 / 5.00 | 0.8 / 0.8 / 0.8% | 4.8 / 4.8 / 4.8% | 1.67 / 1.70 / 1.00 ms | 11.8 / 12.1 / 11.6 ms |
| Data sharing | 30 fps | 30.31 / 30.31 / 30.31 | 1.5 / 1.5 / 1.6% | 24.0 / 24.0 / 23.9% | 0.19 / 0.31 / 0.29 ms | 10.3 / 10.3 / 10.3 ms |
| Image message | 30 fps | 30.31 / 30.31 / 30.31 | 4.7 / 4.6 / 4.7% | 26.7 / 26.9 / 26.6% | 1.43 / 1.69 / 1.00 ms | 11.6 / 11.7 / 11.2 ms |

Memory, averaged over the same windows:

| Arm | Publisher RSS | Publisher PSS | Subscriber RSS | Subscriber PSS |
| --- | --- | --- | --- | --- |
| Data sharing, 5 and 30 fps | 77 MB | 48.6 to 48.7 MB | 71.5 to 72.2 MB | 45.8 to 47.5 MB |
| Image message, 5 and 30 fps | 75 MB | 48.1 to 48.2 MB | 69.2 to 69.6 MB | 45.2 to 47.1 MB |

"Reported receive delay" and "Pipeline time" come from the `dnn_node_example` log. According to the `dnn_node_example` 2.6.1 source code:

- `recved img time delay ms` is the time from the message timestamp to the start of preprocessing in the subscriber callback. It includes middleware, queuing, and scheduling time, not only the shared-memory transfer.
- `pipeline time ms` is the time from the message timestamp to the end of postprocessing for that frame. It doesn't include publishing the detection results or any downstream display.
- `comm delay`, logged only in data-sharing mode, is the time from the message timestamp to a point early in the callback. It's printed at most once every 3 seconds.

The image publisher sets the message timestamp after it copies the image into the message, so none of these values include reading the file, converting the image, or that copy. The node reports whole milliseconds for the first two values, and the table averages the log lines in the window. Treat these values as indicators, not as precise sub-millisecond transport measurements.

The received rate of about 30.3 per second, instead of 30, comes from the publisher's timer. It uses an integer period of `1000 / fps` milliseconds, which is 33 ms at 30 fps, or about 30.3 messages per second.

## Interpret the results

What the data shows, for this pipeline as shipped:

- Both modes deliver messages at the publish rate, about 5 and 30.3 per second, and the inference time that the node reports is the same in both arms, so the subscriber does the same work. A matching rate doesn't prove that no message was dropped or delayed.
- In data-sharing mode, the publisher used less CPU: 1.5% to 1.6% versus 4.6% to 4.7% of a core at 30 frames per second. The subscriber also used less: about 24% versus about 27%.
- The reported receive delay was lower in data-sharing mode: 0.04 to 0.31 ms, versus 1.00 to 1.70 ms in Image-message mode. Because the node reports whole milliseconds per log line, read these averages as a direction, not as precise values.
- RSS is about 2 to 3 MB higher per process in data-sharing mode, but PSS is almost the same in both modes. RSS counts the shared pool pages in each process that maps them, so compare total memory cost with PSS.

Why you can't attribute all of the difference to transport:

- The arms use different QoS reliability settings.
- In Image-message mode, the publisher builds and serializes a `sensor_msgs/msg/Image`. In data-sharing mode, it fills a fixed-size sample. Part of the publisher CPU difference comes from building the message, not from moving it.
- The subscriber's CPU time is mostly preprocessing, postprocessing, and ROS 2 work. Transport is a small part of it.

For these reasons, this Learning Path reports the two modes side by side and doesn't calculate a speedup.

In the data-sharing arm, the inference node also logs a `comm delay` value about once every 3 seconds. In these windows it averaged 0.26 to 0.30 ms, with a maximum of 0.37 ms. In an earlier set of runs, two single samples reached 67 ms and 100 ms, which is about two to three frame periods at 30 fps. Their cause wasn't found, and because the value is only printed every 3 seconds, short delays between samples aren't visible in the log.

## Troubleshooting

| Symptom | Cause and fix |
| --- | --- |
| A node exits with `NotEnoughMemoryException` | The Fast DDS profile didn't load, so the default history buffers are too small for large messages. Check `FASTRTPS_DEFAULT_PROFILES_FILE` and look for `XMLPARSER Error` in the log. |
| Subscriber receives nothing in data-sharing mode | Both nodes must use `/hbmem_img` (or the same topic), and the subscriber must have `is_shared_mem_sub:=1` |
| Subscriber receives nothing in Image-message mode | Set `ros_img_topic_name` on the subscriber to the publisher's `msg_pub_topic_name` |
| CPU numbers vary from run to run | Fix the CPU governor and keep other workloads off the RDK X5 |

## Restore the CPU governor

When you finish, restore the governor you recorded:

```bash
cat ~/lp1_governor.txt | sudo tee /sys/devices/system/cpu/cpufreq/policy0/scaling_governor
```

## What you have learned

You have:

- Ran the same X5 publisher and subscriber with two shared-memory parameter settings, which also selected different message types and QoS policies.
- Checked which conditions match, and found two that don't: message type and QoS reliability.
- Measured CPU, memory, and delivery delay at two rates, and reported the differences together with what limits their interpretation.

These measurements show how the two TogetheROS message configurations behave with the same local vision workload. On the next page, profile the inference node to find where its Arm CPU time goes and where image copies remain.
