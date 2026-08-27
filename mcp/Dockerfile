FROM node:22-bookworm-slim

WORKDIR /opt

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        ca-certificates \
        git \
        python3 \
        make \
        g++ \
    && rm -rf /var/lib/apt/lists/*

ARG QDRANT_MCP_REPO=https://github.com/mhalder/qdrant-mcp-server.git
ARG QDRANT_MCP_REF=v3.3.5

RUN git clone --depth 1 --branch "$QDRANT_MCP_REF" "$QDRANT_MCP_REPO" qdrant-mcp-server

WORKDIR /opt/qdrant-mcp-server

RUN npm ci && npm run build

ENTRYPOINT ["node", "/opt/qdrant-mcp-server/build/index.js"]
