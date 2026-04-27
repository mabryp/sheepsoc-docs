# Services

**Purpose:** What runs on sheepsoc, where it lives, and how to check it.

## Service Catalog

Every service running on sheepsoc, its systemd unit name, port, and current operational status. Services marked **hold** are installed but intentionally stopped pending a rebuild or external dependency.

| Service | Unit / cmd | Port | Purpose | Status |
|---|---|---|---|---|
| **sheepsoc-landing** | `sheepsoc-landing.service` | 80 | LAN landing page and these docs (Python http.server) | up |
| **Vikunja** | `vikunja.service` | 3000 | Self-hosted kanban / task manager | up |
| **Elasticsearch** | `elasticsearch.service` | 9200 | Elasticsearch 8.19.14 · single-node cluster *sheepsoc* · data at `/mnt/elastic_data` · **auth enabled** (`xpack.security` on, no TLS) · vector store for OpenWebUI RAG | up |
| **Kibana** | `kibana.service` | 5601 | Log & metrics visualization (Filebeat / Metricbeat / syslog) | up |
| **Logstash** | `logstash.service` | 5514/udp | Syslog ingestion from ASUS router + OPNsense | up |
| **Filebeat** | `filebeat.service` | — | Ships local log files to Elasticsearch | up |
| **Metricbeat** | `metricbeat.service` | — | Ships host metrics (CPU, RAM, disk, net) to Elasticsearch | up |
| **Open WebUI** | `open-webui.service` | 8080 | **Primary AI interface.** OpenWebUI 0.6.12 · browser-based chat and RAG frontend · connects to Ollama for LLM inference · RAG via Elasticsearch (`nomic-embed-text`, 768d, HNSW/cosine) · runs in `openwebui` conda env (Python 3.11) | up |
| **Jupyter Notebook** | `jupyter.service` | 8888 | Notebook server, notebook dir `~/repositories/sheepsoc` | up |
| **Ollama** | `ollama.service` | 11434 | Local LLM inference (uses RTX 5060 Ti) | up |
| **SSH** | `ssh.service` | 22 | Remote shell · key auth only | up |
| **cron** | `cron.service` | — | Scheduled tasks | up |
| **MicroK8s** | `snap.microk8s.*` | — | Kubernetes — *stopped*, needs rebuild | **hold** |
| **Matrix Bot** | `matrix-bot.service` | — | E2EE Matrix bot (`@sheepsoc-bot:matrix.pmabry.com`) · bridges Element rooms to OpenWebUI RAG + Ollama · runs in `matrixbot` conda env | up |
| **OpenProject** | `docker: openproject` | 3001 | Project management (tasks, milestones, Gantt) · OpenProject 15 all-in-one Docker image · data at `/mnt/ssd_working/openproject/` · installed 2026-04-24 | up |

## Health Checks

The first question is always: *is it up?* The second is *is it healthy?* These commands answer both for every service on the box.

### Check One Service

```bash
pmabry@sheepsoc:~$ systemctl status ollama.service
pmabry@sheepsoc:~$ systemctl status elasticsearch.service
pmabry@sheepsoc:~$ systemctl status kibana.service
pmabry@sheepsoc:~$ systemctl status jupyter.service
```

### Check Everything at Once

```bash
# list every service and whether it is running or failed
pmabry@sheepsoc:~$ systemctl list-units --type=service --state=running
pmabry@sheepsoc:~$ systemctl list-units --type=service --state=failed

# the short, human-friendly version
pmabry@sheepsoc:~$ systemctl --failed
```

### Poke Each Service at Its Port

```bash
# Elasticsearch cluster health — auth required (xpack.security enabled 2026-04-21)
pmabry@sheepsoc:~$ curl -s -u elastic:<password> http://localhost:9200/_cluster/health | jq

# Ollama — lists loaded models
pmabry@sheepsoc:~$ curl -s http://localhost:11434/api/tags | jq

# Kibana status
pmabry@sheepsoc:~$ curl -s http://localhost:5601/api/status | jq

# Vikunja info
pmabry@sheepsoc:~$ curl -s http://localhost:3000/api/v1/info | jq

# OpenProject (Docker) — HTTP 200 = healthy
pmabry@sheepsoc:~$ curl -s -o /dev/null -w "%{http_code}" http://192.168.50.100:3001/
```

### GPU Sanity Check (for Ollama)

```bash
pmabry@sheepsoc:~$ nvidia-smi
# When a query is running, you should see an `ollama` process holding VRAM.
# Idle Ollama may hold 0 MB — that is fine.
```

## Start / Stop / Restart

!!! warning "Careful"
    `systemctl` actions that change state require `sudo`. These are not silent — tell Phillip what you are restarting and why before doing it.

```bash
# standard lifecycle
pmabry@sheepsoc:~$ sudo systemctl restart ollama.service
pmabry@sheepsoc:~$ sudo systemctl stop    jupyter.service
pmabry@sheepsoc:~$ sudo systemctl start   jupyter.service

# enable/disable on boot
pmabry@sheepsoc:~$ sudo systemctl enable  metricbeat.service
pmabry@sheepsoc:~$ sudo systemctl disable <service>
```

## Reading Service Logs

Per-service logs, most recent first, with optional follow:

```bash
pmabry@sheepsoc:~$ journalctl -u ollama.service -n 200 --no-pager
pmabry@sheepsoc:~$ journalctl -u elasticsearch.service -f
pmabry@sheepsoc:~$ journalctl -u logstash.service --since "10 min ago"
```

For ES / syslog / beats data, Kibana is usually easier — see [SOPs: Reading Logs in Kibana](sops.md#5-read-logs-in-kibana).

## Web Endpoints

All web interfaces are LAN-only (UFW restricts access to 192.168.50.0/24).

| Service | URL | Notes |
|---|---|---|
| Landing | `http://192.168.50.100/` | Dashboard |
| Docs (this site) | `http://192.168.50.100/docs/` | You are here |
| Vikunja | `http://192.168.50.100:3000` | Kanban |
| OpenProject | `http://192.168.50.100:3001` | Project management — Docker container |
| Jupyter | `http://192.168.50.100:8888` | Token in `journalctl -u jupyter` |
| Kibana | `http://192.168.50.100:5601` | Log / metric dashboards |
| Elasticsearch | `http://192.168.50.100:9200` | REST API, not a UI |
| Open WebUI | `http://192.168.50.100:8080` | Primary AI interface — chat + RAG |
| Ollama | `http://192.168.50.100:11434` | REST API, not a UI |
| ASUS router | `http://192.168.50.1` | Admin |
| OPNsense | `https://192.168.50.253` | Self-signed cert |

## OpenWebUI RAG Configuration

OpenWebUI is configured to use Elasticsearch as its vector store rather than the default Chroma backend. This was set up on 2026-04-20. The configuration lives in a systemd drop-in file that injects environment variables into the service at startup.

### Key Config File

`/etc/systemd/system/open-webui.service.d/elasticsearch.conf`

```ini
[Service]
Environment="VECTOR_DB=elasticsearch"
Environment="ELASTICSEARCH_URL=http://127.0.0.1:9200"
Environment="ELASTICSEARCH_USERNAME=elastic"
Environment="ELASTICSEARCH_PASSWORD=<elastic-password>"
Environment="RAG_EMBEDDING_ENGINE=ollama"
Environment="RAG_EMBEDDING_MODEL=nomic-embed-text"
Environment="ELASTICSEARCH_INDEX_PREFIX=open_webui_collections"
```

!!! note "Auth Note"
    `xpack.security` was enabled on 2026-04-21. The drop-in now includes `ELASTICSEARCH_USERNAME` and `ELASTICSEARCH_PASSWORD`. Kibana connects as `kibana_system`. Filebeat, Metricbeat, and Logstash connect as the custom `beats_writer` user — see [Elasticsearch Auth for Data Shippers](#elasticsearch-authentication-for-data-shippers) for why `beats_system` must *not* be used.

### ES Index

| Key | Value |
|---|---|
| Index name | `open_webui_collections_d768` |
| Embedding model | `nomic-embed-text:latest` via Ollama · 768 dimensions |
| Similarity | HNSW / cosine |
| Vector field | `vector` — 768-dim dense float · written by OpenWebUI at ingest |
| Sparse field | `text_elser` — ELSER sparse tokens · written by ingest pipeline (added 2026-04-23) |
| Ingest pipeline | `elser-embed-openwebui` — set as index default; auto-runs on every new OpenWebUI upload |
| ELSER model | `.elser-2-elasticsearch` — built-in, adaptive allocation (scales to zero when idle) |
| Document count | 8,136 chunks as of 2026-04-23 · all backfilled with ELSER tokens |

### OpenWebUI RAG Troubleshooting

| Symptom | Check |
|---|---|
| OpenWebUI won't start | `journalctl -u open-webui.service -n 50` — look for `ImportError` on `elasticsearch`. If the `openwebui` conda env was rebuilt, the `elasticsearch==8.19.3` pip package must be reinstalled manually (it is not bundled). |
| Uploads fail silently | Confirm Ollama is up (`systemctl status ollama`) and `nomic-embed-text` is present (`ollama list`). |
| RAG returns nothing | Check ES is up: `curl -u elastic:<password> http://localhost:9200/_cluster/health` |
| Roll back to Chroma | Delete the drop-in: `sudo rm /etc/systemd/system/open-webui.service.d/elasticsearch.conf`, then `sudo systemctl daemon-reload && sudo systemctl restart open-webui`. |

### Check RAG Index Document Count

```bash
pmabry@sheepsoc:~$ curl -s -u elastic:<password> \
    "http://localhost:9200/open_webui_collections_d768/_count" | python3 -m json.tool
```

## ELSER Sparse Embeddings — Querying from Outside OpenWebUI

Since 2026-04-23, the `open_webui_collections_d768` index is dual-use. OpenWebUI continues to do kNN search on the `vector` field exactly as before. In addition, every document now carries a `text_elser` sparse vector field populated by the `elser-embed-openwebui` ingest pipeline.

!!! note "OpenWebUI Unchanged"
    OpenWebUI chat behaviour is completely unaffected. It still performs kNN search on `vector`. The ELSER field and the query patterns below are for direct Elasticsearch queries you run yourself — from the CLI, scripts, or Kibana Dev Tools.

### Pattern 1 — Pure ELSER Search (`text_expansion`)

Passes your query through ELSER at query time and matches against stored sparse vectors. No pre-computed query vector needed. Good for keyword-intent queries.

```bash
pmabry@sheepsoc:~$ curl -s -u elastic:<password> \
    -X POST "http://localhost:9200/open_webui_collections_d768/_search" \
    -H "Content-Type: application/json" \
    -d '{
  "size": 5,
  "query": {
    "text_expansion": {
      "text_elser": {
        "model_id": ".elser-2-elasticsearch",
        "model_text": "your search query here"
      }
    }
  },
  "_source": ["text", "metadata.name", "metadata.page"]
}' | python3 -m json.tool
```

### Pattern 2 — Hybrid RRF Search (ELSER + kNN)

Combines keyword-intent retrieval (ELSER) and semantic similarity retrieval (kNN) using Reciprocal Rank Fusion. Best overall recall.

```bash
# Step 1: get query embedding from Ollama
pmabry@sheepsoc:~$ QUERY_VEC=$(curl -s "http://localhost:11434/v1/embeddings" \
    -H "Content-Type: application/json" \
    -d '{"model": "nomic-embed-text", "input": "your search query here"}' \
    | python3 -c "import json,sys; print(json.load(sys.stdin)['data'][0]['embedding'])")

# Step 2: hybrid RRF query
pmabry@sheepsoc:~$ curl -s -u elastic:<password> \
    -X POST "http://localhost:9200/open_webui_collections_d768/_search" \
    -H "Content-Type: application/json" \
    -d "{
  \"size\": 5,
  \"retriever\": {
    \"rrf\": {
      \"retrievers\": [
        {\"knn\": {\"field\": \"vector\", \"query_vector\": $QUERY_VEC, \"k\": 10, \"num_candidates\": 50}},
        {\"standard\": {\"query\": {\"text_expansion\": {\"text_elser\": {\"model_id\": \".elser-2-elasticsearch\", \"model_text\": \"your search query here\"}}}}}
      ],
      \"rank_window_size\": 10,
      \"rank_constant\": 60
    }
  },
  \"_source\": [\"text\", \"metadata.name\", \"metadata.page\"]
}" | python3 -m json.tool
```

!!! note
    A detailed walk-through of the pipeline setup, backfill procedure, field mapping, and all three query patterns lives at `~/infrastructure/elasticsearch/howto-elser-openwebui-pipeline.md`.

### Verify ELSER Coverage

```bash
# Docs with ELSER tokens (should equal total doc count)
pmabry@sheepsoc:~$ curl -s -u elastic:<password> \
    "http://localhost:9200/open_webui_collections_d768/_count" \
    -H "Content-Type: application/json" \
    -d '{"query": {"exists": {"field": "text_elser"}}}' \
    | python3 -c "import json,sys; d=json.load(sys.stdin); print(f'With text_elser: {d[\"count\"]}')"

# Docs still missing ELSER tokens (should be 0)
pmabry@sheepsoc:~$ curl -s -u elastic:<password> \
    "http://localhost:9200/open_webui_collections_d768/_count" \
    -H "Content-Type: application/json" \
    -d '{"query": {"bool": {"must_not": {"exists": {"field": "text_elser"}}}}}' \
    | python3 -c "import json,sys; d=json.load(sys.stdin); print(f'Still missing: {d[\"count\"]}')"
```

!!! warning "Rebuild Note"
    If the `openwebui` conda env is ever rebuilt, you must manually reinstall the Elasticsearch client: `pip install elasticsearch==8.19.3` (inside the activated env). It is not bundled with OpenWebUI and is easy to miss.

## Elasticsearch Authentication for Data Shippers

Filebeat, Metricbeat, Logstash, and any future Elastic data shipper must use the `beats_writer` custom user when connecting to Elasticsearch. Using the built-in `beats_system` account for this purpose is a common mistake — do not repeat it.

!!! danger "Gotcha — Discovered 2026-04-21"
    **Do not use `beats_system` for data ingestion.** Despite its name, `beats_system` only has monitor access and can only write to internal `.monitoring-beats-*` indices. It cannot perform ILM setup calls or write to regular data streams. Any shipper configured with this account will fail immediately with:

    `action [cluster:admin/ilm/get] is unauthorized for user [beats_system]`

### Why beats_system Fails

Elasticsearch ships with a small set of reserved built-in users. Each is scoped to a specific internal purpose. `beats_system` exists only so the beats monitoring module can report its own health to Elasticsearch — it was never intended for general data ingestion. When a shipper tries to set up ILM policies, create index templates, or write to a data stream like `filebeat-8.18.3`, those actions all require cluster-level privileges that `beats_system` does not have.

### The Correct User: beats_writer

`beats_writer` is a custom role and user created on sheepsoc on 2026-04-21. It holds exactly the privileges that Filebeat, Metricbeat, and Logstash need to operate fully — no more, no less.

#### Cluster Privileges

| Privilege | Why It Is Needed |
|---|---|
| `monitor` | Read cluster health and node info |
| `manage_ilm` | Create and update ILM lifecycle policies |
| `manage_index_templates` | Register composable index templates |
| `manage_pipeline` | Install ingest pipelines (e.g. geoip) |
| `manage_ingest_pipelines` | Alias; covers pipeline CRUD via beats APIs |

#### Index Patterns and Privileges

| Index Pattern | Privileges |
|---|---|
| `filebeat-*` · `.ds-filebeat-*` | `create_index`, `create`, `write`, `manage`, `auto_configure` |
| `metricbeat-*` · `.ds-metricbeat-*` | (same) |
| `logstash-*` · `.ds-logstash-*` | (same) |
| `logs-*` · `.ds-logs-*` | (same) |

### Services Currently Using beats_writer

| Service | Config File | Field |
|---|---|---|
| Filebeat | `/home/pmabry/infrastructure/filebeat/filebeat-8.18.3-linux-x86_64/filebeat.yml` | `output.elasticsearch.username` |
| Metricbeat | `/home/pmabry/infrastructure/metricbeat/metricbeat-8.18.2-linux-x86_64/metricbeat.yml` | `output.elasticsearch.username` |
| Logstash | `/etc/logstash/conf.d/99-elasticsearch-output.conf` | `user` field in each `elasticsearch` output block |

### Rule for Adding New Data Shippers

Any new Filebeat, Metricbeat, Logstash, or other Elastic shipper added to sheepsoc must use `beats_writer` credentials in its Elasticsearch output block. If the new shipper writes to index patterns not yet covered by the role, update the role first before starting the shipper.

```bash
# Update the beats_writer role to add new index patterns
pmabry@sheepsoc:~$ curl -s -u elastic:<password> \
    -X PUT "http://localhost:9200/_security/role/beats_writer" \
    -H "Content-Type: application/json" \
    -d '{
  "cluster": ["monitor","manage_ilm","manage_index_templates","manage_pipeline","manage_ingest_pipelines"],
  "indices": [
    {
      "names": ["filebeat-*",".ds-filebeat-*","metricbeat-*",".ds-metricbeat-*",
                "logstash-*",".ds-logstash-*","logs-*",".ds-logs-*"],
      "privileges": ["create_index","create","write","manage","auto_configure"]
    }
  ]
}' | python3 -m json.tool
```

## Kibana to Ollama Connector

Kibana includes a built-in OpenAI-compatible connector (`.gen-ai` connector type) that can be pointed at the local Ollama instance. `xpack.security` must be enabled for connectors to be available in Kibana (enabled 2026-04-21).

### The URL Gotcha — Read This First

When the **API Provider** is set to *Other*, Kibana posts directly to the `apiUrl` value as-is. It does **not** treat it as a base URL and append `/chat/completions`.

!!! warning "Gotcha"
    The API URL field must contain the **full endpoint path**, not just the base URL.

| Setting | Wrong | Right |
|---|---|---|
| API URL | `http://192.168.50.100:11434/v1` | `http://192.168.50.100:11434/v1/chat/completions` |

### Working Connector Configuration

Configure via Kibana → Stack Management → Connectors → Create connector → OpenAI.

| Key | Value |
|---|---|
| Name | Sheepsoc |
| Connector type | OpenAI (`.gen-ai`) |
| API Provider | Other |
| API URL | `http://192.168.50.100:11434/v1/chat/completions` |
| Default Model | `qwen3` (or any model name from `ollama list`) |
| API Key | Leave blank or use any dummy value — Ollama does not require authentication |

### Listing Available Models

```bash
pmabry@sheepsoc:~$ curl -s http://localhost:11434/v1/models | python3 -m json.tool
# Returns a list of model ids, e.g.: qwen3:latest, gemma3:12b, nomic-embed-text:latest, ...
# Use the bare name without the :tag suffix in the Default Model field (e.g. "qwen3")
```

!!! note
    Embedding models such as `nomic-embed-text` will appear in the model list but cannot be used for chat completions. Use a generation model (e.g. `qwen3`, `gemma3:12b`, `mistral:7b`).

## Matrix Bot

The Matrix bot (`@sheepsoc-bot:matrix.pmabry.com`) is a separate systemd service that connects Element chat rooms to OpenWebUI's RAG and Ollama LLM inference. It uses full E2EE via the `matrix-nio[e2e]` library.

| Key | Value |
|---|---|
| Systemd unit | `matrix-bot.service` — enabled at boot, runs as `pmabry` |
| Bot MXID | `@sheepsoc-bot:matrix.pmabry.com` |
| Homeserver | `matrix.pmabry.com` (Synapse) |
| OpenWebUI API | `http://localhost:8080/api/chat/completions` |
| Default model | `gemma3:12b` |
| Conda env | `matrixbot` |

For full configuration details, room mapping procedures, file locations, and troubleshooting, see the dedicated [Matrix Bot](matrix-bot.md) page.
