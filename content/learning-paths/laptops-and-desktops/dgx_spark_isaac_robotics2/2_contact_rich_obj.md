---
title: Set up Isaac Sim and Isaac Lab on DGX Spark
weight: 3

### FIXED, DO NOT MODIFY
layout: learningpathall
---

### Open-Drawer task: articulated object interaction

Train a Franka robot to open a cabinet drawer:

```bash
./isaaclab.sh -p scripts/reinforcement_learning/rsl_rl/train.py \
    --task=Isaac-Open-Drawer-Franka-v0 \
    --headless \
    --num_envs=2048
```

This task adds the complexity of interacting with articulated objects. The robot must grasp the drawer handle and pull it open. The environment includes contact forces between the gripper and the handle.

### Comparison of manipulation tasks

| **Environment** | **Robot** | **Task** | **Difficulty** | **Key challenge** |
|----------------|----------|----------|---------------|-------------------|
| `Isaac-Reach-Franka-v0` | Franka | Reach a target pose | Easy | Inverse kinematics through RL |
| `Isaac-Reach-UR10-v0` | UR10 | Reach a target pose | Easy | Different robot kinematics |
| `Isaac-Lift-Cube-Franka-v0` | Franka | Pick up and lift a cube | Medium | Coordinating reach + grasp + lift |
| `Isaac-Open-Drawer-Franka-v0` | Franka | Open a cabinet drawer | Medium | Contact-rich manipulation |
| `Isaac-Stack-Cube-Franka-v0` | Franka | Stack three cubes | Hard | Long-horizon sequential skills |
| `Isaac-Repose-Cube-Allegro-v0` | Allegro Hand | In-hand cube reorientation | Hard | Dexterous multi-finger control |
| `Isaac-Repose-Cube-Shadow-Direct-v0` | Shadow Hand | In-hand cube reorientation | Very hard | 24-DOF dexterous manipulation |

## Contact-rich manipulation with Factory environments

Isaac Lab includes Factory environments for high-precision assembly tasks. These environments simulate contact forces with high fidelity:

```bash
# Peg insertion
./isaaclab.sh -p scripts/reinforcement_learning/rl_games/train.py \
    --task=Isaac-Factory-PegInsert-Direct-v0 \
    --headless

# Gear meshing
./isaaclab.sh -p scripts/reinforcement_learning/rl_games/train.py \
    --task=Isaac-Factory-GearMesh-Direct-v0 \
    --headless

# Nut threading
./isaaclab.sh -p scripts/reinforcement_learning/rl_games/train.py \
    --task=Isaac-Factory-NutThread-Direct-v0 \
    --headless
```

{{% notice Note %}}
Factory tasks use the `rl_games` library rather than `rsl_rl`. Isaac Lab provides separate training scripts for each supported RL library under `scripts/reinforcement_learning/`.
{{% /notice %}}

The following table shows the Factory tasks and their precision requirements:

| **Environment** | **Task** | **Precision required** |
|----------------|----------|----------------------|
| `Isaac-Factory-PegInsert-Direct-v0` | Insert a peg into a socket | Sub-millimeter alignment |
| `Isaac-Factory-GearMesh-Direct-v0` | Mesh a gear with other gears | Rotational and translational alignment |
| `Isaac-Factory-NutThread-Direct-v0` | Thread a nut onto a bolt | Precise torque and position control |
