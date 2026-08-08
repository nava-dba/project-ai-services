# embedding — CLIP Embedding Service

Deploys a **CLIP ViT-B/32 embedding sidecar** on an IBM Power Linux LPAR (ppc64le, RHEL 9).
Exposes a standard **OpenAI-compatible `POST /v1/embeddings` endpoint** on a host port so any
application, pipeline, or database can call it over plain HTTP — no middleware required.

---

## What it does

- Serves the `openai/clip-vit-base-patch32` model via vLLM 0.19.1 as a single Podman pod
- Accepts text strings and base64-encoded images, returns **512-dimensional float32 vectors**
- No OpenSearch, no RAG pipeline, no vector database required — the calling application handles storage
- Any client that can make an HTTP POST request can use it

**Supported callers (non-exhaustive):**

| Caller | How |
|---|---|
| Python application | `openai` SDK, `requests`, `httpx` |
| RAG pipeline | Point `EMB_ENDPOINT` at this service; uses same API as all AI Services components |
| Any application | Plain HTTP `POST /v1/embeddings` |
| Database (e.g. Oracle 26ai) | HTTP call from PL/SQL using `UTL_HTTP` |
| Shell / CI pipeline | `curl` |

---

## Architecture

```
Any client (Python app, RAG pipeline, database, shell script, …)
  └─ HTTP POST /v1/embeddings
        │
        ▼
  Linux LPAR (RHEL 9, ppc64le, IBM Power)
  └─ podman pod: embedding--embedding
        └─ container: embedding
              └─ vLLM 0.19.1 serving CLIP ViT-B/32
                    └─ /var/lib/ai-services/models/openai/clip-vit-base-patch32/
```

---

## Prerequisites

| Requirement | Detail |
|---|---|
| Hardware | IBM Power11 LPAR, ppc64le |
| OS | RHEL 9.6 or higher |
| Container runtime | Podman |
| AI Services CLI | `ai-services` binary — see [Step 1](#step-1--install-the-ai-services-cli) |
| Internet access | Required by default; see [Air-gapped environments](#air-gapped-environments) if offline |
| Memory | 6 GB minimum |
| CPU cores | 16 recommended |

---

## Step 1 — Install the AI Services CLI

Download the pre-built ppc64le binary. The `embedding` template is already compiled inside it:

```bash
VERSION="v0.3.0"
curl -LO "https://github.com/IBM/project-ai-services/releases/download/${VERSION}/ai-services-linux-ppc64le"
chmod +x ai-services-linux-ppc64le
sudo mv ai-services-linux-ppc64le /usr/local/bin/ai-services
ai-services version
```

> **Deploying from a cloned repo with local template changes?**
> The CLI embeds all templates under `assets/` at build time (`//go:embed applications`).
> Rebuild after any change:
>
> ```bash
> cd $HOME/project-ai-services/ai-services
> export PATH=$PATH:/usr/local/go/bin
> CGO_ENABLED=0 GOOS=linux GOARCH=ppc64le go build \
>   -o /usr/local/bin/ai-services \
>   -tags "exclude_graphdriver_btrfs containers_image_openpgp remote" \
>   ./cmd/ai-services
> ```

---

## Step 2 — Deploy

One command deploys everything. The CLI automatically pulls the container image and downloads
the model from HuggingFace using the built-in tools container:

```bash
ai-services application create embedding \
  --template embedding \
  --runtime podman \
  --legacy \
  --skip-validation spyre,numa
```

| Flag | Reason |
|---|---|
| `--template embedding` | Selects this application template |
| `--runtime podman` | Deploys as a Podman pod on this LPAR |
| `--legacy` | Uses the embedded filesystem path; no Catalog API server required |
| `--skip-validation spyre,numa` | This application is CPU-only — no Spyre accelerator cards needed |

Expected output:
```
✔ Application 'embedding' deployed successfully
```

---

## Step 3 — Verify

```bash
# Get the assigned host port and IP
ai-services application info embedding --runtime podman

# Health check
curl http://localhost:<PORT>/health
# {"status":"ok"}

# Test — text embedding
curl -s http://localhost:<PORT>/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{"model":"clip-vit-base-patch32","input":["blue running shoes"]}' \
  | python3 -c "
import json, sys
data = json.load(sys.stdin)
vec = data['data'][0]['embedding']
print(f'Dimensions : {len(vec)}')
print(f'First 5    : {vec[:5]}')
"
# Dimensions : 512
# First 5    : [0.123, -0.045, ...]
```

---

## Calling the endpoint

Replace `<LPAR_IP>` and `<PORT>` with the values from Step 3.

### Python (`openai` SDK)

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

### Python (plain `requests`)

```python
import requests

resp = requests.post(
    "http://<LPAR_IP>:<PORT>/v1/embeddings",
    json={"model": "clip-vit-base-patch32", "input": ["blue running shoes"]}
)
vector = resp.json()["data"][0]["embedding"]  # list of 512 floats
```

### curl

```bash
curl -s http://<LPAR_IP>:<PORT>/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{"model":"clip-vit-base-patch32","input":["search query here"]}'
```

### As a RAG pipeline embedding backend

All AI Services components (chatbot, digitize, similarity) call `POST /v1/embeddings` through
`services/common/emb_utils.py`. To use this sidecar as their embedding backend, set:

```bash
EMB_ENDPOINT=http://<LPAR_IP>:<PORT>
```

---

## Air-gapped environments

For LPARs with no internet access, pre-stage both assets, then deploy with skip flags.

**1 — Export the container image on a connected machine and import on the LPAR:**

```bash
# Connected machine:
podman pull icr.io/ppc64le-oss/vllm-ppc64le:0.19.1
podman save icr.io/ppc64le-oss/vllm-ppc64le:0.19.1 -o vllm-ppc64le-0.19.1.tar
# Transfer tar to LPAR, then:
podman load -i vllm-ppc64le-0.19.1.tar
```

**2 — Pre-download the model** into `/var/lib/ai-services/models/`:

```bash
pip install huggingface_hub
huggingface-cli download openai/clip-vit-base-patch32 \
  --local-dir /var/lib/ai-services/models/openai/clip-vit-base-patch32
# Transfer the directory to the LPAR at the same path.
```

**3 — Deploy with skip flags:**

```bash
ai-services application create embedding \
  --template embedding \
  --runtime podman \
  --legacy \
  --skip-validation spyre,numa \
  --skip-model-download \
  --image-pull-policy IfNotPresent
```

---

## Lifecycle commands

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

## Example use case: Oracle 26ai on AIX

Oracle 26ai on AIX can call this endpoint directly from PL/SQL using `UTL_HTTP` to generate vectors
and store them in a `VECTOR` column — no middleware, no ODBC driver, no extra service required.

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

Store the returned 512-d float32 vector directly into an Oracle VECTOR column:

```sql
UPDATE products
SET    embedding = TO_VECTOR(:json_array)
WHERE  product_id = :id;
```

Oracle then handles all vector storage, HNSW indexing, and `VECTOR_DISTANCE()` similarity search.

---

## Template files

| File | Purpose |
|---|---|
| `metadata.yaml` | Application name and description |
| `podman/metadata.yaml` | Podman runtime metadata and pod template list |
| `podman/values.yaml` | Default values — image, model path, port |
| `podman/templates/clip-embedding.yaml.tmpl` | Pod spec rendered by the CLI at deploy time |
| `podman/steps/next.md` | Post-deploy message printed by the CLI |
| `podman/steps/info.md` | Output template for `ai-services application info` |
| `podman/steps/vars_file.yaml` | Variable definitions used by step templates |

---

## Technical notes — vLLM IPC fix

vLLM 0.19.1 V1 engine uses multiprocessing with TCP IPC between the `APIServer` (pid 1) and the
`EngineCore` subprocess. Inside a `podman kube play` pod the TCP socket connection fails silently,
causing `RuntimeError: Engine core initialization failed`.

**Fix applied in `clip-embedding.yaml.tmpl`:**

```yaml
env:
  - name: VLLM_RPC_BASE_PATH
    value: "/run/vllm_ipc"        # forces Unix domain socket instead of TCP
  - name: VLLM_WORKER_MULTIPROC_METHOD
    value: "fork"                  # fork instead of spawn — avoids re-import overhead
  - name: VLLM_CPU_KVCACHE_SPACE
    value: "1"                     # 1 GiB KV cache (CLIP context is only 77 tokens)
```

Env vars that do **not** work in this build:
- `VLLM_USE_V1=0` — not present in `vllm.envs` for 0.19.1
- `VLLM_ENABLE_V1_MULTIPROCESSING=0` — present but only affects a random seed warning; does not disable multiprocessing

---

## Model details

| Property | Value |
|---|---|
| Model | CLIP ViT-B/32 (`openai/clip-vit-base-patch32`) |
| Vector dimensions | 512 |
| Max sequence length | 77 tokens |
| Modalities | Text + Image |
| Disk size | ~1.82 GB |
| vLLM runner | `pooling` (auto-resolved) |
| Architecture | `CLIPModel` |
