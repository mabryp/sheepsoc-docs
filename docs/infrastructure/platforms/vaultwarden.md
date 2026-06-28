# Vaultwarden

**Purpose:** Self-hosted password and secrets manager replacing LastPass as Phillip's primary credential store following the 2022 LastPass vault-exfiltration breach.

## Overview

Vaultwarden is an unofficial, API-compatible reimplementation of the Bitwarden server written in Rust. It exposes the full Bitwarden client protocol, so the official Bitwarden browser extension, desktop app, and mobile app all work against it without modification.

| Key | Value |
|---|---|
| Image | `vaultwarden/server:1.36.0-alpine` (pinned) |
| Deployment path | `/home/pmabry/infrastructure/vaultwarden/` |
| Internal binding | `127.0.0.1:8222` (loopback only — not LAN-exposed) |
| Tailnet access URL | `https://sheepsoc-1.tail0f68e4.ts.net:8444/` |
| Admin panel | `https://sheepsoc-1.tail0f68e4.ts.net:8444/admin` |
| TLS | Auto-provisioned by Tailscale ACME via `tailscale serve` |
| Persistent data | `./data/` bind-mount (SQLite DB + attachments) |
| Deployed | 2026-05-17 |

Vaultwarden is accessible **only from devices enrolled in Phillip's tailnet** (`tail0f68e4`). It is not reachable from the LAN and has no UFW rule — access is entirely gated by Tailscale.

This is an ongoing migration. Not all credentials have been moved in from LastPass yet. The parent tracking item is "Personal Secrets Manager" on the Monday SheepSOC board.

## What Is Stored Here

Vaultwarden is the authoritative store for:

- Account credentials (web logins, service accounts)
- API tokens and API keys
- SSH private keys
- GPG keys and TLS material
- Recovery codes and two-factor backup codes
- License keys and software serials

## Dependencies

- [Tailscale](tailscale.md) — Vaultwarden is exposed exclusively via `tailscale serve`. Without Tailscale running, the vault is unreachable from any client.
- Docker — the container runtime. The service runs under `docker compose` in `/home/pmabry/infrastructure/vaultwarden/`.

## Configuration

All runtime configuration is driven by environment variables in `vaultwarden.env`. The container image is unmodified — configuration changes never require rebuilding the image.

### Key Files

| File | Purpose |
|---|---|
| `docker-compose.yml` | Container definition — image pin, port binding, volume mount |
| `vaultwarden.env` | Runtime config (mode 600) — holds `ADMIN_TOKEN` and signup flags |
| `data/` | Persistent volume — SQLite vault DB and file attachments |

### Key Environment Variables

| Variable | Description | Current Value |
|---|---|---|
| `ADMIN_TOKEN` | Token required to access `/admin`. Stored as an argon2id hash (Bitwarden preset: m=65540, t=3, p=4). See [Rotate the Admin Token](#rotate-the-admin-token) for escaping requirements. | `<hash>` (see `vaultwarden.env`) |
| `SIGNUPS_ALLOWED` | Controls public account registration. Set to `false` after Phillip's account was created. | `false` |
| `SMTP_*` | SMTP configuration for password-reset emails. | Not configured — see [No SMTP](#no-smtp) |

### Tailscale Serve Rule

Vaultwarden is published via Tailscale Serve. The rule is persistent (written with `--bg`) and survives reboots.

```bash
pmabry@sheepsoc:~$ tailscale serve --bg --https=8444 http://localhost:8222
```

To verify the rule is active:

```bash
pmabry@sheepsoc:~$ tailscale serve status
```

For full Tailscale Serve management commands, see [Tailscale Operations — Managing Serve Rules](../runbooks/tailscale-ops.md#5-managing-tailscale-serve-rules).

## Health Checks

```bash
# Confirm the container is running and healthy
pmabry@sheepsoc:~$ docker compose -f /home/pmabry/infrastructure/vaultwarden/docker-compose.yml ps

# Confirm the port is bound on loopback
pmabry@sheepsoc:~$ ss -tlnp | grep 8222

# Probe the liveness endpoint directly (the same URL the Docker healthcheck uses)
pmabry@sheepsoc:~$ curl -sf http://127.0.0.1:8222/alive && echo "ok"

# Confirm the service responds over Tailscale (from any enrolled device)
# Open https://sheepsoc-1.tail0f68e4.ts.net:8444/ in a browser — the Bitwarden login page should load.
```

The container's built-in healthcheck polls `http://127.0.0.1:80/alive` every 30 seconds (3-second timeout, 3 retries). When healthy, `docker compose ps` shows `(healthy)` in the status column. See [Healthcheck must use `127.0.0.1`, not `localhost`](#healthcheck-must-use-127001-not-localhost) for the reason the probe uses an explicit IP address.

## Day-to-Day Operations

### View Logs

```bash
pmabry@sheepsoc:~$ cd /home/pmabry/infrastructure/vaultwarden
pmabry@sheepsoc:~/infrastructure/vaultwarden$ docker compose logs --tail=200 -f
```

### Change Configuration

All config lives in `vaultwarden.env`. After editing it, the container must be re-created — a plain `restart` does **not** pick up environment variable changes.

```bash
pmabry@sheepsoc:~$ cd /home/pmabry/infrastructure/vaultwarden
pmabry@sheepsoc:~/infrastructure/vaultwarden$ vim vaultwarden.env
# ... make your edits ...
pmabry@sheepsoc:~/infrastructure/vaultwarden$ docker compose up -d --force-recreate
```

!!! warning "Restart vs. recreate"
    `docker compose restart vaultwarden` reloads the process but reads the **existing** environment baked into the running container. It will **not** pick up `vaultwarden.env` changes. Always use `docker compose up -d --force-recreate` when you have edited `vaultwarden.env`.

### Restart (no config change)

```bash
pmabry@sheepsoc:~$ cd /home/pmabry/infrastructure/vaultwarden
pmabry@sheepsoc:~/infrastructure/vaultwarden$ docker compose restart vaultwarden
```

### Stop and Start

```bash
# Stop (container removed, data persists in ./data/)
pmabry@sheepsoc:~/infrastructure/vaultwarden$ docker compose down

# Start
pmabry@sheepsoc:~/infrastructure/vaultwarden$ docker compose up -d
```

### Update Vaultwarden

1. Edit the image tag in `docker-compose.yml` (e.g., change `1.36.0-alpine` to the new version).
2. Pull and recreate:

```bash
pmabry@sheepsoc:~/infrastructure/vaultwarden$ docker compose pull
pmabry@sheepsoc:~/infrastructure/vaultwarden$ docker compose up -d
```

Data in `./data/` persists across re-creates because it is a bind-mount, not a Docker-managed volume. No migration step is normally needed for minor version upgrades — Vaultwarden applies DB migrations automatically at startup. Check logs after the update to confirm a clean start.

### Enable New User Signups (Admin Task)

Signups are currently disabled (`SIGNUPS_ALLOWED=false`). If you need to add another user:

**Option A — Temporary signup window:** Set `SIGNUPS_ALLOWED=true` in `vaultwarden.env`, recreate the container, let the user register, then set it back to `false` and recreate again.

**Option B — Admin panel invite:** Navigate to `https://sheepsoc-1.tail0f68e4.ts.net:8444/admin`, authenticate with the `ADMIN_TOKEN` from `vaultwarden.env`, and use the "Invite User" function. Requires SMTP to be configured for the invitation email to be delivered.

### Rotate the Admin Token

The admin token authenticates access to `/admin`. The current token is stored as an **argon2id hash** (Bitwarden preset: m=65540, t=3, p=4) in `vaultwarden.env` — not plaintext. Vaultwarden accepts both forms; argon2id is strongly preferred.

!!! danger "Escape every `$` as `$$` in the env file"
    Docker Compose performs variable substitution on values loaded via `env_file:`. An argon2id hash contains multiple literal `$` characters (field separators in the PHC string format). Compose v29.4.1 — the version running on sheepsoc — treats each unescaped `$` as the start of a variable reference and silently drops anything it cannot resolve.

    **Result:** the hash lands in the container truncated and mangled, the admin panel is unreachable, and there is no error message explaining why.

    **Rule:** every `$` in the `ADMIN_TOKEN` value must be written as `$$` in `vaultwarden.env`. This is what the file contains now, and what any future rotation must also do.

    Example — a raw hash looks like:

    ```
    $argon2id$v=19$m=65540,t=3,p=4$<salt>$<hash>
    ```

    What must be written to `vaultwarden.env`:

    ```ini
    ADMIN_TOKEN=$$argon2id$$v=19$$m=65540,t=3,p=4$$<salt>$$<hash>
    ```

    Docker Compose un-escapes `$$` → `$` when injecting the value into the container, so Vaultwarden receives the correct PHC string.

To rotate the token:

1. Generate a new plaintext token with sufficient entropy (48+ bytes):

```python
import secrets
plaintext = secrets.token_urlsafe(48)
print(plaintext)
```

2. Hash it using the argon2id Bitwarden preset and escape the result for the env file:

```python
from argon2 import PasswordHasher

ph = PasswordHasher(time_cost=3, memory_cost=65540, parallelism=4, hash_len=32, salt_len=16)
hashed = ph.hash(plaintext)
escaped = hashed.replace("$", "$$")
print(escaped)   # this is what goes in vaultwarden.env
```

Alternatively, use the Vaultwarden built-in hasher (produces the same preset):

```bash
pmabry@sheepsoc:~$ docker run --rm -it vaultwarden/server /vaultwarden hash --preset bitwarden
# Enter the plaintext when prompted; copy the output hash
# Then manually replace every $ with $$ before writing to vaultwarden.env
```

3. Update `ADMIN_TOKEN` in `vaultwarden.env` (use the `$$`-escaped hash, not the plaintext):

```bash
pmabry@sheepsoc:~/infrastructure/vaultwarden$ vim vaultwarden.env
```

4. Recreate the container:

```bash
pmabry@sheepsoc:~/infrastructure/vaultwarden$ docker compose up -d --force-recreate
```

5. Verify the admin panel loads at `https://sheepsoc-1.tail0f68e4.ts.net:8444/admin` using the **plaintext** token (not the hash — you authenticate with the original secret, Vaultwarden verifies it against the stored hash).

To inspect the current hash value:

```bash
pmabry@sheepsoc:~$ grep ADMIN_TOKEN /home/pmabry/infrastructure/vaultwarden/vaultwarden.env
```

## Backup

Automated nightly backups are in place as of 2026-06-18. The backup script takes a consistent online copy of the vault database (no container stop required), bundles all vault assets, encrypts the result, and writes it to local storage and — once the rclone remote is configured — to Google Drive.

### What Is Backed Up

| Asset | Path |
|---|---|
| Vault database (online copy) | `data/db.sqlite3` — copied via Python's `sqlite3` backup API; WAL-safe, no container shutdown needed |
| File attachments | `data/attachments/` |
| RSA signing key | `data/rsa_key.pem` |
| Container config (if present) | `data/config.json` |

### Backup Script

| Key | Value |
|---|---|
| Script | `/home/pmabry/infrastructure/vaultwarden/backup/vaultwarden-backup.sh` |
| Restore docs | `/home/pmabry/infrastructure/vaultwarden/backup/RESTORE.md` |
| Local destination | `/mnt/data_extra/backups/vaultwarden/` (separate physical disk from the vault) |
| Local retention | 14 copies (oldest removed automatically) |
| Off-site destination | `gdrive:sheepsoc-backups/vaultwarden` via rclone |
| Off-site retention | 90 days |
| Encryption | `age` — encrypted to the existing SOPS/age recipient (`~/.config/sops/age/keys.txt`) |
| Schedule | Daily at 02:30 (`30 2 * * *` user crontab) |

### What the Script Does

1. Creates a consistent online copy of `db.sqlite3` using the Python `sqlite3` backup API. No container stop is required; the API handles any open WAL transaction safely.
2. Runs `PRAGMA integrity_check` on the copy. If this fails, the script aborts and leaves no archive.
3. Bundles `db.sqlite3`, `attachments/`, `rsa_key.pem`, and `config.json` (if present) into a single tar.gz archive.
4. Encrypts the archive with `age` using the SOPS/age recipient public key. The archive is unreadable without the private key.
5. Writes the encrypted archive to `/mnt/data_extra/backups/vaultwarden/` and removes backups older than 14 copies.
6. Uploads the encrypted archive to Google Drive via `rclone remote gdrive:sheepsoc-backups/vaultwarden`. The script skips this step gracefully if the `gdrive:` remote does not exist.

### Cron Entry

```bash
# Vaultwarden nightly backup — 02:30 daily (30 minutes after rag_sync)
30 2 * * * /home/pmabry/infrastructure/vaultwarden/backup/vaultwarden-backup.sh >> /home/pmabry/infrastructure/vaultwarden/backup/backup.log 2>&1
```

### Status as of 2026-06-18

| Component | Status |
|---|---|
| Local encrypted backup | Working — integrity_check verified; full decrypt + restore round-trip confirmed |
| Off-site Google Drive upload | Not yet active — pending rclone install and interactive Google OAuth (`rclone config`) |

!!! warning "Off-site upload pending"
    The Google Drive upload step is configured in the script but currently skipped. Off-site backup is not active until rclone is installed and `rclone config` is completed by Phillip to authorize the `gdrive:` remote.

### Restoring from Backup

See `/home/pmabry/infrastructure/vaultwarden/backup/RESTORE.md` for the full step-by-step restore procedure.

!!! danger "Protect the age private key"
    The age private key (`~/.config/sops/age/keys.txt`) is required to decrypt these backups. It **must** be stored safely off this machine — for example in a separate password manager export, printed to paper, or on a hardware key. Do **not** store the private key in the same Google Drive folder as the encrypted backups. If sheepsoc is lost and the private key is only on it, the backups are permanently unreadable.

## Known Issues / Gotchas

### Healthcheck Must Use `127.0.0.1`, Not `localhost`

The Docker healthcheck polls Vaultwarden's liveness endpoint. The probe **must** use the explicit IPv4 loopback address `127.0.0.1`, not the hostname `localhost`.

**Root cause:** Inside the alpine-based container, `localhost` resolves to the IPv6 loopback `::1` first (standard glibc/musl behaviour). Vaultwarden's Rocket HTTP server binds to `0.0.0.0` (IPv4 only) and does not listen on `::1`. The probe therefore gets "connection refused" even when the service is fully functional, and the container is reported `unhealthy`.

This was the root cause of the container being stuck in `unhealthy` state from initial deployment (2026-05-17) until it was diagnosed and corrected on 2026-06-18, despite the service serving HTTP 200 on `/alive` throughout.

**The fix in `docker-compose.yml`:**

```yaml
healthcheck:
  test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://127.0.0.1:80/alive"]
  interval: 30s
  timeout: 3s
  retries: 3
```

If you see the container reporting `unhealthy` after any configuration change or image update, verify the healthcheck URL uses `127.0.0.1`, not `localhost`.

### Docker Compose `$$` Escaping for argon2 Hashes

Docker Compose performs variable substitution on values loaded via `env_file:`. Any literal `$` in a value — such as the field separators in an argon2id PHC hash string — is treated as a variable reference and dropped. This was observed under Docker Compose v29.4.1.

The `ADMIN_TOKEN` in `vaultwarden.env` is an argon2id hash with every `$` escaped as `$$`. This is required, not optional. If you rotate the token and forget to escape, the container starts normally but the admin panel will reject all authentication attempts with no helpful error.

See [Rotate the Admin Token](#rotate-the-admin-token) for the full procedure and example. This escaping requirement applies to any `env_file:`-loaded value that contains literal `$` characters — it is not specific to Vaultwarden.

See also [Known Issues — Docker Compose v29 env-file variable substitution](../known-issues.md#docker-compose-v29-env-file-variable-substitution).

### No SMTP

SMTP is not configured. Consequence: password-reset emails will not be sent. This is intentional for a single-user setup — password reset can be performed via the admin panel at `/admin`. If multi-user operation is ever needed, configure `SMTP_*` variables in `vaultwarden.env`.

### Tailnet-Only Access

Vaultwarden is not reachable from the LAN. A client that is not enrolled in Phillip's tailnet (`tail0f68e4`) — including a fresh browser on the sheepsoc LAN — cannot reach the vault UI or API. Enroll the device in Tailscale first.

## Runbooks

- [Vaultwarden Bootstrap](../runbooks/vaultwarden-bootstrap.md) — push catalog-defined credentials into Vaultwarden using the Python bootstrap tool
- [LastPass to Vaultwarden Migration](../runbooks/lastpass-migration.md) — bulk import of LastPass web logins via `bw import`

## See Also

- [Tailscale](tailscale.md) — the network layer that exposes Vaultwarden
- [Services](../services.md) — Vaultwarden entry in the service catalog
- [Known Issues](../known-issues.md) — Docker Compose env-file variable substitution footgun; Docker healthcheck `localhost` resolves to IPv6 in alpine containers
