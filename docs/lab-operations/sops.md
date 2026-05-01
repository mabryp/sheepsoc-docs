# Standard Operating Procedures

**Purpose:** The repeatable, step-by-step procedures for common sheepsoc operations.

## 1. Run a Query in OpenWebUI

OpenWebUI at `http://192.168.50.100:8080` is the primary interface for all AI queries on sheepsoc. It handles chat, document upload, and Knowledge base RAG in one browser UI.

### Prerequisites — Verify Services Are Up

```bash
pmabry@sheepsoc:~$ systemctl is-active ollama open-webui elasticsearch
active
active
active
```

If any service is down, start it:

```bash
pmabry@sheepsoc:~$ sudo systemctl start ollama.service
pmabry@sheepsoc:~$ sudo systemctl start open-webui.service
pmabry@sheepsoc:~$ sudo systemctl start elasticsearch.service
```

### Running a Query

1. Open `http://192.168.50.100:8080` in a browser.
2. Start a new chat.
3. To query against a Knowledge base, type `#` in the message box and select the collection from the dropdown.
4. Type your question and submit. OpenWebUI retrieves relevant document chunks from Elasticsearch, injects them as context, and streams the LLM answer back.

!!! note "Tip"
    Ollama idles at ~0 MB VRAM when no query is running — leave it running at all times. You almost never need to stop it.

## 2. Check Service Health

### Quick "Is Everything Fine?" Dashboard

```bash
# Anything failed?
pmabry@sheepsoc:~$ systemctl --failed

# The big six — should all be active (running)
pmabry@sheepsoc:~$ for s in ollama open-webui elasticsearch kibana logstash jupyter; do
                    printf "%-16s %s\n" "$s" "$(systemctl is-active $s)"
                  done
```

### Deep Health Pokes

```bash
# Elasticsearch cluster state — auth required (xpack.security enabled 2026-04-21)
pmabry@sheepsoc:~$ curl -s -u elastic:<password> http://localhost:9200/_cluster/health?pretty

# Disk usage — watch /, /mnt/ssd_working, /mnt/nvme_working
pmabry@sheepsoc:~$ df -h | grep -E '^/|Filesystem'

# Memory + load
pmabry@sheepsoc:~$ free -h
pmabry@sheepsoc:~$ uptime

# GPU
pmabry@sheepsoc:~$ nvidia-smi
```

## 3. Switch the LLM Model in OpenWebUI

OpenWebUI uses a per-chat model selector. Switching models does not require restarting any service — just change the selection in the UI.

1. Confirm the target model is present on Ollama: `ollama list`
2. Open OpenWebUI at `http://192.168.50.100:8080`.
3. In any chat, use the model dropdown at the top of the chat window to select the desired model.

```bash
# Verify which models are available before switching
pmabry@sheepsoc:~$ ollama list
NAME                             ID            SIZE      MODIFIED
dolphin3:latest                  ...           4.7 GB    ...
gemma3:12b                       ...           8.9 GB    ...
mistral:7b                       ...           4.1 GB    ...
```

!!! note "Recommendation"
    `gemma3:12b` gives the best quality/speed tradeoff for RAG queries. `dolphin3:latest` is faster for simple questions that do not need deep reasoning over retrieved context.

## 4. Add Documents to Elasticsearch

There are two distinct ingest paths depending on which system you are feeding.

### 4a. OpenWebUI Knowledge Bases (Bulk Ingest Script)

Use this path to populate an OpenWebUI Knowledge base from a directory of pre-extracted `.txt` files in `/mnt/ssd_working/Data/Ingest_Ready/`. The script at `~/repositories/sheepsoc/ingest_to_openwebui.py` writes to both Elasticsearch and SQLite simultaneously and is idempotent — already-indexed files are skipped on re-run.

!!! note "Dedicated Page"
    Full step-by-step SOP, dataset reference table, architecture notes, and troubleshooting guide live on the dedicated page: **[OpenWebUI KB Bulk Ingest](../infrastructure/runbooks/openwebui-kb-bulk-ingest.md)**

### 4b. Legacy CLI RAG Index (Deprecated — Do Not Use)

!!! danger "Deprecated"
    The CLI RAG prototype (`~/repositories/sheepsoc`) and its associated Elasticsearch index (`intfloat_e5_large_v2__512__125`) are no longer maintained. Do not add documents to this index. Use the OpenWebUI Knowledge base path (4a above) for all document ingestion.

The legacy CLI RAG app used `intfloat/e5-large-v2` (1024-dim) embeddings and a hybrid KNN + BM25 pipeline with CrossEncoder reranking. Its ingestion scripts live in `~/repositories/sheepsoc/tests/`. This pipeline is not compatible with the OpenWebUI RAG index (`open_webui_collections_d768`) — the two systems use separate ES indexes with different embedding models.

## 5. Read Logs in Kibana

Open `http://192.168.50.100:5601`. The useful views are:

| View | What You Get |
|---|---|
| Observability → Logs | Tailed view of all log streams in ES |
| Discover | Ad-hoc query across any data stream or index |
| Dashboards | Saved dashboards for host and service overviews |

### Relevant Data Streams

| Stream | Source |
|---|---|
| `logs-syslog.asus-default` | ASUS RT-AX5400 router syslog |
| `logs-syslog.opnsense-default` | OPNsense firewall syslog |
| `filebeat-*` | sheepsoc system and service logs (filebeat) |
| `metricbeat-*` | sheepsoc host metrics (CPU/RAM/disk/net) |

Common KQL filters in Discover:

```
# Only high-severity ASUS events
data_stream.dataset : "syslog.asus" and log.syslog.severity.code <= 4

# Filebeat errors on sheepsoc from the last hour
log.level : "error" and host.name : "sheepsoc"
```

## 6. Connect to ASUS Router and OPNsense

### ASUS RT-AX5400

- Web admin: `http://192.168.50.1`
- SSH: port **1024**, key at `~/.ssh/asus_router`

```bash
pmabry@sheepsoc:~$ ssh -i ~/.ssh/asus_router -p 1024 admin@192.168.50.1
```

### OPNsense

- Web admin: `https://192.168.50.253` — the browser will complain about the self-signed cert; accept it.
- SSH: port 22 (if enabled in the OPNsense UI).

## 7. Conda Environment Management

Short reference — for the "why" of conda see the [Conda Guide](../infrastructure/platforms/conda.md).

```bash
# List every environment you have
pmabry@sheepsoc:~$ conda env list

# Activate one
pmabry@sheepsoc:~$ conda activate sheepsoc

# See what's installed in the active env
(sheepsoc) pmabry@sheepsoc:~$ conda list

# Leave the current env (back to base or plain shell)
(sheepsoc) pmabry@sheepsoc:~$ conda deactivate

# If conda isn't initialized in a new shell (e.g., a systemd unit)
pmabry@sheepsoc:~$ source ~/infrastructure/miniconda3/etc/profile.d/conda.sh
```

## 9. Planned Shutdown and Startup

For a safe, ordered shutdown and post-boot verification sequence, follow the dedicated runbook:

**[Shutdown & Startup Runbook](../infrastructure/runbooks/shutdown-startup.md)**

The runbook covers the correct stop order (OpenWebUI → Elasticsearch → Ollama), emergency shutdown, post-boot service verification, and common startup failure troubleshooting.

## 8. What NOT to Do

!!! danger "Do Not"
    **Do not start MicroK8s.** The previous install's OpenEBS NDM leaked to 247 GB, starved the machine, and drove load average to 100+. The cluster is stopped until a proper rebuild plan exists. See [Known Issues](../infrastructure/known-issues.md).

!!! danger "Do Not"
    **Do not uncomment the NFS entry in `/etc/fstab`.** `san01.mabry.lan` is offline — uncommenting it will hang boot. Only restore after the NFS server is back and reachable.

!!! danger "Do Not"
    **Do not update the NVIDIA driver without checking first.** The RTX 5060 Ti currently runs 570.169 / CUDA 12.8 and Ollama is stable on this combo. Unplanned updates can take GPU inference offline.

!!! warning "Avoid"
    **Do not run silent `sudo`.** Announce what needs root and why. Phillip's rule: nothing happens on this machine without him knowing about it.

!!! warning "Avoid"
    **Do not delete configs when "cleaning up".** Back up or comment out instead. Reversibility beats tidiness.
