---
title: Add Local LLM Inference
weight: 5
layout: "learningpathall"
---

## Add Local LLM Inference
## Introduction

- Introduce local LLM inference in persistent AI runtimes

- Explain how GPUs accelerate model inference

- Explain how orchestration runtimes coordinate inference workflows

---

## Configure Ollama Runtime Access

- Connect Hermes to the Ollama runtime

- Configure local inference endpoints

- Verify container networking between Hermes and Ollama

---

## Pull Local Language Models

- Pull a local instruction-tuned model

- Verify model availability inside Ollama

Example models:

- `qwen2.5:7b`

- `gemma`

- `llama3`

---

## Verify Local Inference

- Run local inference directly from Ollama

- Verify model execution

- Verify GPU usage during inference

---

## Add Inference Support to Hermes

- Import the Ollama Python SDK

- Configure the Ollama client

- Create reusable inference functions

---

## Implement AI Summarization

- Add document summarization workflows

- Send workspace content to the local LLM

- Generate concise AI summaries

Example workflow:

```text

new document

→ local inference

→ AI summary

```

---

## Configure Summarization Prompts

- Add system prompts

- Define summarization behavior

- Limit context size for runtime efficiency

---

## Add Runtime Logging

- Log inference execution

- Log summarization results

- Track orchestration workflow progress

Example logs:

```text

[Agent] Running summarization inference...

```

---

## Verify GPU-accelerated Inference

- Open the Ollama container

- Monitor GPU activity

- Verify inference acceleration using `nvidia-smi`

---

## Runtime Responsibilities

### Hermes Runtime

Responsible for:

- orchestration

- prompt preparation

- runtime coordination

- inference scheduling

---

### Ollama Runtime

Responsible for:

- token generation

- model execution

- summarization inference

---

## CPU and GPU Responsibilities

### Arm Grace CPU

- orchestration

- workflow coordination

- runtime scheduling

- document preprocessing

---

### Blackwell GPU

- local LLM inference

- token generation

- AI summarization

---

## Event-driven Inference Workflow

Example runtime pipeline:

```text

filesystem event

→ document parsing

→ inference request

→ local LLM execution

→ AI summary

```

---

## Key Concepts

- Local LLM inference

- GPU-accelerated inference

- Runtime orchestration

- Event-driven summarization

- CPU/GPU workload coordination

---

## Summary

Readers should understand:

- how local LLM inference integrates into persistent runtimes

- how Hermes orchestrates inference workflows

- how Ollama executes GPU-accelerated inference

- how Arm CPUs coordinate runtime scheduling and preprocessing

- how local AI summarization operates continuously on DGX Spark

---

## Next Steps

Proceed to:

# Build Persistent Semantic Memory

Add:

- embeddings generation

- vector storage

- semantic memory pipelines

- persistent contextual memory