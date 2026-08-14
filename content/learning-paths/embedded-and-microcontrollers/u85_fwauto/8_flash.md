---
title: Flash and verify the firmware
description: Flash the compiled firmware to the Alif E8 DevKit with SETOOLS and verify the board is running by reading serial output.
weight: 8
layout: "learningpathall"
---

## Flash the firmware

Navigate back to the project root and run the deploy script:

{{< tabpane code=true >}}
  {{< tab header="macOS" language="bash" >}}
cd ..
python deploy_setools.py "alif_vscode-template/out/stories260k_runner/E8-HE/debug/stories260k_runner.bin" --com /dev/cu.usbmodem1101
  {{< /tab >}}

  {{< tab header="Linux" language="bash" >}}
cd ..
python deploy_setools.py "alif_vscode-template/out/stories260k_runner/E8-HE/debug/stories260k_runner.bin" --com /dev/ttyACM0
  {{< /tab >}}

  {{< tab header="Windows (PowerShell)" language="powershell" >}}
cd ..
python deploy_setools.py "alif_vscode-template\out\stories260k_runner\E8-HE\debug\stories260k_runner.bin" --com COM3
  {{< /tab >}}
{{< /tabpane >}}

Replace `COM3` with the serial port assigned to your board:

| How to find your port | |
|---|---|
| Windows | Device Manager > Ports (COM & LPT) > J-Link CDC UART |
| Linux | `/dev/ttyACM0` |
| macOS | `/dev/cu.usbmodem1101` |

{{% notice Important %}}
The deploy script `deploy_setools.py` uses the [Alif Security Toolkit (SETOOLS)](https://alifsemi.com/support/software-tools/ensemble/). Download **SETOOLS V1.110.000 or later** for your platform. After downloading, initialize the toolkit:

- **Windows**: `updateSystemPackage.exe -d`
- **Linux/macOS**: `./updateSystemPackage -d`

This updates the board firmware if needed.
{{% /notice %}}

Leave the SW4 UART selector switch in the default **SEUART** position -- the script enters SE maintenance mode automatically over UART.

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

## Verify the firmware is running

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

## What you've learned and what's next

You've now flashed the stories260K firmware to the Alif E8 DevKit using SETOOLS and verified that the board is running by reading serial output. The board is ready to accept prompts.

Next, you set up FWAuto's closed-loop workflow so that builds, deploys, and verifications run automatically.
