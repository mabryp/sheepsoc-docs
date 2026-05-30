# Samsung TV Network Control (expands prior WoL runbook)

**Purpose:** Full network control for Samsung TV using `tv_control.py` (volume prioritized, WoL power-on via wakeonlan, raw keys, **--youtube-search "QUERY"**). **Major refinement** (after repeated keyboard/cursor/send_text failures and user note on onscreen keyboard): now uses `tv.open_browser()` with `https://www.youtube.com/results?search_query={quote(query)}` (`from urllib.parse import quote`). Completely bypasses YouTube app, on-screen keyboard, cursor navigation, clear loops, and per-char KEY_* simulation. Much simpler/reliable. Updated imports, argparse help, docstring, and logic in current script. Re-tested successfully (opens search results directly in browser). Token/WoL/volume unchanged. Script at `infrastructure/scripts/tv_control.py` (executable; uses samsungtvws + wakeonlan; token in `~/.config/samsung-tv-token.json`). Builds on prior DHCP/WoL/volume/pairing success from 2026-05-30 (MAC from ASUS, TV 192.168.50.175). All prior WoL/volume content preserved below for reversibility (git revert possible). Wiki is living record of current open_browser browser-bypass method per CLAUDE.md.

| Key | Value |
|---|---|
| Hostname | `Samsung` |
| IP | `192.168.50.175` (DHCP lease from ASUS RT-AX5400) |
| MAC | `54:3A:D6:5D:B0:EC` |
| Script Path | `~/infrastructure/scripts/tv_control.py` (or relative from root) |
| Conda Env | `sheepsoc` (Python 3.12; `conda activate sheepsoc`) |
| Install | `conda activate sheepsoc && pip install samsungtvws wakeonlan` (confirmed safe, no other env changes) |
| Primary Use | Volume control first (`--volume 40`, `--up`, `--down`, `--mute`), power (`--power on` uses WoL), raw keys (`--key KEY_VOLUP`), **YouTube search** (`--youtube-search "QUERY"`) — opens browser directly to search results |
| Pairing | One-time on first run (enter PIN from TV); token saved to `~/.config/samsung-tv-token.json` |
| Prerequisites | Wired Ethernet, WoL enabled, **Network Remote / Device Connect Manager > Access Notification enabled** on TV (Settings > General > Network > Expert Settings) |
| Verified | 2026-05-30 (DHCP/WoL, volume, pairing, and new `--youtube-search "Try not to laugh"` via browser method tested via `conda run -n sheepsoc`; opens search results directly with no errors); updated per major procedure change trigger |

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

## Usage Examples (Volume First, as Prioritized in Script; YouTube Search Now Uses Browser Bypass)

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
# Uses `tv.open_browser()` with properly quoted YouTube search URL (`https://www.youtube.com/results?search_query=...`). Completely bypasses YouTube app, on-screen keyboard, cursor, clear, and char-by-char keys (after repeated failures with previous methods). Much simpler and reliable. Opens search results page directly on TV. Token present, no re-pairing needed; TV should be on or use `--power on` first.
```

**Pairing (first run only):** Script auto-detects no token, opens connection, prompts for PIN shown on TV. Saves token automatically to `~/.config/samsung-tv-token.json`. Re-run with `--power on` if TV off. YouTube search now works via browser even if TV was off (after WoL).

## How It Works

- **Power-on:** Uses `wakeonlan` with known MAC (integrates prior successful WoL test exactly; sends to broadcast; 8s sleep).
- **Volume/Keys:** `SamsungTVWS` over websocket (port 8002). Loads/saves token for authenticated session. `send_key()` for volume (looped for absolute levels), mute, power off, raw keys.
- **YouTube Search (`--youtube-search`):** Uses `tv.open_browser(url)` where `url = f"https://www.youtube.com/results?search_query={quote(query)}"` (with `urllib.parse.quote` for safe encoding). **Completely bypasses** the YouTube app, on-screen keyboard, cursor navigation, field-clearing, and per-character KEY_* loops. This is the major refinement after repeated failures with `send_text()`, cursor movement without text, and on-screen keyboard issues noted by user. Much simpler, faster, and more reliable. No app launch, no navigation keys, no sleeps for typing. Opens the search results page directly in the TV's browser. Token/WoL/volume logic unchanged. Current implementation (browser bypass) is the living wiki record per CLAUDE.md and schema.md.
- **Error Handling:** Catches exceptions, prints troubleshooting (TV on/wired, re-pair by deleting token, port 8002, firewall). Uses try/finally for `tv.close()`.
- **Token File:** JSON at `~/.config/samsung-tv-token.json` (created on successful pair; contains token + IP). One-time only.
- Builds on existing `sheepsoc` conda env (samsungtvws + wakeonlan + urllib stdlib) and 2026-05-30 DHCP/WoL/volume success (no new hardware changes; tested with `conda run -n sheepsoc`).

(See script docstring and code for full implementation — argparse, get_tv(), open_browser call; reversible via git to prior char-key commits.)

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
- **YouTube search fails or does not open results** → New browser method is far more reliable (no app/keyboard/cursor issues). Verify TV is powered on (or use `--power on` first), network reachable, and query has no problematic chars (quote() handles encoding). Test with simple query like "test". Check TV browser opens YouTube results page. Re-test with current script (uses `open_browser()` + `quote()`). If websocket fails, same as above. Current browser-bypass behavior is the wiki living record.
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

- [Services](../services.md#tv-control) — **see also** TV Control section documenting current browser-bypass YouTube search via `open_browser()` (reciprocal).
- [Topology](../topology.md#network-topology) — **see also** full network diagram and Samsung TV control notes (reciprocal link).
- [Infrastructure Overview](../index.md) — links to all operational procedures including this runbook (reciprocal).
- [Known Issues](../known-issues.md#history-log) — **see also** latest history entry on tv_control browser bypass refinement (reciprocal; new entry added).
- [Conda](../platforms/conda.md) — **see also** for sheepsoc env details where script runs (uses `conda run -n sheepsoc`).

This runbook revised per schema.md section 6 update triggers for major tv_control.py procedure change (YouTube search to browser bypass). Updated **only** YouTube sections (purpose, usage, How It Works, troubleshooting) + services.md TV section, schema.md registry, known-issues.md history per instructions. All prior WoL/volume content preserved (reversible via git). Links follow §4 exactly (relative paths, first-mention per section, **see also**/**runbook** labels, reciprocals where meaningful; no topology/index updates; verified). No mkdocs build. Wiki reflects *current* code (open_browser + quote) as living record per CLAUDE.md. Documentation agent task complete. Commit: "refine tv_control --youtube-search to browser bypass (open_browser + quote)".
