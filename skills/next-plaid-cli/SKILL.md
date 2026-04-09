---
name: next-plaid-cli
description: Use this skill when interacting with the Next Plaid ColBERT search server via CLI commands -- managing indices, adding/deleting documents, searching (semantic, keyword, hybrid), querying metadata, encoding text, or reranking documents. Triggers on `next-plaid` commands, Next Plaid server operations, or ColBERT index management tasks.
---

# Next Plaid CLI

## Overview

The `next-plaid` CLI provides non-interactive, agent-friendly access to the Next Plaid ColBERT Search API. It follows resource-verb patterns (`next-plaid <resource> <verb>`) with `--json` output, `--stdin` input, `--yes` confirmation bypass, and `--dry-run` previews. This skill teaches correct command construction and avoids the non-obvious pitfalls in search mode dispatch, parameter parsing, and destructive operations.

## Command Map

```
next-plaid health                          # server status
next-plaid index   list|get|create|delete|config
next-plaid document add|delete
next-plaid search  <index> <queries...>    # semantic, keyword, or hybrid
next-plaid metadata list|count|check|query|get|update
next-plaid encode  <texts...>
next-plaid rerank  -q <query> -d <docs...>
```

## Global Options

Every command accepts these before the subcommand:

| Flag | Env var | Default | Purpose |
|------|---------|---------|---------|
| `--url, -u` | `NEXT_PLAID_URL` | `http://localhost:8080` | Server URL |
| `--timeout, -t` | -- | `30.0` | Request timeout (seconds) |
| `--header, -H` | -- | -- | Extra header as `Key: Value` (repeatable) |
| `--json` | -- | off | Full JSON output (essential for agent parsing) |

## Agent Usage Patterns

When using the CLI from an agent or script, always:

1. **Use `--json`** for parseable output: `next-plaid --json search my_index "query"`
2. **Use `--yes`** on destructive ops to skip interactive prompts: `next-plaid index delete my_index --yes`
3. **Use `--stdin`** for bulk/dynamic input: `echo '["doc1","doc2"]' | next-plaid document add my_index --stdin`
4. **Use `--dry-run`** before destructive operations to verify intent

## Search Modes

The `search` command dispatches to three different server endpoints based on input combination:

| Mode | Semantic queries | `--text-query` | Notes |
|------|-----------------|----------------|-------|
| **Semantic** | required | omitted | Uses ColBERT embeddings, requires model on server |
| **Keyword** | omitted | required | FTS5 BM25 over metadata text |
| **Hybrid** | required | required | Fuses both; queries and text-queries must be same count |

### Semantic search
```bash
next-plaid search my_index "What is machine learning?" --top-k 5
```

### Keyword search
```bash
next-plaid search my_index --text-query "machine learning"
```

### Hybrid search
```bash
next-plaid search my_index "What is ML?" --text-query "machine learning" --alpha 0.75 --fusion rrf
```

### Search with metadata filter
```bash
next-plaid search my_index "AI papers" --filter "year > ?" --filter-param 2022
```

## Common Workflows

### Create index, add docs, search
```bash
next-plaid index create my_index --nbits 4
next-plaid document add my_index --text "Paris is the capital of France" --text "Berlin is the capital of Germany"
next-plaid search my_index "capital of France" --top-k 1 --json
```

### Bulk add from file or pipe
```bash
# From JSON file
next-plaid document add my_index --file documents.json

# From pipe
cat docs.json | next-plaid document add my_index --stdin

# With metadata
next-plaid document add my_index --file texts.json --metadata-file meta.json
```

### Delete by metadata condition
```bash
next-plaid document delete my_index --condition "category = ?" --param draft --dry-run
next-plaid document delete my_index --condition "category = ?" --param draft --yes
```

### Rerank candidates
```bash
next-plaid rerank -q "capital of France" -d "Paris is in France" -d "Berlin is in Germany" --json
```

## Bundled Resources

- `references/cli-reference.md` -- Complete flag-by-flag reference for every command. Consult when you need exact option names, types, defaults, or less common flags.

## Gotchas

### Hybrid search requires equal-length query lists

When combining semantic queries with `--text-query` for hybrid search, you must provide the same number of each. The server pairs them 1:1. Passing 2 semantic queries and 1 text query will fail.

### `--max-documents 0` removes the limit, doesn't set it to zero

In `next-plaid index config my_index --max-documents 0`, the CLI converts 0 to `None` before sending to the server. This removes the document cap entirely rather than setting it to zero. There is no way to set max_documents to literal 0 via the CLI.

### Document add and delete are asynchronous

Both `document add` and `document delete` return immediately with a "queued" status. The actual indexing/deletion happens in the background on the server. Don't assume documents are searchable immediately after `add` returns.

### Destructive commands require `--yes` for non-interactive use

`index delete`, `document delete`, and `metadata update` all prompt for confirmation by default. Agents must pass `--yes` (or `-y`) to skip the prompt, otherwise the command will hang waiting for stdin.

### `--centroid-threshold 0` disables pruning

Passing `--centroid-threshold 0` sets the threshold to `None` (disabled), not to 0.0. This means all centroids are scored, which is slower but more thorough. Any positive float value enables pruning at that threshold.

### Filter/condition parameters are auto-parsed to numbers

`--param` and `--filter-param` values are automatically parsed: if a value looks like an integer it becomes `int`, if it looks like a float it becomes `float`, otherwise it stays `str`. If you need a string that happens to be numeric (e.g., a ZIP code "90210"), this auto-coercion may cause type mismatches in SQL conditions.

### Text operations require a model loaded on server

`search` (semantic mode), `document add` (with text), `encode`, and `rerank` (with text) all require the server to have a ColBERT model loaded. If no model is loaded, these fail with a `ModelNotLoadedError`. Keyword-only search does not require a model.

### No `--stdin` for hybrid query combination

You can use `--stdin` to pipe semantic queries OR you can use `--text-query` flags, but there's no stdin mode for text queries. For hybrid search with dynamic inputs, build the semantic query arguments on the command line and pass `--text-query` flags.
