# LAN Outage Triage — Post-Relocation Network Recovery

**Date:** June 2026
**Duration:** Evening troubleshooting session
**Severity:** High — full network outage, no internet, no local services
**Resolution:** Corrected physical interface orientation on router,
power cycled ISP modem, renewed WAN DHCP lease

---

## Incident Summary

After physically relocating networking equipment (modem, router,
managed switch, and access point) to accommodate a new AC unit,
the network failed to come back online. What appeared to be
multiple simultaneous failures across many systems was ultimately
caused by two separate issues:

1. Incorrect WAN/LAN cable orientation on OPNsense during reconnection
2. ISP modem retaining a stale DHCP lease, preventing OPNsense from
   obtaining a WAN address after the physical change

Diagnosed and resolved remotely using only a phone with no local
network connectivity. Primary challenge was methodical layer-by-layer
triage with AI assistance while maintaining focus on the physical
layer as the likely failure point.

---

## Initial Symptoms

- No internet access
- No WiFi connectivity
- No DHCP on workstation
- OPNsense web interface inaccessible
- DNS Server unreachable
- Multiple servers appearing offline
- Switch ports showed link lights throughout

The presence of link lights suggested physical hardware was functional
but masked the actual failure point.

---

## Troubleshooting Process

### Layer 1 — Physical

Confirmed all hardware powered and showing link lights.
Conclusion: physical hardware failure unlikely.

### Layer 2 — Workstation isolation

Workstation reported no IP address and Ethernet disconnected.
Connected workstation directly to modem — after NIC
reinitialization, internet access confirmed.

Conclusion: workstation NIC functional, cable functional,
modem functional. Failure exists upstream of the workstation
but downstream of the modem.

### Layer 3 — Router interface orientation

Discovered WAN and LAN cable orientation on OPNsense was
incorrect after reconnection. ETH0 is WAN, ETH1 is LAN —
this was undocumented and the ports were reconnected in
reverse. Swapping cables to correct orientation restored
internal connectivity. Most servers became reachable.
OPNsense web interface accessible again.

Conclusion: primary failure was incorrect physical interface
assignment during relocation.

### Layer 4 — WAN recovery

Internal connectivity restored but no internet access remained.
OPNsense reported: "No gateways are configured."
WAN interface showed DHCP enabled but no IPv4 lease obtained.

Tested with ping 8.8.8.8 — failed.
Conclusion: not a DNS problem — routing failure.

### Layer 5 — Modem power cycle

ISP modem was retaining a stale DHCP/MAC association from the
previous connection and would not issue a new WAN lease.

Recovery steps:
1. Power off modem
2. Wait approximately one minute
3. Power modem back on
4. Allow modem to fully synchronize
5. Renew WAN connection in OPNsense

Result: WAN obtained IPv4 address, default gateway restored,
internet connectivity returned, all services came back online.

---

## Root Cause

**Primary:** Incorrect WAN/LAN cable orientation on OPNsense
after physical relocation caused complete loss of internal
routing. ETH0 is WAN, ETH1 is LAN — this was undocumented
and reconnected in reverse.

**Secondary:** ISP modem retained stale DHCP lease and would
not issue a new WAN address until power cycled.

---

## Key Lessons

**Document physical infrastructure before touching it.**
The root cause of this incident was undocumented interface
orientation — ETH0 on OPNsense is WAN, ETH1 is LAN. This was
known during initial setup but never written down. When
reconnecting after relocation the ports were assumed incorrectly.
Had this been documented the outage would not have occurred.
Physical port assignments, cable orientations, and interface
roles belong in documentation just as much as software
configuration.

**Everything working 30 minutes prior narrows the search space.**
When a full outage follows an intentional physical change, the
failure is almost certainly in what was just touched. Start
there before considering software, configuration, or
multi-system failure scenarios.

**Troubleshoot by layers.**
Physical → Link → IP → Routing → DNS → Application.
Never skip ahead. DNS cannot be the problem if ping 8.8.8.8 fails.

**Link lights are not proof of connectivity.**
A green Ethernet LED only confirms physical link negotiation.
It does not confirm DHCP, VLAN correctness, routing, or internet
connectivity.

**Separate routing from DNS early.**
If ping 8.8.8.8 fails — routing problem.
If ping 8.8.8.8 works but ping google.com fails — DNS problem.
Test in this order every time.

**Test components in isolation.**
Connecting the workstation directly to the modem rapidly confirmed
which components were functional and eliminated the search space.

---

## Recovery Checklist for Future Outages

Use this checklist in order. Do not skip steps.

1. Verify modem is powered and synchronized
2. Verify OPNsense WAN/LAN cabling orientation (ETH0=WAN, ETH1=LAN)
3. Verify switch uplink to router
4. Test workstation directly to router
5. Confirm WAN interface has IPv4 address in OPNsense
6. Confirm default gateway exists
7. Test in order:
   - ping [gateway IP]
   - ping 8.8.8.8
   - ping google.com
8. Only begin DNS/Pi-hole troubleshooting after routing confirmed
9. If WAN has no lease — power cycle modem, then renew WAN DHCP
   in OPNsense

---

## Final Resolution

Network outage resolved by:
- Correcting physical router interface orientation (ETH0=WAN, ETH1=LAN)
- Power cycling ISP modem
- Renewing WAN DHCP lease in OPNsense

No VLAN changes, firewall modifications, or service configuration
changes were required.
