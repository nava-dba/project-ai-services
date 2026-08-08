<!-- SPDX-License-Identifier: Apache-2.0 -->

- Your embedding service is running and ready to accept requests.

- Endpoint: POST http://{{ .HOST_IP }}:{{ .EMBEDDING_PORT }}/v1/embeddings

- Text embedding example:
    curl -s http://{{ .HOST_IP }}:{{ .EMBEDDING_PORT }}/v1/embeddings \
      -H "Content-Type: application/json" \
      -d '{"model":"{{ .EMBEDDING_MODEL }}","input":["your text here"]}'

- Run "ai-services application info <app-name> --runtime podman" to view endpoint details.
