# Sheepsoc Lab Hub

**Purpose:** A read-only, LAN-hosted "State of the Lab" web dashboard that probes systemd unit states and TCP ports, computes drift against the [Services catalog](../services.md), and renders health tiles and a drift list for the household.

## Overview

Phase 1 (current). The Lab Hub (internal name: **labhub**) is a two-component service deployed on 2026-06-29:

**Collector** (`labhub-collect.timer` / `labhub-collect.service`): a oneshot Python process that runs every 3 minutes. It probes each service by calling `systemctl is-active` and opening TCP connections to known ports, using the Service Catalog table in [Services](../services.md) as the authoritative expected-state source of truth. Two drift categories are computed:

- *documented-but-down* — a service listed in the catalog as expected-up that is not responding
- *running-but-undocumented* — a TCP listener or active unit not referenced in the catalog

Snapshots are written as JSON to a local SQLite store at `~/repositories/labhub/var/labhub.sqlite3`. The collector is strictly read-only and runs as user `pmabry` with no elevated privileges.

**Web UI** (`labhub-web.service`): a FastAPI + Jinja2 application that reads the latest snapshot from SQLite and renders health tiles and a drift list in the browser. Exposed on port 8800, bound to the LAN IP only. No mutating operations.

Code lives at `~/repositories/labhub`.

## Dependencies

- [Services](../services.md) — **reads from** the Service Catalog table as the authoritative expected-state source of truth for drift detection
- Collector is stdlib-only Python and requires no external packages at runtime

## Configuration

### systemd Units

Three units manage the service. Unit files live in `/etc/systemd/system/`.

| Unit | Type | Description |
|---|---|---|
| `labhub-web.service` | long-running | FastAPI web UI; bound to `192.168.50.100:8800` |
| `labhub-collect.timer` | timer | Fires every 3 minutes; triggers the collector oneshot |
| `labhub-collect.service` | oneshot | Collector — probes unit states and TCP ports; writes JSON snapshot to SQLite |

### Conda Environment

The web UI runs in a dedicated conda environment. The collector is stdlib-only and requires no conda packages, but both components share this environment for consistency.

| Key | Value |
|---|---|
| Env name | `labhub` |
| Python | 3.12 |
| Path | `/home/pmabry/infrastructure/miniconda3/envs/labhub` |
| Runtime deps (pip) | `fastapi`, `uvicorn`, `jinja2`, `httpx` |
| Collector deps | stdlib only — no extra packages needed |

See [Conda Guide](conda.md) for environment management commands.

### Firewall

A LAN-only UFW rule was added on 2026-06-29:

```bash
# LAN access only — no WAN, no Tailscale tailnet exposure
pmabry@sheepsoc:~$ sudo ufw allow from 192.168.50.0/24 to any port 8800 proto tcp
```

Port 8800 is not exposed over Tailscale Serve. Access is restricted to `192.168.50.0/24`.

### API Endpoints

| Path | Description |
|---|---|
| `/` | Browser dashboard — health tiles and drift list |
| `/api/state` | Latest snapshot as JSON |
| `/healthz` | Liveness probe — returns `{"ok":true}` when healthy |

### Data Store

`~/repositories/labhub/var/labhub.sqlite3` — local SQLite database holding time-series JSON snapshots of each collection run. Written by the collector, read by the web tier. All data remains on sheepsoc.

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

## Source Relationship — Services Catalog

The Lab Hub **reads from** [Services](../services.md) as its expected-state source of truth. The Service Catalog table in that page is the authoritative list of what *should* be running. When a service is added to or removed from the catalog, the hub incorporates the change automatically on its next collection cycle.

!!! note "Keeping the Catalog Accurate"
    Accurate drift detection depends on an accurate catalog. A service that is running but absent from the catalog is flagged as *running-but-undocumented*. A service in the catalog but not responding is flagged as *documented-but-down*. If the catalog drifts from reality, so does the hub's output.

## Known Issues / Gotchas

None as of 2026-06-29.

## Roadmap (Phases 2 and 3 — Not Yet Built)

Future phases are planned but not yet implemented. The implementation plan lives at `~/infrastructure/lab-hub-implementation-plan.md` (outside the wiki).

| Phase | Status | Summary |
|---|---|---|
| Phase 1 | **current** | Service health tiles + documentation-drift detection |
| Phase 2 | planned | Local Ollama summaries of drift state |
| Phase 3 | planned | Cloud Claude Opus 4.8 synthesis + security digest + Matrix notification push |

## See Also

- [Services](../services.md) — Service Catalog (**reads from** this page as expected-state source of truth; backlink present)
- [Conda Guide](conda.md) — Python environment management
- [Known Issues](../known-issues.md) — site-wide active issues and watchlist
