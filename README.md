# Qdrant + Ollama MCP Server

Local semantic memory for Codex:

```text
Codex -> HTTP MCP server -> Ollama embeddings -> Qdrant vector DB
```

The MCP implementation is the ready-made upstream server:

https://github.com/mhalder/qdrant-mcp-server

This project only wraps it with Docker Compose, Qdrant, Ollama and `.env`
configuration.

## Services

- `qdrant`: vector database, exposed on `localhost:${QDRANT_HOST_PORT}`.
- `ollama`: local embedding provider, exposed on `localhost:${OLLAMA_HOST_PORT}`.
- `mcp-server`: ready-made `mhalder/qdrant-mcp-server`, exposed as Streamable HTTP MCP on `localhost:${MCP_HOST_PORT}/mcp`.
- `pull-embedding-model`: setup helper that pulls the configured Ollama embedding model.

## Persistent Data

Database and model files are stored in project directories:

- `./data/qdrant` -> `/qdrant/storage`
- `./data/ollama` -> `/root/.ollama`

These directories are bind-mounted into the containers, so data remains visible
and backup-friendly from the project folder.

## Start

```bash
cp .env.example .env
docker compose up -d qdrant ollama
docker compose --profile setup run --rm pull-embedding-model
docker compose up -d --build mcp-server
```

## Quick Checks

```bash
curl http://localhost:6333/healthz
curl http://localhost:11434/api/tags
curl http://localhost:3000/health
docker compose ps
```

MCP endpoint:

```text
http://localhost:3000/mcp
```

## Codex MCP Config

Add this to `~/.codex/config.toml`, then restart Codex:

```toml
[mcp_servers.qdrant-codebase]
url = "http://localhost:3000/mcp"
```

## Available Tools

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
TRANSPORT_MODE=http
HTTP_PORT=3000
MCP_HOST_PORT=3000
FRONT_PROJECT_PATH=/home/sham/PhpstormProjects/front
```
