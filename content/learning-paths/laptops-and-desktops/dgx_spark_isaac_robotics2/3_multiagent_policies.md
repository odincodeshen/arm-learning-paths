---
title: Toward Collective Intelligence and Multi-Agent Training
weight: 4

### FIXED, DO NOT MODIFY
layout: learningpathall
---

## From working alone to working together

In the previous section, you used an Arm-based Isaac Sim / Isaac Lab environment to run single-arm manipulation and contact-rich tasks. This section continues on the same development platform and moves into **multi-agent reinforcement learning (MARL)**, where multiple agents learn to cooperate inside the same simulation.

In real logistics centers, automated production lines, and dual-arm robotics systems, a single agent is often not enough. A task may require two hands to transfer an object, multiple controllers to stabilize a system, or several agents to coordinate under partial observation. The challenge is no longer only about controlling one robot correctly. It is about **coordination, role allocation, and shared task success**.

In this section, you will use the **skrl** library to explore how **MAPPO** and **IPPO** work in Isaac Lab, and how centralized training with decentralized execution helps agents learn cooperative behavior.

As in the earlier section, this section also highlights the value of Arm-based systems in **workflow control**. Developers can use Python scripts, task flags, and algorithm options to control multi-agent training flows, switch configurations, and continue running GPU-backed simulation on the same platform.

## Learning objectives

After completing this section, you will be able to:

* Understand the main differences between single-agent and multi-agent training.
* Use **MAPPO** and **IPPO** for cooperative tasks in Isaac Lab.
* Run the **Shadow Hand Over** task and understand how two dexterous hands coordinate object transfer.
* Observe multi-agent interaction in a simpler control system.
* Understand how scripts and algorithm flags support multi-agent simulation workflows on an Arm-based system.


## Task 1: Shadow Hand Over — coordinated transfer between hands

This is a classic cooperation task. One Shadow Hand holds an object and must transfer it accurately to the other hand. Each agent must not only control its own motion, but also anticipate the timing and behavior of the other agent from local observations.

### Scenario goal

Train two agents to complete a coordinated object handover, where each Shadow Hand must act correctly based on partial information.

### Run

In this task, you use **MAPPO**, a common multi-agent policy optimization method designed for cooperative environments:

```bash
./isaaclab.sh -p scripts/reinforcement_learning/skrl/train.py \
    --task=Isaac-Shadow-Hand-Over-Direct-v0 \
    --headless \
    --algorithm MAPPO
```

### What this script controls

From a workflow-control perspective, this command does more than start training. Inside the same Isaac Lab environment, it controls:

* which multi-agent task is loaded
* which training entry point is used
* which MARL algorithm is selected
* runtime behavior such as headless execution

This kind of Python-script and command-line control reflects the practical role of the Arm CPU in this tutorial. The CPU handles tooling, task selection, and workflow orchestration, while the GPU handles the heavy simulation workload.

### Why MAPPO matters

The key idea behind MAPPO is **centralized training with decentralized execution**:

* **During training**, the critic can use more global information to help agents learn coordinated behavior.
* **During execution**, each agent still makes decisions from its own local observations.

This makes MAPPO especially useful for dual-hand manipulation, team coordination, and other tasks where agents must act independently while still contributing to a shared goal.

### Verify

After training, look for the following behaviors:

* The two hands no longer move independently without coordination.
* The hand holding the object adjusts its pose to create a feasible transfer path.
* Drops, collisions, and action conflicts decrease as training progresses.


```bash


### Not working
./isaaclab.sh -p scripts/reinforcement_learning/skrl/play.py \
    --task=Isaac-Shadow-Hand-Over-Direct-v0
    --num_envs=16


```

![img5 alt-text#center](demo_5.gif "Figure 5: Shadow-Hand-Over")


## Task 2: Cart-Double-Pendulum — observing cooperation in a simpler system

To study multi-agent interaction more directly, you can also use a simpler control system. In this task, two agents control different parts of a cart-double-pendulum system and must work together to keep the system balanced.

### Scenario goal

Use a simplified system to observe multi-agent interaction and build intuition for cooperative control and role separation.

### Run

This example uses **IPPO** (Independent PPO):

```bash
./isaaclab.sh -p scripts/reinforcement_learning/skrl/train.py \
    --task=Isaac-Cart-Double-Pendulum-Direct-v0 \
    --headless \
    --algorithm IPPO
```

### What changes here

Compared with the previous task, you are changing not only the task itself but also the way learning is structured:

* **MAPPO** emphasizes shared information during centralized training
* **IPPO** treats the other agents as part of the environment and lets each agent learn independently

This lets you compare different MARL behaviors on the same platform and within the same workflow, simply by changing the algorithm flag.

### Verify

When you run this task, observe the following:

* The system becomes more stable over time instead of diverging quickly.
* The two agents develop complementary behavior rather than interfering with each other.
* Convergence speed and stability differ between algorithms.

```bash

### Not working

./isaaclab.sh -p scripts/reinforcement_learning/skrl/play.py \
    --task=Isaac-Cart-Double-Pendulum-Direct-v0
    --num_envs=16
```

![img6 alt-text#center](demo_6.gif "Figure 6: Cart-Double-Pendulum")


### Why this task is useful

Compared with Shadow Hand Over, Cart-Double-Pendulum is structurally simpler. That makes it a good environment for building intuition about:

* how roles are divided across agents
* how local decisions affect global system behavior
* how algorithm design changes cooperative dynamics

## Core comparison: single-agent vs multi-agent training

When you move from single-agent tasks to multi-agent training, the change is not just about adding more controllers. The problem definition itself becomes different.

| Feature | Single-agent | Multi-agent (MAPPO / IPPO) |
| --- | --- | --- |
| Policy | One policy controls the whole robot | Each agent has its own policy, or partially shared policies |
| Observations | Often one observation vector | Each agent receives its own local observations |
| Actions | One action vector | Each agent outputs its own actions |
| Training paradigm | Standard PPO or other single-agent RL | Centralized training with decentralized execution, or independent learning |
| Algorithm flag | Usually not required | Selected with `--algorithm MAPPO` or `--algorithm IPPO` |

{{% notice Note %}}
In Isaac Lab, multi-agent training is currently driven mainly by the **skrl** library. If you try to run a multi-agent environment with another library such as **rsl_rl**, the task may fall back to a single-agent mode or lack full multi-agent support.
{{% /notice %}}

## Wrap-up

This section showed the main shift from single-agent control to multi-agent cooperation in Isaac Lab. Instead of focusing only on whether one robot moves correctly, you now have to think about how multiple agents form a coordinated strategy under partial information.

On an Arm-based system, the key value in this section is not raw CPU performance. It is the ability of the **Arm CPU to control the overall simulation workflow**. Through Python scripts, task options, and algorithm flags, you can switch multi-agent scenarios quickly and continue developing, training, and comparing workflows on the same platform.

## Next up

Your robots can now perform precise actions and even cooperate, but the resulting motion may still look rigid. For humanoid robots that must coexist with people, moving in a more natural way is another important challenge.

In the next section, you will explore **AMP (Adversarial Motion Priors)** and learn how robots can use reference motion data to produce more natural and fluent behavior.