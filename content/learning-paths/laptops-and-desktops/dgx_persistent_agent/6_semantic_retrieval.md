---
title: Add Semantic Retrieval and Contextual Reasoning
weight: 7
layout: "learningpathall"
---

## Add Semantic Retrieval and Contextual Reasoning

In this section, you will add semantic retrieval to Hermes Agent.

The runtime can already ingest documents, summarize them, generate embeddings, and store semantic memory in Qdrant. You will now add a query workflow so Hermes can search memory and use retrieved context to answer questions.

The workflow becomes:

```text
[Question]
       |
       v
[Embedding generation]
       |
       v
[Semantic retrieval]
       |
       v
[Context assembly]
       |
       v
[Local LLM reasoning]
       |
       v
[Contextual response]
```

This is the first stage where Hermes uses memory as reasoning context instead of only storing it.

## Contextual Retrieval Architecture

The retrieval pipeline uses all three runtime services:

| Component | Responsibility |
|---|---|
| Hermes Agent | Detects queries, generates query embeddings, assembles context |
| Ollama | Generates embeddings and contextual answers |
| Qdrant | Searches stored semantic memory |

The runtime keeps the same fixed memory configuration:

| Component | Value |
|---|---|
| Embedding model | `nomic-embed-text` |
| Vector dimension | `768` |
| Qdrant collection | `workspace_memory` |
| Retrieval limit | `3` |

Hermes will watch for a query file at:

```text
/workspace/query.txt
```

When the file exists, Hermes reads the question, deletes the query file, searches memory, and prints the answer in the container logs.

## Add Retrieval Functions to Hermes

Open the Hermes agent:

```bash
nano ~/dgx-hermes-agent/hermes/agent.py
```

Replace the file with the following version:

```python
import os
import uuid
import time
import ollama

from qdrant_client import QdrantClient
from qdrant_client.models import (
    Distance,
    VectorParams,
    PointStruct
)

from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler

WATCH_DIR = "/workspace/inbox"

SUPPORTED_EXTENSIONS = [
    ".txt",
    ".md",
    ".log"
]

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

class WorkspaceHandler(FileSystemEventHandler):

    def on_created(self, event):

        if event.is_directory:
            return

        filename = os.path.basename(event.src_path)

        if filename.startswith("."):
            return

        ext = os.path.splitext(filename)[1]

        if ext not in SUPPORTED_EXTENSIONS:
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

    try:

        while True:

            time.sleep(1)

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

The `search_memory()` function converts the user question into an embedding:

```python
embedding = generate_embedding(query)
```

It searches Qdrant using the current Qdrant Python client API:

```python
results = qdrant.query_points(
    collection_name=COLLECTION_NAME,
    query=embedding,
    limit=3
).points
```

The current API uses `query=embedding`. Do not use older examples that pass `query_vector=embedding`.

Qdrant returns scored point objects. Hermes reads the payload from each result:

```python
payload = result.payload
```

Only the path and summary are assembled into the retrieval context:

```python
memories.append({
    "path": payload.get("path"),
    "summary": payload.get("summary")
})
```

The context is converted into a prompt:

```python
context = "\n\n".join([
    f"Document: {m['path']}\nSummary:\n{m['summary']}"
    for m in memories
])
```

Hermes sends the question and retrieved memory to the local model:

```python
f"Question:\n{question}\n\n"
f"Relevant workspace memory:\n{context}"
```

The runtime loop checks for a query file:

```python
if os.path.exists("/workspace/query.txt"):
```

After reading the question, Hermes removes the query file:

```python
os.remove("/workspace/query.txt")
```

This makes query processing event-like while keeping the runtime simple and local.

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

Follow the Hermes logs:

```bash
docker logs -f hermes
```

Expected startup output:

```text
[Hermes Agent] Starting workspace watcher...
[Hermes Agent] Monitoring: /workspace/inbox
```

## Create Test Memory

Before testing retrieval, create a few new documents so Qdrant contains useful semantic memory.

Create a document about CPU orchestration:

```bash
cat > ~/dgx-hermes-agent/workspace/inbox/cpu-orchestration-note.txt <<'EOF'
Arm CPUs are responsible for orchestration in persistent AI runtimes.
They coordinate filesystem events, runtime scheduling, container services,
document parsing, metadata handling, and vector database operations.
EOF
```

Create a document about GPU inference:

```bash
cat > ~/dgx-hermes-agent/workspace/inbox/gpu-inference-note.txt <<'EOF'
NVIDIA GPUs accelerate local model inference, token generation,
summarization, embedding generation, and contextual reasoning workloads.
EOF
```

Create a document about semantic memory:

```bash
cat > ~/dgx-hermes-agent/workspace/inbox/semantic-memory-note.txt <<'EOF'
Semantic memory stores embeddings and metadata in a vector database.
This allows persistent AI systems to retrieve relevant prior context
based on meaning instead of exact keyword matching.
EOF
```

Watch the Hermes logs until each document is summarized, embedded, and stored.

Expected log lines include:

```text
[Agent] Generating embeddings...
[Memory] Stored document:
```

## Test Semantic Retrieval

Create a query file:

```bash
echo "How do CPUs help persistent AI systems?" \
> ~/dgx-hermes-agent/workspace/query.txt
```

Hermes checks for `/workspace/query.txt` in the runtime loop. When it sees the file, it reads the question and performs retrieval.

Watch the logs for:

```text
[Memory] Searching semantic memory...
```

Then:

```text
[Workspace Query]
How do CPUs help persistent AI systems?

[Retrieved Memories]
```

And finally:

```text
[AI Response]
```

The exact answer will vary, but it should refer to retrieved memory about CPU orchestration, filesystem events, scheduling, and runtime coordination.

## Verify Contextual Reasoning

Ask a second question:

```bash
echo "Why does the runtime need semantic memory?" \
> ~/dgx-hermes-agent/workspace/query.txt
```

Expected behavior:

- Hermes embeds the question
- Qdrant retrieves relevant summaries
- Hermes assembles the retrieved summaries into context
- Ollama generates an answer grounded in that context

The logs should include a retrieved memory from the semantic memory document you created earlier.

## Retrieval Workflow

The full retrieval workflow is:

```text
[/workspace/query.txt]
       |
       v
[Hermes reads question]
       |
       v
[Ollama generates query embedding]
       |
       v
[Qdrant searches workspace_memory]
       |
       v
[Hermes assembles context]
       |
       v
[Ollama generates response]
       |
       v
[Hermes prints answer]
```

This creates a local contextual reasoning loop using persistent memory.

## CPU and GPU Responsibilities

The Arm Grace CPU coordinates retrieval:

- Watches for `query.txt`
- Reads and deletes the query file
- Calls Ollama for query embeddings
- Calls Qdrant for vector search
- Parses Qdrant result payloads
- Assembles retrieved context
- Calls Ollama for contextual reasoning

The Blackwell GPU accelerates:

- Query embedding generation
- Contextual LLM inference
- Response generation

Qdrant performs the vector similarity search and returns the most relevant memory payloads.

## Runtime Compatibility Notes

This section uses the current Qdrant semantic retrieval API:

```python
qdrant.query_points(
    collection_name=COLLECTION_NAME,
    query=embedding,
    limit=3
).points
```

This section also uses the current Ollama embedding API:

```python
response = client.embed(
    model="nomic-embed-text",
    input=content[:4000]
)

return response["embeddings"][0]
```

The collection vector dimension remains:

```text
768
```

for:

```text
nomic-embed-text
```

## Summary

You added semantic retrieval and contextual reasoning to Hermes Agent.

You implemented:

- Query embedding generation
- Qdrant semantic search with `query_points(...)`
- Result payload parsing
- Retrieved memory assembly
- Query-driven local reasoning with `qwen2.5:7b`
- File-based interactive queries using `/workspace/query.txt`

The runtime can now store memory and reason over it.

Next, you will add autonomous workspace cognition.
