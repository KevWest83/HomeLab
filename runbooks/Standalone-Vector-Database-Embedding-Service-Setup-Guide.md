```markdown
# Standalone Vector Database & Embedding Service Setup Guide

## Overview

By default, Open WebUI utilizes an embedded ChromaDB instance and handles document embedding locally. Decoupling the vector database and embedding model offloads compute tasks to a dedicated node and ensures stored vectors remain independent of Open WebUI updates or migrations.

This guide walks through deploying **Qdrant** in Docker alongside **Ollama** running a dedicated embedding model on a dedicated compute node (`vector-node`), and connecting an existing Open WebUI instance (`app-node`) to it.

```
+---------------------------------+        +-----------------------------------+
|            app-node             |        |            vector-node            |
|  (Open WebUI / Web Interface)   |        |    (Compute & Vector Engine)      |
|                                 |        |                                   |
|  +---------------------------+  |        |  +-----------------------------+  |
|  | Open WebUI                |  |        |  | Qdrant Container           |  |
|  | Configured to use remote  |----Search--->| Port 6333                     |  |
|  | embedding & vector DB     |  |        |  +-----------------------------+  |
|  +---------------------------+  |        |                                   |
|               |                 |        |  +-----------------------------+  |
|               +---Embed Query--------+--->| Ollama Service                |  |
|                                 |    |   | Port 11434                    |  |
+---------------------------------+    |   | (Qwen 3 Embedding 8B)          |  |
                                       |   +-----------------------------+  |
                                       +-----------------------------------+
```

---

## Prerequisites

* **Compute Node (`vector-node`):**
  * Ubuntu Server LTS (or equivalent Linux distribution)
  * Docker Engine and Docker Compose plugin installed
  * Ollama installed and configured to listen on network interfaces (`0.0.0.0`)
  * GPU with sufficient VRAM if running larger embedding models (e.g., 8B parameters)
* **Application Node (`app-node`):**
  * Open WebUI up and running

---

## Step 1: Deploy the Embedding Model on the Compute Node

1. SSH into `vector-node`:
   ```bash
   ssh user@<VECTOR_NODE_IP>
   ```

2. Ensure Ollama is configured to bind to `0.0.0.0` (or your specific private interface) in its service configuration (`/etc/systemd/system/ollama.service.d/override.conf`):
   ```ini
   [Service]
   Environment="OLLAMA_HOST=0.0.0.0:11434"
   ```

3. Reload and restart Ollama if changes were made:
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl restart ollama
   ```

4. Pull the embedding model (e.g., `qwen3-embedding:8b` or your preferred embedding model):
   ```bash
   ollama pull qwen3-embedding:8b
   ```

5. Verify the model is pulled and available:
   ```bash
   ollama list
   ```

---

## Step 2: Deploy Qdrant Vector Database

1. On `vector-node`, create a dedicated directory structure for Qdrant:
   ```bash
   mkdir -p ~/docker/qdrant/data
   cd ~/docker/qdrant
   ```

2. Create a `docker-compose.yml` file:

   ```yaml
   services:
     qdrant:
       image: qdrant/qdrant:v1.10.0
       container_name: qdrant
       restart: unless-stopped
       ports:
         - "6333:6333" # REST API
         - "6334:6334" # gRPC API
       volumes:
         - ./data:/qdrant/storage:z
       environment:
         - QDRANT__SERVICE__HTTP_PORT=6333
   ```

3. Start the container:
   ```bash
   docker compose up -d
   ```

4. Check that Qdrant is running and responding:
   ```bash
   curl http://localhost:6333/readyz
   ```
   *Expected response: `all subsystems are ready`*

---

## Step 3: Network Verification

Before configuring Open WebUI, test cross-node reachability from `app-node` to `vector-node`.

1. Log into `app-node`:
   ```bash
   ssh user@<APP_NODE_IP>
   ```

2. Test reachability to Ollama's embedding API endpoint:
   ```bash
   curl http://<VECTOR_NODE_IP>:11434/api/tags
   ```

3. Test reachability to Qdrant's REST API endpoint:
   ```bash
   curl http://<VECTOR_NODE_IP>:6333/readyz
   ```

---

## Step 4: Configure Open WebUI

Update Open WebUI's settings via the web interface to switch from internal defaults to the external services.

1. Open **Open WebUI** -> **Admin Panel** -> **Settings** -> **Documents**.
2. **Embedding Model Settings:**
   * **Embedding Engine:** Select `Ollama`.
   * **Ollama API Base URL:** `http://<VECTOR_NODE_IP>:11434`
   * **Embedding Model:** `qwen3-embedding:8b`
3. **Vector Database Settings:**
   * **Vector DB Client:** Select `Qdrant`.
   * **Qdrant Host / URL:** `http://<VECTOR_NODE_IP>:6333`
   * *(Optional)* **API Key:** Leave blank unless Qdrant authentication was enabled.
4. Save settings.

---

## Step 5: Verification & End-to-End Test

1. In Open WebUI, navigate to **Workspace** -> **Knowledge**.
2. Create a new Knowledge Base collection and upload a test document (`test.txt` or `test.pdf`).
3. Confirm document processing completes without network or embedding timeouts.
4. Open a new chat session, attach the Knowledge Base, and ask a question grounded in the test document's contents.
5. Verify Qdrant collection creation on `vector-node`:
   ```bash
   curl http://<VECTOR_NODE_IP>:6333/collections
   ```
   *You should see a collection generated by Open WebUI listed in the JSON response.*
```
