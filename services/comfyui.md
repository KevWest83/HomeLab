## ComfyUI

**Host:** Primary AI Server
**Deployment:** Docker Compose
**Port:** 8188
**Access:** Direct via port 8188
**Role:** Local AI image generation, integrated with Open WebUI

### What It Is
ComfyUI is a node-based image generation interface running locally
on the Primary AI Server. It runs PonyDiffusion V6 XL, a Stable
Diffusion model selected for its compatibility with AMD ROCm GPU
acceleration. Image generation is GPU-accelerated using the RX 7900
XT (20GB VRAM).

Day to day, ComfyUI is not accessed directly. Prompts are written
in Open WebUI, which translates them into a format ComfyUI and
PonyDiffusion can process, and returns the generated image. ComfyUI
operates as a backend service in normal use.

### Architecture
ComfyUI runs as a single container on the same Docker network as
Ollama and Open WebUI. This allows Open WebUI to reach ComfyUI
directly over the internal Docker network without any additional
routing.

```
Open WebUI → ComfyUI (port 8188) → PonyDiffusion V6 XL → RX 7900 XT (ROCm)
```

Model files and assets are stored on a dedicated AI storage volume
on the Primary AI Server, shared with other AI workloads running
on the same host.

### ROCm GPU Configuration
ComfyUI uses the `yanwk/comfyui-boot:rocm` image, which is built
for AMD ROCm GPU acceleration. Getting ROCm working inside the
container required specific configuration that was resolved through
troubleshooting:

| Setting | Purpose |
|---|---|
| `/dev/kfd` device passthrough | Exposes AMD GPU compute interface to the container |
| `/dev/dri` device passthrough | Exposes GPU render nodes to the container |
| Group IDs 44 and 109 | Grants container permission to access the GPU devices |
| `HSA_OVERRIDE_GFX_VERSION: 11.0.0` | Tells ROCm the GPU architecture version |
| `PYTORCH_HIP_ALLOC_CONF` | Controls GPU memory allocation behaviour |
| `ipc: host` | Required for shared memory access between container and host |

### Design Decisions

**Why ComfyUI**
ComfyUI was selected because it integrates directly with Open WebUI,
allowing image generation to be triggered from the same interface
used for LLM inference. The goal was a single interface for both
text and image generation rather than managing separate tools.

**Why PonyDiffusion V6 XL**
PonyDiffusion V6 XL was selected for its compatibility with AMD
ROCm. Running GPU-accelerated image generation on AMD hardware
narrows the field of well-supported models compared to NVIDIA.
PonyDiffusion V6 XL was a reliable choice for this stack.

**Why the Ollama network**
ComfyUI is placed on the same Docker network as Ollama and Open
WebUI. This was a deliberate decision from the start — the intent
was always to use ComfyUI as a backend for Open WebUI rather than
as a standalone tool. Shared networking makes that integration
straightforward.

**Running two GPU workloads on one host**
The Primary AI Server runs both Ollama (LLM inference, bare metal)
and ComfyUI (image generation, Docker) on the same RX 7900 XT.
These workloads do not run simultaneously in normal use — Ollama
handles text inference and ComfyUI handles image generation as
separate tasks. The 20GB VRAM on the RX 7900 XT is sufficient
for either workload independently.

### Connected Services

| Service | Role |
|---|---|
| Open WebUI | Sends image generation requests to ComfyUI |
| Ollama network | Shared Docker network enabling Open WebUI integration |

### Update Method
```bash
docker compose pull && docker compose up -d
```
Run from the ComfyUI directory on the Primary AI Server.
