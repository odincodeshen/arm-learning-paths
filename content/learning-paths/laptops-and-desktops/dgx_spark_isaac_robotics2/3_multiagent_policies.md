---
title: Train Multi-Agent Policies for Collaborative Tasks
weight: 4

### FIXED, DO NOT MODIFY
layout: learningpathall
---

## Multi-agent reinforcement learning

Some robotic tasks require multiple agents to cooperate. Isaac Lab supports multi-agent training through the [skrl](https://skrl.readthedocs.io/) library using IPPO (Independent PPO) and MAPPO (Multi-Agent PPO) algorithms.

### Shadow Hand Over: passing objects between hands

This task requires two Shadow Hands to coordinate: one hand holds an object and passes it to the other:

```bash
./isaaclab.sh -p scripts/reinforcement_learning/skrl/train.py \
    --task=Isaac-Shadow-Hand-Over-Direct-v0 \
    --headless \
    --algorithm MAPPO
```

The key differences from single-agent training are:

| **Aspect** | **Single-agent** | **Multi-agent (MAPPO)** |
|-----------|-----------------|------------------------|
| Policy | One policy controls the entire robot | Each agent has its own policy (or shares one) |
| Observations | Global observation vector | Each agent sees its own local observations |
| Actions | Single action vector | Each agent produces its own actions |
| Training | Standard PPO | MAPPO uses a centralized critic with decentralized actors |
| Algorithm flag | Not needed | `--algorithm MAPPO` or `--algorithm IPPO` |

### Cart-Double-Pendulum: multi-agent classic control

A simpler multi-agent environment for testing:

```bash
./isaaclab.sh -p scripts/reinforcement_learning/skrl/train.py \
    --task=Isaac-Cart-Double-Pendulum-Direct-v0 \
    --headless \
    --algorithm IPPO
```

In this task, two agents control different joints of a cart-double-pendulum system and must coordinate to balance both pendulums.

{{% notice Note %}}
Multi-agent training is currently supported only through the skrl library. If you run multi-agent environments with other libraries (rsl_rl, rl_games, sb3), they are automatically converted to single-agent environments.
{{% /notice %}}
