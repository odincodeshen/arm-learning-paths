---
title: Add Autonomous Workspace Cognition
weight: 8
layout: "learningpathall"
---

## Add Autonomous Workspace Cognition

In this section, you will add ***autonomous workspace cognition*** to Hermes Agent.

The runtime can already ingest documents, build semantic memory, and answer questions using retrieved context. You will now add a ***periodic cognition workflow*** that reviews stored memory and generates a workspace-level summary.

The workflow becomes:

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
[Workspace summary]
```

This is the final stage of the Learning Path. Hermes becomes a persistent autonomous local AI runtime that can monitor, remember, retrieve, and periodically reason about workspace state.

## Autonomous Cognition Overview

Autonomous cognition means the runtime performs useful reasoning without waiting for a new document or explicit query.

Hermes will:

- Load runtime policy from `/workspace/config/runtime.json`
- Continue watching `workspace/inbox/`
- Continue ingesting supported files into semantic memory
- Continue answering questions from `/workspace/query.txt`
- Periodically summarize the stored workspace memory
- Write the summary to `/workspace/memory/workspace-summary.txt`

The runtime remains local-first. Files, models, vector memory, and summaries stay on the DGX Spark system.

## Create the Runtime Config Directory

Create the configuration directory if it does not already exist:

```bash
mkdir -p ~/dgx-hermes-agent/workspace/config
```

Create and edit the file `~/dgx-hermes-agent/workspace/config/runtime.json`.

Add the following content:

```json
{
  "summary_interval_hours": 8,
  "supported_extensions": [
    ".txt",
    ".md",
    ".log"
  ],
  "retrieval_limit": 3,
  "summary_output": "/workspace/memory/workspace-summary.txt"
}
```

The policy file controls runtime behavior without rebuilding the container.

| Policy | Purpose |
|---|---|
| `summary_interval_hours` | Controls how often Hermes generates a workspace summary |
| `supported_extensions` | Controls which file types Hermes ingests |
| `retrieval_limit` | Records the intended semantic retrieval depth for the runtime policy |
| `summary_output` | Defines where Hermes writes the workspace summary |

The verified code in this section keeps semantic retrieval at `limit=3`, matching the policy value shown above. The policy file makes this setting visible for later hardening, where the retrieval function can load it dynamically.

## Add Autonomous Cognition to Hermes

Open and edit the file `~/dgx-hermes-agent/hermes/agent.py`.

Replace the file with the following version:

```python
import os
import json
import uuid
import time
import ollama

from datetime import datetime

from qdrant_client import QdrantClient
from qdrant_client.models import (
    Distance,
    VectorParams,
    PointStruct
)

from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler

WATCH_DIR = "/workspace/inbox"

CONFIG_PATH = "/workspace/config/runtime.json"

OLLAMA_HOST = os.getenv(
    "OLLAMA_HOST",
    "http://ollama:11434"
)

QDRANT_HOST = os.getenv(
    "QDRANT_HOST",
    "qdrant"
)

COLLECTION_NAME = "workspace_memory"

client = ollama.Client(host=OLLAMA_HOST)

qdrant = QdrantClient(
    host=QDRANT_HOST,
    port=6333
)

def ensure_collection():

    collections = qdrant.get_collections().collections
    names = [c.name for c in collections]

    if COLLECTION_NAME not in names:

        qdrant.create_collection(
            collection_name=COLLECTION_NAME,
            vectors_config=VectorParams(
                size=768,
                distance=Distance.COSINE
            )
        )

        print(f"[Memory] Created collection: {COLLECTION_NAME}")

def load_runtime_config():

    with open(CONFIG_PATH, "r") as f:

        return json.load(f)

class WorkspaceHandler(FileSystemEventHandler):

    def on_created(self, event):

        if event.is_directory:
            return

        filename = os.path.basename(event.src_path)

        # Ignore hidden files
        if filename.startswith("."):
            return

        ext = os.path.splitext(filename)[1]

        config = load_runtime_config()

        supported_extensions = config.get(
            "supported_extensions",
            [".txt"]
        )

        if ext not in supported_extensions:
            return

        print(f"\n[Agent] New file detected:")
        print(event.src_path)

        process_file(event.src_path)

def generate_summary(content):

    response = client.chat(
        model="qwen2.5:7b",
        messages=[
            {
                "role": "system",
                "content": (
                    "You are a local AI workspace assistant. "
                    "Summarize the document in 3 concise bullet points."
                )
            },
            {
                "role": "user",
                "content": content[:4000]
            }
        ]
    )

    return response["message"]["content"]

def generate_embedding(content):

    response = client.embed(
        model="nomic-embed-text",
        input=content[:4000]
    )

    return response["embeddings"][0]

def store_memory(path, content, summary, embedding):

    point_id = str(uuid.uuid4())

    qdrant.upsert(
        collection_name=COLLECTION_NAME,
        points=[
            PointStruct(
                id=point_id,
                vector=embedding,
                payload={
                    "path": path,
                    "summary": summary,
                    "content": content[:4000]
                }
            )
        ]
    )

    print(f"[Memory] Stored document: {path}")

def search_memory(query):

    print("\n[Memory] Searching semantic memory...")

    embedding = generate_embedding(query)

    results = qdrant.query_points(
        collection_name=COLLECTION_NAME,
        query=embedding,
        limit=3
    ).points

    memories = []

    for result in results:

        payload = result.payload

        memories.append({
            "path": payload.get("path"),
            "summary": payload.get("summary")
        })

    return memories

def query_workspace(question):

    memories = search_memory(question)

    context = "\n\n".join([
        f"Document: {m['path']}\nSummary:\n{m['summary']}"
        for m in memories
    ])

    response = client.chat(
        model="qwen2.5:7b",
        messages=[
            {
                "role": "system",
                "content": (
                    "You are a persistent AI workspace assistant. "
                    "Answer questions using the retrieved workspace memory."
                )
            },
            {
                "role": "user",
                "content": (
                    f"Question:\n{question}\n\n"
                    f"Relevant workspace memory:\n{context}"
                )
            }
        ]
    )

    answer = response["message"]["content"]

    print("\n[Workspace Query]")
    print(question)

    print("\n[Retrieved Memories]")
    print(context)

    print("\n[AI Response]")
    print(answer)

def generate_workspace_summary():

    print("\n[Cognition] Generating workspace summary...")

    results = qdrant.scroll(
        collection_name=COLLECTION_NAME,
        limit=10,
        with_payload=True
    )[0]

    summaries = []

    for result in results:

        payload = result.payload

        summaries.append(
            payload.get("summary", "")
        )

    combined = "\n\n".join(summaries)

    response = client.chat(
        model="qwen2.5:7b",
        messages=[
            {
                "role": "system",
                "content": (
                    "You are an autonomous workspace cognition agent. "
                    "Analyze the workspace summaries and identify "
                    "important recurring themes and insights."
                )
            },
            {
                "role": "user",
                "content": combined[:6000]
            }
        ]
    )

    workspace_summary = response["message"]["content"]

    config = load_runtime_config()

    output_path = config.get(
        "summary_output",
        "/workspace/memory/workspace-summary.txt"
    )

    with open(output_path, "w") as f:

        f.write(
            f"Workspace Summary\n"
            f"Generated: {datetime.now()}\n\n"
        )

        f.write(workspace_summary)

    print("\n[Cognition] Workspace summary updated:")
    print(output_path)

def process_file(path):

    try:

        with open(path, "r", encoding="utf-8") as f:
            content = f.read()

        print("\n[Agent] Running summarization inference...")

        summary = generate_summary(content)

        print("\n[Agent] AI Summary:")
        print(summary)

        print("\n[Agent] Generating embeddings...")

        embedding = generate_embedding(content)

        store_memory(
            path,
            content,
            summary,
            embedding
        )

    except Exception as e:

        print(f"[Agent] Error: {e}")

if __name__ == "__main__":

    print("\n[Hermes Agent] Starting workspace watcher...")
    print(f"[Hermes Agent] Monitoring: {WATCH_DIR}")

    ensure_collection()

    observer = Observer()

    observer.schedule(
        WorkspaceHandler(),
        WATCH_DIR,
        recursive=False
    )

    observer.start()

    last_summary_time = 0

    try:

        while True:

            time.sleep(5)

            config = load_runtime_config()

            summary_interval_hours = config.get(
                "summary_interval_hours",
                8
            )

            interval_seconds = (
                summary_interval_hours * 3600
            )

            current_time = time.time()

            # Periodic autonomous cognition
            if (
                current_time - last_summary_time
                > interval_seconds
            ):

                generate_workspace_summary()

                last_summary_time = current_time

            # Interactive semantic retrieval
            if os.path.exists("/workspace/query.txt"):

                with open("/workspace/query.txt", "r") as f:
                    question = f.read().strip()

                os.remove("/workspace/query.txt")

                query_workspace(question)

    except KeyboardInterrupt:

        observer.stop()

    observer.join()
```

## Code Trace

This version adds JSON configuration loading:

```python
CONFIG_PATH = "/workspace/config/runtime.json"
```

```python
def load_runtime_config():

    with open(CONFIG_PATH, "r") as f:

        return json.load(f)
```

File filtering now comes from runtime policy:

```python
config = load_runtime_config()

supported_extensions = config.get(
    "supported_extensions",
    [".txt"]
)
```

The cognition function reads stored memory from Qdrant:

```python
results = qdrant.scroll(
    collection_name=COLLECTION_NAME,
    limit=10,
    with_payload=True
)[0]
```

It extracts stored summaries:

```python
summaries.append(
    payload.get("summary", "")
)
```

It asks the local model to analyze recurring themes:

```python
"You are an autonomous workspace cognition agent. "
"Analyze the workspace summaries and identify "
"important recurring themes and insights."
```

It writes the result to the configured summary output path:

```python
output_path = config.get(
    "summary_output",
    "/workspace/memory/workspace-summary.txt"
)
```

The main loop reloads runtime policy every cycle:

```python
config = load_runtime_config()
```

This allows changes to `runtime.json` to affect runtime behavior without rebuilding the container.

## Rebuild Hermes

Rebuild the Hermes container:

```bash
cd ~/dgx-hermes-agent/compose
docker compose build hermes
```

Restart the runtime:

```bash
docker compose up -d
```

Follow the logs:

```bash
docker logs -f hermes
```

On startup, the first cognition cycle runs immediately because `last_summary_time` starts at `0`.

Expected output:

```text
[Cognition] Generating workspace summary...
[Cognition] Workspace summary updated:
/workspace/memory/workspace-summary.txt
```

This startup behavior is expected and validates that the cognition pipeline can read memory, call Ollama, and write the summary file.

## Verify Workspace Summary Output

View the generated summary on the host:

```bash
cat ~/dgx-hermes-agent/workspace/memory/workspace-summary.txt
```

Expected structure:

```text
Workspace Summary
Generated: 2026-...

...
```

The summary content will vary because it is generated by the local model from stored memory.

## Validate Real-time Ingestion

Create a new file:

```bash
cat > ~/dgx-hermes-agent/workspace/inbox/autonomous-runtime-note.txt <<'EOF'
Autonomous workspace cognition allows a persistent AI runtime to analyze
stored memory on a schedule. This helps the system identify recurring
themes, summarize activity, and maintain awareness of workspace state.
EOF
```

Expected Hermes logs:

```text
[Agent] New file detected:
/workspace/inbox/autonomous-runtime-note.txt

[Agent] Running summarization inference...
[Agent] Generating embeddings...
[Memory] Stored document:
```

The new document is added to semantic memory and can be included in future workspace summaries.

## Validate Semantic Retrieval Still Works

Create a query:

```bash
echo "What is autonomous workspace cognition?" \
> ~/dgx-hermes-agent/workspace/query.txt
```

Expected logs:

```text
[Memory] Searching semantic memory...

[Workspace Query]
What is autonomous workspace cognition?

[Retrieved Memories]

[AI Response]
```

This confirms that autonomous cognition was added without removing the query workflow from the previous section.

## Validate Runtime Policy Reload

Open and edit the file `~/dgx-hermes-agent/workspace/config/runtime.json`.

Change the supported extensions so Hermes only ingests Markdown files:

```json
{
  "summary_interval_hours": 8,
  "supported_extensions": [
    ".md"
  ],
  "retrieval_limit": 3,
  "summary_output": "/workspace/memory/workspace-summary.txt"
}
```

Wait 5 to 10 seconds for the runtime loop to reload the policy.

Create a `.txt` file:

```bash
echo "This text file should be ignored by the current policy." \
> ~/dgx-hermes-agent/workspace/inbox/ignored-policy-test.txt
```

Hermes should not ingest it because `.txt` is no longer in `supported_extensions`.

Now create a Markdown file:

```bash
cat > ~/dgx-hermes-agent/workspace/inbox/accepted-policy-test.md <<'EOF'
# Policy Test

This Markdown file should be ingested because the runtime policy allows
files with the .md extension.
EOF
```

Expected logs:

```text
[Agent] New file detected:
/workspace/inbox/accepted-policy-test.md
```

This validates that Hermes reloads runtime configuration dynamically.

Restore the original policy when you are done:

```json
{
  "summary_interval_hours": 8,
  "supported_extensions": [
    ".txt",
    ".md",
    ".log"
  ],
  "retrieval_limit": 3,
  "summary_output": "/workspace/memory/workspace-summary.txt"
}
```

## Trigger a Faster Cognition Cycle

For validation, you can temporarily reduce the summary interval.

Open and edit the file `~/dgx-hermes-agent/workspace/config/runtime.json`.

Set a very small interval:

```json
{
  "summary_interval_hours": 0.001,
  "supported_extensions": [
    ".txt",
    ".md",
    ".log"
  ],
  "retrieval_limit": 3,
  "summary_output": "/workspace/memory/workspace-summary.txt"
}
```

This is approximately 3.6 seconds.

Follow the logs:

```bash
docker logs -f hermes
```

Expected output:

```text
[Cognition] Generating workspace summary...
[Cognition] Workspace summary updated:
```

Restore the interval to `8` after validation to avoid continuous summary generation:

```json
{
  "summary_interval_hours": 8,
  "supported_extensions": [
    ".txt",
    ".md",
    ".log"
  ],
  "retrieval_limit": 3,
  "summary_output": "/workspace/memory/workspace-summary.txt"
}
```

## Validate Persistent Runtime Lifecycle

Restart the stack:

```bash
cd ~/dgx-hermes-agent/compose
docker compose restart hermes
```

Follow the logs:

```bash
docker logs -f hermes
```

Expected output:

```text
[Hermes Agent] Starting workspace watcher...
[Hermes Agent] Monitoring: /workspace/inbox
```

The `workspace_memory` collection remains in Qdrant because the Qdrant storage directory is persisted on the host.

Verify that the summary file still exists:

```bash
ls ~/dgx-hermes-agent/workspace/memory/
```

You should see:

```text
workspace-summary.txt
```

This confirms that the runtime state persists across container restarts.

## Runtime Validation Summary

At this point, the local runtime supports:

| Capability | Status |
|---|---|
| Workspace monitoring | Complete |
| Local summarization | Complete |
| Embedding generation | Complete |
| Persistent vector memory | Complete |
| Semantic retrieval | Complete |
| Contextual reasoning | Complete |
| Autonomous workspace cognition | Complete |
| Dynamic runtime policy reload | Complete |

## CPU and GPU Responsibilities

The Arm Grace CPU coordinates the autonomous runtime:

- Filesystem monitoring
- Runtime policy loading
- Dynamic configuration reload
- Background scheduling
- Semantic memory aggregation
- Query workflow coordination
- Workspace summary lifecycle

The Blackwell GPU accelerates:

- Summarization
- Embedding generation
- Contextual reasoning
- Autonomous workspace analysis

The result is a heterogeneous local AI system where the CPU coordinates persistent workflows and the GPU accelerates model execution.

## Runtime Compatibility Notes

This final runtime uses the current Ollama embedding API:

```python
client.embed(...)
```

and reads embeddings from:

```python
response["embeddings"][0]
```

Semantic retrieval uses the current Qdrant client API:

```python
qdrant.query_points(
    collection_name=COLLECTION_NAME,
    query=embedding,
    limit=3
).points
```

Qdrant result payloads are read from:

```python
payload = result.payload
```

The current implementation primarily handles file creation events through:

```python
on_created()
```

For validation, use new filenames. Existing files or file modifications may not trigger ingestion. File modification handling is a natural next improvement for a hardened runtime.

## Summary

You completed the ***persistent autonomous local AI runtime***. Hermes now combines event-driven workspace ingestion, local summarization and embedding generation, persistent semantic memory, contextual retrieval, autonomous workspace summaries, and dynamic runtime policy.

This Learning Path demonstrates that persistent AI systems are ***distributed orchestration systems***, not just single inference calls. You have built a local-first AI runtime on DGX Spark using Arm CPU orchestration, GPU-accelerated inference, semantic memory, and autonomous workspace cognition.
