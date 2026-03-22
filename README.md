# Atlas Memory

> Self-hosted long-term memory server for AI agents. Single binary, zero dependencies.

![LongMemEval](https://img.shields.io/badge/LongMemEval--s-88.18%25_%232_globally-brightgreen)
![LoCoMo](https://img.shields.io/badge/LoCoMo-87.05%25_Judge-blue)
![License](https://img.shields.io/badge/license-MIT-green)

## What is Atlas Memory?

Atlas Memory is a self-hosted long-term memory server for AI agents. It stores, indexes, and retrieves conversation memories with an enriched retrieval pipeline that understands temporal reasoning, preferences, and cross-session context.

Download the binary, run it, and call a simple REST API — no cloud dependency, no vendor lock-in, no Python environment to manage.

## How It Compares

| System | LongMemEval-s | Notes |
|--------|--------------|-------|
| HydraDB | 90.79% | #1 globally |
| **Atlas Memory** | **88.18%** | **#2 globally** |
| Supermemory | 85.20% | |
| Zep | 71.20% | |
| Full-context GPT-4o | 60.20% | |
| Mem0-oss | 29.07% | |

Atlas beats every open/commercial memory system except HydraDB.

## Quick Start

```bash
# Download from Releases
curl -L -o atlas-memory https://github.com/dddabtc/atlas-memory-releases/releases/download/v4.2.4-rs/atlas-memory-linux-x86_64
chmod +x atlas-memory

# Run
./atlas-memory --port 6420

# Test
curl http://localhost:6420/health

# Store a memory
curl -X POST http://localhost:6420/memories \
  -H 'Content-Type: application/json' \
  -d '{"content": "User prefers dark roast coffee, no sugar."}'

# Search
curl -X POST http://localhost:6420/memories/search \
  -H 'Content-Type: application/json' \
  -d '{"query": "coffee preferences", "limit": 5}'
```

## Features

- 🧠 **Enriched retrieval** — query-type detection routes each question to a specialized handler (temporal, preference, aggregation, multi-session)
- 🔍 **Hybrid search** — semantic embeddings + token-overlap fallback
- 🕐 **Temporal reasoning** — date-injection and time-normalization for "what did I say last week?" queries
- 🔗 **Knowledge graph** — entity extraction, relation decomposition, cross-session entity resolution
- ⚡ **Configurable LLM chain** — OpenAI → Ollama → standalone fallback
- 📊 **Benchmark-grade accuracy** — 88.18% LongMemEval-s (#2 globally), 87.05% LoCoMo
- 🦀 **Single binary** — no runtime dependencies, no Python, no Docker required
- 💾 **SQLite storage** — zero-config, embedded database

## API Reference

Base URL: `http://localhost:6420`

### Health

```bash
GET /health
curl http://localhost:6420/health
# → {"status":"ok","version":"4.2.4-rs","memory_count":2994}
```

### Create Memory

```bash
POST /memories
curl -X POST http://localhost:6420/memories \
  -H 'Content-Type: application/json' \
  -d '{"content": "User prefers dark roast coffee.", "title": "Coffee preference", "session_id": "sess-001", "labels": ["preference"]}'
```

### Search Memories

```bash
POST /memories/search
curl -X POST http://localhost:6420/memories/search \
  -H 'Content-Type: application/json' \
  -d '{"query": "coffee preferences", "limit": 10}'
```

### List Memories

```bash
GET /memories?limit=20&offset=0
curl 'http://localhost:6420/memories?limit=20&offset=0'
```

### Get Memory

```bash
GET /memories/{id}
curl http://localhost:6420/memories/abc123
```

### Update Memory

```bash
PATCH /memories/{id}
curl -X PATCH http://localhost:6420/memories/abc123 \
  -H 'Content-Type: application/json' \
  -d '{"content": "Updated content."}'
```

### Delete Memory

```bash
DELETE /memories/{id}
curl -X DELETE http://localhost:6420/memories/abc123
```

### Stats

```bash
GET /memories/stats
curl http://localhost:6420/memories/stats
# → {"total_memories":2994,"total_sessions":42,"total_labels":15}
```

### Reindex

```bash
POST /memories/reindex
curl -X POST http://localhost:6420/memories/reindex
```

### Distill

```bash
POST /memories/distill
curl -X POST http://localhost:6420/memories/distill \
  -H 'Content-Type: application/json' \
  -d '{"session_id": "sess-001"}'
```

## Enhanced Retrieval Pipeline

```
Query
  │
  ▼
Classify (query type detection)
  │
  ├─ temporal    → Temporal Handler  (date-inject, time-normalize)
  ├─ preference  → Preference Handler (preference store lookup)
  ├─ aggregation → Aggregation Handler (multi-pass collect)
  ├─ multi-session → MS Handler (cross-session entity resolution)
  └─ default     → Standard Handler
  │
  ▼
Retrieve (hybrid semantic + token-overlap)
  │
  ▼
Synthesize (LLM answer generation with type-specific prompt)
  │
  ▼
Verify (judge + fallback if confidence low)
  │
  ▼
Answer
```

## Configuration

Create `config.yaml` in the working directory or set `ATLAS_CONFIG` env var:

```yaml
server:
  host: 0.0.0.0
  port: 6420

database:
  path: atlas_memory.db

embedding:
  provider: builtin     # builtin (local) | openai

llm:
  providers:
    - name: openai
      model: gpt-4.1-mini
      api_key: env:OPENAI_API_KEY
    - name: ollama
      model: qwen3.5:9b
      base_url: http://localhost:11434

search:
  default_limit: 10
  hybrid_weight: 0.7    # semantic vs token-overlap balance

enrichment:
  enabled: true
  extract_entities: true
  extract_relations: true
  extract_events: true
  extract_preferences: true
```

All configuration is optional — the server works out of the box with no config file.

## Benchmarks

### LoCoMo (502 QA pairs)

| Configuration | Judge | Hit@10 | F1 |
|---|---|---|---|
| Enhanced pipeline (Gemini 2.5 Pro) | **87.05%** | — | — |
| Production server (text-embedding-3-large + gpt-4.1-mini) | 74.7% | 29.5% | 11.9% |
| Standalone no-LLM (token-overlap only) | 43.63% | 15.94% | 7.5% |

### Production Per-Bucket Breakdown

| Bucket | Judge |
|--------|-------|
| open-domain | 91.2% |
| multi-hop | 90.6% |
| single-hop | 83.8% |
| temporal | 73.3% |
| adversarial | 24.3% |

### Three Tiers of Capability

**Standalone (43.6%)** → No LLM required, token-overlap search, valid zero-dependency fallback.

**Production (74.7%)** → OpenAI embeddings + LLM judge. Closes most of the gap on open-domain and multi-hop queries.

**Enhanced (87.05%)** → Full enriched retrieval chain with temporal handlers, preference routing, and multi-session resolution.

## Platform

| Platform | Status |
|----------|--------|
| Linux x86_64 | ✅ Available |
| Linux ARM64 | 🔜 Coming soon |
| macOS x86_64 | 🔜 Coming soon |
| macOS ARM64 (Apple Silicon) | 🔜 Coming soon |
| Windows x86_64 | 🔜 Coming soon |

## License

MIT
