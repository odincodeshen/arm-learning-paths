---
title: Use FWAuto for a closed-loop workflow
description: Replace the manual build-and-flash workflow with FWAuto's AI-assisted inspection, configuration, and slash-command automation.
weight: 7
layout: "learningpathall"
---

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

## What you've learned and what's next

You've now built the SLM firmware from source and flashed it to the Alif E8 DevKit, either through manual commands or through FWAuto's `/build` and `/deploy` slash commands. The board is running the stories260K inference engine and is ready to accept prompts.

Next, you start the Flask web server and interact with the model through the browser dashboard.
