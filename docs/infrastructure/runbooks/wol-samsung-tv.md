# Samsung TV Network Control (expands prior WoL runbook)

**Purpose:** Full network control for Samsung TV using `tv_control.py` (volume prioritized, WoL power-on, keys, **--youtube-search "QUERY"** with pre-clear of search field via 15x KEY_BACKSPACE loop + sleeps to prevent stale text). Builds on successful DHCP discovery, WoL test, and prior volume/key/pairing work from 2026-05-30 (MAC from ASUS DHCP, TV at 192.168.50.175 responded to `conda run -n sheepsoc --youtube-search 'Try not to laugh'`). Script at `infrastructure/scripts/tv_control.py` (executable, tested with no errors). Uses `samsungtvws` (run_app for YouTube ID 111299001912 + navigation/send_text/send_key + clear) + wakeonlan. One-time pairing (PIN on TV) saves token to `~/.config/samsung-tv-token.json`. Comprehensive error handling. All prior WoL/volume content preserved below for reversibility.

| Key | Value |
|---|---|
| Hostname | `Samsung` |
| IP | `192.168.50.175` (DHCP lease from ASUS RT-AX5400) |
| MAC | `54:3A:D6:5D:B0:EC` |
| Script Path | `~/infrastructure/scripts/tv_control.py` (or relative from root) |
| Conda Env | `sheepsoc` (Python 3.12; `conda activate sheepsoc`) |
| Install | `conda activate sheepsoc && pip install samsungtvws wakeonlan` (confirmed safe, no other env changes) |
| Primary Use | Volume control first (`--volume 40`, `--up`, `--down`, `--mute`), power (`--power on` uses WoL), raw keys (`--key KEY_VOLUP`), **YouTube search** (`--youtube-search "QUERY"`) |
| Pairing | One-time on first run (enter PIN from TV); token saved to `~/.config/samsung-tv-token.json` |
| Prerequisites | Wired Ethernet, WoL enabled, **Network Remote / Device Connect Manager > Access Notification enabled** on TV (Settings > General > Network > Expert Settings) |
| Verified | 2026-05-30 (DHCP/WoL, volume, pairing, and `--youtube-search "Try not to laugh"` tested via `conda run -n sheepsoc`; TV responded with no errors); updated per new service capability trigger |

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

## Usage Examples (Volume First, as Prioritized in Script; YouTube Search Added)

Activate conda env then run (CLI help via no args or `--help`; `conda run -n sheepsoc` also works for one-offs):

```bash
pmabry@sheepsoc:~$ conda activate sheepsoc
pmabry@sheepsoc:~$ python infrastructure/scripts/tv_control.py --volume 40
# Sets exact volume level (0-100; loops KEY_VOLUP from low/mute state)

pmabry@sheepsoc:~$ python infrastructure/scripts/tv_control.py --up
# Volume up (or --down, --mute)

pmabry@sheepsoc:~$ python infrastructure/scripts/tv_control.py --power on
# Uses integrated WoL + 8s wait, then connects

pmabry@sheepsoc:~$ python infrastructure/scripts/tv_control.py --key KEY_VOLUP
# Or KEY_MUTE, KEY_POWEROFF, KEY_HOME, KEY_CHUP, etc. (full Samsung key list supported by library)

pmabry@sheepsoc:~$ python infrastructure/scripts/tv_control.py --youtube-search "Try not to laugh"
# Launches YouTube app (ID 111299001912 via run_app), navigates to search bar (KEY_UP/RIGHT/ENTER), sends text via send_text(), submits. Sleeps tuned for load time; verify on TV (may need minor manual adjustment per firmware). Tested successfully.
```

**Pairing (first run only):** Script auto-detects no token, opens connection, prompts for PIN shown on TV. Saves token automatically to `~/.config/samsung-tv-token.json`. Re-run with `--power on` if TV off. New YouTube feature requires TV to be on or use `--power on` first.

## How It Works

- **Power-on:** Uses `wakeonlan` with known MAC (integrates prior successful WoL test exactly; sends to broadcast; 8s sleep).
- **Volume/Keys:** `SamsungTVWS` over websocket (port 8002). Loads/saves token for authenticated session. `send_key()` for volume (looped for absolute levels), mute, power off, raw keys.
- **YouTube Search (`--youtube-search`):** `tv.run_app("111299001912")` to launch YouTube app, followed by navigation keys (`KEY_UP`, `KEY_RIGHT`, `KEY_ENTER` to reach search field), **loop of 15x `KEY_BACKSPACE` (with 0.1s sleeps) to clear any stale text from prior searches**, then `send_text(query)`, final `KEY_ENTER`. Sleeps (6s for app load, 1-2s between actions, 0.5s after clear) tuned for reliability. Prevents stale query text. Note: full typing/navigation is simulated and firmware-dependent; manual adjustment may be needed.
- **Error Handling:** Catches exceptions, prints troubleshooting (TV on/wired, re-pair by deleting token, port 8002, firewall). Uses try/finally for `tv.close()`.
- **Token File:** JSON at `~/.config/samsung-tv-token.json` (created on successful pair; contains token + IP). One-time only.
- Builds on existing `sheepsoc` conda env (samsungtvws + wakeonlan) and 2026-05-30 DHCP/WoL/volume success (no new hardware changes; tested with `conda run -n sheepsoc`).

(See script docstring and code for full implementation; reversible by commenting lines or restoring .bak from prior version.)

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
- `ERROR: Missing dependencies...` → Run the install command above (`conda activate sheepsoc && pip install samsungtvws wakeonlan`); confirm with Phillip first per CLAUDE.md.
- Pairing fails / "PIN" not shown → Ensure **Access Notification** (Network Remote) enabled in TV Expert Settings (see Prerequisites). TV must be powered on for pairing.
- Connection error / timeout → Verify TV is on (use `--power on` first), port 8002 reachable (`nc -z 192.168.50.175 8002`), firewall allows from sheepsoc. Re-pair by deleting token file and re-running.
- Volume/key commands fail after power-on → Increase sleep in script or wait 10-15s after WoL; TV boot takes time for websocket.
- **YouTube search fails or lands wrong place / stale text remains** → The script now clears the field with 15x `KEY_BACKSPACE` (0.1s sleeps) immediately after focusing the input box and before `send_text()`. TV firmware variations in app UI may still require increasing sleeps or minor manual navigation. Test with short query like "test". App ID `111299001912` is standard for YouTube on many Samsung models (confirm with `tv.app_list()` if needed).
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

- [Services](../services.md#tv-control) — **see also** TV Control section documenting the `--youtube-search` + clear-search behavior (reciprocal link updated).
- [Topology](../topology.md#network-topology) — **see also** full network diagram and Samsung TV control notes (reciprocal link).
- [Infrastructure Overview](../index.md) — links to all operational procedures including this runbook (reciprocal).
- [Known Issues](../known-issues.md#history-log) — **see also** latest history entry on clear-search update to tv_control.py (reciprocal; new entry added).
- [Conda](../platforms/conda.md) — **see also** for sheepsoc env details where script runs (uses `conda run -n sheepsoc`).

This runbook revised per schema.md section 6 update triggers for code change to TV control procedure, runbook, and services (also "service reconfigured" patterns). Added clear step to How It Works, usage summary, troubleshooting; updated services.md TV section and schema.md registry. All prior WoL/volume content preserved (reversible via .bak or git). Links follow section 4 rules exactly: relative paths from current location in `docs/infrastructure/runbooks/`, first-mention only per section, explicit relationship labels (**see also**, **runbook**), reciprocal links to affected pages (services.md, topology.md, index.md, known-issues.md, schema.md) where meaningful (no noise). Backlinks and interlinks verified — no broken links. No mkdocs build performed. Wiki remains single source of truth per CLAUDE.md. Brief to Documentation agent covered the clear-search loop (15x KEY_BACKSPACE post-focus, pre-send_text) + successful test.
