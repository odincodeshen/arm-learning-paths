---
title: Build the DGX Spark AI Runtime Foundation
weight: 3
layout: "learningpathall"
---

## Build the DGX Spark AI Runtime Foundation
## Verify the DGX Spark Environment

- Verify Arm architecture

- Verify GPU visibility

- Verify CUDA runtime

- Verify NVIDIA drivers

Commands:

- `uname -m`

- `nvidia-smi`

- `lsb_release -a`

---

## Install Docker

- Install Docker Engine

- Install Docker Compose

- Configure Docker permissions

---

## Install NVIDIA Container Toolkit

- Configure GPU-enabled containers

- Configure Docker GPU runtime

- Verify GPU container access

---

## Configure Docker Runtime Settings

- Configure Docker DNS

- Configure NVIDIA runtime

- Restart Docker daemon

---

## Verify GPU-enabled Containers

- Run CUDA validation container

- Verify GPU passthrough inside containers

---

## Create the Persistent Workspace

- Create project directory structure

- Create shared runtime workspace

- Create persistent storage directories

Example structure:

```text

workspace/

├── inbox/

├── memory/

├── logs/

├── processed/

└── config/

```

---

## Build the Runtime Service Stack

Introduce the runtime services used throughout the Learning Path:

- Ollama

- Qdrant

- Open WebUI

---

## Create the Docker Compose Stack

- Create `docker-compose.yml`

- Configure container networking

- Configure persistent volumes

- Configure GPU runtime access

---

## Runtime Service Roles

### Ollama

- Local inference runtime

- LLM execution

- Embeddings generation

---

### Qdrant

- Vector database

- Semantic memory storage

---

### Open WebUI

- Local browser interface

- Runtime interaction layer

---

## Start the Runtime Stack

- Start containers using Docker Compose

- Verify running services

- Verify container networking

Commands:

- `docker compose up -d`

- `docker ps`

---

## Pull Local Models

- Pull Qwen model

- Pull embedding model

Example:

- `qwen2.5:7b`

- `nomic-embed-text`

---

## Verify Local Inference

- Run local LLM inference

- Verify Ollama runtime

- Verify GPU usage during inference

---

## Verify Shared Workspace

- Verify container volume mounts

- Verify shared runtime storage

- Verify workspace visibility inside containers

---

## Verify Qdrant

- Open Qdrant dashboard

- Verify vector database availability

---

## Key Concepts

- Containerized AI runtimes

- GPU-enabled containers

- Persistent runtime services

- Shared workspace orchestration

- Local AI runtime infrastructure

---

## Summary

Readers should understand:

- how to deploy the local AI runtime stack

- how runtime services interact

- how GPU-enabled containers operate

- how persistent runtime storage is configured

- how DGX Spark supports heterogeneous AI workloads