# Topology

**Purpose:** Network map, host layout, storage, and data-flow diagrams.

## Network Topology

Sheepsoc lives on a flat home LAN behind an ASUS router that does DHCP and upstream NAT, with OPNsense acting as the internal DNS resolver for `mabry.lan`. There is no public inbound port-forwarding — LAN services are reached directly on the LAN, and remotely via Tailscale (see [Remote Access — Tailscale](#remote-access-tailscale) below).

```
                      ┌─────────────────────────────┐
                      │  Internet (Starlink · CGNAT) │
                      └──────────────┬──────────────┘
                                     │ WAN
                      ┌──────────────┴──────────────┐
                      │  ASUS RT-AX5400             │
                      │  192.168.50.1               │
                      │  DHCP · NAT · gateway       │
                      │  SSH :1024  ·  syslog → LS  │
                      └──────────────┬──────────────┘
                                     │ LAN 192.168.50.0/24
        ┌────────────────────────────┼────────────────────────────┐
        │                            │                            │
┌───────┴────────┐         ┌─────────┴─────────┐          ┌───────┴────────┐
│ OPNsense       │         │ sheepsoc           │          │ Printer        │
│ .253           │         │ .100 static        │          │ .213           │
│ DNS · FW       │         │ Xeon · 251G · GPU  │          │                │
│ *.mabry.lan    │         │ Ubuntu 24.04       │          │                │
│ syslog → LS    │         │ all services       │          │                │
└────────────────┘         └──────┬────────────┘          └────────────────┘
                                  │ tailscale0 (WireGuard)
                           100.117.117.43
                                  │ (outbound to Tailscale coordination
                                  │  server · DERP relay for remote peers)
                                  │
                    [ Tailscale peers · tail0f68e4 tailnet ]
                    phillips-macbook-pro-3  100.88.90.20

 DNS chain : client → OPNsense (192.168.50.253) → 8.8.8.8 / 1.1.1.1 upstream
 Logs      : ASUS + OPNsense → sheepsoc:5514/udp (Logstash) → Elasticsearch
 Remote    : Tailscale WireGuard overlay · no inbound port-forwarding needed
```

| Key | Value |
|---|---|
| Gateway | 192.168.50.1 · ASUS RT-AX5400 · SSH on port 1024, key auth |
| Firewall/DNS | 192.168.50.253 · OPNsense · resolves `*.mabry.lan` |
| DNS upstream | 8.8.8.8 · 1.1.1.1 (configured in netplan) |
| sheepsoc | 192.168.50.100 · static on `eno1` · `sheepsoc.mabry.lan` · Tailscale `100.117.117.43` |
| Printer | 192.168.50.213 · admin console |
| Scope | LAN + Tailscale remote access (WireGuard overlay, no inbound WAN port-forwarding) |

## Remote Access — Tailscale

Sheepsoc's internet uplink is Starlink, which uses CGNAT (Carrier-Grade NAT). The ASUS RT-AX5400 does not hold a routable public IP address, making inbound port-forwarding from the public internet impossible at the ASUS layer. Remote access is provided instead by [Tailscale](platforms/tailscale.md), a WireGuard-based mesh VPN.

Tailscale was installed on 2026-05-09 and enrolled to the `tail0f68e4` tailnet (Google SSO). The key configuration choices are:

| Setting | Value | Rationale |
|---|---|---|
| Tailscale IPv4 | `100.117.117.43` | Stable tailnet address for sheepsoc |
| MagicDNS | `sheepsoc-1.tail0f68e4.ts.net` | Human-readable hostname for the tailnet (see [Known Issues](known-issues.md) for the `-1` suffix) |
| Subnet routing | Off | Phillip only needs to reach sheepsoc itself; no LAN subnet advertisement needed |
| Exit node | Off | Would route remote-device internet traffic through Starlink; undesirable |
| Tailscale SSH | Off | Existing hardened OpenSSH is preserved; SSH works normally over the Tailscale tunnel |
| UFW rule | `allow in on tailscale0` | Permits all authenticated tailnet traffic to reach sheepsoc services |

Due to Starlink CGNAT, direct peer-to-peer WireGuard connections to sheepsoc from external devices are not possible. Tailscale falls back to its DERP relay network (`derp-9`, auto-selected), which is transparent and expected for this network configuration.

To reach any sheepsoc service remotely, substitute `100.117.117.43` for `192.168.50.100` in any URL or SSH command. OpenWebUI, Jupyter, and the docs site also have dedicated HTTPS URLs via Tailscale Serve — see [Services — Tailnet HTTPS URLs](services.md#tailnet-https-urls-tailscale-serve--configured-2026-05-09).

## Host Layout

Everything runs on sheepsoc itself as systemd units — no containers for the primary stack. MicroK8s is installed but stopped pending a rebuild (see [Known Issues](known-issues.md)).

```
sheepsoc  (192.168.50.100 · tailscale 100.117.117.43)
├─ systemd services
│  ├─ sheepsoc-landing.service  → :80     # docs site (MkDocs, Python http.server)
│  ├─ ollama.service            → :11434  # LLM inference, uses GPU
│  ├─ open-webui                → :8080   # chat UI + RAG (conda: openwebui, Py 3.11)
│  ├─ jupyter.service           → :8888   # notebook dir ~/repositories/sheepsoc
│  ├─ elasticsearch             → :9200   # DECOMMISSIONED 2026-05-10 — moved to Elastic Cloud (GCP us-central1, 9.4.0)
│  ├─ kibana                    → :5601   # log/metrics UI
│  ├─ logstash                  → :5514/udp # syslog ingest from ASUS + OPNsense
│  ├─ filebeat                  → ES      # local logs → ES
│  ├─ metricbeat                → ES      # local metrics → ES
│  ├─ vikunja                   → :3000   # kanban / task mgmt
│  ├─ matrix-bot.service                 # E2EE Matrix bot
│  ├─ tailscaled.service        → tailscale0 (100.117.117.43) # WireGuard mesh VPN
│  ├─ ssh.service               → :22    # key auth only
│  └─ cron.service                       # scheduled tasks
├─ conda environments  (~/infrastructure/miniconda3/)
│  ├─ sheepsoc         # Python 3.12 — legacy CLI RAG prototype + Jupyter kernel
│  ├─ openwebui        # Python 3.11 — OpenWebUI service (elasticsearch pip-installed)
│  └─ datawrangler     # data work
├─ docker compose stacks
│  └─ /mnt/ssd_working/emulatorjs/      # RomM + MariaDB (romm :3080, db internal)
├─ applications
│  ├─ ~/repositories/sheepsoc/          # legacy CLI RAG prototype + bulk-ingest script
│  ├─ ~/repositories/sheepsoc_refactor/ # refactor in progress
│  ├─ ~/repositories/embedding_testing/ # embedding experiments
│  └─ ~/repositories/pytorch/           # PyTorch experiments
└─ stopped / dormant
   ├─ snap.microk8s.*  [stopped, do not start — see Known Issues]
   └─ NFS mount to san01.mabry.lan [fstab commented, server down]
```

## Hardware

| Component | Specification |
|---|---|
| CPU | Dual Intel Xeon E5-2680 v4 @ 2.40 GHz · 56 logical / 28 physical · 2 NUMA nodes |
| RAM | 251 GB |
| GPU | NVIDIA GeForce RTX 5060 Ti · 16 GB VRAM · driver 570.169 · CUDA 12.8 |
| NIC | `eno1` · 192.168.50.100 static (netplan) |

The dual-socket CPU means two NUMA nodes. For most of what this box does (Ollama, ES, Python) that is invisible, but be aware it exists if you ever start pinning processes.

## Storage Map

*Last updated: 2026-05-14 (evening) — PNY 4TB SSD repartitioned to single partition; P3-1TB formatted and mounted. See [Known Issues — history](known-issues.md#2026-05-14-evening-storage-reshuffle-pny-4tb-repartitioned-and-p3-1tb-commissioned) for details.*

### Physical Drives

| Linux dev | Bus | Model | Capacity | Serial | SMART | Wear | Power-on hrs |
|---|---|---|---|---|---|---|---|
| `nvme0n1` | NVMe | T-FORCE TM8FFE001T | 1.02 TB | TPBF2503060040100397 | PASSED | 36% | — |
| `nvme1n1` | NVMe | Samsung 990 PRO 2TB | 2.00 TB | S7KHNU0Y529972M | PASSED | 0% | — |
| `nvme2n1` | NVMe | T-FORCE TM8FFE001T | 1.02 TB | TPBF2503060040100087 | PASSED | 3% | — |
| `nvme3n1` | NVMe | Samsung 990 PRO 2TB | 2.00 TB | S7KHNU0Y529975Z | PASSED | 4% | — |
| `sda` | SATA | PNY CS900 4TB SSD | 4.00 TB | PNY250725021101043D7 | PASSED | — | 7316 |
| `sdb` | SATA | P3-1TB | 1.02 TB | 0039914A03508 | PASSED | — | 0 |

!!! note
    `sdb` (P3-1TB) was added 2026-05-14 with 0 power-on hours. Formatted ext4 (label `data_extra`) and mounted at `/mnt/data_extra` on 2026-05-14 (evening). Currently empty.

!!! warning
    `nvme3n1` (Samsung 990 PRO, S/N …975Z) is the drive in `vg_elastic` that was previously loose in its mini-PCIe carrier. It was reseated 2026-05-14 and came back cleanly. Monitor for recurrence of I/O errors in `dmesg` or `journalctl -u elasticsearch`. A strategic decision on how to mitigate the structural risk to the linear `vg_elastic` LV is pending — see [Known Issues — Watchlist](known-issues.md#watchlist) for the three options (mirror, demote, hardware fix) and the open question that gates the decision.

### Logical Layout and Mounts

| Mount | Device | Filesystem | Size | Used | Purpose |
|---|---|---|---|---|---|
| `/boot/efi` | `nvme0n1p1` | vfat | 1 GiB | — | EFI system partition |
| `/boot` | `nvme0n1p2` | ext4 | 2 GiB | — | Kernel and initrd |
| `/` | `nvme0n1p3` → LV `ubuntu-lv` in `ubuntu-vg` | ext4 | 936 GiB | 23% | OS + home + applications |
| `/mnt/nvme_working` | `nvme2n1` | ext4 | 938 GiB | ~1% | General working storage (hot NVMe) |
| `/mnt/elastic_data` | LV `lv_elastic_data` in `vg_elastic` | ext4 | 3.6 TiB | 3% (87 GiB) | Elasticsearch data — see note below |
| `/mnt/ssd_working` | `sda1` (single partition spanning full disk) | ext4 | 3.6 TiB | 2% (48 GiB) | General working storage — label `ssd_working`, UUID `a08d2cf0-5d95-4616-9dbe-54b1595df98d` |
| `/mnt/data_extra` | `sdb1` | ext4 | ~938 GiB | ~0% (empty) | Bulk additional data storage — label `data_extra`, UUID `207e03a8-dbe0-4025-9f1e-6c9d4bd13e5c` |

**LVM volume groups:**

| VG | PVs | Total | LV | Mount |
|---|---|---|---|---|
| `ubuntu-vg` | `nvme0n1p3` | ~936 GiB | `ubuntu-lv` | `/` |
| `vg_elastic` | `nvme1n1`, `nvme3n1` | 3.64 TiB | `lv_elastic_data` | `/mnt/elastic_data` |

!!! note "Elastic Cloud"
    As of 2026-05-10, the local `elasticsearch.service` on sheepsoc is **decommissioned**. The `/mnt/elastic_data` volume is physically healthy but no longer actively written by ES. All ES traffic now goes to Elastic Cloud (GCP us-central1). See [Known Issues — history](known-issues.md#2026-05-10-elasticsearch-migrated-to-elastic-cloud-local-es-decommissioned) for the migration record.

## Data Flow — RAG + Observability

### OpenWebUI RAG Path (Browser-Based)

```
[ Browser / LAN client ]
   │  chat message + uploaded doc or Knowledge collection
   ▼
┌────────────────────────────┐
│ OpenWebUI 0.9.2            │  :8080 · conda env: openwebui (Py 3.11)
└──────┬─────────────────────┘
       │  embed query / doc chunks
       ▼
┌────────────────────────────┐
│ Ollama (local :11434)      │  nomic-embed-text:latest · 768d
└──────┬─────────────────────┘
       │  vector
       ▼
┌────────────────────────────┐
│ Elasticsearch (local :9200)│  index: open_webui_collections_d768 · HNSW/cosine
└──────┬─────────────────────┘
       │  retrieved chunks
       ▼
┌────────────────────────────┐
│ Ollama (local :11434)      │  LLM inference · uses RTX 5060 Ti
└──────┬─────────────────────┘
       │  answer (streamed)
       ▼
[ Browser ]
```

### Legacy CLI RAG Query Path (Deprecated — For Reference Only)

```
[ CLI / notebook ]
   │  query string
   ▼
┌────────────────────────────┐
│ sheepsoc.cli               │
└──────────────┬─────────────┘
               │
               ▼
┌────────────────────────────┐      embed query
│ pipeline.py (orchestrator) │ ──▶ HuggingFace e5-large-v2
└──────┬─────────────────────┘
       │  hybrid search (KNN + BM25, RRF)
       ▼
┌────────────────────────────┐
│ Elasticsearch (local :9200)│  index: intfloat_e5_large_v2__512__125
└──────┬─────────────────────┘
       │  top-K candidates
       ▼
┌────────────────────────────┐
│ CrossEncoder reranker      │
└──────┬─────────────────────┘
       │  top-N reranked
       ▼
┌────────────────────────────┐
│ prompt_builder.py          │  assembles context + template
└──────┬─────────────────────┘
       │  final prompt
       ▼
┌────────────────────────────┐
│ Ollama (local :11434)      │  LLM: dolphin3:latest (default)
│ RTX 5060 Ti / 16 GB VRAM  │
└──────┬─────────────────────┘
       │  answer
       ▼
[ user ]
```

### Observability Flow

```
ASUS RT-AX5400  ──syslog udp──▶  sheepsoc:5514  ──▶  Logstash
OPNsense        ──syslog udp──▶  sheepsoc:5514  ──▶  Logstash
                                                        │
                                                        ▼
sheepsoc logs    ──filebeat──────────────────────▶  Elasticsearch
sheepsoc metrics ──metricbeat────────────────────▶  (:9200)
                                                        │
                                                        ▼
                                                    Kibana  (:5601)

Data streams:
  logs-syslog.asus-default      ← ASUS
  logs-syslog.opnsense-default  ← OPNsense
```

!!! note "Verified"
    The ASUS → Logstash syslog pipeline was re-verified on 2026-04-20 after setting `log_remote=1` in NVRAM and applying the `syslogd -R` flag.
