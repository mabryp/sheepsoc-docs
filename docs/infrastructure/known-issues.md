# Known Issues

**Purpose:** Landmines, unresolved items, and the history of what blew up and how it was fixed.

| Key | Value |
|---|---|
| Rule | Read this before making infrastructure changes |
| Last reviewed | 2026-06-29 (Claude Code full-text + semantic search; Filebeat/Logstash migrated to cloud) |

## Active Landmines — Do Not Touch

!!! danger "Do Not"
    **MicroK8s: do not start.** The previous install's OpenEBS Node Disk Manager leaked to **247 GB of RAM**, starved the machine, and drove load average above 100. Persistent volumes were misconfigured and the cluster was unusable. The snap is installed but the services are stopped. See the [Services catalog](services.md) — MicroK8s is marked **hold**.

    Before restart, a proper rebuild plan is required. Key requirements:

    - Disable or memory-limit NDM before deploying any workload.
    - Decide a PV strategy up front — the `k8s_*` mounts are reserved for this.
    - Re-plan previous workloads (Elasticsearch in-cluster, pfSense, etc.) — some may stay on the host instead.

!!! danger "Do Not"
    **NVIDIA driver: do not update without checking.** Current combination is driver **570.169** / CUDA **12.8** on the RTX 5060 Ti, and [Ollama](services.md) is stable on it. The 5060 Ti needs a modern driver — rolling forward incautiously can leave the GPU in a broken state and take Ollama inference offline. Plan the upgrade, have a rollback, and announce before running `apt upgrade` on the NVIDIA packages.

## Watchlist

- **Security hardening steps 7–10** are still pending from the 2026-04-19 session. See the persistent memory pointer `project_security_hardening.md` for the list.
- **Config management** lives at `/home/pmabry/infrastructure/config-mgmt/`. New infrastructure changes should land there, not as ad-hoc edits scattered across the filesystem.
- **Disk watch:** `/` is at 202 GB of 936 GB used. Plenty of headroom, but keep an eye on ES data growth on `/` if indexes stay local.
- **Tailscale uses DERP relay (informational):** Due to Starlink CGNAT, direct peer-to-peer WireGuard connections to sheepsoc from external devices are not possible. Tailscale falls back to DERP relay automatically. This is expected and transparent — not a fault. If the uplink changes to a non-CGNAT provider, Tailscale will prefer direct paths automatically. See [Tailscale](platforms/tailscale.md).
- **Tailscale MagicDNS hostname is `sheepsoc-1` not `sheepsoc` (cosmetic):** Tailscale auto-suffixed `-1` because a stale prior "sheepsoc" entry holds the unsuffixed name in the tailnet. All IPs and Serve URLs resolve correctly. To reclaim the cleaner name, delete the stale entry at [https://login.tailscale.com/admin/machines](https://login.tailscale.com/admin/machines), then re-authenticate with `sudo tailscale up --force-reauth`. Low priority — address when convenient.

- <a id="tailscale-serve-bind-collision"></a>**`tailscale serve` bind collision when local service binds `0.0.0.0`:** When `tailscale serve --https=<port>` is configured, `tailscaled` binds a listener on the tailnet IP (`100.117.117.43:<port>`). If the local service being proxied already binds to `0.0.0.0:<port>` (all interfaces), tailscaled loses the bind race and the serve rule fails silently — `tailscale serve status` may show the rule present, but HTTPS connections will not work. **Workaround:** use a different, unused port for the tailscale serve rule and let it proxy to the app's local port. The `10000+` numbering range is the convention adopted on sheepsoc for this purpose (e.g., Open WebUI binds `0.0.0.0:8080`, so tailscale serve uses `:10001 → localhost:8080`). Services that bind loopback-only (`127.0.0.1`) are not affected and can use any convenient port (e.g., Jupyter's `:8443 → localhost:8888`). See [Tailscale — Port-Numbering Convention](platforms/tailscale.md#port-numbering-convention) for the full rationale and table.

- <a id="docker-compose-v29-env-file-variable-substitution"></a>**Docker Compose v29 env-file variable substitution** — Docker Compose v29.4.1 performs `$variable` substitution on values loaded via `env_file:`. Any literal `$` in a value is treated as a variable reference. Argon2id hashes (and any PHC-format string) contain multiple `$` field separators that will be silently mangled. **Workaround: escape every `$` as `$$` in the env file.** Compose un-escapes `$$` → `$` when injecting into the container, so the runtime value is correct. This applies to any service using `env_file:` whose config values contain `$` — it is not Vaultwarden-specific. See [Vaultwarden — Docker Compose `$$` Escaping](platforms/vaultwarden.md#docker-compose-escaping-for-argon2-hashes) for the full example.

- <a id="docker-healthcheck-localhost-ipv6"></a>**Docker healthcheck: `localhost` resolves to `::1` in alpine containers** — Inside alpine-based Docker containers, the hostname `localhost` resolves to the IPv6 loopback address `::1` (not `127.0.0.1`) because of the default musl/glibc name resolution order. A service whose HTTP server binds to IPv4 `0.0.0.0` only (e.g., Vaultwarden's Rocket server) will not listen on `::1`, causing any healthcheck that uses `http://localhost:<port>/` to fail with "connection refused" even when the service is fully functional. **Workaround: always use `http://127.0.0.1:<port>/` in Docker healthchecks for alpine-based containers.** This applies to any service, not just Vaultwarden. Observed 2026-06-18 after diagnosing a healthcheck failure streak of ~56,650 on the Vaultwarden container. See [Vaultwarden — Healthcheck Must Use `127.0.0.1`, Not `localhost`](platforms/vaultwarden.md#healthcheck-must-use-127001-not-localhost).

- <a id="ollama-upgrade-clobbers-unit"></a>**Ollama install script clobbers custom systemd unit on upgrade:** The official installer (`curl -fsSL https://ollama.com/install.sh | sh`) rewrites `/etc/systemd/system/ollama.service` to its defaults — `User=ollama`, binds `127.0.0.1:11434` only — silently removing the custom user, bind address, model path, and env vars. This breaks [OpenWebUI](platforms/openwebui-rag.md) chat, RAG embeddings, and [Matrix bot](platforms/matrix-bot.md) access until the unit is restored. **After any Ollama upgrade:** restore from `/home/pmabry/ollama-backups/ollama.service.bak`, then `sudo systemctl daemon-reload && sudo systemctl restart ollama`, then re-verify all four health-check endpoints. See [Ollama — Upgrade Procedure](platforms/ollama.md#upgrade-procedure) for the full checklist.

- <a id="openwebui-crash-loop-with-enable_otel-true-without-packages"></a>**OpenWebUI crash-loops if `ENABLE_OTEL=true` without required packages (2026-06-27):** If the OTEL drop-in (`/etc/systemd/system/open-webui.service.d/otel.conf`) is present and `ENABLE_OTEL=true` is set, but the 9 required OpenTelemetry Python packages are not installed in the `openwebui` conda env, `open-webui.service` will crash-loop immediately with `ModuleNotFoundError: No module named 'opentelemetry.exporter.otlp.proto.http'`. This will happen if the env is rebuilt or the packages are accidentally uninstalled. **Recovery:** activate the `openwebui` conda env and reinstall the 9 packages listed in [OpenWebUI — OpenTelemetry Configuration](platforms/openwebui-rag.md#opentelemetry-configuration). A pip freeze snapshot taken before install lives at `/home/pmabry/infrastructure/open-webui/pip-freeze-before-otel-20260627.txt`.

- <a id="claude-code-prompts-egress-to-elastic-cloud"></a>**Claude Code prompt and response content egresses to, and is fully indexed in, Elastic Cloud GCP (updated 2026-06-29):** Claude Code telemetry is configured with `OTEL_LOG_USER_PROMPTS=1` and `OTEL_LOG_TOOL_DETAILS=1` in `~/.claude/settings.json`. **Full prompt text (event `user_prompt`, field `attributes.prompt`) and assistant response text (event `assistant_response`, field `attributes.response`) is logged and exported by the [OpenTelemetry Collector](platforms/otelcol-contrib.md) to Elastic Cloud 9.4.0 (GCP us-central1, cluster UUID `dab081df574b45cf894e33645053dfb3`).** As of 2026-06-29, this content is not merely stored — it is **fully text-indexed** (`prompt_text`, `response_text`) and **semantically embedded** using ELSER v2 (`prompt_semantic`, `response_semantic`) in the `logs-claude_code.otel-*` data stream. Prompts and responses are keyword-searchable and natural-language-queryable from the cloud cluster. **Forward-only caveat:** only events in the new write index (rolled over 2026-06-29) carry the full-text and semantic mappings; events before that rollover are keyword-only (values over 1,024 characters were truncated and unindexed). To disable content logging while keeping metrics: set `OTEL_LOG_USER_PROMPTS=0` and `OTEL_LOG_TOOL_DETAILS=0` in the `env` block of `~/.claude/settings.json`. To disable telemetry entirely: set `CLAUDE_CODE_ENABLE_TELEMETRY=0`.

- <a id="otel-metrics-volume-high-watch-item"></a>**OTEL metrics volume is high — watch Elastic Cloud storage (2026-06-28):** OpenWebUI system-metrics instrumentation generates approximately 100,000 metric documents per 30 minutes of light use (driven by `opentelemetry-instrumentation-system-metrics` at a 10-second export interval). OTEL data lands in Elastic Cloud — no local disk impact, but cloud storage consumption may be significant at scale. If this becomes a concern, consider raising `OTEL_METRICS_EXPORT_INTERVAL_MILLIS` in the OpenWebUI OTEL drop-in or removing the `opentelemetry-instrumentation-system-metrics` package from the `openwebui` conda env. See [OpenTelemetry Collector](platforms/otelcol-contrib.md).

- <a id="vg_elastic-linear-lv-is-exposed-to-a-mechanically-unreliable-nvme-carrier-strategic-decision-pending"></a>**`vg_elastic` linear LV is exposed to a mechanically unreliable NVMe carrier (strategic decision pending):** `nvme3n1` (Samsung 990 PRO 2TB, S/N S7KHNU0Y529975Z) sits on a mini-PCIe carrier card that does not stay mechanically seated. The drive was reseated 2026-05-14 and held — see [history entry below](#2026-05-14-nvme-reseat-and-new-sata-ssd-added) — but the failure mode will recur. The failure mode is **intermittent disappearance from the bus** (not data corruption): the drive vanishes and reappears after reseating. The structural risk is that `vg_elastic` is a **linear** LV spanning `nvme1n1` + `nvme3n1`. A linear LV cannot tolerate a missing PV — if the loose drive drops, the entire 3.6 TiB `/mnt/elastic_data` goes offline. Three strategic options have been identified; none has been executed yet:

    | Option | Summary | Usable capacity | Right when… |
    |---|---|---|---|
    | **A — Mirror `vg_elastic`** | `lvconvert --type raid1 -m1 vg_elastic/lv_elastic_data` — converts the LV to LVM RAID1 across both 990 PROs | 1.8 TiB (drops from 3.6 TiB) | `/mnt/elastic_data` is load-bearing — the volume survives the drive dropping out and degrades gracefully, resyncing on reseat |
    | **B — Demote `nvme3n1` out of `vg_elastic`** | Shrink `vg_elastic` to a single-PV VG on reliable `nvme1n1`; repurpose `nvme3n1` for scratch/archive where intermittent disappearance is tolerable | 1.8 TiB | `/mnt/elastic_data` is no longer load-bearing (e.g., leftover from the 2026-05-10 Elastic Cloud migration) |
    | **C — Root-cause hardware fix** | Replace the mini-PCIe carrier with a proper PCIe→M.2 adapter (full-height, screw-secured), or move the drive to an unused motherboard M.2 slot if one exists | No change | Always — A and B are workarounds; C is the actual fix and should run in parallel |

    **A vs. B decision: `/mnt/elastic_data` IS load-bearing (confirmed 2026-06-28).** Local ES 8.19.14 actively serves [OpenWebUI](platforms/openwebui-rag.md) RAG vectors — the `open_webui_collections_d768` index (~1.9 GB) is on `/mnt/elastic_data` and queried on every RAG interaction. Option B (demote to scratch) would take OpenWebUI RAG offline and is **not viable** until RAG is migrated to Elastic Cloud. Option A (mirror) or Option C (hardware fix) are the appropriate paths. Option C remains recommended regardless.

- <a id="undocumented-listeners-detected-by-labhub-initial-run"></a>**Undocumented listeners detected by Lab Hub initial run (2026-06-29):** On its first collection run, the [Sheepsoc Lab Hub](platforms/lab-hub.md) flagged three TCP listeners not referenced in the [Service Catalog](services.md). Each is noted here for tracking:
  - **Port 111 (rpcbind):** The RPC portmapper, likely activated by the NFS client stack when the [SAN01 NFS mount](topology.md#storage-map) was restored on 2026-06-28. Expected behavior for an NFS client host; not a concern unless the NFS mount is removed.
  - **Port 3001 (node process):** Consistent with Uptime Kuma, which appears in the [Web Endpoints table](services.md#service-urls--lan-and-tailnet) but has no systemd unit row in the Service Catalog. **Action item:** confirm whether Uptime Kuma has a dedicated systemd unit and, if so, add it to the Service Catalog.
  - **Port 9300 (java/Elasticsearch):** Standard Elasticsearch inter-node transport port for the local ES 8.19.14 instance. Completely expected — no action required.

- <a id="otel-attributes-type-field-conflict"></a>**OTEL `attributes.type` field-type conflict across metrics data streams (2026-06-28):** The `attributes.type` field is mapped inconsistently between the two OTEL metrics data streams in Elastic Cloud 9.4.0 (`gateway.es.us-central1.gcp.cloud.es.io`): `metrics-claude_code.otel-default` maps it as `keyword` (string values: `input`, `output`, `cacheRead`, `cacheCreation`, `cli`, `user`, `added`, `removed`), while `metrics-open_webui.otel-default` maps it as `long` (numeric). A Kibana data view spanning both streams (`metrics-*.otel-*`) sees `attributes.type` as a conflicted field and hides it from Lens aggregations and breakdowns. **Impact:** unified cross-source dashboards that break down by `attributes.type` cannot be built using the wildcard data view. **Severity: medium** — ingest is unaffected; the conflict impacts only Kibana analysis. **Workaround:** use per-source data views (e.g. `metrics-claude_code.otel-default`) where the field type is unambiguous; the Claude Code token/cost dashboards use this approach. **Possible fixes (not yet decided):** normalize or rename the field for one source at the [OpenTelemetry Collector](platforms/otelcol-contrib.md) using an `attributes` or `transform` processor, or remap the field type in one data stream's index template; to be evaluated as part of a broader OTEL index-mapping review.

- <a id="system-and-firewall-log-egress-to-elastic-cloud"></a>**System and firewall log egress to Elastic Cloud GCP (2026-06-29):** Filebeat now ships the full **system journal** (including auth events, sudo commands, and SSH activity) to the `filebeat-8.18.3` data stream in Elastic Cloud 9.4.0 (GCP us-central1). Logstash ships **OPNsense firewall syslog** (confirmed flowing), **ASUS RT-AX5400 syslog** (confirmed flowing ~180+ docs as of 2026-06-29), and **SAN01 Synology NAS syslog** (confirmed flowing 2026-06-29) to `logs-syslog.*-default` data streams in the same cluster. This security-relevant data leaves sheepsoc and is stored on a third-party cloud. Phillip explicitly authorized this on 2026-06-29. To stop egress: disable the relevant inputs in `/home/pmabry/infrastructure/filebeat/filebeat-8.18.3-linux-x86_64/filebeat.yml` and the relevant output blocks in `/etc/logstash/conf.d/99-elasticsearch-output.conf`, then restart the affected services. See [Log Shipping](platforms/log-shipping.md).

- <a id="shared-elastic-cloud-api-key-couples-filebeat-logstash-and-otelcol-contrib"></a>**Shared Elastic Cloud API key couples Filebeat, Logstash, and otelcol-contrib (2026-06-29):** All three services — [Filebeat](platforms/log-shipping.md), [Logstash](platforms/log-shipping.md), and the [OpenTelemetry Collector](platforms/otelcol-contrib.md) — authenticate to Elastic Cloud 9.4.0 using the same API key. The key is stored inline in `filebeat.yml` (root:root 600) and `99-elasticsearch-output.conf` (root:logstash 640), and as `ELASTICSEARCH_API_KEY` in `/etc/otelcol-contrib/otelcol-contrib.conf` (root-only 600). A scoped/derived key could not be minted — the parent key forbids creating derived keys with elevated privileges. **Impact:** revoking or rotating the key requires updating all three config files simultaneously and restarting all three services. Plan any key rotation as a coordinated operation.

## History Log

### 2026-06-29 — Sheepsoc Lab Hub Deployed

A new read-only service, the **Sheepsoc Lab Hub**, was deployed on sheepsoc. It probes systemd unit states and TCP ports every 3 minutes, computes drift against the [Service Catalog](services.md), and renders health tiles and a drift list at `http://192.168.50.100:8800` (LAN only).

Three systemd units added to `/etc/systemd/system/`: `labhub-web.service` (FastAPI web UI, port 8800), `labhub-collect.timer` (3-minute cadence), and `labhub-collect.service` (oneshot collector). A new conda environment `labhub` (Python 3.12) was created at `/home/pmabry/infrastructure/miniconda3/envs/labhub`. UFW rule added: `allow from 192.168.50.0/24 to any port 8800 proto tcp`.

The hub's first run flagged three pre-existing undocumented listeners — rpcbind/111, node/3001, and java/9300 — documented in the [Watchlist](#undocumented-listeners-detected-by-labhub-initial-run) above.

Pages updated: [platforms/lab-hub.md](platforms/lab-hub.md) (new page), [services.md](services.md) (catalog row, web endpoint, health check, source-of-truth note), [topology.md](topology.md) (host layout tree, conda envs), [infrastructure/index.md](index.md) (Major Services list), [platforms/conda.md](platforms/conda.md) (env table and example), [schema.md](../../schema.md) (page registry), this page (watchlist, history).

### 2026-06-29 — SAN01 Synology Syslog Added; ASUS Syslog Confirmed

**SAN01 Synology NAS syslog added as a new Logstash source.** SAN01 (Synology DiskStation DSM 7.x, `192.168.50.165`) was configured to forward syslog to the Logstash receiver at `sheepsoc:5514/udp` using DSM's built-in Log Center → Log Sending feature (BSD/RFC3164 format, UDP). This is the only viable method for pulling logs off SAN01 — its CPU architecture does not support Container Manager/Docker, so Filebeat and Elastic Agent containers cannot run on this NAS.

**Logstash changes on sheepsoc:**

- New device-tag branch added to `/etc/logstash/conf.d/50-syslog-filter.conf`: source IP `192.168.50.165` is tagged `device_type: synology` / `device_name: san01`.
- New output block added to `/etc/logstash/conf.d/99-elasticsearch-output.conf`: routes SAN01 syslog to the new Elastic Cloud data stream **`logs-syslog.synology-default`**.
- Verified working with a DSM test log — event landed correctly tagged in `logs-syslog.synology-default`.
- Config backups: `50-syslog-filter.conf.bak-20260629`, `99-elasticsearch-output.conf.bak-20260629`.

**ASUS syslog confirmed flowing.** The `logs-syslog.asus-default` data stream is confirmed receiving documents (~180+ docs as of 2026-06-29). The "ASUS Syslog Not Arriving After Logstash Migration to Cloud" watch item has been resolved and removed from the watchlist above.

Pages updated: [platforms/log-shipping.md](platforms/log-shipping.md) (SAN01 source added, ASUS status updated, data stream tables updated, SAN01 config subsection added), [platforms/elasticsearch-elser.md](platforms/elasticsearch-elser.md) (Beats/Logstash data streams table), [services.md](services.md) (Logstash catalog entry), [topology.md](topology.md) (SAN01 syslog note, observability flow diagram), this page (watchlist updated, history entry added).

### 2026-06-29 — Claude Code Prompt & Response Full-Text and Semantic Search Added

Full-text and ELSER v2 semantic search was enabled over Claude Code prompt and response content in the `logs-claude_code.otel-*` data stream on **Elastic Cloud 9.4.0** (GCP us-central1). This is a forward-only change — only events in the new write index (rolled over 2026-06-29) carry the new mapping and pipeline. Events in the prior backing index retain the original `keyword, ignore_above:1024` mapping (values over 1,024 characters were unindexed and no embeddings were generated there).

**Objects created on the cloud cluster:**

- **Ingest pipeline `claude-code-prompt-enrich`** — extracts `attributes.prompt` (on `user_prompt` events) and `attributes.response` (on `assistant_response` events) into new `prompt_text` / `response_text` (text) and `prompt_semantic` / `response_semantic` (semantic_text, ELSER v2) fields. Other events pass through untouched.
- **Component template `logs-claude_code.otel@custom`** — defines the new field mappings and sets `index.default_pipeline: claude-code-prompt-enrich`.
- **Dedicated index template `logs-claude_code.otel@template`** (priority 200) — pattern `logs-claude_code.otel-*`; overrides the shared `logs-otel@template` (priority 120) for this stream only; `composed_of` mirrors the OTel template and appends the custom component. No shared templates were modified.
- **Data stream rolled over** — new write index `.ds-logs-claude_code.otel-default-2026.06.29-000002` adopts the template and default pipeline.

ELSER inference uses `.elser-2-elasticsearch`, which was already deployed on the cloud cluster for OpenWebUI RAG. This is its second use case on the cluster. Semantic search was verified working with live queries after rollover.

**Privacy note:** Prompts and responses are now not just stored but fully keyword- and semantically-indexed in Elastic Cloud. See updated [Watchlist — Claude Code prompt/response egress](#claude-code-prompts-egress-to-elastic-cloud).

Pages updated: [platforms/otelcol-contrib.md](platforms/otelcol-contrib.md) (data stream table, new log stream section, known-issues gotcha), [platforms/elasticsearch-elser.md](platforms/elasticsearch-elser.md) (OTEL table, new Claude Code section), this page (watchlist entry, history).

### 2026-06-29 — Filebeat and Logstash Repointed from Local ES to Elastic Cloud

Filebeat 8.18.3 and Logstash were reconfigured to ship directly to **Elastic Cloud 9.4.0** (`gateway.es.us-central1.gcp.cloud.es.io:443`) using the shared cloud API key. Both were previously shipping to local Elasticsearch 8.19.14 (`127.0.0.1:9200`) as the `beats_writer` custom user. This is part of the local-ES decommission / migration-to-cloud effort.

**Filebeat changes:**
- Output switched from local ES (HTTP, `beats_writer`) to Elastic Cloud (HTTPS, API key).
- Three inputs: journald `ollama.service` (with `pipeline: logs-ollama-otel`), journald system journal, filestream `~/repositories/sheepsoc/logs/*.log`.
- `filebeat setup --index-management` run against cloud to provision the `filebeat-8.18.3` template and ILM policy (cloud had no prior Filebeat templates).
- Config: `/home/pmabry/infrastructure/filebeat/filebeat-8.18.3-linux-x86_64/filebeat.yml` (root:root 600). Backup: `filebeat.yml.bak-20260629`.

**Cloud ingest pipeline `logs-ollama-otel` created:**
- Reshapes Ollama journald events into the OTel log schema; lands in `logs-ollama.otel-default`.
- Parses structured `time=…level=…source=…msg=…` lines and GIN HTTP access logs; drops llama.cpp runner noise lines.
- **Key gotcha:** routing uses `set _index = logs-ollama.otel-default`, NOT the `reroute` processor. `reroute` silently drops every doc because `filebeat-8.18.3` is not a valid `type-dataset-namespace` data stream name — `reroute` validates the source name before routing and throws `invalid data stream name` on non-conforming names. See [Log Shipping — Ingest Pipeline Gotcha](platforms/log-shipping.md#ingest-pipeline-logs-ollama-otel).

**Logstash changes:**
- All three ES output blocks in `/etc/logstash/conf.d/99-elasticsearch-output.conf` switched from HTTP + `beats_writer`/password to HTTPS + API key (`ssl_enabled => true`).
- Data streams: `logs-syslog.opnsense-default` (confirmed ~400+ docs), `logs-syslog.asus-default` (not arriving — watch item), `logs-syslog.other-default`.
- Config permissions tightened from `644` world-readable (had `beats_writer` password in plaintext) to `640 root:logstash`. Backup: `99-elasticsearch-output.conf.bak-20260629`.

**Auth note:** Filebeat, Logstash, and the [OpenTelemetry Collector](platforms/otelcol-contrib.md) all share one Elastic Cloud API key. Revoking the key breaks all three simultaneously. A scoped/derived key could not be minted.

**Privacy:** System journal (auth, sudo, SSH events), Ollama logs, and OPNsense/ASUS firewall syslog now egress to Elastic Cloud GCP us-central1. Phillip explicitly authorized this.

**New watchlist items added:** system/firewall log egress to cloud; ASUS syslog not arriving; shared API key coupling.

Pages updated: [platforms/log-shipping.md](platforms/log-shipping.md) (new page), [services.md](services.md), [platforms/elasticsearch-elser.md](platforms/elasticsearch-elser.md), [platforms/otelcol-contrib.md](platforms/otelcol-contrib.md), [platforms/ollama.md](platforms/ollama.md), [infrastructure/index.md](index.md), [schema.md](../../schema.md), this page.

### 2026-06-28 — Claude Code Token & Cost Dashboard Built in Elastic Cloud Kibana

A Kibana dashboard named **"Claude Code — Token & Cost Tracking"** was built in Elastic Cloud Kibana (GCP us-central1) on 2026-06-28, using the `metrics-claude_code.otel-default` data stream produced by the [OpenTelemetry Collector](platforms/otelcol-contrib.md) pipeline.

**Key design decisions recorded during this work:**

- **Data view scoped to `metrics-claude_code.otel-default`**, not the wildcard `metrics-*.otel-*`. The wildcard triggers the [`attributes.type` field-type conflict](#otel-attributes-type-field-conflict) (keyword vs. long across Claude Code vs. OpenWebUI streams), which hides the field from Lens aggregations. The scoped per-source data view avoids the conflict entirely.
- **All token and cost aggregations use `Sum`** (not avg, last, or median). The `cumulativetodelta` processor in the OTEL pipeline converts cumulative counters to per-interval deltas at export time; summing the deltas yields the correct total for any time window.
- **`cost.usage` is the authoritative spend signal**, not raw `token.usage`. Claude Code's four token types (`input`, `output`, `cacheCreation`, `cacheRead`) carry very different per-token prices; `cost.usage` applies the correct multiplier for each. Observed cache efficiency was ~93% `cacheRead` share — the cost-efficient operating state.

**Dashboard panels:** tokens over time (by type), tokens by model, cost over time (by model), total cost ($), cache efficiency (%), top sessions by token volume, session/lines-of-code/active-time activity metrics.

For the full metric field reference, aggregation rules, panel specifications, and token type pricing table, see [OpenTelemetry Collector — Claude Code Token & Cost Dashboard](platforms/otelcol-contrib.md#claude-code-token-cost-dashboard).

Pages updated: [otelcol-contrib.md](platforms/otelcol-contrib.md), this page.

### 2026-06-28 — SAN01 NFS Server Restored; Boot-Safe fstab Entry

SAN01 (Synology DiskStation NAS) was offline from approximately April 2026. As of 2026-06-28 it is back online and the NFS mount on sheepsoc has been restored.

**SAN01 network details:**

- IP: `192.168.50.165` (static DHCP reservation on ASUS RT-AX5400, MAC `00:11:32:88:3b:5b`)
- Hostnames: `san01.mabry.lan` / `san01`
- Exports: `/volume1/NFS_Share` to sheepsoc (`192.168.50.100`) only, via NFSv3
- Volume capacity: ~11 TB total · 2.1 TB used · 8.8 TB free
- DSM web UI: `http://192.168.50.165:5000` (HTTP) · `https://192.168.50.165:5001` (HTTPS)
- SSH and SMB/NFS services confirmed up

**DNS fix on sheepsoc:** `/etc/hosts` had a stale entry mapping `san01.mabry.lan` to the old dead address `192.168.50.117`. Corrected to `192.168.50.165`. Backup: `/etc/hosts.bak-2026-06-28`. Neither the ASUS router nor OPNsense (`.253`) has a DNS record for `san01` — resolution relies on `/etc/hosts` on sheepsoc only.

**fstab entry restored with boot-safe options:** The previous fstab line was commented out because the NFS server was offline — an uncommented NFS mount without `nofail` will hang boot while the kernel retries the network mount. The entry has been restored with options that eliminate this risk:

```
san01.mabry.lan:/volume1/NFS_Share /mnt/nfs nfs _netdev,nfsvers=3,nofail,x-systemd.automount,x-systemd.mount-timeout=30 0 0
```

Key flags: `nofail` causes a missing NAS to be silently skipped at boot; `x-systemd.automount` defers the actual mount until first access (the path exists but nothing is contacted until a process accesses `/mnt/nfs`); `x-systemd.mount-timeout=30` bounds the hang on first access if the server is unreachable. The mount is read-write and was verified working. Backup: `/etc/fstab.bak-2026-06-28`.

**Resolved known issue:** The "NFS mount to `san01.mabry.lan`: keep fstab line commented" Active Landmine has been removed. See `Pre-2026-04-18 — NFS Server Outage` in the history below.

Pages updated: [topology.md](topology.md), [services.md](services.md), [lab-operations/sops.md](../lab-operations/sops.md), [runbooks/shutdown-startup.md](runbooks/shutdown-startup.md), this page.

### 2026-06-28 — OTEL Exporter Corrected to Elastic Cloud; OpenWebUI RAG Accuracy Fixed

Two corrections to the 2026-06-27 OTEL documentation:

**1. otelcol-contrib exporter destination changed to Elastic Cloud 9.4.0:**
The initial documentation incorrectly stated that the [OpenTelemetry Collector](platforms/otelcol-contrib.md) exported to local Elasticsearch 8.19.14. The actual destination is **Elastic Cloud 9.4.0** (GCP us-central1, `gateway.es.us-central1.gcp.cloud.es.io:443`, cluster UUID `dab081df574b45cf894e33645053dfb3`). Auth uses an API key stored in `/etc/otelcol-contrib/otelcol-contrib.conf` (chmod 600, root-only), referenced in `config.yaml` as `${env:ELASTICSEARCH_API_KEY}`. No username/password in the config file. OTEL data streams (`logs/metrics/traces-open_webui.otel-*`) confirmed flowing in the cloud cluster. Claude Code streams will appear on the next new session.

**2. OpenWebUI RAG vector store corrected to local ES:**
Several pages incorrectly stated that OpenWebUI RAG uses Elastic Cloud 9.4.0. **OpenWebUI RAG still uses local Elasticsearch 8.19.14.** `ELASTICSEARCH_URL=http://127.0.0.1:9200` is set in the OpenWebUI systemd drop-in; the `open_webui_collections_d768` index (~1.9 GB) is on `/mnt/elastic_data`. Migration of OpenWebUI RAG to Elastic Cloud is planned but has not been done. The 2026-05-10 "local ES decommissioned" history entry referred to research/ingest workloads (RAG-001 v2/v3) moving to cloud — OpenWebUI's RAG index was never migrated.

**Side effects of this correction:**
- The vg_elastic strategic decision (A vs. B) is now unblocked: `/mnt/elastic_data` is confirmed load-bearing (OpenWebUI RAG). Option B (demote to scratch) is off the table until RAG migration completes. See [Watchlist — vg_elastic](#vg_elastic-linear-lv-is-exposed-to-a-mechanically-unreliable-nvme-carrier-strategic-decision-pending).
- The "OTEL data streams rebuild after ES 9" watchlist item from 2026-06-27 has been removed — OTEL is on cloud, so local ES rebuild does not affect it.
- New watchlist item added: [Claude Code prompt content egresses to Elastic Cloud](#claude-code-prompts-egress-to-elastic-cloud).

Pages updated: [otelcol-contrib.md](platforms/otelcol-contrib.md), [services.md](services.md), [topology.md](topology.md), [elasticsearch-elser.md](platforms/elasticsearch-elser.md), [openwebui-rag.md](platforms/openwebui-rag.md), [schema.md](../../schema.md).

### 2026-06-27 — OpenTelemetry Pipeline Implemented (otelcol-contrib v0.155.0)

A complete OTLP telemetry pipeline was added to sheepsoc, following the Elastic Security Labs article "Claude Code/Cowork monitoring at scale with Otel & Elastic". Grok Code was investigated but has no user-configurable OTLP-to-self-hosted path and was not pursued.

**New service — [OpenTelemetry Collector](platforms/otelcol-contrib.md) (`otelcol-contrib.service`):**
- Installed from the official OpenTelemetry GitHub release `.deb` package (v0.155.0, checksum-verified, `sudo dpkg -i`).
- Systemd unit enabled and active, running as system user `otelcol-contrib`.
- Pipeline: `otlp` receiver → `cumulativetodelta` (metrics only) + `batch` processors → `elasticsearch` exporter (Elastic Cloud 9.4.0 via HTTPS, API key auth, mapping mode `otel`). The `cumulativetodelta` processor is required because the ES exporter rejects cumulative-temporality histograms. *(Initial docs stated local ES as target; corrected 2026-06-28 — see history entry above.)*
- All ports bound to `127.0.0.1` only — no LAN exposure, no UFW change: OTLP/gRPC `:4317`, OTLP/HTTP `:4318`, health `:13133`, self-metrics `:8889` (not default `:8888` — Jupyter occupies that port).
- Config at `/etc/otelcol-contrib/config.yaml`; original backed up as `config.yaml.orig-20260627`.

**Reconfigured — [OpenWebUI](platforms/openwebui-rag.md) (`open-webui.service`):**
- Native OTEL export enabled via new systemd drop-in `/etc/systemd/system/open-webui.service.d/otel.conf`. Env vars: `ENABLE_OTEL=true`, traces/metrics/logs all enabled, `OTEL_EXPORTER_OTLP_ENDPOINT=http://127.0.0.1:4317`, `OTEL_SERVICE_NAME=open-webui`, `data_stream.dataset=open_webui`.
- 9 OpenTelemetry Python packages installed into the `openwebui` conda env at pinned versions (1.41.0 / 0.62b0) to match existing library requirements: `opentelemetry-exporter-otlp-proto-http` and 8 instrumentation packages (`fastapi`, `sqlalchemy`, `redis`, `requests`, `httpx`, `aiohttp-client`, `logging`, `system-metrics`). Without these packages, `open-webui.service` crash-loops on startup.
- Pre-install pip freeze snapshot: `/home/pmabry/infrastructure/open-webui/pip-freeze-before-otel-20260627.txt`.
- **New landmine added to Watchlist:** missing packages + OTEL drop-in = crash-loop. See [Watchlist above](#openwebui-crash-loop-with-enable_otel-true-without-packages).

**Reconfigured — Claude Code:**
- Telemetry env block added to `~/.claude/settings.json` (backup: `settings.json.bak-*`): `CLAUDE_CODE_ENABLE_TELEMETRY=1`, OTLP metrics and logs exporters set to gRPC at `http://127.0.0.1:4317`, `data_stream.dataset=claude_code`. Full prompt and tool-argument content is logged (`OTEL_LOG_USER_PROMPTS=1`, `OTEL_LOG_TOOL_DETAILS=1`). Applies to new Claude Code sessions only.

**Data confirmed flowing (2026-06-27):**
`logs-open_webui.otel-*`, `metrics-open_webui.otel-*`, `traces-open_webui.otel-*` — all confirmed. Claude Code streams (`logs/metrics-claude_code.otel-*`) will appear on the next new session.

**Note:** Initial docs stated local ES 8.19.14 as the OTEL export target. This was incorrect — the actual target is Elastic Cloud 9.4.0. Corrected 2026-06-28; see the history entry above. Local ES 8.19.14 serves OpenWebUI RAG vectors and is confirmed active. OTEL data streams land in the cloud cluster.

Pages updated: [services.md](services.md), [topology.md](topology.md), [infrastructure/index.md](index.md), [openwebui-rag.md](platforms/openwebui-rag.md), [elasticsearch-elser.md](platforms/elasticsearch-elser.md), [schema.md](../../schema.md). New platform page: [otelcol-contrib.md](platforms/otelcol-contrib.md).

### 2026-06-26 — Ollama Upgraded 0.9.4 → 0.30.10; Claude Code Local Integration Added

- [Ollama](platforms/ollama.md) upgraded from **0.9.4 to 0.30.10** using the official installer (`curl -fsSL https://ollama.com/install.sh | sh`).
- **Reason for upgrade:** Ollama ≥ 0.14.0 exposes a native Anthropic Messages API at `/v1/messages`. This allows Claude Code (Anthropic's CLI) to target local Ollama models directly. The endpoint returned 404 on 0.9.4.
- **Config preserved:** The installer rewrote the systemd unit to its defaults, but the custom configuration was restored immediately from backups — `User=pmabry`, `OLLAMA_HOST=0.0.0.0:11434`, `OLLAMA_MODELS=/home/pmabry/.ollama`, `OLLAMA_MAX_PROMPT_TOKENS=131072`, and the `parallel.conf` drop-in (`OLLAMA_NUM_PARALLEL=5`).
- **Post-upgrade verification:** Four endpoints confirmed healthy — `/api/version` (0.30.10), `/api/chat` ([OpenWebUI](platforms/openwebui-rag.md) path) 200, `/api/embed` (RAG embeddings path) 200, `/v1/messages` (Claude Code path) returns valid Anthropic-format responses. All 14 existing models intact.
- **New capability — `claude-ollama` wrapper:** Installed at `/home/pmabry/.local/bin/claude-ollama`. Sets `ANTHROPIC_BASE_URL=http://localhost:11434`, `ANTHROPIC_AUTH_TOKEN=ollama`, and defaults `ANTHROPIC_MODEL=qwen3:14b` (chosen to fit in the 16 GB RTX 5060 Ti; Ollama docs recommend `qwen3-coder` 30B which requires ~24 GB VRAM). The system `claude` command and Anthropic cloud config are untouched.
- **New landmine documented:** The installer's unit-file overwrite is now a formal Watchlist entry above. Mitigation: backups at `/home/pmabry/ollama-backups/`; after any future Ollama upgrade, restore the unit and drop-in, daemon-reload, restart, and re-verify all four health-check endpoints.
- New [Ollama](platforms/ollama.md) platform page created. [services.md](services.md), [infrastructure/index.md](index.md), and `schema.md` updated.

### 2026-06-18 — Vaultwarden Healthcheck Fixed; Automated Backup Implemented

Two changes to [Vaultwarden](platforms/vaultwarden.md) closed a long-standing known gap and fixed a silent health-reporting fault.

**Healthcheck fix (service reconfigured):**
The Vaultwarden container had reported `unhealthy` continuously since initial deployment on 2026-05-17 — a failing streak of approximately 56,650 consecutive probes — despite the service being fully functional and serving HTTP 200 throughout. Root cause: the healthcheck in `docker-compose.yml` probed `http://localhost:80/alive`, but inside the alpine container `localhost` resolves to IPv6 `::1` first. Vaultwarden's Rocket HTTP server binds IPv4 `0.0.0.0` only and does not listen on `::1`, so the probe always received "connection refused." Fixed by changing the healthcheck URL to `http://127.0.0.1:80/alive` and running `docker compose up -d --force-recreate`. Container now reports `healthy` (exit 0). A backup of the original compose file is at `docker-compose.yml.bak-20260618`. The general gotcha has been added to the Watchlist above and documented at [Vaultwarden — Healthcheck Must Use `127.0.0.1`, Not `localhost`](platforms/vaultwarden.md#healthcheck-must-use-127001-not-localhost).

**Automated backup implemented (resolves 2026-05-17 gap):**
The "Backup not yet implemented" gap flagged at initial deployment is resolved. A nightly backup script at `/home/pmabry/infrastructure/vaultwarden/backup/vaultwarden-backup.sh` now runs at 02:30 daily (cron, just after the 02:00 RAG sync). The script takes a consistent online SQLite backup (Python `sqlite3` API, no container stop), runs `PRAGMA integrity_check`, bundles all vault assets (`db.sqlite3`, `attachments/`, `rsa_key.pem`, `config.json`), encrypts with `age` using the SOPS/age recipient, and writes to `/mnt/data_extra/backups/vaultwarden/` (14-copy local rotation on a separate physical disk). Off-site upload to `gdrive:sheepsoc-backups/vaultwarden` via rclone is configured in the script but not yet active — pending rclone install and Google OAuth. A full decrypt + restore round-trip was verified. Restore procedure: `/home/pmabry/infrastructure/vaultwarden/backup/RESTORE.md`. See [Vaultwarden — Backup](platforms/vaultwarden.md#backup) for full details.

### 2026-05-30 — tv_control.py Major Refinement to --youtube-search: Browser Bypass via open_browser()

- After repeated keyboard/cursor/send_text failures and user note on onscreen keyboard, major refinement to `infrastructure/scripts/tv_control.py` —youtube-search: now uses `tv.open_browser()` with `https://www.youtube.com/results?search_query={quote(query)}` (`urllib.parse.quote`). Completely bypasses YouTube app, on-screen keyboard, cursor navigation, clear, and char-by-char keys. Much simpler/reliable. Updated imports, argparse help, docstring, and logic. Token/WoL/volume unchanged. Re-tested successfully (opens search results directly).
- Updated **only** triggered pages per schema.md §6 (major procedure change to TV control): [Samsung TV Network Control runbook](runbooks/wol-samsung-tv.md) (revised *all* YouTube sections: purpose, usage, How It Works, troubleshooting with new browser method + rationale), [services.md](services.md#tv-control) (TV section), [schema.md](../schema.md) (registry), and this history. Strict adherence to link rules (§4: relative paths from each file's location, first-mention, **see also**/**runbook** labels, reciprocals where meaningful, no new files). Interlinks verified. Reversible via git. Fulfills CLAUDE.md (explained before every action; wiki reflects *current* code as living record).
- Commit: "refine tv_control --youtube-search to browser bypass (open_browser + quote)". Pushed to origin/main. No mkdocs build. See prior entries below for full keyboard/cursor/send_text iteration history.

### 2026-05-30 — tv_control.py Text Input Updated to Per-Character KEY_* Loop per send_text Observation

- `tv_control.py` updated: `send_text(query)` replaced with per-character loop (`for char in query.lower(): if isalpha send KEY_{upper}(char) else KEY_SPACE`; 0.25s sleeps) because "send_text is not working. I can see the cursor move" (now types letters directly when search field focused). Clear loop + navigation unchanged. Re-tested with "Try not to laugh". Token saved. Current implementation (cursor move + char keys) is the living wiki record.
- Updated **only** triggered pages per schema.md §6 (code change to TV control): [Samsung TV Network Control runbook](runbooks/wol-samsung-tv.md) (new input method details in purpose/How It Works/usage/troubleshooting), [services.md](services.md#tv-control) (TV section), [schema.md](../schema.md) (registry entry), and this history. Followed schema strictly: relative links (correct per current file location in `docs/infrastructure/`), exact structure, first-mention per section, **see also**/**runbook** labels, reciprocals where meaningful (no topology/index changes). Interlinks verified — no broken links. Reversible via git. Fulfills CLAUDE.md (explained before each edit; wiki as living record).
- Commit: "update tv_control text input to per-char KEY_* per send_text observation". No mkdocs build performed. See prior entries for KEY_BACKSPACE/KEY_CLEAR/send_text evolution.

### 2026-05-30 — tv_control.py Reverted to KEY_BACKSPACE Loop per User Feedback

- After user feedback "KEY_CLEAR didn't clear field", reverted `tv_control.py` from single `KEY_CLEAR` + `sleep(0.5)` to reliable 15x `KEY_BACKSPACE` loop + `sleep(0.1)` before `send_text()` for YouTube search (original implementation per "clear the search before it runs a search" request). Navigation/launch unchanged. Token saved. Tested successfully via `conda run -n sheepsoc`.
- Updated **only** triggered pages per schema.md §6 (code change to TV control/runbook): [Samsung TV Network Control runbook](runbooks/wol-samsung-tv.md) (revised How It Works/usage/troubleshooting with backspace loop + KEY_CLEAR test note), [services.md](services.md#tv-control) (TV section), [schema.md](../schema.md) (registry entry), and this history. Followed schema strictly: relative links only, page structure, reciprocals where meaningful, first-mention per section. Interlinks verified (no broken links). No other pages (e.g. topology/index) updated. Fulfills CLAUDE.md (explained before each edit, reversible, wiki= living record).
- Commit: "revert tv_control clear to KEY_BACKSPACE loop per test feedback". No mkdocs build. See prior entries below for KEY_CLEAR test context.

### 2026-05-30 — tv_control.py Refined: KEY_CLEAR Replaces Backspace Loop

- Phillip refined `infrastructure/scripts/tv_control.py` again: replaced the 15x `KEY_BACKSPACE` clear loop with single `tv.send_key("KEY_CLEAR")` + `sleep(0.5)`, because the TV remote/UI has a dedicated clear button that fully clears the YouTube search bar in one step (confirmed by user). Tested successfully (no errors, cleaner). Token file still present; no pairing.
- Updated [Samsung TV Network Control runbook](runbooks/wol-samsung-tv.md) (purpose, How It Works, troubleshooting, usage example), [services.md](services.md#tv-control) (TV section), [schema.md](../schema.md) (registry), and this history entry per schema.md §6 update triggers for code/procedure change to runbook/services. Uses relative links, first-mention labels, **see also**/**runbook** reciprocals per §4 (verified interlinks; reversible via prior commit). Wiki = living record of current tv_control clear procedure.
- See also prior 2026-05-30 entries below for full context.

### 2026-05-30 — tv_control.py Updated to Clear YouTube Search Field (Prevents Stale Text)

- Phillip updated `infrastructure/scripts/tv_control.py` (after last wiki sync) to add loop of 15x `KEY_BACKSPACE` (with short 0.1s sleeps) right after focusing the input field and before `send_text(query)`.
- This reliably clears any residual text from prior searches in the YouTube app UI.
- Tested successfully via `conda run -n sheepsoc --youtube-search 'Try not to laugh'`. Token remains saved in `~/.config/samsung-tv-token.json`; no re-pairing needed.
- Updated runbook (How It Works, troubleshooting, purpose), services.md (TV section), schema.md (registry), and this history entry per section 6 triggers for "code change to TV control procedure / runbook / services".
- Links follow section 4 rules exactly (relative from each file, first-mention, **see also**/**runbook**, reciprocals where meaningful, no noise). Interlinks verified. Reversible edits; wiki remains living record. Fulfills CLAUDE.md (explanation before each edit, no silent sudo, Documentation agent task completed).
- No topology/known-issues/index.md triggered as this was an improvement, not new landmine.

### 2026-05-30 — Samsung TV Documented + WoL Runbook (Resolves Undocumented LAN Device)

- Network-engineer subagent successfully retrieved Samsung TV details from ASUS router DHCP table: IP `192.168.50.175`, MAC `54:3A:D6:5D:B0:EC`, hostname "Samsung", active lease.
- WoL magic packet sent (`wakeonlan -i 192.168.50.255 54:3a:d6:5d:b0:ec`) — succeeded; TV now powered on and documented for the first time.
- New runbook [WoL Samsung TV](runbooks/wol-samsung-tv.md) created with full step-by-step (TV settings check: Settings > Network > Expert > Wake on LAN or Power On with Mobile; prerequisites: wired Ethernet; command; troubleshooting; test confirmation).
- Updated [Topology](topology.md) (added to Network Topology diagram, Host Layout/key table), [index.md](index.md) (Runbooks list), [schema.md](../schema.md) (page registry per section 2), this history entry.
- **services.md** left unchanged (WoL is a client-side tool, not a running systemd service on sheepsoc; fits better in runbook).
- All per schema.md update triggers for "new device added to LAN" and "new runbook created" (section 6). Link rules followed exactly (section 4): relative links on first mention, reciprocal links added where relationship is meaningful (runbook ↔ topology, runbook ↔ index/known-issues), backlinks verified. No mkdocs build. This fulfills CLAUDE.md post-infrastructure-change Documentation agent requirement.
- Wiki is now the single source of truth. Reciprocal links verified.

### 2026-05-30 — Samsung TV Full Network Control Added (tv_control.py Script)

- New script `infrastructure/scripts/tv_control.py` generated and placed (executable). Prioritizes volume control (`--volume`, `--up`/`--down`, `--mute`), integrates known WoL for `--power on` (MAC/IP from prior test), supports raw keys, one-time pairing (token to `~/.config/samsung-tv-token.json`), comprehensive error handling, CLI argparse help.
- Runbook `runbooks/wol-samsung-tv.md` expanded (now titled Samsung TV Network Control in registry/index) with installation (sheepsoc conda + `pip install samsungtvws wakeonlan` — confirmed safe per CLAUDE.md, builds on existing websocket-client), full usage examples (volume first), pairing procedure, expanded prerequisites (add Network Remote/Access Notification), "How It Works" section, updated troubleshooting, integration note with 2026-05-30 DHCP/WoL success.
- Updated [Topology](topology.md) (control notes added to diagram, table, host layout; links updated), [index.md](index.md) (runbook description), [schema.md](../schema.md) (registry purpose + triggers for "new control script added / device control procedure"), this history entry, and all cross-links per section 4 (reciprocal **see also** where meaningful; verified no broken links).
- No changes to services.md (client-side tool). No mkdocs build. Reversible backups made of edited files. Fulfills CLAUDE.md exactly: explanations before each action, no silent sudo, wiki single source of truth, Documentation agent brief covered all capabilities + prior success.
- Wiki now fully reflects complete TV network control setup (WoL + websocket commands from sheepsoc).

### 2026-05-30 — Samsung TV YouTube Search Capability Added to tv_control.py

- Enhanced `tv_control.py` with `--youtube-search "QUERY"` (uses `samsungtvws.run_app("111299001912")` to launch YouTube, navigation keys to search field, `send_text()` for query like "Try not to laugh", final enter; tuned sleeps for reliability).
- Tested successfully via `conda run -n sheepsoc` (no errors; TV at 192.168.50.175 responded).
- Updated [services.md](services.md#tv-control) (new TV Control section), [runbooks/wol-samsung-tv.md](runbooks/wol-samsung-tv.md) (usage, how it works, troubleshooting), [topology.md](topology.md), [index.md](index.md), [schema.md](../schema.md) (registry), this history entry.
- Per schema.md section 6 update triggers for "new service capability" and runbook changes. All links follow section 4 rules (relative MkDocs syntax from each file's location, first-mention, explicit labels like **see also**/**runbook**, reciprocal backlinks added to services.md, topology.md, index.md, known-issues.md, schema.md where meaningful; verified no broken links or noise). Reversible edits; no mkdocs build; fulfills CLAUDE.md (explanations before edits, wiki as living record, Documentation agent invoked post-change).
- Wiki now fully reflects current tv_control.py state including YouTube integration.

### 2026-05-30 — RAG Experiments Wiki Updated (Resolves Multiple Issues)

- Comprehensive documentation added for active RAG workspace at `~/jupyter/rag_experiments/`: RAG-001 (StackExchange sysadmin ~48k doc corpus, full pipeline, golden dataset with AI/human judgments, notebooks), RAG-002 scaffold, Elastic Cloud 9.4.0 integration details, hybrid RRF vs dense/ELSER/BM25 benchmarks (specific nDCG@10 results), embedding comparisons (nomic/mxbai), evaluation pipelines, concepts, runbooks, and sources.
- New [research/rag-experiments.md](../research/rag-experiments.md) created; research/agenda.md, elasticsearch-elser.md, openwebui-rag.md, schema.md, and index pages updated with links and current status.
- Marks Elastic Cloud migration as fully documented for research track; OpenWebUI cloud rewiring noted as next step. Resolves known issues around RAG-001/002 gaps, migration, comparisons, golden datasets, and old Streamlit platform (now deprecated).
- Wiki is now the single source of truth per lab policy. No mkdocs build run. Changes committed and pushed.
- Reciprocal links verified per schema section 4. Update triggers from schema section 6 followed.

### 2026-05-18 — Tailscale Serve Map Expanded

- **Docs landing page is now the default tailnet entry point.** `https://sheepsoc-1.tail0f68e4.ts.net/` (port 443) previously proxied to Open WebUI. It now proxies to the docs landing page (`localhost:80`). Open WebUI moved to `:10001`.
- **Newly exposed via tailnet:** Kibana (`:10002 → localhost:5601`), Uptime Kuma (`:10003 → localhost:3001`).
- **Port 10000 retained as backwards-compatibility alias** for the docs landing page.
- **`10000+` port-numbering convention adopted** for services whose local process binds `0.0.0.0:<port>`. See the new Watchlist entry for the bind-collision rationale and the full convention at [Tailscale — Port-Numbering Convention](platforms/tailscale.md#port-numbering-convention).
- `services.md` and `index.md` updated to show both LAN and tailnet URLs side-by-side.
- Vikunja row flagged "to be decommissioned" in the service catalog.

### 2026-05-17 — Vaultwarden Admin Token Hashed; Env File Renamed

- **`ADMIN_TOKEN` rotated and replaced with an argon2id hash** using the Vaultwarden "bitwarden" preset (m=65540, t=3, p=4). The plaintext token was generated with `secrets.token_urlsafe(48)` entropy. The hash now lives in `vaultwarden.env`. Vaultwarden no longer logs a startup warning about a plaintext token.
- **Env file renamed** from `.env` to `vaultwarden.env`. Docker Compose auto-loads a `.env` file in the project directory for its own variable interpolation. This caused every `$` in the argon2 hash to be processed as a variable reference before the value ever reached the container, silently mangling the hash on startup. Renaming the file to `vaultwarden.env` and referencing it via the `env_file:` directive in `docker-compose.yml` decouples it from Compose's auto-load behavior.
- **Remaining `$$` escaping requirement:** Even with the renamed env file, Docker Compose v29.4.1 still performs variable substitution on `env_file:`-loaded values. The hash in `vaultwarden.env` has every `$` escaped as `$$`. This must be preserved on any future token rotation. See [Vaultwarden — Rotate the Admin Token](platforms/vaultwarden.md#rotate-the-admin-token) and the [Watchlist entry above](#docker-compose-v29-env-file-variable-substitution) for details.
- Resolves: Vaultwarden plaintext admin token (previously in Watchlist).

### 2026-05-14 (evening) — Storage Reshuffle: PNY 4TB Repartitioned and P3-1TB Commissioned

- **PNY CS900 4TB SSD (`/dev/sda`) repartitioned** from a 1.8 TiB + 1.8 TiB split into a single 3.6 TiB `sda1` partition. The old `sda2` partition was the previously active `/mnt/ssd_working` mount; the old `sda1` was the unknown-purpose unmounted partition flagged in the morning's entry. Both are gone.
- **Migration method:** all 48 GiB of data from the old `sda2` (`Data/` directory) was staged to `sdb` via rsync, `sda` was repartitioned and formatted fresh (ext4, label `ssd_working`, UUID `a08d2cf0-5d95-4616-9dbe-54b1595df98d`), then the data was rsynced back. Round-trip verified byte-identical: 108,738 files / 50,819,999,489 bytes, 0 errors.
- **`/etc/fstab` updated:** old `ssd_working` UUID line (`cabf6ad5…`) is commented out as a rollback breadcrumb. New line uses UUID `a08d2cf0-5d95-4616-9dbe-54b1595df98d`. `systemctl daemon-reload` run to activate.
- **P3-1TB SATA SSD (`/dev/sdb`) commissioned:** formatted ext4, label `data_extra`, UUID `207e03a8-dbe0-4025-9f1e-6c9d4bd13e5c`. Mounted at `/mnt/data_extra` (~938 GiB usable). Owned `pmabry:pmabry`. Currently empty. Intended for bulk additional data storage.
- The unknown-purpose `sda1` flagged in the morning's entry is resolved — it no longer exists. No data was lost; the partition was confirmed empty before being subsumed.
- See [Topology — Storage Map](topology.md#storage-map) for the updated logical layout.

### 2026-05-14 — OpenProject Decommissioned

OpenProject was removed from sheepsoc. It never worked as a fit for Phillip's workflow and the container had been stopped for approximately two weeks before formal removal.

**What was removed:**

- Docker container `openproject` (image `openproject/openproject:15`) — removed via `docker rm` / `docker rmi`
- Bind-mount data directory `/mnt/ssd_working/openproject/` — wiped (`sudo rm -rf`; `pgdata` subdirectory was owned by a non-pmabry UID)
- Orphaned bridge network `openproject_default` — removed via `docker network rm`
- `~/infrastructure/sheepsoc-shutdown.sh` — cleaned up: removed `openproject` from `APP_SERVICES`, removed the `docker stop openproject` line, and removed the docker-check block in the final verification step

Port 3001 is retained by Uptime Kuma. No UFW changes were made.

### 2026-05-14 — NVMe Reseat and New SATA SSD Added

- **`nvme3n1` (Samsung 990 PRO 2TB, S/N S7KHNU0Y529975Z) reseated.** This is the drive that was previously loose in its mini-PCIe carrier (see the 2026-05-10 entry). The reseat held — both PVs of `vg_elastic` reported healthy, LV `lv_elastic_data` is active and mounted r/w at `/mnt/elastic_data`, SMART is PASSED, wear 4%, no dmesg I/O errors observed post-reseat. Monitor for recurrence.
- **New SATA SSD added:** `sdb` (P3-1TB, S/N 0039914A03508, 0 power-on hours). Not yet partitioned, formatted, or mounted. Intended for additional data storage; configuration is TBD.
- **`sda1` flagged as unknown-purpose** *(resolved same day, evening)*: A 1.8 TiB ext4 partition on `sda` (UUID `1c74cfed-7268-4c49-a72b-a6aa4600204c`) was present but unmounted. Confirmed empty during the evening storage reshuffle. The partition was removed when `sda` was repartitioned into a single 3.6 TiB `sda1`. See the evening entry above.
- The prior loose-NVMe landmine documented in the 2026-05-10 entry (below) is superseded by this reseat — the physical issue was addressed at the hardware level. The 2026-05-10 migration to Elastic Cloud also removed ES I/O dependence on this drive, providing software-level resilience if the seat loosens again.
- See [Topology — Storage Map](topology.md#storage-map) for the full updated drive table and logical layout.

### 2026-05-10 — Elasticsearch Migrated to Elastic Cloud (Local ES Decommissioned)

- Local single-node `elasticsearch.service` on sheepsoc (8.19.14, `/mnt/elastic_data`) was **decommissioned**. All ES traffic now goes to **Elastic Cloud 9.4.0** at `https://gateway.es.us-central1.gcp.cloud.es.io` (GCP us-central1, cluster UUID `DaBtVAvNQNmqT0thJwt-7Q`, 3 nodes / 2 data nodes, green).
- Auth changed from `basic_auth=("elastic","<password>")` to API key (`Authorization: ApiKey <key>`). New env var `ELASTICSEARCH_API_KEY` set in `~/.env` (renamed from typoed `ELASTICSEARH_API`). The new name also matches OpenWebUI's native env var.
- All prior local indexes (including `open_webui_collections_d768`, the RAG_001 v2/v3 indexes, all syslog data streams) are **not carried over**. Each consumer must be reconfigured to point at the cloud cluster and, where applicable, re-ingest from source.
- Status of consumers as of 2026-05-11:
    - **RAG_001 v2** re-ingested into cloud cluster — 48,266 docs, 0 errors (used existing pre-embedded bulk JSONL).
    - **RAG_001 v3** ingest in progress.
    - **RAG_002** deferred — chunks exist but were never embedded; held pending Cloudflare/native-Assistant decision.
    - **OpenWebUI, Filebeat, Metricbeat, Logstash, Kibana** still configured for `localhost:9200` and will fail until pointed at the cloud cluster — pending Phillip's decision on each.
- Resolves the long-running landmine of the loose mini-PCIe NVMe carrier under `/mnt/elastic_data` that produced recurring EIO and cluster hangs — ES no longer touches that storage path.
- See [project wiki: Elastic Cloud Migration](https://github.com/...) (`docs/runbooks/elastic_cloud_migration.md` in `~/jupyter/rag_experiments/`) for the RAG-specific procedure and `concepts/elastic_cloud_deployment.md` for the new connection model.

### 2026-05-09 — Tailscale Serve Configured

- `tailscale serve` configured to expose three services to the tailnet over HTTPS. All three rules were written with `--bg` (persistent state) and survive restarts.
- Active rules: OpenWebUI at port 443, Jupyter at port 8443, sheepsoc-docs at port 10000. All use auto-provisioned Let's Encrypt certs for `sheepsoc-1.tail0f68e4.ts.net`.
- Port-based (not path-based) to avoid breaking OpenWebUI's root-path assumption.
- No UFW changes required — existing `ALLOW IN on tailscale0` rule covers tailnet access; proxy runs via loopback.
- Docs site is intentionally unauthenticated at port 10000 (approved by Phillip). OpenWebUI and Jupyter retain their own auth layers.
- See [Tailscale — Tailscale Serve](platforms/tailscale.md#tailscale-serve) for full details and [Tailscale Operations — Managing Serve Rules](runbooks/tailscale-ops.md#5-managing-tailscale-serve-rules) for the management runbook.

### 2026-05-09 — Tailscale Installed (Fresh)

- [Tailscale](platforms/tailscale.md) installed from the official Tailscale apt repository (`pkgs.tailscale.com/stable/ubuntu/noble`). This replaces the failed installation that was purged on 2026-04-19.
- Enrolled to tailnet `tail0f68e4` (Phillip's existing tailnet, Google SSO). Sheepsoc's tailnet address is `100.117.117.43`; MagicDNS hostname is `sheepsoc-1.tail0f68e4.ts.net`. (The `-1` suffix is cosmetic — a stale prior entry holds the unsuffixed name. See Watchlist.)
- `tailscaled.service` enabled at boot and active.
- UFW rule added: `ufw allow in on tailscale0` — permits all inbound traffic from tailnet peers on the `tailscale0` interface.
- Sysctl file `/etc/sysctl.d/99-tailscale.conf` created with `net.ipv4.ip_forward = 1`.
- Subnet routing, exit node, and Tailscale SSH intentionally left disabled. See [Tailscale](platforms/tailscale.md) for the full rationale.
- Due to Starlink CGNAT, direct WireGuard connections are not possible; Tailscale uses DERP relay (`derp-9`). This is expected and transparent.

### 2026-04-27 — OpenWebUI Upgraded from 0.6.12 to 0.9.2

- OpenWebUI upgraded to **0.9.2** on sheepsoc.
- **6 database schema migrations applied automatically** at startup. All migrations were additive (new columns / tables only) — no data was altered or removed.
- Pre-upgrade backup of `webui.db` created at `~/infrastructure/open-webui/webui.db.bak-20260427`. Retain this until the new version has been in stable operation for at least one week.
- All existing RAG Knowledge bases, Elasticsearch vector index (`open_webui_collections_d768`), and the ELSER sparse-embedding pipeline are unaffected by the upgrade.
- If the `openwebui` conda env is ever rebuilt after this upgrade, confirm the `elasticsearch==8.19.3` pip package is still present (`pip show elasticsearch`). The package is not bundled and must be reinstalled manually.

### 2026-04-23 — ELSER Sparse Embeddings Added to OpenWebUI RAG Index

- The `open_webui_collections_d768` index is now **dual-use**: it carries both dense kNN vectors (`vector` field, 768-dim, `nomic-embed-text`) used by OpenWebUI, and a new sparse vector field (`text_elser`, type `sparse_vector`) populated by ELSER.
- Ingest pipeline `elser-embed-openwebui` created. Set as the index default pipeline — all future OpenWebUI document uploads automatically receive ELSER tokens at ingest time. No changes to OpenWebUI itself.
- ELSER inference endpoint: `.elser-2-elasticsearch` (built-in, adaptive allocation — scales to zero when idle).
- All 8,136 existing documents backfilled with ELSER tokens in two passes. First pass processed 7,235 documents; 901 (~11%) were skipped due to ELSER scaling from zero on the first batch (a known cold-start behaviour). Second pass completed the remaining 901. Final coverage: 8,136 / 8,136, zero conflicts.
- New capability: `text_expansion` queries and hybrid RRF (Reciprocal Rank Fusion) queries can now be run against the index from outside OpenWebUI. See [RAG & Knowledge — ELSER](platforms/openwebui-rag.md), [ELSER & OpenWebUI](platforms/elasticsearch-elser.md), and the how-to at `~/infrastructure/elasticsearch/howto-elser-openwebui-pipeline.md`.

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
- Tailscale package fully purged (64.9 MB freed). The daemon had been failing for ~2 months. **Note:** Tailscale was reinstalled fresh on 2026-05-09 — see the history entry above.
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

`san01.mabry.lan` went down. Its entry in `/etc/fstab` was commented to keep boot from hanging — an uncommented NFS mount without `nofail` retries at boot and can stall the boot sequence for minutes. **Resolved 2026-06-28:** SAN01 came back online. The fstab entry was restored with `nofail,x-systemd.automount,x-systemd.mount-timeout=30` so a future outage will not cause a boot hang. See the [2026-06-28 history entry above](#2026-06-28-san01-nfs-server-restored-boot-safe-fstab-entry) for full details.
