# Caddy Internal CA — 1-Year Certificate Lifetime & Workstation Trust Resolution

**Host:** Security Server (Caddy)
**Client:** Workstation (Kubuntu)
**Severity:** Low — services remained functional, browser trust warnings only
**Outcome:** Full resolution, 1-year internal certificates issued and trusted

---

## Incident Summary

Internal `.home.arpa` services proxied by Caddy were issuing
short-lived TLS certificates (12 hours to 7 days) despite a
1-year lifetime being configured. After correcting the PKI
configuration and regenerating the internal certificate authority,
the workstation began rejecting all internal certificates with
ERR_CERT_AUTHORITY_INVALID — even after re-importing the new root
certificate. Root cause was a combination of certificate authority
configuration limits, a polluted system trust store from previous
CA regenerations, and Flatpak browser sandboxing that prevented
the browser from inheriting host-trusted certificates at all.

Four separate issues were identified and resolved across this
incident, each masking the next until resolved.

---

## Phase 1 — Leaf Certificate Lifetime Clamping

**Symptom:** Configuring a 1-year leaf certificate lifetime
(8760h) had no effect — Caddy continued issuing short-lived
certificates.

**Root Cause:** Caddy's internal intermediate CA issues
certificates with a 7-day lifetime by default. A leaf certificate
cannot outlive the intermediate that signs it, so Caddy
automatically clamped the requested 1-year lifetime down to
match the intermediate's remaining validity. Caddy logged this
as a clamping warning rather than an error, making it easy to miss.

**First attempt failed:** Setting the intermediate CA lifetime
to 10 years (87600h) caused Caddy to fail on startup — the
intermediate lifetime must be strictly less than the root CA's
lifetime, which defaults to approximately 10 years.

**Resolution:** Set the intermediate CA lifetime to 5 years
(43800h) — safely below the root CA lifetime while allowing
1-year leaf certificates.

---

## Phase 2 — PKI Regeneration and Root CA Drift

**Symptom:** After correcting the intermediate lifetime, the
existing PKI directory and certificate cache were wiped to force
regeneration. Caddy began issuing 1-year certificates correctly,
but the workstation immediately rejected them with
ERR_CERT_AUTHORITY_INVALID.

**Root Cause:** Wiping the PKI directory generated a brand new
root CA. The workstation still trusted the old root certificate —
the server was now signing with a new root key the client had
never seen.

**Resolution:**

Export the new root certificate from the Caddy container:

```bash
docker cp caddy:/data/caddy/pki/authorities/local/root.crt ~/root.crt
```

Pull the new root certificate to the workstation:

```bash
scp [user]@[security-server-ip]:~/root.crt ~/root.crt
```

---

## Phase 3 — Workstation Trust Store Pollution

**Symptom:** Running `update-ca-certificates` after importing the
new root certificate reported 0 added, 0 removed. OpenSSL
verification still failed with: unable to get local issuer
certificate.

**Root Cause:** The system trust store contained three different
Caddy root certificates from previous CA regenerations, all
sharing the same Common Name but with different fingerprints.
Stale symlinks and duplicate certificates with identical names
caused OpenSSL to build the trust chain using the wrong root.

**Resolution:**

Remove stale certificate files:

```bash
sudo rm -f /usr/local/share/ca-certificates/[stale-cert-files].crt
```

Copy the current root certificate into place:

```bash
sudo cp ~/root.crt /usr/local/share/ca-certificates/caddy-local-root.crt
```

Rebuild the trust store from scratch to purge dead symlinks:

```bash
sudo update-ca-certificates --fresh
```

Verify the chain validates correctly:

```bash
openssl s_client -connect openwebui.home.arpa:443 \
  -servername openwebui.home.arpa \
  -CAfile /etc/ssl/certs/ca-certificates.crt </dev/null 2>&1 \
  | grep "Verify return code"
# Output: Verify return code: 0 (ok)
```

---

## Phase 4 — Flatpak Sandboxing and Browser Profile Corruption

**Symptom:** OpenSSL verified the certificate chain successfully
at the OS level, but Brave and Firefox continued to display
"Not Secure" with ERR_CERT_AUTHORITY_INVALID.

**Root Cause — two separate issues:**

1. **Flatpak sandboxing.** The browser was running as a Flatpak,
   which sandboxes the application and does not automatically
   inherit host-installed CA certificates from the system trust
   store.

2. **Browser profile corruption.** Even after exposing the
   certificate to the Flatpak sandbox, the existing browser
   profile retained stale certificate validation state cached
   from multiple previous CA regenerations.

Confirmed by launching the browser with a temporary clean profile:

```bash
flatpak run com.brave.Browser --user-data-dir=/tmp/brave-clean
```

The clean profile immediately trusted the certificate and
displayed "Connection is secure" — confirming the issue was
profile-level cache corruption, not the certificate chain itself.

**Resolution:**

1. Created a new clean browser profile
2. Migrated necessary extensions to the new profile
3. Removed the corrupted default profile and made the new
   profile the default

---

## Final Working Caddyfile Configuration

```caddyfile
{
    pki {
        ca local {
            intermediate_lifetime 43800h
        }
    }
}

(one_year_internal) {
    tls {
        issuer internal {
            lifetime 8760h
        }
    }
}

vaultwarden.home.arpa {
    import one_year_internal
    reverse_proxy vaultwarden:80
}

openwebui.home.arpa {
    import one_year_internal
    reverse_proxy [primary-ai-server-ip]:8080
}

karakeep.home.arpa {
    import one_year_internal
    reverse_proxy [primary-ai-server-ip]:3000
}

ntfy.home.arpa {
    import one_year_internal
    reverse_proxy ntfy:80
}

searxng.home.arpa {
    import one_year_internal
    reverse_proxy [security-server-ip]:8082
}

search-ui.home.arpa {
    import one_year_internal

    handle /api/search* {
        uri strip_prefix /api
        reverse_proxy [security-server-ip]:8082
    }

    handle /api/llm* {
        rewrite * /api/generate
        reverse_proxy [primary-ai-server-ip]:11434
    }

    handle {
        reverse_proxy searxng-ui:80
    }
}
```

NOTE: search-ui.home.arpa is a custom search interface that is
live but not yet documented in this repository.

---

## Verification

Confirmed leaf certificate lifetime from the workstation:

```bash
echo | openssl s_client -connect openwebui.home.arpa:443 \
  -servername openwebui.home.arpa 2>/dev/null \
  | openssl x509 -noout -dates -issuer
```

Result confirmed a 1-year validity period issued from the
internal intermediate CA as expected.

---

## Key Lessons

**Intermediate CA lifetime constrains leaf certificate lifetime.**
A leaf certificate cannot outlive the intermediate that signs it.
Caddy will silently clamp leaf lifetimes to fit — check logs for
clamping warnings if a configured lifetime doesn't take effect.

**Intermediate lifetime must be less than root lifetime.**
Caddy enforces this at startup. Setting intermediate lifetime too
high causes a hard failure, not a clamp.

**Regenerating a CA breaks existing client trust.**
Wiping the PKI directory generates a new root certificate. Every
client that trusted the old root must import the new one.

**Trust store pollution from repeated CA regeneration is real.**
Multiple root certificates sharing the same Common Name but
different fingerprints can confuse OpenSSL's chain building.
Use --fresh when rebuilding the trust store after CA changes.

**Flatpak applications do not inherit host CA trust.**
Sandboxed applications need certificates imported separately or
exposed via Flatpak overrides — host-level trust is not enough.

**Browser certificate caches can outlive the problem that caused them.**
When a certificate chain is verified correct at the OS level but
a browser still rejects it, test with a clean profile before
assuming the certificate itself is wrong.

**Only the root certificate is ever distributed to clients.**
The root private key must never leave the server under any
circumstances.

---

## Recovery Checklist for Future Certificate Issues

1. Check Caddy logs for clamping warnings if lifetime config
   doesn't take effect
2. Verify intermediate lifetime is less than root lifetime,
   and greater than desired leaf lifetime
3. If PKI is regenerated — export and redistribute new root
   certificate to all clients
4. Clean stale certificates from client trust store before
   reimporting — use --fresh when rebuilding
5. Verify chain with openssl s_client before troubleshooting
   browser-specific issues
6. If OS-level verification passes but browser fails — check
   for Flatpak/sandboxing first
7. If sandboxing is not the issue — test with a clean browser
   profile to rule out cache corruption
