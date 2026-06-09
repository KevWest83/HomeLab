## Network Switching

**Hardware:** TP-Link TL-SG116E (Easy Smart Managed Switch)
**Ports:** 16x Gigabit Ethernet (10 in active use)
**Role:** Core LAN switching with 802.1Q VLAN segmentation

### What It Is
The TL-SG116E is a 16-port gigabit smart managed switch providing
802.1Q VLAN tagging, QoS, IGMP snooping, and LAG support. It sits
between OPNsense and all wired network devices, enforcing VLAN
segmentation at the physical layer.

### Why This Switch
Selected for the right combination of managed features, port count,
and price point at $60. VLAN support was the primary requirement.
TP-Link ecosystem compatibility was also a factor — the Omada access
point and security cameras are both TP-Link devices, ensuring clean
interoperability across the wired and wireless stack.

### Physical Topology
Internet → Modem → OPNsense (Router/Firewall) → TL-SG116E → All hosts

Port 1 is the sole uplink to OPNsense. All inter-VLAN routing and
firewall policy is enforced by OPNsense — the switch handles Layer 2
switching and VLAN tagging only.

### Port Assignments

| Port | Device                  | VLAN        |
|------|-------------------------|-------------|
| 1    | OPNsense (uplink)       | Trunk       |
| 2    | Access Point            | Trunk       |
| 3    | DNS Server              | VLAN 10     |
| 4    | Primary AI Server       | VLAN 50     |
| 5    | Productivity Server     | VLAN 50     |
| 6    | Workstation             | VLAN 50     |
| 7    | Family HTPC             | VLAN 40     |
| 8    | Media Center            | VLAN 50     |
| 9    | (unused)                | —           |
| 10   | Phillips Hue Bridge     | VLAN 30     |
| 11   | Security Server         | VLAN 10     |
| 12   | (unused)                | —           |
| 13   | (unused)                | —           |
| 14   | (unused)                | —           |
| 15   | (unused)                | —           |
| 16   | Emergency access port   | Default     |

### VLAN Configuration (802.1Q)

| VLAN ID | Name       | Member Ports      | Tagged Ports | Untagged Ports    |
|---------|------------|-------------------|--------------|-------------------|
| 1       | Default    | 1-2,9-10,12-14,16 | —            | 1-2,9-10,12-14,16 |
| 10      | Management | 1-3,11,15         | 1            | 2-3,11,15         |
| 20      | Security   | 1-2               | 1-2          | —                 |
| 30      | IoT        | 1-2,10            | 1-2          | 10                |
| 40      | WestFamily | 1-2,7,9           | 1-2          | 7,9               |
| 50      | FapNet     | 1-2,4-6,8         | 1-2          | 4-6,8             |

### Recovery Port
Port 16 is reserved as an emergency access port on the default VLAN.
In the event of a misconfiguration that locks out management access,
connecting directly to port 16 restores access to the switch
interface without requiring a factory reset. This is a standard
sysadmin practice for managed switch deployments.

### Design Principles
- Layer 2 VLAN enforcement at the switch level
- All inter-VLAN routing handled by OPNsense, not the switch
- Single uplink trunk port carries all VLAN tags to OPNsense
- Access Point on trunk port to support multiple SSIDs across VLANs
- IoT devices physically isolated to VLAN 30 at the port level
- Emergency recovery port maintained on default VLAN at all times
- TP-Link ecosystem consistency across switch, AP, and cameras

### Management Interface
Accessed via web UI at direct IP address on the local network.
No CLI — TL-SG116E uses a browser-based management interface only.

### Update Method
Firmware updates applied via the switch web management interface.
