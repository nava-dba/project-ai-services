<!-- SPDX-License-Identifier: Apache-2.0 -->

- Your CLIP embedding sidecar is running. Oracle 26ai on AIX can now call it via UTL_HTTP.

- Run "ai-services application info {{ .AppName }} --runtime podman" to view the endpoint address and port.

- Call the endpoint from Oracle PL/SQL to generate embeddings:

    POST http://<linux-lpar-ip>:{{ .CLIP_PORT }}/v1/embeddings

  Example PL/SQL snippet:
    l_req  := UTL_HTTP.begin_request('http://<linux-lpar-ip>:{{ .CLIP_PORT }}/v1/embeddings', 'POST');
    UTL_HTTP.set_header(l_req, 'Content-Type', 'application/json');
    UTL_HTTP.write_text(l_req, '{"model":"openai/clip-vit-base-patch32","input":["' || :text || '"]}');

- Store the returned 512-d float array directly into an Oracle VECTOR column:
    UPDATE fashion_products SET description_embedding = :vec WHERE product_id = :id;
