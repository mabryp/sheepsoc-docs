# Standard Operating Procedures

**Purpose:** The repeatable, step-by-step procedures for common sheepsoc operations.

## 1. Run a Query in OpenWebUI

OpenWebUI at `http://192.168.50.100:8080` is the primary interface for all AI queries on sheepsoc. It handles chat, document upload, and Knowledge base RAG in one browser UI.

### Prerequisites — Verify Services Are Up

```bash
pmabry@sheepsoc:~$ systemctl is-active ollama open-webui elasticsearch
active
active
active
```

If any service is down, start it:

```bash
pmabry@sheepsoc:~$ sudo systemctl start ollama.service
pmabry@sheepsoc:~$ sudo systemctl start open-webui.service
pmabry@sheepsoc:~$ sudo systemctl start elasticsearch.service
```

### Running a Query

1. Open `http://192.168.50.100:8080` in a browser.
2. Start a new chat.
3. To query against a Knowledge base, type `#` in the message box and select the collection from the dropdown.
4. Type your question and submit. OpenWebUI retrieves relevant document chunks from Elasticsearch, injects them as context, and streams the LLM answer back.

!!! note "Tip"
    Ollama idles at ~0 MB VRAM when no query is running — leave it running at all times. You almost never need to stop it.

## 2. Check Service Health

### Quick "Is Everything Fine?" Dashboard

```bash
# Anything failed?
pmabry@sheepsoc:~$ systemctl --failed

# The big six — should all be active (running)
pmabry@sheepsoc:~$ for s in ollama open-webui elasticsearch kibana logstash jupyter; do
                    printf "%-16s %s\n" "$s" "$(systemctl is-active $s)"
                  done
```

### Deep Health Pokes

```bash
# Elasticsearch cluster state — auth required (xpack.security enabled 2026-04-21)
pmabry@sheepsoc:~$ curl -s -u elastic:<password> http://localhost:9200/_cluster/health?pretty

# Disk usage — watch /, /mnt/ssd_working, /mnt/nvme_working
pmabry@sheepsoc:~$ df -h | grep -E '^/|Filesystem'

# Memory + load
pmabry@sheepsoc:~$ free -h
pmabry@sheepsoc:~$ uptime

# GPU
pmabry@sheepsoc:~$ nvidia-smi
```

## 3. Switch the LLM Model in OpenWebUI

OpenWebUI uses a per-chat model selector. Switching models does not require restarting any service — just change the selection in the UI.

1. Confirm the target model is present on Ollama: `ollama list`
2. Open OpenWebUI at `http://192.168.50.100:8080`.
3. In any chat, use the model dropdown at the top of the chat window to select the desired model.

```bash
# Verify which models are available before switching
pmabry@sheepsoc:~$ ollama list
NAME                             ID            SIZE      MODIFIED
dolphin3:latest                  ...           4.7 GB    ...
gemma3:12b                       ...           8.9 GB    ...
mistral:7b                       ...           4.1 GB    ...
```

!!! note "Recommendation"
    `gemma3:12b` gives the best quality/speed tradeoff for RAG queries. `dolphin3:latest` is faster for simple questions that do not need deep reasoning over retrieved context.

## 4. Add Documents to Elasticsearch

There are two distinct ingest paths depending on which system you are feeding.

### 4a. OpenWebUI Knowledge Bases (Bulk Ingest Script)

Use this path to populate an OpenWebUI Knowledge base from a directory of pre-extracted `.txt` files in `/mnt/ssd_working/Data/Ingest_Ready/`. The script at `~/repositories/sheepsoc/ingest_to_openwebui.py` writes to both Elasticsearch and SQLite simultaneously and is idempotent — already-indexed files are skipped on re-run.

!!! note "Dedicated Page"
    Full step-by-step SOP, dataset reference table, architecture notes, and troubleshooting guide live on the dedicated page: **[OpenWebUI KB Bulk Ingest](../infrastructure/runbooks/openwebui-kb-bulk-ingest.md)**

### 4b. Legacy CLI RAG Index (Deprecated — Do Not Use)

!!! danger "Deprecated"
    The CLI RAG prototype (`~/repositories/sheepsoc`) and its associated Elasticsearch index (`intfloat_e5_large_v2__512__125`) are no longer maintained. Do not add documents to this index. Use the OpenWebUI Knowledge base path (4a above) for all document ingestion.

The legacy CLI RAG app used `intfloat/e5-large-v2` (1024-dim) embeddings and a hybrid KNN + BM25 pipeline with CrossEncoder reranking. Its ingestion scripts live in `~/repositories/sheepsoc/tests/`. This pipeline is not compatible with the OpenWebUI RAG index (`open_webui_collections_d768`) — the two systems use separate ES indexes with different embedding models.

## 5. Read Logs in Kibana

Open `http://192.168.50.100:5601`. The useful views are:

| View | What You Get |
|---|---|
| Observability → Logs | Tailed view of all log streams in ES |
| Discover | Ad-hoc query across any data stream or index |
| Dashboards | Saved dashboards for host and service overviews |

### Relevant Data Streams

| Stream | Source |
|---|---|
| `logs-syslog.asus-default` | ASUS RT-AX5400 router syslog |
| `logs-syslog.opnsense-default` | OPNsense firewall syslog |
| `filebeat-*` | sheepsoc system and service logs (filebeat) |
| `metricbeat-*` | sheepsoc host metrics (CPU/RAM/disk/net) |

Common KQL filters in Discover:

```
# Only high-severity ASUS events
data_stream.dataset : "syslog.asus" and log.syslog.severity.code <= 4

# Filebeat errors on sheepsoc from the last hour
log.level : "error" and host.name : "sheepsoc"
```

## 6. Connect to ASUS Router and OPNsense

### ASUS RT-AX5400

- Web admin: `http://192.168.50.1`
- SSH: port **1024**, key at `~/.ssh/asus_router`

```bash
pmabry@sheepsoc:~$ ssh -i ~/.ssh/asus_router -p 1024 admin@192.168.50.1
```

### OPNsense

- Web admin: `https://192.168.50.253` — the browser will complain about the self-signed cert; accept it.
- SSH: port 22 (if enabled in the OPNsense UI).

## 7. Conda Environment Management

Short reference — for the "why" of conda see the [Conda Guide](../infrastructure/platforms/conda.md).

```bash
# List every environment you have
pmabry@sheepsoc:~$ conda env list

# Activate one
pmabry@sheepsoc:~$ conda activate sheepsoc

# See what's installed in the active env
(sheepsoc) pmabry@sheepsoc:~$ conda list

# Leave the current env (back to base or plain shell)
(sheepsoc) pmabry@sheepsoc:~$ conda deactivate

# If conda isn't initialized in a new shell (e.g., a systemd unit)
pmabry@sheepsoc:~$ source ~/infrastructure/miniconda3/etc/profile.d/conda.sh
```

## 8. Restore Vaultwarden from Backup

!!! danger "High stakes — read RESTORE.md first"
    Vaultwarden is the authoritative store for all credentials on sheepsoc. A restore operation replaces the live vault database. Do not proceed without reading the full restore procedure at `/home/pmabry/infrastructure/vaultwarden/backup/RESTORE.md`.

The age private key (`~/.config/sops/age/keys.txt`) is required to decrypt backup archives. If restoring to a new machine, ensure the key is available before starting.

Quick reference:

```bash
# List available local backups
pmabry@sheepsoc:~$ ls -lth /mnt/data_extra/backups/vaultwarden/

# Decrypt a backup archive (replace <archive> with the filename)
pmabry@sheepsoc:~$ age --decrypt \
    -i ~/.config/sops/age/keys.txt \
    /mnt/data_extra/backups/vaultwarden/<archive>.tar.gz.age \
    -o /tmp/vw-restore.tar.gz

# Stop the container before replacing vault data
pmabry@sheepsoc:~$ cd /home/pmabry/infrastructure/vaultwarden
pmabry@sheepsoc:~/infrastructure/vaultwarden$ docker compose down

# Extract and restore (verify paths match before overwriting)
pmabry@sheepsoc:~$ tar -xzf /tmp/vw-restore.tar.gz -C /tmp/vw-restore/
# Then copy the restored files into data/ — see RESTORE.md for exact steps

# Start the container and verify
pmabry@sheepsoc:~/infrastructure/vaultwarden$ docker compose up -d
```

For the complete step-by-step procedure including integrity verification, see:
**`/home/pmabry/infrastructure/vaultwarden/backup/RESTORE.md`**

See also [Vaultwarden — Backup](../infrastructure/platforms/vaultwarden.md#backup) for backup schedule, encryption details, and off-site status.

## 9. Migrate LastPass Credentials into Vaultwarden

!!! note "Dedicated Runbook"
    Full step-by-step procedure, security warnings (plaintext CSV handling, master password pre-flight, shredding), and post-migration pruning guidance live on the dedicated runbook: **[LastPass to Vaultwarden Migration](../infrastructure/runbooks/lastpass-migration.md)**

Quick sequence:

1. Pre-flight — run `bw unlock` and confirm the master password works **before** exporting from LastPass. If it is forgotten, reset the account now while the vault is empty (see the runbook for the reset path via the admin panel).
2. Export from LastPass: **LastPass web vault → Advanced Options → Export**.
3. Move the CSV to `/tmp/vw-import/` (mode 600). Do not write it under `~/infrastructure/` or `/mnt/data_extra/` — both are backed up, including off-site to Google Drive.
4. Unlock for a non-interactive session: `export BW_SESSION=$(bw unlock --raw)`
5. Import: `bw import lastpasscsv /tmp/vw-import/lastpass_export.csv` (confirm format identifier with `bw import --formats` first)
6. Verify item count and spot-check entries in the vault UI at `https://sheepsoc-1.tail0f68e4.ts.net:8444/`
7. Shred immediately: `shred -u /tmp/vw-import/lastpass_export.csv && rmdir /tmp/vw-import`
8. Lock and clear: `bw lock && unset BW_SESSION`
9. Prune stale/dead entries from the vault UI at your own pace.

## 10. Push New Credentials into Vaultwarden

When a new service is added to sheepsoc and introduces credentials, those credentials should be imported into Vaultwarden using the bootstrap tool.

!!! note "Dedicated Runbook"
    Full step-by-step procedure, catalog conventions, the item-type mapping table, and instructions for growing `catalog.py` live on the dedicated runbook: **[Vaultwarden Bootstrap](../infrastructure/runbooks/vaultwarden-bootstrap.md)**

Quick reference:

```bash
pmabry@sheepsoc:~$ cd /home/pmabry/infrastructure/vaultwarden/bootstrap
pmabry@sheepsoc:~/infrastructure/vaultwarden/bootstrap$ export BW_SESSION=$(bw unlock --raw)
pmabry@sheepsoc:~/infrastructure/vaultwarden/bootstrap$ ./bootstrap.py --dry-run          # verify
pmabry@sheepsoc:~/infrastructure/vaultwarden/bootstrap$ ./bootstrap.py --category <name>  # push
pmabry@sheepsoc:~/infrastructure/vaultwarden/bootstrap$ bw lock && unset BW_SESSION
```

## 11. Remote Access via Tailscale

Tailscale provides encrypted remote access to all sheepsoc services from any enrolled device. There are two ways to reach services off the LAN.

### Option A — Tailscale Serve (HTTPS, easier for browser-based services)

Three services have dedicated HTTPS URLs via `tailscale serve` (configured 2026-05-09). These are the simplest way to reach OpenWebUI, Jupyter, and the docs site from a remote device — no port-substitution needed, and the browser gets a valid TLS cert automatically.

| Service | URL |
|---|---|
| Open WebUI | `https://sheepsoc-1.tail0f68e4.ts.net/` |
| Jupyter Notebook | `https://sheepsoc-1.tail0f68e4.ts.net:8443/` |
| Docs (this site) | `https://sheepsoc-1.tail0f68e4.ts.net:10000/` |

These URLs are reachable only from devices enrolled in the `tail0f68e4` tailnet. They are not public.

### Option B — IP Substitution (all services)

For all other services (Kibana, Elasticsearch, Ollama, Vikunja, etc.), substitute `100.117.117.43` for `192.168.50.100` in any LAN URL or SSH command.

```bash
# SSH to sheepsoc over Tailscale (same key, same port — port 22)
ssh pmabry@100.117.117.43

# Or via MagicDNS
ssh pmabry@sheepsoc-1.tail0f68e4.ts.net

# Check tailnet status from sheepsoc
pmabry@sheepsoc:~$ tailscale status
```

For procedures covering peer enrollment, peer removal, key rotation, managing serve rules, and full uninstall, see the dedicated runbook:

**[Tailscale Operations](../infrastructure/runbooks/tailscale-ops.md)**

## 12. Planned Shutdown and Startup

For a safe, ordered shutdown and post-boot verification sequence, follow the dedicated runbook:

**[Shutdown & Startup Runbook](../infrastructure/runbooks/shutdown-startup.md)**

The runbook covers the correct stop order (OpenWebUI → Elasticsearch → Ollama), emergency shutdown, post-boot service verification, and common startup failure troubleshooting.

## 13. What NOT to Do

!!! note "NFS Mount — Restored and Boot-Safe (2026-06-28)"
    The `/etc/fstab` entry for `san01.mabry.lan:/volume1/NFS_Share → /mnt/nfs` is **active** as of 2026-06-28. This entry was previously commented out because the NAS was offline — an uncommented NFS mount without `nofail` will hang boot while the kernel retries the network mount. It is now restored with boot-safe options: `_netdev,nfsvers=3,nofail,x-systemd.automount,x-systemd.mount-timeout=30`. The `nofail` flag means a dead NAS will not block boot; `x-systemd.automount` means the kernel does not contact the NAS until a process first accesses `/mnt/nfs`. If SAN01 goes offline again, the mount will fail silently on access — it will not hang the machine. See [Known Issues — history](../infrastructure/known-issues.md#2026-06-28-san01-nfs-server-restored-boot-safe-fstab-entry) for the full record.

!!! danger "Do Not"
    **Do not update the NVIDIA driver without checking first.** The RTX 5060 Ti currently runs 570.169 / CUDA 12.8 and Ollama is stable on this combo. Unplanned updates can take GPU inference offline.

!!! warning "Avoid"
    **Do not run silent `sudo`.** Announce what needs root and why. Phillip's rule: nothing happens on this machine without him knowing about it.

!!! warning "Avoid"
    **Do not delete configs when "cleaning up".** Back up or comment out instead. Reversibility beats tidiness.
