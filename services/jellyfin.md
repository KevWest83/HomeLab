## Jellyfin

**Host:** Media Center
**Deployment:** Docker (Docker Compose)
**Port:** 8096

### What It Is
Jellyfin is a self-hosted media server for streaming movies and TV shows
to devices across the network. It replaces commercial streaming services
with full local control over media libraries.

### Architecture
The Media Center host runs Jellyfin in Docker but holds no data itself.
All media storage lives on the Primary AI Server which doubles as a NAS.
Storage is mounted remotely via NFS4 at boot through fstab, giving
Jellyfin access to the full media library while keeping storage and
compute on separate dedicated hosts. This separation of concerns mirrors
how production media and storage infrastructure is designed.

### Storage
- Media library hosted on Primary AI Server (NAS role)
- Two 10TB drives pooled via MergeFS for unified storage
- Mounted on Media Center via NFS4
- Dedicated Jellyfin directory within the entertainment pool

### Hardware Transcoding
Media Center houses an Intel Arc A310 GPU dedicated to Jellyfin
transcoding. This offloads video processing from the CPU, enabling
smooth streaming of high resolution content to multiple clients
simultaneously without impacting other services.

### Local Access
Accessed via IP address and port on the local network.

### Update Method
Standard Docker Compose pull and recreate:
docker compose pull
docker compose up -d
