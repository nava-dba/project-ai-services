# embedding — CLIP Embedding Sidecar for Oracle 26ai

Deploys a CLIP ViT-B/32 embedding sidecar on an IBM Power Linux LPAR (ppc64le, RHEL 9).  
Oracle 26ai on AIX calls `POST /v1/embeddings` via `UTL_HTTP` PL/SQL to generate 512-d vectors
for similarity search using `VECTOR_DISTANCE()`.

---

## Architecture

```
Oracle 26ai (AIX)
  └─ UTL_HTTP POST /v1/embeddings
        │  TCP over network
        ▼
  Linux LPAR (RHEL 9, ppc64le, IBM Power)
  └─ podman pod: embedding--embedding
        └─ container: embedding
              └─ vLLM 0.19.1 serving CLIP ViT-B/32
                    └─ /var/lib/ai-services/models/openai/clip-vit-base-patch32/
```

Oracle handles all vector storage, HNSW indexing, and `VECTOR_DISTANCE()` — no OpenSearch or RAG pipeline needed.

---

## Prerequisites

| Requirement | Detail |
|---|---|
| Hardware | IBM Power11 LPAR, ppc64le |
| OS | RHEL 9.6 or higher |
| Container runtime | Podman |
| AI Services CLI | `ai-services` binary built from this repo |
| vLLM image | `icr.io/ppc64le-oss/vllm-ppc64le:0.19.1` (pulled locally) |
| Model | CLIP ViT-B/32 downloaded to `/var/lib/ai-services/models/openai/clip-vit-base-patch32/` |

---

## Step 1 — Pull the vLLM image

```bash
podman pull icr.io/ppc64le-oss/vllm-ppc64le:0.19.1
```

Verify:
```bash
podman images | grep vllm
# icr.io/ppc64le-oss/vllm-ppc64le   0.19.1   cc15dc1b2669   ...   3.45 GB
```

---

## Step 2 — Download the CLIP model

```bash
mkdir -p /var/lib/ai-services/models/openai/clip-vit-base-patch32

# Using huggingface-cli (requires huggingface_hub installed)
huggingface-cli download openai/clip-vit-base-patch32 \
  --local-dir /var/lib/ai-services/models/openai/clip-vit-base-patch32

# Verify (should show ~11 files, ~1.82 GB)
ls -lh /var/lib/ai-services/models/openai/clip-vit-base-patch32/
```

---

## Step 3 — Build the AI Services CLI binary

The CLI embeds all application templates at build time. Rebuild after any template change.

```bash
cd $HOME/project-ai-services/ai-services
export PATH=$PATH:/usr/local/go/bin

CGO_ENABLED=0 GOOS=linux GOARCH=ppc64le go build \
  -o /usr/local/bin/ai-services \
  -tags "exclude_graphdriver_btrfs containers_image_openpgp remote" \
  ./cmd/ai-services

ai-services --version
```

> **Go location:** `/usr/local/go/bin/go`
> **Repo location on LPAR:** `$HOME/project-ai-services/`

---

## Step 4 — Deploy the embedding sidecar

```bash
ai-services application create embedding \
  --template embedding \
  --runtime podman \
  --legacy \
  --skip-validation spyre,numa \
  --skip-model-download \
  --image-pull-policy IfNotPresent
```

Flag reference:

| Flag | Reason |
|---|---|
| `--template embedding` | Matches `assets/applications/embedding/` |
| `--legacy` | Bypasses Catalog API; reads templates directly from embedded FS |
| `--skip-validation spyre,numa` | No Spyre accelerator cards on this LPAR (CPU-only) |
| `--skip-model-download` | Model already present at `/var/lib/ai-services/models/` |
| `--image-pull-policy Never` | Image is already pulled locally; no registry access needed |

Expected output ends with:
```
✔ Application 'embedding' deployed successfully
```

---

## Step 5 — Verify the endpoint

```bash
# Get the assigned host port
ai-services application info embedding --runtime podman

# Quick health check
curl http://localhost:<PORT>/health
# {"status":"ok"}

# Test embedding generation
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

---

## Step 6 — Call from Oracle PL/SQL (UTL_HTTP)

Replace `<LINUX_LPAR_IP>` with the IP of your Linux LPAR and `<PORT>` with the port from Step 5.

```sql
DECLARE
  l_req   UTL_HTTP.req;
  l_resp  UTL_HTTP.resp;
  l_buf   VARCHAR2(32767);
  l_body  CLOB := EMPTY_CLOB();
  l_url   VARCHAR2(200) := 'http://<LINUX_LPAR_IP>:<PORT>/v1/embeddings';
  l_input VARCHAR2(4000) := 'blue running shoes';
BEGIN
  l_req := UTL_HTTP.begin_request(l_url, 'POST', 'HTTP/1.1');
  UTL_HTTP.set_header(l_req, 'Content-Type', 'application/json');
  UTL_HTTP.set_header(l_req, 'Accept',       'application/json');

  DECLARE
    l_payload VARCHAR2(4000) :=
      '{"model":"clip-vit-base-patch32","input":["' || l_input || '"]}';
  BEGIN
    UTL_HTTP.set_header(l_req, 'Content-Length', LENGTH(l_payload));
    UTL_HTTP.write_text(l_req, l_payload);
  END;

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

Store the resulting vector into an Oracle 26ai VECTOR column:
```sql
UPDATE fashion_products
SET    description_embedding = TO_VECTOR(:embedding_json_array)
WHERE  product_id = :id;
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
  --skip-validation spyre,numa --skip-model-download --image-pull-policy IfNotPresent
```

---

## Template files

| File | Purpose |
|---|---|
| `metadata.yaml` | Application name and description |
| `podman/metadata.yaml` | Podman runtime metadata |
| `podman/values.yaml` | Default values (image, model path, port) |
| `podman/templates/clip-embedding.yaml.tmpl` | Pod spec rendered by the CLI |
| `podman/steps/next.md` | Post-deploy instructions printed by CLI |
| `podman/steps/info.md` | Info command output template |
| `podman/steps/vars_file.yaml` | Variable definitions for step templates |

---

## Key discoveries (IPC fix)

vLLM 0.19.1 V1 engine uses multiprocessing with TCP IPC between the `APIServer` (pid=1)
and the `EngineCore` subprocess. Inside a `podman kube play` pod the TCP socket
connection fails silently, causing `RuntimeError: Engine core initialization failed`.

**Fix applied in `clip-embedding.yaml.tmpl`:**

```yaml
env:
  - name: VLLM_RPC_BASE_PATH
    value: "/run/vllm_ipc"          # forces Unix domain socket instead of TCP
  - name: VLLM_WORKER_MULTIPROC_METHOD
    value: "fork"                    # fork instead of spawn — avoids re-import overhead
  - name: VLLM_CPU_KVCACHE_SPACE
    value: "1"                       # 1 GiB KV cache (CLIP has tiny context: 77 tokens)
```

Env vars that do **not** work in this build:
- `VLLM_USE_V1=0` — not present in `vllm.envs` for 0.19.1
- `VLLM_ENABLE_V1_MULTIPROCESSING=0` — present but only affects random seed warning; does not disable multiprocessing

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
