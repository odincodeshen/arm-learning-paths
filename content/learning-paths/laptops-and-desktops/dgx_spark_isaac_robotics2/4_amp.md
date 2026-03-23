---
title: Reproducing Natural Motion with Adversarial Motion Priors (AMP)
weight: 5

### FIXED, DO NOT MODIFY
layout: learningpathall
---


## From completing tasks to moving naturally

In the previous section, you used an Arm-based Isaac Sim / Isaac Lab environment to run manipulation, contact-rich interaction, and multi-agent training tasks. This section continues on the same platform and introduces **Adversarial Motion Priors (AMP)**, a workflow that helps reinforcement learning policies produce motion that looks more natural and human-like.

Traditional reinforcement learning can teach a robot to walk, run, or satisfy control objectives, but the resulting motion is often effective rather than natural. For robots that must coexist with people, interact in human environments, or demonstrate expressive behavior, that is usually not enough. Isaac Lab therefore supports **AMP**, which uses reference **motion-capture data** to guide policy learning toward smoother and more realistic movement.

In this section, you will use the **skrl** library with the `--algorithm AMP` flag to run humanoid walking, running, and dancing tasks.

As in the previous section, the Arm value in this workflow is mainly about **workflow control**. Developers can use Python scripts, task flags, and algorithm options to switch tasks, control training flow, and iterate on experiments, while the GPU continues to support the heavy simulation and training workload.

## Learning objectives

After completing this section, you will be able to:

* Understand the basic idea of AMP and the role of the **discriminator** in training.
* Use **skrl** to launch AMP training in Isaac Lab.
* Run humanoid walking, running, and dancing tasks.
* Compare traditional RL with motion-prior-guided training in terms of motion quality.
* Understand how scripts and algorithm flags support natural-motion simulation workflows on an Arm-based system.


## The core idea behind AMP: guiding policy learning with reference motion

The key idea of AMP is to make **natural motion** part of the training signal.

In a standard RL setup, the agent usually learns only from task-related rewards such as forward velocity, balance, or avoiding falls. A learned policy may therefore complete the task without producing realistic movement. AMP adds a **discriminator** so that the agent is encouraged not only to solve the task, but also to make its state transitions look similar to the reference motion data.

### AMP workflow at a glance

At a high level, AMP training includes three components:

* **Reference motion data** that provides natural walking, running, or dancing priors
* A **discriminator** that learns to distinguish reference motion from agent-generated motion
* A **policy network** that tries to solve the task while also producing motion that can fool the discriminator

This design reduces the need to handcraft complex style-related reward functions while still making motion quality part of the optimization target.

### Why AMP matters

AMP is especially useful for:

* humanoid locomotion that needs more natural gait patterns
* character control tasks that emphasize continuity and rhythm
* imitation-guided RL workflows that want less reward-engineering complexity


## Task 1: Humanoid Walk — learning a natural gait

### Scenario goal

Use human walking reference data to train a humanoid robot to produce stable and natural walking behavior.

### Run

Use the **skrl** library together with `--algorithm AMP` to launch training:

```bash
./isaaclab.sh -p scripts/reinforcement_learning/skrl/train.py \
    --task=Isaac-Humanoid-AMP-Walk-Direct-v0 \
    --headless \
    --algorithm AMP
```

### What this script controls

From a workflow-control perspective, this command does more than start a training job. Within the same development environment, it controls:

* which AMP task is loaded
* which RL training entry point is used
* which algorithm mode is enabled (`AMP`)
* runtime behavior such as headless execution

This type of Python-script and command-line control reflects the practical role of the Arm CPU in this tutorial. The CPU manages tools, task switching, and workflow orchestration, while the GPU handles simulation and training at scale.

### Verify

After training, look for the following behaviors:

* The humanoid moves forward stably instead of losing balance frequently.
* The gait shows smoother center-of-mass transfer instead of stiff hopping-like motion.
* The left and right leg timing resembles a more natural walking pattern.

```bash
./isaaclab.sh -p scripts/reinforcement_learning/skrl/play.py \
    --task=Isaac-Humanoid-AMP-Walk-Direct-v0
    --num_envs=16
```

![img7 alt-text#center](demo_7.gif "Figure 7: Humanoid AMP Walk")


## Task 2: Humanoid Run — adding speed and coordination

If walking is mainly about stability and rhythm, running introduces a higher level of dynamic coordination. The robot must generate propulsion in a shorter contact window, keep the body balanced, and avoid losing control as motion amplitude increases.

### Scenario goal

Use human running reference data to train a humanoid robot to maintain a natural and controllable running pattern at higher speed.

### Run

```bash
./isaaclab.sh -p scripts/reinforcement_learning/skrl/train.py \
    --task=Isaac-Humanoid-AMP-Run-Direct-v0 \
    --headless \
    --algorithm AMP
```

### What changes in the workflow

Compared with the walking task, you do not need to change platforms or rebuild the project. You only switch the task to enter another motion-prior-driven scenario. This is a clear example of workflow control: the same Python entry point, toolchain, and development environment can be reused while exploring different natural-motion tasks.

### Verify

After training, confirm the following:

* As forward speed increases, the robot remains stable rather than falling immediately.
* Arm swing, leg lift, and landing timing become more coordinated.
* The motion looks like a recognizable running pattern rather than just aggressive forward movement.

```bash
./isaaclab.sh -p scripts/reinforcement_learning/skrl/play.py \
    --task=Isaac-Humanoid-AMP-Run-Direct-v0
    --num_envs=16
```

![img8 alt-text#center](demo_8.gif "Figure 8: Humanoid AMP Run")


## Task 3: Humanoid Dance — testing expressiveness and style control

Dance tasks often make the benefits of AMP easier to see than walking or running. The goal is not only stability or speed. The motion must also show temporal rhythm, recognizable poses, and style characteristics inherited from the reference data.

### Scenario goal

Use human dance reference data to train a humanoid robot to produce rhythmic and expressive continuous motion.

### Run

```bash
./isaaclab.sh -p scripts/reinforcement_learning/skrl/train.py \
    --task=Isaac-Humanoid-AMP-Dance-Direct-v0 \
    --headless \
    --algorithm AMP
```

### Verify

In this kind of task, observe the following:

* Motion unfolds as a continuous sequence rather than isolated single-step actions.
* Body poses and center-of-mass shifts show visible rhythm.
* The policy maintains expressive behavior over time instead of quickly collapsing into unstable motion.

```bash
./isaaclab.sh -p scripts/reinforcement_learning/skrl/play.py \
    --task=Isaac-Humanoid-AMP-Dance-Direct-v0
    --num_envs=16
```

![img8 alt-text#center](demo_9.gif "Figure 9: Humanoid AMP Dance")

### Why this task is interesting

Dance tasks are a strong example of the difference between AMP and standard RL. They show that there can be another optimization target beyond simple task completion: **motion quality**. For humanoids that interact with people, this is not only a visual improvement. It can also affect balance, timing, and the overall predictability of behavior.


## AMP task overview

| Task | Reference motion data | Expected outcome |
|---|---|---|
| Isaac-Humanoid-AMP-Walk-Direct-v0 | Human walking capture data | Natural and stable walking gait |
| Isaac-Humanoid-AMP-Run-Direct-v0 | Human running capture data | Smoother high-speed running behavior |
| Isaac-Humanoid-AMP-Dance-Direct-v0 | Human dance capture data | Rhythmic and expressive dance motion |

{{% notice Tip %}}
For humanoid robots that must coexist with people, the value of AMP is not only that the motion looks more human. It can also improve center-of-mass transfer and dynamic stability, which may improve behavior quality in more complex environments.
{{% /notice %}}


## Wrap-up

In this section, you completed an introduction to natural-motion training in Isaac Lab:

* Learned how AMP uses motion priors and a discriminator to guide policy learning
* Used `skrl` and `--algorithm AMP` to run walking, running, and dancing tasks
* Switched between natural-motion scenarios through Python training scripts and command-line execution
* Understood that, in this workflow, the Arm CPU mainly serves as the control plane for development and iteration

## Next up

You have now worked through the main workflows in this series, from basic manipulation to high-precision assembly, multi-agent cooperation, and natural-motion imitation.

In the next section, you will summarize the full tutorial and compare the main RL libraries supported by Isaac Lab.
