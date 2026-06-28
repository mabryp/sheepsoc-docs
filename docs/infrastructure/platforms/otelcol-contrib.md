# OpenTelemetry Collector

**Purpose:** Central OTLP telemetry hub — receives traces, metrics, and logs from OpenWebUI and Claude Code and exports them to Elastic Cloud 9.4.0 as structured data streams.

## Overview

`otelcol-contrib` v0.155.0 (OpenTelemetry Collector, contrib distribution) was installed on 2026-06-27 from the official OpenTelemetry GitHub release `.deb` package (checksum-verified, installed via `sudo dpkg -i`). It runs as a systemd service under the `otelcol-contrib` system user and is the single point of telemetry aggregation on sheepsoc.

The collector implements a single pipeline:

```
receivers [otlp] → processors [cumulativetodelta (metrics only), batch] → exporter [elasticsearch]
```

Data lands in [Elastic Cloud 9.4.0](elasticsearch-elser.md) (GCP us-central1) as OTel-format data streams. As of 2026-06-27, traces, metrics, and logs from [OpenWebUI](openwebui-rag.md) are confirmed flowing. Claude Code telemetry will appear in new data streams once a new Claude Code session starts.

!!! warning "Privacy — Claude Code Content Egresses to Elastic Cloud"
    Claude Code telemetry is configured with `OTEL_LOG_USER_PROMPTS=1` and `OTEL_LOG_TOOL_DETAILS=1`. **Full prompt text and tool-argument content is logged and exported to Elastic Cloud (GCP us-central1, cluster UUID `dab081df574b45cf894e33645053dfb3`).** This data leaves sheepsoc and is stored in a third-party cloud. See [Known Issues](../known-issues.md#claude-code-prompts-egress-to-elastic-cloud).

This implementation follows the approach from the Elastic Security Labs article "Claude Code/Cowork monitoring at scale with Otel & Elastic". Grok Code was investigated but has no user-configurable OTLP-to-self-hosted path and was not pursued.

## Dependencies

- [Elastic Cloud 9.4.0](elasticsearch-elser.md) (GCP us-central1, `gateway.es.us-central1.gcp.cloud.es.io:443`) — **stores data in** — the only export target; the collector will fail to export if the cloud cluster is unreachable
- [OpenWebUI & RAG](openwebui-rag.md) — **monitored by** this collector via OTLP/gRPC; OpenWebUI sends traces, metrics, and logs when `ENABLE_OTEL=true` is configured
- Claude Code — **monitored by** this collector via OTLP/gRPC when `CLAUDE_CODE_ENABLE_TELEMETRY=1` is set in `~/.claude/settings.json`

## Ports

All ports are bound to `127.0.0.1` only. No LAN exposure. No UFW changes were made.

| Port | Protocol | Purpose |
|---|---|---|
| `4317` | OTLP/gRPC receiver | Primary OTLP ingestion endpoint |
| `4318` | OTLP/HTTP receiver | HTTP-based OTLP ingestion endpoint |
| `13133` | HTTP | Health check endpoint |
| `8889` | HTTP | Collector self-metrics (Prometheus format) — **note:** default `8888` was not used because [Jupyter](../services.md) already occupies port `8888` |

## Configuration

### Files

| Path | Purpose |
|---|---|
| `/etc/otelcol-contrib/config.yaml` | Primary pipeline configuration (original backed up as `config.yaml.orig-20260627`) |
| `/etc/otelcol-contrib/otelcol-contrib.conf` | Environment file injected into the systemd unit — **chmod 600, root-only** — contains the `ELASTICSEARCH_API_KEY` secret |

### Pipeline Summary

```yaml
# Condensed representation of /etc/otelcol-contrib/config.yaml

receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 127.0.0.1:4317
      http:
        endpoint: 127.0.0.1:4318

processors:
  cumulativetodelta:   # metrics pipeline only — see note below
  batch:

exporters:
  elasticsearch:
    endpoints: [https://gateway.es.us-central1.gcp.cloud.es.io:443]
    api_key: ${env:ELASTICSEARCH_API_KEY}   # resolved from /etc/otelcol-contrib/otelcol-contrib.conf (chmod 600, root-only)
    mapping:
      mode: otel

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [elasticsearch]
    metrics:
      receivers: [otlp]
      processors: [cumulativetodelta, batch]
      exporters: [elasticsearch]
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [elasticsearch]
```

!!! note "`cumulativetodelta` Processor — Required"
    The `cumulativetodelta` processor is required in the metrics pipeline. The Elasticsearch exporter rejects cumulative-temporality histograms; this processor converts them to delta temporality centrally before they reach the exporter. Removing it will cause the metrics pipeline to fail.

### Systemd Unit

```bash
# Installed via dpkg; unit file managed by the package
pmabry@sheepsoc:~$ systemctl status otelcol-contrib.service
# Runs as: otelcol-contrib (dedicated system user)
# Enabled: yes (starts on boot)
```

## Health Checks

```bash
# Service status
pmabry@sheepsoc:~$ systemctl status otelcol-contrib.service

# Health check endpoint — returns HTTP 200 with "Server available" when healthy
pmabry@sheepsoc:~$ curl -s http://127.0.0.1:13133

# Confirm OTLP gRPC port is listening on loopback only (should show 127.0.0.1:4317)
pmabry@sheepsoc:~$ ss -tlnp | grep 431

# Collector logs — most recent 50 lines
pmabry@sheepsoc:~$ journalctl -u otelcol-contrib.service -n 50 --no-pager

# Verify OTEL data streams exist — check in Kibana or via the Elastic Cloud REST API
# (Data streams land in Elastic Cloud, not on local ES; no localhost:9200 path for this)
# In Kibana: Stack Management → Index Management → Data Streams → filter "otel"
```

## Data Streams in Elastic Cloud

OTEL data lands in Elastic Cloud 9.4.0 (GCP us-central1) using the OTel mapping mode. Index patterns follow Elastic's data stream naming convention, using the `data_stream.dataset` resource attribute set on each source.

| Data Stream Pattern | Source | `data_stream.dataset` | Status as of 2026-06-27 |
|---|---|---|---|
| `logs-open_webui.otel-*` | OpenWebUI | `open_webui` | Confirmed flowing |
| `metrics-open_webui.otel-*` | OpenWebUI | `open_webui` | Confirmed flowing |
| `traces-open_webui.otel-*` | OpenWebUI | `open_webui` | Confirmed flowing |
| `logs-claude_code.otel-*` | Claude Code | `claude_code` | Appears on next new Claude Code session |
| `metrics-claude_code.otel-*` | Claude Code | `claude_code` | Appears on next new Claude Code session |

!!! warning "Metrics Volume"
    OpenWebUI system-metrics instrumentation generates approximately 100,000 metric documents per 30 minutes of light use, driven by the `opentelemetry-instrumentation-system-metrics` package at a 10-second export interval. This data lands in Elastic Cloud — while there is no local disk pressure, high ingestion volume may affect cloud storage consumption. If the rate becomes a concern, consider raising `OTEL_METRICS_EXPORT_INTERVAL_MILLIS` in the OpenWebUI drop-in or removing the `opentelemetry-instrumentation-system-metrics` package from the `openwebui` conda env. See [Known Issues — OTEL Metrics Volume](../known-issues.md#otel-metrics-volume-high-watch-item).

## Claude Code Token & Cost Dashboard

**Dashboard name:** Claude Code — Token & Cost Tracking
**Location:** Elastic Cloud Kibana (GCP us-central1) — **not** the local Kibana at `http://192.168.50.100:5601`. The local Kibana is connected to local Elasticsearch 8.19.14, which does not contain the `metrics-claude_code.otel-*` data streams. All OTEL data lives in Elastic Cloud.

This dashboard was built in Kibana Lens on 2026-06-28 to track token consumption and API cost for Claude Code sessions running on sheepsoc.

### Data View

Use a data view scoped to `metrics-claude_code.otel-default` — the single concrete data stream that holds Claude Code metrics.

!!! danger "Do Not Use a Wildcard Data View"
    Do NOT use a data view that matches `metrics-*.otel-*` or any other pattern spanning both `metrics-claude_code.otel-default` and `metrics-open_webui.otel-default`. The `attributes.type` field is mapped as `keyword` in the Claude Code stream but as `long` in the OpenWebUI stream. Kibana treats this as a field-type conflict and hides `attributes.type` from Lens aggregations and breakdowns, making per-type token breakdowns impossible. See [Known Issues — OTEL `attributes.type` field conflict](../known-issues.md#otel-attributes-type-field-conflict). Note: `attributes.model` is `keyword` in all streams and does not conflict; OpenWebUI metrics carry no `attributes.model` field, so model breakdowns remain valid in a scoped Claude Code view.

### Metric Fields

`metrics-claude_code.otel-default` uses the `metrics-otel@template` index template and is stored as a TSDB (time series database) data stream. All value fields are `double` and typed as `gauge`.

**Value fields** (aggregatable):

| Field | Meaning |
|---|---|
| `metrics.claude_code.token.usage` | Token count per export interval, segmented by `attributes.type` and `attributes.model` |
| `metrics.claude_code.cost.usage` | Dollar cost per export interval, with per-type pricing multipliers applied |
| `metrics.claude_code.session.count` | Distinct Claude Code sessions active in the interval |
| `metrics.claude_code.lines_of_code.count` | Lines of code added or removed in the interval |
| `metrics.claude_code.commit.count` | Git commits made during the interval |
| `metrics.claude_code.code_edit_tool.decision` | Code edit tool invocation count |
| `metrics.claude_code.active_time.total` | Active session time in the interval |

**TSDB dimension fields** (type `keyword`):

| Field | Representative values |
|---|---|
| `attributes.type` | `input`, `output`, `cacheRead`, `cacheCreation` (token/cost metrics); others for activity metrics |
| `attributes.model` | Model identifier, e.g. `claude-sonnet-4-6` |
| `attributes.session.id` | Unique identifier per Claude Code session |
| `attributes.user.email` | User email from Claude Code authentication |

### Critical Aggregation Rule

!!! warning "Always Aggregate with Sum — Never Avg, Last, or Median"
    The `cumulativetodelta` processor in the [otelcol-contrib pipeline](#configuration) converts cumulative counters to per-interval deltas before export. Each document in the data stream represents the count or cost that occurred in that specific export interval — not a running total. **To compute totals over any time window, sum the deltas.** Using average, last value, or median will produce meaningless results that severely undercount spend or token usage.

### Dashboard Panels

All panels use the `metrics-claude_code.otel-default` data view. All y-axis aggregations use `Sum`.

| Panel | Visualization | Key settings |
|---|---|---|
| **Tokens over time** | Stacked vertical bar chart | x = `@timestamp`; y = `Sum(metrics.claude_code.token.usage)`; breakdown by `attributes.type`; panel filter `attributes.type: ("input" or "output" or "cacheRead" or "cacheCreation")` |
| **Tokens by model** | Vertical bar chart | x = `attributes.model`; y = `Sum(metrics.claude_code.token.usage)` |
| **Cost over time** | Stacked vertical bar chart | x = `@timestamp`; y = `Sum(metrics.claude_code.cost.usage)`; breakdown by `attributes.model` |
| **Total cost ($)** | Metric (big number) | `Sum(metrics.claude_code.cost.usage)` |
| **Cache efficiency** | Metric (big number, percent format) | Lens formula: `sum(metrics.claude_code.token.usage, kql='attributes.type:cacheRead') / sum(metrics.claude_code.token.usage)` — observed ~93% |
| **Top sessions** | Horizontal bar chart or table | Top 10 `attributes.session.id` by `Sum(metrics.claude_code.token.usage)` |
| **Activity — sessions** | Metric | `Sum(metrics.claude_code.session.count)` |
| **Activity — lines of code** | Metric | `Sum(metrics.claude_code.lines_of_code.count)` |
| **Activity — active time** | Metric | `Sum(metrics.claude_code.active_time.total)` |

### Token Type Reference

Claude Code reports four token types via `attributes.type`. Each carries a different billing weight:

| Type | Meaning | Relative cost |
|---|---|---|
| `input` | Standard prompt tokens processed by the model | Baseline (1×) |
| `output` | Tokens generated in the model's response | ~3× input price |
| `cacheCreation` | Tokens written to the prompt cache (first-time cache population) | ~1.25× input price |
| `cacheRead` | Tokens served from the prompt cache on subsequent requests | ~0.1× input price (~90% cheaper than input) |

Because these types carry very different billing rates, **`cost.usage` is the authoritative spend signal** — it applies the correct multiplier for each type. Raw token totals (from `token.usage`) overstate cost when `cacheRead` volume is high.

A `cacheRead` share of ~93% (as observed) represents the cost-efficient operating state: the large majority of tokens are served from cache rather than re-processed by the model.

## Known Issues / Gotchas

- **Claude Code prompt content egresses to Elastic Cloud:** With `OTEL_LOG_USER_PROMPTS=1` and `OTEL_LOG_TOOL_DETAILS=1` set, full prompt and tool-argument text is logged and sent to Elastic Cloud GCP us-central1. This data leaves sheepsoc. See [Known Issues](../known-issues.md#claude-code-prompts-egress-to-elastic-cloud).
- **OpenWebUI crash-loop if packages are missing:** Enabling `ENABLE_OTEL=true` in the OpenWebUI drop-in without the 9 required OpenTelemetry Python packages installed in the `openwebui` conda env causes an immediate crash-loop. See [Known Issues](../known-issues.md#openwebui-crash-loop-with-enable_otel-true-without-packages) and [OpenWebUI — OpenTelemetry Configuration](openwebui-rag.md#opentelemetry-configuration) for the package list.
- **Port 8889, not 8888:** The collector self-metrics port was moved from the default `8888` to `8889` because Jupyter already uses `8888`. Any tooling expecting Prometheus metrics at `:8888` must target `:8889` instead.
- **`attributes.type` field-type conflict across metrics data streams (2026-06-28):** `metrics-claude_code.otel-default` maps `attributes.type` as `keyword`; `metrics-open_webui.otel-default` maps it as `long`. A Kibana data view spanning both streams (`metrics-*.otel-*`) sees the field as conflicted and hides it from Lens aggregations and breakdowns, blocking unified cross-source dashboards that break down by token or event type. **Workaround:** use per-source data views. **Possible fix:** add an `attributes` or `transform` processor to the collector pipeline to normalize the field before export, or remap the field type in one data stream's index template. See [Known Issues — OTEL `attributes.type` field conflict](../known-issues.md#otel-attributes-type-field-conflict).

## See Also

- [OpenWebUI & RAG](openwebui-rag.md) — **monitored by** this collector; OTEL configuration lives in a systemd drop-in on the OpenWebUI service
- [Elasticsearch & ELSER](elasticsearch-elser.md) — **stores data in** — Elastic Cloud 9.4.0 is the export target for all OTEL data streams
- [Services](../services.md) — service catalog entry, port reference, and health check commands
- [Known Issues](../known-issues.md) — active watchlist items for this pipeline
