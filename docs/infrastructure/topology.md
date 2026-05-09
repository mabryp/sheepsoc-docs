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
| MagicDNS | `sheepsoc.tail0f68e4.ts.net` | Human-readable hostname for the tailnet |
| Subnet routing | Off | Phillip only needs to reach sheepsoc itself; no LAN subnet advertisement needed |
| Exit node | Off | Would route remote-device internet traffic through Starlink; undesirable |
| Tailscale SSH | Off | Existing hardened OpenSSH is preserved; SSH works normally over the Tailscale tunnel |
| UFW rule | `allow in on tailscale0` | Permits all authenticated tailnet traffic to reach sheepsoc services |

Due to Starlink CGNAT, direct peer-to-peer WireGuard connections to sheepsoc from external devices are not possible. Tailscale falls back to its DERP relay network (`derp-9`, auto-selected), which is transparent and expected for this network configuration.

To reach any sheepsoc service remotely, substitute `100.117.117.43` for `192.168.50.100` in any URL or SSH command.

## Host Layout

Everything runs on sheepsoc itself as systemd units — no containers for the primary stack. MicroK8s is installed but stopped pending a rebuild (see [Known Issues](known-issues.md)).

```
sheepsoc  (192.168.50.100 · tailscale 100.117.117.43)
├─ systemd services
│  ├─ sheepsoc-landing.service  → :80     # docs site (MkDocs, Python http.server)
│  ├─ ollama.service            → :11434  # LLM inference, uses GPU
│  ├─ open-webui                → :8080   # chat UI + RAG (conda: openwebui, Py 3.11)
│  ├─ jupyter.service           → :8888   # notebook dir ~/repositories/sheepsoc
│  ├─ elasticsearch             → :9200   # 8.19.14 · single-node "sheepsoc" · /mnt/elastic_data
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

| Mount | Device Class | Size | Purpose |
|---|---|---|---|
| `/` | NVMe (LVM) | 936 GB (202 GB used) | OS + home + most applications |
| `/mnt/ssd_working` | SSD | 1.8 TB | General working storage |
| `/mnt/nvme_working` | NVMe | 1.8 TB | General working storage (hot) |
| `/mnt/k8s_ssd_1` | SSD | 1.8 TB | Reserved for k8s rebuild |
| `/mnt/k8s_nvme_1` | NVMe | ~900 GB | Reserved for k8s rebuild |
| `/mnt/k8s_nvme_2` | NVMe | ~900 GB | Reserved for k8s rebuild |
| `/mnt/k8s_nvme_3` | NVMe | ~900 GB | Reserved for k8s rebuild |

!!! warning
    Do not place long-term data on the `k8s_*` mounts — they are earmarked for the MicroK8s rebuild and will be reformatted / reprovisioned when that happens.

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
