## SearXNG

**Host:** Security Server
**Deployment:** Docker Compose
**Port:** 8082 (HTTP internally — Caddy handles HTTPS termination)
**Access:** https://searxng.home.arpa (via Caddy reverse proxy, local only)
**Role:** Self-hosted metasearch engine

### What It Is
SearXNG is a self-hosted metasearch engine that aggregates results
from multiple search engines without tracking queries or exposing
search history to third parties. It provides private web search
for both direct browser use and as a search backend for AI
inference via Open WebUI.

### Why Self-Hosted
Commercial search engines track every query and build detailed
profiles of search behavior. Self-hosting SearXNG means search
queries stay local — no search history is stored by a third party,
no profiling occurs, and no data leaves the local network.
Consistent with the privacy-first philosophy applied across the
entire stack.

### Architecture
SearXNG runs in Docker Compose on the Security Server. Caddy handles
HTTPS termination — SearXNG runs HTTP internally on port 8082.
Configuration is managed via a bind-mounted config directory,
allowing search engine sources and behavior to be customized without
modifying the container.

### Usage

**Direct browser access**
Accessible at https://searxng.home.arpa for private web searches
from any device on the local network only. Not accessible remotely
via WireGuard — local use only.

**Open WebUI integration**
SearXNG serves as the web search backend for Open WebUI, enabling
AI models to search the web during conversations without routing
queries through commercial search APIs. This keeps the entire
AI workflow — inference, search, and results — local and private.

Open WebUI connects to SearXNG via local network address using
the format: http://[host]:8082/search?q=<query>
NOTE: Direct IP used for Open WebUI integration rather than the
home.arpa domain to avoid TLS/DNS resolution issues between
Docker containers.

### Design Principles
- No search query logging or tracking
- No third-party search API keys required
- Aggregates multiple search sources behind a single private interface
- Config managed via bind mount for easy customization
- HTTPS handled by Caddy via local PKI
- Dual use — browser search and AI search backend from single instance
- Local network access only

### Update Method
Standard Docker Compose pull and recreate:
docker compose pull
docker compose up -d
