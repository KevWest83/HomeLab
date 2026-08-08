## Purpose

Configure the current Linux AMD64 release of Ollama on the **Productivity Server** to use the Intel Arc B570 through Ollama's Vulkan backend. 

This setup makes **Ollama the primary inference engine** on this node, keeping it consistent with the rest of the environment. `llama.cpp` via Vulkan is retained purely as a fallback for workloads Ollama cannot handle. This allows the Productivity Server to handle background AI tasks (like embeddings and text extraction) without consuming resources on the **Primary AI Server** or the **AI Automation Server**.

### Hardware / Software

* **Host:** Productivity Server
* **GPU:** Intel Arc B570, 10 GB VRAM
* **OS:** Debian 13
* **Ollama:** Current official Linux AMD64 release
* **GPU Backend:** Vulkan
* **Fallback Engine:** llama.cpp + Vulkan
* **Ollama API:** `127.0.0.1:11434` (Localhost)

---

# 1. Install OS Prerequisites

Before installing Ollama, ensure the host OS has the necessary Vulkan drivers and Intel monitoring tools installed.

```bash
sudo apt update
sudo apt install mesa-vulkan-drivers vulkan-tools intel-gpu-tools
```
*Note: `vulkan-tools` provides the `vulkaninfo` command to verify the GPU is visible to the Vulkan API, and `intel-gpu-tools` provides `intel_gpu_top` for monitoring VRAM and compute usage.*

---

# 2. Install Ollama

Use the official installer:

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

The installer will:
* Install Ollama under `/usr/local`
* Create the `ollama` system user
* Add `ollama` to the `render` and `video` groups (granting access to `/dev/dri`)
* Create, enable, and start `ollama.service`

**Expected Warning:**
```text
WARNING: No NVIDIA/AMD GPU detected. Ollama will run in CPU-only mode.
```
*Ignore this warning.* The installer only scans for CUDA (NVIDIA) and ROCm (AMD). It does not mean the Intel Arc GPU cannot be used; it just means Ollama's Vulkan backend must be explicitly enabled.

---

# 3. Verify Base Installation

Check the version and service status:

```bash
ollama --version
systemctl status ollama
```

Verify the API is listening:
```text
127.0.0.1:11434
```

---

# 4. Test Vulkan Manually

Before making the configuration permanent, verify the B570 can successfully process a model via Vulkan.

Stop the background service:
```bash
sudo systemctl stop ollama
```

Start Ollama manually in the foreground with Vulkan enabled:
```bash
OLLAMA_VULKAN=1 OLLAMA_DEBUG=1 ollama serve
```

Leave that terminal running. Open a **second terminal** and run a small test model:
```bash
ollama run llama3.2:3b "Hello"
```

While it is generating, check the processor status:
```bash
ollama ps
```

**Success Criteria:** The output should show `100% GPU` under the PROCESSOR column. This confirms the pipeline: `Ollama → Vulkan → Intel Arc B570` is working.

Once confirmed, return to the first terminal and stop the manual process with `Ctrl+C`.

---

# 5. Configure Vulkan Permanently

To ensure Ollama always uses Vulkan (even after reboots or service restarts), the environment variable must be injected into systemd. 

*Note: Do not rely on `systemctl edit ollama` if it returns an empty file error. Creating the drop-in directory manually is the safest method.*

### 5.1 Create the systemd drop-in directory
```bash
sudo mkdir -p /etc/systemd/system/ollama.service.d
```

### 5.2 Create the override file
```bash
sudo nano /etc/systemd/system/ollama.service.d/override.conf
```

### 5.3 Add the environment variable
Paste the following exactly:
```ini
[Service]
Environment="OLLAMA_VULKAN=1"
```
Save and exit (`Ctrl+O`, `Enter`, `Ctrl+X`).

### 5.4 Apply and Restart
```bash
sudo systemctl daemon-reload
sudo systemctl start ollama
```

---

# 6. Verify Persistent Configuration

Confirm systemd is permanently passing the variable:
```bash
systemctl show ollama --property=Environment
```
*Expected output should include:* `OLLAMA_VULKAN=1`

Run a final test to ensure the background service is using the GPU:
```bash
ollama run llama3.2:3b "Test"
```
```bash
ollama ps
```
*Expected output:* `100% GPU`

---

# 7. Model Considerations on the B570

The B570 has **10 GB of VRAM**. 

Model file size alone does not determine if it will fit entirely in VRAM. Memory is also required for:
* Context Window (KV cache)
* Compute buffers
* Vulkan runtime and driver allocations

**Testing Capacity:**
When pulling new models for background tasks, test them progressively and monitor with `ollama ps` to ensure they aren't offloading to the CPU. 

For real-time hardware monitoring during inference, use:
```bash
sudo intel_gpu_top
```

---

# 8. Node Architecture

```text
                     PRODUCTIVITY SERVER
                             │
                       Intel Arc B570
                             │
                          Vulkan
                             │
                ┌────────────┴────────────┐
                │                         │
             Ollama                   llama.cpp
            PRIMARY                   FALLBACK
                │                         │
         Automated workloads       Specialized workloads
                │                         │
         ┌──────┼──────────┐
         │      │          │
    Embeddings  Text       Data
    Generation  Extraction Structuring
```

**Why this structure?**
* **Ollama** is the preferred interface because it matches the standard inference engine used on the Primary AI Server and AI Automation Server, allowing for seamless API integration and automatic model switching.
* **llama.cpp** remains available because its Vulkan implementation provides a reliable alternative if a specific model performs poorly under Ollama, or if an Ollama update temporarily breaks Vulkan compatibility.

---

# 9. Troubleshooting

### Check Ollama logs
```bash
journalctl -u ollama --no-pager -n 100
```

### Verify Vulkan sees the GPU
```bash
vulkaninfo --summary
```
*The Intel Arc B570 should be listed as a valid Vulkan device.*

### Verify OS sees the GPU
```bash
lspci | grep -i -E 'vga|display|3d'
```

### If Vulkan stops working after an update
1. Check if the environment variable dropped: `systemctl show ollama --property=Environment`
2. If missing, verify `/etc/systemd/system/ollama.service.d/override.conf` still exists.
3. Reload the daemon: `sudo systemctl daemon-reload && sudo systemctl restart ollama`

### Important Distinction: Vulkan vs. IPEX-LLM
This configuration uses the standard Linux AMD64 build of Ollama routed through Vulkan. It is **not** the Intel-specific IPEX-LLM fork. Do not attempt to install older Intel-specific Ollama builds just because the standard installer says "No NVIDIA/AMD GPU detected."
