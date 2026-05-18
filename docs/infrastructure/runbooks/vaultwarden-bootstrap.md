# Vaultwarden Bootstrap

**Purpose:** Populate the local [Vaultwarden](../platforms/vaultwarden.md) instance with catalog-defined credentials from sheepsoc using the Python bootstrap tool.

## Overview

The bootstrap tool at `/home/pmabry/infrastructure/vaultwarden/bootstrap/` reads a declarative catalog (`catalog.py`) and pushes each entry into Vaultwarden via the Bitwarden CLI (`bw`). It is designed to be run incrementally — the catalog grows one credential category at a time as secrets are migrated from their current on-disk locations into the vault.

The tool is idempotent: re-running it is always safe. If an item with the given name already exists in the vault, the tool skips it. Nothing is ever modified or deleted by the bootstrap; it only creates.

The full technical reference — including all CLI flags, the `lib.py` helper API, and known caveats — lives in the on-disk README at `/home/pmabry/infrastructure/vaultwarden/bootstrap/README.md`.

## Prerequisites

1. [Vaultwarden](../platforms/vaultwarden.md) is running and reachable at `https://sheepsoc-1.tail0f68e4.ts.net:8444/`.
2. The `bw` CLI is installed at `~/.local/bin/bw` and configured against the local server (`bw config server` has been run).
3. You are logged in to the vault (`bw login` — status should show `locked`, not `unauthenticated`). If you have never logged in, run `bw login` first; you will be prompted for the master password.

!!! warning "Vault must be logged in, not just unlocked"
    There are two distinct states: **logged out** (no local account data), **locked** (account data present, session needed), and **unlocked** (session active, commands work). The bootstrap requires an unlocked session. If `bw status` shows `unauthenticated`, run `bw login` before proceeding.

## 1. Unlock the Vault

```bash
pmabry@sheepsoc:~$ export BW_SESSION=$(bw unlock --raw)
```

`bw unlock --raw` prompts for your master password and prints only the raw session token. Capturing it into `BW_SESSION` is what lets all subsequent `bw` commands run non-interactively.

## 2. Navigate to the Bootstrap Directory

```bash
pmabry@sheepsoc:~$ cd /home/pmabry/infrastructure/vaultwarden/bootstrap
```

## 3. Review What Will Be Pushed

```bash
# List available categories defined in catalog.py
pmabry@sheepsoc:~/infrastructure/vaultwarden/bootstrap$ ./bootstrap.py --list

# Dry-run: show exactly what would be created without writing anything
pmabry@sheepsoc:~/infrastructure/vaultwarden/bootstrap$ ./bootstrap.py --dry-run
```

Always run `--dry-run` first when the catalog has changed since the last run. It prints each item and the action it would take (create or skip).

## 4. Push Credentials to Vaultwarden

To push a single category:

```bash
pmabry@sheepsoc:~/infrastructure/vaultwarden/bootstrap$ ./bootstrap.py --category sops
```

To push all categories in the catalog:

```bash
pmabry@sheepsoc:~/infrastructure/vaultwarden/bootstrap$ ./bootstrap.py
```

The script prints each item name, its resolved folder, and whether it was created or skipped. Failures are printed with the reason; a failure on one item does not abort the run.

## 5. Lock the Vault and Clear the Session

```bash
pmabry@sheepsoc:~/infrastructure/vaultwarden/bootstrap$ bw lock
pmabry@sheepsoc:~/infrastructure/vaultwarden/bootstrap$ unset BW_SESSION
```

Always do this when finished. `BW_SESSION` lives only in the current shell — closing the terminal also clears it — but locking explicitly is the safer habit.

---

## Catalog Reference

### Custom Fields — Required for Every Item

Every `CatalogItem` in `catalog.py` should set these `custom_fields`:

| Field | Meaning |
|---|---|
| `source-path` | Filesystem path where this credential currently lives on sheepsoc. |
| `var-name` | Env var or config key it populates. Use `(file-based; not env var)` if no variable is involved. |
| `consumer` | Which service or process reads this credential. |
| `restart-cmd` | Exact command to apply a change after rotating the credential. Use `(none)` if not applicable. |
| `last-rotated` | ISO date of last real rotation, or `imported YYYY-MM-DD` if the value was just captured as-is. |

### Naming Convention

Item names follow the pattern `sheepsoc / <area> / <name>`. This ensures that a vault search by area (e.g., `sheepsoc / ssh`) returns a coherent, scannable group rather than a flat undifferentiated list.

### Item Type Mapping

Use this table to decide what vault item type to use for a given source artefact:

| Source artefact | Vault item type |
|---|---|
| SSH private key + `.pub` file | `SSH Key` (type 5) — uses native `sshKey` block; fingerprint computed via `ssh-keygen -lf` |
| `.env` file containing multiple vars | `Secure Note` — whole file pasted into the notes body; preserves grouping |
| Standalone password or OAuth token | `Login` — URI set if applicable |
| Binary key material (`.p12`, `.gpg`, PKCS12) | `Secure Note` + file attachment |
| age private key | `Secure Note` + file attachment — treat as critical; physical paper backup recommended |

---

## Growing the Catalog

The full intended catalog is derived from `/home/pmabry/infrastructure/vaultwarden/CREDENTIAL_INVENTORY.md` (mode 600, kept off-wiki — it enumerates every secret path on sheepsoc). The bootstrap is designed to be incremental; not everything has to be imported at once.

To add a new credential:

1. Open `catalog.py` in an editor:

    ```bash
    pmabry@sheepsoc:~$ vim /home/pmabry/infrastructure/vaultwarden/bootstrap/catalog.py
    ```

2. Add a `CatalogItem` entry following the existing examples, setting all five custom fields.

3. Dry-run to verify the new entry parses and resolves correctly:

    ```bash
    pmabry@sheepsoc:~/infrastructure/vaultwarden/bootstrap$ ./bootstrap.py --dry-run
    ```

4. Push only the new category (safer than pushing everything):

    ```bash
    pmabry@sheepsoc:~/infrastructure/vaultwarden/bootstrap$ ./bootstrap.py --category <category-name>
    ```

After a real credential rotation, update its `last-rotated` field in `catalog.py`, delete the existing vault item (via the web UI or `bw delete item <id>`), and re-run bootstrap for that category. The tool will not overwrite an existing item — deletion is the prerequisite for re-creation.

!!! danger "Do not commit catalog.py to a public repository"
    `catalog.py` enumerates the filesystem path of every secret on sheepsoc. No actual secret values are in the file, but the path inventory is sensitive metadata. Keep it in the local working directory only. It is `.gitignore`d in any repository that contains this tooling.

---

## Current Catalog Contents

As of `2026-05-18`, the catalog contains two smoke-test entries:

| Item name | Vault type | Source path |
|---|---|---|
| `sheepsoc / sops / age-key` | Secure Note + attachment | `/home/pmabry/.config/sops/age/keys.txt` |
| `sheepsoc / ssh / id_rsa` | SSH Key | `/home/pmabry/.ssh/id_rsa` |

These were imported to verify the end-to-end flow. The catalog will grow as credentials are migrated from the `CREDENTIAL_INVENTORY.md` list.

---

## See Also

- [Vaultwarden](../platforms/vaultwarden.md) — platform page covering deployment, configuration, and health checks
- On-disk README: `/home/pmabry/infrastructure/vaultwarden/bootstrap/README.md` — full flag reference, `lib.py` API notes, and known caveats (CLI version compatibility, re-create procedure)
- `/home/pmabry/infrastructure/vaultwarden/CREDENTIAL_INVENTORY.md` — full inventory of credential paths that the catalog should eventually cover (mode 600, not on wiki)
