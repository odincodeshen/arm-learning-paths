---
title: Autonomous obstacle avoidance with Python
weight: 6

### FIXED, DO NOT MODIFY
layout: learningpathall
---

Now that you have a robot streaming LiDAR data over ROS 2, you can write a simple Python script that reads the point cloud and drives the robot around the warehouse on its own, no map, no Nav2 stack, just ~50 lines of reactive obstacle avoidance.

{{% notice Note %}}
This is intentionally a **minimal, dummy example**. The script uses three hard-coded rules to dodge obstacles, there is no path planning, no SLAM, and no AI involved. The goal is simply to prove that you can close the loop between Isaac Sim sensors and ROS 2 commands, and see a robot move on its own. In a future Learning Path you will replace these simple rules with a proper AI-based navigation stack.
{{% /notice %}}

## How the algorithm works

The idea is dead simple, three rules and nothing else:

1. **Read** the 3D LiDAR point cloud and keep a 1-second rolling buffer (the LiDAR delivers partial sweeps, so buffering gives you a full 360° picture).
2. **Split** the front 180° into 9 sectors and find the minimum distance in each one.
3. **Decide**:
   - Front clear (nothing within 1.5 m dead ahead) → drive forward.
   - Front blocked but a side sector is open → turn toward the most open sector.
   - Everything blocked → back up and spin.

That's it. Three rules.

## Create the script

Create a file called `explore.py`. The walkthrough below explains each piece before showing the full listing.

### Imports and constants

```python
#!/usr/bin/env python3
"""Obstacle-avoidance explorer for Nova Carter in Isaac Sim.

Reads the 3D LiDAR point cloud, finds the most open direction,
and steers toward it. Run with: python3 explore.py
"""

import math, struct, time
import rclpy
from rclpy.node import Node
from rclpy.qos import QoSProfile, ReliabilityPolicy
from sensor_msgs.msg import PointCloud2
from geometry_msgs.msg import Twist

SPEED = 1.0        # m/s, forward speed when path is clear
TURN = 1.5         # rad/s, rotation speed when turning
STOP_DIST = 1.5    # m, obstacle closer than this = blocked
NUM_SECTORS = 9    # split the front 180° into this many slices
```

`PointCloud2` is the standard ROS 2 message for 3D point clouds. `Twist` is the message used to command velocity (linear + angular) to the robot. The four constants at the top are the only knobs you need to touch, more on that later.

### Node setup

```python
class Explorer(Node):
    def __init__(self):
        super().__init__('explorer')
        self.pub = self.create_publisher(Twist, '/cmd_vel', 10)
        qos = QoSProfile(depth=1, reliability=ReliabilityPolicy.BEST_EFFORT)
        self.create_subscription(
            PointCloud2, '/front_3d_lidar/lidar_points', self.cloud_cb, qos)
        self.points = []
        self.create_timer(0.1, self.drive)
        self.get_logger().info('Explorer started, Ctrl+C to stop')
```

The node does three things: publish velocity commands on `/cmd_vel`, subscribe to the LiDAR point cloud, and run the `drive` method 10 times per second. The `BEST_EFFORT` QoS matches what Isaac Sim publishes. If you use the default (`RELIABLE`), the subscription silently receives nothing.

### Reading the LiDAR

```python
    def cloud_cb(self, msg):
        """Parse point cloud and keep a 1-second rolling buffer."""
        now = time.monotonic()
        pts = []
        for i in range(0, msg.width * msg.point_step, msg.point_step):
            x, y, z = struct.unpack_from('fff', msg.data, i)
            if math.isnan(x) or z < -2.0 or z > 1.5:
                continue
            dist = math.sqrt(x * x + y * y)
            deg = math.degrees(math.atan2(y, x))
            if dist > 0.2 and -90 <= deg <= 90:
                pts.append((deg, dist, now))
        self.points = [p for p in self.points + pts if now - p[2] < 1.0]
```

Each `PointCloud2` message from the LiDAR is a packed binary blob. The loop unpacks every point as three floats (`x, y, z`), skips anything that is NaN or too high/low on the Z axis (floor reflections, ceiling), and converts the remaining points to polar coordinates (angle in degrees, distance in meters). Points farther than 90° to either side of the robot are discarded since you only care about what is in front.

The 1-second rolling buffer (`now - p[2] < 1.0`) exists because the LiDAR delivers partial sweeps per message. Buffering several messages gives you a full 360° picture.

### Deciding where to go

```python
    def drive(self):
        """Find the most open sector and drive toward it."""
        sectors = [30.0] * NUM_SECTORS
        for deg, dist, _ in self.points:
            i = min(int((deg + 90) / 180 * NUM_SECTORS), NUM_SECTORS - 1)
            sectors[i] = min(sectors[i], dist)

        front = min(
            (d for deg, d, _ in self.points if -15 <= deg <= 15), default=30.0)
        best = max(range(NUM_SECTORS), key=lambda i: sectors[i])
        center = NUM_SECTORS // 2

        cmd = Twist()
        if front > STOP_DIST:
            cmd.linear.x = SPEED
        elif sectors[best] > STOP_DIST:
            cmd.angular.z = TURN if best > center else -TURN
        else:
            cmd.linear.x = -0.5
            cmd.angular.z = TURN

        self.pub.publish(cmd)
```

This is where the three rules from the diagram above live. The front 180° is divided into 9 equal sectors and the closest obstacle in each sector is recorded. Then:

- **Front clear** (nothing within 1.5 m in the center ±15°) → drive forward at `SPEED`.
- **Front blocked, but a side sector is open** → rotate toward the sector with the most room.
- **Everything blocked** → back up slowly and spin until something opens up.

### Entry point

```python
rclpy.init()
try:
    rclpy.spin(Explorer())
except KeyboardInterrupt:
    pass
finally:
    rclpy.shutdown()
```

The `try/finally` block ensures ROS 2 shuts down cleanly when you press Ctrl+C.

### Full listing

For convenience, here is the complete `explore.py` you can copy-paste:

```python
#!/usr/bin/env python3
"""Obstacle-avoidance explorer for Nova Carter in Isaac Sim.

Reads the 3D LiDAR point cloud, finds the most open direction,
and steers toward it. Run with: python3 explore.py
"""

import math, struct, time
import rclpy
from rclpy.node import Node
from rclpy.qos import QoSProfile, ReliabilityPolicy
from sensor_msgs.msg import PointCloud2
from geometry_msgs.msg import Twist

SPEED = 1.0        # m/s, forward speed when path is clear
TURN = 1.5         # rad/s, rotation speed when turning
STOP_DIST = 1.5    # m, obstacle closer than this = blocked
NUM_SECTORS = 9    # split the front 180° into this many slices


class Explorer(Node):
    def __init__(self):
        super().__init__('explorer')
        self.pub = self.create_publisher(Twist, '/cmd_vel', 10)
        qos = QoSProfile(depth=1, reliability=ReliabilityPolicy.BEST_EFFORT)
        self.create_subscription(
            PointCloud2, '/front_3d_lidar/lidar_points', self.cloud_cb, qos)
        self.points = []
        self.create_timer(0.1, self.drive)
        self.get_logger().info('Explorer started, Ctrl+C to stop')

    def cloud_cb(self, msg):
        """Parse point cloud and keep a 1-second rolling buffer."""
        now = time.monotonic()
        pts = []
        for i in range(0, msg.width * msg.point_step, msg.point_step):
            x, y, z = struct.unpack_from('fff', msg.data, i)
            if math.isnan(x) or z < -2.0 or z > 1.5:
                continue
            dist = math.sqrt(x * x + y * y)
            deg = math.degrees(math.atan2(y, x))
            if dist > 0.2 and -90 <= deg <= 90:
                pts.append((deg, dist, now))
        self.points = [p for p in self.points + pts if now - p[2] < 1.0]

    def drive(self):
        """Find the most open sector and drive toward it."""
        sectors = [30.0] * NUM_SECTORS
        for deg, dist, _ in self.points:
            i = min(int((deg + 90) / 180 * NUM_SECTORS), NUM_SECTORS - 1)
            sectors[i] = min(sectors[i], dist)

        front = min(
            (d for deg, d, _ in self.points if -15 <= deg <= 15), default=30.0)
        best = max(range(NUM_SECTORS), key=lambda i: sectors[i])
        center = NUM_SECTORS // 2

        cmd = Twist()
        if front > STOP_DIST:
            cmd.linear.x = SPEED
        elif sectors[best] > STOP_DIST:
            cmd.angular.z = TURN if best > center else -TURN
        else:
            cmd.linear.x = -0.5
            cmd.angular.z = TURN

        self.pub.publish(cmd)


rclpy.init()
try:
    rclpy.spin(Explorer())
except KeyboardInterrupt:
    pass
finally:
    rclpy.shutdown()
```

## Run the explorer

Make sure Isaac Sim is running with the Nova Carter scene loaded and playing (from the previous step).

In a new terminal:

```bash
source /opt/ros/jazzy/setup.bash
python3 explore.py
```

The robot will start moving immediately. It drives forward through the warehouse corridors and turns when it detects an obstacle ahead.

![Nova Carter autonomous navigation in Isaac Sim#center](carter.gif "The Nova Carter robot navigating autonomously through the warehouse using the reactive obstacle-avoidance script.")

## Watch it in RViz2

If you still have RViz2 open from the previous section, you are all set, just watch the robot move on its own. If not, open a new terminal and launch it:

```bash
source /opt/ros/jazzy/setup.bash
rviz2
```

Then configure the display (same steps as in the teleoperation section):

1. In the **Displays** panel (left), expand **Global Options** and change **Fixed Frame** from `map` to `odom` (`odom` is the only fixed reference frame in this scene).
2. Click the **Add** button at the bottom of the Displays panel → **By topic** tab → expand `/front_3d_lidar` → `/lidar_points` → select **PointCloud2** → **OK**. Then expand the new PointCloud2 entry in the panel and set **Size (m)** to `0.05`.

<!-- IMAGE_PLACEHOLDER: Screenshot of RViz2 showing the Nova Carter's LiDAR point cloud (white/colored dots forming the warehouse walls and shelves) while the robot is autonomously moving. The point cloud should clearly show the corridor structure around the robot. -->

You can see the LiDAR point cloud updating in real time as the robot explores on its own. The corridor walls and shelves appear as dense clusters of dots around the robot as it navigates through the warehouse.

## Tweak it

The script has four constants at the top that you can play with:

| Constant | Default | What it does |
|----------|---------|-------------|
| `SPEED` | `1.0` | Forward speed in m/s. Crank it up for a faster robot. |
| `TURN` | `1.5` | Rotation speed in rad/s. Higher = sharper turns. |
| `STOP_DIST` | `1.5` | How close an obstacle needs to be (in meters) before the robot reacts. |
| `NUM_SECTORS` | `9` | More sectors = finer-grained obstacle detection. |

Try setting `SPEED = 5.0` for a more dramatic demo, or `STOP_DIST = 0.5` if you like living dangerously.
