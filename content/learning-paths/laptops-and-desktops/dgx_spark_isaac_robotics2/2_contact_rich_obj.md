---
title: Fine Manipulation and Contact-Rich Interaction
weight: 3

### FIXED, DO NOT MODIFY
layout: learningpathall
---

## From grasping to operating: understanding mechanical constraints

In the previous section, you trained the Franka arm on the basic Reach and Lift tasks. This section continues the same Arm-based Isaac Sim / Isaac Lab workflow and moves into **contact-rich manipulation**: interacting with objects that include mechanical constraints, contact forces, and high precision requirements.

In real industrial environments, a robot does more than pick up free objects. Drawers move along rails, pegs must be inserted into tight sockets, and nuts must align with bolts before threading can begin. These tasks require a policy to understand **contact**, **constrained motion**, and **failure modes caused by small errors**.

This section starts with the **Open-Drawer** task to introduce interaction with articulated objects, and then moves into Isaac Lab's **Factory** environments, where you explore higher-precision industrial assembly workflows.

As in the previous section, this section also highlights the role of Arm-based systems in **workflow control**. Developers can use Python scripts and command-line tools to switch tasks, choose training entry points, and iterate on experiment flows, while the GPU continues to handle the high-load simulation work.

## Learning objectives

After completing this section, you will be able to:

* Work with **articulated objects** in manipulation tasks.
* Understand why **contact forces** matter in physically realistic robot interaction.
* Run high-precision industrial assembly tasks in Isaac Lab's **Factory** environments.
* Compare the technical challenges across different manipulation tasks.
* Understand how scripts and task switching support contact-rich simulation workflows on an Arm-based system.

## Task 1: Open-Drawer — interacting with articulated objects

In this task, the Franka arm must grasp a handle and pull a drawer open along its rail. Unlike the Lift task from the previous section, the object here is not a freely moving rigid body. It is an articulated object with mechanical structure and constrained motion.

### Scenario goal

Train the robotic arm to approach the drawer handle, establish stable contact, and pull the drawer open in the correct direction.

### Run

```bash
./isaaclab.sh -p scripts/reinforcement_learning/rsl_rl/train.py \
    --task=Isaac-Open-Drawer-Franka-v0 \
    --headless \
    --num_envs=2048
```

### What this script controls

From a workflow perspective, this command does more than launch training. It switches the development environment to a different simulation scenario while keeping the same platform and tooling. It controls:

* which manipulation task is loaded
* which RL training entry point is used
* runtime behavior such as headless execution and the number of environments

This kind of script-level control is especially useful in advanced robotics tutorials, because you can move quickly between tasks and evaluate how a policy behaves in different contact-rich scenarios without rebuilding the full setup.

### Why this task is harder

The main challenge in Open-Drawer is that the policy must handle several things at once:

* stable contact between the gripper and the handle
* the constrained motion imposed by the drawer rail
* friction, collision, and pose errors during pulling

This makes the task more than simply moving an end-effector to a target point. After contact is established, the robot must continue interacting correctly with the object throughout the motion.

### Verify

After training, confirm the following:

* The robotic arm approaches and aligns with the handle instead of stopping in front of the drawer.
* Once contact is established, the drawer moves along the rail direction.
* The opening motion remains stable rather than failing through slipping, shaking, or force in the wrong direction.

```bash
./isaaclab.sh -p scripts/reinforcement_learning/rsl_rl/play.py \
    --task=Isaac-Open-Drawer-Franka-v0 \
    --num_envs=16
```

![img3 alt-text#center](demo_3.gif "Figure 3: Open-Drawer")


## Task 2: Factory environments — moving toward sub-millimeter precision

To support industrial automation scenarios, Isaac Lab provides the **Factory** family of environments. These tasks emphasize high-fidelity contact simulation and are designed for insertion, threading, and other precision assembly actions.

### Scenario goal

Explore industrial assembly tasks that require high-precision contact control, and understand how they differ from general manipulation tasks.

### Run

Note that Factory tasks often use **rl_games** instead of **rsl_rl**. This means you are not only switching tasks, but also switching the training workflow entry point.

```bash
# Peg insertion: insert a peg into a socket with sub-millimeter alignment
./isaaclab.sh -p scripts/reinforcement_learning/rl_games/train.py \
    --task=Isaac-Factory-PegInsert-Direct-v0 \
    --headless

# Nut threading: thread a nut onto a bolt with precise pose and torque control
./isaaclab.sh -p scripts/reinforcement_learning/rl_games/train.py \
    --task=Isaac-Factory-NutThread-Direct-v0 \
    --headless
```

### What changes in the workflow

This section is a good example of workflow control because you make two changes at once:

* you switch from a general manipulation task to a Factory task
* you switch from the `rsl_rl` training script to the `rl_games` training script

This shows the CPU-side role clearly. On an Arm-based development system, the CPU handles tool execution, task switching, and experiment control. Developers can keep the same platform and environment, then move into a more demanding simulation workflow through script and command-line changes.

### Why these tasks matter

Factory tasks are common in industrial automation and assembly scenarios. They are challenging because:

* alignment tolerances are very small
* physical feedback after contact is highly sensitive
* position, orientation, and force control become tightly coupled

For example, peg insertion requires stable alignment before insertion, while nut threading adds even more demanding pose control and rotational behavior. These tasks are usually much more sensitive to small errors than Reach, Lift, or drawer interaction.

### Verify

When running these tasks, verify the following:

* The training script correctly switches to the `rl_games` workflow.
* The Factory environment loads successfully.
* The simulation shows high-precision alignment and contact behavior rather than immediate failure at first contact.
* The same platform can continue switching between tasks and training workflows without reconfiguring the whole environment.


{{% notice Note %}}
For Factory tasks, high-fidelity contact-force simulation is essential. Whether the agent can respond to sub-millimeter physical feedback directly affects the success rate of insertion, threading, and assembly tasks.
{{% /notice %}}



## Comparing manipulation task depth

As tasks evolve from simple reaching to precision assembly, the technical demands increase significantly.

| Environment | Task | Difficulty | Key challenge |
|---|---|---|---|
| Isaac-Reach-Franka-v0 | Reach a target pose | Easy | Learn basic inverse control through RL |
| Isaac-Open-Drawer-Franka-v0 | Open a drawer | Medium | Contact-rich manipulation with mechanical constraints |
| Isaac-Factory-NutThread-Direct-v0 | Thread a nut onto a bolt | Hard | Precise torque and pose control |
| Isaac-Stack-Cube-Franka-v0 | Stack three cubes | Hard | Chaining a long sequence of manipulation skills |

This comparison also helps show that not all manipulation tasks are simple object relocation problems. Once a workflow includes articulated object interaction and industrial assembly, the importance of contact stability, precision, and experiment control rises quickly.


## Wrap-up

In this section, you pushed manipulation training beyond basic grasping and into more realistic contact-rich scenarios:

* Used **Open-Drawer** to understand articulated-object interaction
* Explored **Factory** tasks for high-precision industrial assembly workflows
* Switched tasks and training entry points to understand the role of Python scripts in simulation workflow control
* Reinforced that, in this tutorial, the Arm CPU mainly adds value through development workflow control and rapid iteration, rather than by directly driving GPU simulation performance

## Next up

A single robotic arm can already complete more precise interactions, but more complex automation scenarios often require multiple agents working together.

In the next section, you will move beyond single-robot operation and explore how multiple robotic agents can cooperate to complete a task.