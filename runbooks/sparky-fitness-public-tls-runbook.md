# SparkyFitness: Public Trusted TLS via OPNsense ACME + DuckDNS

## Status: ✅ Working end-to-end (browser, LAN DNS, and mobile app)

## Why

[SparkyFitness](https://github.com/CodeTheChange/SparkyFitness) runs self-hosted behind a reverse proxy on my local network. Locally-issued/self-signed certs work fine in a browser (you can click through the warning), but the companion **Android app enforces strict TLS validation** and will not connect at all to an untrusted or self-signed certificate — there's no "proceed anyway" option.

That meant getting a **publicly trusted certificate** for a hostname that still resolves to an internal, LAN-only IP — without exposing the service to the internet or paying for a domain. The solution: a free dynamic DNS subdomain, a DNS-01 ACME challenge (no port 80/443 exposure required), and local DNS overrides so the public hostname always resolves internally.

This doc captures the actual working configuration, including where the real implementation diverged from the initial plan.

## Architecture / Roles

| Role | Function |
|---|---|
| Firewall/Router (OPNsense) | Runs ACME client plugin, handles DNS-01 challenge, pushes cert to app host |
| App Host (Docker) | Runs SparkyFitness + Caddy reverse proxy, receives cert via automation |
| DNS Host (Pi-hole) | Local DNS override so the public hostname resolves to the internal app host IP |
| Dynamic DNS Provider | DuckDNS — free subdomain + API token used for the DNS-01 challenge |

## What Was Built

### 1. OPNsense ACME Client
- Plugin: `os-acme-client`
- Challenge type: **DuckDNS (DNS-01)** — no inbound port exposure needed
- Account registered with Let's Encrypt
- Certificate issued for `<your-subdomain>.duckdns.org`, auto-renewal enabled (60-day interval)

### 2. Automations (cert delivery + service reload)
This version of OPNsense manages its own SSH identities per automation (via a "Show Identity" button) rather than accepting a user-supplied private key. Workflow per automation:
1. Create automation → generate identity (OPNsense creates its own ED25519 keypair)
2. Copy the displayed public key
3. Append it to the target host's `~/.ssh/authorized_keys`

Two automations attached to the certificate, run in order:
1. **Push cert** — SFTP upload to the app host, dedicated identity
2. **Reload reverse proxy** — SSH remote command on the app host, separate dedicated identity
   - Command: `docker exec caddy caddy reload --config /etc/caddy/Caddyfile`

### 3. Certificate file layout (confirmed via testing — differs from assumption)
SFTP delivery lands files in a **subfolder named after the domain**, not flat in the target directory:
```
<remote-path>/<your-subdomain>.duckdns.org/
├── ca.pem
├── cert.pem
├── fullchain.pem   ← used
└── key.pem         ← used (not "privkey.pem")
```
Always verify actual delivered paths with `find` before wiring up the reverse proxy — don't assume filenames.

### 4. Reverse Proxy — Docker Compose volume mount
```yaml
volumes:
  - ./Caddyfile:/etc/caddy/Caddyfile:ro
  - ./data:/data
  - ./config:/config
  - ./certs/<your-subdomain>:/etc/caddy/certs/<your-subdomain>:ro
```

### 5. Reverse Proxy — Caddyfile block
```
<your-subdomain>.duckdns.org {
    tls /etc/caddy/certs/<your-subdomain>/<your-subdomain>.duckdns.org/fullchain.pem /etc/caddy/certs/<your-subdomain>/<your-subdomain>.duckdns.org/key.pem
    reverse_proxy <app-host-internal-ip>:<app-port>
}
```

### 6. Local DNS override (Pi-hole)
Local DNS Record: `<your-subdomain>.duckdns.org` → app host's internal IP.
This keeps all traffic on the LAN — the public hostname never actually routes over the internet for local clients.

### 7. App-level trusted origins
SparkyFitness needed an explicit trusted-origins env var updated to include the new hostname, then the container recreated to pick it up:
```
SPARKY_FITNESS_EXTRA_TRUSTED_ORIGINS=https://<your-subdomain>.duckdns.org,https://<internal-hostname>
```

## Issues Hit (useful if repeating this pattern elsewhere)

1. **SSH identity model mismatch** — this OPNsense version generates its own keypair per automation rather than accepting a pasted private key. No blocker once understood — just use "Show Identity" → copy public key → append to target `authorized_keys`.
2. **SFTP delivers to a subfolder, not flat** — cert files land at `<remote-path>/<common-name>/`, named `cert.pem`/`key.pem`/`fullchain.pem`/`ca.pem`, not directly in the target folder as originally assumed.
3. **Reverse proxy crash-loop** — `docker restart` does not pick up newly added volume mounts. Root cause was the cert volume never being added to `docker-compose.yml` in the first place; required `docker compose up -d` to recreate the container, not just restart it. **Compose file changes always need a recreate, not a restart.**
4. **Login worked via raw IP but failed via the new hostname** — classic trusted-origin/CORS rejection, not a networking or cert issue. Resolved by adding the new hostname to the app's trusted-origins env var and recreating the container.
5. **Android app still failed after everything else worked** — turned out to be app-level cached connection state (stale server URL), not DNS, not cert, not Private DNS. Fixed by fully closing the app and re-entering the server URL fresh.

## Verified Working
- ✅ Browser (LAN, local DNS): clean padlock, no warnings
- ✅ SparkyFitness login via public hostname
- ✅ Android app: connects successfully after fresh server URL entry

## Key Takeaway
DNS-01 ACME challenges are a clean way to get a publicly trusted cert for an internally-routed service — no port forwarding, no public exposure, and it plays nicely with strict-TLS mobile clients that self-signed certs can't satisfy. The combination of dynamic DNS + local DNS override + DNS-01 challenge keeps the whole thing LAN-only while still presenting a cert any client will trust.
