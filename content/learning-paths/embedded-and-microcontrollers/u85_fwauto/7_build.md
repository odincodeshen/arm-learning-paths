---
title: Build the firmware
description: Compile the stories260K firmware for the Alif E8 Cortex-M55 HE core using CMake and the Arm GNU Toolchain.
weight: 7
layout: "learningpathall"
---

## Understand the manual workflow

In a manual firmware deployment workflow, you identify the correct project settings, build commands, and deployment procedures yourself. This means inspecting the repository structure, reading any available documentation, and executing a series of commands by hand.

The `.fwauto/config.toml` configuration file is pre-configured in the repository. In a manual workflow, you can either use the existing configuration or override the values by passing them as command-line arguments. You determine the correct firmware directory, CMake target, binary output path, serial port, and board target on your own.

## Build the firmware

Open a terminal, navigate to the firmware directory, and build:

```bash
cd alif_slm_r/alif_vscode-template
cmake --build tmp --target stories260k_runner.debug+E8-HE
```

{{% notice Note %}}
The `tmp` directory is the CMake build tree. It is created by the project's build system when you first run the build command shown above. If the build fails because `tmp` does not exist, check that you have run `git clone --recursive` so the CMSIS board-library submodule is present.
{{% /notice %}}

This compiles the firmware for the [Cortex-M55](https://developer.arm.com/Processors/Cortex-M55) HE core. The build produces:

- `alif_vscode-template/out/stories260k_runner/E8-HE/debug/stories260k_runner.elf` (ELF with debug symbols)
- `alif_vscode-template/out/stories260k_runner/E8-HE/debug/stories260k_runner.bin` (raw binary for flashing)

You should see output similar to:

```output
[0/6] Performing build step for 'stories260k_runner.debug+E8-HE'

Building CMake target 'stories260k_runner.debug+E8-HE'
Using compiler: GCC V14.2.1

[1/40] Building C object .../llm_infer.c.obj
[2/40] Building C object .../main.c.obj
...
[40/40] Linking C executable stories260k_runner.elf

MRAM: 1149296 B / 2 MB (54.80%)

[5/5] Completed 'stories260k_runner.debug+E8-HE'
```

The MRAM usage line shows that the firmware (model weights and code) occupies about 55% of the available 2 MB MRAM.

## What you've learned and what's next

You've now built the stories260K firmware from source using CMake and the Arm GNU Toolchain. The build produces an ELF binary with debug symbols and a raw binary for flashing, and the MRAM usage report confirms that the model fits within the Alif E8's 2 MB MRAM.

Next, you flash the firmware to the Alif E8 DevKit and verify the board is running.
