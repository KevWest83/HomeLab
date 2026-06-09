## OPNsense

**Host:** Dedicated Firewall Appliance
**Deployment:** Bare metal
**Role:** Router, Firewall, DHCP Server, Gateway, VPN Endpoint

### Hardware
Purpose-built firewall micro appliance selected specifically for this
role rather than repurposing a general PC. Key hardware decisions:

- **CPU:** Intel Celeron N5105 Quad Core @ 2.9GHz
- **RAM:** 8GB
- **Storage:** 100GB NVMe SSD
- **NICs:** 4x Intel i226-V 2.5GbE LAN ports
- **AES-NI:** Hardware-accelerated encryption support

The Intel i226-V NICs were a deliberate choice — they are enterprise-grade
network controllers with strong driver support in FreeBSD (the OS
underlying OPNsense). AES-NI hardware acceleration ensures VPN
operations do not bottleneck on encryption overhead.

### Why OPNsense
OPNsense is an open-source firewall and routing platform built on
FreeBSD. It was chosen over consumer routers and alternatives because:

- Full control over firewall rules, routing, and network policy
- Active development with regular security updates
- Enterprise-grade feature set at no licensing cost
- Strong community and documentation
- Privacy-first — no phone-home telemetry or cloud dependency

### Network Architecture
OPNsense serves as the core of the entire network. All traffic between
VLANs passes through OPNsense where firewall rules enforce strict
inter-VLAN policy. No two VLANs can communicate unless explicitly
permitted. This mirrors how enterprise network segmentation is designed
and enforced.

### VLAN Design
Five VLANs designed around security zones and workload separation:

| VLAN | Name           | Purpose                                        |
|------|----------------|------------------------------------------------|
| 10   | Infrastructure | DNS, monitoring, network management            |
| 20   | Security       | Cameras and security devices, isolated         |
| 30   | IoT            | Smart home devices, isolated from all servers  |
| 40   | Clients        | Family devices, laptops, consoles              |
| 50   | Homelab        | Servers and self-hosted services               |

### VLAN Security Policy

**VLAN 30 — IoT Isolation**
IoT devices require internet access to function but represent an
uncontrolled attack surface. Firewall rules allow IoT devices to
reach the internet but block all traffic leaving the VLAN toward
any other internal network segment. The VLAN functions as a
container — devices can talk to each other and the internet but
cannot reach family devices, servers, or infrastructure under
any circumstance.

**VLAN 40 — Family Network**
Designed to be invisible to non-technical family members. From
their perspective it functions identically to a standard home
WiFi network. Underneath, firewall rules enforce least-privilege
access — TVs can reach Jellyfin and Navidrome for media streaming
but have no access to Nextcloud, the Productivity server, or any
infrastructure services. Family members get a seamless experience
while the network enforces appropriate boundaries automatically.

**VLAN 50 — Homelab**
All self-hosted servers live here. Access from other VLANs is
explicitly controlled — only permitted traffic reaches this segment.

### Services Running on OPNsense
- **Firewall:** Stateful packet inspection with least-privilege rules
- **Router:** Inter-VLAN routing with explicit allow/deny policy
- **DHCP:** Centralized address management across all VLANs
- **Gateway:** Single exit point for all network traffic
- **VPN:** WireGuard endpoint for secure remote access
- **DNS Resolution:** Unbound (recursive DNS resolver, runs natively 
  within OPNsense — handles local DNS resolution without forwarding 
  queries to upstream third-party resolvers)

### WireGuard Remote Access
WireGuard VPN runs directly on OPNsense, leveraging AES-NI hardware
acceleration for encryption. A static IP address ensures the VPN
endpoint is always reachable without dynamic DNS complexity.

Rather than exposing individual services to the internet, WireGuard
tunnels the mobile client into the home network as a full network
participant. This means all self-hosted services are accessible
remotely exactly as they are locally — no port forwarding, no exposed
services, no attack surface on the public internet.

Services accessed remotely via WireGuard:
- Navidrome — music streaming
- Jellyfin — video streaming
- Immich — photo management
- Kavita — book library
- Nextcloud — documents and files
- Open WebUI — local AI interface
- IoT devices — home automation and monitoring

This approach effectively replaces commercial cloud services with
fully self-hosted equivalents accessible anywhere via encrypted tunnel.

### Design Principles
- Least-privilege access — traffic denied by default, allowed explicitly
- IoT devices fully isolated — internet access only, no internal reach
- Family network transparent to end users, controlled at firewall level
- No services exposed directly to the internet — VPN only remote access
- Single firewall appliance as authoritative network policy enforcer
- Static IP eliminates dynamic DNS dependency for VPN reliability

### DNS Architecture
DNS resolution is handled by Unbound running natively inside OPNsense
rather than forwarding queries to an upstream provider like Google or
Cloudflare. This means DNS queries are resolved recursively from root
servers directly, with no third party in the chain. Pi-hole sits in
front of Unbound for network-wide ad and tracker blocking before
queries are resolved.

The full DNS chain is:
Client → Pi-hole (filtering) → Unbound in OPNsense (resolution) → Root servers

### Update Method
Updates applied via OPNsense web UI:
System → Firmware → Updates
