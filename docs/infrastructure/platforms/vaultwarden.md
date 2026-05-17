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

All runtime configuration is driven by environment variables in `.env`. The container image is unmodified — configuration changes never require rebuilding the image.

### Key Files

| File | Purpose |
|---|---|
| `docker-compose.yml` | Container definition — image pin, port binding, volume mount |
| `.env` | Runtime config (mode 600) — holds `ADMIN_TOKEN` and signup flags |
| `data/` | Persistent volume — SQLite vault DB and file attachments |

### Key Environment Variables

| Variable | Description | Current Value |
|---|---|---|
| `ADMIN_TOKEN` | Token required to access `/admin`. Currently plaintext — see [Known Issues](../known-issues.md#vaultwarden-plaintext-admin-token). | `<token>` (see `.env`) |
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
# Confirm the container is running
pmabry@sheepsoc:~$ docker compose -f /home/pmabry/infrastructure/vaultwarden/docker-compose.yml ps

# Confirm the port is bound on loopback
pmabry@sheepsoc:~$ ss -tlnp | grep 8222

# Confirm the service responds over Tailscale (from any enrolled device)
# Open https://sheepsoc-1.tail0f68e4.ts.net:8444/ in a browser — the Bitwarden login page should load.
```

## Day-to-Day Operations

### View Logs

```bash
pmabry@sheepsoc:~$ cd /home/pmabry/infrastructure/vaultwarden
pmabry@sheepsoc:~/infrastructure/vaultwarden$ docker compose logs --tail=200 -f
```

### Change Configuration

All config lives in `.env`. After editing it, the container must be re-created — a plain `restart` does **not** pick up environment variable changes.

```bash
pmabry@sheepsoc:~$ cd /home/pmabry/infrastructure/vaultwarden
pmabry@sheepsoc:~/infrastructure/vaultwarden$ vim .env
# ... make your edits ...
pmabry@sheepsoc:~/infrastructure/vaultwarden$ docker compose up -d --force-recreate
```

!!! warning "Restart vs. recreate"
    `docker compose restart vaultwarden` reloads the process but reads the **existing** environment baked into the running container. It will **not** pick up `.env` changes. Always use `docker compose up -d --force-recreate` when you have edited `.env`.

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

**Option A — Temporary signup window:** Set `SIGNUPS_ALLOWED=true` in `.env`, recreate the container, let the user register, then set it back to `false` and recreate again.

**Option B — Admin panel invite:** Navigate to `https://sheepsoc-1.tail0f68e4.ts.net:8444/admin`, authenticate with the `ADMIN_TOKEN` from `.env`, and use the "Invite User" function. Requires SMTP to be configured for the invitation email to be delivered.

### Rotate the Admin Token

The admin token authenticates access to `/admin`. To rotate it:

1. Optionally generate an argon2-hashed token (preferred — see [Known Issues](../known-issues.md#vaultwarden-plaintext-admin-token)):

```bash
# Generate an argon2 hash (use the hash value, not the plaintext, in .env)
pmabry@sheepsoc:~$ docker run --rm vaultwarden/server /vaultwarden hash
```

2. Update `ADMIN_TOKEN` in `.env`.
3. Recreate the container:

```bash
pmabry@sheepsoc:~/infrastructure/vaultwarden$ docker compose up -d --force-recreate
```

To view the current token value:

```bash
pmabry@sheepsoc:~$ grep ADMIN_TOKEN /home/pmabry/infrastructure/vaultwarden/.env
```

## Backup

!!! warning "Backup not yet implemented"
    No automated backup of the vault is in place as of 2026-05-17. The planned approach is nightly encrypted exports via the `bw` CLI piped to Google Drive via `rclone`, with a weekly USB copy. This is a known gap — the vault is the only copy of all stored credentials. Do not wait long to implement this.

The vault SQLite database is at `/home/pmabry/infrastructure/vaultwarden/data/db.sqlite3`. Attachments are in `/home/pmabry/infrastructure/vaultwarden/data/attachments/`. A manual backup can be done by stopping the container and copying the `data/` directory:

```bash
pmabry@sheepsoc:~/infrastructure/vaultwarden$ docker compose down
pmabry@sheepsoc:~$ cp -a /home/pmabry/infrastructure/vaultwarden/data/ \
    /path/to/backup/vaultwarden-data-$(date +%Y%m%d)/
pmabry@sheepsoc:~/infrastructure/vaultwarden$ docker compose up -d
```

## Known Issues / Gotchas

### Plaintext ADMIN_TOKEN

The `ADMIN_TOKEN` in `.env` is currently a plaintext string. Vaultwarden logs a startup notice about this. The fix is to replace it with an argon2 hash — see [Rotate the Admin Token](#rotate-the-admin-token) above for the command. This is queued as an open item on the Monday SheepSOC board.

See [Known Issues — Vaultwarden plaintext admin token](../known-issues.md#vaultwarden-plaintext-admin-token) for the site-wide tracking entry.

### No SMTP

SMTP is not configured. Consequence: password-reset emails will not be sent. This is intentional for a single-user setup — password reset can be performed via the admin panel at `/admin`. If multi-user operation is ever needed, configure `SMTP_*` variables in `.env`.

### Tailnet-Only Access

Vaultwarden is not reachable from the LAN. A client that is not enrolled in Phillip's tailnet (`tail0f68e4`) — including a fresh browser on the sheepsoc LAN — cannot reach the vault UI or API. Enroll the device in Tailscale first.

## See Also

- [Tailscale](tailscale.md) — the network layer that exposes Vaultwarden
- [Services](../services.md) — Vaultwarden entry in the service catalog
- [Known Issues](../known-issues.md) — plaintext admin token tracking entry
