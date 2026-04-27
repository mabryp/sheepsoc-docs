# Known Issues

**Purpose:** Landmines, unresolved items, and the history of what blew up and how it was fixed.

| Key | Value |
|---|---|
| Rule | Read this before making infrastructure changes |
| Last reviewed | 2026-04-23 |

## Active Landmines — Do Not Touch

!!! danger "Do Not"
    **MicroK8s: do not start.** The previous install's OpenEBS Node Disk Manager leaked to **247 GB of RAM**, starved the machine, and drove load average above 100. Persistent volumes were misconfigured and the cluster was unusable. The snap is installed but the services are stopped.

    Before restart, a proper rebuild plan is required. Key requirements:

    - Disable or memory-limit NDM before deploying any workload.
    - Decide a PV strategy up front — the `k8s_*` mounts are reserved for this.
    - Re-plan previous workloads (Elasticsearch in-cluster, pfSense, etc.) — some may stay on the host instead.

!!! danger "Do Not"
    **NFS mount to `san01.mabry.lan`: keep fstab line commented.** The NFS server is offline. Uncommenting the mount will hang boot while the kernel retries. Only restore after the NFS server is back *and* reachable from sheepsoc.

!!! danger "Do Not"
    **NVIDIA driver: do not update without checking.** Current combination is driver **570.169** / CUDA **12.8** on the RTX 5060 Ti, and Ollama is stable on it. The 5060 Ti needs a modern driver — rolling forward incautiously can leave the GPU in a broken state and take Ollama inference offline. Plan the upgrade, have a rollback, and announce before running `apt upgrade` on the NVIDIA packages.

!!! warning "Handle with Care"
    **Reserved disks:** `/mnt/k8s_ssd_1`, `/mnt/k8s_nvme_1`, `/mnt/k8s_nvme_2`, `/mnt/k8s_nvme_3` are earmarked for the k8s rebuild. Do not put permanent data there — they will be reformatted when the cluster is rebuilt.

## Watchlist

- **Security hardening steps 7–10** are still pending from the 2026-04-19 session. See the persistent memory pointer `project_security_hardening.md` for the list.
- **Config management** lives at `/home/pmabry/infrastructure/config-mgmt/`. New infrastructure changes should land there, not as ad-hoc edits scattered across the filesystem.
- **Disk watch:** `/` is at 202 GB of 936 GB used. Plenty of headroom, but keep an eye on ES data growth on `/` if indexes stay local.

## History Log

### 2026-04-23 — ELSER Sparse Embeddings Added to OpenWebUI RAG Index

- The `open_webui_collections_d768` index is now **dual-use**: it carries both dense kNN vectors (`vector` field, 768-dim, `nomic-embed-text`) used by OpenWebUI, and a new sparse vector field (`text_elser`, type `sparse_vector`) populated by ELSER.
- Ingest pipeline `elser-embed-openwebui` created. Set as the index default pipeline — all future OpenWebUI document uploads automatically receive ELSER tokens at ingest time. No changes to OpenWebUI itself.
- ELSER inference endpoint: `.elser-2-elasticsearch` (built-in, adaptive allocation — scales to zero when idle).
- All 8,136 existing documents backfilled with ELSER tokens in two passes. First pass processed 7,235 documents; 901 (~11%) were skipped due to ELSER scaling from zero on the first batch (a known cold-start behaviour). Second pass completed the remaining 901. Final coverage: 8,136 / 8,136, zero conflicts.
- New capability: `text_expansion` queries and hybrid RRF (Reciprocal Rank Fusion) queries can now be run against the index from outside OpenWebUI. See [RAG & Knowledge — ELSER](platforms/openwebui-rag.md) and the how-to at `~/infrastructure/elasticsearch/howto-elser-openwebui-pipeline.md`.

### 2026-04-21 — Elasticsearch xpack.security Enabled

- `xpack.security.enabled: true` set in `/etc/elasticsearch/elasticsearch.yml`. TLS left disabled (HTTP plaintext, LAN-only). Enrollment disabled.
- Built-in account passwords set: `elastic` (superuser), `kibana_system`, `beats_system`. All three verified via API before service restarts.
- Kibana updated: `/etc/kibana/kibana.yml` now carries `elasticsearch.username: kibana_system` and the corresponding password.
- Filebeat and Metricbeat updated: both `filebeat.yml` and `metricbeat.yml` carry `username: beats_system` and password under `output.elasticsearch`.
- OpenWebUI drop-in updated: `/etc/systemd/system/open-webui.service.d/elasticsearch.conf` now includes `ELASTICSEARCH_USERNAME=elastic` and `ELASTICSEARCH_PASSWORD`.
- All five services (Kibana, Filebeat, Metricbeat, OpenWebUI) restarted in sequence and verified active. ES cluster health: **green**, 45/45 shards. Kibana login page reachable at `http://localhost:5601`.
- Any `curl` commands to ES now require `-u elastic:<password>`. See [Services](services.md) for updated command examples.

### 2026-04-21 — Bulk Ingest Script and Gardening Knowledge Base

- Script `~/repositories/sheepsoc/ingest_to_openwebui.py` written to bulk-ingest pre-extracted `.txt` chunks into OpenWebUI Knowledge Bases. Writes to both Elasticsearch and SQLite simultaneously. Idempotent — uses `uuid5`-based file IDs so re-runs safely skip already-indexed files.
- **Gardening** knowledge base populated: 134 files, 1,352 chunks from Texas A&M AgriLife extension texts. Queryable in OpenWebUI chat via `#Gardening`.
- Key architectural insight documented: the ES `collection` field must be the bare knowledge base UUID from `webui.db`. Using `file-{uuid}` (the namespace for standalone uploads) breaks KB retrieval silently.
- Dataset inventory mapped: `/mnt/ssd_working/Data/Ingest_Ready/` contains 63 pre-extracted text collections ready to ingest. Mother Earth News archive (19,688 files / ~38 GB) identified as future work requiring a dedicated script and separate ES index.

### 2026-04-20 — OpenWebUI RAG Configured

- OpenWebUI 0.6.12 switched from Chroma to **Elasticsearch** as its vector store.
- `nomic-embed-text:latest` pulled to Ollama — the embedding model for OpenWebUI RAG (768 dimensions).
- ES index `open_webui_collections_d768` pre-created with HNSW/cosine mapping.
- `elasticsearch==8.19.3` pip-installed into the `openwebui` conda env (not bundled — must reinstall if env is rebuilt).
- Systemd drop-in created: `/etc/systemd/system/open-webui.service.d/elasticsearch.conf`.
- ES cluster health resolved from yellow to green by fixing replica counts on filebeat/metricbeat data streams.

### 2026-04-20 — ASUS Syslog Pipeline Verified and Fixed

- Set `log_remote=1` in ASUS NVRAM.
- Applied the `syslogd -R` flag to ship to sheepsoc:5514/udp.
- Confirmed `logs-syslog.asus-default` data stream is receiving events in Elasticsearch.

### 2026-04-19 — Security Hardening Session

- UFW firewall enabled. Ports 22, 80, 443, 8080, 8888, 9200, 5601, 11434, and 3000 restricted to LAN (`192.168.50.0/24`) or localhost only.
- Tailscale package fully purged (64.9 MB freed). The daemon had been failing for ~2 months.
- Config management system bootstrapped at `~/infrastructure/config-mgmt/`.
- Steps 7–10 of the hardening plan remain pending.

### 2026-04-18 — Major Recovery Session

The box was in a degraded state across networking, boot, and shell. Fixes:

| Problem | Fix |
|---|---|
| Gateway was set to `192.168.50.253` — a dead host — causing no internet connectivity. | Set gateway to `192.168.50.1` (the ASUS). |
| DNS was broken. | Configured `8.8.8.8` and `1.1.1.1` via netplan. LAN `mabry.lan` lookups still go through OPNsense at `192.168.50.253`. |
| `tailscaled` had been failing for ~2 months, flooding logs. | Tailscale removed from the box. |
| `systemd-networkd-wait-online` was blocking boot. | Service disabled. |
| Login hung for ~5 minutes due to the subprocess-based conda shell hook. | `~/.bashrc` switched to the static loader: `source ~/infrastructure/miniconda3/etc/profile.d/conda.sh`. Do not revert. |
| No Claude Code on the box. | Claude Code `v2.1.114` installed. |

### Pre-2026-04-18 — MicroK8s Incident

OpenEBS NDM memory leak ballooned to 247 GB of RAM. The machine became unresponsive; load averages exceeded 100. All workloads that had been running in the cluster (Elasticsearch, pfSense, others) were migrated off or rebuilt on the host. MicroK8s has been stopped since.

### Pre-2026-04-18 — NFS Server Outage

`san01.mabry.lan` went down. Its entry in `/etc/fstab` was commented to keep boot from hanging. Still offline as of last review.
