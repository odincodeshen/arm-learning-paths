---
title: Manipulate Objects with a 7-DOF Robot Arm
weight: 2

### FIXED, DO NOT MODIFY
layout: learningpathall
---

## From locomotion to interaction

In the previous [Learning Path](https://learn.arm.com/learning-paths/laptops-and-desktops/dgx_spark_isaac_robotics/), you installed and ran [Isaac Sim](https://developer.nvidia.com/isaac/sim) and [Isaac Lab](https://developer.nvidia.com/isaac/lab) on an Arm-based [DGX Spark](https://www.nvidia.com/en-gb/products/workstations/dgx-spark/) system. In this section, you move from locomotion to manipulation.

You will train a [Franka 7-DOF](https://franka.de/franka-research-3) robotic arm on two tasks:

* **Reach** to build spatial control of the end-effector.
* **Lift** to add contact, grasping, and stable object motion.

This workflow also shows how DGX Spark maps work across Arm CPU and GPU resources. Python orchestration and task control run on the Grace CPU, while simulation and learning kernels run on the Blackwell GPU.


## Task 1: Reach — Building Spatial Awareness

The Reach task trains the Franka arm to move its end-effector to a randomly sampled target pose. This is your first manipulation baseline because it teaches position control before adding grasping.


### Run

Use your existing Isaac Lab setup from the previous Learning Path, then run:

```bash
cd ~/IsaacLab

# Improve runtime compatibility on aarch64 systems
export LD_PRELOAD="$LD_PRELOAD:/lib/aarch64-linux-gnu/libgomp.so.1"

./isaaclab.sh -p scripts/reinforcement_learning/rsl_rl/train.py \
    --task=Isaac-Reach-Franka-v0 \
    --headless \
    --num_envs=2048
```

### What this script controls

This command does more than start training. The Python entry point controls:

* which task configuration is loaded
* which RL training entry point is used
* runtime behavior such as headless execution and the number of environments

In Isaac Lab, an **environment** is one simulated instance of the task. For example, one environment includes one Franka arm, one target, and one physics rollout. When you set `--num_envs=2048`, Isaac Lab runs 2048 instances in parallel. PPO then uses trajectories from all environments to update the actor and critic networks each iteration.

This script-level control is well suited to rapid iteration on an Arm-based system. The Arm CPU handles task launch and workflow control, while the GPU handles simulation and learning throughput.

### Task structure

* **Goal**: Move the end-effector of the Franka 7-DOF arm to a randomly sampled target pose.
* **Observation space**: Joint positions, joint velocities, and target position.
* **Action space**: Joint position targets.

### Verify

After training, run the following command to observe the learned policy in simulation:

```bash
./isaaclab.sh -p scripts/reinforcement_learning/rsl_rl/play.py \
    --task=Isaac-Reach-Franka-Play-v0 \
    --num_envs=2 \
    --checkpoint=logs/rsl_rl/franka_reach/<date_of_training>/model_<iteration_number>.pt
```

{{% notice Tip %}}

To inspect the Franka arm, right-click in the viewport and use `W`, `A`, `S`, and `D` to fly the camera. These are standard industry viewport navigation controls used in many 3D tools.

{{% /notice %}}

![Franka Reach training comparison that shows early and late policy behavior. The left side shows less stable motion around iteration 100, and the right side shows improved target tracking near iteration 999.#center](./reach.gif "Franka 7-DOF arm learning the Reach task. The left panel shows behavior after 100 training iterations, and the right panel shows behavior after 999 iterations.")

You should observe the following:

* The robotic arm can consistently move its end-effector to the target position.
* Multiple environments execute the reaching behavior in parallel.
* The policy no longer shows obvious random oscillation or unstable motion.

### Why it matters on Arm

The Arm value in this example is not about claiming CPU-dominant simulation performance. Instead, it demonstrates the **Arm CPU as a control plane**: on an Arm-based development system, you can switch tasks, launch experiments, and evaluate policies directly through scripts, enabling fast iteration for robotics simulation workflows.


## Task 2: Lift — balancing force and precision

Once the robot can reach reliably, the next step is physical interaction. In the Lift task, you train the arm to grasp a cube on the table and lift it to a target height. The policy must coordinate approach, alignment, gripper closure, and stable lifting under contact and gravity.

### Run

```bash
./isaaclab.sh -p scripts/reinforcement_learning/rsl_rl/train.py \
    --task=Isaac-Lift-Cube-Franka-v0 \
    --headless \
    --num_envs=2048
```

{{% notice Please Note %}}

After an initial run, the end-effector might still fail to lift consistently. To continue training from a checkpoint, add:

```bash
--resume \
  --experiment_name=franka_lift \
  --load_run=<time stamp of run> \
  --checkpoint=model_<iteration>.pt \
  --max_iterations=<number of additional iterations>
```

{{% /notice %}}

The training log prints a **learning-iteration summary** each cycle. Watch `Mean reward` to judge whether the policy is still improving. You can see jumps and plateaus during PPO training, so short flat periods are normal. Use the broader trend across many iterations, together with ETA, to decide whether to keep training.

```output
################################################################################
                          Learning iteration 902/2650                            

                            Total steps: 37011456 
                       Steps per second: 68548 
                        Collection time: 0.600s 
                          Learning time: 0.117s 
                        Mean value loss: 2.1550
                    Mean surrogate loss: -0.0023
                      Mean entropy loss: 7.1831
                            Mean reward: 79.76
                    Mean episode length: 246.64
                        Mean action std: 0.64
         Episode_Reward/reaching_object: 0.7022
          Episode_Reward/lifting_object: 11.2984
    Episode_Reward/object_goal_tracking: 5.6466
Episode_Reward/object_goal_tracking_fine_grained: 0.0891
             Episode_Reward/action_rate: -0.7642
               Episode_Reward/joint_vel: -1.4792
                 Curriculum/action_rate: -0.1000
                   Curriculum/joint_vel: -0.1000
     Metrics/object_pose/position_error: 0.2638
  Metrics/object_pose/orientation_error: 0.8218
           Episode_Termination/time_out: 0.9782
    Episode_Termination/object_dropping: 0.0218
--------------------------------------------------------------------------------
                         Iteration time: 0.72s
                           Time elapsed: 00:09:59
                                    ETA: 00:23:11
```


### What changes in the workflow

Compared with Reach, you do not rebuild the project or switch platforms. You keep the same training entry point and environment, and only change `--task`. That lets you move quickly between manipulation scenarios while keeping the same Arm-based workflow.

### Verify

After training, confirm the following:

* The robotic arm can approach the cube and adjust its gripper position.
* The gripper closes at an appropriate time.
* The cube is lifted off the table rather than slipping away or bouncing after collision.


You can use the same way to verify the result.

```bash
./isaaclab.sh -p scripts/reinforcement_learning/rsl_rl/play.py \
    --task=Isaac-Lift-Cube-Franka-v0 \
  --num_envs=2 \
    --checkpoint=<path to model*.pt file>
```

![Franka 7-DOF arm progressing through Reach and Lift. The left panel shows iteration 150, where grasp stability is still developing. The right panel shows around iteration 900, where the policy keeps the end-effector inverted to reduce cube drops during lifting.#center](./reach_and_lift.gif "Franka 7-DOF Reach and Lift training progression. Left: iteration 150. Right: iteration 900. As training progresses, the arm keeps the end-effector more upright, which makes cube retention more stable and reduces drops which incurs a penalty.")




## Extended exploration: comparing different locomotion robots

Isaac Lab also includes locomotion environments you can switch to with the same script pattern. If you want a quick comparison, run one quadruped task and one biped task to observe convergence differences.

| Environment | Robot | Type | Terrain | Training difficulty |
|---|---|---|---|---|
| Isaac-Velocity-Flat-Unitree-Go2-v0 | Unitree Go2 | Quadruped | Flat | Easy |
| Isaac-Velocity-Rough-H1-v0 | Unitree H1 | Biped humanoid | Rough | Hard |

Quadrupeds often converge faster because they are more statically stable. Bipeds usually need longer training because balance is harder to learn.

## Next up

The Franka robotic arm now has basic grasping ability. However, objects in the real world often introduce more complex mechanical constraints.

In the next section, you will explore how a robot can interact with joint-constrained objects such as drawers, and move one step closer to high-precision industrial manipulation tasks.
