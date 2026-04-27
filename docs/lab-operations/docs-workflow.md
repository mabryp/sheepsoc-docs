# Docs Workflow

**Purpose:** How this documentation site is built, hosted, and updated — covering the GitHub repository, the self-hosted CI runner, the live serving stack, and the correct update process for both humans and agents.

| Key | Value |
|---|---|
| Repo | https://github.com/mabryp/sheepsoc-docs |
| Local clone | `/home/pmabry/infrastructure/mkdocs-site/` |
| Branch | `main` |
| Live URL | http://192.168.50.100 |
| Updated | 2026-04-27 |

---

## Architecture Overview

The site is a static MkDocs build served by a Python HTTP server. The publish pipeline runs entirely on sheepsoc with no external build service.

```
GitHub (mabryp/sheepsoc-docs)
        |
        |  push to main
        v
Self-hosted Actions Runner
(actions.runner.mabryp-sheepsoc-docs.sheepsoc.service)
        |
        |  mkdocs build
        v
/home/pmabry/infrastructure/mkdocs-site/site/
        |
        |  serves via
        v
sheepsoc-landing.service  (Python HTTP server, port 80)
        |
        v
http://192.168.50.100
```

End-to-end from `git push` to live site: approximately 20 seconds.

---

## GitHub Repository

The documentation source lives in a public GitHub repository under the personal org `mabryp`.

| Property | Value |
|---|---|
| URL | https://github.com/mabryp/sheepsoc-docs |
| Visibility | Public |
| Default branch | `main` |
| All work goes to | `main` — there is no staging or feature branch convention |

The repository contains:

- `mkdocs.yml` — site configuration and full navigation structure
- `docs/` — all Markdown source files

---

## Self-Hosted GitHub Actions Runner

A GitHub Actions runner is installed on sheepsoc and registered to the `mabryp/sheepsoc-docs` repository. It replaces the need for any cloud build service.

| Property | Value |
|---|---|
| Systemd service | `actions.runner.mabryp-sheepsoc-docs.sheepsoc.service` |
| Runner directory | `~/infrastructure/github-runner/` |
| Trigger | Push to `main` |
| Build command | `mkdocs build` |
| Build output | `/home/pmabry/infrastructure/mkdocs-site/site/` |

The runner executes the workflow defined in `.github/workflows/` in the repository. On each push to `main`, it runs `mkdocs build` locally and writes the compiled static site into `site/`. The `sheepsoc-landing.service` Python HTTP server reads from that directory and serves the result on port 80.

### Checking Runner Status

```bash
# Is the runner process healthy?
systemctl status actions.runner.mabryp-sheepsoc-docs.sheepsoc.service

# Did the last push trigger a successful build?
gh run list --repo mabryp/sheepsoc-docs --limit 5
```

Expected output from `gh run list` when healthy:

```
STATUS  TITLE                  WORKFLOW  BRANCH  EVENT  ID          ELAPSED  AGE
✓       your commit message    CI        main    push   12345678    12s      1m
```

If the most recent run shows a failure (`X`), inspect the logs:

```bash
gh run view <run-id> --repo mabryp/sheepsoc-docs --log
```

---

## Live Site — sheepsoc-landing.service

The compiled site is served by a systemd-managed Python HTTP server.

| Property | Value |
|---|---|
| Service | `sheepsoc-landing.service` |
| Serves from | `/home/pmabry/infrastructure/mkdocs-site/site/` |
| Port | 80 |
| Access | http://192.168.50.100 (LAN only — UFW rule applied) |

This service does not need to be restarted after a build. The runner overwrites the `site/` directory in place, and the HTTP server picks up the new files on the next request.

---

## How Humans Update Docs

The recommended human workflow does not require SSH access to sheepsoc.

1. Clone the repository on your laptop:
   ```bash
   git clone https://github.com/mabryp/sheepsoc-docs.git
   cd sheepsoc-docs
   ```
2. Edit or create Markdown files under `docs/` in VS Code (GitHub Copilot is available).
3. If you are adding a new page, also add it to the `nav:` section in `mkdocs.yml`. The nav structure uses two-space indentation and must match the file path exactly:
   ```yaml
   nav:
     - Lab Operations:
       - New Page Title: lab-operations/new-page.md
   ```
4. Stage, commit, and push:
   ```bash
   git add docs/lab-operations/new-page.md mkdocs.yml
   git commit -m "docs: add new-page to lab-operations"
   git push origin main
   ```
5. Wait approximately 20 seconds, then reload http://192.168.50.100 to confirm the change is live.

!!! note "No local preview required"
    You do not need to install MkDocs locally. The runner builds on push. If you want to preview before pushing, you can install MkDocs and run `mkdocs serve` locally — but this is optional, not required.

---

## How Agents Update Docs

Agents (Claude Code, automated scripts) work directly on the sheepsoc filesystem and commit via the local git clone.

1. Edit or create Markdown files in `/home/pmabry/infrastructure/mkdocs-site/docs/`.
2. If adding a new page, add it to the `nav:` section of `/home/pmabry/infrastructure/mkdocs-site/mkdocs.yml`.
3. Stage only the files that were changed:
   ```bash
   git -C /home/pmabry/infrastructure/mkdocs-site add docs/path/to/file.md mkdocs.yml
   ```
4. Commit with a descriptive message:
   ```bash
   git -C /home/pmabry/infrastructure/mkdocs-site commit -m "docs: describe the change"
   ```
5. Push to `main`:
   ```bash
   git -C /home/pmabry/infrastructure/mkdocs-site push origin main
   ```
6. Confirm the runner picked it up:
   ```bash
   gh run list --repo mabryp/sheepsoc-docs --limit 3
   ```

!!! warning "Do not run mkdocs build manually"
    Do not run `mkdocs build` or `mkdocs serve` on sheepsoc. The runner owns the build. Running the build manually risks overwriting a runner-in-progress output or producing a build from a dirty working tree that does not match the committed state.

---

## Adding a New Page — Checklist

- [ ] Markdown file created in the correct subdirectory under `docs/`
- [ ] File starts with an H1 heading, a one-sentence purpose statement, and a metadata table
- [ ] Page added to the `nav:` section of `mkdocs.yml`
- [ ] Changes committed and pushed to `main`
- [ ] `gh run list` confirms the build succeeded
- [ ] Live site at http://192.168.50.100 shows the new page
