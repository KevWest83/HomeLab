# Runbook: Homelab AI Documentation System

## 1. Overview (What it does)
This system provides a Git-native, AI-assisted documentation pipeline. It separates the creative drafting process from the mechanical publishing and indexing process.

*   **Drafting:** Done manually in Open WebUI (Primary AI Server) with LLM assistance.
*   **Publishing:** The finished Markdown is pasted into Gitea (Secondary AI Server) and committed. Gitea acts as the manual gate and Source of Truth.
*   **Indexing:** A Gitea webhook silently triggers n8n (Secondary AI Server) to fetch the new Markdown, chunk it by headers, generate vector embeddings via Ollama, and upsert the data into Qdrant for future AI retrieval.

## 2. Architecture (How it works)
*   **Primary AI Server (<PRIMARY_AI_SERVER_IP>):** Hosts Open WebUI and the underlying NFS mass storage (`/mnt/primary-ai-server-6tb-a/HomeLab/`).
*   **Secondary AI Server (<SECONDARY_AI_SERVER_IP>):** Hosts the backend pipeline:
    *   **Gitea (Port 3001):** The Git frontend and webhook initiator. Uses SQLite3 for lightweight local persistence.
    *   **n8n (Port 5678):** The automation engine.
    *   **Ollama (Port 11434):** Generates embeddings using `qwen3-embedding:8b`.
    *   **Qdrant (Port 6333):** The vector database storing the `runbooks` collection (4096 dimensions, Cosine distance).
*   **Production Server:** (Not directly in this pipeline, but referenced for broader homelab backups and production workloads).

---

## 3. Rebuild Instructions

If the Secondary AI Server is ever lost or the containers need to be rebuilt from scratch, follow these steps.

### Step A: Rebuild Gitea
1. Create the directory and `docker-compose.yml` on the Secondary AI Server:
   ```bash
   mkdir -p ~/docker/gitea
   cd ~/docker/gitea
   nano docker-compose.yml
   ```
2. Paste the Compose configuration:
   ```yaml
   version: "3"
   services:
     server:
       image: gitea/gitea:1.21
       container_name: gitea
       restart: always
       environment:
         - USER=admin
         - GITEA__database__DB_TYPE=sqlite3
         - GITEA__server__DOMAIN=secondary-ai-server.local
         - GITEA__server__SSH_PORT=2222
         - GITEA__server__HTTP_PORT=3000
       ports:
         - "3001:3000" # 3001 avoids conflict with Karakeep
         - "2222:22"
       volumes:
         - ./data:/data
         - ./config:/config
         - ./logs:/logs
   ```
3. Deploy: `docker compose up -d`
4. **CRITICAL - Fix SSRF Webhook Blocking:** By default, Gitea blocks webhooks to local IPs (like n8n). You must allow it:
   ```bash
   nano ~/docker/gitea/data/gitea/conf/app.ini
   ```
   Add this to the bottom of the file:
   ```ini
   [webhook]
   ALLOWED_HOST_LIST = *
   ```
   Restart Gitea: `docker restart gitea`

### Step B: Rebuild the n8n Indexing Pipeline
Create a new workflow in n8n with the following node sequence:

1. **Webhook:** Method `POST`, Path `gitea-docs-push`, Respond `Immediately`.
2. **Code (Extract File Paths):** Parses the Gitea commit payload for `.md` files.
   ```javascript
   const filesToIndex = [];
   const commits = $input.item.json.body.commits || $input.item.json.commits;
   for (const commit of commits) {
     const allFiles = [...(commit.added || []), ...(commit.modified || [])];
     for (const file of allFiles) {
       if (file.endsWith('.md')) {
         filesToIndex.push({
           json: { filepath: file, commit_id: commit.id, title: file.split('/').pop().replace('.md', '') }
         });
       }
     }
   }
   return filesToIndex;
   ```
3. **HTTP Request (Fetch Raw Markdown):** 
   * Method: `GET`
   * URL: `http://<SECONDARY_AI_SERVER_IP>:3001/KevWest83/homelab-docs/raw/commit/{{ $json.commit_id }}/{{ $json.filepath }}`
   * Response Format: `Text` (Prevents n8n from crashing trying to parse Markdown as JSON).
4. **Code (Chunking):** Splits the downloaded `data` by `## ` headers.
   ```javascript
   const chunks = [];
   const httpItems = $input.all();
   const metaItems = $items("Extract File Paths"); // Must match the name of Node 2

   for (let i = 0; i < httpItems.length; i++) {
     const content = httpItems[i].json.data; 
     const title = metaItems[i].json.title || "Untitled Document";
     const filePath = metaItems[i].json.filepath || "unknown";

     if (!content || typeof content !== 'string') continue;

     const lines = content.split('\n');
     let currentHeader = title;
     let currentLines = [];

     for (const line of lines) {
       if (line.startsWith('## ')) {
         if (currentLines.length > 0) {
           chunks.push({ header: currentHeader, text: currentLines.join('\n').trim(), source_file: filePath, source_title: title });
         }
         currentHeader = line.replace(/^##\s+/, '').trim();
         currentLines = [line];
       } else {
         currentLines.push(line);
       }
     }
     if (currentLines.length > 0) {
       chunks.push({ header: currentHeader, text: currentLines.join('\n').trim(), source_file: filePath, source_title: title });
     }
   }
   return chunks.map(c => ({ json: c }));
   ```
5. **HTTP Request (Ollama Embed):** 
   * Method: `POST`, URL: `http://<SECONDARY_AI_SERVER_IP>:11434/api/embed`
   * Body: `{"model": "qwen3-embedding:8b", "input": "={{ $json.text }}"}`
6. **Crypto:** Operation `Generate`, Type `UUID`, Property Name `point_id`.
7. **Code (Assemble Payload):**
   ```javascript
   const chunk = $('Chunking Node Name').item.json; // Update to match Node 4's exact name
   const embedding = $json.embeddings[0];
   const pointId = $json.point_id;

   return {
     json: {
       id: pointId,
       vector: embedding,
       payload: { text: chunk.text, header: chunk.header, indexed_at: new Date().toISOString() }
     }
   };
   ```
8. **HTTP Request (Qdrant Upsert):**
   * Method: `PUT`, URL: `http://<SECONDARY_AI_SERVER_IP>:6333/collections/runbooks/points`
   * Body: `{"points": [ { "id": "={{$json.id}}", "vector": "={{$json.vector}}", "payload": "={{$json.payload}}" } ]}`

---

## 4. Troubleshooting

### Gitea Webhook is failing to trigger n8n
* **Symptom:** Gitea webhook logs show `deny '<SECONDARY_AI_SERVER_IP>'`.
* **Cause:** Gitea's SSRF protection is blocking calls to local network IPs.
* **Fix:** Ensure `ALLOWED_HOST_LIST = *` is present under `[webhook]` in `~/docker/gitea/data/gitea/conf/app.ini` and restart the Gitea container.

### n8n throws a 404 Error on the "Fetch Raw Markdown" node
* **Symptom:** n8n pipeline starts, but fails at the HTTP GET request with a 404 Page Not Found (which is actually an Access Denied error).
* **Cause:** The Gitea repository is set to Private, meaning the raw file URL requires an API token to read.
* **Fix:** Ensure the `homelab-docs` repository in Gitea is set to **Public**. Since Gitea is only exposed to the internal homelab network, a public repo is safe and bypasses the need to manage API tokens in n8n.

### Qdrant Upsert Fails (Dimension Mismatch)
* **Symptom:** The final Qdrant HTTP node throws a 400 Bad Request regarding vector sizes.
* **Cause:** The Ollama embedding model used in n8n does not match the dimensions of the Qdrant collection.
* **Fix:** Verify that the Ollama HTTP node is explicitly calling `qwen3-embedding:8b` (which outputs 4096 dimensions). If you changed the model, you must wipe the Qdrant collection and re-index, as Qdrant collections are locked to the dimension size they were created with.

### n8n "Convert to File" or "JSON Body" errors
* **Symptom:** n8n complains about "Bad control character in string literal".
* **Cause:** You are trying to pass raw Markdown (which contains unescaped newlines) directly into a raw JSON text field.
* **Fix:** Always use n8n's **"Using Fields Below"** option in HTTP Request nodes rather than hand-typing JSON strings. This forces n8n to safely escape the Markdown newlines before sending the payload.
