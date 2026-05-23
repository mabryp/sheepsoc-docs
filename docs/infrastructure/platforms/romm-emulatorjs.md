# RomM / EmulatorJS

**Purpose:** Self-hosted ROM library manager with an integrated in-browser retro game emulator, running on sheepsoc as a Docker Compose stack.

## Overview

RomM (version 4.8.1 at install, 2026-05-23) is an actively maintained, open-source ROM library manager. It bundles EmulatorJS as its in-browser player, making it the current recommended path to running EmulatorJS on a self-hosted server.

!!! note "Why RomM, not the standalone EmulatorJS image"
    The standalone `linuxserver/emulatorjs` Docker image is officially deprecated by both LinuxServer.io and the EmulatorJS upstream project. RomM is the actively maintained replacement and bundles EmulatorJS as its in-browser player. Installing RomM is the modern path to running EmulatorJS.

The stack runs two containers managed by Docker Compose:

| Container | Image | Role |
|---|---|---|
| `romm` | `rommapp/romm:latest` (4.8.1 at install) | Web frontend, ROM manager, EmulatorJS player |
| `romm-db` | `mariadb:latest` | Relational database backend |

## Dependencies

- [Docker Compose](../services.md) — the stack is managed via `docker compose` in `/mnt/ssd_working/emulatorjs/`
- MariaDB (`romm-db` container) — RomM requires the database to be healthy before starting

## Configuration

### Stack Location

| Item | Path |
|---|---|
| Stack root | `/mnt/ssd_working/emulatorjs/` |
| Compose file | `/mnt/ssd_working/emulatorjs/docker-compose.yml` |
| Secrets / env | `/mnt/ssd_working/emulatorjs/.env` (mode 600, not in git) |

### Ports

| Host port | Container port | Purpose |
|---|---|---|
| `3080` | `8080` | RomM web frontend · bound `0.0.0.0:3080` · LAN-only via UFW |
| (none exposed) | `3306` | MariaDB — container network only, not exposed to host |

### Firewall Rule

```
ALLOW IN from 192.168.50.0/24 to any port 3080 proto tcp
Comment: RomM/EmulatorJS LAN
```

Applied via:

```bash
pmabry@sheepsoc:~$ sudo ufw allow from 192.168.50.0/24 to any port 3080 proto tcp comment "RomM/EmulatorJS LAN"
```

### Volume Layout

| Host path | Container path | Purpose |
|---|---|---|
| `./library/roms/<platform>/` | `/romm/library/roms/<platform>/` | ROM files, organized by platform subdirectory |
| `./library/bios/<platform>/` | `/romm/library/bios/<platform>/` | BIOS files, organized by platform |
| `./assets/` | `/romm/assets/` | Save states, in-game saves, screenshots |
| `./config/` | `/romm/config/` | Optional `config.yml` for RomM settings |
| `./mariadb/` | `/var/lib/mysql` | MariaDB data directory |
| `romm-resources` (named) | `/romm/resources/` | Internal resource cache (managed by Docker) |
| `romm-redis` (named) | `/romm/redis/` | Internal Redis cache (managed by Docker) |

All host paths under `./` are relative to the stack root `/mnt/ssd_working/emulatorjs/`.

### Secrets

Secrets are stored in `/mnt/ssd_working/emulatorjs/.env` (mode 600, not committed to git). The file contains database credentials and the RomM application secret key. Do not write actual values into this page — use `<placeholder>` form when referencing them.

!!! warning "Docker Compose `$$` escaping"
    Docker Compose v29 performs `$variable` substitution on values loaded via `env_file:`. Any literal `$` in a value must be escaped as `$$` in the `.env` file so Compose passes the correct value into the container. This affects any PHC-format string (e.g., Argon2id hashes). See [Known Issues — Docker Compose v29 env-file variable substitution](../known-issues.md#docker-compose-v29-env-file-variable-substitution).

## Health Checks

### Is the stack running?

```bash
pmabry@sheepsoc:~$ docker ps --filter name=romm
# Expect two containers: romm and romm-db, both with status "Up"
```

### Reach the web frontend

```bash
pmabry@sheepsoc:~$ curl -s -o /dev/null -w "%{http_code}" http://localhost:3080/
# Expect: 200
```

Or open [http://192.168.50.100:3080/](http://192.168.50.100:3080/) in a browser on the LAN.

### Check container logs

```bash
pmabry@sheepsoc:~$ docker compose -f /mnt/ssd_working/emulatorjs/docker-compose.yml logs romm --tail 50
pmabry@sheepsoc:~$ docker compose -f /mnt/ssd_working/emulatorjs/docker-compose.yml logs romm-db --tail 50
```

## Start / Stop

```bash
# Start
pmabry@sheepsoc:~$ docker compose -f /mnt/ssd_working/emulatorjs/docker-compose.yml up -d

# Stop
pmabry@sheepsoc:~$ docker compose -f /mnt/ssd_working/emulatorjs/docker-compose.yml down

# Restart RomM only (leave DB running)
pmabry@sheepsoc:~$ docker compose -f /mnt/ssd_working/emulatorjs/docker-compose.yml restart romm
```

!!! note "Restart policy"
    Both containers are configured with `restart: unless-stopped` in the Compose file. They will start automatically after a host reboot unless manually stopped with `docker compose down`.

## Known Issues / Gotchas

### Metadata API keys not configured

At install (2026-05-23), no external metadata API keys were configured. RomM can enrich your ROM library with cover art, descriptions, and metadata from:

- **IGDB** (Internet Games Database)
- **ScreenScraper**
- **RetroAchievements**
- **SteamGridDB**

Without these keys, RomM still functions as a file manager and in-browser emulator, but artwork and descriptions will not be automatically populated. Adding these keys is a planned followup — see [Future Improvements](../future-improvements.md#romm-metadata-api-keys).

### ROM library structure

ROMs must be placed in `./library/roms/<platform>/` using RomM's expected platform folder names. Refer to the [RomM documentation](https://github.com/rommapp/romm) for the canonical platform directory names — they are case-sensitive and must match exactly for RomM to detect and categorize games correctly.

## See Also

- [Services — Service Catalog](../services.md) — port and status reference
- [Known Issues](../known-issues.md) — active system-wide landmines
- [Future Improvements — RomM metadata API keys](../future-improvements.md#romm-metadata-api-keys)
