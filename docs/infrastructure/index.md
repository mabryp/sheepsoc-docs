# Infrastructure

Infrastructure documentation covers the machine, services, platforms, runbooks, backups, recovery paths, and operational risks that keep sheepsoc running.

Use this section when you are checking system health, changing services, recovering from failure, managing data pipelines, or updating the RAG platform.

## Major Services

- [Services](services.md) — full service catalog, ports, health checks, and web endpoints
- [Topology](topology.md) — network map, host layout, storage, and data-flow diagrams
- [OpenWebUI & RAG](platforms/openwebui-rag.md) — primary AI interface, RAG configuration, and Elasticsearch vector store
- [Elasticsearch & ELSER](platforms/elasticsearch-elser.md) — ELSER sparse-vector search and dual-use index configuration
- [Knowledge Bases](platforms/knowledge-bases.md) — catalog of all OpenWebUI RAG Knowledge Base collections
- [Matrix Bot](platforms/matrix-bot.md) — E2EE Matrix bot bridging Element rooms to OpenWebUI RAG
- [Conda Guide](platforms/conda.md) — Python environment management

## Runbooks

- [Shutdown & Startup](runbooks/shutdown-startup.md) — safe ordered shutdown and post-boot verification
- [OpenWebUI KB Bulk Ingest](runbooks/openwebui-kb-bulk-ingest.md) — bulk-loading documents into Knowledge Base collections
- [Nightly Backups](runbooks/nightly-backups.md) — scheduled RAG knowledge base sync via cron

## Backup & Recovery

- [Config Backup & Encryption](backup-and-recovery/config-backup.md) — age/SOPS encryption and GitHub offsite backup
- [GitHub & RAG Sync](backup-and-recovery/github-rag-sync.md) — automated sync of docs and configs to GitHub

## Operational Reference

- [Known Issues](known-issues.md) — active landmines, watchlist, and history log
- [Future Improvements](future-improvements.md) — planned enhancements not yet implemented
