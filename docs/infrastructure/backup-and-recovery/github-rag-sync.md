# GitHub & RAG Sync

**Purpose:** Offsite backup via private GitHub repository, secret encryption with age/SOPS, and automatic RAG sync after every commit.

| Key | Value |
|---|---|
| Repo | `mabryp/SheepSOC` (private) — local at `~/repositories/sheepsoc-backup/` |
| Added | 2026-04-22 |
| Status | Active — post-commit hook fires on every commit |
| Depends on | OpenWebUI (port 8080) · conda env `sheepsoc` · age/SOPS |

## 1. Overview

The GitHub & RAG Sync system serves two complementary purposes from a single git repository:

1. **Offsite backup.** A private GitHub repository (`mabryp/SheepSOC`) stores sheepsoc configuration files and documentation. Secret files (e.g. service credentials, bot configuration) are encrypted with **age/SOPS** before being committed, so the repository can be stored on a third-party host without exposing plaintext secrets.
2. **Automatic RAG sync.** A post-commit hook fires on every `git commit` in the local backup repository. It runs a sync script that strips and uploads documentation HTML and configuration files into OpenWebUI's Knowledge Base collections. This keeps the AI assistant's knowledge current without any manual upload step.

The two Knowledge Base collections kept in sync are:

| Collection | Content |
|---|---|
| `sheepsoc-docs` | Documentation pages from `~/repositories/sheepsoc/landing/` — stripped to plain text |
| `sheepsoc-configs` | YAML / JSON / Markdown / conf files from `~/infrastructure/config-mgmt/` |

## 2. System Components

| Component | What It Does | Location |
|---|---|---|
| GitHub repository | Private remote store for configs and docs; SSH deploy key restricts access | `github.com/mabryp/SheepSOC` (remote) · `~/repositories/sheepsoc-backup/` (local clone) |
| age / SOPS encryption | Encrypts secret files before they are staged for commit; decrypts them on the local machine for use by services | age key: `~/.config/sops/age/keys.txt` · Rules: `~/repositories/sheepsoc-backup/.sops.yaml` |
| RAG sync script | Uploads docs and configs to OpenWebUI Knowledge Base collections via the OpenWebUI HTTP API | `~/repositories/sheepsoc-backup/scripts/rag_sync.py` |
| Post-commit hook | Triggers the RAG sync script automatically after every `git commit` | `~/repositories/sheepsoc-backup/.git/hooks/post-commit` |
| Credentials file | Holds OpenWebUI credentials and collection UUIDs; never committed to git | `~/repositories/sheepsoc-backup/.env` (mode 600, gitignored) |

## 3. GitHub Repository and SSH Auth

The local clone at `~/repositories/sheepsoc-backup/` tracks the `master` branch of the private `mabryp/SheepSOC` repository on GitHub.

### SSH Deploy Key Isolation

Rather than using the default GitHub SSH key, a dedicated deploy key is used. This isolates access: even if another SSH key were compromised, it would not grant push access to this repository.

| Key | Value |
|---|---|
| Private key | `~/.ssh/sheepsoc_backup_deploy` |
| SSH alias | `github-sheepsoc-backup` (defined in `~/.ssh/config`) |
| Remote URL | `git@github-sheepsoc-backup:mabryp/SheepSOC.git` |

To verify the SSH connection is working:

```bash
pmabry@sheepsoc:~$ ssh -T git@github-sheepsoc-backup
```

A successful response: `Hi mabryp/SheepSOC! You've successfully authenticated...`

Normal push/pull uses the alias transparently:

```bash
# Push commits to GitHub (deploy key is used automatically via the alias)
pmabry@sheepsoc:~/repositories/sheepsoc-backup$ git push
```

## 4. Secret Encryption with age and SOPS

SOPS (Secrets OPerationS) is a tool that encrypts individual values inside YAML, JSON, and other structured files, leaving the file structure readable while making the values unreadable without the decryption key. age is the underlying cryptographic tool that SOPS uses to perform the encryption.

When you commit `config.yaml` to git with SOPS encryption, the file on disk in the git repository looks like a normal YAML file, but every sensitive value has been replaced with a ciphertext blob. The plaintext original remains on your local machine and is not committed.

### age Keypair

| Key | Value |
|---|---|
| Key file | `~/.config/sops/age/keys.txt` — contains both the public and private age key |
| Public key | `age104ncqpkeeynaly32sgs4k2m3y6p6gaemgntj9nyk7vz28z6wmplsyclnsl` |

!!! danger "Critical"
    `~/.config/sops/age/keys.txt` is the only key that can decrypt the SOPS-encrypted files in the repository. If this file is lost, every encrypted file in the git history becomes permanently unrecoverable. Back this file up offline immediately — see [Offline Backup Requirements](#9-offline-backup-requirements).

### SOPS Rules

The file `~/repositories/sheepsoc-backup/.sops.yaml` defines which files SOPS encrypts and with which key. Two patterns are encrypted automatically:

- `**/config.yaml` — service configuration files (e.g. matrix-bot)
- `**/secrets/**` — any file inside a directory named `secrets/`

### How Encryption and Decryption Work in Practice

Live service config files on disk (e.g. `~/repositories/matrix-bot/config.yaml`) are always kept as **plaintext** — services read them directly. When a copy is added to the backup repository, SOPS encrypts it before the file is staged for commit.

To encrypt a file before staging it:

```bash
# Encrypt a config file in-place. The plaintext is replaced with ciphertext on disk.
# Do this inside the sheepsoc-backup directory, not in the live service directory.
pmabry@sheepsoc:~$ sops --encrypt --in-place ~/repositories/sheepsoc-backup/matrix-bot/config.yaml
```

After encryption, the file contains `ENC[AES256_GCM,...]` markers in place of plaintext values and a `sops:` metadata block at the bottom.

To decrypt a file (e.g. to read or edit a backup copy):

```bash
# Decrypt a file in-place. This restores plaintext values.
pmabry@sheepsoc:~$ sops --decrypt --in-place ~/repositories/sheepsoc-backup/matrix-bot/config.yaml
```

!!! warning "Important"
    Never run `sops --decrypt --in-place` and then `git add` the result. That would commit your plaintext secrets to the repository. Always encrypt before staging, and never stage `config.yaml` or files in `secrets/` without verifying that SOPS has encrypted them first.

To verify a file is encrypted before committing:

```bash
pmabry@sheepsoc:~$ grep -q "ENC\[" myfile.yaml && echo "encrypted" || echo "PLAINTEXT - do not commit"
```

## 5. RAG Sync Script

`~/repositories/sheepsoc-backup/scripts/rag_sync.py` is a Python script that reads documentation and configuration files from local directories on the machine and uploads them to OpenWebUI Knowledge Base collections via OpenWebUI's HTTP API. It runs in the `sheepsoc` conda environment and uses only Python standard library modules plus `requests`.

### What It Syncs

| Collection | UUID | Source Directory | File Types |
|---|---|---|---|
| `sheepsoc-docs` | `2a2ea492-c23c-4053-a9cb-08f171d8c83e` | `~/repositories/sheepsoc/landing/` | `.html` — stripped to plain text before upload |
| `sheepsoc-configs` | `c54b22c3-299b-4db1-930b-668289a16496` | `~/infrastructure/config-mgmt/` | `.yaml`, `.json`, `.md`, `.conf` |

### Authentication

The script authenticates to OpenWebUI at `http://localhost:8080` as `sheepsoc@pmabry.com`. This account must retain admin role in OpenWebUI. Credentials are loaded from `~/repositories/sheepsoc-backup/.env` at runtime and are never hardcoded in the script or committed to git.

### Idempotency

The script is safe to run multiple times against the same collection. Before uploading a new version of a file, it searches the collection for any existing file with the same name and removes it first. This prevents duplicate entries from accumulating in the Knowledge Base across multiple sync runs.

### Encrypted File Detection

Files are skipped automatically if they contain either of the following markers, which indicate SOPS encryption:

- `ENC[` — the start of a SOPS-encrypted value
- `sops:` — the SOPS metadata block

This ensures that ciphertext is never uploaded to the Knowledge Base, where it would be meaningless as RAG context.

### Running the Script Manually

```bash
# Activate the sheepsoc conda environment first
# (conda activate switches your shell into an isolated Python environment)
pmabry@sheepsoc:~$ conda activate sheepsoc

# Run the sync
(sheepsoc) pmabry@sheepsoc:~$ python ~/repositories/sheepsoc-backup/scripts/rag_sync.py
```

## 6. Post-Commit Hook

Git post-commit hooks are shell scripts that run automatically after a successful `git commit` completes. Unlike pre-commit hooks, post-commit hooks cannot abort a commit — they run after the commit is already recorded.

The hook at `~/repositories/sheepsoc-backup/.git/hooks/post-commit` does three things in sequence:

1. Sources `~/repositories/sheepsoc-backup/.env` to load OpenWebUI credentials into the environment.
2. Activates the `sheepsoc` conda environment so that the script's Python dependencies are available.
3. Runs `rag_sync.py`. The hook always exits with code 0 regardless of whether the sync succeeds, so a sync failure does not appear to fail the commit from git's perspective.

!!! note "Hook Not Tracked by Git"
    The post-commit hook lives in `.git/hooks/`, which is not tracked by git. If the repository is re-cloned from GitHub, the hook must be recreated manually. The hook script itself is not included in the repository's committed content.

## 7. Credentials File (.env)

`~/repositories/sheepsoc-backup/.env` stores the runtime credentials used by the RAG sync script. This file is listed in `.gitignore` and must never be committed to the repository.

| Variable | Value / Description |
|---|---|
| `OPENWEBUI_EMAIL` | OpenWebUI account email — default: `sheepsoc@pmabry.com` |
| `OPENWEBUI_PASSWORD` | Password for the OpenWebUI admin account |
| `OPENWEBUI_DOCS_COLLECTION_ID` | UUID of the `sheepsoc-docs` Knowledge Base collection: `2a2ea492-c23c-4053-a9cb-08f171d8c83e` |
| `OPENWEBUI_CONFIGS_COLLECTION_ID` | UUID of the `sheepsoc-configs` Knowledge Base collection: `c54b22c3-299b-4db1-930b-668289a16496` |

File permissions must be set to 600 (owner read/write only):

```bash
pmabry@sheepsoc:~$ chmod 600 ~/repositories/sheepsoc-backup/.env
```

## 8. Key Files and Paths

| Path | Purpose |
|---|---|
| `~/repositories/sheepsoc-backup/` | Git repository root — local clone of `mabryp/SheepSOC` |
| `~/repositories/sheepsoc-backup/.sops.yaml` | SOPS encryption rules — defines which file patterns are encrypted and with which age public key |
| `~/repositories/sheepsoc-backup/.gitignore` | Excludes sensitive files from git: `.env`, `session.json`, `crypto_store/`, `*.key`, `*.pem` |
| `~/repositories/sheepsoc-backup/scripts/rag_sync.py` | RAG sync script — uploads docs and configs to OpenWebUI Knowledge Base |
| `~/repositories/sheepsoc-backup/.git/hooks/post-commit` | Git post-commit hook — triggers `rag_sync.py` on every commit; not tracked by git |
| `~/repositories/sheepsoc-backup/.env` | OpenWebUI credentials and collection UUIDs — mode 600, gitignored |
| `~/.ssh/sheepsoc_backup_deploy` | SSH deploy key private half — grants push access to the GitHub repository. **Back up offline.** |
| `~/.config/sops/age/keys.txt` | age private key — required to decrypt any SOPS-encrypted file. **Back up offline.** |

## 9. Offline Backup Requirements

Two files on this machine are irreplaceable. If either is lost without a backup, the consequences cannot be undone by any software means. Both must be stored in a secure location that is physically separate from this machine (e.g. a password manager with secure notes, or an encrypted USB drive stored offsite).

!!! danger "Back Up Now"
    **`~/.ssh/sheepsoc_backup_deploy`** — the SSH deploy key private half. Losing this file does not destroy data, but it does require generating a new keypair and re-registering the public key as a deploy key on the GitHub repository before pushes will work again.

!!! danger "Back Up Now"
    **`~/.config/sops/age/keys.txt`** — the age private key used by SOPS. Losing this file makes every SOPS-encrypted file in the git repository permanently unrecoverable. There is no recovery path. Every secret stored in the repository would need to be regenerated and re-committed.

## 10. Workflow: Making a Change

This is the normal sequence for adding or updating a file in the backup repository. Follow this order every time to avoid accidentally committing plaintext secrets.

1. **Edit or copy the file into the backup repository.**

    ```bash
    pmabry@sheepsoc:~$ cp ~/repositories/matrix-bot/config.yaml \
        ~/repositories/sheepsoc-backup/matrix-bot/config.yaml
    ```

2. **Encrypt secret files before staging.** If the file matches any pattern in `.sops.yaml` (e.g. `config.yaml` or anything in a `secrets/` directory), encrypt it first. Verify it contains `ENC[` markers before proceeding.

    ```bash
    pmabry@sheepsoc:~$ sops --encrypt --in-place ~/repositories/sheepsoc-backup/matrix-bot/config.yaml
    pmabry@sheepsoc:~$ grep "ENC\[" ~/repositories/sheepsoc-backup/matrix-bot/config.yaml | head -1
    ENC[AES256_GCM,data:...,type:str]
    ```

3. **Stage and commit as normal.** The post-commit hook fires immediately after the commit completes. You will see output from `rag_sync.py` in the terminal as it uploads files to OpenWebUI.

    ```bash
    pmabry@sheepsoc:~/repositories/sheepsoc-backup$ git add matrix-bot/config.yaml
    pmabry@sheepsoc:~/repositories/sheepsoc-backup$ git commit -m "update matrix-bot config backup"
    ```

4. **Push to GitHub.**

    ```bash
    pmabry@sheepsoc:~/repositories/sheepsoc-backup$ git push
    ```

## 11. Troubleshooting

| Symptom | Check / Fix |
|---|---|
| Push fails with `Permission denied (publickey)` | Test the SSH deploy key: `ssh -T git@github-sheepsoc-backup`. If that fails, verify `~/.ssh/sheepsoc_backup_deploy` exists and the SSH config alias `github-sheepsoc-backup` is defined in `~/.ssh/config`. Also confirm the public key is registered as a deploy key on the repository at `github.com/mabryp/SheepSOC/settings/keys`. |
| RAG sync fails with authentication error (HTTP 401) | Check that `OPENWEBUI_PASSWORD` in `~/repositories/sheepsoc-backup/.env` matches the current OpenWebUI admin password. Also verify that `sheepsoc@pmabry.com` still has admin role in OpenWebUI (Admin Panel → Users). |
| SOPS decrypt fails with `no matching keys` | Verify `~/.config/sops/age/keys.txt` exists and contains the private key corresponding to public key `age104ncqpkeeynaly32sgs4k2m3y6p6gaemgntj9nyk7vz28z6wmplsyclnsl`. If the file is missing, restore it from your offline backup. |
| Files not appearing in OpenWebUI Knowledge Base after sync | Check OpenWebUI is running: `systemctl status open-webui.service`. Run the sync script manually: `conda activate sheepsoc && python ~/repositories/sheepsoc-backup/scripts/rag_sync.py`. Also confirm the collection UUIDs in `.env` match the collections visible in OpenWebUI under Workspace → Knowledge. |
| Encrypted file committed as plaintext (accidental) | Remove the file from git history immediately using `git filter-repo` or contact GitHub support to scrub the commit. Treat any exposed credentials as compromised and rotate them. Going forward: always grep for `ENC[` before staging files that match `.sops.yaml` patterns. |
| Post-commit hook not firing | Verify the hook file exists and is executable: `ls -l ~/repositories/sheepsoc-backup/.git/hooks/post-commit`. The file must have execute permission (`chmod +x`). Git hooks are not tracked by git — if the repository was re-cloned, the hook must be recreated manually. |
| Sync script errors with `ModuleNotFoundError: requests` | The script must run inside the `sheepsoc` conda environment. Run: `conda activate sheepsoc && pip install requests`. |
