# The Agile Team

**Purpose:** How sheepsoc is managed — Claude Code as team lead, specialized agents as delegates.

| | |
|---|---|
| Added | 2026-04-22 |
| Tool | Claude Code CLI (`claude`) · Anthropic claude-sonnet-4-6 |

## Overview

Sheepsoc is not managed by clicking through admin panels or running shell commands by hand. It is managed by describing what you want in plain English and letting a team of specialized AI agents work out the details, ask for clarification where needed, and explain every action before taking it.

The tool is **Claude Code** — Anthropic's CLI for the Claude language model, invoked as `claude` from any terminal on the machine. Claude Code acts as the team lead: it reads context about the system from CLAUDE.md files, plans the work, delegates to specialists where appropriate, and synthesizes the results into a clear summary for Phillip.

The specialist agents each own a tightly-defined domain. When Claude Code recognizes that a task falls within a specialist's area, it delegates to that agent rather than attempting the work itself. This keeps each agent focused, reduces errors from context overload, and makes the division of responsibility explicit and auditable.

!!! note "Important"
    Nothing happens on this machine without Phillip knowing about it first. Every agent is explicitly instructed to explain what it plans to do and why before taking any action, never run silent `sudo` commands, prefer reversible changes, and check before installing anything new. This is a non-negotiable rule, not a suggestion.

## The Team

Eight agents make up the team. Claude Code (the CLI itself) is the team lead. The remaining seven are specialized delegates, each invoked via the `/agent` mechanism inside a Claude Code session.

### Claude Code — Team Lead

Claude Code is the entry point for every interaction. It reads the system's CLAUDE.md context files, understands the full picture of what is being asked, breaks down complex requests, and decides which specialist agents to invoke and in what order. It also handles tasks that don't belong to a specific specialist: asking clarifying questions, synthesizing results from multiple agents, and confirming with Phillip before any significant action.

| Key | Value |
|---|---|
| Invoked as | `claude` from any terminal on sheepsoc |
| Context | Reads `CLAUDE.md` files automatically from the working directory and parent directories |
| Example | "Set up a new service for X" — Claude Code assesses the system, plans the work, and delegates to infrastructure-admin, docs-site-manager, and vikunja-scrum-master in sequence |

### infrastructure-admin — System Administration

The infrastructure-admin agent handles anything that touches the OS, services, or hardware. It knows the sheepsoc machine's specific configuration: the systemd service layout, the UFW firewall rules, the conda environments, the storage mounts, the GPU driver constraints. When a task requires root access, infrastructure-admin explains exactly what the privileged command does before running it.

| Key | Value |
|---|---|
| Domain | OS configuration, systemd service management, package installation, SSH, UFW firewall, kernel and driver decisions, storage, networking |
| Example | Installing a new service — creates the systemd unit, opens the correct UFW port for LAN access, confirms the service starts and survives a restart |

### polyglot-dev — Implementation

The polyglot-dev agent writes clean, production-quality code from a specification. It handles Python, JavaScript, and TypeScript. When a new feature or script is designed and the architecture is agreed upon, polyglot-dev receives the spec and implements it. It does not invent requirements — it executes them.

| Key | Value |
|---|---|
| Domain | Python, JavaScript, TypeScript implementation; translating specifications into working code |
| Example | Writing the RAG sync script (`rag_sync.py`) from a spec describing what files to sync, how to authenticate to OpenWebUI, and how to handle encrypted files |

### docs-site-manager — Documentation

The docs-site-manager agent creates and maintains this documentation site. It is invoked automatically after any major system change to ensure the docs reflect the current state of the machine. It reads existing pages before touching anything, matches the site's conventions exactly, and explains every change before making it.

| Key | Value |
|---|---|
| Domain | Documentation pages in `~/repositories/sheepsoc/landing/` (MkDocs), HTML/CSS structure, nav |
| Example | Adding a new service page after infrastructure-admin installs the service |

### vikunja-scrum-master — Kanban Board

The vikunja-scrum-master agent manages the Vikunja task tracker at [http://localhost:3000](http://localhost:3000). It is invoked at the end of multi-step tasks to record what was done, update task status, and log completion notes. A task is not considered done until it is recorded on the board.

| Key | Value |
|---|---|
| Domain | Vikunja kanban board at `http://localhost:3000` — creating tasks, updating status, logging completion notes |
| Example | After a security hardening session completes, marking the corresponding task done and adding notes about what was changed and what remains pending |

### data-wrangler-elastic — Elasticsearch & Data

The data-wrangler-elastic agent handles the data layer: Elasticsearch index design, Logstash and Beats pipeline configuration, document ingestion, and vector search configuration. It understands the sheepsoc Elasticsearch cluster specifically — the current index mappings, the HNSW/cosine vector configuration, and the Filebeat and Metricbeat data streams.

| Key | Value |
|---|---|
| Domain | Elasticsearch index design, Logstash/Beats pipelines, data ingestion, vector search configuration, Kibana |
| Example | Designing an index mapping for a new document collection, troubleshooting why Filebeat is not shipping logs |

### network-engineer — Home Network

The network-engineer agent handles the home network infrastructure: the ASUS RT-AX5400 router (gateway), the OPNsense firewall and DNS resolver, port forwarding rules, DHCP, and VPN configuration.

| Key | Value |
|---|---|
| Domain | ASUS RT-AX5400 router, OPNsense firewall/DNS, port forwarding, DHCP, VPN, `mabry.lan` DNS records |
| Example | Opening a new port on OPNsense, diagnosing why `mabry.lan` DNS resolution is failing |

### Explore — Codebase Search

The Explore agent is a fast codebase search tool. It answers "where is X defined?", "which file handles Y?", and "what calls this function?" questions without requiring a full read-through of the code.

| Key | Value |
|---|---|
| Domain | Searching files by pattern, grepping for symbols, tracing call paths, understanding project structure |
| Example | Finding which file in the matrix-bot repository handles the JWT refresh logic before polyglot-dev adds a new feature |

### Plan — Architecture & Design

The Plan agent handles software architecture and implementation design. It is invoked before writing code when the approach is not obvious — to think through the design, identify edge cases, and agree on a structure before any implementation starts.

| Key | Value |
|---|---|
| Domain | Software architecture, schema design, implementation planning, trade-off analysis |
| Example | Designing the schema for a new Elasticsearch index before any ingestion code is written |

## The Standard Workflow

Every task on sheepsoc follows the same pattern, regardless of complexity. The steps may compress for simple tasks, but they are never skipped entirely.

1. **Assess.** Before anything is touched, the current state is understood. Claude Code reads CLAUDE.md files, checks relevant service status, and reads existing config or code. It never assumes the system is in a known state.
2. **Explain.** Claude Code describes what it plans to do and why. This is not optional. If the plan is wrong or if Phillip has a preference that conflicts with the proposal, this is the moment to surface it — before any changes are made.
3. **Delegate.** Tasks that fall within a specialist's domain are handed off to the appropriate agent. Claude Code provides the agent with the relevant context and a clear description of the expected outcome.
4. **Parallelize.** When multiple tasks are independent of each other, they are run across agents simultaneously.
5. **Log.** After the work completes, vikunja-scrum-master is invoked to record what was done on the Vikunja kanban board.
6. **Document.** After any major system change, docs-site-manager is invoked to update the documentation site so it reflects current reality.

## The "Nothing Happens Without Phillip Knowing" Rule

Every agent that works on sheepsoc is given the same core instruction: *explain before acting, and never make a silent change.*

In practice, this means:

- **Explain before acting.** Before running any command that modifies state, the agent describes what the command does, why it is being run, and what the expected outcome is.
- **No silent sudo.** Every command that requires root access is announced explicitly, along with what it changes on the system.
- **Prefer reversible actions.** Back up files before editing. Comment out lines rather than deleting them. Prefer `systemctl stop` over removing a service entirely.
- **Check before installing.** No new packages, services, or dependencies are installed without Phillip confirming.

This rule exists because this machine is a single-box production environment with no staging equivalent. A bad change can take down Ollama inference, corrupt an Elasticsearch index, or break the bot.

## CLAUDE.md Context Files

Every time Claude Code is invoked, it automatically reads one or more `CLAUDE.md` files to understand the project context before doing anything. Claude Code reads these in a hierarchical order:

- `~/.claude/CLAUDE.md` — global rules that apply to every session on this machine
- `~/CLAUDE.md` — the main sheepsoc system context: hardware, services, conda environments, known issues, and rules for Claude
- `~/repositories/sheepsoc/CLAUDE.md` — project-specific context for the sheepsoc RAG application

Any directory can have its own CLAUDE.md. When Claude Code is run from that directory, it loads the CLAUDE.md for that directory as well as any parent-directory CLAUDE.md files up to the home directory.
