# LastPass to Vaultwarden Migration

**Purpose:** Bulk-migrate web login credentials from LastPass into the local [Vaultwarden](../platforms/vaultwarden.md) instance using a one-shot `bw import` operation.

## Overview

This runbook covers importing the existing LastPass credential export into Vaultwarden using the Bitwarden CLI's bulk import command. It is the mechanism for moving the body of web logins — usernames, passwords, and URLs — that currently live in LastPass into the self-hosted vault.

!!! note "How this differs from the Bootstrap tool"
    The [Vaultwarden Bootstrap](vaultwarden-bootstrap.md) runbook covers a separate, complementary mechanism: a declarative catalog that pushes infrastructure secrets already on disk on sheepsoc (SSH keys, `.env` files, API tokens) into Vaultwarden with rich custom fields and idempotent create-or-skip logic. Each catalog run is always safe to repeat.

    **This runbook** is for a one-shot `bw import` of the LastPass web login vault — a completely different input format and a different re-run semantic. `bw import` is **not idempotent**: running it a second time creates duplicate entries rather than skipping existing ones. The two operations target different credential categories and should not be confused with each other.

### Migration Status

This migration is the tracked **Personal Secrets Manager** effort on the Monday SheepSOC board. As of 2026-06-28, Vaultwarden holds only two smoke-test entries (`age-key` and `id_rsa`), making this the lowest-risk moment to perform the import.

---

## Before You Start — Critical Warnings

Read all three sections below before beginning. They describe failure modes that cannot be recovered from after the fact.

!!! danger "The master password is unrecoverable"
    Vaultwarden is zero-knowledge: the master password derives the key that encrypts and decrypts the vault. The server never receives or stores it. There is no recovery mechanism — [SMTP is not configured](../platforms/vaultwarden.md#no-smtp), so no password-reset email will ever be sent.

    A Bitwarden API key (Client ID and Client Secret) is sometimes described as an alternative to a master password. This does not apply here: `bw login --apikey` replaces the *login* step, but after login the vault remains `locked`. Running `bw unlock` still requires the master password to decrypt the vault for any read or write operation. No API key is currently configured anyway.

    **There is no way to recover a forgotten master password.** A reset requires wiping the user account and starting over.

!!! warning "Test your master password BEFORE exporting from LastPass"
    The vault currently holds only two non-critical smoke-test entries. This is the lowest-risk window to discover a forgotten master password. **Do not export from LastPass until you have confirmed `bw unlock` succeeds in Step 1.**

    If the unlock fails, the vault is essentially empty — resetting now costs nothing. After the import, a reset means permanent loss of every migrated credential.

!!! danger "The LastPass CSV export is every password you own, in plaintext"
    The LastPass export is a single unencrypted CSV file containing every username, password, and URL in the vault. Handle it accordingly:

    - **Do not write it to any backed-up location.** Both `~/infrastructure/` and `/mnt/data_extra/` are subject to nightly backup including off-site upload to Google Drive. A plaintext credential dump must not appear in either location.
    - **Write it to a non-backed-up, mode-600 location.** A session directory under `/tmp` is appropriate — `/tmp` is not included in the nightly backup.
    - **Shred it immediately after a verified import** using `shred -u`. Plain `rm` marks the blocks free but does not overwrite them; the data remains on disk until those blocks are reused.

---

## Prerequisites

1. [Vaultwarden](../platforms/vaultwarden.md) is running and reachable. Verify:
   ```bash
   pmabry@sheepsoc:~$ curl -sf http://127.0.0.1:8222/alive && echo "ok"
   ```

2. The Bitwarden CLI `bw` is installed at `~/.local/bin/bw` and configured against the local server. Verify:
   ```bash
   pmabry@sheepsoc:~$ bw config server
   https://sheepsoc-1.tail0f68e4.ts.net:8444
   ```

3. [Tailscale](../platforms/tailscale.md) is running — `bw` routes through the tailnet to reach Vaultwarden:
   ```bash
   pmabry@sheepsoc:~$ tailscale status
   ```

4. You are logged in to the vault (status is `locked`, not `unauthenticated`). Check:
   ```bash
   pmabry@sheepsoc:~$ bw status
   ```
   If `"status"` shows `"unauthenticated"`, run `bw login` first. You will be prompted for your email address and master password.

---

## Step 1: Verify the Master Password

Before exporting anything from LastPass, confirm the master password is known and working. Type your master password when prompted:

```bash
pmabry@sheepsoc:~$ bw unlock
```

A successful unlock prints the session token and reports `Your vault is now unlocked!`. Immediately re-lock after this test — you do not need the active session yet:

```bash
pmabry@sheepsoc:~$ bw lock
```

If the unlock succeeds, proceed to Step 2.

If the unlock fails (wrong password or repeated errors), **stop here** and follow the path below while the vault is still empty.

### If the Master Password Is Forgotten

Because the vault currently holds only two non-critical smoke-test entries, resetting is low-cost — the existing entries are re-creatable from disk using the [Vaultwarden Bootstrap](vaultwarden-bootstrap.md) tool after re-registration.

1. Open the admin panel at `https://sheepsoc-1.tail0f68e4.ts.net:8444/admin` and authenticate with the `ADMIN_TOKEN` from `vaultwarden.env`.
2. Under **Users**, find `phillip.mabry@gmail.com` and delete the account. This destroys the two existing smoke-test entries.
3. Temporarily enable signups to permit re-registration. Follow the **Option A — Temporary signup window** procedure in the [Enable New User Signups](../platforms/vaultwarden.md#enable-new-user-signups-admin-task) section of the Vaultwarden platform page: set `SIGNUPS_ALLOWED=true` in `vaultwarden.env`, force-recreate the container, register a new account with a known master password, then immediately set `SIGNUPS_ALLOWED=false` and force-recreate again.
4. Return to the top of this runbook and start from the Prerequisites section with the new account.

---

## Step 2: Export from LastPass

This is a manual action performed by Phillip under his LastPass master password. Choose one of the two options below.

### Option A — LastPass Web Vault (no CLI required)

1. Log in at [lastpass.com](https://lastpass.com).
2. Navigate to **Advanced Options → Export**.
3. Confirm your LastPass master password when prompted.
4. LastPass downloads a CSV file (the exact filename depends on the browser, e.g., `lastpass_export.csv`). Save it somewhere accessible — you will move it to a secured location in the next step.

### Option B — LastPass CLI (if `lpass` is installed)

```bash
pmabry@sheepsoc:~$ lpass export > ~/lastpass_export.csv
```

`lpass export` requires an active `lpass` session. If not currently logged in, run `lpass login phillip.mabry@gmail.com` first. Do not leave the file in the home directory for long — move it to a secured location immediately in Step 3.

---

## Step 3: Secure the Export File

Before doing anything else with the CSV, move it to a non-backed-up, mode-600 location.

```bash
# Create a secured working directory under /tmp
pmabry@sheepsoc:~$ mkdir -m 700 /tmp/vw-import

# Move the CSV in and restrict its permissions
pmabry@sheepsoc:~$ mv <downloaded-csv-path> /tmp/vw-import/lastpass_export.csv
pmabry@sheepsoc:~$ chmod 600 /tmp/vw-import/lastpass_export.csv
```

Verify the permissions before proceeding:

```bash
pmabry@sheepsoc:~$ ls -la /tmp/vw-import/
# Expected output:
# -rw------- 1 pmabry pmabry ... lastpass_export.csv
```

`/tmp` is not included in the nightly backup. Do not move the file under `~/infrastructure/`, `/mnt/data_extra/`, or any other path that is synced off-machine.

---

## Step 4: Unlock the Vault for a Non-Interactive Session

```bash
pmabry@sheepsoc:~$ export BW_SESSION=$(bw unlock --raw)
```

`bw unlock --raw` prompts for your master password and prints only the raw session token to stdout. Capturing it into `BW_SESSION` allows all subsequent `bw` commands to run non-interactively without re-prompting. The master password is not stored anywhere — it is typed once, used to derive the session token in memory, and is not persisted to disk.

Verify the session is active before proceeding:

```bash
pmabry@sheepsoc:~$ bw status
# "status" should be "unlocked"
```

---

## Step 5: Import

First, confirm the exact format identifier for LastPass CSV imports. This query requires an unlocked session:

```bash
pmabry@sheepsoc:~$ bw import --formats | grep -i lastpass
```

The expected identifier is `lastpasscsv`. Use whatever `bw import --formats` reports — do not rely on training data or docs for this value, as Bitwarden CLI versioning can affect format identifiers.

Run the import:

```bash
pmabry@sheepsoc:~$ bw import lastpasscsv /tmp/vw-import/lastpass_export.csv
```

The command imports all items in a single pass and prints a summary of the results. LastPass folder groups map automatically to Bitwarden folders in Vaultwarden — no manual folder creation is needed.

!!! warning "`bw import` is NOT idempotent"
    Running this command a second time creates duplicate entries — it does not check for existing items. If the import fails partway through or you need to re-import for any reason, you must first delete all previously imported items via the Vaultwarden web UI at `https://sheepsoc-1.tail0f68e4.ts.net:8444/`, then re-run.

    This is a fundamental difference from the [Vaultwarden Bootstrap](vaultwarden-bootstrap.md) tool, which checks for existing items by name and skips them on re-run.

---

## Step 6: Verify the Import

Count the imported items and spot-check a few known entries.

```bash
# Count items in the vault (requires BW_SESSION to still be set)
pmabry@sheepsoc:~$ bw list items | python3 -c "import sys, json; items = json.load(sys.stdin); print(len(items), 'items in vault')"
```

Then open the vault web UI from any Tailscale-enrolled device and verify:

- The total item count is consistent with the number of logins in the LastPass export.
- A few spot-checked entries show the expected usernames, passwords, and URLs.
- LastPass folders appear as Bitwarden folders.

Only proceed to Step 7 once the import is confirmed correct.

---

## Step 7: Shred the Export File

After a verified import, destroy the plaintext CSV immediately.

```bash
pmabry@sheepsoc:~$ shred -u /tmp/vw-import/lastpass_export.csv
pmabry@sheepsoc:~$ rmdir /tmp/vw-import
```

`shred -u` overwrites the file's contents with multiple passes of random data before unlinking it, making the data unrecoverable from the underlying blocks. Plain `rm` does not do this.

Confirm the file is gone:

```bash
pmabry@sheepsoc:~$ ls /tmp/vw-import 2>&1
# Expected: ls: cannot access '/tmp/vw-import': No such file or directory
```

---

## Step 8: Lock the Vault and Clear the Session

```bash
pmabry@sheepsoc:~$ bw lock
pmabry@sheepsoc:~$ unset BW_SESSION
```

`BW_SESSION` exists only in the current shell process — closing the terminal clears it automatically. Explicitly locking and unsetting is the safer habit, especially if the terminal session will remain open.

---

## Step 9: Post-Migration Pruning

The chosen migration strategy is **import everything first, then prune**. After the import is complete and verified:

1. Open `https://sheepsoc-1.tail0f68e4.ts.net:8444/` and review the imported items.
2. Delete stale entries: dead accounts, expired trials, and obvious duplicates.
3. Consolidate or rename folders if the LastPass folder structure did not map cleanly to what you want in Vaultwarden.

There is no time pressure on pruning — the import is complete and the vault is encrypted at rest. Pruning can be done incrementally over time.

---

## See Also

- [Vaultwarden](../platforms/vaultwarden.md) — platform page: deployment, configuration, health checks, backup schedule, and known gotchas
- [Vaultwarden Bootstrap](vaultwarden-bootstrap.md) — the complementary tool for pushing catalog-defined infrastructure secrets (SSH keys, `.env` files) into Vaultwarden with custom fields and idempotent logic
- [Tailscale](../platforms/tailscale.md) — the network layer that exposes Vaultwarden over the tailnet; must be running for `bw` to reach the server
