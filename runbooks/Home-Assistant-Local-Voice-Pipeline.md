# Service Document: Home Assistant Local Voice Pipeline

**Host:** Secondary AI Server  
**Deployment Method:** Docker Compose & Native Systemd  
**Port:** 8123, 10300, 10200  
**Role:** Voice Processing & Automation Hub

### What it is
A 100% local, privacy-first voice pipeline for Home Assistant. It enables voice commands to be processed entirely on-premises without sending audio data to external cloud providers.

### Architecture
The stack utilizes a hybrid deployment approach:
*   **Containerized Services (Docker):** Home Assistant (Core), Faster-Whisper (STT), and Piper (TTS) run in Docker to ensure dependency isolation and easy portability.
*   **Native Service (Systemd):** Ollama runs as a native systemd service to allow direct, high-performance access to the NVIDIA GPU drivers and simplified management of large model weights.
*   **Communication:** Services communicate via the Wyoming protocol over TCP.

### Why decisions were made
*   **Image Selection:** `linuxserver/faster-whisper` was chosen over `rhasspy/wyoming-whisper` because the latter encountered `libcublas.so.12` errors on newer CUDA runtimes.
*   **Model Selection:** `gemma4:12b` was selected over larger models (like `qwen3:14b`) to ensure sufficient VRAM headroom (~4GB) remains available for the Whisper STT engine during active transcription.
*   **VRAM Management:** The Ollama integration `keep-alive` is set to 60 seconds. This prevents the LLM from permanently occupying VRAM, allowing the STT engine to utilize the GPU during voice triggers.
*   **Privacy:** All components are configured for local-only network access to ensure no data leaves the local network.

### Connected services
*   Home Assistant (Automation Engine)
*   Wyoming Protocol (Communication Layer)

### Update method
1.  Navigate to the stack directory: `cd /opt/homeassistant/`
2.  Pull new images: `docker compose pull`
3.  Recreate containers: `docker compose up -d`
4.  For Ollama: `sudo systemctl restart ollama`

---

## Prerequisites

### 1. Install Docker Engine
```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker
```

### 2. Install NVIDIA Container Toolkit
Required for GPU passthrough to Docker containers.

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/docker-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/docker-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt update && sudo apt install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

Verify:
```bash
docker run --rm --gpus all nvidia/cuda:12.0.0-base-ubuntu22.04 nvidia-smi
```

### 3. Install Ollama
```bash
curl -fsSL https://ollama.com/install.sh | sh
ollama pull gemma4:12b
```

---

## Deployment Steps

### Step 1: Create Directory Structure
```bash
mkdir -p /opt/homeassistant/{config,whisper-data,piper-data}
```

### Step 2: Configure Environment
Create a `.env` file in `/opt/homeassistant/` to manage timezones and paths.

### Step 3: Deploy the Stack
```bash
cd /opt/homeassistant/
docker compose up -d
```

### Step 4: Add Wyoming Integrations in Home Assistant
1.  Open `http://[host-ip]:8123`
2.  Go to **Settings → Devices & Services → Add Integration**
3.  Add **Whisper (STT)**:
    *   Host: `[host-api-server-ip]`
    *   Port: `10300`
4.  Add **Piper (TTS)**:
    *   Host: `[host-api-server-ip]`
    *   Port: `10200`

### Step 5: Configure the Voice Assistant
1.  Go to **Settings → Voice Assistants**
2.  Set **Conversation Agent:** Ollama (`gemma4:12b`)
3.  Set **Speech-to-Text:** `wyoming-whisper`
4.  Set **Text-to-Speech:** `wyoming-piper`
5.  **Crucial:** Set `keep-alive` to 60 seconds in the Ollama integration settings to prevent VRAM starvation.

---

## Docker Compose Configuration

### docker-compose.yml
```yaml
services:
  homeassistant:
    container_name: homeassistant
    image: "ghcr.io/home-assistant/home-assistant:stable"
    volumes:
      - ${STACK_PATH}/config:/config
      - /etc/localtime:/etc/localtime:ro
      - /run/dbus:/run/dbus:ro
    restart: unless-stopped
    privileged: true
    network_mode: host
    environment:
      TZ: ${TZ}

  whisper:
    image: linuxserver/faster-whisper:latest
    container_name: wyoming-whisper
    restart: unless-stopped
    ports:
      - "10300:10300"
    volumes:
      - ${STACK_PATH}/whisper-data:/data
    environment:
      - TZ=${TZ}
      - PUID=1000
      - PGID=1000
      - WHISPER_MODEL=base-int8
      - WHISPER_LANG=en
      - WHISPER_BEAM=5
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]

  piper:
    image: rhasspy/wyoming-piper:latest
    container_name: wyng-piper
    restart: unless-stopped
    ports:
      - "10200:10200"
    volumes:
      - ${STACK_PATH}/piper-data:/data
    environment:
      - TZ=${TZ}
    command: [
      "--voice", "en_US-amy-medium"
    ]
```

### .env.example
```env
# Path to the service stack
STACK_PATH=/opt/homeassistant

# System Timezone
TZ=America/Los_Angeles

# User IDs for LinuxServer images
PUID=1000
PGID=1000
```
