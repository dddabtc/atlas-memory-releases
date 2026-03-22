# Atlas Memory

> Self-hosted long-term memory server for AI agents. Single binary, zero dependencies.

![LongMemEval](https://img.shields.io/badge/LongMemEval--s-88.18%25_%232_globally-brightgreen)
![LoCoMo](https://img.shields.io/badge/LoCoMo-87.05%25_Judge-blue)
![License](https://img.shields.io/badge/license-MIT-green)

## What is Atlas Memory?

Atlas Memory is a self-hosted long-term memory server for AI agents. It stores, indexes, and retrieves conversation memories using hybrid search (semantic embeddings + token overlap). Run a single binary and integrate via REST API — no cloud, no vendor lock-in.

## How It Compares

| System | LongMemEval-s | Notes |
|--------|--------------|-------|
| HydraDB | 90.79% | #1 globally |
| **Atlas Memory** | **88.18%** | **#2 globally** |
| Supermemory | 85.20% | |
| Zep | 71.20% | |
| Full-context GPT-4o | 60.20% | |
| Mem0-oss | 29.07% | |

## Installation

### Download Binary

```bash
# Linux x86_64
curl -L -o atlas-memory \
  https://github.com/dddabtc/atlas-memory-releases/releases/download/v4.2.4-rs/atlas-memory-linux-x86_64
chmod +x atlas-memory
```

More platforms (macOS, Windows, Linux ARM64) coming in the next release.

### Run

```bash
# Start with default settings (port 6420, SQLite in current directory)
./atlas-memory

# Custom port
./atlas-memory --port 8080

# Custom host + port
./atlas-memory --host 0.0.0.0 --port 6420

# With config file
./atlas-memory --config /path/to/config.yaml
```

### Systemd Service (Linux)

```bash
sudo tee /etc/systemd/system/atlas-memory.service << 'EOF'
[Unit]
Description=Atlas Memory Server
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/atlas-memory --port 6420
WorkingDirectory=/var/lib/atlas-memory
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo mkdir -p /var/lib/atlas-memory
sudo cp atlas-memory /usr/local/bin/
sudo systemctl enable --now atlas-memory
```

### Docker (coming soon)

```bash
docker run -p 6420:6420 -v atlas-data:/data dddabtc/atlas-memory:latest
```

### Verify Installation

```bash
curl http://localhost:6420/health
# → {"status":"ok","version":"4.2.4-rs","memory_count":0}
```

## CLI Reference

```
atlas-memory [OPTIONS]

Options:
  -p, --port <PORT>      Server port [default: 6420]
  -H, --host <HOST>      Bind address [default: 0.0.0.0]
  -c, --config <FILE>    Path to config.yaml
  -d, --db <PATH>        Database file path [default: atlas_memory.db]
  -h, --help             Print help
  -V, --version          Print version
```

### Examples

```bash
# Development — default settings
./atlas-memory

# Production — custom config
./atlas-memory --config /etc/atlas-memory/config.yaml --port 6420

# Testing — in-memory (ephemeral)
./atlas-memory --db :memory: --port 9999

# Multiple instances
./atlas-memory --port 6420 --db production.db &
./atlas-memory --port 6421 --db staging.db &
```

## API Reference

Base URL: `http://localhost:6420`

### Health Check

```bash
GET /health
```

```bash
$ curl http://localhost:6420/health
{
  "status": "ok",
  "version": "4.2.4-rs",
  "memory_count": 2994,
  "has_embeddings": false,
  "llm_available": false,
  "embedding_provider": "none"
}
```

### Create Memory

```bash
POST /memories
Content-Type: application/json
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `content` | string | ✅ | Memory content text |
| `title` | string | | Optional title |
| `session_id` | string | | Group by conversation session |
| `labels` | string[] | | Tags for filtering |

```bash
$ curl -X POST http://localhost:6420/memories \
  -H 'Content-Type: application/json' \
  -d '{
    "content": "User prefers dark roast coffee, no sugar, with oat milk.",
    "title": "Coffee preference",
    "session_id": "chat-2024-01-15",
    "labels": ["preference", "food"]
  }'
{
  "id": "a1b2c3d4-...",
  "content": "User prefers dark roast coffee...",
  "created_at": "2024-01-15T10:30:00Z"
}
```

### Search Memories

```bash
POST /memories/search
Content-Type: application/json
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `query` | string | ✅ | Search query |
| `limit` | int | | Max results [default: 10] |
| `labels` | string[] | | Filter by labels |
| `session_id` | string | | Filter by session |
| `threshold` | float | | Min similarity score |

```bash
$ curl -X POST http://localhost:6420/memories/search \
  -H 'Content-Type: application/json' \
  -d '{"query": "what kind of coffee does the user like?", "limit": 5}'
{
  "results": [
    {
      "id": "a1b2c3d4-...",
      "content": "User prefers dark roast coffee, no sugar, with oat milk.",
      "score": 0.847,
      "title": "Coffee preference"
    }
  ]
}
```

### List Memories

```bash
GET /memories?limit=20&offset=0
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `limit` | int | 20 | Page size |
| `offset` | int | 0 | Page offset |

```bash
$ curl 'http://localhost:6420/memories?limit=2&offset=0'
{
  "memories": [...],
  "total": 2994,
  "limit": 2,
  "offset": 0
}
```

### Get Memory

```bash
GET /memories/{id}
```

```bash
$ curl http://localhost:6420/memories/a1b2c3d4-...
{
  "id": "a1b2c3d4-...",
  "content": "User prefers dark roast coffee...",
  "title": "Coffee preference",
  "labels": ["preference", "food"],
  "session_id": "chat-2024-01-15",
  "created_at": "2024-01-15T10:30:00Z",
  "updated_at": "2024-01-15T10:30:00Z"
}
```

### Update Memory

```bash
PATCH /memories/{id}
Content-Type: application/json
```

```bash
$ curl -X PATCH http://localhost:6420/memories/a1b2c3d4-... \
  -H 'Content-Type: application/json' \
  -d '{"content": "User now prefers light roast.", "labels": ["preference", "food", "updated"]}'
```

### Delete Memory

```bash
DELETE /memories/{id}
```

```bash
$ curl -X DELETE http://localhost:6420/memories/a1b2c3d4-...
{"status": "deleted"}
```

### Stats

```bash
GET /memories/stats
```

```bash
$ curl http://localhost:6420/memories/stats
{
  "total_memories": 2994,
  "total_sessions": 42,
  "total_labels": 15,
  "label_counts": [
    {"label": "preference", "count": 230},
    {"label": "project", "count": 180}
  ],
  "oldest_memory": "2024-01-01T00:00:00Z",
  "newest_memory": "2024-03-22T04:00:00Z"
}
```

### Reindex Embeddings

```bash
POST /memories/reindex
```

Re-computes embeddings for all memories. Use after changing embedding provider.

```bash
$ curl -X POST http://localhost:6420/memories/reindex
{"status": "ok", "reindexed": 2994}
```

### Distill Session

```bash
POST /memories/distill
Content-Type: application/json
```

Summarize and consolidate memories from a session.

```bash
$ curl -X POST http://localhost:6420/memories/distill \
  -H 'Content-Type: application/json' \
  -d '{"session_id": "chat-2024-01-15"}'
```

## Integration Examples

### Python

```python
import requests

BASE = "http://localhost:6420"

# Store
requests.post(f"{BASE}/memories", json={
    "content": "User's birthday is March 15th",
    "labels": ["personal"]
})

# Search
results = requests.post(f"{BASE}/memories/search", json={
    "query": "when is the user's birthday?",
    "limit": 5
}).json()

for r in results["results"]:
    print(f"[{r['score']:.2f}] {r['content']}")
```

### JavaScript / Node.js

```javascript
const BASE = "http://localhost:6420";

// Store
await fetch(`${BASE}/memories`, {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    content: "User prefers dark mode in all apps",
    labels: ["preference", "ui"]
  })
});

// Search
const res = await fetch(`${BASE}/memories/search`, {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ query: "UI preferences", limit: 5 })
});
const { results } = await res.json();
```

### curl Cheat Sheet

```bash
# Store
curl -X POST localhost:6420/memories \
  -H 'Content-Type: application/json' \
  -d '{"content":"remember this"}'

# Search
curl -X POST localhost:6420/memories/search \
  -H 'Content-Type: application/json' \
  -d '{"query":"what to remember","limit":5}'

# List (paginated)
curl 'localhost:6420/memories?limit=10&offset=0'

# Get one
curl localhost:6420/memories/{id}

# Update
curl -X PATCH localhost:6420/memories/{id} \
  -H 'Content-Type: application/json' \
  -d '{"content":"updated content"}'

# Delete
curl -X DELETE localhost:6420/memories/{id}

# Stats
curl localhost:6420/memories/stats

# Health
curl localhost:6420/health
```

## Configuration

Create `config.yaml` in the working directory or set `ATLAS_CONFIG` env var:

```yaml
server:
  host: 0.0.0.0
  port: 6420

database:
  path: atlas_memory.db      # SQLite file path

embedding:
  provider: builtin           # builtin (local) | openai
  # OpenAI embedding config (optional)
  # openai_model: text-embedding-3-large
  # openai_api_key: env:OPENAI_API_KEY

llm:
  # Optional: LLM providers for enrichment (query classification, entity extraction)
  # Server works fine without any LLM configured
  providers:
    - name: openai
      model: gpt-4.1-mini
      api_key: env:OPENAI_API_KEY
    - name: ollama
      model: qwen3.5:9b
      base_url: http://localhost:11434

search:
  default_limit: 10
  hybrid_weight: 0.7          # 0.0 = pure token-overlap, 1.0 = pure semantic

enrichment:
  enabled: true               # Enable write-path enrichment
  extract_entities: true
  extract_relations: true
  extract_events: true
  extract_preferences: true
```

**All configuration is optional.** The server works out of the box with zero config — SQLite in current directory, token-overlap search, no LLM required.

### Environment Variables

| Variable | Description |
|----------|-------------|
| `ATLAS_CONFIG` | Path to config.yaml |
| `ATLAS_PORT` | Override server port |
| `OPENAI_API_KEY` | OpenAI API key (for embeddings/LLM) |

### Config Priority

`--port` CLI flag → `config.yaml` → `ATLAS_PORT` env → `6420` default

## Benchmarks

| Benchmark | Score | Rank |
|-----------|-------|------|
| LongMemEval-s (499Q) | **88.18%** | #2 globally |
| LoCoMo Judge (502Q) | **87.05%** | Enhanced pipeline |
| LoCoMo Production | **74.7%** | Server mode |
| LoCoMo Standalone | **43.6%** | No LLM |

## Features

- 🔍 **Hybrid search** — semantic embeddings + token-overlap, works without any LLM
- 📊 **Benchmark-grade** — 88.18% LongMemEval (#2 globally), 87.05% LoCoMo
- 🦀 **Single binary** — no runtime dependencies, no Python, no Docker
- 💾 **SQLite** — zero-config embedded database
- ⚡ **LLM enrichment** — optional OpenAI/Ollama for entity extraction, query classification
- 🔄 **Graceful degradation** — no LLM → regex + token-overlap, never crashes
- 🏷️ **Labels & sessions** — organize memories with tags and conversation groups
- 📄 **Pagination** — list memories with limit/offset
- 🔧 **Reindex** — re-compute embeddings after config change
- 📝 **Distill** — summarize and consolidate session memories

## Platform

| Platform | Status |
|----------|--------|
| Linux x86_64 | ✅ Available |
| Linux ARM64 | 🔜 Coming soon |
| macOS ARM64 (Apple Silicon) | 🔜 Coming soon |
| macOS x86_64 | 🔜 Coming soon |
| Windows x86_64 | 🔜 Coming soon |

## License

MIT
