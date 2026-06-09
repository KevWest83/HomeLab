## Karakeep

**Host:** Primary AI Server
**Deployment:** Docker Compose
**Port:** 3000
**Access:** karakeep.home.arpa
**Role:** Self-hosted bookmark manager

### What It Is
Karakeep is a privacy-focused bookmarking application that runs
entirely on local infrastructure. Bookmarks are saved, organised,
and searchable through a web interface accessible from any device
on the network. No browser account, no cloud sync, no third-party
dependency.

Karakeep runs as a two-container stack. The core application handles
the web interface and bookmark management. Meilisearch runs alongside
it as a dedicated full-text search engine, powering internal search
without any external API.

| Container            | Image                                      | Role                        |
|----------------------|--------------------------------------------|-----------------------------|
| karakeep             | ghcr.io/karakeep-app/karakeep:release      | Core application            |
| karakeep-meilisearch | getmeili/meilisearch:v1.13.3               | Full-text search engine     |

### Architecture
Request chain:

Client → Pi-hole (DNS) → Caddy (reverse proxy) → Karakeep (port 3000)

Karakeep is resolved via Pi-hole local DNS and proxied through Caddy
on the Security Server. Meilisearch is internal to the stack and not
exposed outside of it.

### Design Decisions

**Why self-host bookmarks**
Browser-native bookmarking ties data to a specific browser and in
most cases a cloud sync account. Karakeep decouples bookmarks from
the browser entirely, making it straightforward to switch browsers
without data loss and removing reliance on third-party sync services.
Recent high-profile data breaches reinforced the decision to keep
personal data off cloud platforms where possible.

**Why Karakeep**
Karakeep is privacy-focused by design, actively maintained, and ships
with a full-text search engine as part of its stack. No external
search API, no telemetry, no cloud dependency. It fits the broader
homelab philosophy of owning the full stack.

**Why the Primary AI Server**
Karakeep includes AI-assisted functionality. Placing it on the
Primary AI Server keeps that integration available for future testing
without requiring a service migration later.

### Connected Services

| Service  | Role                                          |
|----------|-----------------------------------------------|
| Caddy    | Reverse proxy, HTTPS via internal PKI         |
| Pi-hole  | Local DNS resolution for karakeep.home.arpa   |

### Update Method
```bash
docker compose pull && docker compose up -d
Run from the Karakeep directory on the Primary AI Server.
