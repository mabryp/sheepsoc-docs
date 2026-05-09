# Tailscale Operations

**Purpose:** Step-by-step procedures for managing sheepsoc's Tailscale installation — adding and removing peers, rotating auth keys, and performing a full uninstall.

**Service page:** [Tailscale](../platforms/tailscale.md)

---

## Prerequisites

All procedures on this page require:

- SSH access to sheepsoc (LAN or via the tailnet itself if you are already enrolled)
- Access to the Tailscale admin console at [https://login.tailscale.com/admin](https://login.tailscale.com/admin) (Google SSO, Phillip's account)

---

## 1. Add a New Device to the Tailnet

Use this procedure to enroll a new computer, phone, or server into the `tail0f68e4` tailnet.

### On the new device

1. Install Tailscale following the official instructions for the device's operating system: [https://tailscale.com/download](https://tailscale.com/download)

2. Authenticate to the tailnet. On a Linux host:

    ```bash
    sudo tailscale up --authkey=<auth-key>
    ```

    On macOS or Windows, the Tailscale client will open a browser window for Google SSO authentication.

3. Verify the device appears in the tailnet:

    ```bash
    # From sheepsoc or any enrolled device
    pmabry@sheepsoc:~$ tailscale status
    ```

    The new device should appear with a `100.x.x.x` address.

### Generate a reusable auth key (if needed)

If enrolling a headless device that cannot complete an interactive browser login, generate a pre-auth key in the admin console:

1. Go to [https://login.tailscale.com/admin/settings/keys](https://login.tailscale.com/admin/settings/keys).
2. Click **Generate auth key**.
3. Choose whether the key is one-time-use or reusable. For a server, a one-time key is safer.
4. Copy the key and use it in the `tailscale up --authkey=<auth-key>` command on the new device.
5. Delete the key from the admin console after enrollment if it was not already marked one-time.

!!! warning "Key Expiry"
    Auth keys expire after the period configured in the admin console (default: 90 days). Devices that fail to re-authenticate before key expiry will disconnect. For long-running servers, configure key expiry to match your maintenance window cadence, or use ephemeral keys and re-enroll after each reinstall.

---

## 2. Remove a Device from the Tailnet

Use this procedure when a device is decommissioned or should no longer have access to sheepsoc.

### Option A — From the admin console (preferred)

1. Go to [https://login.tailscale.com/admin/machines](https://login.tailscale.com/admin/machines).
2. Find the device in the list.
3. Click the **...** menu next to the device.
4. Click **Remove**.

The device will immediately lose tailnet access. Any active WireGuard sessions to sheepsoc from that device will drop within seconds.

### Option B — From the device itself

If you have access to the device you want to remove:

```bash
sudo tailscale logout
```

This deauthenticates the device from the tailnet. The device entry will be marked inactive in the admin console and can be removed from there as well.

---

## 3. Rotate sheepsoc's Node Key

Tailscale node keys expire periodically (configurable in the admin console; the default is 180 days). When a node key is about to expire, the admin console will warn you and the device will prompt for re-authentication.

To renew sheepsoc's node key manually before expiry:

```bash
pmabry@sheepsoc:~$ sudo tailscale up --force-reauth
```

This re-authenticates sheepsoc to the coordination server using Google SSO. A browser URL will be printed to stdout — open it on any device to complete authentication. Alternatively, if MagicDNS is working, you can open the URL from the MacBook without needing physical access to sheepsoc.

After re-authentication, verify:

```bash
pmabry@sheepsoc:~$ tailscale status
```

The `Expiry` field in the admin console should update to show the new key validity period.

---

## 4. Full Uninstall

Use this procedure to completely remove Tailscale from sheepsoc. After this procedure, sheepsoc will no longer be reachable over the tailnet.

!!! danger "Remote Access Warning"
    If you are performing this procedure over SSH via the tailnet (i.e., connected to `100.117.117.43`), your session will be terminated when `tailscale down` runs. Use a LAN connection (`192.168.50.100`) or open a second LAN SSH session before proceeding.

### Step 1 — Disconnect from the tailnet

```bash
pmabry@sheepsoc:~$ sudo tailscale down
pmabry@sheepsoc:~$ sudo tailscale logout
```

`tailscale down` brings the WireGuard interface down and terminates active tunnels. `tailscale logout` deauthenticates the node from the coordination server.

### Step 2 — Remove the package

```bash
pmabry@sheepsoc:~$ sudo apt-get purge tailscale
```

This removes the `tailscale` package and the `tailscaled` daemon. The systemd unit is also removed.

### Step 3 — Remove persistent data

```bash
pmabry@sheepsoc:~$ sudo rm -rf /var/lib/tailscale
```

This removes the WireGuard keys, node state, and authentication credentials stored on disk. After this step, the prior node identity cannot be recovered.

### Step 4 — Remove the apt repository

```bash
pmabry@sheepsoc:~$ sudo rm /etc/apt/sources.list.d/tailscale.list
pmabry@sheepsoc:~$ sudo rm /usr/share/keyrings/tailscale-archive-keyring.gpg
```

### Step 5 — Remove the sysctl config

```bash
pmabry@sheepsoc:~$ sudo rm /etc/sysctl.d/99-tailscale.conf
pmabry@sheepsoc:~$ sudo sysctl --system
# Verify net.ipv4.ip_forward is no longer explicitly set to 1 (unless another config sets it)
```

### Step 6 — Remove the UFW rule

```bash
pmabry@sheepsoc:~$ sudo ufw delete allow in on tailscale0
pmabry@sheepsoc:~$ sudo ufw status verbose
# Confirm the tailscale0 rule is gone
```

### Step 7 — Remove from the admin console

1. Go to [https://login.tailscale.com/admin/machines](https://login.tailscale.com/admin/machines).
2. Find `sheepsoc` in the device list (it will appear as offline or disconnected after Step 1).
3. Click **...** → **Remove**.

### Verify

After all steps, sheepsoc should no longer appear as an active machine in the admin console, the `tailscale0` interface should not exist on the host, and the UFW rule should be absent:

```bash
pmabry@sheepsoc:~$ ip link show tailscale0
# Expected: "Device "tailscale0" does not exist."

pmabry@sheepsoc:~$ sudo ufw status verbose | grep tailscale
# Expected: no output
```

---

---

## 5. Managing Tailscale Serve Rules

`tailscale serve` proxies local HTTP services to tailnet peers over HTTPS. The rules are written to Tailscale's persistent state and survive restarts — no systemd unit needed.

### Where the State Lives

Serve configuration is stored by `tailscaled` at `/var/lib/tailscale/`. It is **not** a file you edit directly — always use the `tailscale serve` CLI to modify it.

### Verify Current Config

```bash
pmabry@sheepsoc:~$ tailscale serve status
```

This lists all active serve rules: which HTTPS ports are in use and which backend each one proxies to.

### Add a New Service to Serve

```bash
# Expose a local service on a chosen HTTPS port
# Format: sudo tailscale serve --bg --https=<port> http://localhost:<local-port>
pmabry@sheepsoc:~$ sudo tailscale serve --bg --https=<port> http://localhost:<local-port>
```

The `--bg` flag writes the rule to persistent state immediately. The rule takes effect without restarting any service. After adding, verify with `tailscale serve status`.

Example — to expose a service running on port 9090 at `https://sheepsoc-1.tail0f68e4.ts.net:9090/`:

```bash
pmabry@sheepsoc:~$ sudo tailscale serve --bg --https=9090 http://localhost:9090
```

!!! note "Auth is your responsibility"
    `tailscale serve` adds no authentication of its own. Ensure the service at the local port has its own auth before exposing it, or explicitly acknowledge the exposure is intentional if left unauthenticated.

### Remove a Specific Rule

```bash
# Remove the serve rule on a specific HTTPS port
pmabry@sheepsoc:~$ sudo tailscale serve --https=<port> off
```

This removes only the Tailscale proxy layer. The underlying local service is unaffected.

### Remove All Serve Rules

```bash
pmabry@sheepsoc:~$ sudo tailscale serve reset
```

This clears the entire serve configuration. All three currently-active rules (ports 443, 8443, 10000) would be removed by this command. Use it with care.

### Persistence Behaviour

Serve rules are written to `/var/lib/tailscale/` by `tailscaled` and restored automatically at daemon startup. They survive:

- `tailscaled` restarts
- System reboots

They do **not** survive a full Tailscale uninstall (see [section 4 — Full Uninstall](#4-full-uninstall), which removes `/var/lib/tailscale/`). After a reinstall, the serve rules must be re-added manually.

---

## See Also

- [Tailscale](../platforms/tailscale.md) — platform page covering configuration, rationale, health checks, and the full Serve configuration
- [Services](../services.md) — service catalog entry for `tailscaled.service`
- [Topology](../topology.md) — network context, including the Starlink CGNAT situation
