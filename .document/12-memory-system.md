# 12 — Memory System

**Source files**: `src/memory/manager.ts`, `src/memory/memory-schema.ts`, `src/memory/embeddings.ts`, `src/memory/types.ts`, `src/memory/backend-config.ts`

---

## 1. Overview

The memory system gives agents **long-term recall** beyond the current session's context window. It indexes memory files (markdown, text, etc.) into a SQLite database using:

- **Vector search** — semantic similarity via embeddings
- **Full-text search (FTS)** — keyword/BM25 ranking
- **Hybrid search** — combines both with configurable weights

Memory is scoped per agent and stored at `~/.openclaw/agents/<agentId>/memory.db`.

---

## 2. Architecture

```
Memory files (workspace + configured paths)
    │
    ▼
MemoryIndexManager
    ├── File watcher (watch for changes)
    ├── Chunker (split files into searchable chunks)
    ├── Embedding provider (generate vector embeddings)
    ├── SQLite index (store chunks + embeddings + FTS)
    └── Search (vector + FTS + hybrid)
```

`src/memory/manager.ts` — `MemoryIndexManager` is the main class:
- Extends `MemoryManagerEmbeddingOps` (embedding provider management)
- Provides `search()`, `sync()`, `probeEmbeddingAvailability()`, `status()`
- Handles provider lifecycle and fallback chains

---

## 3. SQLite Schema

`src/memory/memory-schema.ts` defines the SQLite tables:

| Table | Purpose |
|-------|---------|
| `meta` | Schema version, last sync timestamp |
| `files` | Indexed file records (path, mtime, size) |
| `chunks` | Text chunks with source file reference |
| `embedding_cache` | Cached embedding vectors (avoid recomputation) |
| FTS virtual table | Full-text search index over chunk content |

Schema migrations are handled automatically on startup.

---

## 4. Embedding Providers

`src/memory/embeddings.ts` — factory for embedding clients. Supported providers:

| Provider | Notes |
|----------|-------|
| OpenAI | `text-embedding-3-small` default |
| Gemini | Google Generative AI embeddings |
| Voyage | voyage-3-lite default |
| Mistral | mistral-embed |
| Ollama | Local self-hosted embeddings |

Provider selection:
1. Reads `cfg.agents[id].memory.embeddingProvider` config
2. Falls back to first available provider with a valid API key
3. Supports auto-fallback chain if primary fails

---

## 5. Search Modes

### 5.1 Vector Search

Uses sqlite-vec extension to compute cosine similarity between the query embedding and stored chunk embeddings. Returns top-K results by similarity score.

### 5.2 Full-Text Search

Uses SQLite FTS5 virtual table with BM25 ranking. Handles tokenization and stopwords automatically.

### 5.3 Hybrid Search

`src/memory/hybrid.ts` merges vector and FTS results:
- Applies reciprocal rank fusion (RRF) or configurable weight blend
- Deduplicates results from both pipelines
- Returns merged result list with combined relevance score

---

## 6. Memory Tool Integration

The agent accesses memory via tools registered by the memory plugin:

| Tool | Description |
|------|-------------|
| `memory_search` | Semantic + keyword search |
| `memory_get` | Retrieve specific memory entry |
| `memory_update` | Write/update a memory entry |

Memory search results are formatted with source citations so the agent can reference where information came from.

---

## 7. Memory Backends

`src/memory/backend-config.ts` — resolves which backend to use:

- **Builtin** (`extensions/memory-core/`): file-based memory with SQLite index (default)
- **LanceDB** (`extensions/memory-lancedb/`): vector database for large-scale semantic search
- **QMD**: alternative backend mode

Only one memory backend can be active at a time (enforced via plugin slot — see `07-plugin-sdk.md` §9).

---

## 8. Session Memory via Hooks

`src/hooks/bundled/session-memory/` — a bundled internal hook that:
- Fires on `session:end` events
- Extracts notable content from the session transcript
- Writes it to the memory index for future recall

This creates the "long-term memory" experience: facts learned in one session are available in future sessions.

---

## 9. Key Files Summary

| File | Role |
|------|------|
| `src/memory/manager.ts` | Main `MemoryIndexManager` orchestrator |
| `src/memory/memory-schema.ts` | SQLite schema + migrations |
| `src/memory/embeddings.ts` | Embedding provider factory + clients |
| `src/memory/backend-config.ts` | Backend configuration resolution |
| `src/memory/manager-embedding-ops.ts` | Embedding generation + caching |
| `src/memory/manager-search.ts` | Vector + FTS search implementation |
| `src/memory/hybrid.ts` | BM25 + vector result merging |
| `src/memory/types.ts` | `MemorySearchManager`, `MemorySearchResult`, `MemoryProviderStatus` |
| `extensions/memory-core/` | Default file-based memory backend |
| `extensions/memory-lancedb/` | LanceDB vector memory backend |
