---
title: Overview and install dependencies
description: Learn how the stories260K language model runs on the Alif E8 DevKit and install Python, Node.js, uv, SEGGER J-Link, CMake, Ninja, and the Arm GNU Toolchain.
weight: 2
layout: "learningpathall"
---

## Overview

You deploy a small language model (SLM) on the [Alif E8 Development Kit](https://alifsemi.com/support/kits/ensemble-e8devkit/) and interact with it through a web browser.

The Alif E8 DevKit contains an [Arm Cortex-M55](https://developer.arm.com/Processors/Cortex-M55) microcontroller with enough memory to run a tiny 260,000-parameter language model. You install all required tools, build and flash the firmware, then start a web server to send prompts from your browser.

### What stories260K is

[stories260K](https://github.com/karpathy/llama2.c) is a tiny language model created by Andrej Karpathy as part of the llama2.c project. It has 260,000 parameters, five transformer layers, and a 512-token vocabulary. It is trained on children's stories, so it generates short, coherent text continuations.

The model is small enough to run entirely on the Cortex-M55 HE core of the Alif E8. The KV cache sits in SRAM and the model weights sit in MRAM.

### Architecture

The system connects a browser dashboard to the Alif E8 board through a [Flask](https://flask.palletsprojects.com/) web server. The web GUI provides an interface where you type prompts and view model-generated responses in real time, without a serial terminal.

```console
  Browser (localhost:5000)
      |
      | HTTP / SSE
      v
  Flask web server (web_demo_server.py)
      |
      | UART serial (COM3, 115200 baud)
      v
  Alif E8 Board (Cortex-M55 HE)
      |
      | stories260K inference engine
      v
  Model weights (MRAM) + KV cache (SRAM)
```

The browser connects to the Flask server via [Server-Sent Events (SSE)](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events) for real-time streaming. The server sends prompts to the board over UART and streams model output back to the browser.

## Install Python

[FWAuto](https://fwauto.ai/) needs [Python](https://www.python.org/) 3.10 or later.

{{< tabpane code=true >}}
  {{< tab header="macOS and Linux" language="bash" >}}
sudo apt update
sudo apt install python3 python3-pip
  {{< /tab >}}

  {{< tab header="Windows" language="text" >}}
Download the installer from python.org/downloads. Check **"Add Python to PATH"** during installation.
  {{< /tab >}}
{{< /tabpane >}}

{{% notice Note %}}
On macOS, you can also use `brew install python` if you have Homebrew installed.
{{% /notice %}}

Verify:

```bash
python --version
```

You should see output similar to:

```output
Python 3.12.4
```

## Install Node.js

FWAuto needs [Node.js](https://nodejs.org/) 20 or later.

{{< tabpane code=true >}}
  {{< tab header="macOS and Linux" language="bash" >}}
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
  {{< /tab >}}

  {{< tab header="Windows" language="powershell" >}}
winget install OpenJS.NodeJS.LTS
  {{< /tab >}}
{{< /tabpane >}}

{{% notice Note %}}
On macOS, you can also use `brew install node@20` if you have Homebrew installed.
{{% /notice %}}

Verify:

```bash
node --version
```

You should see output similar to:

```output
v20.15.0
```

## Install uv

[uv](https://docs.astral.sh/uv/) is a Python package manager that FWAuto uses.

{{< tabpane code=true >}}
  {{< tab header="macOS and Linux" language="bash" >}}
curl -LsSf https://astral.sh/uv/install.sh | sh
  {{< /tab >}}

  {{< tab header="Windows (PowerShell)" language="powershell" >}}
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
  {{< /tab >}}
{{< /tabpane >}}

Close and reopen your terminal after installation. Verify:

```bash
uv --version
```

You should see output similar to:

```output
uv 0.4.0
```

## Install SEGGER J-Link software

The [SEGGER J-Link](https://www.segger.com/downloads/jlink/) software provides the debug probe driver and tools for communicating with the Alif E8 board.

1. Go to [segger.com/downloads/jlink](https://www.segger.com/downloads/jlink/).
2. Download the **J-Link Software and Documentation Pack** for your operating system.
3. Run the installer with default settings.

Verify:

```bash
JLinkExe --version
```

## Install build tools

The firmware build needs [CMake](https://cmake.org/) 3.31.5 or later, [Ninja](https://ninja-build.org/) 1.12.0 or later, the [Arm GNU Toolchain](https://developer.arm.com/downloads/-/gnu-rm), and the [CMSIS-Toolbox](https://github.com/Open-CMSIS-Pack/cmsis-toolbox).

**Install CMake:**

{{< tabpane code=true >}}
  {{< tab header="macOS and Linux" language="bash" >}}
sudo apt install cmake
  {{< /tab >}}

  {{< tab header="Windows" language="text" >}}
Download from cmake.org/download and select **"Add CMake to the system PATH"** during installation.
  {{< /tab >}}
{{< /tabpane >}}

{{% notice Note %}}
On macOS, you can also use `brew install cmake` if you have Homebrew installed.
{{% /notice %}}

**Install Ninja:**

{{< tabpane code=true >}}
  {{< tab header="macOS and Linux" language="bash" >}}
sudo apt install ninja-build
  {{< /tab >}}

  {{< tab header="Windows" language="powershell" >}}
winget install Ninja-build.Ninja
  {{< /tab >}}
{{< /tabpane >}}

{{% notice Note %}}
On macOS, you can also use `brew install ninja` if you have Homebrew installed.
{{% /notice %}}

**Install the Arm GNU Toolchain:**

Go to [developer.arm.com/downloads](https://developer.arm.com/downloads/-/gnu-rm) and download the installer for your operating system. On Windows, select **"Add to PATH"** during installation. On Linux and macOS, extract the archive and add the `bin` directory to your `PATH`.

**Install the CMSIS-Toolbox:**

The [CMSIS-Toolbox](https://github.com/Open-CMSIS-Pack/cmsis-toolbox) manages CMSIS software packs and the build flow. Download the latest release from the [Arm Tools Artifactory](https://artifactory.keil.arm.com/artifactory/cmsis-toolbox/) for your platform and extract it to a directory such as `C:\cmsis-toolbox`.

Add the `bin` directory to your `PATH` and register the compiler:

{{< tabpane code=true >}}
  {{< tab header="Windows (PowerShell)" language="powershell" >}}
# Add CMSIS-Toolbox to PATH (current session)
$env:PATH += ";C:\cmsis-toolbox\bin"

# Register GCC toolchain (adjust version and path to match your installation)
$env:GCC_TOOLCHAIN_14_2_1 = "C:\Program Files\Arm GNU Toolchain arm-none-eabi\14.2\bin"

# Initialize the pack index
cpackget init https://www.keil.com/pack/index.pidx
  {{< /tab >}}

  {{< tab header="macOS and Linux" language="bash" >}}
export PATH=~/cmsis-toolbox/bin:$PATH
export GCC_TOOLCHAIN_14_2_1=/opt/gcc-arm-none-eabi/bin
cpackget init https://www.keil.com/pack/index.pidx
  {{< /tab >}}
{{< /tabpane >}}

Verify all four tools:

```bash
cmake --version
ninja --version
arm-none-eabi-gcc --version
cbuild --version
```

## Verify your setup

Before proceeding, verify that all tools are installed:

```bash
python --version
node --version
uv --version
cmake --version
ninja --version
arm-none-eabi-gcc --version
JLinkExe --version
```

You should see output similar to:

```output
Python 3.12.4
v20.15.0
uv 0.4.0
cmake version 3.28.0
1.11.1
arm-none-eabi-gcc (GNU Arm Embedded Toolchain) 14.2.1
J-Link Commander V7.94
```

## What you've learned and what's next

You've now learned how the stories260K model runs on the Alif E8 DevKit and installed all required dependencies: Python, Node.js, uv, SEGGER J-Link, CMake, Ninja, and the Arm GNU Toolchain.

Next, you set up the hardware and FWAuto development environment.
