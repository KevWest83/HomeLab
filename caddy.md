## Caddy

**Host:** Security Server
**Deployment:** Docker (Docker Compose)
**Role:** Reverse proxy, local HTTPS termination

### What It Is
Caddy is a modern open-source web server used here as a reverse proxy.
It sits in front of selected self-hosted services and handles HTTPS
termination, routing incoming requests by hostname to the correct
backend service. This allows services to be accessed by clean local
domain names rather than IP addresses and port numbers.

### Architecture
Caddy receives HTTPS requests for *.home.arpa domains and forwards
them to the appropriate backend service on the local network. Pi-hole
resolves the *.home.arpa hostnames to Caddy's address, and Caddy
routes the request to the correct service based on the hostname.

The full request chain for a proxied service:

Client → Pi-hole (DNS resolution) → Caddy (HTTPS + routing) → Backend service

### Services Currently Proxied

| Service     | Local Domain                    |
|-------------|---------------------------------|
| Vaultwarden | vaultwarden.home.arpa           |
| Open WebUI  | openwebui.home.arpa             |
| Karakeep    | karakeep.home.arpa              |
| ntfy        | ntfy.home.arpa                  |
| SearXNG     | searxng.home.arpa               |

### TLS Certificates
Caddy manages self-signed TLS certificates for all proxied services.
A local certificate authority was created and the CA certificate is
manually distributed to and installed on trusted devices. This allows
all proxied services to use proper HTTPS with no browser warnings
rather than relying on users to accept security exceptions.

This approach mirrors how enterprise internal PKI infrastructure works —
a trusted internal CA issues certificates for internal services rather
than relying on public certificate authorities for private infrastructure.

### Configuration
Caddy is configured via a Caddyfile — a simple, human-readable
configuration format. Each proxied service has a corresponding block
defining its hostname and backend address.

### Design Principles
- Clean hostname-based access for all proxied services
- Proper GAINS on all proxied services — no browser warnings
- Self-signed CA keeps certificate management fully local
- No dependency on public certificate authorities for internal services
- New services can be added to the proxy with minimal configuration

### Update Method
Standard Docker Compose pull and recreate:
docker compose pull
docker compose up -d
