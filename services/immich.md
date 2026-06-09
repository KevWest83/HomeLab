## Immich

**Host:** Productivity Server
**Deployment:** Docker Compose (multi-container stack)
**Port:** 2283
**Role:** Self-hosted photo management and personal archive

### What It Is
Immich is a self-hosted photo and video management platform with a
modern mobile-friendly interface. It provides the core functionality
of Google Photos — automatic organization, facial recognition, smart
search, and mobile backup — with all data stored locally on personal
hardware rather than a third-party cloud service.

### Why Self-Hosted
A personal photo archive spanning 25 years is irreplaceable. Storing
it on a commercial cloud service means accepting that a company holds
your most personal data, can change their terms, can be breached, or
can shut down. Self-hosting keeps the archive fully under personal
control with no third-party dependency.

### Architecture
Immich runs as a multi-container Docker Compose stack with each
component in a dedicated container:

- **immich-server** — core application, handles API and web interface
- **immich-machine-learning** — on-device AI for facial recognition,
  object detection, and semantic photo search
- **immich-postgres** — dedicated PostgreSQL database with vector
  extensions required for machine learning features
- **immich-redis** — caching layer for application performance
- **immich-db-backup** — dedicated database backup container

All data is stored on the Productivity Server's locally attached
RAID1 storage array, providing hardware-level redundancy.

### Machine Learning Features
The machine learning container runs entirely on local hardware with
no external API calls. This enables:

- Facial recognition and person grouping
- Object and scene detection
- Semantic search — search photos by content rather than filename
  or date (e.g. "beach", "dog", "birthday")

All AI processing happens on-device. No photos or metadata are sent
to any external service.

### Storage
All Immich data lives on the Productivity Server's RAID1 array:

- Photo and video library
- PostgreSQL database
- Machine learning model cache
- Redis cache

RAID1 mirroring provides hardware redundancy — a single drive failure
does not result in data loss.

### Database Backup
A dedicated postgres-backup container runs alongside Immich and
performs automated daily database backups with a 7-day retention
policy. Database backups are stored separately from the photo library,
ensuring the application state and metadata can be restored
independently of the media files.

### Remote Access
Accessible remotely via WireGuard VPN tunnel. No port forwarding
or direct internet exposure — remote access requires an authenticated
VPN connection.

### Local Access
Accessed via IP address and port on the local network.

### Update Method
Standard Docker Compose pull and recreate:
docker compose pull
docker compose up -d
