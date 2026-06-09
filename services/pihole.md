## Pi-hole

**Host:** DNS Server
**Deployment:** Docker (Docker Compose)
**Role:** Network-wide DNS filtering, ad blocking, local DNS records

### What It Is
Pi-hole is a network-wide DNS sinkhole that filters ad and tracker
traffic at the DNS level before it reaches any device on the network.
Rather than installing ad blockers on individual devices, Pi-hole
provides filtering for every device on every VLAN from a single
point of control.

### Architecture
Pi-hole sits at the front of the DNS chain for the entire network:

Client → Pi-hole (filtering) → Unbound in OPNsense (resolution) → Root servers

All DNS queries from all VLANs are forced through Pi-hole via an
OPNsense firewall rule that intercepts port 53 traffic and redirects
it regardless of what DNS server a device is configured to use. This
is particularly important for IoT devices which commonly use hardcoded
DNS servers like 8.8.8.8 in an attempt to bypass local filtering.
No device on any VLAN can circumvent DNS filtering.

### Local DNS Records
Pi-hole manages all local DNS records for the network. This enables
all self-hosted services to be accessible by hostname rather than
IP address, mirroring how production environments handle internal
service routing. Examples:

- vaultwarden.home.arpa
- openwebui.home.arpa
- And all other self-hosted services on the *.home.arpa domain

### Coverage
All VLANs route DNS through Pi-hole:

| VLAN | Name           | DNS Filtering |
|------|----------------|---------------|
| 10   | Infrastructure | Forced via firewall rule |
| 20   | Security       | Forced via firewall rule |
| 30   | IoT            | Forced via firewall rule |
| 40   | Clients        | Forced via firewall rule |
| 50   | Homelab        | Forced via firewall rule |

### Why This Matters
IoT devices are notorious for attempting to bypass local DNS by using
hardcoded upstream resolvers. By enforcing DNS redirection at the
firewall level, all devices are subject to filtering regardless of
their configuration. This closes a common security gap in home and
small business networks.

### Design Principles
- Single point of DNS control for all VLANs
- No device can bypass filtering regardless of configuration
- Local DNS records eliminate IP-based service addressing
- Filtering happens at network level — no per-device configuration needed
- Paired with Unbound for fully private recursive resolution

### Update Method
Standard Docker Compose pull and recreate:
docker compose pull
docker compose up -d
