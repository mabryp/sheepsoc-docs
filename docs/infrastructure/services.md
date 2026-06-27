# Services

**Purpose:** What runs on sheepsoc, where it lives, and how to check it.

## Service Catalog

Every service running on sheepsoc, its systemd unit name, port, and current operational status. Services marked **hold** are installed but intentionally stopped pending a rebuild or external dependency.

| Service | Unit / cmd | Port | Purpose | Status |
|---|---|---|---|---|
| **sheepsoc-landing** | `sheepsoc-landing.service` | 80 | LAN landing page and these docs (Python http.server) | up |
| **Vikunja** | `vikunja.service` | 3000 | Self-hosted kanban / task manager — **to be decommissioned** (see Monday board) | up |
| **Elasticsearch (local)** | `elasticsearch.service` | 9200 | ES 8.19.14 (local, `/mnt/elastic_data`) — **active as of 2026-06-27 for OTEL telemetry.** Receives data from the [OpenTelemetry Collector](platforms/otelcol-contrib.md) (`logs/metrics/traces-open_webui.otel-*`, `logs/metrics-claude_code.otel-*`). RAG and research workloads run on **Elastic Cloud 9.4.0** (GCP us-central1) — see [Known Issues — 2026-05-10](known-issues.md) for the migration record. **This instance is planned for a clean ES 9 rebuild** — OTEL data streams must be recreated after; see [Known Issues](known-issues.md#otel-data-streams-must-be-recreated-after-es-9-rebuild). | up |
| **Kibana** | `kibana.service` | 5601 | Log & metrics visualization (Filebeat / Metricbeat / syslog) | up |
| **Logstash** | `logstash.service` | 5514/udp | Syslog ingestion from ASUS router + OPNsense | up |
| **Filebeat** | `filebeat.service` | — | Ships local log files to Elasticsearch | up |
| **Metricbeat** | `metricbeat.service` | — | Ships host metrics (CPU, RAM, disk, net) to Elasticsearch | up |
| **OpenTelemetry Collector** | `otelcol-contrib.service` | 4317/4318 (OTLP) · 13133 (health) · 8889 (self-metrics) — all `127.0.0.1` only | OTLP telemetry hub · v0.155.0 · receives OTLP from [OpenWebUI](platforms/openwebui-rag.md) and Claude Code · exports to local Elasticsearch · all ports loopback-only, no LAN exposure — see [OpenTelemetry Collector](platforms/otelcol-contrib.md) | up |
| **Open WebUI** | `open-webui.service` | 8080 | **Primary AI interface.** OpenWebUI 0.9.2 · browser-based chat and RAG frontend · connects to Ollama for LLM inference · RAG via Elasticsearch (`nomic-embed-text`, 768d, HNSW/cosine) · runs in `openwebui` conda env (Python 3.11) — see [OpenWebUI & RAG](platforms/openwebui-rag.md) | up |
| **Jupyter Notebook** | `jupyter.service` | 8888 | Notebook server, notebook dir `~/repositories/sheepsoc` | up |
| **Ollama** | `ollama.service` | 11434 | Local LLM inference (uses RTX 5060 Ti) · **0.30.10** (upgraded 2026-06-26) · exposes Ollama-native API, OpenAI-compatible `/v1`, and Anthropic Messages `/v1/messages` (≥0.14.0) · 14 models at `~/.ollama` · `claude-ollama` wrapper enables Claude Code to use local models — see [Ollama](platforms/ollama.md) | up |
| **SSH** | `ssh.service` | 22 | Remote shell · key auth only | up |
| **Tailscale** | `tailscaled.service` | — (WireGuard overlay) | Mesh VPN for remote access · tailnet `tail0f68e4` · IP `100.117.117.43` · MagicDNS `sheepsoc-1.tail0f68e4.ts.net` · no subnet routing, no exit node — see [Tailscale](platforms/tailscale.md) | up |
| **cron** | `cron.service` | — | Scheduled tasks | up |
| **MicroK8s** | `snap.microk8s.*` | — | Kubernetes — *stopped*, needs rebuild | **hold** |
| **Matrix Bot** | `matrix-bot.service` | — | E2EE Matrix bot (`@sheepsoc-bot:matrix.pmabry.com`) · bridges Element rooms to OpenWebUI RAG + Ollama · runs in `matrixbot` conda env — see [Matrix Bot](platforms/matrix-bot.md) | up |
| **Vaultwarden** | `docker compose` (`/home/pmabry/infrastructure/vaultwarden/`) | 8222 (loopback) | Self-hosted Bitwarden-compatible password and secrets manager · replacing LastPass · bound to `127.0.0.1:8222` only · exposed to tailnet at `https://sheepsoc-1.tail0f68e4.ts.net:8444/` via Tailscale Serve · no LAN UFW rule — see [Vaultwarden](platforms/vaultwarden.md) | up |
| **RomM / EmulatorJS** | `docker compose` (`/mnt/ssd_working/emulatorjs/`) | 3080 | Self-hosted ROM library manager with integrated in-browser retro emulator (EmulatorJS) · two containers: `romm` (rommapp/romm:latest, 4.8.1) + `romm-db` (MariaDB) · LAN UFW rule + tailnet via `tailscale serve` :10004 · no metadata API keys configured yet — see [RomM / EmulatorJS](platforms/romm-emulatorjs.md) | up |

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

# OpenTelemetry Collector health (returns HTTP 200 with "Server available" when healthy)
pmabry@sheepsoc:~$ curl -s http://127.0.0.1:13133

# Vikunja info
pmabry@sheepsoc:~$ curl -s http://localhost:3000/api/v1/info | jq
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

For ES / syslog / beats data, Kibana is usually easier — see [SOPs: Reading Logs in Kibana](../lab-operations/sops.md#5-read-logs-in-kibana).

## Web Endpoints

All web interfaces are restricted to `192.168.50.0/24` via UFW on the LAN interface. Several services are also reachable from any Tailscale-enrolled device via `tailscale serve` HTTPS — these tailnet URLs are not public, only accessible to devices enrolled in Phillip's `tail0f68e4` tailnet. See [Tailscale](platforms/tailscale.md) for remote-access configuration details.

### Service URLs — LAN and Tailnet

| Service | LAN URL | Tailnet URL | Notes |
|---|---|---|---|
| Landing / Docs | `http://192.168.50.100/` | `https://sheepsoc-1.tail0f68e4.ts.net/` | Default tailnet entry point (port 443) — no auth |
| Docs (alias) | `http://192.168.50.100/` | `https://sheepsoc-1.tail0f68e4.ts.net:10000/` | Backwards-compatibility alias; same backend as default |
| Open WebUI | `http://192.168.50.100:8080` | `https://sheepsoc-1.tail0f68e4.ts.net:10001/` | OpenWebUI login required |
| Jupyter | `http://192.168.50.100:8888` | `https://sheepsoc-1.tail0f68e4.ts.net:8443/` | Jupyter argon2 password required |
| Kibana | `http://192.168.50.100:5601` | `https://sheepsoc-1.tail0f68e4.ts.net:10002/` | Log / metric dashboards |
| Uptime Kuma | `http://192.168.50.100:3001` | `https://sheepsoc-1.tail0f68e4.ts.net:10003/` | Live service status |
| Vaultwarden | — (tailnet only by design) | `https://sheepsoc-1.tail0f68e4.ts.net:8444/` | Loopback-only locally; Bitwarden login required · admin at `/admin` — see [Vaultwarden](platforms/vaultwarden.md) |
| Elasticsearch | `http://192.168.50.100:9200` | n/a (API endpoint) | REST API, not a browser UI |
| Ollama | `http://192.168.50.100:11434` | n/a (API endpoint) | LLM inference REST API, not a browser UI |
| Vikunja | `http://192.168.50.100:3000` | — | **To be decommissioned** — see Monday board |
| RomM / EmulatorJS | `http://192.168.50.100:3080` | `https://sheepsoc-1.tail0f68e4.ts.net:10004/` | ROM library + in-browser emulator · tailnet URL is HTTPS (secure context) — enables SharedArrayBuffer / multi-threaded cores in EmulatorJS |
| ASUS router | `http://192.168.50.1` | — | Gateway admin |
| OPNsense | `https://192.168.50.253` | — | Self-signed cert |

Tailnet URLs use auto-provisioned Let's Encrypt certs. For configuration details, the port-numbering convention, and management commands, see [Tailscale — Tailscale Serve](platforms/tailscale.md#tailscale-serve).

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

For full configuration details, room mapping procedures, file locations, and troubleshooting, see the dedicated [Matrix Bot](platforms/matrix-bot.md) page.

## TV Control

**Purpose:** Client-side network control for the Samsung TV (192.168.50.175) via `tv_control.py`. Provides volume (prioritized), power (WoL), keys, and **YouTube search** (now via `tv.open_browser()` with quoted search URL — completely bypasses YouTube app, on-screen keyboard, cursor, clear, and char-by-char keys after repeated failures). Much simpler/reliable. Not a systemd service — on-demand via sheepsoc conda env (updated imports, argparse, docstring, logic). See [Samsung TV Network Control runbook](runbooks/wol-samsung-tv.md) — **runbook** for full procedure (reciprocal; current browser-bypass impl documented there as living record).

### Key Facts
| Key | Value |
|---|---|
| Script | `~/infrastructure/scripts/tv_control.py` (executable, argparse CLI with `--youtube-search`) |
| Conda Env | `sheepsoc` (tested via `conda run -n sheepsoc`; includes samsungtvws, wakeonlan) |
| TV IP/MAC | 192.168.50.175 / `54:3A:D6:5D:B0:EC` (DHCP from ASUS, wired Ethernet required for WoL/control) |
| Dependencies | `samsungtvws`, `wakeonlan` (in sheepsoc env); token at `~/.config/samsung-tv-token.json` (one-time pairing) |
| YouTube Search | `--youtube-search "QUERY"` uses `open_browser("https://www.youtube.com/results?search_query={quote(query)}")` (`urllib.parse.quote`). Bypasses all prior keyboard/cursor/app issues; opens results directly. Token/WoL/volume unchanged. Re-tested successfully. Current state per wiki and [runbook](runbooks/wol-samsung-tv.md). |

### Usage
```bash
pmabry@sheepsoc:~$ conda run -n sheepsoc python infrastructure/scripts/tv_control.py --help
pmabry@sheepsoc:~$ conda run -n sheepsoc python infrastructure/scripts/tv_control.py --youtube-search "Try not to laugh"
# Opens YouTube search results directly in TV browser (bypasses on-screen keyboard/cursor entirely). Re-tested successfully.
# Other commands: --volume 40, --power on (WoL + 8s), --up/--down/--mute, --key KEY_VOLUP
```

### Verification
```bash
pmabry@sheepsoc:~$ ping -c 2 192.168.50.175
pmabry@sheepsoc:~$ conda run -n sheepsoc python infrastructure/scripts/tv_control.py --youtube-search "test"  # observe TV opens browser to results
```

**See Also:** [Samsung TV Network Control runbook](runbooks/wol-samsung-tv.md) (full steps, troubleshooting, How It Works for browser method).

**Runbook:** [Samsung TV Network Control](../runbooks/wol-samsung-tv.md) — **runbook** for install, prerequisites (Network Remote enabled), full troubleshooting, how it works (pairing, token persistence, error handling).

See also: [Topology](topology.md#network-topology), [Known Issues](known-issues.md#history-log).
