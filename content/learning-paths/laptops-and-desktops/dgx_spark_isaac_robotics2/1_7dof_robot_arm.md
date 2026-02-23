---
title: Manipulate Objects with a 7-DOF Robot Arm
weight: 2

### FIXED, DO NOT MODIFY
layout: learningpathall
---

## Scale reinforcement learning beyond single-task locomotion

In the previous section you trained a single locomotion policy using RSL-RL. Isaac Lab supports a much broader range of tasks and training paradigms. In this final section you will explore manipulation tasks, multi-agent environments, different RL libraries, and advanced training configurations that leverage the full capabilities of DGX Spark.

## Manipulation tasks

Beyond locomotion, Isaac Lab provides environments for robotic manipulation where arms and hands must grasp, lift, and place objects. These tasks use different robots and control modes.

### Reach task: move the end-effector to a target

The simplest manipulation task is reaching a target position. Train a Franka robot to reach:

```bash
cd ~/IsaacLab
export LD_PRELOAD="$LD_PRELOAD:/lib/aarch64-linux-gnu/libgomp.so.1"
./isaaclab.sh -p scripts/reinforcement_learning/rsl_rl/train.py \
    --task=Isaac-Reach-Franka-v0 \
    --headless \
    --num_envs=2048
```

In this task the Franka 7-DOF arm must move its end-effector to a randomly sampled target pose. The observation space includes joint positions, joint velocities, and the target position. The action space is joint position targets.

Evaluate the trained policy:

```bash
./isaaclab.sh -p scripts/reinforcement_learning/rsl_rl/play.py \
    --task=Isaac-Reach-Franka-Play-v0 \
    --num_envs=16
```

### Lift task: pick up a cube

A more challenging task combines reaching with grasping:

```bash
./isaaclab.sh -p scripts/reinforcement_learning/rsl_rl/train.py \
    --task=Isaac-Lift-Cube-Franka-v0 \
    --headless \
    --num_envs=2048
```

The robot must approach a cube on a table, close its gripper to grasp it, and lift it to a target height. This task requires learning a sequence of skills: approach, align, grasp, and lift.


## Optional: Compare Other Locomotion Robots

Isaac Lab provides locomotion environments for many different robots. Try training other robots to compare performance:

### Flat terrain tasks (easier, faster to train)

```bash
# Unitree H1 on flat terrain
./isaaclab.sh -p scripts/reinforcement_learning/rsl_rl/train.py \
    --task=Isaac-Velocity-Flat-H1-v0 --headless

# Unitree Go2 quadruped on flat terrain
./isaaclab.sh -p scripts/reinforcement_learning/rsl_rl/train.py \
    --task=Isaac-Velocity-Flat-Unitree-Go2-v0 --headless

# ANYmal C quadruped on flat terrain
./isaaclab.sh -p scripts/reinforcement_learning/rsl_rl/train.py \
    --task=Isaac-Velocity-Flat-Anymal-C-v0 --headless
```

### Rough terrain tasks (harder, require more iterations)

```bash
# Unitree G1 humanoid on rough terrain
./isaaclab.sh -p scripts/reinforcement_learning/rsl_rl/train.py \
    --task=Isaac-Velocity-Rough-G1-v0 --headless

# ANYmal D quadruped on rough terrain
./isaaclab.sh -p scripts/reinforcement_learning/rsl_rl/train.py \
    --task=Isaac-Velocity-Rough-Anymal-D-v0 --headless

# Boston Dynamics Spot on flat terrain
./isaaclab.sh -p scripts/reinforcement_learning/rsl_rl/train.py \
    --task=Isaac-Velocity-Flat-Spot-v0 --headless
```

The following table compares some of the available locomotion environments:

| **Environment** | **Robot** | **Type** | **Terrain** | **Difficulty** |
|----------------|----------|----------|------------|----------------|
| `Isaac-Velocity-Flat-H1-v0` | Unitree H1 | Bipedal humanoid | Flat | Medium |
| `Isaac-Velocity-Rough-H1-v0` | Unitree H1 | Bipedal humanoid | Rough | Hard |
| `Isaac-Velocity-Flat-G1-v0` | Unitree G1 | Bipedal humanoid | Flat | Medium |
| `Isaac-Velocity-Rough-G1-v0` | Unitree G1 | Bipedal humanoid | Rough | Hard |
| `Isaac-Velocity-Flat-Unitree-Go2-v0` | Unitree Go2 | Quadruped | Flat | Easy |
| `Isaac-Velocity-Rough-Unitree-Go2-v0` | Unitree Go2 | Quadruped | Rough | Medium |
| `Isaac-Velocity-Flat-Anymal-C-v0` | ANYmal C | Quadruped | Flat | Easy |
| `Isaac-Velocity-Rough-Anymal-C-v0` | ANYmal C | Quadruped | Rough | Medium |
| `Isaac-Velocity-Flat-Spot-v0` | Boston Dynamics Spot | Quadruped | Flat | Easy |
| `Isaac-Velocity-Flat-Digit-v0` | Agility Digit | Bipedal humanoid | Flat | Hard |

{{% notice Tip %}}
Quadruped robots (Go2, ANYmal, Spot) are generally easier and faster to train than bipedal humanoids (H1, G1, Digit) because four-legged gaits are inherently more stable.
{{% /notice %}}