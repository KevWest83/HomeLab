# Open WebUI RAG / Embedding Connection Fix After Bare Metal Migration

**Host:** Primary AI Server
**Severity:** Medium — chat functioned normally, document upload and
Knowledge Base queries failed
**Outcome:** Full resolution, RAG and embedding functions restored

---

## Incident Summary

After migrating Ollama from Docker to bare metal on the same host,
document uploads and Knowledge Base queries in Open WebUI began
failing with a DNS resolution error, while standard chat functions
continued working normally.

aiohttp.client_exceptions.ClientConnectorDNSError: Cannot connect
to host ollama:11434 ssl:default [Name or service not known]

The chat interface and the RAG/embedding engine use separate
configuration sources, and only one of them was updated during
the migration.

---

## Root Cause

Open WebUI configures its embedding engine independently of the
main OLLAMA_BASE_URL environment variable. The main chat interface
reads from the environment variable, which was correctly updated
to point to the host IP during the migration. The embedding engine,
however, reads its connection settings from Open WebUI's internal
database — a setting configured separately through the admin panel
and persisted independently of environment variables.

When Ollama moved out of the Docker network, the hostname "ollama"
no longer resolved — that hostname only existed within the old
Docker network's internal DNS. The environment variable was updated,
but the database-stored embedding engine URL still pointed to the
old Docker hostname, causing DNS resolution to fail specifically
for embedding operations.

---

## Why This Was Easy to Miss

Standard chat functionality worked correctly immediately after the
migration, which made the environment variable update appear
complete and successful. The failure only appeared during document
upload and Knowledge Base queries — a separate code path that most
basic testing would not exercise. This is a common pattern after
service migrations: the most visible functionality works while a
secondary, database-persisted configuration silently continues
pointing at the old address.

---

## Resolution

### Step 1 — Verify the Docker environment variable

Confirm the Compose file points to the host IP rather than the old
Docker network hostname:

cd ~/docker/open-webui

Verify docker-compose.yml contains:

environment:
  - OLLAMA_BASE_URL=http://[primary-ai-server-ip]:11434

If changes are made, recreate the container:

docker compose down && docker compose up -d

### Step 2 — Update the embedding engine URL in the admin panel

The embedding engine URL is stored in Open WebUI's internal database
and must be updated separately through the web interface.

1. Log into Open WebUI
2. Navigate to Admin Panel then Settings then Documents
3. Locate Embedding Engine Settings
4. Change the Embedding Engine URL from "http://ollama:11434" to
   the host IP: "http://[primary-ai-server-ip]:11434"
5. Save

---

## Verification

1. Navigate to Workspace then Documents
2. Upload a test document
3. Monitor logs in real time while the document processes:

docker compose logs -f open-webui

4. Confirm the document parses without DNS errors and can be
   queried successfully in a chat session

---

## Key Lessons

**Environment variables and database settings are not the same
configuration source.**
Some applications store certain settings in environment variables
and others in a persisted database, even when both configure
related functionality. Updating one does not update the other.
After any migration affecting connection addresses, check both
sources for every dependent feature, not just the primary
interface.

**Working chat does not mean working RAG.**
After a host or network migration, test every major feature path
independently — not just the most visible one. A migration can
appear fully successful while secondary features silently fail
due to stale configuration in a different location.

**Docker network hostnames stop resolving once a service leaves
that network.**
A hostname like "ollama" only resolves within the Docker network
that provides its internal DNS. Once a service is moved to bare
metal or a different network, any configuration still referencing
the old hostname will fail to resolve — even if the service itself
is reachable by IP.

---

## Related Runbooks

This issue is a direct downstream effect of the Ollama bare metal
migration. See also: docker-compose-recovery.md, which documents
a separate issue from the same migration involving an accidentally
deleted Compose project directory.
