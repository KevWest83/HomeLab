## Mermaid Live Editor

**Host:** Primary AI Server
**Deployment:** Docker Compose
**Port:** 8200
**Access:** Direct IP and port, local network only
**Role:** Self-hosted diagramming and visualization tool

### What It Is
Mermaid Live Editor is a self-hosted instance of the Mermaid
diagramming tool. Mermaid generates diagrams and visualizations
from plain text using a simple markdown-like syntax. It supports
flowcharts, sequence diagrams, network diagrams, Gantt charts,
and more — all defined as code rather than drawn manually.

### Why Self-Hosted
Keeps diagramming work local. No diagram content or infrastructure
details are sent to external services. Consistent with the
privacy-first approach applied across the stack — infrastructure
diagrams contain sensitive topology information that should not
leave the local network.

### Architecture
Mermaid Live Editor runs as a single container in Docker Compose
on the Primary AI Server. No external dependencies, no database,
no persistent storage required. Accessed directly via IP and port
on the local network.

### Usage
Mermaid is integrated into the AI workflow via Open WebUI. AI
models generate Mermaid diagram syntax based on prompts, which
is then rendered in the Mermaid Live Editor. This enables rapid
diagram generation for network topology, service architecture,
and infrastructure documentation without manual drawing tools.

### Workflow Example
The network diagram currently in this GitHub repository originated
as a Mermaid diagram generated through this workflow. The Mermaid
output was then processed through an image generation pipeline
to produce the final polished visual. This demonstrates a
practical AI-assisted documentation workflow — AI generates the
structure, rendering tools produce the output.

### Design Principles
- Diagramming infrastructure stays local — no topology data leaves
  the network
- Text-based diagram definition means diagrams are version
  controllable and reproducible
- Integrated into Open WebUI AI workflow for rapid diagram generation
- No persistent storage required — stateless container

### Update Method
Standard Docker Compose pull and recreate:
docker compose pull
docker compose up -d
