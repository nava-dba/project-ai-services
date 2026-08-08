<!-- SPDX-License-Identifier: Apache-2.0 -->

{{- if ne .CLIP_PORT "" }}
{{- if eq .CLIP_STATUS "running" }}

- CLIP Embedding API is ready at http://{{ .HOST_IP }}:{{ .CLIP_PORT }}/v1/embeddings

  Use this endpoint in Oracle PL/SQL (UTL_HTTP) to generate 512-d vectors:

    POST http://{{ .HOST_IP }}:{{ .CLIP_PORT }}/v1/embeddings
    Content-Type: application/json

    -- Text embedding:
    {"model": "openai/clip-vit-base-patch32", "input": ["blue denim jacket"]}

    -- Image embedding (base64):
    {"model": "openai/clip-vit-base-patch32", "input": [{"type": "image_url", "image_url": {"url": "data:image/jpeg;base64,<BASE64>"}}]}

{{- else }}

- CLIP Embedding service is not yet running. Please make sure '{{ .AppName }}--embedding' pod is running.

{{- end }}
{{- end }}
