# Debian Upgrade Recovery — Interrupted apt Upgrade

**Host:** Productivity Server
**OS:** Debian 13 Trixie
**Severity:** High — host unreachable via SSH, required hard power cycle
**Outcome:** Full recovery, no data loss, RAID healthy, all packages upgraded

---

## Incident Summary

A routine apt upgrade stalled mid-run during package configuration.
Interrupting the frozen process with Ctrl+C left dpkg in a broken
state with stale lock files and half-configured packages. Re-running
dpkg --configure -a triggered systemd to reconfigure itself while
still running, which broke PAM authentication system-wide. GRUB
then stalled writing to the EFI partition with systemd in a degraded
state. With PAM non-functional, no new SSH sessions could authenticate
and systemd was too broken to process reboot requests via any path.
A hard power cycle was required.

Host remained pingable throughout. Verified from Workstation.
Recovered while monitoring from a secondary device with no
disruption to other network services.

---

## Root Cause

**Primary:** Interrupting a running apt upgrade with Ctrl+C left
dpkg with stale lock files and half-configured packages. Re-running
dpkg --configure -a caused systemd to reconfigure itself while
still running, breaking PAM system-wide.

**Secondary:** GRUB stalled writing to the EFI partition while
systemd was in a degraded state, preventing normal or forced
reboot via SSH.

**Contributing factor:** NodeSource Node 18 repository signing
key uses SHA1, which Debian 13 Trixie's OpenPGP verifier began
rejecting. This caused non-fatal warnings on every apt update
but did not cause the upgrade failure itself.

---

## Recovery Steps

### Step 1 — Verify host is alive

From Workstation, confirm host is still responding:

```bash
ping [productivity-server-ip]
```

If host responds to ping, it is alive even if SSH is broken.
Do not assume the host is dead based on SSH failure alone.

### Step 2 — Clear stale dpkg locks

```bash
sudo rm /var/lib/dpkg/lock-frontend
sudo rm /var/lib/dpkg/lock
sudo dpkg --configure -a
```

If dpkg --configure -a appears to freeze — particularly at
"Installing for x86_64-efi platform" — this is GRUB writing
to the EFI partition. On older hardware this can take a very
long time. Wait at least 60 minutes before intervening.

### Step 3 — Hard power cycle (last resort)

When systemd, reboot commands, and SysRq via /proc all fail
due to PAM breakage, a physical 5-second power button hold
is required to force shutdown.

NOTE: SysRq via keyboard (Alt+SysRq+R/E/I/S/U/B) remains
viable even when PAM is broken and should be attempted before
physical power cycle if keyboard access is available:
- R — return keyboard to raw mode
- E — SIGTERM all processes
- I — SIGKILL all processes
- S — sync disks
- U — remount filesystems read-only
- B — reboot

SysRq via /proc/sysrq-trigger requires sudo which is blocked
when PAM is broken. Use keyboard SysRq or physical power only.

### Step 4 — Post-reboot dpkg cleanup

After reboot, SSH will be immediately functional. Run:

```bash
sudo dpkg --configure -a
```

GRUB should complete successfully on this run.

### Step 5 — Fix broken dependencies

```bash
sudo apt install --fix-broken
```

Resolves any remaining broken package dependencies left over
from the interrupted upgrade.

### Step 6 — Complete the upgrade

```bash
sudo apt upgrade
sudo apt autoremove
```

Remaining packages will upgrade cleanly. Autoremove cleans up
old kernels and unused dependencies.

### Step 7 — Remove broken repository if present

If NodeSource Node 18 repo is present and causing SHA1 warnings:

```bash
sudo rm /etc/apt/sources.list.d/nodesource.list
```

Remove if Node.js is not required. Migrate to Node 20 or 22
if Node.js is needed — both use compliant signing keys.

### Step 8 — Verify RAID health

```bash
cat /proc/mdstat
```

Confirm output shows [UU] — both drives healthy.
A hard power cycle should not trigger a RAID resync on a
healthy array but always verify after unclean shutdown.

### Step 9 — Final reboot and verification

```bash
sudo reboot
uname -r
```

Confirm system comes up on the expected kernel version.

---

## Final System State After Recovery

| Component         | Status                          |
|-------------------|---------------------------------|
| Kernel            | 6.12.88+deb13-amd64             |
| All packages      | Upgraded and configured cleanly |
| GRUB              | Updated, installed to EFI       |
| RAID1 /dev/md0    | Healthy [UU]                    |
| SSH / PAM         | Fully functional                |
| Old kernels       | Removed                         |
| NodeSource repo   | Removed                         |

---

## Technical Background — PAM

PAM (Pluggable Authentication Modules) is the authentication
framework Linux uses to verify identity for SSH logins, sudo
commands, and local logins. It sits between the user and the
system and handles credential verification regardless of the
authentication method in use.

PAM is tightly coupled to systemd on Debian. When dpkg
reconfigured systemd mid-upgrade, PAM entered a half-updated
state — the old version was gone but the new version was not
fully in place. During that window no authentication requests
could complete, which is why new SSH sessions hung at the
password prompt and sudo commands were unavailable.

This is expected behavior during systemd self-reconfiguration,
not a bug. It resolves cleanly after a full reboot.

---

## Key Lessons

**Never Ctrl+C a running apt upgrade.**
If the upgrade appears frozen, wait at least 60 minutes before
intervening. GRUB and systemd upgrades can take a very long time
on older hardware with slow EFI writes. The cost of interrupting
is almost always worse than the cost of waiting. This single
action caused the entire incident.

**Systemd reconfiguring itself breaks PAM.**
During the window when systemd is mid-configure, PAM stops
functioning. This blocks all new SSH logins and all sudo
commands, making remote recovery extremely difficult. If dpkg
--configure -a is running and a new SSH session fails — wait,
do not intervene.

**SysRq via /proc requires PAM — keyboard SysRq does not.**
Plan for physical access on servers that lack IPMI or iDRAC.
The keyboard SysRq sequence (Alt+SysRq+REISUB) does not depend
on PAM and should be the last resort before physical power.

**RAID1 handles hard power cuts cleanly.**
A hard power cycle did not trigger a resync. Data integrity
was maintained throughout. Always verify with cat /proc/mdstat
after any unclean shutdown.

**Ping is alive, SSH is not always the same thing.**
A host that does not respond to SSH is not necessarily dead.
Always verify with ping from another host before assuming
total failure.

---

## Recovery Checklist

1. Ping host from another machine to confirm it is alive
2. Attempt SSH — if blocked, PAM may be broken
3. Clear stale dpkg locks and run dpkg --configure -a
4. If frozen at EFI install — wait 60 minutes minimum
5. If reboot commands fail — try keyboard SysRq (REISUB)
6. If keyboard unavailable — hard power cycle (5-second hold)
7. Post-reboot: run dpkg --configure -a again
8. Run apt install --fix-broken
9. Run apt upgrade and apt autoremove
10. Remove broken repos if present
11. Verify RAID health with cat /proc/mdstat
12. Final reboot and confirm kernel version with uname -r
