# Tailscale

**Purpose:** WireGuard-based mesh VPN that provides encrypted remote access to sheepsoc from any Tailscale-enrolled device, without requiring inbound port-forwarding through the Starlink router.

| Key | Value |
|---|---|
| Added | 2026-05-09 |
| Updated | 2026-05-18 |
| Status | Running — `tailscaled.service` |
| Tailscale IPv4 | `100.117.117.43` |
| Tailscale IPv6 | `fd7a:115c:a1e0::1834:752b` |
| MagicDNS hostname | `sheepsoc-1.tail0f68e4.ts.net` |
| Tailnet | `tail0f68e4` (Google SSO) |
| Firewall rule | `ufw allow in on tailscale0` |

!!! note "Why the `-1` suffix?"
    Tailscale auto-suffixed `-1` to this node's hostname because a stale prior "sheepsoc" entry already holds the unsuffixed name in the tailnet. The `-1` suffix is cosmetic and has no functional impact — all IPs and Serve URLs resolve correctly. To reclaim the unsuffixed `sheepsoc.tail0f68e4.ts.net` hostname in the future, delete the stale entry at [https://login.tailscale.com/admin/machines](https://login.tailscale.com/admin/machines), then run `sudo tailscale up --force-reauth` to re-register under the clean name. This is low priority — address it if the extra suffix becomes confusing.

## Overview

Tailscale is a mesh VPN built on top of WireGuard. It assigns each enrolled device a stable private IP address in the `100.x.x.x` range (the "tailnet") and handles NAT traversal, key exchange, and device authentication automatically. No manual WireGuard key management is required.

On sheepsoc, Tailscale provides a single encrypted tunnel for remote access to all services on the box. When Phillip is off the LAN, he connects to `100.117.117.43` (or `sheepsoc-1.tail0f68e4.ts.net` via MagicDNS) and reaches OpenWebUI, Kibana, Jupyter, the docs site, and every other service on the box directly — no port-forwarding and no VPN server to maintain.

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
| MagicDNS | Enabled (tailnet-level) | Resolves `sheepsoc-1.tail0f68e4.ts.net` to `100.117.117.43` |
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
# Healthy output shows: sheepsoc-1   100.117.117.43 linux   -
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

## Tailscale Serve

`tailscale serve` exposes local HTTP services to tailnet peers over HTTPS on the MagicDNS hostname, with Let's Encrypt certificates provisioned and renewed automatically by `tailscaled`. This is **tailnet-only** access — not Tailscale Funnel — so these URLs are only reachable from devices enrolled in Phillip's `tail0f68e4` tailnet.

### Active Serve Configuration (updated 2026-05-18)

| HTTPS Port | URL | Backend | Auth |
|---|---|---|---|
| 443 | `https://sheepsoc-1.tail0f68e4.ts.net/` | `http://localhost:80` (docs landing page) | None — intentional |
| 8443 | `https://sheepsoc-1.tail0f68e4.ts.net:8443/` | `http://localhost:8888` (Jupyter) | Jupyter argon2 password |
| 8444 | `https://sheepsoc-1.tail0f68e4.ts.net:8444/` | `http://localhost:8222` (Vaultwarden) | Bitwarden/Vaultwarden login |
| 10000 | `https://sheepsoc-1.tail0f68e4.ts.net:10000/` | `http://localhost:80` (docs — alias) | None — backwards-compat alias; keep for anyone who has port 10000 bookmarked |
| 10001 | `https://sheepsoc-1.tail0f68e4.ts.net:10001/` | `http://localhost:8080` (Open WebUI) | OpenWebUI's own login |
| 10002 | `https://sheepsoc-1.tail0f68e4.ts.net:10002/` | `http://localhost:5601` (Kibana) | Kibana login |
| 10003 | `https://sheepsoc-1.tail0f68e4.ts.net:10003/` | `http://localhost:3001` (Uptime Kuma) | Uptime Kuma login |

!!! note "Default entry point changed 2026-05-18"
    The bare `https://sheepsoc-1.tail0f68e4.ts.net/` (port 443) previously opened Open WebUI. It now opens the docs landing page. Open WebUI moved to `:10001`. The old `:10000` alias is retained for backwards compatibility.

### Port-Numbering Convention

Two patterns are used when choosing a `tailscale serve` HTTPS port. The pattern is determined by how the local service is bound:

| Pattern | When to use | Example |
|---|---|---|
| **Matching port** (e.g., `:8443` → `localhost:8888`) | The local service binds **only to loopback** (`127.0.0.1`) — no collision possible with tailscaled's tailnet-IP bind | Jupyter: `127.0.0.1:8888` → Tailscale `:8443` |
| **`10000+` offset port** (e.g., `:10001` → `localhost:8080`) | The local service binds **to `0.0.0.0`** (all interfaces, including the tailnet IP) — using the same port would cause a bind collision | Open WebUI: `0.0.0.0:8080` → Tailscale `:10001` |

**Why the collision happens:** `tailscale serve --https=<port>` causes `tailscaled` to bind a listener on the tailnet IP (`100.117.117.43:<port>`). If the local service also binds `0.0.0.0:<port>`, that service already holds `:port` on every interface — including the tailnet IP — so tailscaled loses the race and the serve rule fails. The fix is to pick an unused port for tailscale serve and let it proxy to the local app's port.

The `10000+` range was chosen because it is well above common application port numbers, clearly signals "tailscale serve proxy port" at a glance, and avoids conflicts with anything else on sheepsoc.

### Why Port-Based Rather Than Path-Based

Serve rules are configured on separate HTTPS ports rather than under different paths on the same port (e.g., `/jupyter/`, `/docs/`). The reason is OpenWebUI: its frontend assets and WebSocket connections assume they are mounted at the root path. Path-based proxying breaks its UI and real-time features. Using separate ports avoids this entirely and keeps each service cleanly isolated.

### How It Was Configured

```bash
# Initial rules (2026-05-09)
pmabry@sheepsoc:~$ sudo tailscale serve --bg --https=443   http://localhost:8080  # was OpenWebUI
pmabry@sheepsoc:~$ sudo tailscale serve --bg --https=8443  http://localhost:8888  # Jupyter
pmabry@sheepsoc:~$ sudo tailscale serve --bg --https=10000 http://localhost:80    # Docs

# Updated 2026-05-18: docs become the default; OpenWebUI, Kibana, Uptime Kuma added
pmabry@sheepsoc:~$ sudo tailscale serve --bg --https=443   http://localhost:80    # Docs (new default)
pmabry@sheepsoc:~$ sudo tailscale serve --bg --https=8444  http://localhost:8222  # Vaultwarden
pmabry@sheepsoc:~$ sudo tailscale serve --bg --https=10001 http://localhost:8080  # Open WebUI
pmabry@sheepsoc:~$ sudo tailscale serve --bg --https=10002 http://localhost:5601  # Kibana
pmabry@sheepsoc:~$ sudo tailscale serve --bg --https=10003 http://localhost:3001  # Uptime Kuma
# Port 10000 (docs alias) was already present; retained as-is
```

The `--bg` flag writes serve rules to Tailscale's persistent state at `/var/lib/tailscale/`. Rules survive `tailscaled` restarts and reboots — no separate systemd unit is needed.

### Auto TLS

Let's Encrypt certificates for `sheepsoc-1.tail0f68e4.ts.net` are provisioned and renewed automatically by `tailscaled`. No certbot, no manual DNS records, no separate cert management required.

### Firewall and UFW

No UFW changes were needed. `tailscale serve` proxies traffic via loopback inside the host — the existing `ALLOW IN on tailscale0` rule already covers all tailnet-side access, and loopback traffic is not subject to UFW.

### Per-Node Serve Enablement

Serve was enabled for this node via the per-node grant URL (`https://login.tailscale.com/f/serve?node=...`). The global "Serve / Funnel" toggle in the Tailscale admin console was not visible at the time of configuration, but per-node enablement is sufficient. If sheepsoc is ever recreated or re-enrolled, this per-node grant will need to be re-applied — keep the grant URL or repeat the enablement step from the admin console.

### Removing Serve Rules

Individual rules can be removed without affecting the underlying services — only the Tailscale proxy layer is removed:

```bash
# Remove a specific service
pmabry@sheepsoc:~$ sudo tailscale serve --https=443   off   # remove docs (default)
pmabry@sheepsoc:~$ sudo tailscale serve --https=8443  off   # remove Jupyter
pmabry@sheepsoc:~$ sudo tailscale serve --https=8444  off   # remove Vaultwarden
pmabry@sheepsoc:~$ sudo tailscale serve --https=10000 off   # remove docs alias
pmabry@sheepsoc:~$ sudo tailscale serve --https=10001 off   # remove Open WebUI
pmabry@sheepsoc:~$ sudo tailscale serve --https=10002 off   # remove Kibana
pmabry@sheepsoc:~$ sudo tailscale serve --https=10003 off   # remove Uptime Kuma

# Remove all serve config at once
pmabry@sheepsoc:~$ sudo tailscale serve reset
```

### Security Notes

`tailscale serve` adds no authentication of its own. Each service retains its own auth layer:

- **Docs landing page** (ports 443 and 10000) — intentionally unauthenticated; approved by Phillip. Exposure is equivalent to existing LAN access (any device on `192.168.50.0/24` can already reach it). If a future tailnet device is added that should not have docs access, remove or restrict the rule at that time.
- **Open WebUI** — requires OpenWebUI account login.
- **Jupyter** — requires the argon2 password configured at setup.
- **Kibana** — requires Kibana login.
- **Uptime Kuma** — requires Uptime Kuma login.
- **Vaultwarden** — requires Bitwarden/Vaultwarden account login; admin panel at `/admin` requires the admin token.

### View Current Serve Config

```bash
pmabry@sheepsoc:~$ tailscale serve status
```

## Runbooks

- [Tailscale Operations](../runbooks/tailscale-ops.md) — adding a node to the tailnet, removing a node, rotating auth keys, managing serve rules, and full uninstall procedure

## Known Issues / Gotchas

### `tailscale serve` Bind Collision — `0.0.0.0` Services

When a local service binds to `0.0.0.0:<port>` (all interfaces), `tailscaled` cannot also bind to the tailnet IP on that same port when a `--https=<port>` serve rule is configured. The result is a silent failure — the rule appears in `tailscale serve status` but HTTPS connections fail because tailscaled lost the bind race.

**Workaround:** choose a different port for the tailscale serve rule and let it proxy to the app's local port. The `10000+` range is the convention on sheepsoc for this pattern (see the [Port-Numbering Convention](#port-numbering-convention) section above). Services that bind loopback-only (`127.0.0.1`) are not affected.

See also: [Known Issues — tailscale serve bind collision](../known-issues.md#tailscale-serve-bind-collision).

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
