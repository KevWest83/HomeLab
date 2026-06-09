## Vaultwarden


**Host:** Security Server
**Deployment:** Docker (Docker Compose)
**Access:** https://vaultwarden.home.arpa

### What It Is
Vaultwarden is an open-source, self-hosted implementation of the
Bitwarden password manager. It provides full Bitwarden client
compatibility — browser extensions, mobile apps, and desktop clients
all work natively — while keeping all password data on local hardware
rather than a third-party server.

### Why Self-Hosted
Commercial password managers have demonstrated they are not immune
to breaches. Several major providers have suffered significant
security incidents resulting in encrypted vault data being exposed.
Self-hosting eliminates that attack surface entirely — vault data
never leaves local hardware and is not accessible from the public
internet under any circumstances.

This is a zero-trust approach to credential management: no third
party ever holds the data, no cloud service is in the chain, and
no corporate breach can expose it.

### Architecture
- Runs in Docker Compose on the Security Server
- Accessible via local domain through Caddy reverse proxy
- Internal access only — not exposed to the public internet
- WireGuard tunnel available for remote access if needed

### Backup
Vault data is included in scheduled Kopia backups, ensuring
credential data is protected against hardware failure. Backups
stored locally on dedicated DAS storage.

### Access
- Local: https://vaultwarden.home.arpa (Caddy reverse proxy)
- Remote: Available via WireGuard tunnel if needed

### Design Principles
- Zero dependency on third-party credential storage
- Not exposed to the public internet under any circumstances
- Backed up on schedule alongside all other critical services
- Full Bitwarden client compatibility maintained

### Update Method
Standard Docker Compose pull and recreate:
docker compose pull
docker compose up -d
