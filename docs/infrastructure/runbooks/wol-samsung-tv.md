# WoL Samsung TV

**Purpose:** Step-by-step procedure to wake the Samsung TV (newly documented LAN device) using Wake-on-LAN magic packet. Resolves previously undocumented host.

| Key | Value |
|---|---|
| Hostname | `Samsung` |
| IP | `192.168.50.175` (DHCP lease from ASUS RT-AX5400) |
| MAC | `54:3A:D6:5D:B0:EC` |
| WoL Command | `wakeonlan -i 192.168.50.255 54:3a:d6:5d:b0:ec` |
| Verified | 2026-05-30 by network-engineer subagent (packet sent successfully, TV powered on) |

## Prerequisites

1. **Wired Ethernet required.** The TV must be connected via Ethernet cable to the LAN switch or ASUS router. WiFi WoL is unreliable and not supported here.
2. **TV Settings enabled.** With the TV powered on, navigate to:
   - Settings > General > Network > Expert Settings
   - Enable **Wake on LAN** (or **Power On with Mobile** / **Samsung Instant On** depending on model/firmware).
   - The TV must be in standby mode (not fully powered off or unplugged).
3. **wakeonlan tool.** Installed on sender host (sheepsoc or other LAN machine with `sudo apt install wakeonlan` if missing — confirm with Phillip first per lab policy).
4. **Broadcast address.** Use the LAN broadcast (`192.168.50.255`) or ensure router forwards magic packets.

See [Topology](../topology.md#network-topology) for full LAN diagram and ASUS DHCP details. The MAC was retrieved from the router's DHCP client table.

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

## 3. Troubleshooting

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
   - See [Known Issues](../known-issues.md) for any LAN/WoL landmines.
   - Cross-reference network-engineer agent logs or run `tcpdump -i eno1 udp port 9` while sending packet.

## Runbooks and See Also

- [Topology](../topology.md) — where this device lives in the LAN host layout and network diagram (updated with this entry).
- [Infrastructure Overview](../index.md) — links to all operational procedures.
- [Services](../services.md) — for related network tools if wakeonlan is cataloged.
- [Known Issues](../known-issues.md#history-log) — history of first documentation and successful test.

This runbook was created per schema.md update triggers for new device added to LAN and new operational procedure. All links follow section 4 rules (relative paths, first-mention, reciprocal where meaningful). Reciprocal links added to topology.md, index.md, known-issues.md, and schema.md registry.
