# Sheepsoc Operations Hub

**sheepsoc** is a high-performance homelab server running local LLM inference, Retrieval-Augmented Generation (RAG), observability, and task management — all on a single bare-metal machine on the home LAN.

- **Host:** 192.168.50.100 · `sheepsoc.mabry.lan`
- **OS:** Ubuntu 24.04 LTS · Linux 6.8 · x86\_64
- **Hardware:** Dual Xeon E5-2680v4 · 251 GB RAM · RTX 5060 Ti 16 GB
- **Updated:** 2026-04-25

---

## Service Status

Live service status is monitored by Uptime Kuma. Click the badge below to view current status for all services:

**[View Live Status — http://192.168.50.100:3001/status/sheepsoc](http://192.168.50.100:3001/status/sheepsoc)**

---

## Quick Reference — Services

| Service | URL | Purpose |
|---|---|---|
| Open WebUI | [http://192.168.50.100:8080](http://192.168.50.100:8080) | Primary AI interface — chat + RAG |
| Vikunja | [http://192.168.50.100:3000](http://192.168.50.100:3000) | Kanban / task management |
| OpenProject | [http://192.168.50.100:3001](http://192.168.50.100:3001) | Project management (Docker) |
| Jupyter Notebook | [http://192.168.50.100:8888](http://192.168.50.100:8888) | Python notebooks |
| Kibana | [http://192.168.50.100:5601](http://192.168.50.100:5601) | Log & metrics dashboards |
| Elasticsearch | [http://192.168.50.100:9200](http://192.168.50.100:9200) | REST API — not a UI |
| Ollama | [http://192.168.50.100:11434](http://192.168.50.100:11434) | LLM inference REST API |
| ASUS Router | [http://192.168.50.1](http://192.168.50.1) | Home gateway admin |
| OPNsense | [https://192.168.50.253](https://192.168.50.253) | DNS · firewall (self-signed cert) |

All web interfaces are LAN-only — restricted to 192.168.50.0/24 via UFW.

---

## Documentation Sections

### Infrastructure

- [Topology](infrastructure/topology.md) — network map, host layout, storage, and data-flow diagrams
- [Services](infrastructure/services.md) — every running service, its port, health checks, and configuration references
- [OpenWebUI & RAG](infrastructure/platforms/openwebui-rag.md) — how the RAG stack works and how to use OpenWebUI
- [Elasticsearch & ELSER](infrastructure/platforms/elasticsearch-elser.md) — ELSER sparse-vector search setup and pipeline details
- [Knowledge Bases](infrastructure/platforms/knowledge-bases.md) — catalog of all OpenWebUI RAG Knowledge Base collections
- [Matrix Bot](infrastructure/platforms/matrix-bot.md) — E2EE Matrix bot bridging Element rooms to OpenWebUI RAG
- [Conda Guide](infrastructure/platforms/conda.md) — Python environment management from first principles
- [Shutdown & Startup Runbook](infrastructure/runbooks/shutdown-startup.md) — safe ordered shutdown and post-boot verification
- [OpenWebUI KB Bulk Ingest](infrastructure/runbooks/openwebui-kb-bulk-ingest.md) — SOP for ingesting text collections into OpenWebUI Knowledge Bases
- [Nightly Backups](infrastructure/runbooks/nightly-backups.md) — scheduled RAG knowledge base sync via cron
- [Config Backup & Encryption](infrastructure/backup-and-recovery/config-backup.md) — age and SOPS encryption guide
- [GitHub & RAG Sync](infrastructure/backup-and-recovery/github-rag-sync.md) — offsite backup, age/SOPS encryption, and auto RAG sync
- [Known Issues](infrastructure/known-issues.md) — active landmines, watchlist, and history log
- [Future Improvements](infrastructure/future-improvements.md) — planned enhancements not yet implemented

### Research

- [Research Agenda](research/agenda.md) — strategic research questions for the SheepSOC AI Laboratory
- [Experimental Protocol v1.1](research/rag-001/protocol.md) — retrieval method comparative study design
- [Precision Audit](research/rag-001/precision-audit.md) — citations, metrics, hallucination, and corpus quality audit

### Lab Operations

- [SOPs](lab-operations/sops.md) — standard operating procedures for queries, health, ingest, and more
- [Agent Guide](lab-operations/agent-guide.md) — how AI agents update and maintain this documentation site
- [The Agile Team](lab-operations/agile-team.md) — how sheepsoc is managed using Claude Code and specialized AI agents
