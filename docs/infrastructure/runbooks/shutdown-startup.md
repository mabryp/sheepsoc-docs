# Shutdown & Startup Runbook

**Purpose:** Step-by-step procedures for planned and emergency shutdowns, post-boot verification, and troubleshooting common startup failures.

| Key | Value |
|---|---|
| Hostname | `sheepsoc` |
| IP | `192.168.50.100` |
| OS | Ubuntu 24.04.3 LTS |

## Services Covered

This runbook operates the following services. See each page for configuration details and health check commands.

- [Services](../services.md) — full service catalog with ports and unit names
- [OpenWebUI & RAG](../platforms/openwebui-rag.md) — depends on Elasticsearch; stop first, start last
- [Elasticsearch & ELSER](../platforms/elasticsearch-elser.md) — must be running before OpenWebUI, Kibana, and the Matrix bot
- [Matrix Bot](../platforms/matrix-bot.md) — stopped before Elasticsearch in planned shutdown
- [Conda](../platforms/conda.md) — relevant for the OpenWebUI ImportError troubleshooting step

## 1. Planned Shutdown

Follow this sequence whenever a deliberate shutdown is required (hardware maintenance, power work, etc.). The order matters: services that depend on Elasticsearch should be stopped before Elasticsearch itself.

1. **Stop AI services first.**

    ```bash
    # Stop OpenWebUI (depends on Elasticsearch for vector queries)
    pmabry@sheepsoc:~$ sudo systemctl stop open-webui.service

    # Stop Jupyter
    pmabry@sheepsoc:~$ sudo systemctl stop jupyter.service

    # Stop the Matrix bot
    pmabry@sheepsoc:~$ sudo systemctl stop matrix-bot.service
    ```

2. **Stop observability services.**

    ```bash
    pmabry@sheepsoc:~$ sudo systemctl stop filebeat.service metricbeat.service
    ```

3. **Stop Elasticsearch and Kibana.**

    ```bash
    pmabry@sheepsoc:~$ sudo systemctl stop kibana.service
    pmabry@sheepsoc:~$ sudo systemctl stop elasticsearch.service
    ```

    Wait for Elasticsearch to finish flushing its translog before proceeding. This normally completes within 10–30 seconds.

4. **Stop Ollama.**

    ```bash
    pmabry@sheepsoc:~$ sudo systemctl stop ollama.service
    ```

5. **Shut down the OS.**

    ```bash
    pmabry@sheepsoc:~$ sudo shutdown -h now
    ```

## 2. Emergency Shutdown

If the machine must be shut down immediately (power emergency, thermal event, hardware fault), the `shutdown` command is still preferred over a hard power cut. The kernel will attempt a clean unmount of filesystems even on an accelerated shutdown.

```bash
# Immediate shutdown — OS will attempt clean unmount before powering off
pmabry@sheepsoc:~$ sudo shutdown -h now
```

If the machine is completely unresponsive and a remote SSH session cannot be established, a hard power cut is the last resort. Be aware of the following risks:

| Risk | Detail |
|---|---|
| Elasticsearch translog | Elasticsearch may need to replay its translog on the next boot. This adds time to startup but data is not lost (as long as the data volume is intact). |
| Filesystem consistency | An unclean unmount may trigger fsck on the next boot. This is handled automatically by the OS. |
| NVMe wear | Repeated hard power cuts can accelerate NVMe wear, particularly for the root volume. |

Before cutting power, check whether you can at least run a sync to flush filesystem buffers:

```bash
# Flush all pending writes to disk — run this if you have even 5 seconds
pmabry@sheepsoc:~$ sync
```

## 3. Startup & Post-Boot Verification

After booting, confirm all services are up and healthy before relying on the system. Work through these checks in order.

1. **Confirm network connectivity.**

    ```bash
    pmabry@sheepsoc:~$ ping -c 3 8.8.8.8
    pmabry@sheepsoc:~$ ip addr show eno1 | grep 'inet '
    # Expected: 192.168.50.100/24
    ```

2. **Check all expected services are running.**

    ```bash
    pmabry@sheepsoc:~$ systemctl is-active ollama elasticsearch kibana open-webui jupyter filebeat metricbeat matrix-bot
    # Each service should report: active
    ```

    If any service is not active, check its logs:

    ```bash
    pmabry@sheepsoc:~$ journalctl -u <service>.service -n 50 --no-pager
    ```

3. **Verify Elasticsearch cluster health.**

    ```bash
    # Expects: {"status":"green"} or {"status":"yellow"} — both are acceptable
    pmabry@sheepsoc:~$ curl -s -u elastic:elastic123 http://localhost:9200/_cluster/health | python3 -m json.tool
    ```

    Yellow is normal on a single-node cluster (unplaceable replicas). Red indicates unassigned primary shards — investigate immediately.

4. **Verify Ollama and GPU.**

    ```bash
    # Confirm Ollama API is responding
    pmabry@sheepsoc:~$ curl -s http://localhost:11434/api/tags | python3 -m json.tool | head -5

    # Confirm the GPU is visible and CUDA is active
    pmabry@sheepsoc:~$ nvidia-smi
    ```

5. **Verify OpenWebUI is reachable.**

    Open `http://192.168.50.100:8080` in a browser from any LAN device. Log in and confirm the interface loads.

6. **Check storage mounts.**

    ```bash
    pmabry@sheepsoc:~$ df -h | grep -E 'mnt|/dev'
    ```

    All four secondary volumes should be mounted:

    | Mount point | Expected size |
    |---|---|
    | `/mnt/ssd_working` | ~1.8 TB |
    | `/mnt/nvme_working` | ~1.8 TB |
    | `/mnt/k8s_ssd_1` | ~1.8 TB |
    | `/mnt/k8s_nvme_1` (and `_2`, `_3`) | ~900 GB each |

    !!! warning "NFS"
        The `san01.mabry.lan` NFS share is **not mounted**. Its fstab entry is commented out because the NFS server is currently down. Do not uncomment it until the NFS server is restored.

7. **Check for DKMS or PCIe errors.**

    ```bash
    pmabry@sheepsoc:~$ sudo dmesg -T | grep -iE 'error|fail|warn' | grep -iv 'corrected' | head -30
    ```

    A clean boot should produce no unexpected errors. Correctable PCIe errors (see Troubleshooting) are noted separately.

## 4. Service Quick Reference

| Service | Unit | Port | Data Location |
|---|---|---|---|
| Ollama | `ollama.service` | 11434 | `~/.ollama/` |
| Elasticsearch | `elasticsearch.service` | 9200 | `/mnt/elastic_data/` |
| Kibana | `kibana.service` | 5601 | — |
| OpenWebUI | `open-webui.service` | 8080 | `~/infrastructure/open-webui/` |
| Jupyter | `jupyter.service` | 8888 | `~/repositories/sheepsoc/` |
| Filebeat | `filebeat.service` | — | — |
| Metricbeat | `metricbeat.service` | — | — |
| Matrix bot | `matrix-bot.service` | — | `~/repositories/matrix-bot/` |
| Vikunja | `vikunja.service` | 3000 | — |

## 5. Troubleshooting

### Elasticsearch won't start

| Symptom | Action |
|---|---|
| `journalctl -u elasticsearch` shows `max virtual memory areas vm.max_map_count [65530] is too low` | `sudo sysctl -w vm.max_map_count=262144` — this is normally set persistently in `/etc/sysctl.d/` but may not apply after certain kernel updates. |
| `journalctl -u elasticsearch` shows `failed to obtain node locks` | Another Elasticsearch process is still running. Check `ps aux | grep elasticsearch` and kill it, then retry. |
| Data volume not mounted | Elasticsearch data lives at `/mnt/elastic_data/`. If that path is missing or belongs to an unmounted volume, ES will fail. Run `df -h | grep elastic` to check. |
| Status is red after a hard shutdown | Run `curl -s -u elastic:elastic123 "http://localhost:9200/_cat/shards?v&h=index,shard,state,node"` to find unassigned shards. Allow ES to self-heal for 60 s before intervening. |

### OpenWebUI won't start — ImportError on elasticsearch

The `elasticsearch` Python package is not bundled with OpenWebUI. If the `openwebui` conda env was ever rebuilt, it must be reinstalled manually.

```bash
# Activate the openwebui conda environment
pmabry@sheepsoc:~$ source ~/infrastructure/miniconda3/etc/profile.d/conda.sh
pmabry@sheepsoc:~$ conda activate openwebui

# Reinstall the missing package
(openwebui) pmabry@sheepsoc:~$ pip install elasticsearch==8.19.3

# Restart the service
pmabry@sheepsoc:~$ sudo systemctl restart open-webui.service
```

### Any service is in a failed state

```bash
# See what failed
pmabry@sheepsoc:~$ systemctl --failed

# Read the recent log for the failed service
pmabry@sheepsoc:~$ journalctl -u <service>.service -n 50 --no-pager

# Reset the failed flag and try again once the underlying issue is resolved
pmabry@sheepsoc:~$ sudo systemctl reset-failed <service>.service
pmabry@sheepsoc:~$ sudo systemctl start <service>.service
```

### PCIe correctable errors in dmesg

Lines like `AER: Correctable error received from PCI Express port` or `PCIe Bus Error: severity=Correctable, type=Physical Layer` typically mean an NVMe drive is unseated or has a loose connector. Correctable errors are handled by hardware, so the system will keep running — but they will worsen over time and can eventually become uncorrectable (data loss).

**Recommended action:** power down fully, reseat all NVMe drives, then reboot and recheck dmesg.

### /mnt/nvme_working not mounted

```bash
# Try remounting everything in fstab
pmabry@sheepsoc:~$ sudo mount -a

# If that fails, mount the specific volume by UUID
pmabry@sheepsoc:~$ sudo mount UUID=bd0597fe-792f-49d0-ac98-cb22dc0467ec /mnt/nvme_working

# If mount -a still fails, check the fstab entry
pmabry@sheepsoc:~$ grep nvme_working /etc/fstab
```

### Elasticsearch cluster health stays yellow after boot

On a single-node cluster, any index with a replica count greater than zero will report `yellow` indefinitely because there is nowhere to place the replica. This is normal and benign — all data is present on the primary shards and all reads and writes work correctly. Kibana, Filebeat, Metricbeat, and the OpenWebUI RAG pipeline all function normally with a yellow cluster.

Wait 60 seconds after Elasticsearch reports `started` before calling it a problem. The translog replay after an unclean shutdown can push the status to yellow briefly, and it will recover to green (for indices that have zero replicas) on its own.

If the status is `red`, one or more primary shards are unassigned. That is an emergency. Check `journalctl -u elasticsearch -n 200 --no-pager` immediately and verify the data volume is mounted and not full.

### GPU not visible after boot

```bash
# Basic check
pmabry@sheepsoc:~$ nvidia-smi

# If nvidia-smi fails, check if the kernel module loaded
pmabry@sheepsoc:~$ lsmod | grep nvidia

# Check dmesg for DKMS or PCIe errors related to the GPU
pmabry@sheepsoc:~$ sudo dmesg -T | grep -i nvidia | head -30
```

!!! danger "Driver protection"
    If the GPU is not visible and you suspect a driver issue, do **not** attempt to reinstall the driver without a thorough risk review. The RTX 5060 Ti requires driver 570.169+ (Blackwell architecture support). Downgrading or reinstalling a different driver branch can leave the system without GPU acceleration. See `~/infrastructure/nvidia-drivers/` for the install artifacts used on this machine.
