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

{{< accordion title="What is stories260K and why is it suitable for the Alif E8?" >}}
stories260K is a tiny language model created by Andrej Karpathy as part of the llama2.c project. It has 260,000 parameters, five transformer layers, and a 512-token vocabulary. Its small footprint fits within the Cortex-M55 HE core's SRAM and MRAM on the Alif E8 DevKit, making it one of the few language models that can run entirely on a microcontroller without external memory.
{{< /accordion >}}

{{< accordion title="Do I need an internet connection after the initial setup?" >}}
No. Once you have installed all the required tools, cloned the repository, and built the firmware, the board runs standalone. The Flask web server communicates with the Alif E8 over a local UART connection, so the browser dashboard works offline on the same machine.
{{< /accordion >}}

{{< accordion title="Which operating systems are supported?" >}}
The primary supported platform is Windows 10 or 11. The toolchain installers (Arm GNU Toolchain, SEGGER J-Link, CMake, Ninja, and the CMSIS-Toolbox) are also available for macOS and Linux, but the FWAuto workflow commands and SETOOLS flashing steps in this Learning Path are written for Windows.
{{< /accordion >}}

{{< accordion title="What baud rate does the UART connection use and how do I find the COM port?" >}}
The UART connection runs at 115200 baud. On Windows, open Device Manager and look under **Ports (COM & LPT)** for a device labeled "J-Link" or "SEGGER". The Flask web server script (`web_demo_server.py`) also auto-detects the port by scanning available serial ports, so you usually do not need to set it manually.
{{< /accordion >}}

{{< accordion title="Can I use a different language model on the Alif E8?" >}}
The firmware is specifically compiled for the stories260K model architecture and tokenizer. Deploying a different model would require modifying the inference engine, adjusting memory maps, and retraining or re-quantizing the weights to fit within the Alif E8's available SRAM and MRAM. The llama2.c project includes other small model variants you can experiment with, but expect significant firmware changes.
{{< /accordion >}}

{{< accordion title="How do I troubleshoot if the board is not detected?" >}}
First, verify the USB cable is connected and the board's power LED is on. Check Device Manager for the COM port and J-Link driver. If the driver is missing, re-install the SEGGER J-Link software. Make sure no other application (such as a serial terminal) has the COM port open. Run `fwauto` in the project root and type `/status` to check the board connection.
{{< /accordion >}}

{{< accordion title="What is FWAuto and how does it help with firmware development?" >}}
FWAuto is an AI-assisted firmware development tool. It provides a command-line interface where you describe changes in natural language, and it generates firmware patches, builds, and deploys them. In this Learning Path, FWAuto automates the build, flash, and verification steps so you can focus on the model and web GUI rather than the build system.
{{< /accordion >}}
