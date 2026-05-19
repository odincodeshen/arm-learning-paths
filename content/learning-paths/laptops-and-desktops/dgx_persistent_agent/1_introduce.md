---
title: Understand Persistent AI Runtime Architecture
weight: 2

### FIXED, DO NOT MODIFY
layout: learningpathall
---

## Understand Persistent AI Runtime Architecture

In this Learning Path, you will build a persistent local AI runtime on NVIDIA DGX Spark.

The final system is not a single chatbot process. It is a set of local services that run continuously, share a workspace, react to file events, generate summaries, create embeddings, store vector memory, retrieve context, and periodically reason about the state of the workspace.

The core idea is:

```text
AI systems are orchestration systems, not just inference systems.
```

DGX Spark is well suited to this type of workload because it combines Arm CPU orchestration with local GPU acceleration. The Arm Grace CPU coordinates background services, filesystem events, scheduling, document processing, metadata handling, and service-to-service communication. The Blackwell GPU accelerates local LLM inference, token generation, summarization, and embedding generation.

By the end of this Learning Path, you will have a local runtime with these capabilities:

| Capability | Runtime component |
|---|---|
| Local LLM inference | Ollama |
| Persistent vector memory | Qdrant |
| Workspace orchestration | Hermes Agent |
| Browser-based interaction | Open WebUI |
| Semantic retrieval | Hermes Agent + Qdrant + Ollama |
| Autonomous workspace cognition | Hermes Agent + Ollama |

## Runtime Architecture Overview

The runtime uses four containerized services:

- Hermes Agent
- Ollama
- Qdrant
- Open WebUI

These services communicate over a local Docker network and share a persistent workspace on the host.

```text
                 +------------------+
                 |   Open WebUI     |
                 |  User Interface  |
                 +--------+---------+
                          |
                          | HTTP API
                          v

+------------------------------------------------+
|                                                |
|              Ollama Container                  |
|                                                |
|      Local LLM Inference Runtime               |
|                                                |
+------------------------------------------------+
                          ^
                          |
                          | inference and embedding requests
                          |
+------------------------------------------------+
|                                                |
|              Hermes Container                  |
|                                                |
|    CPU-side Agent Orchestration Runtime        |
|                                                |
+------------------------------------------------+
                          |
                          | vector storage and retrieval
                          v

+------------------------------------------------+
|                                                |
|              Qdrant Container                  |
|                                                |
|        Persistent Vector Memory Service        |
|                                                |
+------------------------------------------------+
```

The important architectural pattern is separation of responsibilities. Each service has a narrow role, and Hermes coordinates the overall workflow.

| Layer | Service | Purpose |
|---|---|---|
| Interaction layer | Open WebUI | Provides browser-based access to local models |
| Inference layer | Ollama | Runs local language and embedding models |
| Memory layer | Qdrant | Stores and searches vector memory |
| Orchestration layer | Hermes Agent | Watches files, schedules work, coordinates services |

## Runtime Components

### Hermes Runtime

Hermes is the orchestration runtime you will build in this Learning Path.

It runs as a persistent Python service inside a container. It watches the shared workspace, detects new files, reads documents, sends requests to Ollama, stores memory in Qdrant, performs semantic retrieval, and later generates autonomous workspace summaries.

Hermes is responsible for:

- Filesystem monitoring
- Workflow orchestration
- Runtime scheduling
- Document parsing
- Prompt preparation
- Inference coordination
- Memory coordination
- Autonomous cognition

Hermes does not run the language model itself. Instead, it coordinates AI workflows across local services.

This is the main CPU-side workload in the system. The Arm CPU keeps the runtime alive, schedules background loops, tracks file events, moves data between services, and manages runtime state.

### Ollama Runtime

Ollama provides the local inference runtime.

In this Learning Path, Hermes uses Ollama for two types of model calls:

- Chat completion, using `qwen2.5:7b`
- Embedding generation, using `nomic-embed-text`

The chat model is used to summarize files, answer questions over retrieved memory, and generate workspace-level insights. The embedding model converts text into vectors so Qdrant can store and search semantic memory.

Ollama is responsible for:

- Local LLM inference
- Token generation
- AI summarization
- Embedding generation

Ollama does not watch files, manage memory, or decide when work should happen. It provides model execution, and Hermes calls it when the workflow requires inference.

### Qdrant Memory Service

Qdrant provides persistent vector memory.

Hermes stores document embeddings in a Qdrant collection named `workspace_memory`. Each stored point includes a vector and payload metadata, such as the document path, generated summary, and source content excerpt.

Qdrant is responsible for:

- Vector storage
- Semantic indexing
- Similarity search
- Long-term memory persistence
- Contextual retrieval

Qdrant does not perform LLM inference. It stores vectors and returns semantically similar memories when Hermes performs a retrieval query.

### Open WebUI

Open WebUI provides a local browser interface for interacting with the Ollama runtime.

It is useful for validating that local models are available, testing prompts, and giving users a simple interface to local inference. In this Learning Path, Open WebUI is not the orchestration layer and it is not the memory system.

Open WebUI is responsible for:

- Browser-based access
- Local chat interaction
- Model testing and exploration

The persistent AI runtime is still coordinated by Hermes.

## Shared Workspace

The services use a shared workspace mounted into the containers.

The workspace structure is:

```text
workspace/
|-- inbox/
|-- memory/
|-- logs/
|-- processed/
`-- config/
```

Each directory has a specific purpose:

| Directory | Purpose |
|---|---|
| `workspace/inbox/` | Input files monitored by Hermes |
| `workspace/memory/` | Generated memory artifacts and workspace summaries |
| `workspace/logs/` | Runtime logs and diagnostics |
| `workspace/processed/` | Optional location for processed files |
| `workspace/config/` | Runtime policy configuration |

The shared workspace is what turns isolated containers into a coordinated local AI runtime. Hermes can observe files created on the host, use Ollama to process them, store memory in Qdrant, and write results back to persistent storage.

## Event-driven AI Workflows

Persistent AI systems are long-running systems. They do not wait for a single prompt and then exit. They monitor runtime state and react when something changes.

In this Learning Path, Hermes starts with a filesystem watcher:

```text
[New document]
       |
       v
[Filesystem event]
       |
       v
[Hermes orchestration]
       |
       v
[Document processing]
```

As you add capabilities, the workflow grows:

```text
[New document]
       |
       v
[CPU filesystem watcher]
       |
       v
[Document parsing]
       |
       v
[GPU summarization]
       |
       v
[GPU embedding generation]
       |
       v
[Qdrant vector storage]
       |
       v
[Persistent semantic memory]
```

This event-driven design is important because it shows how AI systems become continuous local runtimes. The model is only one part of the system. The surrounding runtime decides when to call the model, what context to provide, where to store results, and how later workflows can reuse those results.

## Semantic Memory and Retrieval

Semantic memory gives the runtime a way to retain information over time.

The memory pipeline is:

```text
[Document]
       |
       v
[Summary]
       |
       v
[Embedding]
       |
       v
[Vector storage]
       |
       v
[Semantic memory]
```

Later, Hermes can use a question to search memory:

```text
[Question]
       |
       v
[Query embedding]
       |
       v
[Qdrant similarity search]
       |
       v
[Retrieved memory]
       |
       v
[Contextual LLM response]
```

This is different from storing plain text files and searching for keywords. Vector search allows the runtime to retrieve content based on semantic similarity. For example, a question about "CPU scheduling" can retrieve a document that discusses "runtime orchestration" even if the exact words are different.

## Autonomous Workspace Cognition

The final stage of this Learning Path adds autonomous workspace cognition.

Instead of responding only when a new file appears or when a query is submitted, Hermes periodically reviews the accumulated semantic memory and generates a workspace-level summary.

The cognition workflow is:

```text
[Persistent semantic memory]
       |
       v
[Runtime scheduling]
       |
       v
[Workspace cognition]
       |
       v
[Autonomous analysis]
       |
       v
[Workspace insights]
```

Runtime behavior is controlled by a configuration file:

```text
/workspace/config/runtime.json
```

This allows the runtime to adjust settings such as supported file extensions, retrieval depth, summary interval, and summary output path without hardcoding every behavior into the agent.

## CPU and GPU Responsibilities

This Learning Path highlights heterogeneous AI computing. The CPU and GPU both matter, but they perform different roles.

The Arm Grace CPU coordinates persistent runtime work:

- Filesystem monitoring
- Event scheduling
- Runtime orchestration
- Background service coordination
- Document parsing
- Metadata management
- Vector database coordination
- Runtime policy loading
- Long-running process lifecycle management

The Blackwell GPU accelerates model execution:

- Local LLM inference
- Token generation
- AI summarization
- Embedding generation
- Contextual reasoning
- Workspace summary generation

This separation is central to the architecture. The GPU accelerates model-heavy operations, while the CPU keeps the distributed AI runtime organized and continuously operating.

## Next Step

Next, you will build the DGX Spark runtime foundation: Docker, GPU-enabled containers, the shared workspace, and the initial Ollama, Qdrant, and Open WebUI services.
