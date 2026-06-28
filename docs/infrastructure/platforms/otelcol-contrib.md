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

## Known Issues / Gotchas

- **Claude Code prompt content egresses to Elastic Cloud:** With `OTEL_LOG_USER_PROMPTS=1` and `OTEL_LOG_TOOL_DETAILS=1` set, full prompt and tool-argument text is logged and sent to Elastic Cloud GCP us-central1. This data leaves sheepsoc. See [Known Issues](../known-issues.md#claude-code-prompts-egress-to-elastic-cloud).
- **OpenWebUI crash-loop if packages are missing:** Enabling `ENABLE_OTEL=true` in the OpenWebUI drop-in without the 9 required OpenTelemetry Python packages installed in the `openwebui` conda env causes an immediate crash-loop. See [Known Issues](../known-issues.md#openwebui-crash-loop-with-enable_otel-true-without-packages) and [OpenWebUI — OpenTelemetry Configuration](openwebui-rag.md#opentelemetry-configuration) for the package list.
- **Port 8889, not 8888:** The collector self-metrics port was moved from the default `8888` to `8889` because Jupyter already uses `8888`. Any tooling expecting Prometheus metrics at `:8888` must target `:8889` instead.

## See Also

- [OpenWebUI & RAG](openwebui-rag.md) — **monitored by** this collector; OTEL configuration lives in a systemd drop-in on the OpenWebUI service
- [Elasticsearch & ELSER](elasticsearch-elser.md) — **stores data in** — Elastic Cloud 9.4.0 is the export target for all OTEL data streams
- [Services](../services.md) — service catalog entry, port reference, and health check commands
- [Known Issues](../known-issues.md) — active watchlist items for this pipeline
