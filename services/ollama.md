## Ollama

**Host:** Primary AI Server
**Deployment:** Bare metal (systemd managed)
**GPU:** AMD Radeon 7900 XT (20GB VRAM, ROCm)

### Why Bare Metal
Migrated from Docker to bare metal to resolve VRAM spillover issues.
Bare metal deployment gives Ollama direct GPU access via ROCm,
significantly improving inference speed. Gemma 4 26B now fits cleanly
in VRAM with 20-30 second response times.

### Key Configuration
- Models stored on root SSD in dedicated models directory
- Systemd override at: /etc/systemd/system/ollama.service.d/override.conf
- Flash attention enabled (OLLAMA_FLASH_ATTENTION=1)
- Bound to all interfaces on port 11434 for Open WebUI network access

### Update Method
curl -fsSL https://ollama.com/install.sh | sh

### Connected Services
Open WebUI (Docker) connects to Ollama via local network address
