# Ollama

**Purpose:** Local LLM inference server on sheepsoc — serves all on-device model requests from [OpenWebUI & RAG](openwebui-rag.md), the [Matrix Bot](matrix-bot.md), RAG embeddings, Claude Code, and the [Lab Hub](lab-hub.md) (plain-English summaries and daily brief synthesis).

## Overview

Ollama 0.30.10 runs on sheepsoc as a systemd service, using the RTX 5060 Ti (16 GB VRAM) for GPU-accelerated inference. It stores 14 models at `/home/pmabry/.ollama` and exposes five verified API endpoints including a native Anthropic Messages API (available since 0.14.0). Upgraded from 0.9.4 on 2026-06-26.

Two systemd drop-ins tune GPU memory usage for the 16 GB VRAM ceiling — concurrency and default context length. Both were reconfigured on 2026-07-02; see [Configuration](#configuration) below.

The service runs as `pmabry` (not the Ollama default `ollama` user) and binds to all interfaces on port 11434 so that LAN clients and the OpenWebUI / Matrix bot services can reach it.

## Dependencies

- **NVIDIA GPU** — RTX 5060 Ti, 16 GB VRAM. The service will start without a GPU but inference falls back to CPU. See [Known Issues — NVIDIA driver landmine](../known-issues.md#active-landmines-do-not-touch) before touching driver packages.
- **[OpenWebUI & RAG](openwebui-rag.md)** — **depends on** Ollama for LLM chat inference and RAG embeddings (`nomic-embed-text`). If Ollama goes down or is misconfigured, OpenWebUI chat and all RAG document uploads fail.
- **[Matrix Bot](matrix-bot.md)** — **depends on** Ollama for inference via the OpenWebUI HTTP API path.
- **[Log Shipping — Filebeat & Logstash](log-shipping.md)** — **monitored by** Filebeat; `ollama.service` journald output is shipped to Elastic Cloud 9.4.0 via the `logs-ollama-otel` ingest pipeline and stored in the `logs-ollama.otel-default` data stream. As of 2026-06-30, the pipeline also indexes llama.cpp runner performance metrics — six event types including `generation` (which captures `eval_tokens_per_sec`, the primary throughput metric). See [Log Shipping — Runner Metric Event Types](log-shipping.md#runner-metric-event-types-added-2026-06-30) for the full field catalog.
- **[Lab Hub](lab-hub.md)** — **depends on** Ollama for local inference; Phase 2 uses `qwen3` for per-cycle plain-English summaries; Phase 3a uses `deepseek-r1:14b` for daily brief synthesis. Both models must be present and the service must be running for Lab Hub summaries to succeed.

## Configuration

### systemd Unit

`/etc/systemd/system/ollama.service` — runs as `pmabry`, binds all interfaces.

!!! danger "Do not run the official Ollama installer without restoring the unit file afterward"
    The official install script (`curl -fsSL https://ollama.com/install.sh | sh`) **rewrites this file to its defaults** — `User=ollama`, binds `127.0.0.1` only, no custom env vars. That breaks OpenWebUI, RAG embeddings, and Matrix bot access until the file is restored. See [Upgrade Procedure](#upgrade-procedure) below before using the installer.

Key directives in the unit file:

```ini
[Service]
User=pmabry
Environment="OLLAMA_HOST=0.0.0.0:11434"
Environment="OLLAMA_MODELS=/home/pmabry/.ollama"
Environment="OLLAMA_MAX_PROMPT_TOKENS=131072"
```

### systemd Drop-ins

Two drop-in files under `/etc/systemd/system/ollama.service.d/` tune GPU memory usage for the RTX 5060 Ti's 16 GB VRAM ceiling.

#### Concurrency Limit

`/etc/systemd/system/ollama.service.d/parallel.conf`:

```ini
[Service]
Environment="OLLAMA_NUM_PARALLEL=1"
```

!!! note "Changed 2026-07-02: 5 → 1"
    `OLLAMA_NUM_PARALLEL` sets how many concurrent request slots Ollama reserves KV cache for. At `5`, every context request's KV cache was multiplied roughly 5x — a 14B model at 40K context ballooned to 44 GB and spilled 67% onto the CPU, which is very slow. At `1`, that same load fits in 16 GB at 95% GPU utilization.

    **Tradeoff:** Ollama now processes only one inference request at a time. This suits Phillip's normal single-user, long-context usage. If multiple users or automations — [OpenWebUI](openwebui-rag.md), the [Matrix Bot](matrix-bot.md), the [Lab Hub](lab-hub.md), or `claude-ollama` — call Ollama concurrently, requests now **queue** instead of overlapping. This is an expected behavioral change, not a bug. See [Known Issues — Watchlist](../known-issues.md#ollama-single-request-concurrency-since-2026-07-02).

The prior value is backed up at `/etc/systemd/system/ollama.service.d/parallel.conf.bak`.

#### Context Length

`/etc/systemd/system/ollama.service.d/context.conf` — **new drop-in, added 2026-07-02**:

```ini
[Service]
Environment="OLLAMA_CONTEXT_LENGTH=16384"
```

!!! note "Why this was added"
    Previously no default context length was set, so Ollama silently fell back to its built-in default of **4096 tokens** (roughly 3,000 words) for every client — including [OpenWebUI](openwebui-rag.md) — even for models capable of far more (up to 128K). This capped every conversation and all RAG retrieval at 4K with no indication in the UI. The new global default of 16384 tokens is sized to keep a 14B model fully on the GPU (~11 GB, 100% GPU). Individual clients can still override this per-request by sending their own `num_ctx` value.

**OpenWebUI impact:** OpenWebUI has no per-model `num_ctx` override configured — its one custom model, `network-troubleshooting` (based on `qwen3:14b`), has empty params — so it inherited the old 4096 default and now inherits 16384 automatically. No OpenWebUI database change was made. See [OpenWebUI & RAG](openwebui-rag.md).

### Config Backups

Backups of the binary and custom service config are stored at `/home/pmabry/ollama-backups/` and must be refreshed after each upgrade.

| File | Purpose |
|---|---|
| `ollama-0.9.4` | Rollback binary (prior version before 2026-06-26 upgrade) |
| `ollama.service.bak` | Custom systemd unit file snapshot |
| `ollama.service.d/parallel.conf` | Drop-in snapshot — synced 2026-07-02, reflects current `OLLAMA_NUM_PARALLEL=1` |
| `ollama.service.d/context.conf` | Drop-in snapshot — added 2026-07-02, reflects current `OLLAMA_CONTEXT_LENGTH=16384` |
| `VERSION-before.txt` | Pre-upgrade version string |

!!! note "Backup set current as of 2026-07-02"
    Both drop-ins in `/home/pmabry/ollama-backups/ollama.service.d/` are byte-for-byte identical to the live files in `/etc/systemd/system/ollama.service.d/`. `parallel.conf` was resynced to `OLLAMA_NUM_PARALLEL=1` (previously stale at `=5`), and `context.conf` was added to the backup set for the first time. The full-directory copy (rather than the in-place `.bak` files noted in [Concurrency Limit](#concurrency-limit) above) closes the gap flagged earlier the same day.

## API Endpoints

Ollama exposes the following endpoints, all verified after the 2026-06-26 upgrade to 0.30.10:

| Endpoint | API Style | Used By |
|---|---|---|
| `/api/version` | Ollama native | Health checks |
| `/api/chat` | Ollama native | OpenWebUI, Matrix bot |
| `/api/embed` | Ollama native | RAG embeddings (`nomic-embed-text`) |
| `/v1/chat/completions` | OpenAI-compatible | Kibana connector |
| `/v1/messages` | Anthropic Messages API | Claude Code (`claude-ollama` wrapper) |

The `/v1/messages` endpoint became available in Ollama ≥ 0.14.0. It returns responses in Anthropic SDK format, which is what Claude Code expects. This endpoint returned 404 on 0.9.4, which was the primary reason for the upgrade.

## Claude Code Integration

The `claude-ollama` wrapper at `/home/pmabry/.local/bin/claude-ollama` lets Claude Code target local Ollama models rather than Anthropic's cloud API. It leaves the system `claude` command and all Anthropic credentials untouched — running `claude` still goes to the cloud; running `claude-ollama` goes to `localhost:11434`.

### How It Works

The wrapper sets environment variables that redirect Claude Code's API calls to the local Ollama `/v1/messages` endpoint, then passes all arguments through to `claude`:

```bash
export ANTHROPIC_BASE_URL=http://localhost:11434
export ANTHROPIC_AUTH_TOKEN=ollama
export ANTHROPIC_MODEL=${ANTHROPIC_MODEL:-qwen3:14b}
export ANTHROPIC_SMALL_FAST_MODEL=${ANTHROPIC_SMALL_FAST_MODEL:-qwen3:latest}
export ANTHROPIC_DEFAULT_HAIKU_MODEL=${ANTHROPIC_DEFAULT_HAIKU_MODEL:-qwen3:latest}
exec claude "$@"
```

`ANTHROPIC_AUTH_TOKEN=ollama` is an arbitrary dummy value — Ollama does not enforce authentication on the `/v1/messages` endpoint.

### Default Model Choice

The Ollama documentation recommends `qwen3-coder` (30B, approximately 24 GB VRAM) for coding tasks. The sheepsoc RTX 5060 Ti has 16 GB VRAM, so the default is `qwen3:14b` to fit within that ceiling. Override at invocation time by setting `ANTHROPIC_MODEL`:

```bash
pmabry@sheepsoc:~$ ANTHROPIC_MODEL=gemma3:12b claude-ollama
```

## Health Checks

Verify all endpoints after any restart or upgrade:

```bash
# Version check — confirms the running binary version
pmabry@sheepsoc:~$ curl -s http://localhost:11434/api/version | jq
# Expected: {"version":"0.30.10"}

# Chat endpoint (OpenWebUI / Matrix bot path) — non-streaming
pmabry@sheepsoc:~$ curl -s -X POST http://localhost:11434/api/chat \
    -H "Content-Type: application/json" \
    -d '{"model":"qwen3:14b","messages":[{"role":"user","content":"ping"}],"stream":false}' \
    | jq .done
# Expected: true

# Embed endpoint (RAG embeddings path)
pmabry@sheepsoc:~$ curl -s -X POST http://localhost:11434/api/embed \
    -H "Content-Type: application/json" \
    -d '{"model":"nomic-embed-text","input":"test"}' \
    | jq '.embeddings | length'
# Expected: 1

# Anthropic Messages API (Claude Code path — only available since 0.14.0)
pmabry@sheepsoc:~$ curl -s -X POST http://localhost:11434/v1/messages \
    -H "Content-Type: application/json" \
    -H "x-api-key: ollama" \
    -H "anthropic-version: 2023-06-01" \
    -d '{"model":"qwen3:14b","max_tokens":16,"messages":[{"role":"user","content":"ping"}]}' \
    | jq .type
# Expected: "message"
```

Verify models are intact:

```bash
pmabry@sheepsoc:~$ ollama list
# Should list 14 models. Header line counts as 1, so wc -l should return 15.
```

GPU activity during an active inference request:

```bash
pmabry@sheepsoc:~$ nvidia-smi
# The ollama process should appear with non-zero VRAM usage during a request.
# Idle Ollama holds 0 MB — that is expected behavior.
```

## Upgrade Procedure

!!! danger "Run these steps every time you upgrade Ollama"
    The official installer rewrites the systemd unit to its defaults. Skipping the restore step leaves Ollama bound to `127.0.0.1` as the `ollama` system user — breaking OpenWebUI, RAG embeddings, and Matrix bot access silently.

**Step 1 — Back up before upgrading:**

```bash
pmabry@sheepsoc:~$ sudo cp /usr/local/bin/ollama \
    ~/ollama-backups/ollama-$(ollama --version | awk '{print $NF}')
pmabry@sheepsoc:~$ sudo cp /etc/systemd/system/ollama.service \
    ~/ollama-backups/ollama.service.bak
pmabry@sheepsoc:~$ sudo cp /etc/systemd/system/ollama.service.d/parallel.conf \
    ~/ollama-backups/ollama.service.d/parallel.conf
pmabry@sheepsoc:~$ sudo cp /etc/systemd/system/ollama.service.d/context.conf \
    ~/ollama-backups/ollama.service.d/context.conf
pmabry@sheepsoc:~$ ollama --version > ~/ollama-backups/VERSION-before.txt
```

**Step 2 — Run the upgrade:**

```bash
pmabry@sheepsoc:~$ curl -fsSL https://ollama.com/install.sh | sh
```

**Step 3 — Restore the custom unit immediately after:**

```bash
pmabry@sheepsoc:~$ sudo cp ~/ollama-backups/ollama.service.bak \
    /etc/systemd/system/ollama.service
pmabry@sheepsoc:~$ sudo systemctl daemon-reload
pmabry@sheepsoc:~$ sudo systemctl restart ollama
```

**Step 4 — Verify all four endpoints** using the commands in [Health Checks](#health-checks). Confirm `ollama list` shows all 14 models. Then update the backup files with the new version.

## Known Issues / Gotchas

- **Upgrade clobbers custom unit file** — described in [Upgrade Procedure](#upgrade-procedure) above. This is an active Watchlist entry; see [Known Issues — Watchlist](../known-issues.md#watchlist).
- **NVIDIA driver** — do not update the NVIDIA driver without a plan and rollback. The RTX 5060 Ti requires a modern driver; an incompatible update can take Ollama offline. See [Known Issues — Do Not Touch](../known-issues.md#active-landmines-do-not-touch).
- **VRAM ceiling** — the RTX 5060 Ti has 16 GB VRAM. Models larger than approximately 13–14B parameters (unquantized) will not fit. Check available VRAM before pulling new models.
- **Single-request concurrency (changed 2026-07-02)** — `OLLAMA_NUM_PARALLEL=1` means Ollama now serves one inference request at a time; concurrent callers queue rather than overlap. See [Configuration — Concurrency Limit](#concurrency-limit) and [Known Issues — Watchlist](../known-issues.md#ollama-single-request-concurrency-since-2026-07-02).
- **Default context length resolved (2026-07-02)** — conversations and RAG retrieval are no longer silently capped at 4096 tokens. See [Configuration — Context Length](#context-length).

## See Also

- [OpenWebUI & RAG](openwebui-rag.md) — **depends on** Ollama for LLM inference and RAG embeddings
- [Matrix Bot](matrix-bot.md) — **depends on** Ollama for inference
- [Lab Hub](lab-hub.md) — **depends on** Ollama for local inference; `qwen3` (Phase 2 summaries) and `deepseek-r1:14b` (Phase 3a daily brief) must be present
- [Log Shipping — Filebeat & Logstash](log-shipping.md) — Filebeat collects `ollama.service` journald logs and llama.cpp runner metrics and ships them to Elastic Cloud via the `logs-ollama-otel` ingest pipeline; see the pipeline section for the full field catalog including throughput metrics
- [Services](../services.md) — full service catalog with port and status
- [Known Issues](../known-issues.md) — NVIDIA driver landmine and upgrade history
