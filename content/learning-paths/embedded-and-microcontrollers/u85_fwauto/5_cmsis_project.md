---
title: Understand the CMSIS project configuration
description: Learn how the CMSIS solution file, project file, and device layer define the build for the Alif E8 Cortex-M55 HE core.
weight: 5
layout: "learningpathall"
---

## What CMSIS is

The [CMSIS (Cortex Microcontroller Software Interface Standard)](https://arm-software.github.io/CMSIS_6/) is a hardware abstraction layer from Arm that provides a consistent interface across Cortex-M processors. The [CMSIS-Toolbox](https://github.com/Open-CMSIS-Pack/cmsis-toolbox) extends this with a YAML-based build system that replaces hand-written Makefiles for embedded projects.

In this project, CMSIS-Toolbox manages:

- Which processor core to target (M55 HE or M55 HP)
- Which device and board support packages to link
- Which compiler to use and what flags to pass
- How the linker places code and data into memory

The firmware uses **CMake** for the actual build, but the CMSIS solution files define the project structure that CMake consumes.

## The solution file: alif.csolution.yml

The top-level configuration is `alif_vscode-template/alif.csolution.yml`. It defines the entire build:

```yaml
solution:
  compiler: GCC

  packs:
    - pack: AlifSemiconductor::Ensemble@2.0.4
    - pack: ARM::CMSIS@6.0.0
    - pack: ARM::CMSIS-Compiler@2.1.0

  target-types:
    - type: E8-HE
      device: Alif Semiconductor::AE822FA0E5597LS0:M55_HE
      board: Alif Semiconductor::DevKit-E8
      define:
        - "CORE_M55_HE"

  build-types:
    - type: debug
      optimize: none
      debug: on
    - type: release
      optimize: speed
      debug: on

  projects:
    - project: stories260k_runner/stories260k_runner.cproject.yml
```

Key sections explained:

| Section | Purpose |
|---------|---------|
| `packs` | CMSIS packs to download. `AlifSemiconductor::Ensemble` provides device headers, startup code, and BSP. `ARM::CMSIS` provides the Cortex-M core support. `ARM::CMSIS-Compiler` provides printf/scanf retargeting. |
| `target-types` | Hardware targets. `E8-HE` selects the Cortex-M55 HE core on the AE822FA0E5597LS0 device. The `CORE_M55_HE` define is passed to the compiler so the code can use `#ifdef CORE_M55_HE`. |
| `build-types` | `debug` disables optimization and enables debug symbols. `release` enables speed optimization. |
| `projects` | Lists the project files that belong to this solution. |

The solution supports multiple targets (E7-HE, E7-HP, E8-HE, E8-HP, E1C-HE) so the same code can run on different Alif boards. This Learning Path uses the **E8-HE** target.

## The project file: stories260k_runner.cproject.yml

Each project has a `.cproject.yml` file. For the stories260K runner:

```yaml
project:
  groups:
    - group: App
      files:
        - file: main.c
        - file: llm_infer.c
    - group: Model
      files:
        - file: model_data.h
        - file: tokenizer_data.h

  output:
    base-name: $Project$
    type:
      - elf
      - bin

  layers:
    - layer: ../device/ensemble/alif-ensemble.clayer.yml

  components:
    - component: ARM::CMSIS-Compiler:STDOUT:Custom
    - component: ARM::CMSIS-Compiler:STDIN:Custom
    - component: AlifSemiconductor::Services:Retarget IO:STDOUT
    - component: AlifSemiconductor::Services:Retarget IO:STDIN
```

| Section | Purpose |
|---------|---------|
| `groups` | Source file groups. `App` contains the application logic. `Model` contains the embedded model weights and tokenizer vocabulary as C headers. |
| `output` | Produces both `.elf` (with debug symbols) and `.bin` (raw binary for flashing). |
| `layers` | References the shared device layer (see below). |
| `components` | CMSIS components for I/O retargeting. `STDOUT:Custom` and `STDIN:Custom` redirect printf/scanf through the Alif retarget IO service, which sends output over UART. |

## The device layer: alif-ensemble.clayer.yml

The device layer at `device/ensemble/alif-ensemble.clayer.yml` provides the hardware abstraction:

```yaml
layer:
  components:
    - component: ARM::CMSIS:CORE
    - component: AlifSemiconductor::Device:Startup
    - component: AlifSemiconductor::Device:SOC Peripherals:PINCONF
    - component: AlifSemiconductor::Device:SOC Peripherals:GPIO
    - component: AlifSemiconductor::CMSIS Driver:USART
    - component: Services:Secure Enclave:core&Source
    - component: AlifSemiconductor::BSP:DevKit Config&DevKit-e8
      for-context:
        - +E8-HE
        - +E8-HP
```

| Component | Purpose |
|-----------|---------|
| `ARM::CMSIS:CORE` | Cortex-M55 core support (startup, system init, interrupt handlers) |
| `Device:Startup` | Alif E8 device startup code (clock init, power management) |
| `SOC Peripherals:PINCONF` | Pin configuration driver |
| `SOC Peripherals:GPIO` | GPIO driver for LEDs and switches |
| `CMSIS Driver:USART` | UART driver for serial communication |
| `Secure Enclave:core` | Secure Enclave communication (required for SETOOLS flashing) |
| `BSP:DevKit Config&DevKit-e8` | Board-specific configuration for the E8 DevKit |

## How cbuild builds the project

When you run:

```bash
cbuild alif.csolution.yml --context stories260k_runner.debug+E8-HE
```

CMSIS-Toolbox:

1. Resolves the CMSIS packs (downloads them if not cached)
2. Reads `alif.csolution.yml` to find the E8-HE target and debug build type
3. Reads `stories260k_runner.cproject.yml` to find source files and components
4. Reads `alif-ensemble.clayer.yml` to include device BSP and drivers
5. Generates CMake files in the `tmp/` build directory
6. Invokes CMake with the Ninja generator to compile and link

The context string `stories260k_runner.debug+E8-HE` selects the project (`stories260k_runner`), build type (`debug`), and target (`E8-HE`).

## What you've learned and what's next

You've now understood how the CMSIS solution file, project file, and device layer work together to define the build for the Alif E8 Cortex-M55 HE core. The YAML files describe the processor target, source files, CMSIS components, and hardware abstraction -- and CMSIS-Toolbox translates them into a CMake build.

Next, you learn how the firmware's code and data are placed into the Alif E8's memory regions.
