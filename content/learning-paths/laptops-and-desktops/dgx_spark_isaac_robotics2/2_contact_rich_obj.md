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

```output
################################################################################
                           Learning iteration 399/400                            

                            Total steps: 78643200 
                       Steps per second: 62265 
                        Collection time: 2.945s 
                          Learning time: 0.213s 
                        Mean value loss: 0.0012
                    Mean surrogate loss: 0.0003
                      Mean entropy loss: -3.0812
                            Mean reward: 100.15
                    Mean episode length: 480.00
                        Mean action std: 0.17
      Episode_Reward/approach_ee_handle: 3.9564
         Episode_Reward/align_ee_handle: 0.1917
 Episode_Reward/approach_gripper_handle: 0.2473
Episode_Reward/align_grasp_around_handle: 0.1219
            Episode_Reward/grasp_handle: 0.0212
       Episode_Reward/open_drawer_bonus: 5.6417
 Episode_Reward/multi_stage_open_drawer: 2.3516
          Episode_Reward/action_rate_l2: -0.0091
               Episode_Reward/joint_vel: -0.0012
           Episode_Termination/time_out: 1.0000
--------------------------------------------------------------------------------
                         Iteration time: 3.16s
                           Time elapsed: 00:23:59
                                    ETA: 00:00:00

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
    --num_envs=1 \
    --checkpoint=logs/rsl_rl/franka_open_drawer/2026-05-11_12-03-08/model_50.pt
```

![img3 alt-text#center](./open_drawer.gif "Simulation of Open Drawer task using model at 50 iterations (left) and 399 iterations (right). Showing arm struggling to fully open the drawer.")


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

```

```output
fps step: 416 fps step and policy inference: 409 fps total: 337 epoch: 32/200 frames: 507904
fps step: 408 fps step and policy inference: 401 fps total: 332 epoch: 33/200 frames: 524288
saving next best rewards:  [300.05377]
=> saving checkpoint '/home/kieran/IsaacLab/logs/rl_games/Factory/test/nn/Factory.pth'
```

We can use the `play.py` script to view the peg insertion in Isaac sim. 

```bash
./isaaclab.sh -p scripts/reinforcement_learning/rl_games/play.py \
  --task=Isaac-Factory-PegInsert-Direct-v0 \
  --checkpoint=logs/rl_games/Factory/test/nn/Factory.pth \
  --num_envs=1 \
  --real-time \
  --seed=-1 \
  env.episode_length_s=4.0 \
  env.task.fixed_asset_init_pos_noise=[0.08,0.08,0.02] \
  env.task.hand_init_pos_noise=[0.03,0.03,0.02]
```

![img3 alt-text#center](./peg.gif "Simulation of sub-millimeter control of arm to insert peg into a hole. PPO model trained to 50 epochs")


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