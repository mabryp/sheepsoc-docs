# Tailscale

**Purpose:** WireGuard-based mesh VPN that provides encrypted remote access to sheepsoc from any Tailscale-enrolled device, without requiring inbound port-forwarding through the Starlink router.

| Key | Value |
|---|---|
| Added | 2026-05-09 |
| Status | Running — `tailscaled.service` |
| Tailscale IPv4 | `100.117.117.43` |
| Tailscale IPv6 | `fd7a:115c:a1e0::1834:752b` |
| MagicDNS hostname | `sheepsoc.tail0f68e4.ts.net` |
| Tailnet | `tail0f68e4` (Google SSO) |
| Firewall rule | `ufw allow in on tailscale0` |

## Overview

Tailscale is a mesh VPN built on top of WireGuard. It assigns each enrolled device a stable private IP address in the `100.x.x.x` range (the "tailnet") and handles NAT traversal, key exchange, and device authentication automatically. No manual WireGuard key management is required.

On sheepsoc, Tailscale provides a single encrypted tunnel for remote access to all services on the box. When Phillip is off the LAN, he connects to `100.117.117.43` (or `sheepsoc.tail0f68e4.ts.net` via MagicDNS) and reaches OpenWebUI, Kibana, Jupyter, the docs site, and every other service on the box directly — no port-forwarding and no VPN server to maintain.

The installation was intentionally minimal: no subnet routing, no exit-node advertising, and no Tailscale SSH. The rationale for each decision is documented in the [Configuration](#configuration) section.

## Network Context — Why Tailscale and Not Port Forwarding

Sheepsoc's internet uplink is Starlink. Starlink uses CGNAT (Carrier-Grade NAT), which means the ASUS RT-AX5400 does not hold a routable public IP address. Even if a port-forwarding rule were added on the ASUS, the Starlink router upstream would block inbound connections from the public internet. Direct inbound access is structurally impossible at the ASUS layer.

Tailscale solves this without requiring any changes to the upstream Starlink configuration. Enrolled devices establish outbound connections to Tailscale's coordination server and attempt direct peer-to-peer WireGuard connections via STUN/DERP. When a direct path cannot be established (which is expected here due to CGNAT), Tailscale falls back to its DERP relay network automatically and transparently. There is no manual configuration required and no functional difference to the user — latency may be slightly higher via relay than via a direct peer-to-peer path, but this is acceptable for the use case.

## Registered Peers

| Device | Tailscale IP | Notes |
|---|---|---|
| sheepsoc | `100.117.117.43` | This host |
| phillips-macbook-pro-3 | `100.88.90.20` | Phillip's MacBook — primary remote-access client |

## Configuration

### What Is Enabled

| Feature | State | Rationale |
|---|---|---|
| `tailscaled.service` | Enabled, active | Runs at boot; manages the WireGuard tunnel and coordination |
| MagicDNS | Enabled (tailnet-level) | Resolves `sheepsoc.tail0f68e4.ts.net` to `100.117.117.43` |
| UFW rule on `tailscale0` | Present | Permits all inbound traffic from tailnet peers on the `tailscale0` interface |

### What Is Deliberately Disabled

| Feature | State | Rationale |
|---|---|---|
| Subnet routing (`--advertise-routes`) | Off | Phillip only needs to reach sheepsoc itself; no other LAN hosts need to be reachable from the tailnet. Subnet routing requires admin-console route approval and increases attack surface. |
| Exit node | Off | Advertising sheepsoc as an exit node would route Phillip's MacBook's internet traffic through the Starlink WAN. This is undesirable — Starlink is the uplink for sheepsoc, not a general-purpose exit path. |
| Tailscale SSH | Off | The existing OpenSSH configuration is already hardened (key-only auth, `PasswordAuthentication no`, `PermitRootLogin no`, `MaxAuthTries 3`). Adding Tailscale SSH would duplicate an access path that is already secure and well-understood. SSH over the Tailscale tunnel works using the existing key infrastructure — no changes to `~/.ssh` are needed. |

### Systemd Unit

Tailscale is managed by `tailscaled.service`, which is enabled and active. The CLI tool (`tailscale`) communicates with the daemon over a Unix socket.

```bash
# Check status
pmabry@sheepsoc:~$ systemctl status tailscaled.service

# Check tailnet connection status
pmabry@sheepsoc:~$ tailscale status

# Show assigned IPs and hostname
pmabry@sheepsoc:~$ tailscale ip
```

### Firewall (UFW)

A single UFW rule permits all inbound traffic on the `tailscale0` virtual interface:

```bash
# Rule added at install time (2026-05-09)
pmabry@sheepsoc:~$ sudo ufw allow in on tailscale0
```

This rule covers all services on sheepsoc in a single line. Because `tailscale0` only carries traffic that has already been authenticated and encrypted by Tailscale, no further per-port rules are needed for tailnet peers. No FORWARD rules were added (subnet routing is off). No UDP 41641 rule was added (no benefit under Starlink CGNAT — Tailscale uses DERP relay instead).

!!! note "Rule Scope"
    The `ufw allow in on tailscale0` rule is interface-scoped — it applies only to traffic arriving on `tailscale0`, not to the LAN interface (`eno1`) or any other interface. Existing LAN-only rules on `eno1` are unchanged.

### Sysctl

`/etc/sysctl.d/99-tailscale.conf` was created at install time to make IP forwarding explicit and persistent:

```ini
# /etc/sysctl.d/99-tailscale.conf
net.ipv4.ip_forward = 1
```

IPv6 forwarding was intentionally **not** enabled — it is only required for subnet routing, which is off.

### Install Method

Tailscale was installed from the official Tailscale apt repository:

```bash
# Package source
deb https://pkgs.tailscale.com/stable/ubuntu noble main

# Keyring
/usr/share/keyrings/tailscale-archive-keyring.gpg

# Source list
/etc/apt/sources.list.d/tailscale.list
```

## Health Checks

```bash
# Is the daemon running?
pmabry@sheepsoc:~$ systemctl is-active tailscaled.service

# Is the tailnet connection up?
pmabry@sheepsoc:~$ tailscale status
# Healthy output shows: sheepsoc   100.117.117.43 linux   -
# and any connected peers

# What relay is being used (if no direct path)?
pmabry@sheepsoc:~$ tailscale netcheck
# Shows DERP relay, latency, and UDP port reachability

# Verify MagicDNS is resolving
pmabry@sheepsoc:~$ tailscale ip -4
# Should return: 100.117.117.43
```

!!! note "PollNetMap: unexpected EOF"
    Log entries containing `PollNetMap: unexpected EOF` are normal. This is the Tailscale daemon recycling its long-poll connection to the coordination server. It is not an error.

## Runbooks

- [Tailscale Operations](../runbooks/tailscale-ops.md) — adding a node to the tailnet, removing a node, rotating auth keys, and full uninstall procedure

## Known Issues / Gotchas

### DERP Relay Due to Starlink CGNAT

Tailscale will attempt to establish a direct WireGuard peer-to-peer connection between sheepsoc and any remote peer. Due to Starlink CGNAT, direct inbound connections to sheepsoc from the public internet are not possible. Tailscale will fall back to the `derp-9` relay (auto-selected at install time). This is transparent and expected — the tunnel functions identically via relay, with slightly higher latency than a direct path. No action is required.

To verify which path is in use:

```bash
pmabry@sheepsoc:~$ tailscale netcheck
```

If a direct path ever becomes available (e.g., if the uplink changes), Tailscale will prefer it automatically.

### Tailscale Purged in April 2026

A previous Tailscale installation was on sheepsoc but had been failing for approximately two months prior to 2026-04-18. That installation was fully purged on 2026-04-19 as part of the security hardening session. The current installation (2026-05-09) is fresh from the official Tailscale apt repository and enrolled to the same tailnet. See [Known Issues](../known-issues.md) for the historical context.

## Dependencies

- [Services](../services.md) — service catalog entry for `tailscaled.service`

## See Also

- [Topology](../topology.md) — network map showing Tailscale as the remote-access overlay alongside the LAN topology
- [Known Issues](../known-issues.md) — history of the prior failed Tailscale installation and the CGNAT context
