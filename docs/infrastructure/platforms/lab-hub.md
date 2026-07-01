# Sheepsoc Lab Hub

**Purpose:** A LAN-hosted "State of the Lab" web dashboard that probes service health, computes drift against the [Services catalog](../services.md), generates plain-English summaries via local [Ollama](ollama.md) inference, and delivers a daily security and health brief to Element via the [Matrix Bot](matrix-bot.md) outbox.

## Overview

The Lab Hub (internal name: **labhub**) is a multi-phase service first deployed on 2026-06-29. Code lives at `~/repositories/labhub`; the conda environment is `labhub`.

### Phase 1 — Service Health and Drift Detection (deployed 2026-06-29)

**Collector** (`labhub-collect.timer` / `labhub-collect.service`): a oneshot Python process that runs every 3 minutes. It probes each service by calling `systemctl is-active` and opening TCP connections to known ports, using the Service Catalog table in [Services](../services.md) as the authoritative expected-state source of truth. Two drift categories are computed:

- *documented-but-down* — a service listed in the catalog as expected-up that is not responding
- *running-but-undocumented* — a TCP listener or active unit not referenced in the catalog

Snapshots are written as JSON to a local SQLite store at `~/repositories/labhub/var/labhub.sqlite3`. The collector is strictly read-only and runs as user `pmabry` with no elevated privileges.

**Web UI** (`labhub-web.service`): a FastAPI + Jinja2 application that reads the latest snapshot from SQLite and renders health tiles and a drift list in the browser. Exposed on port 8800, bound to the LAN IP only.

### Phase 2 — Local Ollama Summary (deployed)

Each collector cycle, the `summarize.py` module sends the current health snapshot to the local [Ollama](ollama.md) model **`qwen3`** and generates a short plain-English "State of the Lab" blurb. This blurb is stored with the snapshot and rendered at the top of the web page. Inference is entirely local — no cloud calls. Configured via `OLLAMA_*` environment variables.

### Phase 3a — Daily Brief via `labhub synthesize` (deployed; timer staged, not enabled)

The `labhub synthesize` CLI command runs the `synth.py` module, which uses the local [Ollama](ollama.md) reasoning model **`deepseek-r1:14b`** to write a fuller daily "State of the Lab" brief covering service health, drift, and security. The brief is stored in the `digest` table of the SQLite database and rendered as a "Daily brief" box on the web page.

After synthesis, the brief is delivered to Element by writing a `.txt` file to the [Matrix Bot](matrix-bot.md) outbox at `~/repositories/matrix-bot/outbox/`. The bot picks up files in that directory and posts them to the configured channel.

Two systemd units to automate this — `labhub-synth.service` and `labhub-synth.timer` — are **staged in `~/repositories/labhub/deploy/` but not yet installed or enabled**. Daily delivery is available but not yet scheduled; the command runs successfully when invoked manually.

### Phase 3b — Security Strip in the Daily Brief (deployed)

The `es.py` module reads [Elastic Cloud](otelcol-contrib.md) 9.4.0 (read-only) for security telemetry included in both the daily brief and as a "Security (last 24h)" summary line on the web page. Two data sources are queried:

- **Host authentication** — `logs-system.auth-default` (ECS schema): server-side aggregation of failed vs successful logins, top attacker source IPs and country, most-targeted usernames, sudo event count.
- **OPNsense firewall blocks** — `logs-syslog.opnsense-default` (structured fields available since the [2026-07-01 filterlog ECS fix](log-shipping.md#logstash-filterlog-ecs-field-fix-2026-07-01)): total block count and top blocked IPv4 source IPs/ports. IPv6 breakdown is pending resolution of the IPv4-only filterlog dissect limitation.

Elastic Cloud access uses a **read-only API key** resolved from the `ELASTICSEARCH_API_KEY` environment variable. For manual runs, this variable is sourced from the environment. When the daily timer is enabled, the key will need to be provided to the service unit via a **root-owned `EnvironmentFile`** — this has not yet been wired.

## Dependencies

- [Services](../services.md) — **reads from** the Service Catalog table as the authoritative expected-state source of truth for drift detection
- [Ollama](ollama.md) — **depends on** for local inference; Phase 2 uses `qwen3` (plain-English summaries); Phase 3a uses `deepseek-r1:14b` (daily brief synthesis). Both models must be present in Ollama (`ollama list`). Ollama must be running for summaries and synthesis to succeed.
- [Matrix Bot](matrix-bot.md) — **depends on** the outbox delivery mechanism; Phase 3a drops `.txt` files in `~/repositories/matrix-bot/outbox/` for the bot to post to Element. `matrix-bot.service` must be running for delivery to succeed.
- [Elasticsearch & ELSER / Elastic Cloud](otelcol-contrib.md) — **reads from** Elastic Cloud 9.4.0 (read-only); Phase 3b (`es.py`) queries `logs-system.auth-default` and `logs-syslog.opnsense-default` for security telemetry. Requires `ELASTICSEARCH_API_KEY` in the environment.
- [Log Shipping — Filebeat & Logstash](log-shipping.md) — the `logs-syslog.opnsense-default` data stream that Phase 3b reads is populated by Logstash. The filterlog structured fields (`src_ip`, `dst_ip`, `action`, `firewall` tag) are available since the [2026-07-01 ECS field fix](log-shipping.md#logstash-filterlog-ecs-field-fix-2026-07-01).

## Configuration

### systemd Units

Installed units live in `/etc/systemd/system/`. Staged units are in `~/repositories/labhub/deploy/` and are not yet installed.

| Unit | Type | State | Description |
|---|---|---|---|
| `labhub-web.service` | long-running | **installed** | FastAPI web UI; bound to `192.168.50.100:8800` |
| `labhub-collect.timer` | timer | **installed** | Fires every 3 minutes; triggers the collector oneshot |
| `labhub-collect.service` | oneshot | **installed** | Collector — probes unit states and TCP ports; writes JSON snapshot to SQLite |
| `labhub-synth.service` | oneshot | **staged only** | Synthesis unit — runs `labhub synthesize`; delivers daily brief to Matrix outbox |
| `labhub-synth.timer` | timer | **staged only** | Would trigger `labhub-synth.service` on a daily schedule — not yet active |

### Conda Environment

All Lab Hub components run in a shared conda environment.

| Key | Value |
|---|---|
| Env name | `labhub` |
| Python | 3.12 |
| Path | `/home/pmabry/infrastructure/miniconda3/envs/labhub` |
| Phase 1–2 deps (pip) | `fastapi`, `uvicorn`, `jinja2`, `httpx` |
| Phase 3b deps (pip) | `elasticsearch` (Elastic Cloud read client) |

See [Conda Guide](conda.md) for environment management commands.

### Firewall

A LAN-only UFW rule was added on 2026-06-29:

```bash
# LAN access only — no WAN, no Tailscale tailnet exposure
pmabry@sheepsoc:~$ sudo ufw allow from 192.168.50.0/24 to any port 8800 proto tcp
```

Port 8800 is not exposed over Tailscale Serve. Access is restricted to `192.168.50.0/24`.

### Ollama Configuration

Phase 2 and Phase 3a use local Ollama inference. The following environment variables configure the Ollama endpoint and model selection:

| Variable | Default / Notes |
|---|---|
| `OLLAMA_HOST` | `http://127.0.0.1:11434` |
| `OLLAMA_MODEL_SUMMARY` | `qwen3` — used by `summarize.py` for the per-cycle blurb |
| `OLLAMA_MODEL_SYNTH` | `deepseek-r1:14b` — used by `synth.py` for the daily brief |

Both models must be present in Ollama. Verify with `ollama list`.

### Elastic Cloud Read Access (Phase 3b)

The `es.py` module authenticates to Elastic Cloud 9.4.0 using a read-only API key. The key is never written to disk in the wiki.

| Variable | Notes |
|---|---|
| `ELASTICSEARCH_API_KEY` | Read-only key for Elastic Cloud 9.4.0; resolved from the environment at runtime |
| `ELASTICSEARCH_HOST` | `https://gateway.es.us-central1.gcp.cloud.es.io:443` |

**For manual runs:** export `ELASTICSEARCH_API_KEY=<api-key>` before running `labhub synthesize`.

**When the daily timer is enabled:** the key must be provided to `labhub-synth.service` via a root-owned `EnvironmentFile` (e.g., `/etc/labhub/labhub.env`, `chmod 600 root:root`). This wiring has not yet been done — it is a prerequisite for enabling the timer.

!!! warning "Do Not Write API Keys to the Wiki or to Config Files Tracked by Git"
    The Elastic Cloud API key must never appear in this wiki, in `config.yaml`, or in any file tracked by git. Use `<api-key>` as a placeholder throughout. Store the actual value only in the `EnvironmentFile`, with root-only permissions.

### API Endpoints

| Path | Description |
|---|---|
| `/` | Browser dashboard — health tiles, Ollama summary blurb, security summary, drift list, and daily brief box |
| `/api/state` | Latest snapshot as JSON |
| `/healthz` | Liveness probe — returns `{"ok":true}` when healthy |

## CLI Subcommands

The Lab Hub CLI is invoked as `labhub <subcommand>` from within the `labhub` conda environment.

| Subcommand | Description |
|---|---|
| `collect` | Run a single collection cycle immediately (same as the timer-triggered oneshot) |
| `serve` | Start the FastAPI web UI (same process launched by `labhub-web.service`) |
| `synthesize` | Run the daily brief — queries Ollama (`deepseek-r1:14b`) and Elastic Cloud (`es.py`), writes the brief to the SQLite `digest` table, renders it on the page, and drops a `.txt` in the Matrix bot outbox for Element delivery |

```bash
# Run the synthesizer manually (requires ELASTICSEARCH_API_KEY exported in the current shell)
pmabry@sheepsoc:~$ conda run -n labhub labhub synthesize
```

## Health Checks

```bash
# Confirm the web service is active
pmabry@sheepsoc:~$ systemctl is-active labhub-web.service

# Confirm the timer is scheduled and see its next trigger time
pmabry@sheepsoc:~$ systemctl list-timers labhub-collect.timer

# Probe the health endpoint (expect {"ok":true})
pmabry@sheepsoc:~$ curl http://192.168.50.100:8800/healthz

# Check recent collector output
pmabry@sheepsoc:~$ journalctl -u labhub-collect.service -n 50 --no-pager
```

## Data Store

`~/repositories/labhub/var/labhub.sqlite3` — local SQLite database holding time-series JSON snapshots (from the collector), and a `digest` table for daily brief records (from `labhub synthesize`). Written by the collector and synthesizer, read by the web tier. All data remains on sheepsoc.

## Source Relationship — Services Catalog

The Lab Hub **reads from** [Services](../services.md) as its expected-state source of truth. The Service Catalog table in that page is the authoritative list of what *should* be running. When a service is added to or removed from the catalog, the hub incorporates the change automatically on its next collection cycle.

!!! note "Keeping the Catalog Accurate"
    Accurate drift detection depends on an accurate catalog. A service that is running but absent from the catalog is flagged as *running-but-undocumented*. A service in the catalog but not responding is flagged as *documented-but-down*. If the catalog drifts from reality, so does the hub's output.

## Known Issues / Gotchas

- **`labhub-synth` units staged but not enabled** — `labhub-synth.service` and `labhub-synth.timer` are checked in to `~/repositories/labhub/deploy/` but have not been installed to `/etc/systemd/system/` or enabled. Daily brief delivery to Element is available via `labhub synthesize` but is not yet automated. To enable: install both unit files, provide the Elastic Cloud API key via a root-owned `EnvironmentFile` (see [Elastic Cloud Read Access](#elastic-cloud-read-access-phase-3b)), then `sudo systemctl enable --now labhub-synth.timer`.
- **IPv6 firewall data absent from security strip** — `es.py` reports only IPv4 block data. IPv6 filterlog lines are not yet parsed by Logstash (the dissect pattern is IPv4-only by design). See [Known Issues — Logstash filterlog dissect IPv4-only](../known-issues.md#logstash-filterlog-dissect-ipv4-only).

## Implementation Status

| Phase | Status | Summary |
|---|---|---|
| Phase 1 | **deployed 2026-06-29** | Service health tiles + documentation-drift detection |
| Phase 2 | **deployed** | Local Ollama (`qwen3`) plain-English summary blurb per collector cycle |
| Phase 3a | **deployed; timer staged** | `labhub synthesize` daily brief via `deepseek-r1:14b`; Matrix outbox delivery; synth units in `deploy/`, not yet installed |
| Phase 3b | **deployed** | Security strip: Elastic Cloud auth + OPNsense firewall aggregations in daily brief and web page |

The implementation plan lives at `~/infrastructure/lab-hub-implementation-plan.md` (outside the wiki — do not copy content from it into this page).

## See Also

- [Services](../services.md) — Service Catalog (**reads from** this page as expected-state source of truth; backlink present)
- [Ollama](ollama.md) — **depends on** for local inference; `qwen3` and `deepseek-r1:14b` must be present and Ollama must be running
- [Log Shipping — Filebeat & Logstash](log-shipping.md) — **depends on** via Phase 3b; `logs-syslog.opnsense-default` is the OPNsense firewall data source; filterlog structured fields available since the 2026-07-01 ECS fix
- [OpenTelemetry Collector](otelcol-contrib.md) — **see also** — Elastic Cloud 9.4.0 is the destination for Phase 3b security queries; the shared API key couples this with OTel and log-shipping credentials
- [Matrix Bot](matrix-bot.md) — **depends on** the outbox delivery path for Phase 3a daily brief; the bot must be running for Element delivery to work
- [Conda Guide](conda.md) — Python environment management for the `labhub` env
- [Known Issues](../known-issues.md) — site-wide active issues and watchlist
