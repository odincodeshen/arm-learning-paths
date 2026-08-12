---
title: Deploy a Small Language Model on Alif E8 with an Interactive Web GUI
description: Deploy the stories260K language model on the Alif E8 DevKit (Arm Cortex-M55 HE core), build firmware with FWAuto, flash it via SETOOLS, and interact with it through a real-time web GUI.

minutes_to_complete: 45

who_is_this_for: This Learning Path is for embedded software developers who want to deploy a small language model on an Alif E8 DevKit and interact with it through a browser-based dashboard.

learning_objectives:
    - Set up the Alif E8 DevKit and FWAuto development environment on Windows
    - Build and flash SLM firmware using FWAuto, CMake, and SETOOLS
    - Run a Flask web server that connects to the board over UART
    - Send text prompts from a browser and view model-generated responses in real time

prerequisites:
    - An [Alif E8 Development Kit](https://alifsemi.com/support/kits/ensemble-e8devkit/) with USB cable
    - A Windows PC (Windows 10 or 11)
    - Internet connection for downloading tools
    - A [FWAuto](https://fwauto.ai/) account (free registration)

author_primary: Mason Kuo

author:
    - Mason Kuo
    - Odin Shen
    - Edwin Ts

generate_summary_faq: true
rerun_summary: false
rerun_faqs: false

### Tags
skilllevels: Introductory
subjects: ML
armips:
    - Cortex-M
tools_software_languages:
    - Python
    - C
    - Flask
    - CMake
    - GCC

operatingsystems:
    - Windows

further_reading:
    - resource:
        title: Alif E8 DevKit documentation
        link: https://alifsemi.com/support/kits/ensemble-e8devkit/
        type: website
    - resource:
        title: stories260K - Karpathy's tiny LLM
        link: https://github.com/karpathy/llama2.c
        type: github
    - resource:
        title: FWAuto documentation
        link: https://fwauto.ai/
        type: website
    - resource:
        title: FWAuto on LinkedIn
        link: https://www.linkedin.com/company/fwauto
        type: website
    - resource:
        title: FWAuto on GitHub
        link: https://github.com/orgs/FWAutoOPEN/
        type: github

### FIXED, DO NOT MODIFY
# ================================================================================
weight: 1                       # _index.md always has weight of 1 to order correctly
layout: "learningpathall"       # All files under learning paths have this same wrapper
learning_path_main_page: "yes"  # This should be surfaced when looking for related content. Only set for _index.md of learning path content.
---

## Frequently asked questions

### Which USB port do I use for programming?

Connect a USB-C cable to the **PRG USB** port on the bottom edge of the DevKit. The green LED near the E1 device illuminates when power is applied. Do not use the MCU USB or Device USB ports for programming.

### What should I check before connecting the board?

Verify that the configuration jumpers match the factory defaults documented in the [DK-E8 User Guide](https://alifsemi.com/support/kits/ensemble-e8devkit/). Do not move, remove, or install jumpers with power applied to the board.

### My COM port is not COM3. What do I do?

Open Windows Device Manager and look under **Ports (COM & LPT)** for the J-Link CDC UART entry. Replace `COM3` with your actual port number in all commands and in `.fwauto/config.toml`. The Flask web server script (`web_demo_server.py`) also auto-detects the port by scanning available serial ports.

### The cmake --build tmp command fails because the tmp directory does not exist

The `tmp` directory is the CMake build tree created by the project's build system. Make sure you cloned with `--recursive` so the CMSIS board-library submodule is present:

```bash
git clone --recursive https://github.com/masonkuomeow/alif_slm_r.git
```

If you already cloned without `--recursive`, run `git submodule update --init` inside the repository.

### Do I need to run the FWAuto setup wizard?

No. The repository already includes a pre-configured `.fwauto/` directory. Verify the existing configuration and update the serial port to match your hardware. Only run `fwauto build` if the configuration file is missing or empty.

### The deploy script fails with a serial port error

Ensure no other terminal application (PuTTY, Tera Term, screen) is using the SEUART. The DevKit exposes only one SEUART interface. Also confirm that the SW4 switch is in the default **SEUART** position.

### What version of SETOOLS do I need?

Download **SETOOLS V1.110.000 or later** from the [Alif Software & Tools page](https://alifsemi.com/support/software-tools/ensemble/). After downloading, open a terminal in the SETOOLS directory and run `updateSystemPackage.exe -d` to initialize the toolkit.
