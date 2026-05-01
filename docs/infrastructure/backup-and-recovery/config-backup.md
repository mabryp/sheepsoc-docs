# Config Backup & Encryption

**Purpose:** Offsite backup of configuration files and secrets using a private GitHub repository and age/SOPS encryption.

| Key | Value |
|---|---|
| Added | 2026-04-22 |
| Repo | `mabryp/SheepSOC` (private) — local at `~/repositories/sheepsoc-backup/` |
| See also | [GitHub & RAG Sync](github-rag-sync.md) — technical reference for the full system |

## The Problem Being Solved

Sheepsoc holds configuration files for every service running on the machine — credentials, API keys, room mappings, network settings, and more. Right now, all of those files exist in exactly one place: on this machine's local disk. If the disk fails, if the OS becomes unbootable, or if a misconfigured command corrupts a critical file, there is no recovery path. Starting over from scratch would mean rebuilding the entire configuration from memory.

The obvious fix is to push everything to a version-controlled offsite location — a git repository on GitHub. But that creates a different problem: many of these config files contain passwords and API tokens. Pushing them to GitHub, even a private repository, is a significant security risk. Private repositories have been accidentally made public. GitHub accounts have been compromised. A secret committed to git history lives there forever, even after the file is deleted.

The backup system solves both requirements at once: offsite backup and security.

## The Solution at a Glance

The backup system consists of three parts working together:

1. **A private GitHub repository** (`mabryp/SheepSOC`) that stores backup copies of configuration files and documentation. Even if someone were to access this repository without authorization, any file containing a secret appears as encrypted ciphertext — meaningless without the private key that lives only on this machine.
2. **age and SOPS encryption** that encrypt sensitive files before they ever leave the machine. The encryption happens locally; only the encrypted output is committed to git. The live files on disk are always kept in plaintext because the services that read them need to read them.
3. **An SSH deploy key** that controls push access to the repository. This key is separate from any other GitHub key on the machine, so a compromise of one does not automatically compromise the other.

## How the Encryption Works

Two tools handle the encryption: **age** and **SOPS**. They work as a pair.

### age — the Lock and Key

**age** is a modern encryption tool. Think of it like a padlock with two separate pieces: a public key and a private key.

- The **public key** is the lock. Anyone can see it. You use it to lock (encrypt) data. Once locked, no one can read the contents without the matching private key — not even you, if you only have the public key.
- The **private key** is the only key that opens the lock. It lives in `~/.config/sops/age/keys.txt`. If this file is lost, anything encrypted with the matching public key is permanently unreadable.

age is a low-level tool. It encrypts whole files, but it does not know anything about file formats like YAML or JSON. That is where SOPS comes in.

### SOPS — the Format-Aware Layer

**SOPS** (Secrets OPerationS) sits on top of age and adds one important capability: it can encrypt individual *values* inside a structured file (YAML, JSON, INI) rather than encrypting the whole file as a blob.

This matters because it preserves readability. After SOPS encrypts a YAML config file, you can still open it in a text editor and see the structure — the section names, the field names, the comments. Only the values that contain secrets are replaced with encrypted ciphertext. For example:

```yaml
# Before SOPS encryption
matrix:
  username: "@sheepsoc-bot:matrix.pmabry.com"
  password: "hunter2"

openwebui:
  email: "sheepsoc@pmabry.com"
  password: "mypassword"
```

```yaml
# After SOPS encryption (values replaced with ciphertext, structure intact)
matrix:
  username: "@sheepsoc-bot:matrix.pmabry.com"
  password: ENC[AES256_GCM,data:abc123...,type:str]

openwebui:
  email: "sheepsoc@pmabry.com"
  password: ENC[AES256_GCM,data:xyz789...,type:str]

sops:
    age:
    -   recipient: age104ncqpkeeynaly32sgs4k2m3y6p6gaemgntj9nyk7vz28z6wmplsyclnsl
        enc: |
            -----BEGIN AGE ENCRYPTED FILE-----
            ...
```

The `sops:` block at the bottom is metadata that SOPS adds to every encrypted file. It records which public key was used, so SOPS knows which private key to look for when decrypting.

### SOPS Rules (.sops.yaml)

The file `~/repositories/sheepsoc-backup/.sops.yaml` tells SOPS which files to encrypt automatically and which public key to use. You do not need to specify the key every time you run SOPS — it reads the rules file and applies the correct key based on the file path.

Two file patterns are covered by the current rules:

- `**/config.yaml` — any file named `config.yaml` anywhere in the repository (this covers the matrix-bot config)
- `**/secrets/**` — any file inside a directory named `secrets/`

## What Is Encrypted vs. Plaintext

Not every file in the backup repository needs to be encrypted. Files that contain no secrets can be committed as plaintext, which makes diffs easier to read and reduces operational complexity.

| File / Pattern | In the Backup Repo | Why |
|---|---|---|
| `matrix-bot/config.yaml` | Encrypted (SOPS) | Contains Matrix homeserver password and OpenWebUI password |
| `**/secrets/**` | Encrypted (SOPS) | By convention — any file in a `secrets/` directory is treated as sensitive |
| Netplan configs, changelogs, docs, scripts | Plaintext | No secrets — safe to store and diff as-is |
| `.env` (credentials file) | Not committed | Listed in `.gitignore` — never enters git history at all |

!!! warning "Important"
    The live copies of config files on disk (e.g. `~/repositories/matrix-bot/config.yaml`) are **always plaintext**. The running service reads them directly and needs them unencrypted. Encryption only happens to the backup copy in `~/repositories/sheepsoc-backup/`, and only immediately before staging for commit.

## The Deploy Key

Instead of authenticating to GitHub using a personal username and password (or even the main SSH key), the backup repository uses a **deploy key** — an SSH keypair generated specifically for this one repository.

Here is why this matters: if your GitHub account were ever compromised, an attacker would need two things to push to this repository — access to your GitHub account *and* the private deploy key file on this machine. Removing the deploy key from GitHub (or rotating it) is enough to cut off access without affecting any other GitHub repositories or keys.

| Key | Value |
|---|---|
| Private key | `~/.ssh/sheepsoc_backup_deploy` — lives only on this machine |
| Public key | Registered as a deploy key on the `mabryp/SheepSOC` GitHub repository |
| SSH alias | `github-sheepsoc-backup` — defined in `~/.ssh/config`, maps to `github.com` using only this key |
| Remote URL | `git@github-sheepsoc-backup:mabryp/SheepSOC.git` |

The SSH config alias means that normal `git push` and `git pull` commands in the backup repo directory automatically use the deploy key, with no extra flags required.

## The Automation

Making a backup should not require remembering a sequence of steps. The system includes a post-commit hook that runs automatically every time a commit is made in the backup repository.

A **post-commit hook** is a shell script that git runs immediately after a `git commit` completes. Unlike a pre-commit hook, it cannot cancel the commit — the commit is already recorded by the time the hook runs. If the hook fails for any reason, the commit still succeeds; git does not treat hook failure as commit failure.

The hook at `~/repositories/sheepsoc-backup/.git/hooks/post-commit` does the following after every commit:

1. Loads credentials from `.env` into the environment.
2. Activates the `sheepsoc` conda environment (which has the `requests` library needed by the sync script).
3. Runs the RAG sync script (`scripts/rag_sync.py`), which uploads the current documentation pages and config files to OpenWebUI's Knowledge Base collections. This keeps the AI assistant's knowledge current automatically.

!!! note "Re-clone Warning"
    Git hooks are not tracked by git — they live in the `.git/` directory, which is never committed. If the backup repository is ever re-cloned from GitHub, the post-commit hook must be recreated manually. Keep a copy of the hook script somewhere in the committed content of the repository for reference.

## The Two Things You Must Never Lose

Two files on this machine are in a special category: losing them causes irreversible consequences that no software can fix. Both must be backed up to a location that is physically separate from this machine — a password manager, an encrypted USB drive stored offsite, or similar.

!!! danger "Critical — Back Up Now"
    **`~/.config/sops/age/keys.txt`** — the age private key. This is the only key that can decrypt any SOPS-encrypted file in the GitHub repository. If this file is lost, every encrypted file in the entire git history becomes permanently unreadable. There is no recovery path. The passwords and credentials stored in those files would need to be regenerated and the entire backup re-encrypted with a new key.

!!! danger "Important — Back Up Recommended"
    **`~/.ssh/sheepsoc_backup_deploy`** — the SSH deploy key private half. Losing this file does not destroy any data, but it does mean you cannot push to the GitHub repository until you generate a new keypair and register the new public key as a deploy key at `github.com/mabryp/SheepSOC/settings/keys`. Less catastrophic than losing the age key, but still disruptive.

## Day-to-Day Workflow

This is the sequence to follow whenever a config file changes and needs to be captured in the backup repository. The key discipline is: *encrypt before staging, verify before committing.*

1. **Copy the updated file into the backup repo.**

    ```bash
    # Example: backing up an updated matrix-bot config
    pmabry@sheepsoc:~$ cp ~/repositories/matrix-bot/config.yaml \
        ~/repositories/sheepsoc-backup/matrix-bot/config.yaml
    ```

2. **Encrypt it if it matches a SOPS pattern.** If the file is a `config.yaml` or lives under a `secrets/` directory, encrypt it in place:

    ```bash
    pmabry@sheepsoc:~$ sops --encrypt --in-place ~/repositories/sheepsoc-backup/matrix-bot/config.yaml
    ```

    Verify the encryption succeeded before staging:

    ```bash
    pmabry@sheepsoc:~$ grep "ENC\[" ~/repositories/sheepsoc-backup/matrix-bot/config.yaml | head -1
    ENC[AES256_GCM,data:...,type:str]

    # If you see ENC[ in the output, the file is encrypted. Safe to stage.
    # If you see plaintext passwords, do NOT stage the file.
    ```

3. **Stage and commit.**

    ```bash
    pmabry@sheepsoc:~/repositories/sheepsoc-backup$ git add matrix-bot/config.yaml
    pmabry@sheepsoc:~/repositories/sheepsoc-backup$ git commit -m "update matrix-bot config backup"
    [master abc1234] update matrix-bot config backup
     1 file changed, 3 insertions(+), 3 deletions(-)
    ```

    After the commit, the post-commit hook fires automatically. You will see the RAG sync script output in the terminal as it uploads files to OpenWebUI.

4. **Push to GitHub.**

    ```bash
    pmabry@sheepsoc:~/repositories/sheepsoc-backup$ git push
    ```

## Recovery Scenario

If this machine is lost or destroyed and needs to be rebuilt from scratch, here is the recovery path:

1. **Restore the age private key** from your offline backup into `~/.config/sops/age/keys.txt` on the new machine. This is the prerequisite for decrypting anything.

2. **Clone the backup repository** from GitHub:

    ```bash
    # Set up the SSH deploy key and config alias first, then:
    pmabry@sheepsoc:~$ git clone git@github-sheepsoc-backup:mabryp/SheepSOC.git \
        ~/repositories/sheepsoc-backup
    ```

3. **Decrypt any encrypted files** you need:

    ```bash
    # Decrypt a file in-place (requires the age key from step 1)
    pmabry@sheepsoc:~$ sops --decrypt --in-place \
        ~/repositories/sheepsoc-backup/matrix-bot/config.yaml
    ```

    The decrypted file now contains the original plaintext config. Copy it to wherever the service expects it.

4. **Restore plaintext files** directly — no decryption needed. Netplan configs, systemd unit files, and other non-secret files are ready to use as-is.

!!! note "Scope of the Backup"
    The backup repository does not store live service data — Elasticsearch indices, OpenWebUI knowledge base files, and conda environments are not included. What it stores is the *configuration* needed to rebuild and reconnect those services. Recovering the configuration is the hard part; the data can be re-ingested once the services are running again.

## See Also

- [GitHub & RAG Sync](github-rag-sync.md) — technical reference for the full backup system: SSH deploy key, SOPS rules, RAG sync script, and post-commit hook
- [Nightly Backups](../runbooks/nightly-backups.md) — the cron job that runs the RAG sync on a nightly schedule; conceptually part of the same backup system
- [Matrix Bot](../platforms/matrix-bot.md) — `config.yaml` for the Matrix bot is one of the files backed up and encrypted by this system
