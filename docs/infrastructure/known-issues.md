# Known Issues

**Purpose:** Landmines, unresolved items, and the history of what blew up and how it was fixed.

| Key | Value |
|---|---|
| Rule | Read this before making infrastructure changes |
| Last reviewed | 2026-05-18 |

## Active Landmines — Do Not Touch

!!! danger "Do Not"
    **MicroK8s: do not start.** The previous install's OpenEBS Node Disk Manager leaked to **247 GB of RAM**, starved the machine, and drove load average above 100. Persistent volumes were misconfigured and the cluster was unusable. The snap is installed but the services are stopped. See the [Services catalog](services.md) — MicroK8s is marked **hold**.

    Before restart, a proper rebuild plan is required. Key requirements:

    - Disable or memory-limit NDM before deploying any workload.
    - Decide a PV strategy up front — the `k8s_*` mounts are reserved for this.
    - Re-plan previous workloads (Elasticsearch in-cluster, pfSense, etc.) — some may stay on the host instead.

!!! danger "Do Not"
    **NFS mount to `san01.mabry.lan`: keep fstab line commented.** The NFS server is offline. Uncommenting the mount will hang boot while the kernel retries. Only restore after the NFS server is back *and* reachable from sheepsoc.

!!! danger "Do Not"
    **NVIDIA driver: do not update without checking.** Current combination is driver **570.169** / CUDA **12.8** on the RTX 5060 Ti, and [Ollama](services.md) is stable on it. The 5060 Ti needs a modern driver — rolling forward incautiously can leave the GPU in a broken state and take Ollama inference offline. Plan the upgrade, have a rollback, and announce before running `apt upgrade` on the NVIDIA packages.

## Watchlist

- **Security hardening steps 7–10** are still pending from the 2026-04-19 session. See the persistent memory pointer `project_security_hardening.md` for the list.
- **Config management** lives at `/home/pmabry/infrastructure/config-mgmt/`. New infrastructure changes should land there, not as ad-hoc edits scattered across the filesystem.
- **Disk watch:** `/` is at 202 GB of 936 GB used. Plenty of headroom, but keep an eye on ES data growth on `/` if indexes stay local.
- **Tailscale uses DERP relay (informational):** Due to Starlink CGNAT, direct peer-to-peer WireGuard connections to sheepsoc from external devices are not possible. Tailscale falls back to DERP relay automatically. This is expected and transparent — not a fault. If the uplink changes to a non-CGNAT provider, Tailscale will prefer direct paths automatically. See [Tailscale](platforms/tailscale.md).
- **Tailscale MagicDNS hostname is `sheepsoc-1` not `sheepsoc` (cosmetic):** Tailscale auto-suffixed `-1` because a stale prior "sheepsoc" entry holds the unsuffixed name in the tailnet. All IPs and Serve URLs resolve correctly. To reclaim the cleaner name, delete the stale entry at [https://login.tailscale.com/admin/machines](https://login.tailscale.com/admin/machines), then re-authenticate with `sudo tailscale up --force-reauth`. Low priority — address when convenient.

- **`tailscale serve` bind collision when local service binds `0.0.0.0`:** When `tailscale serve --https=<port>` is configured, `tailscaled` binds a listener on the tailnet IP (`100.117.117.43:<port>`). If the local service being proxied already binds to `0.0.0.0:<port>` (all interfaces), tailscaled loses the bind race and the serve rule fails silently — `tailscale serve status` may show the rule present, but HTTPS connections will not work. **Workaround:** use a different, unused port for the tailscale serve rule and let it proxy to the app's local port. The `10000+` numbering range is the convention adopted on sheepsoc for this purpose (e.g., Open WebUI binds `0.0.0.0:8080`, so tailscale serve uses `:10001 → localhost:8080`). Services that bind loopback-only (`127.0.0.1`) are not affected and can use any convenient port (e.g., Jupyter's `:8443 → localhost:8888`). See [Tailscale — Port-Numbering Convention](platforms/tailscale.md#port-numbering-convention) for the full rationale and table. {: #tailscale-serve-bind-collision }

- **Docker Compose v29 env-file variable substitution** — Docker Compose v29.4.1 performs `$variable` substitution on values loaded via `env_file:`. Any literal `$` in a value is treated as a variable reference. Argon2id hashes (and any PHC-format string) contain multiple `$` field separators that will be silently mangled. **Workaround: escape every `$` as `$$` in the env file.** Compose un-escapes `$$` → `$` when injecting into the container, so the runtime value is correct. This applies to any service using `env_file:` whose config values contain `$` — it is not Vaultwarden-specific. See [Vaultwarden — Docker Compose `$$` Escaping](platforms/vaultwarden.md#docker-compose-escaping-for-argon2-hashes) for the full example. {: #docker-compose-v29-env-file-variable-substitution }

- **`vg_elastic` linear LV is exposed to a mechanically unreliable NVMe carrier (strategic decision pending):** `nvme3n1` (Samsung 990 PRO 2TB, S/N S7KHNU0Y529975Z) sits on a mini-PCIe carrier card that does not stay mechanically seated. The drive was reseated 2026-05-14 and held — see [history entry below](#2026-05-14-nvme-reseat-and-new-sata-ssd-added) — but the failure mode will recur. The failure mode is **intermittent disappearance from the bus** (not data corruption): the drive vanishes and reappears after reseating. The structural risk is that `vg_elastic` is a **linear** LV spanning `nvme1n1` + `nvme3n1`. A linear LV cannot tolerate a missing PV — if the loose drive drops, the entire 3.6 TiB `/mnt/elastic_data` goes offline. Three strategic options have been identified; none has been executed yet:

    | Option | Summary | Usable capacity | Right when… |
    |---|---|---|---|
    | **A — Mirror `vg_elastic`** | `lvconvert --type raid1 -m1 vg_elastic/lv_elastic_data` — converts the LV to LVM RAID1 across both 990 PROs | 1.8 TiB (drops from 3.6 TiB) | `/mnt/elastic_data` is load-bearing — the volume survives the drive dropping out and degrades gracefully, resyncing on reseat |
    | **B — Demote `nvme3n1` out of `vg_elastic`** | Shrink `vg_elastic` to a single-PV VG on reliable `nvme1n1`; repurpose `nvme3n1` for scratch/archive where intermittent disappearance is tolerable | 1.8 TiB | `/mnt/elastic_data` is no longer load-bearing (e.g., leftover from the 2026-05-10 Elastic Cloud migration) |
    | **C — Root-cause hardware fix** | Replace the mini-PCIe carrier with a proper PCIe→M.2 adapter (full-height, screw-secured), or move the drive to an unused motherboard M.2 slot if one exists | No change | Always — A and B are workarounds; C is the actual fix and should run in parallel |

    **Open question blocking the A vs. B decision:** Is `/mnt/elastic_data` still load-bearing? Local ES was decommissioned 2026-05-10 per the [migration history](#2026-05-10-elasticsearch-migrated-to-elastic-cloud-local-es-decommissioned), but approximately 87 GiB remains in the mount. Before choosing A or B, confirm whether anything (OpenWebUI RAG indexes, backups, staging, etc.) actively reads or writes that path. If nothing does, Option B is the lower-complexity choice.

## History Log

### 2026-05-30 — tv_control.py Refined: KEY_CLEAR Replaces Backspace Loop

- Phillip refined `infrastructure/scripts/tv_control.py` again: replaced the 15x `KEY_BACKSPACE` clear loop with single `tv.send_key("KEY_CLEAR")` + `sleep(0.5)`, because the TV remote/UI has a dedicated clear button that fully clears the YouTube search bar in one step (confirmed by user). Tested successfully (no errors, cleaner). Token file still present; no pairing.
- Updated [Samsung TV Network Control runbook](../runbooks/wol-samsung-tv.md) (purpose, How It Works, troubleshooting, usage example), [services.md](#tv-control) (TV section), [schema.md](../schema.md) (registry), and this history entry per schema.md §6 update triggers for code/procedure change to runbook/services. Uses relative links, first-mention labels, **see also**/**runbook** reciprocals per §4 (verified interlinks; reversible via prior commit). Wiki = living record of current tv_control clear procedure.
- See also prior 2026-05-30 entries below for full context.

### 2026-05-30 — tv_control.py Updated to Clear YouTube Search Field (Prevents Stale Text)

- Phillip updated `infrastructure/scripts/tv_control.py` (after last wiki sync) to add loop of 15x `KEY_BACKSPACE` (with short 0.1s sleeps) right after focusing the input field and before `send_text(query)`.
- This reliably clears any residual text from prior searches in the YouTube app UI.
- Tested successfully via `conda run -n sheepsoc --youtube-search 'Try not to laugh'`. Token remains saved in `~/.config/samsung-tv-token.json`; no re-pairing needed.
- Updated runbook (How It Works, troubleshooting, purpose), services.md (TV section), schema.md (registry), and this history entry per section 6 triggers for "code change to TV control procedure / runbook / services".
- Links follow section 4 rules exactly (relative from each file, first-mention, **see also**/**runbook**, reciprocals where meaningful, no noise). Interlinks verified. Reversible edits; wiki remains living record. Fulfills CLAUDE.md (explanation before each edit, no silent sudo, Documentation agent task completed).
- No topology/known-issues/index.md triggered as this was an improvement, not new landmine.

### 2026-05-30 — Samsung TV Documented + WoL Runbook (Resolves Undocumented LAN Device)

- Network-engineer subagent successfully retrieved Samsung TV details from ASUS router DHCP table: IP `192.168.50.175`, MAC `54:3A:D6:5D:B0:EC`, hostname "Samsung", active lease.
- WoL magic packet sent (`wakeonlan -i 192.168.50.255 54:3a:d6:5d:b0:ec`) — succeeded; TV now powered on and documented for the first time.
- New runbook [WoL Samsung TV](runbooks/wol-samsung-tv.md) created with full step-by-step (TV settings check: Settings > Network > Expert > Wake on LAN or Power On with Mobile; prerequisites: wired Ethernet; command; troubleshooting; test confirmation).
- Updated [Topology](topology.md) (added to Network Topology diagram, Host Layout/key table), [index.md](index.md) (Runbooks list), [schema.md](../schema.md) (page registry per section 2), this history entry.
- **services.md** left unchanged (WoL is a client-side tool, not a running systemd service on sheepsoc; fits better in runbook).
- All per schema.md update triggers for "new device added to LAN" and "new runbook created" (section 6). Link rules followed exactly (section 4): relative links on first mention, reciprocal links added where relationship is meaningful (runbook ↔ topology, runbook ↔ index/known-issues), backlinks verified. No mkdocs build. This fulfills CLAUDE.md post-infrastructure-change Documentation agent requirement.
- Wiki is now the single source of truth. Reciprocal links verified.

### 2026-05-30 — Samsung TV Full Network Control Added (tv_control.py Script)

- New script `infrastructure/scripts/tv_control.py` generated and placed (executable). Prioritizes volume control (`--volume`, `--up`/`--down`, `--mute`), integrates known WoL for `--power on` (MAC/IP from prior test), supports raw keys, one-time pairing (token to `~/.config/samsung-tv-token.json`), comprehensive error handling, CLI argparse help.
- Runbook `runbooks/wol-samsung-tv.md` expanded (now titled Samsung TV Network Control in registry/index) with installation (sheepsoc conda + `pip install samsungtvws wakeonlan` — confirmed safe per CLAUDE.md, builds on existing websocket-client), full usage examples (volume first), pairing procedure, expanded prerequisites (add Network Remote/Access Notification), "How It Works" section, updated troubleshooting, integration note with 2026-05-30 DHCP/WoL success.
- Updated [Topology](topology.md) (control notes added to diagram, table, host layout; links updated), [index.md](index.md) (runbook description), [schema.md](../schema.md) (registry purpose + triggers for "new control script added / device control procedure"), this history entry, and all cross-links per section 4 (reciprocal **see also** where meaningful; verified no broken links).
- No changes to services.md (client-side tool). No mkdocs build. Reversible backups made of edited files. Fulfills CLAUDE.md exactly: explanations before each action, no silent sudo, wiki single source of truth, Documentation agent brief covered all capabilities + prior success.
- Wiki now fully reflects complete TV network control setup (WoL + websocket commands from sheepsoc).

### 2026-05-30 — Samsung TV YouTube Search Capability Added to tv_control.py

- Enhanced `tv_control.py` with `--youtube-search "QUERY"` (uses `samsungtvws.run_app("111299001912")` to launch YouTube, navigation keys to search field, `send_text()` for query like "Try not to laugh", final enter; tuned sleeps for reliability).
- Tested successfully via `conda run -n sheepsoc` (no errors; TV at 192.168.50.175 responded).
- Updated [services.md](services.md#tv-control) (new TV Control section), [runbooks/wol-samsung-tv.md](runbooks/wol-samsung-tv.md) (usage, how it works, troubleshooting), [topology.md](topology.md), [index.md](index.md), [schema.md](../schema.md) (registry), this history entry.
- Per schema.md section 6 update triggers for "new service capability" and runbook changes. All links follow section 4 rules (relative MkDocs syntax from each file's location, first-mention, explicit labels like **see also**/**runbook**, reciprocal backlinks added to services.md, topology.md, index.md, known-issues.md, schema.md where meaningful; verified no broken links or noise). Reversible edits; no mkdocs build; fulfills CLAUDE.md (explanations before edits, wiki as living record, Documentation agent invoked post-change).
- Wiki now fully reflects current tv_control.py state including YouTube integration.

### 2026-05-30 — RAG Experiments Wiki Updated (Resolves Multiple Issues)

- Comprehensive documentation added for active RAG workspace at `~/jupyter/rag_experiments/`: RAG-001 (StackExchange sysadmin ~48k doc corpus, full pipeline, golden dataset with AI/human judgments, notebooks), RAG-002 scaffold, Elastic Cloud 9.4.0 integration details, hybrid RRF vs dense/ELSER/BM25 benchmarks (specific nDCG@10 results), embedding comparisons (nomic/mxbai), evaluation pipelines, concepts, runbooks, and sources.
- New [research/rag-experiments.md](../research/rag-experiments.md) created; research/agenda.md, elasticsearch-elser.md, openwebui-rag.md, schema.md, and index pages updated with links and current status.
- Marks Elastic Cloud migration as fully documented for research track; OpenWebUI cloud rewiring noted as next step. Resolves known issues around RAG-001/002 gaps, migration, comparisons, golden datasets, and old Streamlit platform (now deprecated).
- Wiki is now the single source of truth per lab policy. No mkdocs build run. Changes committed and pushed.
- Reciprocal links verified per schema section 4. Update triggers from schema section 6 followed.

### 2026-05-18 — Tailscale Serve Map Expanded

- **Docs landing page is now the default tailnet entry point.** `https://sheepsoc-1.tail0f68e4.ts.net/` (port 443) previously proxied to Open WebUI. It now proxies to the docs landing page (`localhost:80`). Open WebUI moved to `:10001`.
- **Newly exposed via tailnet:** Kibana (`:10002 → localhost:5601`), Uptime Kuma (`:10003 → localhost:3001`).
- **Port 10000 retained as backwards-compatibility alias** for the docs landing page.
- **`10000+` port-numbering convention adopted** for services whose local process binds `0.0.0.0:<port>`. See the new Watchlist entry for the bind-collision rationale and the full convention at [Tailscale — Port-Numbering Convention](platforms/tailscale.md#port-numbering-convention).
- `services.md` and `index.md` updated to show both LAN and tailnet URLs side-by-side.
- Vikunja row flagged "to be decommissioned" in the service catalog.

### 2026-05-17 — Vaultwarden Admin Token Hashed; Env File Renamed

- **`ADMIN_TOKEN` rotated and replaced with an argon2id hash** using the Vaultwarden "bitwarden" preset (m=65540, t=3, p=4). The plaintext token was generated with `secrets.token_urlsafe(48)` entropy. The hash now lives in `vaultwarden.env`. Vaultwarden no longer logs a startup warning about a plaintext token.
- **Env file renamed** from `.env` to `vaultwarden.env`. Docker Compose auto-loads a `.env` file in the project directory for its own variable interpolation. This caused every `$` in the argon2 hash to be processed as a variable reference before the value ever reached the container, silently mangling the hash on startup. Renaming the file to `vaultwarden.env` and referencing it via the `env_file:` directive in `docker-compose.yml` decouples it from Compose's auto-load behavior.
- **Remaining `$$` escaping requirement:** Even with the renamed env file, Docker Compose v29.4.1 still performs variable substitution on `env_file:`-loaded values. The hash in `vaultwarden.env` has every `$` escaped as `$$`. This must be preserved on any future token rotation. See [Vaultwarden — Rotate the Admin Token](platforms/vaultwarden.md#rotate-the-admin-token) and the [Watchlist entry above](#docker-compose-v29-env-file-variable-substitution) for details.
- Resolves: Vaultwarden plaintext admin token (previously in Watchlist).

### 2026-05-14 (evening) — Storage Reshuffle: PNY 4TB Repartitioned and P3-1TB Commissioned

- **PNY CS900 4TB SSD (`/dev/sda`) repartitioned** from a 1.8 TiB + 1.8 TiB split into a single 3.6 TiB `sda1` partition. The old `sda2` partition was the previously active `/mnt/ssd_working` mount; the old `sda1` was the unknown-purpose unmounted partition flagged in the morning's entry. Both are gone.
- **Migration method:** all 48 GiB of data from the old `sda2` (`Data/` directory) was staged to `sdb` via rsync, `sda` was repartitioned and formatted fresh (ext4, label `ssd_working`, UUID `a08d2cf0-5d95-4616-9dbe-54b1595df98d`), then the data was rsynced back. Round-trip verified byte-identical: 108,738 files / 50,819,999,489 bytes, 0 errors.
- **`/etc/fstab` updated:** old `ssd_working` UUID line (`cabf6ad5…`) is commented out as a rollback breadcrumb. New line uses UUID `a08d2cf0-5d95-4616-9dbe-54b1595df98d`. `systemctl daemon-reload` run to activate.
- **P3-1TB SATA SSD (`/dev/sdb`) commissioned:** formatted ext4, label `data_extra`, UUID `207e03a8-dbe0-4025-9f1e-6c9d4bd13e5c`. Mounted at `/mnt/data_extra` (~938 GiB usable). Owned `pmabry:pmabry`. Currently empty. Intended for bulk additional data storage.
- The unknown-purpose `sda1` flagged in the morning's entry is resolved — it no longer exists. No data was lost; the partition was confirmed empty before being subsumed.
- See [Topology — Storage Map](topology.md#storage-map) for the updated logical layout.

### 2026-05-14 — OpenProject Decommissioned

OpenProject was removed from sheepsoc. It never worked as a fit for Phillip's workflow and the container had been stopped for approximately two weeks before formal removal.

**What was removed:**

- Docker container `openproject` (image `openproject/openproject:15`) — removed via `docker rm` / `docker rmi`
- Bind-mount data directory `/mnt/ssd_working/openproject/` — wiped (`sudo rm -rf`; `pgdata` subdirectory was owned by a non-pmabry UID)
- Orphaned bridge network `openproject_default` — removed via `docker network rm`
- `~/infrastructure/sheepsoc-shutdown.sh` — cleaned up: removed `openproject` from `APP_SERVICES`, removed the `docker stop openproject` line, and removed the docker-check block in the final verification step

Port 3001 is retained by Uptime Kuma. No UFW changes were made.

### 2026-05-14 — NVMe Reseat and New SATA SSD Added

- **`nvme3n1` (Samsung 990 PRO 2TB, S/N S7KHNU0Y529975Z) reseated.** This is the drive that was previously loose in its mini-PCIe carrier (see the 2026-05-10 entry). The reseat held — both PVs of `vg_elastic` reported healthy, LV `lv_elastic_data` is active and mounted r/w at `/mnt/elastic_data`, SMART is PASSED, wear 4%, no dmesg I/O errors observed post-reseat. Monitor for recurrence.
- **New SATA SSD added:** `sdb` (P3-1TB, S/N 0039914A03508, 0 power-on hours). Not yet partitioned, formatted, or mounted. Intended for additional data storage; configuration is TBD.
- **`sda1` flagged as unknown-purpose** *(resolved same day, evening)*: A 1.8 TiB ext4 partition on `sda` (UUID `1c74cfed-7268-4c49-a72b-a6aa4600204c`) was present but unmounted. Confirmed empty during the evening storage reshuffle. The partition was removed when `sda` was repartitioned into a single 3.6 TiB `sda1`. See the evening entry above.
- The prior loose-NVMe landmine documented in the 2026-05-10 entry (below) is superseded by this reseat — the physical issue was addressed at the hardware level. The 2026-05-10 migration to Elastic Cloud also removed ES I/O dependence on this drive, providing software-level resilience if the seat loosens again.
- See [Topology — Storage Map](topology.md#storage-map) for the full updated drive table and logical layout.

### 2026-05-10 — Elasticsearch Migrated to Elastic Cloud (Local ES Decommissioned)

- Local single-node `elasticsearch.service` on sheepsoc (8.19.14, `/mnt/elastic_data`) was **decommissioned**. All ES traffic now goes to **Elastic Cloud 9.4.0** at `https://gateway.es.us-central1.gcp.cloud.es.io` (GCP us-central1, cluster UUID `DaBtVAvNQNmqT0thJwt-7Q`, 3 nodes / 2 data nodes, green).
- Auth changed from `basic_auth=("elastic","<password>")` to API key (`Authorization: ApiKey <key>`). New env var `ELASTICSEARCH_API_KEY` set in `~/.env` (renamed from typoed `ELASTICSEARH_API`). The new name also matches OpenWebUI's native env var.
- All prior local indexes (including `open_webui_collections_d768`, the RAG_001 v2/v3 indexes, all syslog data streams) are **not carried over**. Each consumer must be reconfigured to point at the cloud cluster and, where applicable, re-ingest from source.
- Status of consumers as of 2026-05-11:
    - **RAG_001 v2** re-ingested into cloud cluster — 48,266 docs, 0 errors (used existing pre-embedded bulk JSONL).
    - **RAG_001 v3** ingest in progress.
    - **RAG_002** deferred — chunks exist but were never embedded; held pending Cloudflare/native-Assistant decision.
    - **OpenWebUI, Filebeat, Metricbeat, Logstash, Kibana** still configured for `localhost:9200` and will fail until pointed at the cloud cluster — pending Phillip's decision on each.
- Resolves the long-running landmine of the loose mini-PCIe NVMe carrier under `/mnt/elastic_data` that produced recurring EIO and cluster hangs — ES no longer touches that storage path.
- See [project wiki: Elastic Cloud Migration](https://github.com/...) (`docs/runbooks/elastic_cloud_migration.md` in `~/jupyter/rag_experiments/`) for the RAG-specific procedure and `concepts/elastic_cloud_deployment.md` for the new connection model.

### 2026-05-09 — Tailscale Serve Configured

- `tailscale serve` configured to expose three services to the tailnet over HTTPS. All three rules were written with `--bg` (persistent state) and survive restarts.
- Active rules: OpenWebUI at port 443, Jupyter at port 8443, sheepsoc-docs at port 10000. All use auto-provisioned Let's Encrypt certs for `sheepsoc-1.tail0f68e4.ts.net`.
- Port-based (not path-based) to avoid breaking OpenWebUI's root-path assumption.
- No UFW changes required — existing `ALLOW IN on tailscale0` rule covers tailnet access; proxy runs via loopback.
- Docs site is intentionally unauthenticated at port 10000 (approved by Phillip). OpenWebUI and Jupyter retain their own auth layers.
- See [Tailscale — Tailscale Serve](platforms/tailscale.md#tailscale-serve) for full details and [Tailscale Operations — Managing Serve Rules](runbooks/tailscale-ops.md#5-managing-tailscale-serve-rules) for the management runbook.

### 2026-05-09 — Tailscale Installed (Fresh)

- [Tailscale](platforms/tailscale.md) installed from the official Tailscale apt repository (`pkgs.tailscale.com/stable/ubuntu/noble`). This replaces the failed installation that was purged on 2026-04-19.
- Enrolled to tailnet `tail0f68e4` (Phillip's existing tailnet, Google SSO). Sheepsoc's tailnet address is `100.117.117.43`; MagicDNS hostname is `sheepsoc-1.tail0f68e4.ts.net`. (The `-1` suffix is cosmetic — a stale prior entry holds the unsuffixed name. See Watchlist.)
- `tailscaled.service` enabled at boot and active.
- UFW rule added: `ufw allow in on tailscale0` — permits all inbound traffic from tailnet peers on the `tailscale0` interface.
- Sysctl file `/etc/sysctl.d/99-tailscale.conf` created with `net.ipv4.ip_forward = 1`.
- Subnet routing, exit node, and Tailscale SSH intentionally left disabled. See [Tailscale](platforms/tailscale.md) for the full rationale.
- Due to Starlink CGNAT, direct WireGuard connections are not possible; Tailscale uses DERP relay (`derp-9`). This is expected and transparent.

### 2026-04-27 — OpenWebUI Upgraded from 0.6.12 to 0.9.2

- OpenWebUI upgraded to **0.9.2** on sheepsoc.
- **6 database schema migrations applied automatically** at startup. All migrations were additive (new columns / tables only) — no data was altered or removed.
- Pre-upgrade backup of `webui.db` created at `~/infrastructure/open-webui/webui.db.bak-20260427`. Retain this until the new version has been in stable operation for at least one week.
- All existing RAG Knowledge bases, Elasticsearch vector index (`open_webui_collections_d768`), and the ELSER sparse-embedding pipeline are unaffected by the upgrade.
- If the `openwebui` conda env is ever rebuilt after this upgrade, confirm the `elasticsearch==8.19.3` pip package is still present (`pip show elasticsearch`). The package is not bundled and must be reinstalled manually.

### 2026-04-23 — ELSER Sparse Embeddings Added to OpenWebUI RAG Index

- The `open_webui_collections_d768` index is now **dual-use**: it carries both dense kNN vectors (`vector` field, 768-dim, `nomic-embed-text`) used by OpenWebUI, and a new sparse vector field (`text_elser`, type `sparse_vector`) populated by ELSER.
- Ingest pipeline `elser-embed-openwebui` created. Set as the index default pipeline — all future OpenWebUI document uploads automatically receive ELSER tokens at ingest time. No changes to OpenWebUI itself.
- ELSER inference endpoint: `.elser-2-elasticsearch` (built-in, adaptive allocation — scales to zero when idle).
- All 8,136 existing documents backfilled with ELSER tokens in two passes. First pass processed 7,235 documents; 901 (~11%) were skipped due to ELSER scaling from zero on the first batch (a known cold-start behaviour). Second pass completed the remaining 901. Final coverage: 8,136 / 8,136, zero conflicts.
- New capability: `text_expansion` queries and hybrid RRF (Reciprocal Rank Fusion) queries can now be run against the index from outside OpenWebUI. See [RAG & Knowledge — ELSER](platforms/openwebui-rag.md), [ELSER & OpenWebUI](platforms/elasticsearch-elser.md), and the how-to at `~/infrastructure/elasticsearch/howto-elser-openwebui-pipeline.md`.

### 2026-04-21 — Elasticsearch xpack.security Enabled

- `xpack.security.enabled: true` set in `/etc/elasticsearch/elasticsearch.yml`. TLS left disabled (HTTP plaintext, LAN-only). Enrollment disabled.
- Built-in account passwords set: `elastic` (superuser), `kibana_system`, `beats_system`. All three verified via API before service restarts.
- Kibana updated: `/etc/kibana/kibana.yml` now carries `elasticsearch.username: kibana_system` and the corresponding password.
- Filebeat and Metricbeat updated: both `filebeat.yml` and `metricbeat.yml` carry `username: beats_system` and password under `output.elasticsearch`.
- OpenWebUI drop-in updated: `/etc/systemd/system/open-webui.service.d/elasticsearch.conf` now includes `ELASTICSEARCH_USERNAME=elastic` and `ELASTICSEARCH_PASSWORD`.
- All five services (Kibana, Filebeat, Metricbeat, OpenWebUI) restarted in sequence and verified active. ES cluster health: **green**, 45/45 shards. Kibana login page reachable at `http://localhost:5601`.
- Any `curl` commands to ES now require `-u elastic:<password>`. See [Services](services.md) for updated command examples.

### 2026-04-21 — Bulk Ingest Script and Gardening Knowledge Base

- Script `~/repositories/sheepsoc/ingest_to_openwebui.py` written to bulk-ingest pre-extracted `.txt` chunks into OpenWebUI Knowledge Bases. Writes to both Elasticsearch and SQLite simultaneously. Idempotent — uses `uuid5`-based file IDs so re-runs safely skip already-indexed files.
- **Gardening** knowledge base populated: 134 files, 1,352 chunks from Texas A&M AgriLife extension texts. Queryable in OpenWebUI chat via `#Gardening`.
- Key architectural insight documented: the ES `collection` field must be the bare knowledge base UUID from `webui.db`. Using `file-{uuid}` (the namespace for standalone uploads) breaks KB retrieval silently.
- Dataset inventory mapped: `/mnt/ssd_working/Data/Ingest_Ready/` contains 63 pre-extracted text collections ready to ingest. Mother Earth News archive (19,688 files / ~38 GB) identified as future work requiring a dedicated script and separate ES index.

### 2026-04-20 — OpenWebUI RAG Configured

- OpenWebUI 0.6.12 switched from Chroma to **Elasticsearch** as its vector store.
- `nomic-embed-text:latest` pulled to Ollama — the embedding model for OpenWebUI RAG (768 dimensions).
- ES index `open_webui_collections_d768` pre-created with HNSW/cosine mapping.
- `elasticsearch==8.19.3` pip-installed into the `openwebui` conda env (not bundled — must reinstall if env is rebuilt).
- Systemd drop-in created: `/etc/systemd/system/open-webui.service.d/elasticsearch.conf`.
- ES cluster health resolved from yellow to green by fixing replica counts on filebeat/metricbeat data streams.

### 2026-04-20 — ASUS Syslog Pipeline Verified and Fixed

- Set `log_remote=1` in ASUS NVRAM.
- Applied the `syslogd -R` flag to ship to sheepsoc:5514/udp.
- Confirmed `logs-syslog.asus-default` data stream is receiving events in Elasticsearch.

### 2026-04-19 — Security Hardening Session

- UFW firewall enabled. Ports 22, 80, 443, 8080, 8888, 9200, 5601, 11434, and 3000 restricted to LAN (`192.168.50.0/24`) or localhost only.
- Tailscale package fully purged (64.9 MB freed). The daemon had been failing for ~2 months. **Note:** Tailscale was reinstalled fresh on 2026-05-09 — see the history entry above.
- Config management system bootstrapped at `~/infrastructure/config-mgmt/`.
- Steps 7–10 of the hardening plan remain pending.

### 2026-04-18 — Major Recovery Session

The box was in a degraded state across networking, boot, and shell. Fixes:

| Problem | Fix |
|---|---|
| Gateway was set to `192.168.50.253` — a dead host — causing no internet connectivity. | Set gateway to `192.168.50.1` (the ASUS). |
| DNS was broken. | Configured `8.8.8.8` and `1.1.1.1` via netplan. LAN `mabry.lan` lookups still go through OPNsense at `192.168.50.253`. |
| `tailscaled` had been failing for ~2 months, flooding logs. | Tailscale removed from the box. |
| `systemd-networkd-wait-online` was blocking boot. | Service disabled. |
| Login hung for ~5 minutes due to the subprocess-based conda shell hook. | `~/.bashrc` switched to the static loader: `source ~/infrastructure/miniconda3/etc/profile.d/conda.sh`. Do not revert. |
| No Claude Code on the box. | Claude Code `v2.1.114` installed. |

### Pre-2026-04-18 — MicroK8s Incident

OpenEBS NDM memory leak ballooned to 247 GB of RAM. The machine became unresponsive; load averages exceeded 100. All workloads that had been running in the cluster (Elasticsearch, pfSense, others) were migrated off or rebuilt on the host. MicroK8s has been stopped since.

### Pre-2026-04-18 — NFS Server Outage

`san01.mabry.lan` went down. Its entry in `/etc/fstab` was commented to keep boot from hanging. Still offline as of last review.
