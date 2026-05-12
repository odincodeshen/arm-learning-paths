---
title: Choosing RL Libraries
weight: 6

### FIXED, DO NOT MODIFY
layout: learningpathall
---

## From task runner to workflow architect

In the previous sections, you used an Arm-based Isaac Sim / Isaac Lab environment to work through manipulation, contact-rich interaction, multi-agent training, and AMP-based natural motion. In this final section, the focus is no longer only on whether a task can run. Instead, the goal is to ask a more important architectural question: 

**Given different task characteristics, which RL library should you choose, and how should you manage and scale the simulation workflow on one development platform?**


## Choosing your technical toolkit

One of Isaac Lab's main strengths is that it is not tied to a single RL framework. Instead, it provides an open and extensible ecosystem, allowing you to choose different RL libraries and training entry points depending on the problem you want to solve.

For an architect, this choice is mostly about tradeoffs. Different tasks impose different requirements for throughput, observation complexity, algorithm features, debugging convenience, and extensibility. Library selection is therefore not only personal preference. It directly affects how fast you can iterate and how well the workflow scales for your specific use case.

### Scenario goal

Choose an RL library for an Isaac Lab workflow based on task type and development goals.


## Library tradeoffs and decision guidance

The following table summarizes four commonly used RL libraries in Isaac Lab, with links to their repositories and the task profiles they fit best.

| Library | Core strength | Best fit |
|---|---|---|
| [**RSL-RL**](https://github.com/leggedrobotics/rsl_rl) | Lightweight, fast, memory-efficient | Locomotion, fast iteration, large parallel training |
| [**rl_games**](https://github.com/Denys88/rl_games) | Supports LSTM and visual encoders | Complex observation spaces, contact-rich manipulation, Factory tasks |
| [**skrl**](https://github.com/Toni-SM/skrl) | Modular design with MARL and AMP support | Multi-agent training, natural-motion imitation, flexible workflow extension |
| [**Stable Baselines3**](https://github.com/DLR-RM/stable-baselines3) | Strong documentation and standardized API | Teaching, prototyping, baseline comparison |

### How to think about the choice

You can make a first-pass decision by matching task requirements to algorithm and library behavior:

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

### Run

`torch.distributed.run` is PyTorch's distributed launcher. It starts one training process per GPU and coordinates communication across ranks for synchronous distributed training. The following example uses one node with two GPUs:

If your setup includes multiple networked systems, the same pattern extends to clusters, for example Grace Hopper servers or DGX Spark systems connected over a high-speed network. In those cases, `--nnodes` and rank settings are expanded to span the full cluster.

```bash
python -m torch.distributed.run --nnodes=1 --nproc_per_node=2 \
    scripts/reinforcement_learning/rsl_rl/train.py \
    --task=<example task> \
    --headless \
    --distributed
```

This command defines the training entry point, worker-process count, distributed mode, and task selection in one place. In this workflow, the CPU side handles launch and orchestration while GPUs handle simulation and learning throughput.

### Verify

After launching the workflow, confirm the following:

* Multiple worker processes start successfully without rank initialization errors.
* Training logs show that distributed mode is enabled.
* GPU utilization rises during training.
* Compared with a single-GPU run, the workflow shows the potential for higher experiment throughput.


{{% notice Note %}}
Although multi-GPU training can accelerate larger workloads, a single high-performance GPU is often already sufficient for many locomotion and manipulation tasks. Whether distributed training is worth enabling depends on the scale of the task and the goals of the experiment.
{{% /notice %}}


### Verify

After running it, you should see a large collection of built-in tasks. From there, you can check:

* whether there is already a manipulation or locomotion task close to your target use case
* which environments belong to direct tasks, manager-based tasks, or specific library workflows
* whether an existing task can serve as the starting point for your own robot or scene



## Full-series summary: the capability map you built

You have now completed a full progression from basic manipulation to architecture-level workflow decisions. Across this series, you developed the following capabilities:

* **Basic manipulation**: trained 7-DOF arm tasks such as Reach and Lift to build manipulation fundamentals
* **Precision interaction**: handled articulated objects with mechanical constraints and explored high-precision Factory tasks
* **Cooperative behavior**: used MARL workflows to study object handover and coordination across multiple agents
* **Natural motion**: used AMP to move from task completion toward more natural and expressive behavior
* **Architectural decision-making**: learned how to choose RL libraries based on task requirements and when to consider multi-GPU distributed training

More importantly, you are no longer only running isolated examples. You are starting to think in terms of **workflow design**: how to switch tasks, change training stacks, manage experiments, verify results, and continue iterating on a GPU-backed robotics workflow from the same Arm-based platform.

## Next steps

You now have the foundation to run, switch, extend, and manage multiple simulation scenarios in an Arm-based Isaac Sim / Isaac Lab environment.

From here, there are two natural directions to continue. First, treat the scripts in this Learning Path as reference implementations and adapt them to USD assets from your own robotics projects, including your own robot models, scenes, and task constraints. Second, package the workflows into a reproducible project structure with Docker, version pinning, and automated validation to improve portability and maintainability.
