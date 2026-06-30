# Log Shipping — Filebeat & Logstash

**Purpose:** Collect and forward local service logs and network device syslog to Elastic Cloud 9.4.0 for centralized observability.

## Overview

Two Elastic data shippers run on sheepsoc: **Filebeat 8.18.3** and **Logstash**. Both were reconfigured on 2026-06-29 to ship directly to **Elastic Cloud 9.4.0** (GCP us-central1, `gateway.es.us-central1.gcp.cloud.es.io:443`) using an API key. Prior to this change both shippers sent data to local Elasticsearch 8.19.14 (`127.0.0.1:9200`) as the `beats_writer` custom user.

This is part of the local-ES decommission / migration-to-cloud effort. Local Elasticsearch 8.19.14 continues to serve [OpenWebUI RAG](openwebui-rag.md) vectors only.

| Shipper | Unit | Config | Destination |
|---|---|---|---|
| Filebeat 8.18.3 | `filebeat.service` | `/home/pmabry/infrastructure/filebeat/filebeat-8.18.3-linux-x86_64/filebeat.yml` | Elastic Cloud 9.4.0 |
| Logstash | `logstash.service` | `/etc/logstash/conf.d/99-elasticsearch-output.conf` | Elastic Cloud 9.4.0 |

## Dependencies

- [Elasticsearch & ELSER](elasticsearch-elser.md) — **stores data in** Elastic Cloud 9.4.0 (`gateway.es.us-central1.gcp.cloud.es.io:443`) — the sole export target for both shippers; connectivity to the cloud cluster is required for any data to land
- [Ollama](ollama.md) — **monitored by** Filebeat; the `ollama.service` journald log is one of the three Filebeat inputs and is reshaped by the `logs-ollama-otel` ingest pipeline

## Configuration

### Auth — API Key

Both Filebeat and Logstash authenticate to Elastic Cloud 9.4.0 using an API key. This is the same API key used by the [OpenTelemetry Collector](otelcol-contrib.md).

!!! warning "Shared API Key — Revocation Coupling"
    The API key stored in these configs is the same key used by `otelcol-contrib`. Revoking this key will simultaneously break Filebeat, Logstash, and the OpenTelemetry Collector. A scoped/derived key could not be minted because the parent key forbids derived keys with elevated privileges.

    The key is stored inline in the config files: `filebeat.yml` (root:root 600) and `99-elasticsearch-output.conf` (root:logstash 640). Use `<api_key>` as a placeholder — the actual value must never appear in documentation or commit history.

Config backups taken 2026-06-29:

- `filebeat.yml.bak-20260629`
- `/etc/logstash/conf.d/50-syslog-filter.conf.bak-20260629`
- `/etc/logstash/conf.d/99-elasticsearch-output.conf.bak-20260629`

---

### Filebeat

**Config:** `/home/pmabry/infrastructure/filebeat/filebeat-8.18.3-linux-x86_64/filebeat.yml`
**Permissions:** root:root 600 (contains API key inline)

Filebeat is configured with three inputs. All three now ship to Elastic Cloud.

#### Output Block

```yaml
output.elasticsearch:
  hosts: ["https://gateway.es.us-central1.gcp.cloud.es.io:443"]
  api_key: "<api_key>"
  ssl.enabled: true
```

#### Input 1 — Ollama Service Journal (`log_type: ollama`)

Ships `ollama.service` journald output. Configured with `pipeline: logs-ollama-otel` so every event is processed by the `logs-ollama-otel` ingest pipeline on Elastic Cloud before storage.

**Destination data stream:** `logs-ollama.otel-default`

#### Input 2 — System Journal (`log_type: systemd`)

Ships the full systemd journal (all units, including auth, sudo, SSH, and kernel events). No ingest pipeline applied.

**Destination data stream:** `filebeat-8.18.3`

!!! warning "Privacy — System Journal Egresses to Elastic Cloud"
    This input ships authentication events, SSH login attempts, sudo commands, and other security-relevant system activity to Elastic Cloud (GCP us-central1). This data leaves sheepsoc and is stored on a third-party cloud. Phillip explicitly authorized this on 2026-06-29. See [Known Issues — System and Firewall Log Egress to Elastic Cloud](../known-issues.md#system-and-firewall-log-egress-to-elastic-cloud).

#### Input 3 — Sheepsoc Application Logs (`log_type: sheepsoc`)

Filestream input watching `/home/pmabry/repositories/sheepsoc/logs/*.log`. Currently idle — no active log files exist in that path as of 2026-06-29.

**Destination data stream:** `filebeat-8.18.3`

#### Index Template Setup

After switching the Filebeat output to cloud, the following command was run once to provision the `filebeat-8.18.3` index template and ILM policy on the cloud cluster (the cloud cluster had no prior Filebeat templates):

```bash
pmabry@sheepsoc:~$ sudo /home/pmabry/infrastructure/filebeat/filebeat-8.18.3-linux-x86_64/filebeat \
    setup --index-management \
    -e \
    -E output.elasticsearch.hosts=["https://gateway.es.us-central1.gcp.cloud.es.io:443"] \
    -E output.elasticsearch.api_key="<api_key>"
```

---

### Ingest Pipeline — `logs-ollama-otel`

This pipeline was created on Elastic Cloud 9.4.0 on 2026-06-29 and extended on 2026-06-30 from 27 to 62 processors. It reshapes raw Ollama journald events (shipped by Filebeat input 1) into the **OTel log document schema**, placing them in the same schema family as Claude Code and OpenWebUI OTEL logs.

**Pipeline source:** `/home/pmabry/infrastructure/elasticsearch/logs-ollama-otel.pipeline.json`
**Pre-extension backup:** `/home/pmabry/infrastructure/elasticsearch/backups/logs-ollama-otel.pipeline.bak.20260630.json`
**Restore command:** `curl -X PUT .../logs-ollama-otel -H 'Content-Type: application/json' -d @logs-ollama-otel.pipeline.bak.20260630.json`

**Why this pipeline exists:** The [OpenTelemetry Collector](otelcol-contrib.md) receives native OTLP telemetry from OpenWebUI and Claude Code. Ollama does not emit OTLP natively — it emits log lines to journald. This pipeline bridges that gap by reshaping the journald events into the OTel schema, making Ollama logs queryable alongside OTEL data in Kibana without format divergence.

**What it parses:** Three Ollama log formats:

1. **Structured key=value lines** — `time=… level=… source=… msg="…"`
2. **GIN HTTP access logs** — `[GIN] … | <status> | <latency> | <ip> | <METHOD> "<path>"`
3. **llama.cpp runner metric events** — six named event types produced by the llama.cpp inference engine (added 2026-06-30); previously dropped with a blanket `drop` processor; now parsed and indexed with typed numeric fields

The prior blanket `drop` for all llama.cpp runner lines has been narrowed to a targeted set of noise patterns. Six named runner event types now survive and are indexed.

#### Runner Metric Event Types (Added 2026-06-30)

Each runner metric document carries `attributes.ollama.event` as a keyword discriminator. The six indexed event types and their key fields:

| Event type | Key fields | Description |
|---|---|---|
| `new_prompt` | `prompt_tokens`, `n_ctx_slot`, `n_keep` | Recorded when a new prompt is loaded into a slot |
| `prompt_eval` | `prompt_eval_ms`, `prompt_eval_tokens`, `prompt_tokens_per_sec` | Prompt prefill (time-to-first-token phase) |
| `generation` | `eval_ms`, `eval_tokens`, `eval_tokens_per_sec`, `ms_per_token` | Token generation — `eval_tokens_per_sec` is the primary throughput metric |
| `total` | `total_ms`, `total_tokens` | End-to-end summary for a completed request |
| `release` | `final_tokens`, `truncated` | Recorded when a slot is released; `truncated` is `boolean` |
| `slot_load` | `n_ctx` | Context window size logged when a model slot is initialized |

All runner metric fields are under the `attributes.ollama.*` namespace. `slot_id` and `task_id` are present on most event types and can be used to correlate the full set of documents for a single inference request.

#### Lines Still Dropped (Noise)

The following llama.cpp runner patterns continue to be dropped before storage — they carry no per-request telemetry value:

- Streaming decode progress ticks (`n_decoded` / `tg t/s` pattern)
- Sampler chain dumps
- `llama_model_loader` kv array dumps
- Cache and prompt-cache bookkeeping lines
- Idle slot notices
- `init_sampler` and `graphs reused` lines

GIN and structured log parsing are unchanged by the 2026-06-30 extension (regression confirmed via live test).

#### Fields Set by the Pipeline

**Shared / structured log and GIN fields:**

| Field | Value / Source |
|---|---|
| `body.text` | Full raw log line |
| `severity_text` | From `level=` field, or derived from HTTP status code for GIN lines |
| `severity_number` | Numeric OTel severity (9=INFO, 13=WARN, 17=ERROR) |
| `resource.attributes.service.name` | `ollama` (static) |
| `resource.attributes.host.name` | Sheepsoc hostname |
| `scope.name` | `ollama` (static) |
| `attributes.ollama.source` | Source file/function from `source=` |
| `attributes.ollama.msg` | Message from `msg=` |
| `attributes.error.message` | Set on error-level lines |
| `attributes.http.request.method` | From GIN access lines |
| `attributes.url.path` | From GIN access lines |
| `attributes.http.response.status_code` | From GIN access lines |
| `attributes.client.address` | Client IP from GIN access lines |
| `attributes.ollama.duration_ns` | Latency in nanoseconds from GIN access lines |

**Runner metric fields (added 2026-06-30 — all under `attributes.ollama.*`):**

| Field | ES type | Present on event types | Description |
|---|---|---|---|
| `attributes.ollama.event` | `keyword` | all runner events | Discriminator: `new_prompt` / `prompt_eval` / `generation` / `total` / `release` / `slot_load` |
| `attributes.ollama.slot_id` | `long` | most | Slot identifier for correlating multi-doc request records |
| `attributes.ollama.task_id` | `long` | most | Task ID (may be −1 for model-init tasks) |
| `attributes.ollama.prompt_tokens` | `long` | `new_prompt` | Input token count |
| `attributes.ollama.n_ctx_slot` | `long` | `new_prompt` | Context window tokens allocated for this slot |
| `attributes.ollama.n_keep` | `long` | `new_prompt` | KV-cache tokens retained on context shift |
| `attributes.ollama.prompt_eval_ms` | `float` | `prompt_eval` | Prompt prefill time in ms |
| `attributes.ollama.prompt_eval_tokens` | `long` | `prompt_eval` | Tokens evaluated during prefill |
| `attributes.ollama.prompt_tokens_per_sec` | `float` | `prompt_eval` | Prefill throughput |
| `attributes.ollama.eval_ms` | `float` | `generation` | Generation time in ms |
| `attributes.ollama.eval_tokens` | `long` | `generation` | Generated (output) token count |
| `attributes.ollama.eval_tokens_per_sec` | `float` | `generation` | **Primary throughput metric** — generation tokens/sec |
| `attributes.ollama.ms_per_token` | `float` | `generation` | Inverse throughput — ms per generated token |
| `attributes.ollama.total_ms` | `float` | `total` | End-to-end request latency in ms |
| `attributes.ollama.total_tokens` | `long` | `total` | Total tokens (prompt + generated) |
| `attributes.ollama.final_tokens` | `long` | `release` | Final token count at slot release |
| `attributes.ollama.truncated` | `boolean` | `release` | Whether context was truncated |
| `attributes.ollama.n_ctx` | `long` | `slot_load` | Context window size at model load |

All 18 runner metric fields are correctly typed and fully aggregatable. They are dynamically mapped by the existing `logs-ollama.otel-attrs@custom` component template — no index template changes were required. Verified via `field_caps` API on 2026-06-30 using mistral:7b (11 docs indexed); `avg(eval_tokens_per_sec)` confirmed aggregatable, returning 84.57 t/s.

#### Example Query — Generation Throughput Over Time

```json
GET logs-ollama.otel-default/_search
{
  "size": 0,
  "query": {"term": {"attributes.ollama.event": "generation"}},
  "aggs": {
    "throughput_over_time": {
      "date_histogram": {"field": "@timestamp", "calendar_interval": "1h"},
      "aggs": {"avg_tps": {"avg": {"field": "attributes.ollama.eval_tokens_per_sec"}}}
    }
  }
}
```

!!! danger "Gotcha — Use `set _index`, Not `reroute`"
    The pipeline routes docs to `logs-ollama.otel-default` using a **`set` processor** that assigns `_index = logs-ollama.otel-default`. Do **not** use the `reroute` processor here.

    `reroute` was attempted first and silently dropped every document. It fails because it validates the *source* data stream name (`filebeat-8.18.3`) as a `type-dataset-namespace` triple before routing — `filebeat-8.18.3` does not match that format, so `reroute` throws `invalid data stream name` and drops the doc. The `set _index` approach bypasses this validation entirely and is the correct method whenever the source data stream has a non-standard name.

<a id="ollama-otel-mapping-fix"></a>
### Attribute Field Type Fix — `logs-ollama.otel-*` (2026-06-29)

After the `logs-ollama-otel` pipeline was in production, the attribute sub-objects in `logs-ollama.otel-default` were found to be incorrectly typed as Elasticsearch `flattened`. The `flattened` type stores numeric leaf values as keyword strings, which made numeric aggregations (avg, percentiles, range queries) on `attributes.ollama.duration_ns` and `attributes.http.response.status_code` impossible — avg aggregations failed with "type [flattened] is not supported".

**Root cause:** The managed `otel@mappings` component template includes a dynamic template named `complex_attributes` with `path_match: "attributes.*"`. Because `*` in an Elasticsearch path match is greedy and matches dots, `attributes.*` captures not only `attributes.ollama` and `attributes.http` but also `attributes.ollama.duration_ns`, `attributes.http.response.status_code`, and every other nested level — mapping all of them as `type: flattened`.

**Fix — two objects created on the cloud cluster:**

| Object | Name | Notes |
|---|---|---|
| Component template | `logs-ollama.otel-attrs@custom` | 8 dynamic templates mapping `attributes.{http,ollama,url,client}` and their `.*` sub-objects as `type: object` (dynamic), exempting those subtrees from the `flattened` catch-all |
| Index template | `logs-ollama.otel@template` (priority 200) | Pattern `logs-ollama.otel-*`; overrides the generic `logs-otel@template` (priority 120) for this stream only; `logs-ollama.otel-attrs@custom` is placed **before** `otel@mappings` in `composed_of` |

The ordering in `composed_of` is the critical mechanism. Elasticsearch applies dynamic templates in first-match-wins order across component templates in sequence. Placing `logs-ollama.otel-attrs@custom` before `otel@mappings` ensures the custom `object` mappings for the attribute subtrees fire before `complex_attributes` can apply `flattened`.

!!! warning "Do not edit `logs-otel@custom` for this fix"
    The shared `logs-otel@custom` component template — the conventional hook for OTel-stream customizations — cannot be used here. Editing its `attributes.properties` block breaks an alias (`error.exception.message → attributes.exception.message`) that is validated across multiple OTel index templates. The stream-specific index template (`logs-ollama.otel@template`) is the only safe path.

**Data stream rollover and reindex:** The data stream was rolled over to new write index `.ds-logs-ollama.otel-default-2026.06.29-000003`, which adopts the correct mapping. All 1,203 historical documents were reindexed into the new write index.

**Resulting field types (verified after rollover):**

| Field | Type | Aggregatable |
|---|---|---|
| `attributes.ollama.duration_ns` | `long` | Yes — avg, percentiles, range |
| `attributes.http.response.status_code` | `long` | Yes |
| `attributes.http.request.method` | `keyword` | Yes |
| `attributes.url.path` | `wildcard` | Yes |
| `attributes.client.address` | `keyword` | Yes |

!!! note "Resolved 2026-06-30 — Old Backing Indices Deleted"
    Backing indices `.ds-logs-ollama.otel-default-2026.06.29-000001` and `.ds-logs-ollama.otel-default-2026.06.29-000002` were deleted by Phillip on 2026-06-30. Duplicate results eliminated — document count reduced from ~3,822 to 2,743. Aggregations on `attributes.ollama.duration_ns` and `attributes.http.response.status_code` verified working. See [Known Issues — 2026-06-30 history](../known-issues.md#2026-06-30-ollama-otel-old-backing-indices-deleted).

---

### Logstash

**Config:** `/etc/logstash/conf.d/99-elasticsearch-output.conf`
**Permissions:** root:logstash 640

!!! note "Security Improvement — 2026-06-29"
    This file was previously `644` world-readable and contained the `beats_writer` password in plaintext. It is now `640 root:logstash`, restricting read access to root and the logstash system user only. The API key that replaced the password is not visible to other users.

Logstash receives syslog on UDP/TCP port **5514** from three network devices:

| Device | IP | Syslog Status |
|---|---|---|
| ASUS RT-AX5400 (primary router) | `192.168.50.1` | Confirmed flowing (~180+ docs as of 2026-06-29) |
| OPNsense firewall | `192.168.50.253` | Confirmed flowing (~400+ docs as of 2026-06-29) |
| SAN01 Synology NAS (DSM 7.x) | `192.168.50.165` | Confirmed flowing (2026-06-29 — verified with DSM test log) |

All three Elasticsearch output blocks in the config were switched from HTTP + `beats_writer`/password to HTTPS + API key:

```ruby
elasticsearch {
  hosts => ["https://gateway.es.us-central1.gcp.cloud.es.io:443"]
  api_key => "<api_key>"
  ssl_enabled => true
  data_stream => true
  data_stream_type => "logs"
  data_stream_dataset => "syslog.opnsense"   # or syslog.asus / syslog.other
  data_stream_namespace => "default"
}
```

Destination data streams:

| Data Stream | Content | Status |
|---|---|---|
| `logs-syslog.opnsense-default` | OPNsense firewall syslog | Confirmed flowing |
| `logs-syslog.asus-default` | ASUS RT-AX5400 router syslog | Confirmed flowing (~180+ docs as of 2026-06-29) |
| `logs-syslog.synology-default` | SAN01 Synology NAS syslog | Confirmed flowing (2026-06-29) |
| `logs-syslog.other-default` | Syslog from any other device on port 5514 | Active |

#### SAN01 Synology NAS — Log Center Forwarding (Added 2026-06-29)

SAN01 (`192.168.50.165`, Synology DiskStation DSM 7.x) forwards syslog to sheepsoc via DSM's built-in **Log Center → Log Sending** feature (Control Panel → Log Center → Log Sending). Settings applied on the NAS:

| Setting | Value |
|---|---|
| Server | `192.168.50.100` |
| Port | `5514` |
| Protocol | UDP |
| Log format | BSD (RFC3164) |

!!! note "No Containers on SAN01 — Syslog Is the Ceiling"
    SAN01's CPU architecture does **not** support Container Manager (Docker). Filebeat and Elastic Agent containers cannot run on this NAS. DSM Log Center syslog forwarding is the only viable method for shipping logs off SAN01. Do not propose a container-based shipper for this device.

A new device-tag branch in `/etc/logstash/conf.d/50-syslog-filter.conf` identifies SAN01 events by source IP:

```ruby
if [host] == "192.168.50.165" {
  mutate {
    add_field => { "device_type" => "synology" }
    add_field => { "device_name" => "san01" }
  }
}
```

A dedicated output block in `/etc/logstash/conf.d/99-elasticsearch-output.conf` routes SAN01 syslog to its own Elastic Cloud data stream:

```ruby
elasticsearch {
  hosts => ["https://gateway.es.us-central1.gcp.cloud.es.io:443"]
  api_key => "<api_key>"
  ssl_enabled => true
  data_stream => true
  data_stream_type => "logs"
  data_stream_dataset => "syslog.synology"
  data_stream_namespace => "default"
}
```

Verified working 2026-06-29: a DSM test log landed in `logs-syslog.synology-default`, correctly tagged with `device_type: synology` and `device_name: san01`.

!!! warning "Privacy — Network Device and NAS Syslog Egresses to Elastic Cloud"
    OPNsense firewall logs, ASUS router logs, and SAN01 NAS syslog contain network flow records, DNS queries, NAT events, and device system events. This data egresses to Elastic Cloud (GCP us-central1). Phillip explicitly authorized this on 2026-06-29. See [Known Issues — System and Firewall Log Egress to Elastic Cloud](../known-issues.md#system-and-firewall-log-egress-to-elastic-cloud).

---

## Cloud Data Streams (Added 2026-06-29)

All data streams below are in Elastic Cloud 9.4.0 (GCP us-central1). Most use the generic `logs-*-*` index template; `logs-ollama.otel-default` uses the dedicated `logs-ollama.otel@template` (priority 200) — see [Attribute Field Type Fix](#ollama-otel-mapping-fix) above.

| Data Stream | Source | Shipper | Status |
|---|---|---|---|
| `logs-ollama.otel-default` | Ollama journald + llama.cpp runner metrics → `logs-ollama-otel` pipeline (62 processors, extended 2026-06-30) | Filebeat | Active |
| `filebeat-8.18.3` | System journal + sheepsoc app logs | Filebeat | Active (app log input currently idle) |
| `logs-syslog.opnsense-default` | OPNsense firewall syslog on port 5514 | Logstash | Confirmed flowing |
| `logs-syslog.asus-default` | ASUS RT-AX5400 syslog on port 5514 | Logstash | Confirmed flowing (~180+ docs as of 2026-06-29) |
| `logs-syslog.synology-default` | SAN01 Synology NAS syslog on port 5514 | Logstash | Confirmed flowing (2026-06-29) |
| `logs-syslog.other-default` | Any other syslog on port 5514 | Logstash | Active |

## Health Checks

```bash
# Filebeat service status
pmabry@sheepsoc:~$ systemctl status filebeat.service

# Filebeat logs — look for successful export messages, no connection errors
pmabry@sheepsoc:~$ journalctl -u filebeat.service -n 50 --no-pager

# Logstash service status
pmabry@sheepsoc:~$ systemctl status logstash.service

# Logstash logs — look for "Pipeline started" and no export errors
pmabry@sheepsoc:~$ journalctl -u logstash.service -n 50 --no-pager

# Verify Logstash syslog port is listening (UDP and TCP)
pmabry@sheepsoc:~$ ss -ulnp | grep 5514
pmabry@sheepsoc:~$ ss -tlnp | grep 5514

# Confirm data arriving in Elastic Cloud:
# Kibana (Cloud) → Stack Management → Index Management → Data Streams → filter "syslog" or "ollama"
```

## Known Issues / Gotchas

- **Shared API key** — Filebeat, Logstash, and the [OpenTelemetry Collector](otelcol-contrib.md) all authenticate to Elastic Cloud with the same API key. Revoking or rotating the key affects all three simultaneously. See [Known Issues — Shared Cloud API Key](../known-issues.md#shared-elastic-cloud-api-key-couples-filebeat-logstash-and-otelcol-contrib).
- **`reroute` processor drops docs — use `set _index`** — documented in detail in the [Ingest Pipeline gotcha above](#ingest-pipeline-logs-ollama-otel). This applies to any future pipeline where the source data stream name is not a valid `type-dataset-namespace` triple.
- **Data egress** — system journal (auth, sudo, SSH), Ollama logs, and firewall syslog all leave sheepsoc and are stored on Elastic Cloud GCP us-central1. See [Known Issues — System and Firewall Log Egress to Elastic Cloud](../known-issues.md#system-and-firewall-log-egress-to-elastic-cloud).

## See Also

- [Elasticsearch & ELSER](elasticsearch-elser.md) — **stores data in** Elastic Cloud 9.4.0 — destination for all log-shipping data streams
- [OpenTelemetry Collector](otelcol-contrib.md) — **see also** — ships OpenWebUI and Claude Code OTLP telemetry to cloud via a separate OTLP path; shares the cloud API key with Filebeat and Logstash
- [Ollama](ollama.md) — **monitored by** Filebeat; `ollama.service` journald output and llama.cpp runner metrics are shipped via the `logs-ollama-otel` pipeline to `logs-ollama.otel-default`
- [Services](../services.md) — service catalog entries for Filebeat and Logstash
- [Known Issues](../known-issues.md) — active watchlist items for this pipeline
