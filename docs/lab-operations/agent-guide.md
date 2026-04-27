# Agent Guide

**Audience:** Claude Code, AI agents, and automated scripts maintaining this site.

**Purpose:** How to update, extend, and maintain the sheepsoc documentation system.

| Key | Value |
|---|---|
| Updated | 2026-04-24 |

!!! note "Primary Rule"
    **Nothing happens without Phillip knowing.** All changes to this site should be reviewed by Phillip before deployment. Agents may prepare changes but must not push to production without explicit approval unless operating under a standing permission grant.

## File Structure

The site is a set of static HTML files served from the sheepsoc host. The MkDocs-built version lives at `/home/pmabry/infrastructure/mkdocs-site/`.

```
~/infrastructure/mkdocs-site/
├── mkdocs.yml                        # MkDocs configuration and nav structure
└── docs/
    ├── index.md                      # Homepage — links dashboard and service quick reference
    ├── infrastructure/
    │   ├── topology.md               # network and hardware topology
    │   ├── services.md               # all running services reference
    │   ├── known-issues.md           # active issues, watchlist, history
    │   ├── future-improvements.md    # planned improvements
    │   ├── platforms/                # OpenWebUI, Elasticsearch, KBs, Matrix, Conda
    │   ├── runbooks/                 # startup, ingest, and backup runbooks
    │   └── backup-and-recovery/      # config backup, encryption, and GitHub sync
    ├── research/
    │   ├── agenda.md                 # strategic research agenda
    │   └── rag-001/                  # RAG-001 protocol and audits
    └── lab-operations/
        ├── agent-guide.md            # this file
        ├── agile-team.md             # the agile team and scrum process
        └── sops.md                   # standard operating procedures
```

The original HTML site remains at `~/repositories/sheepsoc/landing/` and is the authoritative source until the MkDocs migration is fully deployed.

## Navigation Structure

The MkDocs nav is defined in `mkdocs.yml`. When adding a new page:

1. Create the Markdown file in the appropriate subdirectory under `docs/`.
2. Add the page to the `nav:` section of `mkdocs.yml`.
3. Run `mkdocs build` to verify the page builds without errors.

## Adding a New Documentation Page

1. Create a new `.md` file in the appropriate subdirectory under `docs/`.
2. Begin with an H1 heading, a one-sentence purpose statement, and a metadata table.
3. Use the existing pages as templates for structure and tone — `infrastructure/services.md` and `infrastructure/platforms/openwebui-rag.md` cover the widest range of component types.
4. Add the page to `mkdocs.yml` under the `nav:` key.
5. Run `mkdocs build` to confirm there are no broken links or syntax errors.

## Content Component Reference

### Admonitions

MkDocs Material uses admonitions instead of the HTML `.callout` divs used in the original site:

```markdown
!!! note "Optional title"
    Note body text.

!!! warning "Optional title"
    Warning body text.

!!! danger "Optional title"
    Danger body text.
```

Equivalences:

| Original HTML class | MkDocs admonition |
|---|---|
| `callout note` | `!!! note` |
| `callout warn` | `!!! warning` |
| `callout danger` | `!!! danger` |

### Code Blocks

Use fenced code blocks with a language identifier:

````markdown
```bash
pmabry@sheepsoc:~$ systemctl status open-webui.service
```

```yaml
vector_db: elasticsearch
elasticsearch_url: http://127.0.0.1:9200
```
````

Language tags used in this site: `bash`, `yaml`, `json`, `python`, `text` (for plain output), `markdown`.

### Tables

Standard Markdown tables with a header separator row:

```markdown
| Column A | Column B | Column C |
|---|---|---|
| value 1 | value 2 | value 3 |
| value 4 | value 5 | value 6 |
```

### Metadata Tables

Each page begins with a metadata table immediately below the H1 heading, before the first H2. Use two-column format:

```markdown
| Key | Value |
|---|---|
| Status | Active |
| Port | 8080 |
| Added | 2026-04-20 |
```

## Updating the Links Dashboard (index.md)

The homepage at `docs/index.md` contains a service quick reference table. To add or update a service:

1. Read `docs/index.md`.
2. Locate the relevant table.
3. Add or update the row in Markdown table format.
4. Update the "last updated" date if the page has one.

## Conventions and Rules

| Rule | Detail |
|---|---|
| Explain before acting | Describe what you are about to change and why before making any edits. |
| Read first | Always read the target file before editing it. |
| Preserve tone | Match the formal-but-approachable tone of surrounding content. Never document a broken service as operational. |
| No placeholder text | Do not leave `TODO`, `TBD`, or `Lorem ipsum` in any file. |
| Code blocks require language tags | Every fenced code block must have a language identifier. |
| Admonitions not callouts | Use `!!! note/warning/danger` — not HTML `.callout` divs. |
| MicroK8s is STOPPED | Document it as stopped. Do not describe any k8s service as running. |
| LAN only | This site contains internal IPs and service details. Do not expose it to the public internet. |
| Notify scrum master | After every documentation change, invoke the vikunja-scrum-master agent to log the change on the kanban board. A task is not complete until it is logged. |

## Quick Task Reference

| Task | How to Do It |
|---|---|
| Add a new doc page | Create `.md` in the appropriate `docs/` subdirectory, add to `mkdocs.yml` nav, run `mkdocs build` |
| Update an existing page | Read the file, make the edit, verify the section reads correctly, notify scrum master |
| Add a service to the homepage | Edit `docs/index.md` — update the service quick reference table |
| Change a known issue | Edit `docs/infrastructure/known-issues.md` — add to history or update watchlist |
| Add a future improvement | Edit `docs/infrastructure/future-improvements.md` — follow the existing entry structure |
| Rebuild the site | `cd ~/infrastructure/mkdocs-site && conda activate sheepsoc && mkdocs build` |
| Preview the site locally | `cd ~/infrastructure/mkdocs-site && conda activate sheepsoc && mkdocs serve --dev-addr=0.0.0.0:8001` |

## Design Conventions

The MkDocs Material theme handles visual styling. Key configuration in `mkdocs.yml`:

- **Theme**: Material with `slate` dark palette and `default` light palette, togglable
- **Extensions**: superfences (nested code blocks), highlight (code syntax), tabbed (tab groups), details (collapsible sections), inlinehilite
- **Search**: Built-in via Material theme
- **Navigation**: Instant loading, sticky navigation tabs, table of contents on the right

Do not add custom CSS overrides unless absolutely necessary. Let Material handle the visual presentation.

## Service Restart After Docs Site Rebuild

The sheepsoc documentation site is served by `sheepsoc-landing.service`, a Python `http.server` instance on port 80 pointing at the built `site/` directory. After running `mkdocs build`:

```bash
# Verify the build output exists
pmabry@sheepsoc:~$ ls ~/infrastructure/mkdocs-site/site/

# Restart the web server to serve the updated build
pmabry@sheepsoc:~$ sudo systemctl restart sheepsoc-landing.service

# Verify the service is running
pmabry@sheepsoc:~$ systemctl status sheepsoc-landing.service
```
