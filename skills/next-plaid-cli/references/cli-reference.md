# Next Plaid CLI -- Complete Reference

## `next-plaid health`

Check server health and status.

```bash
next-plaid health
next-plaid health --json
next-plaid -u http://remote:8080 health
```

No additional options. Human output shows status, version, loaded indices, index dir, memory usage, and per-index summary.

---

## `next-plaid index list`

List all index names.

```bash
next-plaid index list
next-plaid index list --json
```

No additional options. Returns newline-separated names or JSON array.

---

## `next-plaid index get <name>`

Get detailed info for an index.

| Argument | Required | Description |
|----------|----------|-------------|
| `name` | yes | Index name |

Output includes: name, num_documents, num_embeddings, num_partitions, avg_doclen, dimension, has_metadata, metadata_count, max_documents.

---

## `next-plaid index create <name>`

Create a new index.

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--nbits` | `2` or `4` | `4` | Quantization bits |
| `--batch-size` | int | `50000` | Documents per indexing batch |
| `--seed` | int | none | Random seed for K-means |
| `--max-documents` | int | none | Max documents (evicts oldest when exceeded) |
| `--fts-tokenizer` | `unicode61` or `trigram` | none | FTS5 tokenizer. `unicode61` for word search, `trigram` for code/substring |

```bash
next-plaid index create my_index
next-plaid index create my_index --nbits 2 --max-documents 10000
next-plaid index create code_index --fts-tokenizer trigram
```

---

## `next-plaid index delete <name>`

Delete an index and all its data. **Destructive and irreversible.**

| Option | Description |
|--------|-------------|
| `--yes, -y` | Skip confirmation prompt |
| `--dry-run` | Show what would be deleted without acting |

```bash
next-plaid index delete my_index --yes
next-plaid index delete my_index --dry-run
```

---

## `next-plaid index config <name>`

Update index configuration.

| Option | Type | Description |
|--------|------|-------------|
| `--max-documents` | int | Set max documents limit. Pass `0` to remove the limit. |

```bash
next-plaid index config my_index --max-documents 10000
next-plaid index config my_index --max-documents 0   # removes limit
```

---

## `next-plaid document add <index_name>`

Add documents to an index. Accepts text strings (server encodes) or pre-computed embeddings as JSON.

| Option | Type | Description |
|--------|------|-------------|
| `--text, -t` | string | Text document to add (repeatable) |
| `--file, -f` | path | JSON file with documents |
| `--stdin` | flag | Read documents from stdin (JSON array) |
| `--metadata-file` | path | JSON file with metadata array |
| `--pool-factor` | int | Token pooling factor (e.g., 2 for 2x reduction) |

Input formats:
- `--text`: one or more plain text strings
- `--file`: JSON array of strings `["doc1", "doc2"]` or objects with embeddings `[{"embeddings": [[...]]}]`
- `--stdin`: same JSON formats piped via stdin

```bash
next-plaid document add my_index --text "Paris is the capital of France"
next-plaid document add my_index --text "Doc 1" --text "Doc 2"
next-plaid document add my_index --file texts.json
echo '["Doc one", "Doc two"]' | next-plaid document add my_index --stdin
next-plaid document add my_index --text "Paris" --metadata-file meta.json
```

---

## `next-plaid document delete <index_name>`

Delete documents matching a metadata condition. **Destructive.**

| Option | Type | Description |
|--------|------|-------------|
| `--condition, -c` | string | **Required.** SQL WHERE condition (e.g., `"category = ?"`) |
| `--param, -p` | string | Condition parameter placeholder (repeatable, in order) |
| `--yes, -y` | flag | Skip confirmation prompt |
| `--dry-run` | flag | Show condition without executing |

```bash
next-plaid document delete my_index -c "category = ?" -p draft --yes
next-plaid document delete my_index -c "year < ?" -p 2020 --yes
next-plaid document delete my_index -c "id IN (?, ?)" -p 1 -p 2 --dry-run
```

---

## `next-plaid search <index_name> [queries...]`

Search an index. Supports semantic, keyword, and hybrid modes.

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--stdin` | flag | -- | Read queries from stdin (JSON array of strings) |
| `--top-k, -k` | int | `10` | Results per query |
| `--n-ivf-probe` | int | `8` | IVF cells to probe |
| `--n-full-scores` | int | `4096` | Candidates for re-ranking |
| `--centroid-threshold` | float | `0.4` | Centroid score threshold. `0` disables pruning. |
| `--filter` | string | -- | SQL WHERE filter on metadata |
| `--filter-param` | string | -- | Filter parameter (repeatable) |
| `--text-query` | string | -- | FTS5 keyword query (repeatable) |
| `--alpha` | float | `0.75` | Hybrid balance: 0=keyword, 1=semantic |
| `--fusion` | `rrf` or `relative_score` | -- | Hybrid fusion strategy |
| `--subset` | int | -- | Restrict to these document IDs (repeatable) |

**Mode dispatch:**
- Positional queries only -> semantic search
- `--text-query` only -> keyword search
- Both positional queries AND `--text-query` -> hybrid search

```bash
# Semantic
next-plaid search my_index "What is machine learning?" --top-k 5

# Keyword
next-plaid search my_index --text-query "machine learning"

# Hybrid
next-plaid search my_index "What is ML?" --text-query "machine learning" --alpha 0.75

# With filter
next-plaid search my_index "AI papers" --filter "year > ?" --filter-param 2022

# Subset
next-plaid search my_index "query" --subset 1 --subset 2 --subset 3

# From stdin
echo '["query 1", "query 2"]' | next-plaid search my_index --stdin
```

---

## `next-plaid metadata list <index_name>`

Get all metadata entries for an index.

```bash
next-plaid metadata list my_index
next-plaid metadata list my_index --json
```

---

## `next-plaid metadata count <index_name>`

Count metadata entries. Returns count and has_metadata flag.

```bash
next-plaid metadata count my_index
```

---

## `next-plaid metadata check <index_name>`

Check which document IDs have metadata.

| Option | Type | Description |
|--------|------|-------------|
| `--ids, -i` | int | **Required.** Document ID to check (repeatable) |

```bash
next-plaid metadata check my_index --ids 1 --ids 2 --ids 3
```

---

## `next-plaid metadata query <index_name>`

Query metadata by SQL condition, returning matching document IDs.

| Option | Type | Description |
|--------|------|-------------|
| `--condition, -c` | string | **Required.** SQL WHERE condition |
| `--param, -p` | string | Condition parameter (repeatable) |

```bash
next-plaid metadata query my_index -c "category = ?" -p science
next-plaid metadata query my_index -c "score > ?" -p 90
```

---

## `next-plaid metadata get <index_name>`

Get metadata by document IDs or SQL condition.

| Option | Type | Description |
|--------|------|-------------|
| `--ids, -i` | int | Document ID (repeatable) |
| `--condition, -c` | string | SQL WHERE condition |
| `--param, -p` | string | Condition parameter (repeatable) |
| `--limit, -l` | int | Max results |

```bash
next-plaid metadata get my_index --ids 1 --ids 2
next-plaid metadata get my_index -c "category = ?" -p science
next-plaid metadata get my_index -c "score > ?" -p 90 --limit 10
```

---

## `next-plaid metadata update <index_name>`

Update metadata rows matching a condition. **Destructive.**

| Option | Type | Description |
|--------|------|-------------|
| `--condition, -c` | string | **Required.** SQL WHERE condition for rows to update |
| `--param, -p` | string | Condition parameter (repeatable) |
| `--set` | string | **Required.** JSON object with column updates |
| `--yes, -y` | flag | Skip confirmation |
| `--dry-run` | flag | Show what would be updated without acting |

```bash
next-plaid metadata update my_index -c "status = ?" -p draft --set '{"status": "published"}' --yes
next-plaid metadata update my_index -c "score > ?" -p 90 --set '{"reviewed": true}' --dry-run
```

---

## `next-plaid encode [texts...]`

Encode texts into ColBERT embeddings. Requires model loaded on server.

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--stdin` | flag | -- | Read texts from stdin (JSON array) |
| `--input-type` | `document` or `query` | `document` | Encoding mode |
| `--pool-factor` | int | -- | Token pooling reduction factor |

```bash
next-plaid encode "Hello world" "Another text"
next-plaid encode --input-type query "What is AI?"
echo '["text1", "text2"]' | next-plaid encode --stdin
```

---

## `next-plaid rerank`

Rerank documents by relevance to a query using ColBERT MaxSim scoring.

| Option | Type | Description |
|--------|------|-------------|
| `--query, -q` | string | **Required.** Query text |
| `--document, -d` | string | Document text (repeatable) |
| `--file, -f` | path | JSON file with documents array |
| `--stdin` | flag | Read documents from stdin (JSON array of strings) |
| `--pool-factor` | int | Token pooling reduction factor |

```bash
next-plaid rerank -q "capital of France" -d "Paris is in France" -d "Berlin is in Germany"
next-plaid rerank -q "machine learning" --file docs.json
echo '["doc1", "doc2"]' | next-plaid rerank -q "my query" --stdin
```
