---
title: Deploy the SLM firmware
description: Understand the Alif SLM project structure, build the firmware manually or with FWAuto, flash it to the Alif E8 DevKit, and verify the board is running.
weight: 4
layout: "learningpathall"
---

## Understand the project structure

Before building and deploying, it helps to understand the key directories and files within the `alif_slm_r` repository. This knowledge is essential for a manual workflow, where you need to locate specific build scripts and binaries yourself.

The project contains two firmware implementations:

| Directory | Description |
|---|---|
| `workshop-ethos-u/` | Batch-mode firmware (runs preset prompts, then stops) |
| `alif_vscode-template/` | Interactive firmware with [UART](https://developer.mozilla.org/en-US/docs/Glossary/UART) input, [BPE](https://huggingface.co/learn/nlp-course/chapter6/5) tokenizer, and READY/DONE markers |

You use the **interactive firmware** in `alif_vscode-template/`.

Key source files:

| File | Description |
|---|---|
| `alif_vscode-template/stories260k_runner/main.c` | Main application: interactive loop, stdin reader, prompt handler |
| `alif_vscode-template/stories260k_runner/llm_infer.c` | LLM inference engine: transformer forward pass, BPE tokenizer |
| `alif_vscode-template/stories260k_runner/llm_infer.h` | Data structures and API declarations |
| `alif_vscode-template/stories260k_runner/model_data.h` | Embedded model weights (260K parameters, ~1.0 MB) |
| `alif_vscode-template/stories260k_runner/tokenizer_data.h` | Embedded BPE tokenizer vocabulary (512 tokens) |

## Understand the manual workflow

In a manual firmware deployment workflow, you identify the correct project settings, build commands, and deployment procedures yourself. This means inspecting the repository structure, reading any available documentation, and executing a series of commands by hand.

The `.fwauto/config.toml` configuration file is pre-configured in the repository. In a manual workflow, you can either use the existing configuration or override the values by passing them as command-line arguments. You determine the correct firmware directory, CMake target, binary output path, serial port, and board target on your own.

### Build the firmware (manual)

Open a terminal, navigate to the firmware directory, and build:

{{< tabpane code=true >}}
  {{< tab header="macOS and Linux" language="bash" >}}
cd alif_slm_r/alif_vscode-template
cmake --build tmp --target stories260k_runner.debug+E8-HE
  {{< /tab >}}

  {{< tab header="Windows (PowerShell)" language="powershell" >}}
cd alif_slm_r/alif_vscode-template
cmake --build tmp --target stories260k_runner.debug+E8-HE
  {{< /tab >}}
{{< /tabpane >}}

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

### Flash the firmware (manual)

Navigate back to the project root and run the deploy script:

{{< tabpane code=true >}}
  {{< tab header="Windows (PowerShell)" language="powershell" >}}
cd ..
python deploy_setools.py "alif_vscode-template/out/stories260k_runner/E8-HE/debug/stories260k_runner.bin" --com COM3
  {{< /tab >}}
{{< /tabpane >}}

Replace `COM3` with the serial port assigned to your board:

| How to find your port | |
|---|---|
| Windows | Device Manager > Ports (COM & LPT) > J-Link CDC UART |
| Linux | `/dev/ttyACM0` |
| macOS | `/dev/cu.usbmodem1101` |

{{% notice Important %}}
The deploy script `deploy_setools.py` uses the Alif Security Toolkit (SETOOLS). Make sure you have installed SETOOLS and that the board firmware version is compatible with the toolkit version. See the [Alif Security Toolkit](https://alifsemi.com/support/kits/ensemble-e8devkit/) page for the latest release and version compatibility notes.
{{% /notice %}}

The board's EN/DIS switch stays in the **EN** position -- the script enters SE maintenance mode automatically over UART.

The script:

1. Copies the binary to the SETOOLS build directory
2. Generates an ATOC (Application Table of Contents) package
3. Enters soft maintenance mode
4. Writes the firmware to [MRAM](https://en.wikipedia.org/wiki/Magnetoresistive_random-access_memory)

You should see:

```output
============================================================
  Alif SETOOLS Deploy
============================================================

[PREP] Source: .../stories260k_runner.bin (1149296 bytes)
[PREP] ATOC OK.

[MAINT] Soft Maintenance Mode ACTIVE.
[FLASH] Completed (rc=0).

============================================================
  DEPLOY SUCCESSFUL
============================================================
```

After flashing, the board reboots automatically and runs the new firmware.

### Verify the firmware is running (manual)

Open a serial terminal to check the board output.

{{< tabpane code=true >}}
  {{< tab header="macOS" language="bash" >}}
screen /dev/cu.usbmodem1101 115200
  {{< /tab >}}

  {{< tab header="Linux" language="bash" >}}
screen /dev/ttyACM0 115200
  {{< /tab >}}

  {{< tab header="Windows (PowerShell)" language="powershell" >}}
powershell -Command "$p = New-Object System.IO.Ports.SerialPort('COM3',115200,'None',8,'One'); $p.ReadTimeout=5000; $p.Open(); Start-Sleep -Seconds 3; $d=$p.ReadExisting(); $p.Close(); Write-Host $d"
  {{< /tab >}}
{{< /tabpane >}}

You should see output similar to:

```output
============================================
 stories260K LLM - Alif E8 Board
 Object Classification Demo
============================================

Model: 260K params, dim=64, 5 layers
Tokenizer: 512 tokens (BPE)

[MAIN] Initializing model...
[LLM] Config: dim=64 hidden=172 layers=5 heads=8 kv=4 vocab=512 seq=512

[MAIN] Model initialized OK

[MAIN] Initializing tokenizer...
[TOK] Loaded 512 tokens

[MAIN] Tokenizer initialized OK

============================================
 Interactive Classification Mode
============================================

READY>
Enter prompt:
```

If you see `READY>` and `Enter prompt:`, the firmware is running correctly.

{{% notice Note %}}
To exit `screen`, press `Ctrl+A` then `K`, then confirm with `y`.
{{% /notice %}}

## Manual configuration is an open loop

In a manual workflow, you are the feedback path. The loop breaks at every point where it needs a value only you can supply -- so it isn't a loop at all, it's a straight line you stitch together by hand.

| What the loop needs to know | Where it breaks without it |
|------|----------------|
| Which directory holds the firmware | The loop stops at the starting line, waiting for you |
| Which build target | Pick the wrong one and the build still succeeds -- an open loop is only as accurate as its model |
| Which board, probe, and port | Flash stops and waits for you -- the second break |
| That these stay in sync | The model goes stale, and the system runs steadily toward the wrong result |

## Use FWAuto to inspect the project

Instead of manually inspecting the repository, you can ask FWAuto to do it for you. FWAuto reads the project structure, identifies the build system, locates the firmware directories, and determines the required configuration values -- all from the repository itself.

This step takes you out of the feedback path. Once FWAuto has written the config, the loop can close on itself for the first time: build, flash, read the board back, find the mismatch, know which target to rebuild and where to reflash, and run again.

## Prompt FWAuto to generate the workflow

To use FWAuto's inspection capabilities, provide it with a prompt that describes the task:

```text
Inspect this repository and generate the firmware update workflow for the Alif SLM project. The configuration file is currently empty. Determine which configuration values need to be updated, explain why each value is needed, and generate the commands required to prepare and deploy the firmware. Use the current repository structure as the source of truth.
```

This prompt is effective because:

- It tells FWAuto to inspect the current repository, preventing assumptions about external or predefined project structures.
- It tells FWAuto to use the local repository as the definitive source for project details.
- It asks FWAuto to explain why each configuration value is needed, which helps you understand the project before executing anything.
- It directs FWAuto to generate executable commands for building and deploying the firmware.

{{% notice Note %}}
The repository includes a pre-configured `.fwauto/config.toml`. If you want to start from scratch, delete the `.fwauto/` directory before running the prompt above.
{{% /notice %}}

## Review the generated configuration updates

After FWAuto inspects the project and processes your prompt, it generates proposed configuration updates for the `.fwauto/config.toml` file, along with the corresponding build and deploy commands.

Review these generated outputs before executing them. Confirm that:

- The firmware directory matches `alif_vscode-template/`
- The build target is `stories260k_runner.debug+E8-HE`
- The deploy script path and arguments are correct
- The serial port matches your hardware setup

This review step ensures that FWAuto has correctly interpreted the project structure and that the generated workflow is safe to execute.

## Deploy the firmware with FWAuto

Once you have reviewed and confirmed the FWAuto configuration, you can use slash commands to build and flash. From the project root, start FWAuto in chat mode:

```bash
fwauto
```

Then run the build command:

```bash
/build
```

`/build` runs the same CMake command shown in the manual workflow. You should see the same compiler output and MRAM usage report.

After the build succeeds, flash the firmware:

```bash
/deploy
```

`/deploy` runs the deploy script, enters SE maintenance mode, and writes the firmware to MRAM. You should see the same SETOOLS output as the manual workflow.

## What you've accomplished and what's next

You've now built the SLM firmware from source and flashed it to the Alif E8 DevKit, either through manual commands or through FWAuto's `/build` and `/deploy` slash commands. The board is running the stories260K inference engine and is ready to accept prompts.

Next, you start the Flask web server and interact with the model through the browser dashboard.
