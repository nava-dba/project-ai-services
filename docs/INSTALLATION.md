# AI Services Installation Guide

Complete installation instructions for AI Services across all supported platforms.

## Table of Contents

1. [Supported Platforms](#supported-platforms)
2. [Prerequisites](#prerequisites)
3. [Quick Installation](#quick-installation)
4. [Verified Installation with Cosign](#verified-installation-with-cosign)
5. [Container Image Verification](#container-image-verification)
6. [Embedding Service](#embedding-service)
7. [Additional Resources](#additional-resources)

---

## Supported Platforms

AI Services provides pre-built binaries for the following platforms:

| Platform | Architecture | Binary Name |
|----------|-------------|-------------|
| macOS | Intel (x86_64) | `ai-services-darwin-amd64` |
| macOS | Apple Silicon (ARM64) | `ai-services-darwin-arm64` |
| Linux | x86_64/AMD64 | `ai-services-linux-amd64` |
| Linux | ppc64le (Power) | `ai-services-linux-ppc64le` |

### Deployment Modes

- **Client-only mode** (macOS, Linux x86_64/AMD64): The CLI acts as a client that connects to a remote OpenShift cluster for application deployment and management.

- **Local + Remote mode** (Linux ppc64le/Power): Supports both local Podman-based deployments and remote OpenShift cluster connections, optimized for IBM Power Systems and IBM Spyre™.

---

## Prerequisites

### All Platforms

- **Internet connection** for downloading binaries
- **Terminal/Command line access**
- **Sudo/Administrator privileges** for system-wide installation

### Optional (Recommended)

- **Podman** or **Docker** for container-based deployments (Linux ppc64le only)
- **Cosign** for signature verification

---

## Quick Installation

Choose your platform and run the appropriate commands:

### macOS (Intel)

```bash
VERSION="v0.3.0"
curl -LO "https://github.com/IBM/project-ai-services/releases/download/${VERSION}/ai-services-darwin-amd64"
chmod +x ai-services-darwin-amd64
sudo mv ai-services-darwin-amd64 /usr/local/bin/ai-services
ai-services version
```

### macOS (Apple Silicon)

```bash
VERSION="v0.3.0"
curl -LO "https://github.com/IBM/project-ai-services/releases/download/${VERSION}/ai-services-darwin-arm64"
chmod +x ai-services-darwin-arm64
sudo mv ai-services-darwin-arm64 /usr/local/bin/ai-services
ai-services version
```

### Linux (x86_64/AMD64)

```bash
VERSION="v0.3.0"
curl -LO "https://github.com/IBM/project-ai-services/releases/download/${VERSION}/ai-services-linux-amd64"
chmod +x ai-services-linux-amd64
sudo mv ai-services-linux-amd64 /usr/local/bin/ai-services
ai-services version
```

### Linux (ppc64le/Power)

**Optimized for IBM Power Systems and IBM Spyre™**

```bash
VERSION="v0.3.0"
curl -LO "https://github.com/IBM/project-ai-services/releases/download/${VERSION}/ai-services-linux-ppc64le"
chmod +x ai-services-linux-ppc64le
sudo mv ai-services-linux-ppc64le /usr/local/bin/ai-services
ai-services version
```

---

## Verified Installation with Cosign

For enhanced security, verify binary signatures before installation.

### Step 1: Install Cosign

**macOS:**
```bash
brew install cosign
```

**Linux (x86_64/AMD64):**
```bash
curl -LO https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64
chmod +x cosign-linux-amd64
sudo mv cosign-linux-amd64 /usr/local/bin/cosign
```

**Linux (ppc64le/Power):**
```bash
curl -LO https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-ppc64le
chmod +x cosign-linux-ppc64le
sudo mv cosign-linux-ppc64le /usr/local/bin/cosign
```

### Step 2: Download and Verify Binary

Replace `BINARY_NAME` with your platform's binary from the table above:

```bash
VERSION="v0.3.0"
BINARY_NAME="ai-services-darwin-amd64"  # Change based on your platform

# Download binary, signature, and public key
curl -LO "https://github.com/IBM/project-ai-services/releases/download/${VERSION}/${BINARY_NAME}"
curl -LO "https://github.com/IBM/project-ai-services/releases/download/${VERSION}/${BINARY_NAME}.sig"
curl -LO "https://github.com/IBM/project-ai-services/releases/download/${VERSION}/cosign.pub"

# Verify signature
cosign verify-blob \
  --key cosign.pub \
  --signature ${BINARY_NAME}.sig \
  --insecure-ignore-tlog=true \
  ${BINARY_NAME}

# Install if verification succeeds
chmod +x ${BINARY_NAME}
sudo mv ${BINARY_NAME} /usr/local/bin/ai-services
ai-services version
```

---

## Container Image Verification

All AI Services container images are signed with Cosign for enhanced security and supply chain integrity.

### List Available Container Images

> **Note:** This command lists container images which are used in the project including third-party components. All images with the `icr.io/ai-services` registry prefix are built and maintained as part of this project. Only these images are signed and can be verified using the methods described in this document. Verification of third-party or custom images is outside the scope of this documentation.

To see all available container images for a specific application:

```bash
# List images for RAG application
ai-services application image list --runtime podman -t rag
```

### Verify Container Images

Ensure Cosign is installed (see [Verified Installation with Cosign](#verified-installation-with-cosign) section).

**Basic verification:**
```bash
VERSION="v0.3.0"
# Download public key if needed
curl -LO https://github.com/IBM/project-ai-services/releases/download/${VERSION}/cosign.pub

# Verify any image (replace with your image:tag)
cosign verify \
  --key cosign.pub \
  --insecure-ignore-tlog=true \
  icr.io/ai-services/tools:0.7
```

**Expected output on success:**
```
Verification for icr.io/ai-services/tools:0.7 --
The following checks were performed on each of these signatures:
  - The cosign claims were validated
  - The signatures were verified against the specified public key
```

---

## Embedding Service

Deploys a **CLIP ViT-B/32 embedding sidecar** on an IBM Power Linux LPAR (ppc64le, RHEL 9).
The service exposes a standard **OpenAI-compatible `POST /v1/embeddings` endpoint** on a host
port so any application, pipeline, or database can call it over HTTP — no proprietary SDK
or middleware required.

> **This is a standard AI Services application.** The CLI handles container image pulls and
> model downloads automatically — just like deploying the RAG stack. No manual pre-steps are
> required in normal (internet-connected) environments.

**Supported models (deployed with this application template):**

| Model | Modality | Vector dimensions |
|---|---|---|
| `openai/clip-vit-base-patch32` | Text + Image (multimodal) | 512-d float32 |
| `ibm-granite/granite-embedding-278m-multilingual` | Text (100+ languages) | 768-d float32 |

**Who can consume this endpoint:**

- Any **Python application** using `requests`, `httpx`, or the `openai` SDK
- Any **RAG pipeline** that accepts an OpenAI-compatible embedding backend (`POST /v1/embeddings`)
- Any **AI Services application** — the chatbot, digitize, and similarity services already consume this same API via [`services/common/emb_utils.py`](../services/common/emb_utils.py)
- Any **database or external system** that can make HTTP calls — e.g. Oracle 26ai on AIX via UTL_HTTP (see [example use case](#example-use-case-oracle-26ai-on-aix))

### Architecture

```
Any client (Python app, RAG pipeline, database, custom service)
  └─ HTTP POST /v1/embeddings
        │
        ▼
  Linux LPAR (RHEL 9, ppc64le, IBM Power)
  └─ podman pod: embedding--embedding
        └─ container: embedding
              └─ vLLM 0.19.1  ←  CLIP ViT-B/32 or Granite Embedding
```

### Prerequisites

| Requirement | Detail |
|---|---|
| Hardware | IBM Power11 LPAR, ppc64le |
| OS | RHEL 9.6 or higher |
| Container runtime | Podman |
| AI Services CLI | `ai-services` binary — download from [GitHub Releases](https://github.com/IBM/project-ai-services/releases) |
| Internet access | Required by default for CLI to pull image + model automatically. See [Air-gapped environments](#air-gapped-environments) if offline. |
| Memory | 6 GB minimum |
| CPU cores | 16 recommended |

### Step 1 — Install the AI Services CLI

Download the pre-built ppc64le binary (the `embedding` template is already embedded inside it):

```bash
VERSION="v0.3.0"
curl -LO "https://github.com/IBM/project-ai-services/releases/download/${VERSION}/ai-services-linux-ppc64le"
chmod +x ai-services-linux-ppc64le
sudo mv ai-services-linux-ppc64le /usr/local/bin/ai-services
ai-services version
```

> **If you are deploying from a cloned repository where the embedding template was added to source**,
> you must rebuild the CLI binary from source so the new template is embedded.
> The CLI compiles all templates under `ai-services/assets/` at build time (`//go:embed`).
>
> ```bash
> cd $HOME/project-ai-services/ai-services
> export PATH=$PATH:/usr/local/go/bin
> CGO_ENABLED=0 GOOS=linux GOARCH=ppc64le go build \
>   -o /usr/local/bin/ai-services \
>   -tags "exclude_graphdriver_btrfs containers_image_openpgp remote" \
>   ./cmd/ai-services
> ```

### Step 2 — Deploy the embedding service

One command deploys everything. The CLI automatically:
- Pulls the `icr.io/ppc64le-oss/vllm-ppc64le:0.19.1` container image (if not already present locally)
- Downloads the CLIP model from HuggingFace into `/var/lib/ai-services/models/` using the built-in tools container (`icr.io/ai-services/tools:0.11`)

```bash
ai-services application create embedding \
  --template embedding \
  --runtime podman \
  --legacy \
  --skip-validation spyre,numa
```

| Flag | Reason |
|---|---|
| `--template embedding` | Selects the embedding application template |
| `--runtime podman` | Deploys as a Podman pod on this LPAR |
| `--legacy` | Uses the embedded filesystem path (no Catalog API server required) |
| `--skip-validation spyre,numa` | Skips Spyre accelerator card check — this application is CPU-only |

Expected output:
```
✔ Application 'embedding' deployed successfully
```

### Step 3 — Verify the endpoint

```bash
# Get assigned host port and IP
ai-services application info embedding --runtime podman

# Health check
curl http://localhost:<PORT>/health
# {"status":"ok"}

# Test embedding generation (text input)
curl -s http://localhost:<PORT>/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{"model":"clip-vit-base-patch32","input":["blue running shoes"]}' \
  | python3 -c "
import json,sys
data=json.load(sys.stdin)
vec=data['data'][0]['embedding']
print(f'Dimensions: {len(vec)}')
print(f'First 5 values: {vec[:5]}')
"
# Dimensions: 512
# First 5 values: [0.123, -0.045, ...]
```

### Calling the endpoint from any application

The service is a plain HTTP server with an OpenAI-compatible API — any client works.

**Python (`openai` SDK):**
```python
from openai import OpenAI

client = OpenAI(base_url="http://<LPAR_IP>:<PORT>/v1", api_key="none")
response = client.embeddings.create(
    model="clip-vit-base-patch32",
    input=["blue running shoes", "red sneakers"]
)
for item in response.data:
    print(f"index={item.index}  dims={len(item.embedding)}")
```

**Python (plain `requests`):**
```python
import requests

resp = requests.post(
    "http://<LPAR_IP>:<PORT>/v1/embeddings",
    json={"model": "clip-vit-base-patch32", "input": ["blue running shoes"]}
)
vector = resp.json()["data"][0]["embedding"]  # list of 512 floats
```

**curl:**
```bash
curl -s http://<LPAR_IP>:<PORT>/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{"model":"clip-vit-base-patch32","input":["search query here"]}'
```

**As a RAG pipeline embedding backend:**

The AI Services chatbot, digitize, and similarity services all call `POST /v1/embeddings` through
[`services/common/emb_utils.py`](../services/common/emb_utils.py). To use this sidecar as their
embedding backend, set the `EMB_ENDPOINT` environment variable to `http://<LPAR_IP>:<PORT>`.

### Air-gapped environments

If the LPAR has no internet access, pre-stage the assets before running `create`:

**1 — Pre-pull the container image on a machine with access, export, and import:**
```bash
# On internet-connected machine:
podman pull icr.io/ppc64le-oss/vllm-ppc64le:0.19.1
podman save icr.io/ppc64le-oss/vllm-ppc64le:0.19.1 -o vllm-ppc64le-0.19.1.tar

# Transfer to LPAR, then:
podman load -i vllm-ppc64le-0.19.1.tar
```

**2 — Pre-download the model** into `/var/lib/ai-services/models/`:
```bash
# Using huggingface-cli (on a machine with access):
pip install huggingface_hub
huggingface-cli download openai/clip-vit-base-patch32 \
  --local-dir /var/lib/ai-services/models/openai/clip-vit-base-patch32
# Transfer the directory to the LPAR at the same path.
```

**3 — Deploy with skip flags** to tell the CLI both assets are already staged:
```bash
ai-services application create embedding \
  --template embedding \
  --runtime podman \
  --legacy \
  --skip-validation spyre,numa \
  --skip-model-download \
  --image-pull-policy IfNotPresent
```

### Example use case: Oracle 26ai on AIX

Oracle 26ai on AIX can call the endpoint directly from PL/SQL using UTL_HTTP — no middleware needed.
Replace `<LPAR_IP>` and `<PORT>` with the values from Step 3.

```sql
DECLARE
  l_req     UTL_HTTP.req;
  l_resp    UTL_HTTP.resp;
  l_buf     VARCHAR2(32767);
  l_body    CLOB    := EMPTY_CLOB();
  l_url     VARCHAR2(200) := 'http://<LPAR_IP>:<PORT>/v1/embeddings';
  l_payload VARCHAR2(4000) :=
    '{"model":"clip-vit-base-patch32","input":["blue running shoes"]}';
BEGIN
  l_req := UTL_HTTP.begin_request(l_url, 'POST', 'HTTP/1.1');
  UTL_HTTP.set_header(l_req, 'Content-Type',   'application/json');
  UTL_HTTP.set_header(l_req, 'Content-Length',  LENGTH(l_payload));
  UTL_HTTP.write_text(l_req, l_payload);
  l_resp := UTL_HTTP.get_response(l_req);
  LOOP
    UTL_HTTP.read_text(l_resp, l_buf, 32766);
    l_body := l_body || l_buf;
  END LOOP;
EXCEPTION
  WHEN UTL_HTTP.end_of_body THEN
    UTL_HTTP.end_response(l_resp);
    DBMS_OUTPUT.put_line(SUBSTR(l_body, 1, 500));
END;
/
```

Store the returned 512-d vector into an Oracle VECTOR column:
```sql
UPDATE products
SET    embedding = TO_VECTOR(:json_array)
WHERE  product_id = :id;
```

### Lifecycle commands

```bash
# Status
ai-services application ps embedding --runtime podman

# Endpoint info
ai-services application info embedding --runtime podman

# Logs
podman logs -f embedding--embedding-embedding

# Stop and remove
ai-services application delete embedding --runtime podman

# Restart (clean redeploy)
podman pod rm -f embedding--embedding
ai-services application create embedding \
  --template embedding --runtime podman --legacy \
  --skip-validation spyre,numa
```

---

## Additional Resources

- [Main README](../README.md) - Project overview and quick start
- [CONTRIBUTING.md](../CONTRIBUTING.md) - Contributing guidelines
- [GitHub Releases](https://github.com/IBM/project-ai-services/releases) - Download binaries
- [Cosign Documentation](https://docs.sigstore.dev/about/overview/) - Signature verification tool
