---
title: Reproduce Human Motion with Adversarial Motion Priors (AMP)
weight: 5

### FIXED, DO NOT MODIFY
layout: learningpathall
---


## Advanced locomotion: Adversarial Motion Priors (AMP)

For more natural and human-like robot motion, Isaac Lab supports Adversarial Motion Priors (AMP). AMP uses reference motion capture data to guide the RL policy toward realistic movements:

```bash
# Humanoid walking with AMP
./isaaclab.sh -p scripts/reinforcement_learning/skrl/train.py \
    --task=Isaac-Humanoid-AMP-Walk-Direct-v0 \
    --headless \
    --algorithm AMP

# Humanoid running with AMP
./isaaclab.sh -p scripts/reinforcement_learning/skrl/train.py \
    --task=Isaac-Humanoid-AMP-Run-Direct-v0 \
    --headless \
    --algorithm AMP

# Humanoid dancing with AMP
./isaaclab.sh -p scripts/reinforcement_learning/skrl/train.py \
    --task=Isaac-Humanoid-AMP-Dance-Direct-v0 \
    --headless \
    --algorithm AMP
```

AMP adds a discriminator network that distinguishes between the agent's behavior and the reference motion data. The agent receives an additional reward for matching the reference style, producing more natural-looking locomotion.

| **AMP task** | **Reference motion** | **Result** |
|-------------|---------------------|-----------|
| `Isaac-Humanoid-AMP-Walk-Direct-v0` | Human walking capture | Natural walking gait |
| `Isaac-Humanoid-AMP-Run-Direct-v0` | Human running capture | Realistic running motion |
| `Isaac-Humanoid-AMP-Dance-Direct-v0` | Human dance capture | Creative dance movements |
