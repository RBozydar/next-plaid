# Next-Plaid API Usage

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

- ONNX Runtime v1.23.0 — set `ORT_DYLIB_PATH` and `LD_LIBRARY_PATH`:
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
  --model lightonai/answerai-colbert-small-v1-onnx \
  --int8 --parallel 16 --batch-size 4
```

Note: HuggingFace model auto-download (`org/model` format) is handled by
`docker-entrypoint.sh`. When running the binary directly, download the model
files manually and pass the local path with `--model /path/to/model`.

## API Workflow

### 1. Create an Index

```bash
curl -X POST http://localhost:8080/indices \
  -H 'Content-Type: application/json' \
  -d '{"name": "my_docs"}'
```

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
      {"title": "France", "category": "geography"},
      {"title": "Germany", "category": "geography"},
      {"title": "ML Intro", "category": "tech"}
    ]
  }'
```

### 3. Search (server encodes the query automatically)

```bash
curl -X POST http://localhost:8080/indices/my_docs/search_with_encoding \
  -H 'Content-Type: application/json' \
  -d '{
    "queries": ["What is the capital of France?"],
    "params": {"top_k": 3}
  }'
```

### 4. Filtered Search (metadata SQL WHERE)

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

### 5. Hybrid Search (semantic + keyword BM25)

```bash
curl -X POST http://localhost:8080/indices/my_docs/search_with_encoding \
  -H 'Content-Type: application/json' \
  -d '{
    "queries": ["capital city"],
    "text_query": ["France"],
    "alpha": 0.75,
    "params": {"top_k": 5}
  }'
```

`alpha` controls the blend: `1.0` = pure semantic, `0.0` = pure keyword.

### 6. Rerank Documents

```bash
curl -X POST http://localhost:8080/rerank_with_encoding \
  -H 'Content-Type: application/json' \
  -d '{
    "query": "What is the capital of France?",
    "documents": [
      "Paris is the capital of France.",
      "Berlin is the capital of Germany."
    ]
  }'
```

### 7. Get Raw Embeddings (without indexing)

```bash
curl -X POST http://localhost:8080/encode \
  -H 'Content-Type: application/json' \
  -d '{
    "texts": ["Paris is the capital of France."],
    "input_type": "document"
  }'
```

Use `"input_type": "query"` for query encoding (applies query expansion with MASK tokens).

## Other Endpoints

| Endpoint | Method | Purpose |
|---|---|---|
| `/health` | GET | Health check and server info |
| `/indices` | GET | List all indices |
| `/indices/{name}` | GET | Index info (doc count, dimensions, etc.) |
| `/indices/{name}` | DELETE | Delete an index |
| `/indices/{name}/documents` | DELETE | Delete docs by metadata filter |
| `/indices/{name}/metadata/get` | POST | Fetch metadata by IDs or condition |
| `/indices/{name}/metadata/update` | POST | Update metadata fields |
| `/indices/{name}/metadata/count` | GET | Count metadata entries |
| `/indices/{name}/config` | PUT | Update index config (e.g. max_documents) |

## Notes

- **Swagger UI** is available at `http://localhost:8080/swagger-ui/`
- **Index storage**: indices are stored under the `--index-dir` path as `<index-dir>/<index-name>/`
- **One model per server**: the API loads a single model at startup. To use multiple models, run separate instances or rebuild indices when switching models. Indices are not portable between models — even if dimensions match, the embedding spaces differ.
