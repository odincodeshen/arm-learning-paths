---
title: Toward Collective Intelligence and Multi-Agent Training
weight: 4

### FIXED, DO NOT MODIFY
layout: learningpathall
---

## From working alone to working together

In the previous section, you used an Arm-based Isaac Sim / Isaac Lab environment to run single-arm manipulation and contact-rich tasks. This section continues on the same development platform and moves into **multi-agent reinforcement learning (MARL)**, where multiple agents learn to cooperate inside the same simulation.

In real logistics centers, automated production lines, and dual-arm robotics systems, a single agent is often not enough. A task may require two hands to transfer an object, multiple controllers to stabilize a system, or several agents to coordinate under partial observation. The challenge is no longer only about controlling one robot correctly. It is about **coordination, role allocation, and shared task success**.

In this section, you will use the **skrl** library to explore multi-agent reinforcement learning (MARL). You'll work with **MAPPO** (Multi-Agent Proximal Policy Optimization) and **IPPO** (Independent PPO). MAPPO trains agents with a shared critic that uses global information during training, but agents execute independently from local observations during deployment. IPPO lets each agent learn independently, treating other agents as part of the environment. Both use the same PPO algorithm updates but differ in how they share information during training.

As in the earlier section, this section also highlights how Arm-based systems enable **workflow control**. You can use Python scripts, task flags, and algorithm options to control multi-agent training flows, switch configurations, and continue running GPU-backed simulation on the same platform.

## Task 1: Shadow Hand Over — coordinated transfer between hands

In this task, the policy must solve a classic cooperation scenario. One Shadow Hand holds an object and transfers it to the other hand. Each agent controls only its own motion but must learn to anticipate the other agent's behavior and timing from partial local observations. MAPPO is well suited for this task because the shared critic can encourage coordinated behavior during training, even though each hand uses only local observations during execution.

### Run

You'll now use the **skrl** library for multi-agent training. Pass the `--algorithm` flag to select MAPPO for this task:

```bash
./isaaclab.sh -p scripts/reinforcement_learning/skrl/train.py \
    --task=Isaac-Shadow-Hand-Over-Direct-v0 \
    --headless \
    --algorithm MAPPO
```

This command loads the task, selects the MAPPO training algorithm, and runs the simulation headless. Like earlier tasks, the Python entry point controls task and algorithm selection, letting you switch workflows without any recompilation.

### Verify

After training, look for the following behaviors:

* The two hands coordinate rather than moving independently.
* The hand holding the object adjusts its pose to create a feasible transfer path.
* Drops, collisions, and action conflicts decrease as training progresses.

To view the trained policy, replace the checkpoint path with your trained model directory and run:

```bash
./isaaclab.sh -p scripts/reinforcement_learning/skrl/play.py \
    --task=Isaac-Shadow-Hand-Over-Direct-v0 \
    --num_envs=1 \
    --algorithm=MAPPO \
    --real-time \
    --checkpoint=<path_to_your_best_agent.pt>
```

![Shadow Hand Over training progress showing two dexterous hands coordinating an object transfer. The left panel shows early training (iteration 3600) where motion is uncoordinated and the object is still held. The right panel shows the policy at convergence using the best_agent.pt checkpoint identified by skrl, where the hands smoothly coordinate the handover.#center](./multi_agent_hand.gif "Shadow Hand Over training progression. Left: iteration 3600. Right: best_agent.pt.")


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
    --algorithm IPPO \
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
./isaaclab.sh -p scripts/reinforcement_learning/skrl/play.py \
    --task=Isaac-Cart-Double-Pendulum-Direct-v0 \
    --num_envs=16 \
    --use_pretrained_checkpoint \
    --algorithms=IPPO
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