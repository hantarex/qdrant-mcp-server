# Qdrant + Ollama + Redis MCP Services

Local semantic memory for Codex:

```text
Codex -> HTTP MCP server -> Ollama embeddings -> Qdrant vector DB
Codex/agents -> stdio Redis MCP server -> Redis
```

The Qdrant MCP implementation is the ready-made upstream server:

https://github.com/mhalder/qdrant-mcp-server

Redis is exposed as a normal TCP service for the official stdio Redis MCP
server:

https://github.com/redis/mcp-redis

This project wraps these services with Docker Compose and `.env` configuration.

## Services

- `qdrant`: vector database, exposed on `localhost:${QDRANT_HOST_PORT}`.
- `qdrant` gRPC: exposed on `localhost:${QDRANT_GRPC_HOST_PORT}`.
- `ollama`: local embedding provider, exposed on `localhost:${OLLAMA_HOST_PORT}`.
- `mcp-server`: ready-made `mhalder/qdrant-mcp-server`, exposed as Streamable HTTP MCP on `localhost:${MCP_HOST_PORT}/mcp`.
- `redis`: Redis service for agent transport experiments, exposed on `localhost:${REDIS_HOST_PORT}`.
- `pull-embedding-model`: setup helper that pulls the configured Ollama embedding model.

## Persistent Data

Database and model files are stored in project directories:

- `./data/qdrant` -> `/qdrant/storage`
- `./data/ollama` -> `/root/.ollama`
- `./data/redis` -> `/data`

These directories are bind-mounted into the containers, so data remains visible
and backup-friendly from the project folder.

## Start

```bash
cp .env.example .env
docker compose up -d qdrant ollama redis
docker compose --profile setup run --rm pull-embedding-model
docker compose up -d --build mcp-server
```

## Quick Checks

These commands assume the default host ports from `.env.example`. If you change
any `*_HOST_PORT` value, use the matching host port in the command or Codex MCP
config.

```bash
curl http://localhost:6333/healthz
curl http://localhost:11434/api/tags
curl http://localhost:3000/health
docker compose exec redis redis-cli ping
docker compose ps
```

MCP endpoint:

```text
http://localhost:${MCP_HOST_PORT}/mcp
```

Redis TCP endpoint:

```text
redis://localhost:${REDIS_HOST_PORT}/0
```

## Codex MCP Config

Add this Qdrant MCP server to `~/.codex/config.toml`, then restart Codex:

```toml
[mcp_servers.qdrant-codebase]
url = "http://localhost:3000/mcp"
```

If `MCP_HOST_PORT` is changed from the default `3000`, update this URL to the
same host port.

The official Redis MCP server currently uses stdio transport. Add it separately
for every Codex/agent runtime that should access the shared Redis bus:

```toml
[mcp_servers.redis-agent-bus]
command = "docker"
args = [
  "run", "--rm", "-i",
  "--network", "host",
  "-e", "REDIS_HOST=localhost",
  "-e", "REDIS_PORT=6379",
  "mcp/redis@sha256:e886a7e9990a084a20d46adcea8e7c89d539271f1b453ffabdce4c3f96f27fa0"
]
```

This launches a short-lived stdio MCP process per Codex/agent runtime. The MCP
transport is stdin/stdout; `localhost:6379` is only the Redis TCP endpoint used
by that process. Do not add a fixed Docker `--name` here, because multiple
agents may start their own Redis MCP process at the same time. The Docker image
is pinned by digest because Docker Hub currently publishes the official
`mcp/redis` image without a versioned tag. If `REDIS_HOST_PORT` is changed from
the default `6379`, update `REDIS_PORT` in this config too.

If you use `uvx` instead of Docker, the equivalent stdio MCP config is:

```toml
[mcp_servers.redis-agent-bus]
command = "uvx"
args = [
  "--from",
  "redis-mcp-server==0.5.0",
  "redis-mcp-server",
  "--url",
  "redis://localhost:6379/0"
]
```

## Qdrant MCP Tools

- `create_collection`
- `list_collections`
- `get_collection_info`
- `add_documents`
- `semantic_search`
- `hybrid_search`
- `index_codebase`
- `search_code`
- `reindex_changes`
- `get_index_status`
- `index_git_history`
- `search_git_history`

## Redis MCP Tools

The Redis MCP tools are provided by the separate stdio `redis-agent-bus`
configuration above. The official Redis MCP server includes string/hash/list/set
operations, Pub/Sub tools, and Redis Streams tools such as `xadd`,
`xreadgroup`, and `xack`.

## Index The Front Project

After adding the MCP config and restarting Codex, ask Codex to call:

```text
index_codebase(path="/workspace/front")
```

The compose file mounts `/home/sham/PhpstormProjects/front` read-only as `/workspace/front`.

Default indexing skips common heavy/generated folders:

- `.git`
- `.idea`
- `.angular`
- `dist`
- `node_modules`
- `coverage`
- `vendor`
- `tmp`

## Environment

Important `.env` values:

```env
QDRANT_URL=http://qdrant:6333
QDRANT_COLLECTION=front-codebase
EMBEDDING_PROVIDER=ollama
EMBEDDING_MODEL=nomic-embed-text
EMBEDDING_BASE_URL=http://ollama:11434
MCP_HOST_PORT=3000
REDIS_HOST_PORT=6379
REDIS_IMAGE=redis:8.10.1-alpine
FRONT_PROJECT_PATH=/home/sham/PhpstormProjects/front
```
