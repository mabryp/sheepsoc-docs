# Infrastructure

Infrastructure documentation covers the machine, services, platforms, runbooks, backups, recovery paths, and operational risks that keep sheepsoc running.

Use this section when you are checking system health, changing services, recovering from failure, managing data pipelines, or updating the RAG platform.

## Major Services

- [Services](services.md) — full service catalog, ports, health checks, and web endpoints
- [Topology](topology.md) — network map, host layout, storage, and data-flow diagrams
- [Ollama](platforms/ollama.md) — local LLM inference server (RTX 5060 Ti) · 0.30.10 · OpenAI, Anthropic Messages, and native APIs
- [OpenWebUI & RAG](platforms/openwebui-rag.md) — primary AI interface, RAG configuration, and Elasticsearch vector store
- [Elasticsearch & ELSER](platforms/elasticsearch-elser.md) — ELSER sparse-vector search and dual-use index configuration
- [OpenTelemetry Collector](platforms/otelcol-contrib.md) — OTLP telemetry hub · v0.155.0 · receives from OpenWebUI and Claude Code · exports to Elastic Cloud 9.4.0 as structured data streams
- [Log Shipping — Filebeat & Logstash](platforms/log-shipping.md) — Filebeat and Logstash configured to ship to Elastic Cloud 9.4.0 · Ollama journal, system journal, and network device syslog (migrated 2026-06-29)
- [Knowledge Bases](platforms/knowledge-bases.md) — catalog of all OpenWebUI RAG Knowledge Base collections
- [Matrix Bot](platforms/matrix-bot.md) — E2EE Matrix bot bridging Element rooms to OpenWebUI RAG
- [Tailscale](platforms/tailscale.md) — WireGuard mesh VPN for remote access to sheepsoc services
- [Vaultwarden](platforms/vaultwarden.md) — self-hosted password and secrets manager (LastPass replacement)
- [RomM / EmulatorJS](platforms/romm-emulatorjs.md) — self-hosted ROM library manager with integrated in-browser retro emulator
- [Sheepsoc Lab Hub](platforms/lab-hub.md) — read-only "State of the Lab" health dashboard and drift detector
- [Conda Guide](platforms/conda.md) — Python environment management

## Runbooks

- [Shutdown & Startup](runbooks/shutdown-startup.md) — safe ordered shutdown and post-boot verification
- [OpenWebUI KB Bulk Ingest](runbooks/openwebui-kb-bulk-ingest.md) — bulk-loading documents into Knowledge Base collections
- [Nightly Backups](runbooks/nightly-backups.md) — scheduled RAG knowledge base sync via cron
- [Tailscale Operations](runbooks/tailscale-ops.md) — adding/removing tailnet peers, key rotation, full uninstall
- [Samsung TV Network Control](runbooks/wol-samsung-tv.md) — full control of Samsung TV (192.168.50.175, MAC 54:3A:D6:5D:B0:EC) via tv_control.py (volume prioritized, WoL power-on, raw keys, --youtube-search via run_app+send_text); expands prior WoL runbook with install (sheepsoc conda), usage examples, how it works, troubleshooting; prerequisites (Network Remote enabled); integrates 2026-05-30 tests (conda run -n sheepsoc succeeded)

## Backup & Recovery

- [Config Backup & Encryption](backup-and-recovery/config-backup.md) — age/SOPS encryption and GitHub offsite backup
- [GitHub & RAG Sync](backup-and-recovery/github-rag-sync.md) — automated sync of docs and configs to GitHub

## Operational Reference

- [Known Issues](known-issues.md) — active landmines, watchlist, and history log (RAG migration and experiments now documented)
- [Future Improvements](future-improvements.md) — planned enhancements not yet implemented
- Research integration: see [RAG Experiments](../research/rag-experiments.md) for active pipelines tying into OpenWebUI and Elasticsearch platforms
