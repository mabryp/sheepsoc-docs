# Nightly Backups

**Purpose:** Scheduled RAG knowledge base sync — keeps OpenWebUI's AI knowledge current with the latest documentation and configuration files.

| Key | Value |
|---|---|
| Script | `~/repositories/sheepsoc-backup/scripts/rag_sync.py` |
| Schedule | Daily at 02:00 via user crontab |
| Added | 2026-04-24 |

!!! note "Important Distinction"
    The "nightly backup" on sheepsoc is **not a traditional filesystem backup**. It is a **RAG knowledge base sync** — it strips, processes, and uploads documentation and configuration files into OpenWebUI's Knowledge Base collections so that the AI assistant always has access to up-to-date system documentation. For offsite config backup via GitHub and SOPS encryption, see [GitHub & RAG Sync](../backup-and-recovery/github-rag-sync.md).

## 1. Overview

OpenWebUI's AI assistant answers questions using a Retrieval-Augmented Generation (RAG) pipeline: it retrieves relevant chunks from a Knowledge Base and injects them as context into the LLM prompt. The quality of those answers depends entirely on the Knowledge Base staying current with the actual state of the system.

The nightly sync solves this automatically. Once per day, a cron job runs `rag_sync.py`, which reads the documentation site and configuration files from disk, strips them to plain text, and uploads them into two OpenWebUI Knowledge Base collections. By 02:01 each morning, the AI assistant's knowledge is synchronized with whatever was on disk at 02:00.

The sync script also fires automatically on every `git commit` in the `sheepsoc-backup` repository via a post-commit hook, keeping the Knowledge Base current whenever a manual backup is committed during the day. The nightly cron job is the safety net that catches any changes that happened without a commit.

## 2. What Gets Synced

Two Knowledge Base collections are maintained. Each collection maps to a specific set of source directories and file types.

| Source | File Types | OpenWebUI Collection | Processing |
|---|---|---|---|
| `~/repositories/sheepsoc/landing/` (recursive) | `.html`, `.htm` | `sheepsoc-docs` | HTML stripped to plain text — nav, header, footer, script, and style elements removed before upload |
| `~/infrastructure/config-mgmt/` (recursive) | `.yaml`, `.yml`, `.conf`, `.json`, `.md` | `sheepsoc-configs` | Uploaded as-is (plain text) |
| `~/CLAUDE.md` | `.md` | `sheepsoc-configs` | Uploaded as-is |
| `~/repositories/sheepsoc/CLAUDE.md` | `.md` | `sheepsoc-configs` | Uploaded as-is |

!!! note "Encrypted File Handling"
    SOPS-encrypted files are detected automatically and skipped. A file is considered encrypted if it contains either `ENC[` (a SOPS-encrypted value marker) or `sops:` (the SOPS metadata block). Ciphertext is never uploaded to the Knowledge Base, where it would be useless as AI context.

## 3. Cron Schedule

The sync runs as a user cron job in `pmabry`'s crontab. To view it:

```bash
pmabry@sheepsoc:~$ crontab -l
```

The relevant entry:

```bash
# RAG knowledge base sync — runs at 02:00 every day
0 2 * * * cd /home/pmabry/repositories/sheepsoc-backup && set -a && . .env && set +a && . /home/pmabry/infrastructure/miniconda3/etc/profile.d/conda.sh && conda activate sheepsoc && python scripts/rag_sync.py >> /home/pmabry/repositories/sheepsoc-backup/rag_sync_cron.log 2>&1
```

Breaking down what the cron command does in sequence:

1. `cd /home/pmabry/repositories/sheepsoc-backup` — changes to the repo root so that relative paths in the script resolve correctly.
2. `set -a && . .env && set +a` — sources the `.env` file and exports all its variables into the environment. `set -a` makes every variable automatically exported; `set +a` turns that off again afterwards.
3. `. /home/pmabry/infrastructure/miniconda3/etc/profile.d/conda.sh` — initializes the conda shell functions. This is necessary because cron does not run `~/.bashrc`, so conda is not available by default.
4. `conda activate sheepsoc` — activates the `sheepsoc` conda environment (Python 3.12) where the script's dependencies live.
5. `python scripts/rag_sync.py` — runs the sync script.
6. `>> .../rag_sync_cron.log 2>&1` — appends all output (stdout and stderr) to the log file.

!!! note "Conda in Cron"
    Cron runs in a minimal shell that does not source your `~/.bashrc` or `~/.bash_profile`. This means conda is not initialized. The cron job works around this by sourcing the conda initialization script directly using its absolute path before calling `conda activate`. This is the correct pattern for any cron job that needs a conda environment.

## 4. Environment and Prerequisites

Three services must be running when the cron job fires at 02:00. If any are down, the corresponding sync operations will fail and the failure will be logged.

| Requirement | Details | Why It Is Needed |
|---|---|---|
| Conda environment `sheepsoc` | Python 3.12 · must have `requests` installed | The script's runtime environment |
| `open-webui.service` | Port 8080 | The script authenticates to OpenWebUI and uses its HTTP API to upload files and manage collections |
| `elasticsearch.service` | Port 9200 | OpenWebUI uses Elasticsearch as its vector store — file uploads will fail if ES is down |
| `ollama.service` | Port 11434 · `nomic-embed-text` model loaded | OpenWebUI uses Ollama to generate embeddings for uploaded chunks |

### Credentials File (.env)

The script reads its credentials from `~/repositories/sheepsoc-backup/.env` at runtime. This file must exist, have mode 600, and contain current values for the following variables:

| Variable | Description |
|---|---|
| `OPENWEBUI_PASSWORD` | Password for the OpenWebUI admin account (`sheepsoc@pmabry.com`) |
| `OPENWEBUI_DOCS_COLLECTION_ID` | UUID of the `sheepsoc-docs` Knowledge Base collection |
| `OPENWEBUI_CONFIGS_COLLECTION_ID` | UUID of the `sheepsoc-configs` Knowledge Base collection |

```bash
# Verify the file exists and has correct permissions
pmabry@sheepsoc:~$ ls -la ~/repositories/sheepsoc-backup/.env
-rw------- 1 pmabry pmabry ... /home/pmabry/repositories/sheepsoc-backup/.env

# Set correct permissions if needed
pmabry@sheepsoc:~$ chmod 600 ~/repositories/sheepsoc-backup/.env
```

## 5. Script Behavior

The script is designed to be safe to run repeatedly. Each run is idempotent: if a file already exists in the Knowledge Base collection with the same name, the old version is removed first and the new version is inserted. This prevents duplicate entries from accumulating across multiple sync runs.

### Processing Steps per File

1. Read the source file from disk.
2. Check for SOPS encryption markers — if found, skip the file entirely.
3. For HTML files: strip navigation, header, footer, script, and style elements to produce clean plain text. This makes the uploaded content more useful as RAG context, since the AI does not need to parse HTML tags.
4. Search the target collection for any existing file with the same name and remove it.
5. Upload the processed content to OpenWebUI via the `POST /api/v1/files/` endpoint.
6. Poll `GET /api/v1/files/{id}` until the file's `data.status` field changes from `pending` — this waits for OpenWebUI to finish chunking and embedding the file before adding it to the collection.
7. Add the file to the target collection.

### Output Format

A healthy run produces lines like:

```
[docs] services.html ... ok
[docs] sheepsoc-rag.html ... ok
[docs] topology.html ... ok
...
[docs] synced ok=12 fail=0
[configs] CLAUDE.md ... ok
[configs] elasticsearch.yml ... ok
...
[configs] synced ok=8 fail=0
```

A healthy run ends with both collection summaries showing `fail=0`.

## 6. Log File

All output from cron runs is appended to:

| Key | Value |
|---|---|
| Path | `/home/pmabry/repositories/sheepsoc-backup/rag_sync_cron.log` |
| Rotation | None — the file grows unbounded. Monitor for disk growth. |
| Mode | Appended by cron; overwritten on manual runs if you redirect stdout |

!!! warning "Watch"
    The log file has no rotation configured. On a system that syncs many large files daily, it will grow indefinitely. Check periodically and truncate if needed: `truncate -s 0 ~/repositories/sheepsoc-backup/rag_sync_cron.log`

View the most recent run:

```bash
# Tail the last 50 lines of the cron log
pmabry@sheepsoc:~$ tail -50 ~/repositories/sheepsoc-backup/rag_sync_cron.log

# Follow the log in real time during a manual run
pmabry@sheepsoc:~$ tail -f ~/repositories/sheepsoc-backup/rag_sync_cron.log
```

## 7. Running Manually

Run the sync at any time outside the cron schedule. This is useful after making documentation edits that you want immediately available in the AI assistant, or when diagnosing a sync failure.

```bash
# Step 1: activate the sheepsoc conda environment
# (this switches your shell into the isolated Python 3.12 environment the script needs)
pmabry@sheepsoc:~$ conda activate sheepsoc

# Step 2: load credentials from .env
(sheepsoc) pmabry@sheepsoc:~$ set -a && source ~/repositories/sheepsoc-backup/.env && set +a

# Step 3: run the sync from the repo root
(sheepsoc) pmabry@sheepsoc:~$ cd ~/repositories/sheepsoc-backup
(sheepsoc) pmabry@sheepsoc:~/repositories/sheepsoc-backup$ python scripts/rag_sync.py
```

Output goes directly to your terminal. Look for `fail=0` on both collection summary lines to confirm a clean run.

## 8. Other Triggers

The nightly cron job is not the only way `rag_sync.py` is invoked. The same script also fires automatically on every `git commit` in the `sheepsoc-backup` repository.

| Trigger | Details |
|---|---|
| Post-commit hook | `~/repositories/sheepsoc-backup/.git/hooks/post-commit` — fires after every successful `git commit` in the backup repo. Always exits with code 0 so a sync failure does not appear to fail the commit. |
| Nightly cron | 02:00 daily — catches changes that happened without a commit during the day. |
| Manual run | On demand — see [Running Manually](#7-running-manually) above. |

!!! note "Hook Not Tracked by Git"
    The post-commit hook lives in `.git/hooks/`, which git does not track. If the `sheepsoc-backup` repository is ever re-cloned, the hook must be recreated manually. See [GitHub & RAG Sync — Post-Commit Hook](../backup-and-recovery/github-rag-sync.md#6-post-commit-hook) for the hook content.

## 9. Troubleshooting

| Symptom in Log | Likely Cause | Fix |
|---|---|---|
| `OpenWebUI login failed` | OpenWebUI is down, or credentials in `.env` are stale | Check service: `systemctl status open-webui.service`. If up, verify `OPENWEBUI_PASSWORD` in `.env` matches the current OpenWebUI admin account password. Also confirm `sheepsoc@pmabry.com` still has admin role in OpenWebUI (Admin Panel → Users). |
| `[fail]` lines during upload | Elasticsearch is down, or OpenWebUI cannot reach it | Check ES: `curl -s -u elastic:<password> http://localhost:9200/_cluster/health | python3 -m json.tool`. Check OpenWebUI service logs: `journalctl -u open-webui.service -n 50`. |
| `WARN: could not activate sheepsoc conda env; skipping` | Conda initialization failed in the cron environment | Verify that `/home/pmabry/infrastructure/miniconda3/etc/profile.d/conda.sh` exists. If the conda installation was moved, update the absolute path in the crontab entry (`crontab -e`). |
| `ModuleNotFoundError: requests` | The `sheepsoc` conda environment is missing the `requests` package | `conda activate sheepsoc && pip install requests` |
| Sync completes but Knowledge Base is not updating in OpenWebUI | Collection UUIDs in `.env` do not match the actual collections | In OpenWebUI, go to Workspace → Knowledge and note the URL of each collection — the UUID is the last segment. Compare against `OPENWEBUI_DOCS_COLLECTION_ID` and `OPENWEBUI_CONFIGS_COLLECTION_ID` in `.env`. |
| Log file is empty or missing entries from last night | Cron did not run, or the job definition was lost | Verify the cron entry is still present: `crontab -l`. If the line is missing, re-add it. Check cron daemon: `systemctl status cron.service`. |
| Ollama embedding errors during upload | Ollama is down or `nomic-embed-text` model is not loaded | `systemctl status ollama.service` and `ollama list`. If `nomic-embed-text:latest` is missing, pull it: `ollama pull nomic-embed-text`. |

## See Also

- [OpenWebUI & RAG](../platforms/openwebui-rag.md) — the sync uploads files into OpenWebUI Knowledge Base collections via the OpenWebUI HTTP API
- [Knowledge Bases](../platforms/knowledge-bases.md) — catalog of the Sheepsoc System Docs and Sheepsoc System Config KBs that this runbook manages
- [GitHub & RAG Sync](../backup-and-recovery/github-rag-sync.md) — the companion offsite backup system; the post-commit hook runs the same sync script
- [Conda Guide](../platforms/conda.md) — the `sheepsoc` conda environment required by the cron job; cron requires special initialization
