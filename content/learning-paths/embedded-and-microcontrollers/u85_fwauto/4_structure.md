---
title: Understand the project structure
description: Learn the key directories and source files in the Alif SLM repository before building and deploying the firmware.
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

## What you've learned and what's next

You've now identified the two firmware implementations in the repository, the interactive firmware you'll use, and the key source files that make up the stories260K runner.

Next, you build the firmware and flash it to the Alif E8 DevKit.
