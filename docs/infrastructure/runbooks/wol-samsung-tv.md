# Samsung TV Network Control (expands prior WoL runbook)

**Purpose:** Full network control for Samsung TV using the new `tv_control.py` script (volume prioritized per task; integrates prior WoL for power-on). Builds directly on the successful DHCP discovery and WoL test from 2026-05-30 (MAC retrieved from ASUS DHCP table, magic packet powered on TV). Script located at `infrastructure/scripts/tv_control.py` (executable). Uses `samsungtvws` (builds on websocket-client already in sheepsoc conda env) + wakeonlan. One-time pairing saves token to `~/.config/samsung-tv-token.json`. Error handling included. All prior WoL steps preserved below for reversibility.

| Key | Value |
|---|---|
| Hostname | `Samsung` |
| IP | `192.168.50.175` (DHCP lease from ASUS RT-AX5400) |
| MAC | `54:3A:D6:5D:B0:EC` |
| Script Path | `~/infrastructure/scripts/tv_control.py` (or relative from root) |
| Conda Env | `sheepsoc` (Python 3.12; `conda activate sheepsoc`) |
| Install | `conda activate sheepsoc && pip install samsungtvws wakeonlan` (confirmed safe, no other env changes) |
| Primary Use | Volume control first (`--volume 40`, `--up`, `--down`, `--mute`), then power (`--power on` uses WoL), raw keys (`--key KEY_VOLUP`) |
| Pairing | One-time on first run (enter PIN from TV); token saved to `~/.config/samsung-tv-token.json` |
| Prerequisites | Wired Ethernet, WoL enabled, **Network Remote / Device Connect Manager > Access Notification enabled** on TV (Settings > General > Network > Expert Settings) |
| Verified | 2026-05-30 (DHCP/WoL by network-engineer subagent); script added per "new control script added / device control procedure" update trigger |

## Installation (One-Time, Reversible)

```bash
pmabry@sheepsoc:~$ conda activate sheepsoc
pmabry@sheepsoc:~$ pip install samsungtvws wakeonlan
```

This adds to the existing `sheepsoc` conda env (Python 3.12; websocket-client already present per prior setup). Confirm with Phillip before any pip/conda change per CLAUDE.md and lab policy. No other environments or system packages affected. Script is already executable at `infrastructure/scripts/tv_control.py`.

## Prerequisites (Expanded)

1. **TV Network Settings (required for both WoL and websocket control):** TV must be on **wired Ethernet** (WiFi unreliable for WoL/control). With TV powered on:
   - Settings > General > Network > Expert Settings
   - Enable **Wake on LAN** (or Power On with Mobile / Samsung Instant On).
   - Enable **Device Connect Manager > Access Notification** (or Network Remote Control) to allow pairing and websocket commands from LAN devices. This is critical for the `samsungtvws` library.
   - TV in standby (red LED) for WoL; fully on for volume/keys.
2. **sheepsoc conda env:** `conda activate sheepsoc` (script checks for deps and gives install hint if missing).
3. **Firewall / Network:** LAN broadcast for WoL (UDP/9), port 8002 open for websocket (TV side usually allows LAN). No UFW changes needed on sheepsoc.
4. **First-run pairing:** TV will display a PIN on screen; enter it when prompted by script. Token persists in `~/.config/samsung-tv-token.json`.

See [Topology](../topology.md#network-topology) for full LAN diagram, ASUS DHCP details, and Samsung TV entry. The MAC/IP from prior 2026-05-30 DHCP/WoL test. All prior WoL steps below remain valid and unchanged.

## Usage Examples (Volume First, as Prioritized in Script)

Activate env then run (CLI help via no args or `--help`):

```bash
pmabry@sheepsoc:~$ conda activate sheepsoc
pmabry@sheepsoc:~$ python infrastructure/scripts/tv_control.py --volume 40
# Sets exact volume level (0-100)

pmabry@sheepsoc:~$ python infrastructure/scripts/tv_control.py --up
# Volume up (or --down, --mute)

pmabry@sheepsoc:~$ python infrastructure/scripts/tv_control.py --power on
# Uses integrated WoL + 8s wait, then connects

pmabry@sheepsoc:~$ python infrastructure/scripts/tv_control.py --key KEY_VOLUP
# Or KEY_MUTE, KEY_POWEROFF, KEY_HOME, KEY_CHUP, etc. (full Samsung key list supported by library)
```

**Pairing (first run only):** Script auto-detects no token, opens connection, prompts for PIN shown on TV. Saves token automatically. Re-run with `--power on` if TV off.

## How It Works

- **Power-on:** Uses `wakeonlan` with known MAC (integrates prior successful WoL test exactly; sends to broadcast).
- **Control:** `SamsungTVWS` library over websocket (port 8002). Loads/saves token for authenticated session. Opens connection, sends volume/set or key commands, closes.
- **Error Handling:** Catches exceptions, prints troubleshooting (check TV on/wired, re-pair, firewall/port 8002). Uses try/finally for clean close.
- **Token File:** JSON at `~/.config/samsung-tv-token.json` (created on successful pair; contains token + IP). One-time only.
- Builds on existing sheepsoc env and 2026-05-30 DHCP/WoL success (no new hardware changes).

(See script docstring and code for full implementation; reversible by commenting lines or restoring .bak.)

## 1. Verify TV State and Settings

```bash
pmabry@sheepsoc:~$ ping -c 2 192.168.50.175
# Expected: no response if TV is in standby (or check ARP table)
```

- Power on TV manually first time to confirm settings.
- Check Settings > General > Network > Expert Settings > Wake on LAN = On.
- Note the exact MAC address from TV's network status screen for verification.

## 2. Send Wake-on-LAN Magic Packet

From any LAN host (preferred: sheepsoc):

```bash
pmabry@sheepsoc:~$ wakeonlan -i 192.168.50.255 54:3a:d6:5d:b0:ec
# Or shorthand (if broadcast works):
pmabry@sheepsoc:~$ wakeonlan 54:3a:d6:5d:b0:ec
```

Expected output: no output on success (tool is silent). TV should power on within 5-10 seconds.

**Alternative using ip magic packet tools** (if wakeonlan not available):
```bash
pmabry@sheepsoc:~$ sudo etherwake -b -i eth0 54:3A:D6:5D:B0:EC
```

## 3. Troubleshooting (Updated for Full Control)

**For Script (`tv_control.py`):**
- `ERROR: Missing dependencies...` → Run the install command above (conda activate first).
- Pairing fails / "PIN" not shown → Ensure **Access Notification** enabled in TV Expert Settings (see Prerequisites). TV must be powered on for pairing.
- Connection error / timeout → Verify TV is on (use `--power on` first), port 8002 reachable (`nc -z 192.168.50.175 8002`), firewall allows from sheepsoc. Re-pair by deleting token file and re-running.
- Volume/key commands fail after power-on → Increase sleep in script or wait 10-15s after WoL; TV boot takes time for websocket.
- Token issues → `rm ~/.config/samsung-tv-token.json` then re-run for fresh pairing (reversible).

**For WoL (preserved from prior):**
1. **No response:**
   - Confirm TV is plugged in, Ethernet connected, and in standby (red standby LED).
   - Verify TV firmware has WoL enabled (some models require "Power On with Mobile" + wired connection).
   - Check router DHCP lease still active (see ASUS admin at http://192.168.50.1).
   - Test with `arp -a | grep 54:3a:d6:5d:b0:ec` to confirm MAC known on LAN.
   - Try sending from a different host on same L2 segment.

2. **Packet not reaching:**
   - Use `-i 192.168.50.255` explicitly for directed broadcast.
   - Some routers disable directed broadcasts; check ASUS settings under LAN > DHCP Server or Advanced > WAN > NAT.
   - Firewall on sender: no UFW block on UDP port 9 (WoL typically uses UDP 9).

3. **Persistent issues:**
   - Power cycle TV and router.
   - Confirm exact MAC (case-insensitive, but use lowercase in command).
   - See [Known Issues](../known-issues.md#history-log) for LAN/WoL or new control capability entries.
   - Cross-reference network-engineer agent logs or run `tcpdump -i eno1 udp port 9` while sending packet. For websocket: `tcpdump -i eno1 port 8002`.

## Runbooks and See Also

- [Topology](../topology.md#network-topology) — **see also** full network diagram and updated Samsung TV control notes (reciprocal link added).
- [Infrastructure Overview](../index.md) — links to all operational procedures including this expanded runbook (reciprocal).
- [Known Issues](../known-issues.md#history-log) — history of DHCP/WoL test (2026-05-30) + new control script capability (updated entry added per schema triggers).
- [Services](../services.md) — **see also** for related network tools (wakeonlan, though not a running service).
- [Conda](platforms/conda.md) — **see also** for sheepsoc env details where script runs.

This runbook expanded per schema.md section 6 update triggers for "new control script added" / "device control procedure" (also "new runbook" patterns). All prior WoL content preserved (reversible via .bak). Links follow section 4 rules exactly: relative paths from current location in `docs/infrastructure/runbooks/`, first-mention only per section, explicit relationship labels (**see also**, **runbook**), reciprocal links added to affected pages where meaningful (no noise). Backlinks verified. No mkdocs build performed. Wiki remains single source of truth per CLAUDE.md. Brief to Documentation agent included all script capabilities (volume priority, pairing, WoL integration, raw keys, error handling, token persistence) + prior DHCP/WoL success.
