---
title: Prepare a public stereo-IMU dataset for VIMS
description: Download the TUM-VI room1 sequence on your workstation, convert it into a ROS 2 bag that the VIMS visual-inertial odometry node accepts, build a VIMS configuration from the dataset calibration, and copy both to the RDK X5.
weight: 4

layout: "learningpathall"
---

<!--
DRAFT (2026-10-30). Not reviewed.
Basis: evidence/run-20261024-02 (first conversion), run-20261024-03 (X5 reads the bag), run-20261026-01 (pages 4-5),
D-ROBOTICS_LP2_REPLY_v1.md (left image on top, mono8 for VIO, modified configuration may be shown with its changes,
GPL v3 parts handled per GPL v3).
The scripts on this page are shortened versions of scripts/tumvi_to_ros2bag.py and scripts/make_vims_tumvi_config.py.
2026-10-30 (evidence/run-20261030-01): every workstation command on this page was run as written on macOS
(Python 3.9.6; rosbags 0.9.23, numpy 2.0.2, opencv-python-headless 5.0.0.93, ruamel.yaml 0.19.1). The X5 was off,
so the scp of the VIMS template was replaced by the identical local copy taken from the X5 on 2026-10-24. The bag
(.db3 md5 cf667806...) and the three configuration files are identical to the ones used on pages 4 and 5.
Open items before publishing:
- 2026-10-31: the RDK X5 steps (template scp, copy, ros2 bag info) were run as written too; outputs match.
- GPL v3: the estimator_config.yaml template comes from VIMS (OpenVINS-based, GPL v3). Confirm how to present it once
  D-Robotics publishes the drobotics_vio source.
- Tested on macOS with Python 3.9.6; check on a Linux workstation.
-->

## Overview

The VIO node in VIMS needs synchronized stereo images, IMU data, and a calibration that describes the cameras and the IMU. On this page, you prepare all three from a public dataset, so you can run VIO on the RDK X5 without a camera:

1. Download the `room1` sequence of the TUM-VI dataset and its calibration on your workstation.
2. Convert the images and IMU data into a ROS 2 bag in the format that the VIMS VIO node expects.
3. Build a VIMS configuration from the dataset calibration.
4. Copy the bag and the configuration to the RDK X5, and check that ROS 2 on the RDK X5 can read the bag.

You run the conversion on your workstation, because it needs about 5 GB of disk space and a few Python packages that you don't need on the RDK X5.

## About the dataset

The [TUM visual-inertial dataset (TUM-VI)](https://cvg.cit.tum.de/data/datasets/visual-inertial-dataset) was recorded by the Technical University of Munich with a handheld stereo camera and an IMU. The cameras and the IMU are synchronized in hardware. The `room1` sequence is about 141 seconds long. It was recorded in a room with a motion-capture system, so it includes the true pose of the IMU for the whole sequence. You use these ground-truth poses on the next page to evaluate the VIO trajectory.

| Property | Value in `dataset-room1_512_16` |
| --- | --- |
| Cameras | 2 global-shutter cameras with fisheye lenses, 512x512 pixels, 16-bit gray, 20 frames per second |
| IMU | 3-axis accelerometer and gyroscope, 200 samples per second |
| Ground truth | Motion capture, 120 samples per second |
| License | [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) |

The dataset is described in D. Schubert, T. Goll, N. Demmel, V. Usenko, J. Stückler and D. Cremers, "The TUM VI Benchmark for Evaluating Visual-Inertial Odometry", IROS 2018.

These cameras aren't the cameras of the GS130WI. The results you get with this dataset describe the dataset, not the GS130WI.

## Download the sequence and the calibration

On your **workstation**, create a working folder and download the sequence in EuRoC format, its checksum, and the calibration files. The sequence is about 1.6 GB:

```bash
mkdir -p ~/tumvi && cd ~/tumvi
curl -fLO https://vision.in.tum.de/tumvi/exported/euroc/512_16/dataset-room1_512_16.tar
curl -fLO https://vision.in.tum.de/tumvi/exported/euroc/512_16/dataset-room1_512_16.tar.md5
curl -fLO https://cvg.cit.tum.de/_media/data/datasets/visual-inertial-dataset/pinhole-equi-512.zip
```

Check the download. On Linux, run:

```bash
md5sum -c dataset-room1_512_16.tar.md5
```

On macOS, compare the two values instead:

```bash
md5 -r dataset-room1_512_16.tar; cat dataset-room1_512_16.tar.md5
```

```output
51d2b8d93ee4ada47ec0b2d10115dcf8 dataset-room1_512_16.tar
51d2b8d93ee4ada47ec0b2d10115dcf8  dataset-room1_512_16.tar
```

Extract the sensor data and the calibration:

```bash
tar -xf dataset-room1_512_16.tar dataset-room1_512_16/mav0
unzip -q pinhole-equi-512.zip
ls dataset-room1_512_16/mav0 pinhole-equi-512
```

```output
dataset-room1_512_16/mav0:
cam0	cam1	imu0	mocap0

pinhole-equi-512:
camchain-imucam-imucalib.yaml	imucalib.log			report-imucam-imucalib.pdf
imu-imucalib.yaml		imucalib-command.txt		results-imucam-imucalib.txt
```

The `mav0` folder has one subfolder for each sensor. Each camera folder has the images as PNG files and a `data.csv` file with the time stamp of each image. The `imu0/data.csv` file has one line for each IMU sample, and `mocap0/data.csv` has the ground-truth poses. The `pinhole-equi-512` folder has the calibration that TUM computed with the [Kalibr](https://github.com/ethz-asl/kalibr) toolbox, for a pinhole camera model with equidistant (fisheye) distortion.

## Set up Python

The conversion uses [rosbags](https://gitlab.com/ternaris/rosbags), a pure Python library that writes ROS 2 bags without a ROS installation, together with NumPy and OpenCV.

On your **workstation**, create a virtual environment and install the packages:

```bash
python3 -m venv ~/tumvi/venv
~/tumvi/venv/bin/pip install -q "rosbags==0.9.23" numpy opencv-python-headless
```

The `rosbags` version is pinned because later versions change the API that the script uses. If `pip` warns that a newer version of `pip` is available, you can ignore the warning.

## Convert the sequence into a ROS 2 bag

The VIMS VIO node subscribes to two topics:

- `/sub_image_combine_raw`, a `sensor_msgs/msg/Image` that contains both camera images in one image, the left image on top of the right image. The VIO node accepts `mono8` gray images.
- An IMU topic of type `sensor_msgs/msg/Imu`. The configuration that you build in the next section sets it to `/drobotics_imu/bmi08x_imu`.

The script below reads the images and IMU samples and writes them to a bag in that form. It does three conversions:

- **Image depth.** The TUM-VI images have 16 bits for each pixel and a linear response. The script shifts each value right by 6 bits and clips it to 255. With 8 bits, most images would be very dark, because the sequence has bright lamps that use the top of the 16-bit range. The same factor applies to every image.
- **Image layout.** The script stacks the left image (`cam0`) on top of the right image (`cam1`) into one 512x1024 image. Both cameras of TUM-VI have the same time stamps, so the script pairs images by time stamp.
- **Bag format.** `rosbags` writes bag metadata version 8, and ROS 2 Humble on the RDK X5 reads versions up to 5. The script saves the metadata as version 5 and removes the fields that later versions added. It keeps the original file as `metadata.v8.yaml`.

Every message keeps the time stamp from the dataset, in nanoseconds.

On your **workstation**, save the script:

```bash
cat > ~/tumvi/tumvi_to_ros2bag.py <<'EOF'
#!/usr/bin/env python3
"""Convert a TUM-VI EuRoC-format sequence into a ROS 2 Humble bag for the VIMS VIO node."""
import argparse
import csv
import sys
from pathlib import Path

import cv2
import numpy as np
from rosbags.rosbag2 import Writer
from rosbags.typesys import Stores, get_typestore
from ruamel.yaml import YAML

IMAGE_TOPIC = '/sub_image_combine_raw'
IMU_TOPIC = '/drobotics_imu/bmi08x_imu'
SHIFT = 6  # 16-bit to 8-bit: value >> 6, clipped to 255


def read_csv(path):
    with open(path, newline='') as f:
        return [row for row in csv.reader(f) if row and not row[0].startswith('#')]


def main():
    ap = argparse.ArgumentParser()
    ap.add_argument('mav0', type=Path)
    ap.add_argument('out', type=Path)
    args = ap.parse_args()
    if args.out.exists():
        sys.exit(f'{args.out} already exists')

    cam0 = {int(r[0]): r[1] for r in read_csv(args.mav0 / 'cam0' / 'data.csv')}
    cam1 = {int(r[0]): r[1] for r in read_csv(args.mav0 / 'cam1' / 'data.csv')}
    stamps = sorted(set(cam0) & set(cam1))
    imu_rows = read_csv(args.mav0 / 'imu0' / 'data.csv')

    ts = get_typestore(Stores.ROS2_HUMBLE)
    Image, Imu = ts.types['sensor_msgs/msg/Image'], ts.types['sensor_msgs/msg/Imu']
    Header, Time = ts.types['std_msgs/msg/Header'], ts.types['builtin_interfaces/msg/Time']
    Quaternion, Vector3 = ts.types['geometry_msgs/msg/Quaternion'], ts.types['geometry_msgs/msg/Vector3']

    def header(ns, frame):
        return Header(stamp=Time(sec=ns // 1_000_000_000, nanosec=ns % 1_000_000_000), frame_id=frame)

    zero9 = np.zeros(9)
    no_orientation = np.zeros(9)
    no_orientation[0] = -1.0  # the IMU message has no orientation
    with Writer(args.out) as w:
        c_img = w.add_connection(IMAGE_TOPIC, Image.__msgtype__, typestore=ts)
        c_imu = w.add_connection(IMU_TOPIC, Imu.__msgtype__, typestore=ts)
        for r in imu_rows:  # timestamp [ns], gyro x y z [rad/s], accel x y z [m/s^2]
            ns = int(r[0])
            g, a = [float(x) for x in r[1:4]], [float(x) for x in r[4:7]]
            msg = Imu(header=header(ns, 'imu_link'),
                      orientation=Quaternion(x=0.0, y=0.0, z=0.0, w=1.0), orientation_covariance=no_orientation,
                      angular_velocity=Vector3(x=g[0], y=g[1], z=g[2]), angular_velocity_covariance=zero9,
                      linear_acceleration=Vector3(x=a[0], y=a[1], z=a[2]), linear_acceleration_covariance=zero9)
            w.write(c_imu, ns, ts.serialize_cdr(msg, Imu.__msgtype__))
        for ns in stamps:
            pair = [cv2.imread(str(args.mav0 / cam / 'data' / files[ns]), cv2.IMREAD_UNCHANGED)
                    for cam, files in (('cam0', cam0), ('cam1', cam1))]
            img = np.vstack([np.clip(p.astype(np.uint32) >> SHIFT, 0, 255).astype(np.uint8) for p in pair])
            msg = Image(header=header(ns, 'camera_link'), height=img.shape[0], width=img.shape[1],
                        encoding='mono8', is_bigendian=0, step=img.shape[1], data=img.reshape(-1))
            w.write(c_img, ns, ts.serialize_cdr(msg, Image.__msgtype__))

    # Save the metadata as version 5 for ROS 2 Humble, and keep the original.
    meta = args.out / 'metadata.yaml'
    (args.out / 'metadata.v8.yaml').write_text(meta.read_text())
    yaml = YAML()
    doc = yaml.load(meta.read_text())
    info = doc['rosbag2_bagfile_information']
    info['version'] = 5
    for key in ('custom_data', 'ros_distro'):
        info.pop(key, None)
    for t in info['topics_with_message_count']:
        t['topic_metadata'].pop('type_description_hash', None)
    with meta.open('w') as f:
        yaml.dump(doc, f)
    print(f'images: {len(stamps)} ({img.shape[1]}x{img.shape[0]} mono8), IMU samples: {len(imu_rows)}, '
          f'span: {(stamps[-1] - stamps[0]) / 1e9:.1f} s')


if __name__ == '__main__':
    main()
EOF
```

Run it. In the test for this Learning Path it took less than a minute and wrote about 1.4 GB:

```bash
cd ~/tumvi
venv/bin/python tumvi_to_ros2bag.py dataset-room1_512_16/mav0 room1_512_vertical
```

```output
images: 2821 (512x1024 mono8), IMU samples: 28122, span: 141.0 s
```

## Build the VIMS configuration

The VIO node reads its configuration from three files in the format of [OpenVINS](https://docs.openvins.com/), on which the VIMS VIO is based:

- `estimator_config.yaml`, the estimator settings. It names the other two files.
- A camera chain file with the intrinsics, distortion, and image size of each camera, and the transformation from the IMU to each camera.
- An IMU file with the noise parameters, the update rate, and the IMU topic.

You start from the estimator settings that VIMS ships for its SC132GS stereo camera, and change only what differs for TUM-VI. The camera chain and IMU files come from the TUM-VI calibration.

On your **workstation**, copy the VIMS estimator settings from the RDK X5:

```bash
scp sunrise@<x5-address>:/userdata/vims/install/share/drobotics_vio/config/sc132gs_stereo/estimator_config.yaml ~/tumvi/sc132gs_estimator_config.yaml
```

Save the script that builds the configuration:

```bash
cat > ~/tumvi/make_vims_config.py <<'EOF'
#!/usr/bin/env python3
"""Build a VIMS VIO configuration for TUM-VI 512 from the Kalibr calibration and the VIMS sc132gs settings."""
import re
import sys
from pathlib import Path

from ruamel.yaml import YAML

IMU_TOPIC = '/drobotics_imu/bmi08x_imu'
EDITS = {  # estimator_config.yaml key -> value for TUM-VI 512
    'verbosity': '"INFO"',
    'print_color': 'false',
    'use_local_imu': 'false',
    'gravity_mag': '9.81',
    'track_frequency': '20.0',
    'downsample_cameras': 'false',
    'downsample_cameras_already': 'false',
    'relative_config_imucam': '"tumvi_512-camchain-imucam.yaml"',
    'relative_config_imu': '"tumvi_512-imu.yaml"',
}


def rows(m):
    return '\n'.join('    - [' + ', '.join(repr(float(v)) for v in row) + ']' for row in m)


def main():
    calib, template, out = map(Path, sys.argv[1:4])
    yaml = YAML(typ='safe')
    cc = yaml.load((calib / 'camchain-imucam-imucalib.yaml').read_text())
    imu = yaml.load((calib / 'imu-imucalib.yaml').read_text())['imu0']
    out.mkdir(parents=True)

    lines = ['%YAML:1.0']
    for cam in ('cam0', 'cam1'):
        c = cc[cam]
        lines += [f'{cam}:', '  T_cam_imu:', rows(c['T_cam_imu'])]
        if 'T_cn_cnm1' in c:
            lines += ['  T_cn_cnm1:', rows(c['T_cn_cnm1'])]
        lines += [f'  cam_overlaps: {c["cam_overlaps"]}',
                  f'  camera_model: {c["camera_model"]}',
                  f'  distortion_coeffs: {[float(v) for v in c["distortion_coeffs"]]}',
                  f'  distortion_model: {c["distortion_model"]}',
                  f'  intrinsics: {[float(v) for v in c["intrinsics"]]}',
                  f'  resolution: {c["resolution"]}',
                  f'  rostopic: /{cam}/image_raw',
                  '  timeshift_cam_imu: 0.0']
    (out / 'tumvi_512-camchain-imucam.yaml').write_text('\n'.join(lines) + '\n')

    eye = '\n'.join('    - [' + ', '.join('1.0' if i == j else '0.0' for j in range(3)) + ']' for i in range(3))
    zero = '\n'.join('    - [0.0, 0.0, 0.0]' for _ in range(3))
    (out / 'tumvi_512-imu.yaml').write_text(f"""%YAML:1.0
imu0:
  T_i_b:
{rows(imu['T_i_b'])}
  accelerometer_noise_density: {imu['accelerometer_noise_density']}
  accelerometer_random_walk: {imu['accelerometer_random_walk']}
  gyroscope_noise_density: {imu['gyroscope_noise_density']}
  gyroscope_random_walk: {imu['gyroscope_random_walk']}
  rostopic: {IMU_TOPIC}
  time_offset: 0.0
  update_rate: {imu['update_rate']}
  model: "kalibr"
  Tw:
{eye}
  R_IMUtoGYRO:
{eye}
  Ta:
{eye}
  R_IMUtoACC:
{eye}
  Tg:
{zero}
""")

    est = template.read_text().splitlines()
    for i, line in enumerate(est):
        m = re.match(r'^([A-Za-z_]+):\s*(.*)$', line)
        if m and m.group(1) in EDITS:
            # No ':' in the comment: OpenCV YAML misreads unquoted booleans followed by such comments.
            est[i] = f'{m.group(1)}: {EDITS[m.group(1)]} # edited for TUM-VI 512, was {m.group(2).split("#")[0].strip()}'
            print(est[i])
    (out / 'estimator_config.yaml').write_text('\n'.join(est) + '\n')


if __name__ == '__main__':
    main()
EOF
```

Run it:

```bash
cd ~/tumvi
venv/bin/python make_vims_config.py pinhole-equi-512 sc132gs_estimator_config.yaml config/tumvi_512
```

The script prints each setting that it changed:

```output
verbosity: "INFO" # edited for TUM-VI 512, was "DEBUG"
print_color: false # edited for TUM-VI 512, was true
use_local_imu: false # edited for TUM-VI 512, was true
gravity_mag: 9.81 # edited for TUM-VI 512, was 9.79494
track_frequency: 20.0 # edited for TUM-VI 512, was 21.0
downsample_cameras: false # edited for TUM-VI 512, was true
downsample_cameras_already: false # edited for TUM-VI 512, was false
relative_config_imucam: "tumvi_512-camchain-imucam.yaml" # edited for TUM-VI 512, was "sc132gs_stereo-camchain-imucam.yaml"
relative_config_imu: "tumvi_512-imu.yaml" # edited for TUM-VI 512, was "sc132gs_stereo-imu.yaml"
```

The changes do the following:

- `use_local_imu: false` makes the node subscribe to an IMU topic instead of reading the IMU device of the RDK X5.
- `gravity_mag` and `track_frequency` match the location of the dataset and its camera rate. With the combined stereo image in this Learning Path, `track_frequency` doesn't limit the frame rate, as the tuning page shows.
- `downsample_cameras: false` keeps the 512x512 images at full size. The VIMS settings halve the images of the SC132GS cameras, which are larger.
- The last two lines point to the new camera chain and IMU files.

All other settings, for example the number of features and the feature tracking parameters, stay as D-Robotics set them for its own cameras. The camera chain file copies the TUM-VI calibration: the intrinsics, the four equidistant distortion coefficients, the 512x512 image size, the transformation from the IMU to each camera, and a camera-to-IMU time offset of zero, because TUM-VI time stamps are already aligned. The IMU file uses the noise values from the TUM-VI calibration.

{{% notice Note %}}
`estimator_config.yaml` is a modified copy of a VIMS configuration file. The VIMS VIO is based on OpenVINS, which is licensed under the GNU General Public License v3. Keep the license terms in mind if you share the modified file.
{{% /notice %}}

## Copy the bag and the configuration to the RDK X5

On your **workstation**, copy the bag and the configuration to the home folder of the `sunrise` user on the RDK X5. The bag is about 1.4 GB:

```bash
ssh sunrise@<x5-address> mkdir -p lp2-tumvi/config
scp -r ~/tumvi/room1_512_vertical sunrise@<x5-address>:lp2-tumvi/
scp -r ~/tumvi/config/tumvi_512 sunrise@<x5-address>:lp2-tumvi/config/
```

On the **RDK X5**, check that ROS 2 Humble can read the bag:

```bash
source /opt/tros/humble/local_setup.bash
ros2 bag info ~/lp2-tumvi/room1_512_vertical
```

```output
Files:             room1_512_vertical.db3
Bag size:          1.4 GiB
Storage id:        sqlite3
Duration:          141.050428000s
Start:             Mar  9 2018 01:31:48.181901469 (1520530308.181901469)
End:               Mar  9 2018 01:34:09.232329469 (1520530449.232329469)
Messages:          30943
Topic information: Topic: /sub_image_combine_raw | Type: sensor_msgs/msg/Image | Count: 2821 | Serialization Format: cdr
                   Topic: /drobotics_imu/bmi08x_imu | Type: sensor_msgs/msg/Imu | Count: 28122 | Serialization Format: cdr
```

The bag has the 2,821 stereo images and 28,122 IMU samples of the sequence, with the time stamps from 2018.

## Troubleshooting

| Symptom | Check |
| --- | --- |
| `pip` installs a different `rosbags` version, or the script fails with an import or API error | Install `rosbags==0.9.23` as shown. |
| The script stops with `already exists` | Delete or rename the output folder, then run the script again. |
| `ros2 bag info` on the RDK X5 can't read the metadata | Check that `metadata.yaml` in the bag folder says `version: 5`. |
| The images look almost black in a viewer | Check the bit shift. A shift of 8 bits makes most `room1` images very dark. |

## What you have learned

You have:

- Downloaded a public stereo-IMU dataset with ground truth, and checked the download.
- Converted its images and IMU samples into a ROS 2 bag with the combined image and topics that the VIMS VIO node expects.
- Built a VIMS configuration from the dataset calibration, and listed each setting that you changed.
- Copied the bag and the configuration to the RDK X5.

On the next page, you run the VIMS VIO node with this bag.
