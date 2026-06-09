## Navidrome

**Host:** Media Center
**Deployment:** Docker Compose
**Port:** 4533
**Role:** Self-hosted music streaming server

### What It Is
Navidrome is a self-hosted music streaming server compatible with
the Subsonic API. It provides personal music library management and
streaming to any Subsonic-compatible client, replacing commercial
streaming services with full local control over a personal music
collection.

### Architecture
Navidrome runs in Docker Compose on the Media Center host. Music
files are stored on the Primary AI Server (NAS role) and mounted
to the Media Center via NFS4 — the same mount used by Jellyfin,
with a dedicated music directory separate from the TV and movie
library. Navidrome reads the music library in read-only mode,
ensuring the container cannot modify the source files.

### Storage
- Music library hosted on Primary AI Server (NAS role)
- Mounted on Media Center via NFS4 (read-only)
- Dedicated music directory within the entertainment pool
- Separate from Jellyfin's TV and movie library

### Clients in Use
Navidrome exposes a Subsonic-compatible API enabling a wide range
of clients:

- **Feishin** — desktop client for local network playback
- **Symfonium** — mobile client for on-the-go streaming

### Remote Access
Accessible remotely via WireGuard VPN tunnel. Primary remote use
case — music is the most common service accessed while away from
home. No port forwarding or direct internet exposure — remote
access requires an authenticated VPN connection.

### Local Access
Accessed via IP address and port on the local network.

### Design Principles
- Music library mounted read-only — container cannot modify source files
- Subsonic API compatibility enables client flexibility
- No dependency on commercial streaming services or subscriptions
- Full library ownership — no licensing restrictions on playback

### Update Method
Standard Docker Compose pull and recreate:
docker compose pull
docker compose up -d
