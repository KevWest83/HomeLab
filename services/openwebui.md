## Open WebUI

**Host:** Primary AI Server
**Deployment:** Docker (Docker Compose)
**Port:** 8080

### What It Is
Open WebUI is a self-hosted web interface for interacting with local
LLM models via Ollama. It provides a ChatGPT-style frontend with
support for multiple models, system prompts, knowledge bases, and
tool integrations.

### Architecture
Open WebUI runs in Docker and connects to Ollama running bare metal
on the same host via the local network. This hybrid approach keeps
the inference engine close to the GPU while the frontend remains
containerized and easy to update.

### Key Features in Use
- Multiple model profiles with custom system prompts
- Knowledge base (RAG) integration for document retrieval
- Tool integrations including web search and image generation
- Image generation via ComfyUI integration
- Workflow automation via n8n webhook integration

### Connected Services
- Ollama (bare metal) — LLM inference backend
- ComfyUI (Docker) — image generation
- SearXNG — self-hosted web search
- n8n — workflow automation via webhook

### Local Access
Accessible via local domain (*.home.arpa) using Pi-hole for internal
DNS resolution and Caddy as a reverse proxy. Services are reachable
by hostname rather than IP and port, mirroring how production
environments handle internal service routing.

### Update Method
Standard Docker Compose pull and recreate:
docker compose pull
docker compose up -d
