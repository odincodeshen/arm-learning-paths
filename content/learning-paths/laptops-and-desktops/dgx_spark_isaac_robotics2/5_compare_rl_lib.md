---
title: Compare RL Libraries for Robotic Training Workflows
weight: 6

### FIXED, DO NOT MODIFY
layout: learningpathall
---


## Using different RL libraries

Isaac Lab supports multiple RL libraries. Each library has its own training script and configuration format. The following shows how to train the same task with different libraries:

```bash
# RSL-RL (PPO) - lightweight, fast for locomotion
./isaaclab.sh -p scripts/reinforcement_learning/rsl_rl/train.py \
    --task=Isaac-Velocity-Rough-H1-v0 --headless

# rl_games (PPO) - feature-rich, supports LSTM and vision
./isaaclab.sh -p scripts/reinforcement_learning/rl_games/train.py \
    --task=Isaac-Velocity-Rough-Anymal-C-v0 --headless

# skrl (PPO) - modular, supports multi-agent and AMP
./isaaclab.sh -p scripts/reinforcement_learning/skrl/train.py \
    --task=Isaac-Velocity-Rough-H1-v0 --headless

# Stable Baselines3 (PPO) - well-documented, Gymnasium compatible
./isaaclab.sh -p scripts/reinforcement_learning/sb3/train.py \
    --task=Isaac-Velocity-Flat-Unitree-A1-v0 --headless
```

### Library comparison

| **Library** | **Strengths** | **Best for** |
|------------|---------------|-------------|
| RSL-RL | Fast, minimal, efficient PPO | Locomotion tasks, quick experiments |
| rl_games | LSTM, vision encoders, Factory tasks | Complex observation spaces, contact-rich tasks |
| skrl | IPPO, MAPPO, AMP, modular design | Multi-agent tasks, imitation learning |
| Stable Baselines3 | Extensive documentation, Gymnasium API | Learning, prototyping, benchmarking |

## Multi-GPU training

For large-scale experiments, Isaac Lab supports multi-GPU training. DGX Spark has a single GPU, but if you connect two DGX Spark systems together (supported via NVLink) or use a multi-GPU workstation, you can distribute training:

```bash
# Multi-GPU training with PyTorch distributed
python -m torch.distributed.run --nnodes=1 --nproc_per_node=2 \
    scripts/reinforcement_learning/rsl_rl/train.py \
    --task=Isaac-Velocity-Rough-H1-v0 \
    --headless \
    --distributed
```

{{% notice Note %}}
Multi-GPU training requires additional setup. Refer to the [Isaac Lab Multi-GPU Training Guide](https://isaac-sim.github.io/IsaacLab/main/source/features/multi_gpu.html) for details. On a single DGX Spark, the Blackwell GPU is powerful enough for most locomotion and manipulation training tasks.
{{% /notice %}}

## Summary of all environment categories

The table below summarizes all the environment categories available in Isaac Lab, with example commands you can try on your DGX Spark:

| **Category** | **Example environment** | **Training command** |
|-------------|------------------------|---------------------|
| Classic control | `Isaac-Cartpole-Direct-v0` | `./isaaclab.sh -p scripts/reinforcement_learning/rsl_rl/train.py --task=Isaac-Cartpole-Direct-v0 --headless` |
| Locomotion (flat) | `Isaac-Velocity-Flat-Unitree-Go2-v0` | `./isaaclab.sh -p scripts/reinforcement_learning/rsl_rl/train.py --task=Isaac-Velocity-Flat-Unitree-Go2-v0 --headless` |
| Locomotion (rough) | `Isaac-Velocity-Rough-H1-v0` | `./isaaclab.sh -p scripts/reinforcement_learning/rsl_rl/train.py --task=Isaac-Velocity-Rough-H1-v0 --headless` |
| Manipulation (reach) | `Isaac-Reach-Franka-v0` | `./isaaclab.sh -p scripts/reinforcement_learning/rsl_rl/train.py --task=Isaac-Reach-Franka-v0 --headless` |
| Manipulation (lift) | `Isaac-Lift-Cube-Franka-v0` | `./isaaclab.sh -p scripts/reinforcement_learning/rsl_rl/train.py --task=Isaac-Lift-Cube-Franka-v0 --headless` |
| Contact-rich | `Isaac-Factory-PegInsert-Direct-v0` | `./isaaclab.sh -p scripts/reinforcement_learning/rl_games/train.py --task=Isaac-Factory-PegInsert-Direct-v0 --headless` |
| Multi-agent | `Isaac-Shadow-Hand-Over-Direct-v0` | `./isaaclab.sh -p scripts/reinforcement_learning/skrl/train.py --task=Isaac-Shadow-Hand-Over-Direct-v0 --headless --algorithm MAPPO` |
| AMP (motion priors) | `Isaac-Humanoid-AMP-Walk-Direct-v0` | `./isaaclab.sh -p scripts/reinforcement_learning/skrl/train.py --task=Isaac-Humanoid-AMP-Walk-Direct-v0 --headless --algorithm AMP` |
| Navigation | `Isaac-Navigation-Flat-Anymal-C-v0` | `./isaaclab.sh -p scripts/reinforcement_learning/rsl_rl/train.py --task=Isaac-Navigation-Flat-Anymal-C-v0 --headless` |
| Dexterous manipulation | `Isaac-Repose-Cube-Allegro-v0` | `./isaaclab.sh -p scripts/reinforcement_learning/rsl_rl/train.py --task=Isaac-Repose-Cube-Allegro-v0 --headless` |

For the full list of over 100 environments, run:

```bash
./isaaclab.sh -p scripts/environments/list_envs.py
```

## What you have accomplished

You have completed this Learning Path on building robotic simulation and reinforcement learning workflows with Isaac Sim and Isaac Lab on DGX Spark.

Throughout this Learning Path you have learned how to:

- Describe the roles of Isaac Sim (physics simulation) and Isaac Lab (RL framework) and how the DGX Spark Grace-Blackwell architecture accelerates robotics workflows
- Build Isaac Sim and Isaac Lab from source on an Arm-based DGX Spark system, producing aarch64-optimized binaries
- Run basic robot simulations in Isaac Sim, including spawning robots and stepping physics
- Train a reinforcement learning policy for the Unitree H1 humanoid robot using PPO via RSL-RL, with a detailed understanding of every hyperparameter
- Explore manipulation tasks (reach, lift, open-drawer, stack), contact-rich assembly (Factory), and dexterous hand manipulation (Allegro, Shadow)
- Run multi-agent training with MAPPO and IPPO using the skrl library
- Use Adversarial Motion Priors (AMP) for natural humanoid locomotion
- Compare multiple RL libraries (RSL-RL, rl_games, skrl, Stable Baselines3) and understand when to use each one

The DGX Spark platform provides a complete environment for developing, training, and evaluating robotic control policies. The unified memory architecture and Blackwell GPU acceleration make it possible to run thousands of parallel simulation environments on a single desktop system, dramatically reducing the time from idea to trained policy.

For additional learning, see the resources in the Further Reading section. Continue experimenting with different environments, tuning hyperparameters, and exploring the Isaac Lab codebase to build increasingly sophisticated robotic AI systems.
