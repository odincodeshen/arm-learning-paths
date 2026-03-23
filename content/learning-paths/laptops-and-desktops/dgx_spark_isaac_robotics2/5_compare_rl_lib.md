---
title: Choosing RL Libraries and Summarizing the Environment Landscape
weight: 6

### FIXED, DO NOT MODIFY
layout: learningpathall
---

## From task runner to workflow architect

In the previous sections, you used an Arm-based Isaac Sim / Isaac Lab environment to work through manipulation, contact-rich interaction, multi-agent training, and AMP-based natural motion. In this final section, the focus is no longer only on whether a task can run. Instead, the goal is to ask a more important architectural question: 

**Given different task characteristics, which RL library should you choose, and how should you manage and scale the simulation workflow on one development platform?**

This section therefore takes a more architectural view of the full series. You will review the major RL libraries supported by Isaac Lab, understand what kinds of tasks they fit best, and think about when to prioritize fast iteration, modularity, or standardized interfaces.

As in the previous sections, the Arm value in this workflow is mainly about **workflow control**. In a GPU-backed simulation pipeline, the Arm CPU does not need to be framed as the primary source of training performance. Its value is in running scripts, selecting training entry points, managing process orchestration, and controlling experiment flow across the toolchain.

## Learning objectives

After completing this section, you will be able to:

* Compare the main RL libraries commonly used with Isaac Lab.
* Choose a more suitable training stack based on task characteristics.
* Understand the role of multi-GPU distributed training in the Isaac Lab workflow.
* Summarize the capabilities developed across the previous sections from a workflow-architect perspective.
* Understand how the Arm CPU contributes as a control plane for simulation development and iteration.


## Choosing your technical toolkit

One of Isaac Lab's main strengths is that it is not tied to a single RL framework. Instead, it provides an open and extensible ecosystem, allowing you to choose different RL libraries and training entry points depending on the problem you want to solve.

For an architect, this choice matters. Different tasks impose different requirements on training speed, observation complexity, modularity, extensibility, and debugging convenience. Library selection is not just a matter of preference. It affects the maintainability of the whole simulation workflow and the efficiency of experimentation.

### Scenario goal

Choose an RL library for an Isaac Lab workflow based on task type and development goals.


## Library comparison and decision guidance

The following table summarizes four commonly used RL libraries in Isaac Lab and the scenarios they fit best.

| Library | Core strength | Best fit |
|---|---|---|
| **RSL-RL** | Lightweight, fast, memory-efficient | Locomotion, fast iteration, large parallel training |
| **rl_games** | Supports LSTM and visual encoders | Complex observation spaces, contact-rich manipulation, Factory tasks |
| **skrl** | Modular design with MARL and AMP support | Multi-agent training, natural-motion imitation, flexible workflow extension |
| **Stable Baselines3** | Strong documentation and standardized API | Teaching, prototyping, baseline comparison |

### How to think about the choice

You can make a first-pass decision like this:

* If you care most about training speed and throughput, start with **RSL-RL**
* If you need more complex observation handling or recurrent policies, consider **rl_games**
* If you need multi-agent workflows or AMP-style training, **skrl** is often the most natural choice
* If you want fast baselines, educational examples, or standardized comparisons, **Stable Baselines3** is usually the easiest place to start

### Why this matters in practice

In Isaac Lab, choosing an RL library often means switching to a different Python training script, a different configuration style, or a different experiment structure. That is part of workflow control. On an Arm-based system, developers can switch between training stacks using script entry points and command-line flags without rebuilding the entire development environment.


## Mapping libraries to task types

To make the decision more concrete, the following table maps the task categories from the earlier sections to common library choices.

| Task type | Suggested library | Why |
|---|---|---|
| Franka Reach / Lift and other basic manipulation tasks | **RSL-RL** | Simple setup and good for fast baselines with large parallel rollout |
| Drawer / Factory and other contact-rich tasks | **rl_games** | Better suited for more complex observation and policy structures |
| Multi-agent object handover | **skrl** | A more natural fit for MARL workflows |
| Humanoid AMP Walk / Run / Dance | **skrl** | Direct fit for AMP-style algorithms and natural-motion tasks |
| Educational examples and standard RL baselines | **Stable Baselines3** | Standardized API and easy comparison workflow |

{{% notice Tip %}}
No single library is the best choice for every task. A practical strategy is to start with the tool that helps you establish a baseline quickly, then move to a more specialized training stack when the task requires it.
{{% /notice %}}


## Scaling up: multi-GPU distributed training

When task scale grows from hundreds of environments to thousands or even tens of thousands of parallel instances, a single GPU may no longer be enough. In that case, you can consider multi-GPU distributed training to reduce time-to-result and increase experiment throughput.

### Scenario goal

Launch Isaac Lab training with a PyTorch distributed workflow on a multi-GPU system.

### Run

The following example uses a single node with two GPUs:

```bash
python -m torch.distributed.run --nnodes=1 --nproc_per_node=2 \
    scripts/reinforcement_learning/rsl_rl/train.py \
    --task=Isaac-Velocity-Rough-H1-v0 \
    --headless \
    --distributed
```

### What this script controls

This command does more than say “use two GPUs.” From a workflow-control perspective, it controls:

* which training entry point is used
* how many local worker processes are launched
* whether distributed mode is enabled
* which task is executed
* runtime behavior such as headless execution

This is also one of the clearest examples of Arm value in the tutorial. The CPU side handles script launch, process orchestration, tool control, and experiment coordination, while the GPUs handle large-scale simulation and training.

### Verify

After launching the workflow, confirm the following:

* Multiple worker processes start successfully without rank initialization errors.
* Training logs show that distributed mode is enabled.
* GPU utilization rises during training.
* Compared with a single-GPU run, the workflow shows the potential for higher experiment throughput.


{{% notice Note %}}
Although multi-GPU training can accelerate larger workloads, a single high-performance GPU is often already sufficient for many locomotion and manipulation tasks. Whether distributed training is worth enabling depends on the scale of the task and the goals of the experiment.
{{% /notice %}}


## Exploring the environment catalog

After completing the previous section, you now understand several kinds of simulation scenarios. The next step is often not to redesign the whole system immediately, but to inspect what Isaac Lab already provides and decide which environments are worth extending.

### Run

Use the following command to list built-in environments:

```bash
./isaaclab.sh -p scripts/environments/list_envs.py
```

### Verify

After running it, you should see a large collection of built-in tasks. From there, you can check:

* whether there is already a manipulation or locomotion task close to your target use case
* which environments belong to direct tasks, manager-based tasks, or specific library workflows
* whether an existing task can serve as the starting point for your own robot or scene

### Why it matters on Arm

This kind of exploration again highlights the practical value of an Arm-based development system. The CPU side manages scripts and tooling, allowing you to inspect, switch, and extend simulation workflows quickly without leaving the same development platform.


## Full-series summary: the capability map you built

You have now completed a full progression from basic manipulation to architecture-level workflow decisions. Across this series, you developed the following capabilities:

* **Basic manipulation**: trained 7-DOF arm tasks such as Reach and Lift to build manipulation fundamentals
* **Precision interaction**: handled articulated objects with mechanical constraints and explored high-precision Factory tasks
* **Cooperative behavior**: used MARL workflows to study object handover and coordination across multiple agents
* **Natural motion**: used AMP to move from task completion toward more natural and expressive behavior
* **Architectural decision-making**: learned how to choose RL libraries based on task requirements and when to consider multi-GPU distributed training

More importantly, you are no longer only running isolated examples. You are starting to think in terms of **workflow design**: how to switch tasks, change training stacks, manage experiments, verify results, and continue iterating on a GPU-backed robotics workflow from the same Arm-based platform.

## Wrap-up

In this chapter, you completed the final piece of the tutorial:

* Compared the core strengths of common RL libraries used with Isaac Lab
* Mapped different libraries to different task categories
* Launched multi-GPU training through a PyTorch distributed workflow
* Explored the environment catalog as a starting point for extension
* Understood the Arm CPU as the control plane for simulation workflow management

## Next steps

You now have the foundation to run, switch, extend, and manage multiple simulation scenarios in an Arm-based Isaac Sim / Isaac Lab environment.

From here, there are two natural directions to continue. You can customize the existing tasks for your own robot models and scenes, or you can package these workflows into a more reproducible project structure with Docker, version pinning, and automated validation to improve portability and maintainability.
