## Wireless Infrastructure

**Hardware:** TP-Link Omada EAP723 (WiFi 7 Access Point)
**Controller:** Omada Controller (Docker Compose, DNS Server)
**Role:** Managed wireless access across all VLANs

### Hardware
The EAP723 is a WiFi 7 (BE5000) dual-band access point with a 2.5G
uplink port, powered via PoE+ from the managed switch. WiFi 7 was
selected for future-proofing and the 2.5G uplink ensures the wireless
infrastructure is not bottlenecked by the wired connection to the
switch.

Key specs:
- **Standard:** WiFi 7 (802.11be)
- **Max throughput:** BE5000
- **Uplink:** 2.5G Ethernet
- **Power:** PoE+ via managed switch (no power adapter required)
- **Security:** WPA3 support
- **Warranty:** 5 years

### Why Omada
TP-Link Omada was selected for ecosystem consistency — the managed
switch and security cameras are also TP-Link hardware. The Omada
platform provides centralized wireless management via a controller,
mirroring how enterprise wireless infrastructure is managed rather
than relying on a standalone consumer access point with no central
management.

### Controller Architecture
The Omada Controller runs in Docker Compose on the DNS Server,
providing centralized management of the access point including
SSID configuration, VLAN assignment, client monitoring, and
firmware updates. The controller is accessed via direct IP address
on the local network.

### Physical Topology
OPNsense → TL-SG116E (Port 2, trunk) → EAP723

The access point connects to the switch on a trunk port carrying
all VLAN tags. Each SSID is mapped to its corresponding VLAN,
with traffic segregation enforced at the switch and OPNsense level.

### SSID Configuration

| SSID             | Band          | VLAN | Notes                              |
|------------------|---------------|------|------------------------------------|
| Family           | 5GHz + 6GHz   | 40   | Primary family network             |
| Family 2.4ghz    | 2.4GHz        | 40   | Legacy band for older devices      |
| HomeLab          | 2.4GHz        | 50   | Homelab VLAN — laptop WiFi access  |
| IOT              | 2.4GHz        | 30   | IoT devices, isolated              |
| Security         | 2.4GHz        | 20   | Security cameras and devices       |
| Management       | 2.4GHz        | 10   | Infrastructure management          |
| Guest            | 5GHz          | —    | Isolated guest network via Omada   |

### Design Decisions

**Dual SSID for family network**
Two SSIDs serve VLAN 40 — one on 5GHz/6GHz for modern devices
and one on 2.4GHz for older devices that do not support newer
bands. Both provide identical access to the family network with
no visible difference to end users.

**HomeLab SSID on WiFi**
The homelab VLAN (50) has a dedicated WiFi SSID to accommodate
a Linux workstation with a problematic wired ethernet driver.
Rather than troubleshooting a hardware compatibility issue,
WiFi provides reliable VLAN 50 access for that device. All other
homelab hosts remain on wired connections.

**Guest network isolation**
The Guest SSID uses Omada's built-in guest network feature rather
than a dedicated VLAN. This provides internet-only access with
client isolation — guests cannot reach any internal network
resources or other guest devices.

**IoT and Security on 2.4GHz only**
IoT and security devices predominantly use 2.4GHz for its better
range and wall penetration. Limiting these SSIDs to 2.4GHz reduces
unnecessary band congestion on 5GHz and 6GHz bands used by
performance-sensitive devices.

### Management Interface
Accessed via direct IP address on the local network through the
Omada Controller web interface.

### Update Method
Firmware updates pushed to the access point via the Omada Controller
web interface. No direct access to the AP required for updates.
