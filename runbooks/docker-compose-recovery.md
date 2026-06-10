# Docker Compose Recovery — Deleted Project Directory

**Host:** Primary AI Server
**Service affected:** Open WebUI
**Severity:** Medium — service remained running, management files lost
**Outcome:** Full recovery, no data loss, service reattached to
new Compose project

---

## Incident Summary

While migrating Ollama from Docker to bare metal, the Docker Compose
project directory for Open WebUI was accidentally deleted because both
services shared the same Compose project folder. Open WebUI continued
running because Docker containers and named volumes exist independently
of their Compose project directory. The deleted folder only contained
management files — the actual application data lived in a Docker named
volume and was never at risk.

Recovery involved inspecting the running container to reconstruct the
original Compose configuration, creating a new project directory, and
reattaching Compose management to the existing running container and
volume.

Key lesson: slow down and be deliberate when making infrastructure
changes. The mistake was caused by moving too quickly without
confirming what else shared the project directory before deleting it.

---

## Why This Happened

Open WebUI and Ollama originally shared a single Compose project
directory. When Ollama was migrated to bare metal the entire project
folder was deleted without checking whether other services depended
on it. Open WebUI's Compose file lived in that same folder.

This is a documentation and process gap — co-located services should
be noted explicitly so that changes to one do not accidentally affect
the other.

---

## Key Concept — What Docker Compose Folders Actually Contain

Deleting a Docker Compose project directory does NOT delete:
- Running containers
- Named volumes (persistent application data)
- Docker networks

It only deletes local management files:
- docker-compose.yml
- .env files
- helper scripts
- any notes stored in that folder

A service can keep running indefinitely after its project directory
is deleted, until the container is stopped or the host reboots.
This is both the reason the service survived and the recovery window
that made data-safe recovery possible.

---

## Root Cause

Shared Compose project directory between two services. Deleting the
directory for one service (Ollama) silently removed the management
files for the other (Open WebUI). No data was lost because application
data lived in a named Docker volume, not in the project folder.

---

## Recovery Steps

### Step 1 — Confirm the container is still running

```bash
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}'
```

If the container is still running the application data is safe.
The goal is to reattach Compose management without touching the
existing container or volume until the new Compose file is confirmed
correct.

### Step 2 — Inspect mounts to locate persistent data

```bash
docker inspect CONTAINER_NAME --format '{{json .Mounts}}'
```

Look for named volumes — these contain the real application data.
Note the exact volume name. This is what must be preserved.

### Step 3 — Inspect labels to recover Compose metadata

```bash
docker inspect CONTAINER_NAME --format '{{json .Config.Labels}}'
```

This reveals the original Compose project name, working directory,
and config file path — even after the directory is deleted.

Key labels to look for:
- com.docker.compose.project
- com.docker.compose.project.working_dir
- com.docker.compose.project.config_files
- com.docker.compose.service

### Step 4 — Recover full runtime configuration

```bash
docker inspect CONTAINER_NAME
```

Note the following sections:
- Config.Env — all environment variables
- HostConfig.PortBindings — port mappings
- Mounts — volume mappings
- NetworkSettings.Networks — Docker networks
- HostConfig.RestartPolicy — restart policy

### Step 5 — Rebuild the project directory

Create a new directory reflecting the actual service:

```bash
mkdir -p /home/[user]/docker/SERVICE-NAME
```

### Step 6 — Recreate the Compose file

Use the recovered runtime configuration to write a new
docker-compose.yml. If the service used a named volume from
another Compose project, use the explicit name mapping pattern
to preserve the existing volume:

```yaml
volumes:
  local-alias:
    external: true
    name: real-existing-volume-name
```

This forces Compose to reuse the existing volume rather than
creating a new one under the new project name.

### Step 7 — Validate before starting

```bash
cd /home/[user]/docker/SERVICE-NAME
docker compose config
```

This validates the Compose file syntax without starting anything.
Do not proceed until this passes cleanly.

### Step 8 — Expect a container name conflict

If the original container is still running, docker compose up -d
will fail with a name conflict. This is expected and safe — it
means the old container is still intact.

### Step 9 — Stop and remove the old container

Only do this after the new Compose file is confirmed correct
and the volume mapping is verified. The application data is in
the named volume, not the container.

```bash
docker stop CONTAINER_NAME
docker rm CONTAINER_NAME
```

### Step 10 — Start under new Compose management

```bash
cd /home/[user]/docker/SERVICE-NAME
docker compose up -d
```

### Step 11 — Confirm health and Compose attachment

```bash
docker ps
docker compose ps
docker logs --tail=100 CONTAINER_NAME
```

The difference between these two commands matters:
- docker ps — shows all running containers on the host
- docker compose ps — shows only containers managed by the
  Compose project in the current directory

Both should show the container running. docker compose ps
confirming the container means Compose management is restored.

---

## Important Docker Commands From This Incident

Find running containers:
```bash
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}'
```

Find mounts on a container:
```bash
docker inspect CONTAINER_NAME --format '{{json .Mounts}}'
```

Find Compose labels on a container:
```bash
docker inspect CONTAINER_NAME --format '{{json .Config.Labels}}'
```

List all containers from a specific Compose project:
```bash
docker ps -a --filter label=com.docker.compose.project=PROJECT_NAME \
  --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}'
```

Validate a Compose file without starting anything:
```bash
docker compose config
```

---

## Key Lessons

**Slow down and be deliberate before deleting anything.**
The mistake was caused by moving too quickly. Before deleting
any Docker project directory, confirm what else lives there.
A thirty second check prevents a thirty minute recovery.

**Check for co-located services before deleting a folder.**
If two services share a project directory, deleting it for one
removes management files for both. Document co-location
explicitly so it is never forgotten.

**Named volumes are the real thing worth protecting.**
Application data in named Docker volumes survives container
deletion, Compose directory deletion, and even accidental
docker rm. The folder is just management scaffolding.

**Inspect labels before rebuilding anything.**
Docker stores the original Compose metadata as container labels.
Even after the project directory is gone, docker inspect reveals
exactly where it was and what project it belonged to.

**docker compose config is a safe checkpoint.**
Always validate the Compose file before stopping the old
container. Once the old container is gone the recovery window
closes if the new file is wrong.

**docker ps and docker compose ps are different.**
docker ps shows everything running. docker compose ps shows
only what the current project directory manages. Use both
to confirm full recovery.

---

## Recovery Checklist

1. Confirm container still running with docker ps
2. Inspect mounts — locate named volume containing app data
3. Inspect labels — recover original Compose project metadata
4. Inspect full container config — recover env, ports, networks
5. Create new project directory
6. Write new Compose file using recovered configuration
7. Use external volume name mapping to preserve existing data
8. Validate with docker compose config — do not skip this
9. Expect name conflict on first docker compose up -d
10. Stop and remove old container only after file is validated
11. Run docker compose up -d
12. Confirm with docker ps and docker compose ps
13. Check logs with docker logs --tail=100 CONTAINER_NAME

---

## Final Outcome

Recovered successfully with no data loss. Open WebUI now managed
from new dedicated project directory. Existing named volume
preserved and reattached. Service running healthy and connected
to bare metal Ollama on the same host.
