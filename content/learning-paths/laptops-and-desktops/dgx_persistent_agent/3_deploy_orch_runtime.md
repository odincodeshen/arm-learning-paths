---
title: Deploy Hermes Orchestration Runtime 
weight: 4
layout: "learningpathall"
---

## Deploy Hermes Orchestration Runtime

## Introduction

- Introduce Hermes as the orchestration runtime

- Explain the role of orchestration in persistent AI systems

- Introduce event-driven runtime workflows

---

## Create the Hermes Runtime Directory

- Create the Hermes project directory

- Organize runtime source files

- Prepare the orchestration runtime workspace

---

## Create the Hermes Container Image

- Create the Hermes Dockerfile

- Install runtime dependencies

- Configure the orchestration runtime environment

Dependencies include:

- watchdog

- qdrant-client

- ollama

- pypdf

- sentence-transformers

---

## Configure Runtime Logging

- Enable unbuffered Python logging

- Configure runtime observability

- Verify container log visibility

---

## Create the Hermes Runtime Service

- Create the initial orchestration runtime

- Configure filesystem monitoring

- Configure event-driven processing

---

## Configure Workspace Monitoring

- Define the workspace watch directory

- Configure filesystem watchers

- Monitor runtime events

Example:

```text

/workspace/inbox

```

---

## Add File Event Handling

- Detect newly created files

- Ignore unsupported files

- Filter hidden and temporary files

Supported examples:

- `.txt`

- `.md`

- `.log`

---

## Configure Runtime Filtering

- Ignore hidden files

- Ignore unsupported extensions

- Prevent ingestion of temporary files

---

## Update Docker Compose

- Add Hermes runtime service

- Configure shared workspace mounts

- Configure runtime networking

- Connect Hermes to Ollama and Qdrant

---

## Build the Hermes Runtime

- Build the Hermes container image

- Verify runtime dependencies

- Verify runtime startup

Commands:

- `docker compose build hermes`

- `docker compose up -d`

---

## Verify Hermes Runtime Logs

- Verify runtime startup

- Verify workspace monitoring

- Verify orchestration logs

Expected output:

```text

[Hermes Agent] Starting workspace watcher...

```

---

## Validate Event-driven Processing

- Create test files in the workspace

- Trigger filesystem events

- Verify runtime detection

Example workflow:

```text

new file

→ filesystem event

→ Hermes orchestration

```

---

## Verify Shared Workspace Access

- Verify mounted workspace visibility

- Verify container access to shared files

- Verify runtime persistence

---

## Runtime Responsibilities

### Hermes Runtime

Responsible for:

- filesystem monitoring

- orchestration

- event scheduling

- workflow coordination

- persistent runtime lifecycle

---

## CPU Orchestration Responsibilities

Introduce the role of the Arm Grace CPU in runtime orchestration:

- filesystem events

- background services

- scheduling

- orchestration coordination

- runtime lifecycle management

---

## Key Concepts

- Event-driven orchestration

- Persistent runtime services

- Filesystem-triggered AI workflows

- Runtime filtering

- Shared workspace coordination

---

## Summary

Readers should understand:

- how Hermes orchestrates persistent AI workflows

- how filesystem-triggered runtimes operate

- how event-driven orchestration works

- how runtime filtering improves orchestration stability

- how Arm CPUs coordinate persistent runtime workflows

---

## Next Steps

Proceed to:

# Add Local LLM Inference

Integrate:

- Ollama inference

- AI summarization

- GPU-accelerated local inference

- runtime orchestration with model execution