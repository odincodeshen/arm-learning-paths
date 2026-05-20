---
title: Deploy Hermes Orchestration Runtime 
weight: 4
layout: "learningpathall"
---

## Deploy Hermes Orchestration Runtime

In this section, you will add ***Hermes Agent*** to the runtime stack.

Hermes is the ***CPU-side orchestration runtime***. It runs continuously, watches the shared workspace, and reacts when new files are created. This is the first step toward a ***persistent local AI agent***.

In this section, Hermes does not call a language model yet. You will first build the event-driven runtime foundation:

```text
[New file]
       |
       v
[Filesystem event]
       |
       v
[Hermes watcher]
       |
       v
[Document preview]
```

Later sections add local inference, persistent memory, semantic retrieval, and autonomous cognition.

## Create the Hermes Runtime Directory

Return to the project root:

```bash
cd ~/dgx-hermes-agent
```

Create the Hermes source directory:

```bash
mkdir -p hermes
```

The project structure now includes a source directory for the orchestration runtime:

```text
dgx-hermes-agent/
|-- compose/
|-- hermes/
|-- models/
|-- qdrant/
`-- workspace/
```

## Create the Hermes Container Image

Create and edit the file `~/dgx-hermes-agent/hermes/Dockerfile`.

Add the following content:

```dockerfile
FROM python:3.11-slim

WORKDIR /app

RUN apt-get update && apt-get install -y \
    git \
    curl \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

RUN pip install --no-cache-dir \
    ollama \
    qdrant-client \
    watchdog \
    sentence-transformers \
    pypdf \
    python-dotenv

COPY . /app

CMD ["python", "-u", "agent.py"]
```

This image installs the dependencies used throughout the Learning Path. Some packages, such as `ollama` and `qdrant-client`, are used in later sections. Installing them now keeps the Hermes container image consistent as the runtime gains capabilities.

The command uses `python -u`:

```dockerfile
CMD ["python", "-u", "agent.py"]
```

The `-u` option enables unbuffered output. This is important for a persistent service because log messages appear immediately when you run:

```bash
docker logs -f hermes
```

## Create the Hermes Runtime Service

Create and edit the file `~/dgx-hermes-agent/hermes/agent.py`.

Add the following content:

```python
import os
import time
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler

WATCH_DIR = "/workspace/inbox"

class WorkspaceHandler(FileSystemEventHandler):

    def on_created(self, event):

        if event.is_directory:
            return

        print(f"\n[Agent] New file detected:")
        print(event.src_path)

        summarize_file(event.src_path)

def summarize_file(path):

    try:

        with open(path, "r") as f:
            content = f.read()

        print("\n[Agent] File content preview:")
        print(content[:500])

    except Exception as e:

        print(f"[Agent] Error reading file: {e}")

if __name__ == "__main__":

    print("\n[Hermes Agent] Starting workspace watcher...")
    print(f"[Hermes Agent] Monitoring: {WATCH_DIR}")

    observer = Observer()

    observer.schedule(
        WorkspaceHandler(),
        WATCH_DIR,
        recursive=False
    )

    observer.start()

    try:

        while True:
            time.sleep(1)

    except KeyboardInterrupt:

        observer.stop()

    observer.join()
```

This first agent performs three important orchestration tasks:

- Starts a long-running runtime process
- Watches `/workspace/inbox`
- Handles file creation events

The `summarize_file()` function does not use an LLM yet. For now, it reads and prints the first 500 characters of the file. This validates the filesystem event pipeline before adding model inference.

## Code Trace

The runtime starts by defining the watched directory:

```python
WATCH_DIR = "/workspace/inbox"
```

The event handler receives filesystem events:

```python
class WorkspaceHandler(FileSystemEventHandler):

    def on_created(self, event):
```

Directory events are ignored:

```python
if event.is_directory:
    return
```

New file events are passed into the document processing function:

```python
summarize_file(event.src_path)
```

The observer keeps the runtime active:

```python
while True:
    time.sleep(1)
```

This is the core pattern for persistent AI orchestration. The CPU keeps the service running, watches for events, and triggers work when the runtime state changes.

## Update Docker Compose

Open and edit the file `~/dgx-hermes-agent/compose/docker-compose.yml`.

Add the Hermes service under `services:`:

```yaml
  hermes:
    build:
      context: ../hermes

    container_name: hermes

    volumes:
      - ../workspace:/workspace

    environment:
      - OLLAMA_HOST=http://ollama:11434
      - QDRANT_HOST=qdrant

    depends_on:
      - ollama
      - qdrant

    restart: unless-stopped
```

The final service structure should be:

```text
services:
  ollama:
  qdrant:
  open-webui:
  hermes:
```

Hermes mounts the same shared workspace as the other services. It also receives environment variables for Ollama and Qdrant, which are used in later sections.

## Build the Hermes Runtime

Build the Hermes container:

```bash
cd ~/dgx-hermes-agent/compose
docker compose build hermes
```

The first build installs the Python dependencies listed in the Dockerfile.

Start the stack:

```bash
docker compose up -d
```

Verify that the Hermes container is running:

```bash
docker ps
```

You should see:

```text
hermes
```

alongside the existing runtime services:

```text
ollama
qdrant
open-webui
```

## Verify Hermes Runtime Logs

Follow the Hermes logs:

```bash
docker logs -f hermes
```

Expected output:

```text
[Hermes Agent] Starting workspace watcher...
[Hermes Agent] Monitoring: /workspace/inbox
```

This confirms that Hermes started and is watching the shared inbox directory.

## Validate Event-driven Processing

Open a second terminal on the host and create a new test file:

```bash
echo "Hermes watches the workspace and reacts to new files." \
> ~/dgx-hermes-agent/workspace/inbox/runtime-test.txt
```

Return to the terminal that is following Hermes logs. You should see output similar to:

```text
[Agent] New file detected:
/workspace/inbox/runtime-test.txt

[Agent] File content preview:
Hermes watches the workspace and reacts to new files.
```

This validates the event-driven pipeline:

```text
[New file]
       |
       v
[Filesystem event]
       |
       v
[Hermes orchestration]
       |
       v
[File processing]
```

## Verify Shared Workspace Access

Hermes sees the host file path through the mounted container path:

| Host path | Container path |
|---|---|
| `~/dgx-hermes-agent/workspace/inbox` | `/workspace/inbox` |

This shared mount is what allows the host, Hermes, Ollama, and later memory workflows to operate on the same persistent runtime state.

## Runtime Responsibilities

Hermes is now responsible for:

- Filesystem monitoring
- Runtime lifecycle management
- Event handling
- File reading
- Workflow triggering

At this stage, Hermes is not performing inference, generating embeddings, or storing vectors. Those capabilities are added incrementally so you can validate each layer of the runtime.

## CPU Orchestration Responsibilities

This section demonstrates the CPU-side work required by persistent AI systems.

The Arm CPU is coordinating:

- A long-running service process
- Filesystem event monitoring
- Runtime scheduling
- File processing
- Containerized service lifecycle

This is the foundation for the rest of the Learning Path. The GPU becomes important when model inference is added, but the persistent runtime itself is coordinated by CPU-side orchestration.

## Summary

You added ***Hermes Agent*** to the DGX Spark runtime stack as a persistent Python service. The runtime now has a Hermes container, a filesystem watcher, and a Docker Compose service that mounts the shared workspace.

You also verified that creating a new file in `workspace/inbox/` triggers Hermes logs, which confirms that the ***event-driven orchestration*** path is working.

Next, you will connect Hermes to Ollama for local LLM summarization.
