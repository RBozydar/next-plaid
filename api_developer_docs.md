# Next-Plaid API Developer Documentation

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Docker Image](#docker-image)
- [Running Without Docker](#running-without-docker)
- [API Workflow](#api-workflow)
- [Search Modes](#search-modes)
- [Reranking](#reranking)
- [Raw Embeddings](#raw-embeddings)
- [Metadata & Filtering](#metadata--filtering)
- [Endpoint Reference](#endpoint-reference)
- [Index Storage](#index-storage)
- [Tokenizer & Chunking](#tokenizer--chunking)
- [Model Compatibility](#model-compatibility)
- [Production Guide](#production-guide)
- [Environment Variables](#environment-variables)

---

## Architecture Overview

Next-Plaid is a ColBERT multi-vector search engine exposed as a REST API. It handles:

- **Encoding** — built-in ColBERT model via ONNX Runtime
- **Indexing** — multi-vector index with quantization (IVF + product quantization)
- **Semantic search** — ColBERT MaxSim scoring
- **Keyword search** — FTS5/BM25 over metadata
- **Hybrid search** — semantic + keyword fusion (RRF or relative score)
- **Metadata filtering** — SQL WHERE clauses over document metadata
- **Reranking** — full-precision MaxSim rescoring
- **CRUD** — add/delete/update documents and metadata
- **Persistence** — indices survive restarts (memory-mapped files + SQLite)

What it does **not** include (handle externally):

- **Document storage** — the index stores quantized vectors and metadata, not the original text. Store full text in metadata or an external store.
- **Chunking** — you must split documents into passages before indexing. The model has a max document length in tokens.
- **LLM / generation** — this is a retrieval engine. Pipe results into an LLM for the "G" in RAG.
- **Ingestion pipeline** — orchestrating read source docs -> chunk -> index -> store original text is application-level glue.

---

## Docker Image

The Dockerfile at `next-plaid-api/Dockerfile` is a multi-stage, multi-target Rust build.

### Build Variants

| Target | Base Image | Features | Use Case |
|---|---|---|---|
| `runtime-cpu` (default) | `debian:bookworm-slim` | `mkl,model` (x86_64) or `openblas,model` (aarch64) | CPU inference |
| `runtime-cuda` | `nvidia/cuda:12.4.1-cudnn-runtime-ubuntu22.04` | `openblas,cuda` | GPU inference |

### Build Stages (CPU path)

1. **chef-cpu** — installs Rust, `cargo-chef`, downloads ONNX Runtime CPU libs (v1.23.0, arch-aware)
2. **planner-cpu** — generates `recipe.json` for dependency caching
3. **builder-cpu** — builds dependencies (cached layer), then builds the binary
4. **runtime-cpu** — minimal Debian image with just the binary, ONNX Runtime libs, and entrypoint

### Docker Compose Files

| File | Variant | Default Model |
|---|---|---|
| `docker-compose.yml` | CPU | `lightonai/answerai-colbert-small-v1-onnx` with `--int8` |
| `docker-compose.cuda.yml` | GPU | `lightonai/GTE-ModernColBERT-v1` |
| `docker-compose.cuda.local.yml` | GPU (local paths) | `lightonai/GTE-ModernColBERT-v1` |

### Docker Run Examples

```bash
# CPU (default)
docker compose up -d

# CUDA
docker compose -f docker-compose.cuda.yml up -d

# Custom model
MODEL=lightonai/my-model docker compose up -d
```

Default CPU compose already passes `--parallel 16 --batch-size 4`. The CUDA compose uses `--batch-size 128` (no `--parallel` — GPU handles parallelism internally).

### Entrypoint

`docker-entrypoint.sh` handles:
- Detecting HuggingFace model IDs (`org/model` format) and auto-downloading model files
- Supporting `--int8` for quantized model downloads
- Atomic downloads (temp dir + rename) to prevent partial downloads
- Rewriting `--model <hf-id>` to `--model /local/path` before exec'ing the binary

---

## Running Without Docker

### Build

```bash
# x86_64 Linux (Intel MKL, statically linked):
cargo build --release -p next-plaid-api --features "mkl,model"

# aarch64 Linux (needs libopenblas-dev):
cargo build --release -p next-plaid-api --features "openblas,model"

# macOS (Accelerate framework):
cargo build --release -p next-plaid-api --features "accelerate,model"
```

### Runtime Dependencies

- **ONNX Runtime v1.23.0** — set environment variables:
  ```bash
  export ORT_DYLIB_PATH=/path/to/libonnxruntime.so.1.23.0
  export LD_LIBRARY_PATH=/path/to/ort/lib:$LD_LIBRARY_PATH
  ```
- `libssl3`, `libsqlite3` (likely already installed)
- `libopenblas` (aarch64 only)

### Run

```bash
./target/release/next-plaid-api \
  --host 0.0.0.0 --port 8080 \
  --index-dir ~/.local/share/next-plaid \
  --model /path/to/local/model \
  --int8 --parallel 16 --batch-size 4
```

Note: HuggingFace model auto-download (`org/model` format) is handled by `docker-entrypoint.sh` only. When running the binary directly, download the model files manually and pass the local path.

---

## API Workflow

### 1. Create an Index

```bash
curl -X POST http://localhost:8080/indices \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "my_docs",
    "config": {
      "nbits": 4,
      "fts_tokenizer": "unicode61"
    }
  }'
```

Configuration options:
- `nbits` — quantization bits (2 or 4, default: 4). Lower = smaller but less accurate.
- `fts_tokenizer` — `"unicode61"` (word-level, default) or `"trigram"` (substring/code search)
- `start_from_scratch` — rebuild threshold (default: 999). Index is rebuilt entirely when doc count <= this value.
- `max_documents` — optional cap. Oldest documents are evicted when exceeded.
- `seed` — random seed for reproducibility.

### 2. Add Documents (server encodes text automatically)

```bash
curl -X POST http://localhost:8080/indices/my_docs/update_with_encoding \
  -H 'Content-Type: application/json' \
  -d '{
    "documents": [
      "Paris is the capital of France.",
      "Berlin is the capital of Germany.",
      "Machine learning is a subset of artificial intelligence."
    ],
    "metadata": [
      {"title": "France", "category": "geography", "text": "Paris is the capital of France."},
      {"title": "Germany", "category": "geography", "text": "Berlin is the capital of Germany."},
      {"title": "ML Intro", "category": "tech", "text": "Machine learning is a subset of artificial intelligence."}
    ]
  }'
```

Tip: store the original text in metadata so search results give you the text back directly without needing a separate document store.

This endpoint returns `202 Accepted` — documents are batched in the background. Poll `GET /indices/my_docs` to confirm the update landed.

### 3. Search

```bash
curl -X POST http://localhost:8080/indices/my_docs/search_with_encoding \
  -H 'Content-Type: application/json' \
  -d '{
    "queries": ["What is the capital of France?"],
    "params": {"top_k": 3}
  }'
```

Response:

```json
{
  "results": [
    {
      "query_id": 0,
      "document_ids": [0, 1, 2],
      "scores": [0.95, 0.87, 0.42],
      "metadata": [
        {"title": "France", "category": "geography", "text": "..."},
        {"title": "Germany", "category": "geography", "text": "..."},
        {"title": "ML Intro", "category": "tech", "text": "..."}
      ]
    }
  ],
  "num_queries": 1
}
```

---

## Search Modes

### Semantic Search (ColBERT MaxSim)

```bash
curl -X POST http://localhost:8080/indices/my_docs/search_with_encoding \
  -H 'Content-Type: application/json' \
  -d '{
    "queries": ["What is the capital of France?"],
    "params": {"top_k": 10}
  }'
```

### Keyword Search (FTS5/BM25)

Pure keyword search over metadata, no embeddings needed:

```bash
curl -X POST http://localhost:8080/indices/my_docs/search_with_encoding \
  -H 'Content-Type: application/json' \
  -d '{
    "text_query": ["machine learning"],
    "params": {"top_k": 10}
  }'
```

### Hybrid Search (semantic + keyword)

```bash
curl -X POST http://localhost:8080/indices/my_docs/search_with_encoding \
  -H 'Content-Type: application/json' \
  -d '{
    "queries": ["What is ML?"],
    "text_query": ["machine learning"],
    "alpha": 0.75,
    "fusion": "rrf",
    "params": {"top_k": 10}
  }'
```

- `alpha` — blend weight. `1.0` = pure semantic, `0.0` = pure keyword. Default: `0.75`.
- `fusion` — `"rrf"` (reciprocal rank fusion, default) or `"relative_score"` (min-max normalize then alpha-weight).
- In hybrid mode, `queries` must have exactly 1 element (single semantic query fused with the text query).

### Filtered Search (metadata SQL WHERE)

```bash
curl -X POST http://localhost:8080/indices/my_docs/search_with_encoding \
  -H 'Content-Type: application/json' \
  -d '{
    "queries": ["capital city"],
    "params": {"top_k": 5},
    "filter_condition": "category = ?",
    "filter_parameters": ["geography"]
  }'
```

Filters can be combined with any search mode (semantic, keyword, or hybrid).

### Search Parameters

| Parameter | Default | Description |
|---|---|---|
| `top_k` | 10 | Number of results per query |
| `n_ivf_probe` | 8 | Number of IVF cells to probe (higher = more accurate, slower) |
| `n_full_scores` | 4096 | Number of documents for exact re-ranking |
| `centroid_score_threshold` | None | Centroid pruning threshold (e.g. 0.4) for faster but less accurate search |

---

## Reranking

Reranking uses ColBERT MaxSim at full precision — for each query token, find the max similarity with any document token, then sum those maximums. Results are sorted by score descending.

### Text-based (server encodes for you)

```bash
curl -X POST http://localhost:8080/rerank_with_encoding \
  -H 'Content-Type: application/json' \
  -d '{
    "query": "What is the capital of France?",
    "documents": [
      "Paris is the capital of France.",
      "Berlin is the capital of Germany.",
      "Pizza is delicious."
    ]
  }'
```

Response:

```json
{
  "results": [
    {"index": 0, "score": 12.5},
    {"index": 1, "score": 8.3},
    {"index": 2, "score": 2.1}
  ],
  "num_documents": 3
}
```

### Pre-computed embeddings

```bash
curl -X POST http://localhost:8080/rerank \
  -H 'Content-Type: application/json' \
  -d '{
    "query": [[0.1, 0.2, ...], [0.3, 0.4, ...]],
    "documents": [
      {"embeddings": [[0.1, 0.2, ...], [0.3, 0.4, ...]]},
      {"embeddings": [[0.5, 0.6, ...], [0.7, 0.8, ...]]}
    ]
  }'
```

### Retrieve-then-Rerank Pipeline

There is no single endpoint that searches an index and reranks results. To get higher-accuracy results:

1. Search the index with a larger `top_k` (e.g. 50) — this uses quantized embeddings (fast but lossy)
2. Rerank the top results with `/rerank_with_encoding` — this encodes from scratch at full precision

This requires the original document text to be available (store it in metadata or an external store).

### Pool Factor

Optional `pool_factor` parameter reduces document token count via hierarchical clustering. E.g. `"pool_factor": 2` halves the token embeddings per doc. Only applies to document encoding.

---

## Raw Embeddings

Get ColBERT embeddings without indexing (for BYO pipelines):

```bash
curl -X POST http://localhost:8080/encode \
  -H 'Content-Type: application/json' \
  -d '{
    "texts": ["Paris is the capital of France."],
    "input_type": "document"
  }'
```

- `"input_type": "query"` — applies query expansion with MASK tokens
- `"input_type": "document"` — filters padding and skiplist tokens
- Optional `"pool_factor": 2` to reduce token count

Response format defaults to JSON. Set `X-Embeddings-Format: base64` header for base64-encoded little-endian f32 (more compact).

---

## Metadata & Filtering

Metadata is stored in SQLite alongside the vector index. Any JSON fields you include are filterable.

### Add Metadata with Documents

Metadata is attached at indexing time via the `metadata` array (must match `documents` length):

```json
{
  "documents": ["chunk text..."],
  "metadata": [
    {
      "file_path": "/docs/guide.pdf",
      "page": 3,
      "chunk_index": 0,
      "title": "User Guide",
      "category": "documentation",
      "text": "chunk text..."
    }
  ]
}
```

### Query Metadata

```bash
# Get all metadata
curl http://localhost:8080/indices/my_docs/metadata

# Get metadata by IDs
curl -X POST http://localhost:8080/indices/my_docs/metadata/get \
  -H 'Content-Type: application/json' \
  -d '{"document_ids": [0, 5, 10]}'

# Query by condition
curl -X POST http://localhost:8080/indices/my_docs/metadata/query \
  -H 'Content-Type: application/json' \
  -d '{
    "condition": "category = ? AND page > ?",
    "parameters": ["documentation", 5]
  }'

# Count metadata entries
curl http://localhost:8080/indices/my_docs/metadata/count

# Check which IDs exist
curl -X POST http://localhost:8080/indices/my_docs/metadata/check \
  -H 'Content-Type: application/json' \
  -d '{"document_ids": [0, 5, 999]}'
```

### Update Metadata

```bash
curl -X POST http://localhost:8080/indices/my_docs/metadata/update \
  -H 'Content-Type: application/json' \
  -d '{
    "condition": "category = ?",
    "parameters": ["geography"],
    "updates": {"status": "reviewed", "updated_at": "2024-01-15"}
  }'
```

### Delete Documents by Metadata

```bash
curl -X DELETE http://localhost:8080/indices/my_docs/documents \
  -H 'Content-Type: application/json' \
  -d '{
    "condition": "category = ? AND year < ?",
    "parameters": ["outdated", 2020]
  }'
```

---

## Endpoint Reference

| Endpoint | Method | Purpose |
|---|---|---|
| `/health` | GET | Health check and server info |
| `/indices` | GET | List all indices |
| `/indices` | POST | Create/declare a new index |
| `/indices/{name}` | GET | Index info (doc count, dimensions, etc.) |
| `/indices/{name}` | DELETE | Delete an index |
| `/indices/{name}/config` | PUT | Update index config (e.g. max_documents) |
| `/indices/{name}/update` | POST | Add documents with pre-computed embeddings |
| `/indices/{name}/update_with_encoding` | POST | Add documents (server encodes text) |
| `/indices/{name}/documents` | POST | Add documents with pre-computed embeddings |
| `/indices/{name}/documents` | DELETE | Delete documents by metadata filter |
| `/indices/{name}/search` | POST | Search with pre-computed query embeddings |
| `/indices/{name}/search_with_encoding` | POST | Search with text queries (server encodes) |
| `/indices/{name}/search/filtered` | POST | Filtered search with pre-computed embeddings |
| `/indices/{name}/search/filtered_with_encoding` | POST | Filtered search with text queries |
| `/indices/{name}/metadata` | GET | Get all metadata |
| `/indices/{name}/metadata/get` | POST | Get metadata by IDs or condition |
| `/indices/{name}/metadata/count` | GET | Count metadata entries |
| `/indices/{name}/metadata/check` | POST | Check which document IDs have metadata |
| `/indices/{name}/metadata/query` | POST | Query metadata with SQL WHERE |
| `/indices/{name}/metadata/update` | POST | Update metadata fields |
| `/encode` | POST | Get raw embeddings (no indexing) |
| `/rerank` | POST | Rerank with pre-computed embeddings |
| `/rerank_with_encoding` | POST | Rerank with text (server encodes) |

Swagger UI is available at `http://localhost:8080/swagger-ui/`.

---

## Index Storage

Indices are stored under the `--index-dir` path as `<index-dir>/<index-name>/`.

Default locations:
- **Native binary**: `~/.local/share/next-plaid/` (or whatever you pass to `--index-dir`)
- **Docker**: `/data/indices` inside the container, mounted to `~/.local/share/next-plaid/` by default

Each index directory contains:
- Quantized vector index files (memory-mapped)
- `metadata.json` — index metadata (doc count, dimensions, etc.)
- `config.json` — index configuration
- SQLite database for document metadata and FTS5

---

## Tokenizer & Chunking

The encoding layer uses the HuggingFace `tokenizers` Rust crate (v0.21.1), loading from the `tokenizer.json` file bundled with each model. The tokenizer is model-specific — whatever the model was trained with (typically BPE or WordPiece).

### Document Length Limits

The model has a max document length in **tokens** (visible in `/health` response as `document_length`). Text exceeding this is silently truncated — no error, no warning, you just lose the tail end.

### Chunking Strategy

You must chunk before indexing. A simple approach:

- Split into chunks of ~300-400 tokens with some overlap (e.g. 50 tokens)
- Store the source location in metadata for traceability:

```bash
curl -X POST http://localhost:8080/indices/my_docs/update_with_encoding \
  -H 'Content-Type: application/json' \
  -d '{
    "documents": [
      "First chunk of the document...",
      "Second chunk with overlap..."
    ],
    "metadata": [
      {"file_path": "/docs/guide.pdf", "page": 1, "chunk": 0, "text": "First chunk of the document..."},
      {"file_path": "/docs/guide.pdf", "page": 1, "chunk": 1, "text": "Second chunk with overlap..."}
    ]
  }'
```

To check what tokenizer your model uses:

```bash
cat /path/to/model/tokenizer.json | python3 -c "import sys,json; print(json.load(sys.stdin)['model']['type'])"
```

---

## Model Compatibility

**Indices are not portable between models.** Even if two models share the same embedding dimension, they produce vectors in different learned embedding spaces. Searching an index built with Model A using query embeddings from Model B returns meaningless results.

- The API loads **one model at startup**
- To use multiple models, run separate server instances
- Each model needs its own set of indices
- The server validates that query embedding dimensions match the index, but dimension match alone doesn't guarantee compatibility

---

## Production Guide

### Indexing Behavior

- **`start_from_scratch` (default: 999)** — when the index has <= this many docs, every update rebuilds the entire index (recomputes centroids). After crossing the threshold, updates become incremental. If bulk-loading, set this to match your initial load size.

- **Updates are async/batched** — `update_with_encoding` and `add_documents` return `202 Accepted`. Documents are batched in the background (up to `MAX_BATCH_DOCUMENTS` or 100ms, whichever comes first). Poll `GET /indices/{name}` to confirm.

- **`max_documents` eviction** — when set, oldest documents (lowest IDs) are evicted on the next add, not immediately when you lower the limit.

### Concurrency & Backpressure

- **`CONCURRENCY_LIMIT` (default: 100)** — max in-flight requests. Beyond this, requests are rejected.
- **`MAX_QUEUED_TASKS_PER_INDEX` (default: 10)** — max queued update tasks per index. Exceeding returns 503. Bump for high-throughput ingestion.
- **Encoding is the bottleneck** — `--model-pool-size` defaults to 1 worker. Increase for heavy concurrent encode workloads (costs more RAM per worker).

### Memory & Storage

- **Indices are memory-mapped** — RSS looks low but the OS pages index data in/out. Ensure enough available RAM for your working set, or search latency spikes from page faults.
- **Quantization** — `nbits: 4` (default) is a good balance. `nbits: 2` saves ~50% storage but loses accuracy. Chosen at index creation time — cannot be changed without rebuilding.
- **`--parallel 16`** means 16 ONNX sessions in memory, each holding model weights. Lower this if memory-constrained (trades throughput for RAM).

### Security & Networking

- **No authentication** — put it behind a reverse proxy (nginx, Caddy) or in a private network.
- **No TLS** — terminate SSL at the proxy.
- **Rate limiting is off by default** — enable via `RATE_LIMIT_ENABLED=true` if exposed to untrusted clients.

### Operational

- **Health check** — `GET /health` is lightweight and exempt from rate limiting. Use for load balancer probes.
- **Index repair** — if the process crashes mid-write, the server auto-repairs index/metadata sync on next access (deletes orphaned records from whichever side has more).

### Data Safety

- **No WAL or transaction log for vector data** — a crash during a write can lose that batch. The repair logic handles consistency but in-flight data is gone.
- **Back up the index directory** — copy `<index-dir>/<index-name>/` when the server is idle or stopped.
- **Metadata is in SQLite** — gets standard SQLite durability guarantees, but the vector index files don't have the same protection.

### Performance Tuning

| Scenario | Tune |
|---|---|
| Faster CPU encoding | `--int8`, increase `--parallel` |
| Faster search | increase `n_ivf_probe` (accuracy) or decrease it (speed) |
| High ingestion throughput | increase `MAX_BATCH_DOCUMENTS`, `BATCH_CHANNEL_SIZE` |
| Memory-constrained | lower `--parallel`, use `nbits: 2` |
| Many concurrent users | increase `CONCURRENCY_LIMIT`, `--model-pool-size` |

---

## Environment Variables

| Variable | Default | Description |
|---|---|---|
| `RUST_LOG` | `info` | Log level |
| `INDEX_DIR` | `/data/indices` | Index storage directory |
| `RATE_LIMIT_ENABLED` | `false` | Enable rate limiting |
| `RATE_LIMIT_PER_SECOND` | `50` | Max requests/s (when enabled) |
| `RATE_LIMIT_BURST_SIZE` | `100` | Burst size (when enabled) |
| `CONCURRENCY_LIMIT` | `100` | Max concurrent in-flight requests |
| `MAX_QUEUED_TASKS_PER_INDEX` | `10` | Max queued updates per index |
| `MAX_BATCH_DOCUMENTS` | `300` | Max documents per batch |
| `BATCH_CHANNEL_SIZE` | `100` | Buffer size for document batch queue |
| `MAX_BATCH_TEXTS` | `64` | Max texts per encode batch |
| `ENCODE_BATCH_CHANNEL_SIZE` | `256` | Buffer size for encode batch queue |
| `MODEL_POOL_SIZE` | `1` | Number of model workers |
| `OPENBLAS_NUM_THREADS` | `1` | OpenBLAS threads (aarch64) |
| `HF_TOKEN` | — | HuggingFace token for private models (Docker only) |
| `MODELS_DIR` | `/models` | Model storage directory (Docker only) |
