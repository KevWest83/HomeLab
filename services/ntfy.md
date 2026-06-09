## ntfy

**Host:** Security Server
**Deployment:** Docker Compose
**Port:** 8080 (HTTP — Caddy handles HTTPS externally)
**Access:** https://ntfy.home.arpa (via Caddy reverse proxy)
**Role:** Self-hosted push notification hub

### What It Is
ntfy is a simple self-hosted notification service that receives
messages from other services and makes them available via a
topic-based subscription model. It acts as a central notification
hub for the homelab, aggregating alerts and status updates from
multiple monitoring services into a single place.

### Why Self-Hosted
Keeps all infrastructure notifications local. No third-party
notification service receives information about homelab services,
update status, or system health. Consistent with the privacy-first
approach applied across the rest of the stack.

### Architecture
ntfy runs in Docker Compose on the Security Server and is connected
to a shared Docker network alongside other monitoring services,
allowing direct communication between containers without exposing
ports unnecessarily. Caddy handles HTTPS termination externally,
ntfy itself runs HTTP only internally.

### Notification Sources

| Source       | Purpose                                        |
|--------------|------------------------------------------------|
| DIUN         | Container image update notifications — runs on each host, reports when new image versions are available |
| Beszel       | System resource and health status across all hosts |
| Uptime Kuma  | Service uptime monitoring and downtime alerts  |

### DIUN Integration
DIUN (Docker Image Update Notifier) is deployed on each host and
monitors running container images for new versions. When an update
is available it sends a notification to ntfy. This provides a
centralized view of pending container updates across the entire
six-host infrastructure without manually checking each host.

### Current Workflow
Notifications are currently reviewed manually via the browser
interface. The ntfy mobile app (available for Android and iOS)
can be configured to push notifications directly to a phone via
WireGuard for real-time alerts — a natural next step when needed.

### Previous Integration
ntfy was previously used as the notification endpoint for an n8n
homelab reporter workflow. That integration is currently inactive
pending a rebuild of the n8n/Open WebUI automation pipeline.

### Network Configuration
ntfy is connected to a shared Docker network (monitoring_shared)
alongside Beszel and Uptime Kuma, enabling direct inter-container
communication between monitoring services.

### Design Principles
- Centralized notification hub for all homelab monitoring
- No third-party notification service in the chain
- Shared Docker network keeps monitoring stack communication internal
- HTTPS handled by Caddy — ntfy runs HTTP internally only

### Update Method
Standard Docker Compose pull and recreate:
docker compose pull
docker compose up -d
