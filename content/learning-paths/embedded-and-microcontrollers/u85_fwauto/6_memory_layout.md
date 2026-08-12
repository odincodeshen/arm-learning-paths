---
title: Understand the memory layout
description: Learn how the stories260K firmware uses the Alif E8's MRAM, SRAM, ITCM, and DTCM regions to store model weights, KV cache, and application code.
weight: 6
layout: "learningpathall"
---

## Alif E8 memory architecture

The Alif E8 DevKit has several memory regions with different characteristics. Understanding how the firmware uses them is essential for fitting the stories260K model within the available space.

| Region | Address | Size | Speed | Purpose |
|--------|---------|------|-------|---------|
| MRAM | `0x80000000` | 2 MB | Slower, non-volatile | Code (execute-in-place) and read-only data (model weights, tokenizer) |
| ITCM | `0x00000000` | 256 KB | Fast | Instruction Tightly Coupled Memory -- code runs from here after copying from MRAM at boot |
| DTCM | `0x20000000` | 256 KB | Fast | Data Tightly Coupled Memory -- stack, heap, global variables, KV cache |
| SRAM0 | `0x02000000` | 4 MB | Medium | LCD and camera buffers (not used by this firmware) |
| SRAM1 | `0x02400000` | 4 MB | Medium | General-purpose SRAM (not used by this firmware) |

## How the stories260K firmware uses memory

The stories260K model has 260,000 parameters stored as 16-bit floats (FP16), which takes approximately 520 KB. The BPE tokenizer vocabulary adds another ~500 KB. Together with the application code, the total firmware binary is about 1.1 MB.

The linker script places these into memory as follows:

| What | Where | Why |
|------|-------|-----|
| Application code (`main.c`, `llm_infer.c`) | MRAM, copied to ITCM at boot | Code needs fast execution from TCM |
| Model weights (`model_data.h`, ~520 KB) | MRAM (read-only, execute-in-place) | Too large for TCM; MRAM is fast enough for weight reads |
| Tokenizer vocabulary (`tokenizer_data.h`, ~500 KB) | MRAM (read-only) | Only accessed during tokenization, not during inference |
| Stack | DTCM (8 KB) | Fast access for function calls and local variables |
| Heap | DTCM (16 KB) | Dynamic allocations during initialization |
| KV cache | DTCM | Transformer key-value cache needs fast read/write during inference |
| Global variables (.data, .bss) | DTCM | Mutable state needs fast RAM |

## Why this layout works

The critical constraint is that the **KV cache must fit in DTCM**. During inference, the transformer reads and writes the KV cache at every token generation step. If the KV cache were in MRAM, the slower access speed would significantly increase inference time.

The model weights, by contrast, are **read-only** and accessed sequentially during the forward pass. MRAM's read speed is sufficient for weight access, and its 2 MB capacity holds the entire model.

The build output confirms this fits:

```output
MRAM: 1149296 B / 2 MB (54.80%)
```

This means the firmware (code + model weights + tokenizer) occupies about 1.1 MB of the 2 MB MRAM, leaving room for larger models or additional data.

## How the linker script works

The linker script (`linker_gnu_mram.ld`) is provided by the Alif Semiconductor CMSIS pack. It defines the memory regions and sections:

```ld
MEMORY
{
  ITCM (rx)  : ORIGIN = 0x00000000, LENGTH = 256K
  DTCM (rwx) : ORIGIN = 0x20000000, LENGTH = 256K
  SRAM0 (rw) : ORIGIN = 0x02000000, LENGTH = 4M
  SRAM1 (rw) : ORIGIN = 0x02400000, LENGTH = 4M
  MRAM (rx)  : ORIGIN = 0x80000000, LENGTH = 2M
}
```

The startup code copies `.data` from MRAM to DTCM and zeros `.bss` in DTCM before `main()` runs. Code marked for ITCM is copied from MRAM to ITCM at boot. The model weights stay in MRAM because they are declared as `const` arrays.

## How model weights are embedded

The model weights in `model_data.h` are a C byte array generated from the trained PyTorch model:

```c
#include <stdint.h>
const uint8_t model_weights[] __attribute__((aligned(16))) = {
    0x3C, 0x00, 0x00, 0x00, // ... 520 KB of FP16 weights
};
```

The `const` keyword tells the compiler to place this array in the `.rodata` section, which the linker puts in MRAM. The `aligned(16)` attribute ensures the data is 16-byte aligned for efficient memory access.

Similarly, `tokenizer_data.h` contains the BPE vocabulary as a const array.

## What you've learned and what's next

You've now understood how the Alif E8's memory regions are organized and how the stories260K firmware places code in ITCM, mutable data in DTCM, and model weights in MRAM. This layout keeps the KV cache in fast SRAM for inference speed while fitting the entire model within the 2 MB MRAM.

Next, you build the firmware using CMake and the CMSIS-Toolbox.
