# Sheepsoc Wiki Schema

**Audience:** LLMs and agents maintaining this documentation  
**Purpose:** Tells you what pages exist, how they relate, when to update them, and how to link them  
**Rule:** Read this before writing or editing any page in `docs/`

---

## 1. What This Wiki Is

This is the single source of truth for the sheepsoc homelab. It documents every running
service, the hardware and network topology, operational procedures, and research experiments.

LLMs using this wiki as first-pass context should read the relevant pages rather than
relying on training data or externally cached facts. The wiki is maintained by agents
and updated after every significant infrastructure change.

---

## 2. Page Registry

Every page in `docs/`, with its path relative to `docs/` and its one-line purpose.

### Top Level

| Page | Path | Purpose |
|---|---|---|
| Home | `index.md` | Landing page — service dashboard and quick nav |

### Infrastructure

| Page | Path | Purpose |
|---|---|---|
| Infrastructure Overview | `infrastructure/index.md` | Section index and summary |
| Topology | `infrastructure/topology.md` | Network map, host layout, storage mounts, data-flow diagrams |
| Services | `infrastructure/services.md` | Full service catalog — every unit, port, health checks, web endpoints |
| Known Issues | `infrastructure/known-issues.md` | Active landmines, watchlist, history log |
| Future Improvements | `infrastructure/future-improvements.md` | Planned work and backlog |

### Infrastructure — Platforms

| Page | Path | Purpose |
|---|---|---|
| Ollama | `infrastructure/platforms/ollama.md` | Ollama LLM inference server — version, systemd config, API endpoints, Claude Code integration, upgrade procedure |
| OpenWebUI & RAG | `infrastructure/platforms/openwebui-rag.md` | OpenWebUI chat UI, RAG configuration, ES vector store, ELSER setup |
| Elasticsearch & ELSER | `infrastructure/platforms/elasticsearch-elser.md` | ES cluster config, auth, index mappings, ELSER model |
| OpenTelemetry Collector | `infrastructure/platforms/otelcol-contrib.md` | otelcol-contrib v0.155.0 — OTLP telemetry hub; receives from OpenWebUI and Claude Code; exports to Elastic Cloud 9.4.0 data streams |
| Knowledge Bases | `infrastructure/platforms/knowledge-bases.md` | Catalog of all OpenWebUI RAG Knowledge Base collections |
| Matrix Bot | `infrastructure/platforms/matrix-bot.md` | Matrix bot setup, room config, E2EE, OpenWebUI integration |
| Conda | `infrastructure/platforms/conda.md` | Conda environments, key packages, known gotchas |

### Infrastructure — Runbooks

| Page | Path | Purpose |
|---|---|---|
| Shutdown & Startup | `infrastructure/runbooks/shutdown-startup.md` | Safe shutdown and startup sequence for all services |
| OpenWebUI KB Bulk Ingest | `infrastructure/runbooks/openwebui-kb-bulk-ingest.md` | Bulk-loading documents into OpenWebUI Knowledge Base collections |
| Nightly Backups | `infrastructure/runbooks/nightly-backups.md` | RAG KB sync and nightly backup procedures |
| Vaultwarden Bootstrap | `infrastructure/runbooks/vaultwarden-bootstrap.md` | Push catalog-defined credentials into Vaultwarden using the Python bootstrap tool |
| LastPass to Vaultwarden Migration | `infrastructure/runbooks/lastpass-migration.md` | One-shot bulk import of LastPass web logins into Vaultwarden via `bw import` |
| Samsung TV Network Control | `infrastructure/runbooks/wol-samsung-tv.md` | Full network control of Samsung TV (hostname "Samsung", IP 192.168.50.175 DHCP, MAC 54:3A:D6:5D:B0:EC) via `tv_control.py` script (volume prioritized, WoL power-on via wakeonlan, raw keys, **--youtube-search "QUERY"** now using `tv.open_browser("https://www.youtube.com/results?search_query={quote(query)}")` with `urllib.parse.quote`; completely bypasses YouTube app, on-screen keyboard, cursor navigation, clear, and char-by-char keys after repeated keyboard/cursor/send_text failures and user note on onscreen keyboard). Much simpler/reliable. Updated imports, argparse help, docstring, logic. Re-tested successfully (opens search results directly). Token/WoL/volume unchanged; builds on prior DHCP/WoL/volume success (verified 2026-05-30); includes install, usage, prerequisites (wired Ethernet, WoL + Network Remote/Access Notification), How It Works (browser bypass rationale), troubleshooting, topology links. Wiki is living record of current open_browser method. |

### Infrastructure — Backup & Recovery

| Page | Path | Purpose |
|---|---|---|
| Config Backup & Encryption | `infrastructure/backup-and-recovery/config-backup.md` | SOPS-encrypted config backup to GitHub |
| GitHub & RAG Sync | `infrastructure/backup-and-recovery/github-rag-sync.md` | Automated sync of docs and configs to GitHub |

### Lab Operations

| Page | Path | Purpose |
|---|---|---|
| Lab Operations Overview | `lab-operations/index.md` | Section index |
| SOPs | `lab-operations/sops.md` | Standard operating procedures for common tasks |
| Docs Workflow | `lab-operations/docs-workflow.md` | How to update this site — CI pipeline, runner, commit flow |
| Agent Guide | `lab-operations/agent-guide.md` | How LLMs and agents should maintain this site |
| The Agile Team | `lab-operations/agile-team.md` | AI agent roster, Vikunja board, sprint process |

### Research

| Page | Path | Purpose |
|---|---|---|
| Research Overview | `research/index.md` | Section index |
| Research Agenda | `research/agenda.md` | Active and planned research tracks |
| RAG-001 Protocol | `research/rag-001/protocol.md` | RAG evaluation protocol |
| RAG-001 Precision Audit | `research/rag-001/precision-audit.md` | Precision audit results |
| RAG Experiments | `research/rag-experiments.md` | Active RAG-001/RAG-002 experiments in `~/jupyter/rag_experiments/`: full reproducible pipeline on StackExchange sysadmin corpus (~48k docs in v2/v3), golden dataset (~200 questions with AI+human judgments), embedding comparisons, dense vs ELSER hybrid (RRF) retrieval, evaluation with nDCG@10. Elastic Cloud 9.4.0 (GCP). Resolves migration, replaces old Streamlit in embedding_testing/. Detailed docs in jupyter/rag_experiments/docs/. |

---

## 3. Relationship Vocabulary

Use these labels to describe the nature of links. Include the label in the link text or
in a sentence introducing the link so the relationship is explicit.

| Label | Meaning |
|---|---|
| **depends on** | This service/component cannot function without the target |
| **configured by** | Points to the config file, env var, or systemd unit governing this |
| **monitored by** | What collects metrics or logs from this |
| **stores data in** | Where this service writes persistent data |
| **exposes** | Port or API this service makes available to others |
| **see also** | Related but not a hard dependency |
| **runbook** | Points to a step-by-step procedure for operating this |

---

## 4. Link Rules

### Syntax

Always use relative markdown links. Paths are relative to the current file's location
in `docs/`:

```
[Page Name](../path/to/page.md)
[Page Name](../path/to/page.md#section-anchor)
```

### Required Links — Service and Platform Pages

Every service/platform page must:

1. Have a **Dependencies** section listing and linking every service it depends on
2. Have a **See Also** or **Runbooks** section linking to any runbook that operates it
3. Be linked from the `infrastructure/services.md` service catalog table
4. Be linked from `infrastructure/index.md` if it is a major service

### Required Links — Runbooks

Every runbook must:

1. Link to the service(s) it operates on
2. Have a reciprocal link from the service page's **Runbooks** section
3. Be linked from `lab-operations/sops.md` if it describes a common task

### Required Links — Known Issues

Every active issue must:

1. Link to the affected service or component page
2. Have a reciprocal note on the affected service page (if the issue meaningfully
   changes how that service should be operated)

### Backlinks

When you add a link from page A to page B, ask: does page B need a link back to page A?
If the relationship is meaningful in both directions, add the backlink. Do not add
backlinks that would be confusing or add noise.

### Link Frequency

- Link a page on first mention per section
- Do not repeat the same link more than once within a section
- When mentioning a service by its common name (e.g. "Elasticsearch", "Ollama"),
  link it on first mention in the body text

---

## 5. Page Anatomy

Standard structure for a service or platform page:

```markdown
# [Service Name]

**Purpose:** One sentence on what this does and why it exists on sheepsoc.

## Overview
Brief description. Version, what it is configured for here, key facts.

## Dependencies
- [Dependency A](../path/to/a.md) — why it is required
- [Dependency B](../path/to/b.md) — why it is required

## Configuration
Key config files, env vars, and systemd drop-ins. Actual values where safe (no secrets).

## Health Checks
Commands to verify the service is up and healthy.

## Runbooks
- [Runbook Name](../runbooks/runbook.md) — what it covers

## Known Issues / Gotchas
Landmines specific to this service. Link to [Known Issues](../known-issues.md) for
site-wide issues.

## See Also
Other related pages.
```

Runbooks follow a different pattern — numbered steps, prerequisites up front, expected
outputs at each step. See existing runbooks for examples.

---

## 6. Update Triggers

When something changes on sheepsoc, these pages must be updated:

| Event | Pages to Update |
|---|---|
| New service installed | `services.md`, `topology.md`, platform page (create if needed), `infrastructure/index.md` |
| Service reconfigured | Its platform page; any page referencing its config |
| Service version upgraded | Its platform page (version), `services.md` catalog entry |
| Service removed | `services.md`, `topology.md`, its platform page (mark deprecated or remove) |
| New known issue found | `known-issues.md`; note on affected service page if it changes how it's operated |
| Known issue resolved | `known-issues.md` (move to history); remove or update the note on the service page |
| New runbook created | Runbook page; service page Runbooks section; `lab-operations/sops.md` |
| Network or IP change | `topology.md`; `services.md` web endpoints table |
| Storage change | `topology.md` storage section |
| Security change | `known-issues.md` watchlist; `services.md` if ports changed |
| New conda environment | `infrastructure/platforms/conda.md` |
| Research experiment started | `research/agenda.md`; create experiment page under `research/` |

---

## 7. What Goes Where

| Content Type | Where It Lives |
|---|---|
| What a service is and how it is configured | Platform page under `infrastructure/platforms/` |
| Step-by-step procedure for operating a service | Runbook under `infrastructure/runbooks/` |
| Quick-reference commands for common tasks | `lab-operations/sops.md` |
| Port / status / health check reference | `infrastructure/services.md` |
| Something broke and why | `infrastructure/known-issues.md` |
| Planned future work | `infrastructure/future-improvements.md` |
| Rules for how LLMs behave on this site | `lab-operations/agent-guide.md` (behavioral) or this file (structural) |
| A research experiment | `research/` subtree |

---

## 8. Writing Conventions

- **Headers:** `#` for page title, `##` for major sections, `###` for subsections
- **Code blocks:** always include the language tag (` ```bash `, ` ```ini `, ` ```python `)
- **Admonitions:** `!!! danger` for do-not-touch items, `!!! warning` for handle-with-care,
  `!!! note` for gotchas and clarifications
- **Tables:** use for catalogs, key-value pairs, and comparison data
- **Command prompts:** always show `pmabry@sheepsoc:~$` so the shell context is clear
- **Secrets:** never write actual passwords, tokens, or API keys — use `<password>`,
  `<token>` as placeholders
- **Dates:** ISO format — `2026-04-21`
- **Page names in prose:** match the exact page title and link it on first mention
