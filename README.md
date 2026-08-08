# AI-Services

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)
[![DCO](https://img.shields.io/badge/DCO-1.1-blue)](DCO1.1.txt)
[![Contributing](https://img.shields.io/badge/contributions-welcome-brightgreen)](CONTRIBUTING.md)
[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/IBM/project-ai-services)

AI Services, part of the [IBM Open-Source AI Foundation for Power](https://www.ibm.com/docs/en/aiservices), deliver pre-built AI capabilities and integration with inferencing solutions like Red Hat AI Inference Server. Optimized for IBM Spyre™ on Power, they enable fast deployment and support models such as LLMs, embeddings, and re-rankers—helping enterprises scale AI efficiently.

## 📺 Demo

<video src="https://github-production-user-asset-6210df.s3.amazonaws.com/20432587/615272192-155afcc0-1baf-412d-8c39-93ef7df6ecf7.mp4?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAVCODYLSA53PQK4ZA%2F20260701%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20260701T041911Z&X-Amz-Expires=300&X-Amz-Signature=113d1a4f6dd186fd3331cc7feac8d70762598d0f447a6f6354e163cb8ea8ca3f&X-Amz-SignedHeaders=host&response-content-type=video%2Fmp4" controls="controls" style="max-width: 100%;">
  Your browser does not support the video tag.
</video>

---

## Quick Start

### Installation

For detailed platform-specific installation instructions, see [Installation Guide](docs/INSTALLATION.md).

### Run the binary to get started

```bash
% ai-services --help
A CLI tool for managing AI Services infrastructure.

Usage:
  ai-services [command]

Available Commands:
  application Deploy and monitor the applications
  bootstrap   Initializes AI Services infrastructure
  catalog     Manage the AI Services catalog
  completion  Generate the autocompletion script for the specified shell
  help        Help about any command
  version     Prints CLI version with more info

Flags:
  -h, --help      help for ai-services
  -v, --version   version for ai-services

Use "ai-services [command] --help" for more information about a command.
```

---

## Repository Structure

```bash
project-ai-services/
├── services/          # Backend microservices
│   ├── common/        # Shared library
│   ├── chatbot/       # RAG chatbot service
│   ├── digitize/      # Document ingestion
│   ├── summarize/     # Summarization service
│   └── similarity/    # Similarity search
├── ui/                # Frontend applications
│   ├── chatbot/       # Chatbot UI
│   ├── digitize/      # Digitize UI
│   └── catalog/       # Catalog UI
├── images/            # Container base images
│   ├── service-base/  # Base image for AI services
│   ├── postgres/      # PostgreSQL image
│   ├── litellm/       # LiteLLM proxy
│   ├── caddy/         # Caddy proxy
│   └── tools/         # Utility tools
└── ai-services/       # CLI tool and embedded service assets
    └── assets/
        ├── services/  # Service templates (rag, clip-embedding, …)
        └── components/# Component templates (embedding, reranker, llm, …)
```

### Service Architecture

The repository follows a microservices architecture with:

- **Layered Container Images**: `service-base` → `services-common` → individual services
- **Independent Services**: Each service has its own Containerfile, Makefile, and versioning
- **Shared Common Layer**: Common utilities and dependencies in `services/common/`
- **Clean Boundaries**: No cross-service dependencies, all shared code in common layer

**Service Images:**
- `chatbot-service` - RAG chatbot backend
- `digitize-service` - Document ingestion and processing
- `summarize-service` - Text summarization
- `similarity-service` - Semantic similarity search

**UI Images:**
- `chatbot-ui` - Chatbot web interface
- `digitize-ui` - Document upload interface
- `catalog-ui` - Service catalog interface

---

## Deploying on IBM Power LPAR (ppc64le, CPU-only, No Spyre)

This section covers deploying AI Services on a Linux ppc64le LPAR using Podman — no IBM Spyre accelerator required. Tested on IBM Power11 running RHEL/Fedora.

### Prerequisites

| Requirement | Minimum | Notes |
|---|---|---|
| Architecture | `ppc64le` | Verify with `uname -m` |
| RAM | 32 GB | 64 GB recommended for multiple services |
| Disk | 100 GB free | For models + container images |
| Podman | 4.x+ | `podman --version` |
| Internet | Required | For pulling container images and models |

Confirm architecture and disk space:

```bash
uname -m          # must output: ppc64le
df -h /           # must have 100 GB+ free
podman --version  # must be 4.x+
```

---

### Step 1 — Install the CLI binary

Download the pre-built ppc64le binary from GitHub Releases:

```bash
VERSION="v0.3.0"
curl -LO "https://github.com/IBM/project-ai-services/releases/download/${VERSION}/ai-services-linux-ppc64le"
chmod +x ai-services-linux-ppc64le
sudo mv ai-services-linux-ppc64le /usr/local/bin/ai-services
ai-services version
```

> **Important:** Always use `/usr/local/bin/ai-services`. Do not use `/usr/bin/ai-services` if it exists — it may be a stale older build.

---

### Step 2 — Registry login

Log in to IBM Container Registry to pull service and model images:

```bash
podman login icr.io
```

---

### Step 3 — Bootstrap

Bootstrap the Podman environment (run once per server):

```bash
ai-services bootstrap --runtime podman
```

---

### Step 4 — Start the Catalog

The catalog is the management plane — it provides the UI and backend API for deploying services.

```bash
ai-services catalog configure --runtime podman
```

This prints two URLs. Save them:
- **Catalog UI** — open in browser to deploy services
- **Catalog Backend** — used for CLI login

Log in to the catalog backend:

```bash
ai-services catalog login \
  --server <catalog_backend_URL> \
  --username admin \
  --runtime podman \
  --insecure
```

---

### Step 5 — Download models to the LPAR

> **Critical:** The vLLM container mounts a local `/models` directory. Models must be present on the host **before** starting any deployment. The container does not download models at runtime.

The default model directory is `/var/lib/ai-services/models/`. Download models into subdirectories matching their HuggingFace `org/model-name` path:

```bash
# Create model directory
mkdir -p /var/lib/ai-services/models

# Install huggingface-cli if not present
pip install huggingface-hub

# Download CLIP (multimodal, 512-d, image + text)
huggingface-cli download openai/clip-vit-base-patch32 \
  --local-dir /var/lib/ai-services/models/openai/clip-vit-base-patch32

# Download Granite Embedding (multilingual text, 768-d)
huggingface-cli download ibm-granite/granite-embedding-278m-multilingual \
  --local-dir /var/lib/ai-services/models/ibm-granite/granite-embedding-278m-multilingual
```

Set SELinux labels so Podman containers can read the model files:

```bash
chcon -R -t container_file_t /var/lib/ai-services/models/
```

Verify the label:

```bash
ls -lZ /var/lib/ai-services/models/openai/
# Expected: system_u:object_r:container_file_t:s0
```

---

### Step 6 — Deploy the Embedding Service

#### Using the Catalog UI (recommended)

1. Open the **Catalog UI URL** in a browser
2. Go to **Services** → **Embedding service**
3. Click **Deploy**
4. Choose your model from the dropdown:
   - `clip-vit-base-patch32` — multimodal image + text (512-d)
   - `granite-embedding-278m-multilingual` — multilingual text (768-d)
5. Leave **port** blank to auto-assign, or enter a fixed port
6. Click **Deploy** and wait for status to show **Running** (~2–5 minutes)

> **Note:** After deployment, clear browser localStorage if the form shows stale options from a previous session:
> ```js
> localStorage.removeItem('service-deploy-storage')
> ```

#### Using the CLI

```bash
# List available services
ai-services application templates --runtime podman

# Deploy embedding service with CLIP model
ai-services application create my-embeddings \
  --template clip-embedding \
  --runtime podman

# Deploy with Granite multilingual model
ai-services application create my-embeddings \
  --template clip-embedding \
  --runtime podman \
  --params model=ibm-granite/granite-embedding-278m-multilingual
```

---

### Step 7 — Verify the deployment

Check pod status:

```bash
podman pod ls
```

Check container logs:

```bash
podman logs --tail 30 <embedding-container-name>
```

The service is ready when you see:
```
Application startup complete.
GET /health HTTP/1.1" 200 OK
```

Test the `/v1/embeddings` endpoint directly (replace `<port>` with the assigned host port):

```bash
curl -s http://localhost:<port>/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{"model":"openai/clip-vit-base-patch32","input":["hello world"]}' \
  | python3 -m json.tool | head -20
```

Expected: a JSON response with an `embedding` array of 512 floats.

---

### Deploying a RAG Application (full stack)

To deploy the full RAG stack (OpenSearch + embedding + reranker + LLM + chatbot + digitize):

```bash
# CPU-only RAG (no Spyre)
ai-services application create my-rag \
  --template rag-cpu \
  --runtime podman
```

Monitor status:

```bash
ai-services application ps my-rag --runtime podman
```

Get service endpoint URLs (ports for similarity API, chatbot API, etc.):

```bash
ai-services application info my-rag --runtime podman
```

---

### Managing deployments

```bash
# List all running applications
ai-services application ps --runtime podman

# Stop an application
ai-services application stop <app-name> --runtime podman

# Start a stopped application
ai-services application start <app-name> --runtime podman

# Delete an application (removes pods + DB record)
ai-services application delete <app-name> --runtime podman
```

---

### Troubleshooting

#### Container killed immediately (OOM)
vLLM defaults to large KV-cache allocation. If you see exit code `-9` (OOM killed):
- The template already sets `VLLM_CPU_KVCACHE_SPACE=1` and `--dtype float32` to prevent this.
- If you built a custom image, confirm these env vars are present: `podman inspect <container> --format '{{json .Config.Env}}'`

#### Model not found
The container mounts the host `/var/lib/ai-services/models` directory. If vLLM logs show `model not found`:
```bash
# Confirm model files exist
ls /var/lib/ai-services/models/openai/clip-vit-base-patch32/

# Confirm SELinux label
ls -lZ /var/lib/ai-services/models/ | grep container_file_t
```

#### SELinux permission denied
```bash
chcon -R -t container_file_t /var/lib/ai-services/models/
```

#### Stale ghost deployment in DB
If the UI shows a deployment that no longer has running pods:
```bash
podman exec -it ai-services--db-postgresql \
  psql -U postgres -d ai_services \
  -c "DELETE FROM applications WHERE name='<app-name>';"
```

#### Catalog not reflecting new service templates
Assets are embedded into the binary at build time. If you modified templates locally:
1. SCP the updated files to the LPAR
2. Rebuild the binary and image on the LPAR:
   ```bash
   cd /root/project-ai-services/ai-services
   make build REGISTRY=localhost IMAGE=ai-services TAG=local
   ```
3. Redeploy the catalog:
   ```bash
   podman pod stop ai-services--catalog && \
   podman pod rm ai-services--catalog && \
   /usr/local/bin/ai-services catalog configure --runtime=podman
   ```
4. Clear browser localStorage: `localStorage.removeItem('service-deploy-storage')`

#### Two binaries on PATH
If you have both `/usr/local/bin/ai-services` and `/usr/bin/ai-services`, always use the one in `/usr/local/bin`:
```bash
which ai-services          # confirm it resolves to /usr/local/bin/ai-services
/usr/local/bin/ai-services version
```

---

## Catalog API

For programmatic access to the catalog, see the [Catalog API Guide](docs/Catalog-API-Guide.md).

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## License

Apache 2.0 — see [LICENSE](LICENSE).
