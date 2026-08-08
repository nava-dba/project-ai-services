<!-- SPDX-License-Identifier: Apache-2.0 -->

Day N:

{{- if ne .EMBEDDING_PORT "" }}
{{- if eq .EMBEDDING_STATUS "running" }}

- Embedding API is available at http://{{ .HOST_IP }}:{{ .EMBEDDING_PORT }}/v1/embeddings

  Text embedding:
    POST http://{{ .HOST_IP }}:{{ .EMBEDDING_PORT }}/v1/embeddings
    Body: {"model":"{{ .EMBEDDING_MODEL }}","input":["your text here"]}

{{- else }}

- Embedding API is unavailable. Please make sure the 'embedding' pod is running.
{{- end }}
{{- end }}
