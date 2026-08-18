## Project Goal
Deploy an MCP (Model Context Protocol) server that fetches messages from a specific NTFY topic (`ServerNotify`) to integrate with Open WebUI, enabling the LLM to access and respond to notification messages.

---

## Project Outcome
Successfully deployed two Docker containers on **[host-ip]** host:
- **Open WebUI**: Main interface at `http://[host-ip]:8080`
- **ntfy-mcp-server**: MCP server at `http://[service-host-ip]:8085` exposing messages from topic `ServerNotify`, accessible via `http://ntfy-mcp:3010/mcp`

---

## Roadblocks & Solutions

### 1. DNS Resolution Inside Containers
**Problem**: Container images often use external DNS (e.g., Google's 8.8.8.8) and cannot resolve local hostnames.

**Solution**: Use the static IP address `[service-host-ip]:8085` instead of hostname, since you control DHCP with a static IP reservation.

### 2. Container Network Isolation
**Problem**: Open WebUI and ntfy-mcp-server were in separate compose files, creating network isolation issues.

**Solution**: Consolidate both services into a single `docker-compose.yml` file so Docker creates an internal network where containers can resolve each other by service name (`ntfy-mcp`).

### 3. HTTP Transport Configuration
**Problem**: The default `stdio` transport mode only works for local processes (like Claude Desktop), not across Docker containers.

**Solution**: Set `MCP_TRANSPORT_TYPE=http` and `MCP_HTTP_HOST=0.0.0.0`. Using `127.0.0.1` would make the service inaccessible to other containers.

### 4. Port Binding
**Problem**: Need to ensure port 3010 is accessible within the Docker network but not exposed to the LAN unnecessarily.

**Solution**: Use `expose: ["3010"]` instead of `ports:` for internal-only access, keeping it private while still reachable by Open WebUI.

---

## Final Docker Compose File

Save this as `~/docker/open-webui/docker-compose.yml`:

```yaml
services:
  open-webui:
    image: ghcr.io/open-webui/open-webui:latest
    container_name: open-webui
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      WEBUI_NAME: "[host-name]"
      ENABLE_SIGNUP: "true"
      AUXILIARY_EMBEDDING_MODEL: ""
      SENTENCE_TRANSFORMERS_HOME: ""
      WEBUI_AUTH: "true"
      ENABLE_API_KEYS: "true"
      RAG_EMBEDDING_ENGINE: "ollama"
      RAG_EMBEDDING_MODEL: "[embedding-model-name]"
      RAG_OLLAMA_BASE_URL: "http://[ollama-host-ip]:11434"
      OLLAMA_BASE_URL: "http://[ollama-host-ip]:11434"
      USE_EMBEDDING_MODEL_DOCKER: "false"
      WEBUI_MARKDOWN_RENDERING: "true"
      WEBUI_ALLOW_SIGNUP: "true"
      ENABLE_RAG_LOCAL_WEB_FETCH: "True"
      HF_HOME: ""
      USE_AUXILIARY_EMBEDDING_MODEL_DOCKER: "false"
      WEBUI_ENABLE_MERMAID: "true"
      USER_PERMISSIONS_FEATURES_API_KEYS: "true"
      RAG_SYSTEM_CONTEXT: "true"
      RAG_RERANKING_MODEL: ""
      WEBUI_SECRET_KEY: "[generate-random-string]"
      USE_OLLAMA_DOCKER: "false"
      USE_CUDA_DOCKER: "false"
      USE_SLIM_DOCKER: "false"
      USE_CUDA_DOCKER_VER: "[cuda-version]"
      USE_RERANKING_MODEL_DOCKER: ""
      OPENAI_API_BASE_URL: ""
      OPENAI_API_KEY: ""
      SCARF_NO_ANALYTICS: "true"
      DO_NOT_TRACK: "true"
      VECTOR_DB: "qdrant"
      QDRANT_URI: "http://[ollama-host-ip]:6333"
      ANONYMIZED_TELEMETRY: "false"
      TZ: "[your-timezone]"
      WHISPER_MODEL: "base"
      WHISPER_MODEL_DIR: "/app/backend/data/cache/whisper/models"
      TIKTOKEN_ENCODING_NAME: "cl100k_base"
      TIKTOKEN_CACHE_DIR: "/app/backend/data/cache/tiktoken"
    volumes:
      - open-webui-data:/app/backend/data

  ntfy-mcp:
    image: ghcr.io/cyanheads/ntfy-mcp-server:latest
    container_name: ntfy-mcp-server
    restart: unless-stopped
    environment:
      MCP_TRANSPORT_TYPE: "http"
      MCP_HTTP_HOST: "0.0.0.0"
      MCP_HTTP_PORT: "3010"
      NTFY_BASE_URL: "http://[service-host-ip]:8085"
      NTFY_DEFAULT_TOPIC: "ServerNotify"

      # Add this only if ServerNotify requires authentication:
      # NTFY_AUTH_TOKEN: "[your-access-token]"

    # This makes the port reachable only by other containers in this
    # Compose project. It is not published to your LAN.
    expose:
      - "3010"

volumes:
  open-webui-data:
    external:
      name: ollama_open-webui
```

---

## Deployment Steps

### Step 1: Navigate to Directory
```bash
cd ~/docker/open-webui
```

### Step 2: Pull Images (Recommended Before Deploy)
```bash
docker compose pull
```

### Step 3: Start Containers
```bash
docker compose up -d
```

### Step 4: Verify Deployment
```bash
# Check container status
docker ps

# Expected output should show both open-webui and ntfy-mcp running
```

### Step 5: Configure Open WebUI MCP Server
1. Open the Open WebUI web interface at `http://[host-ip]:8080`
2. Navigate to **Settings** > **MCP**
3. Add a new server with these settings:
   - **Type**: `SSE` (or `HTTP`)
   - **URL**: `http://ntfy-mcp:3010/mcp`

---

## Important Notes

### Limitations
- `NTFY_DEFAULT_TOPIC=ServerNotify` sets the default topic when a tool call omits one, but does not prevent models from requesting different topics.
- The MCP server exposes publish and message-management tools (not fetch-only).
- Since both ntfy instance and Open WebUI are private, this limitation is acceptable for your use case.

### Maintenance
If you need to update either service:
```bash
# Pull latest images
docker compose pull

# Restart containers
docker compose up -d --no-recreate open-webui ntfy-mcp
```

---

**Document Created**: 2025-01-XX  
**Last Updated**: Based on current chat session  
**Rebuild Date**: Use this guide when rebuilding the server
