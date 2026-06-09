# Home Lab Infrastructure

**Segmented | Self-Hosted | Privacy-First**

This repository documents my personal home lab — a fully self-hosted
private infrastructure built and operated independently as a hands-on
learning environment.This is an active personal project. Documentation 
is continuously updated as the infrastructure evolves.

## What This Is

A production-style network and server environment designed around
privacy-first principles. Full control over every layer of the stack
with cloud services added only when and where practical.

## Network Architecture

See HomeLab_Network_Diagram.png for a full visual overview.

Key design decisions:

- OPNsense firewall and router as the network core
- 5 VLANs with security zone isolation
- OPNsense managing DHCP across all network segments
- Pi-hole + Unbound for local DNS filtering and resolution
- Managed switching with VLAN tagging throughout
- Omada Controller for managed wireless infrastructure
- Chrony for NTP time synchronization across all hosts

## VLAN Layout

| VLAN | Name           | Purpose                                       |
|------|----------------|-----------------------------------------------|
| 10   | Infrastructure | DNS, monitoring, network management           |
| 20   | Security       | Cameras, security devices                     |
| 30   | IoT            | Smart home devices, isolated from main network|
| 40   | Clients        | Family devices, laptops, consoles             |
| 50   | Homelab        | Servers, self-hosted services                 |

## Servers

| Host         | Role                    | Key Services                               |
|--------------|-------------------------|--------------------------------------------|
| AI Server    | Local AI inference      | Ollama, Open WebUI, ComfyUI, Home Assistant|
| Media Center | Media streaming         | Jellyfin, Navidrome                        |
| Productivity | Personal cloud services | Immich, Nextcloud, Collabora, Kopia        |
| DNS Server   | Network services        | Pi-hole, Unbound, Omada Controller, Chrony |
| Security     | Monitoring and proxy    | Caddy, Vaultwarden, Uptime Kuma, ntfy, SearXNG |

For full documentation on each service see the [services](services/) directory.

## Self-Hosted Services

Replacing cloud dependencies with locally controlled alternatives:

- Photos: Immich (replaces Google Photos)
- Media: Jellyfin + Navidrome (replaces streaming services)
- Documents: Nextcloud + Collabora (replaces Microsoft Office and OneDrive)
- AI: Ollama + Open WebUI (local LLM inference with GPU acceleration)
  — hybrid model with selective cloud API use for specialized tasks
- DNS: Pi-hole + Unbound (network-wide filtering and local DNS resolution)
- DHCP: OPNsense (centralized network address management)
- Monitoring: Uptime Kuma + Beszel (uptime and system monitoring)
- Automation: n8n (workflow automation)
- Notifications: ntfy (self-hosted push notifications)
- Proxy: Caddy (reverse proxy with SSL)
- Passwords: Vaultwarden (self-hosted password manager)
- Bookmarks: Karakeep (self-hosted bookmark manager)
- Home Automation: Home Assistant
- Wireless: Omada Controller (managed access point administration)
- Time Sync: Chrony (NTP time synchronization across all hosts)

## Backup Architecture

All hosts are backed up on a scheduled basis using Kopia, managed from
the Productivity server. Backup storage is a locally attached DAS,
keeping all backup data on-premises with no cloud dependency.

## Philosophy

Built for reliability. Designed for privacy first. Cloud services added
when and where it is practical. Made to learn.
