# Future Improvements

**Purpose:** Planned enhancements and improvements to the sheepsoc system.

| Key | Value |
|---|---|
| Scope | Items here are researched and design-ready but not yet implemented |
| Updated | 2026-04-24 |

Planned enhancements and improvements to the sheepsoc system. Each entry documents the rationale, cost, implementation steps, and current status so work can be picked up without re-doing the research phase.

## Cloudflare Tunnel

| Key | Value |
|---|---|
| Status | Planned — backlog |
| Effort | Low — estimated 1–2 hours once domain decision is made |
| Cost | Free (Cloudflare Zero Trust free tier) · optional domain ~$8–$12/year |
| Tool | `cloudflared` daemon · Cloudflare Zero Trust dashboard |

### What It Is

Cloudflare Tunnel (`cloudflared`) creates an outbound encrypted connection from sheepsoc to Cloudflare's edge network, making internal services accessible from anywhere on the internet without port forwarding, a public IP, or an inbound firewall rule. The tunnel is initiated by sheepsoc, so no router configuration is required.

Cloudflare Access sits in front of every exposed service. Visitors hit a Cloudflare-managed authentication page (Google OAuth, email OTP, or other identity providers) before any traffic reaches sheepsoc. No service is exposed unauthenticated.

### Why This Is Needed

Sheepsoc is behind Starlink CGNAT — there is no real public IP address, and port forwarding is not possible from the router. Remote access to services such as OpenWebUI and Jupyter currently requires being on the LAN.

Tailscale was previously installed as a solution to this problem but was removed after approximately two months of failure. That failure is now believed to have been caused by the broken gateway and DNS configuration that was discovered and fixed on 2026-04-18 (gateway was pointing to a dead host; DNS was broken). Tailscale remains a viable alternative and is worth retrying under the corrected network configuration.

Cloudflare Tunnel is the preferred solution because it requires no ongoing maintenance of a relay node, has no bandwidth limits on the free tier, and integrates a full identity-aware access control layer at no additional cost.

### Cost

| Item | Cost | Notes |
|---|---|---|
| Cloudflare Zero Trust (tunnels + access policies) | Free | Free tier covers unlimited tunnels, up to 50 users |
| Custom domain name | ~$8–$12/year | Cloudflare sells domains at cost — no markup. Optional. |
| Temporary subdomain (no domain purchase) | Free | `trycloudflare.com` subdomains — but URLs change on every tunnel restart |

!!! note "Recommendation"
    Purchasing a domain is strongly recommended. The `trycloudflare.com` subdomain approach is useful for quick tests but the URL changes every time the tunnel restarts, making it impractical for bookmarks or API integrations.

### Services to Expose

| Service | Internal Address | Public URL (example) |
|---|---|---|
| **OpenWebUI** | `localhost:8080` | `openwebui.yourdomain.com` |
| **Jupyter Notebook** | `localhost:8888` | `jupyter.yourdomain.com` |
| **SSH** | `localhost:22` | Via `cloudflared access ssh` command — not a web URL |

Kibana (port 5601) and Elasticsearch (port 9200) should **not** be exposed publicly even behind Cloudflare Access, as they are internal observability tools and the Elasticsearch REST API should not have a public surface. Vikunja could be exposed if remote task management access is needed.

### Security Model

Every hostname exposed through Cloudflare Tunnel is protected by a Cloudflare Access policy. The flow for a remote visitor is:

1. Visitor requests `openwebui.yourdomain.com`
2. Cloudflare intercepts the request at the edge before it reaches sheepsoc
3. Cloudflare Access presents a login page (Google OAuth, email OTP, or other configured provider)
4. On successful authentication, Cloudflare forwards the request through the tunnel to `localhost:8080` on sheepsoc
5. OpenWebUI responds — traffic is returned through the tunnel, encrypted

Sheepsoc's UFW firewall is unchanged. No new inbound rules are needed. The tunnel connection from sheepsoc to Cloudflare is outbound on port 443, which is already permitted.

!!! warning "Before Exposing Jupyter"
    Jupyter Notebook currently runs without token authentication in the `sheepsoc` conda environment (the token can be retrieved from `journalctl -u jupyter`). Before exposing Jupyter externally, verify the token or password requirement is active, or enable Cloudflare Access with a strict policy on that hostname. An unauthenticated Jupyter instance provides full shell access to the server.

### Implementation Steps

1. Create a free Cloudflare account and enable the Zero Trust dashboard at `one.dash.cloudflare.com`
2. Decide on domain strategy: purchase a domain via Cloudflare Registrar, or plan to use temporary `trycloudflare.com` URLs for initial testing
3. If using a custom domain, point its nameservers to Cloudflare (done automatically if domain is purchased through Cloudflare)
4. Install `cloudflared` on sheepsoc:

    ```bash
    pmabry@sheepsoc:~$ curl -L --output cloudflared.deb \
        https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
    pmabry@sheepsoc:~$ sudo dpkg -i cloudflared.deb
    ```

5. Authenticate `cloudflared` with your Cloudflare account (this opens a browser for OAuth):

    ```bash
    pmabry@sheepsoc:~$ cloudflared tunnel login
    ```

6. Create a named tunnel:

    ```bash
    pmabry@sheepsoc:~$ cloudflared tunnel create sheepsoc
    ```

    This creates a tunnel UUID and credentials file at `~/.cloudflared/<UUID>.json`.

7. Write the tunnel config at `~/.cloudflared/config.yml`, mapping each service to a public hostname:

    ```yaml
    tunnel: sheepsoc
    credentials-file: /home/pmabry/.cloudflared/<UUID>.json

    ingress:
      - hostname: openwebui.yourdomain.com
        service: http://localhost:8080
      - hostname: jupyter.yourdomain.com
        service: http://localhost:8888
      - service: http_status:404
    ```

8. In the Cloudflare Zero Trust dashboard, create DNS CNAME records pointing each hostname to the tunnel (or run `cloudflared tunnel route dns sheepsoc openwebui.yourdomain.com` for each)
9. Create a Cloudflare Access application and policy for each exposed hostname. Set the identity provider (Google OAuth is the simplest), and restrict access to your email address
10. Install `cloudflared` as a systemd service so the tunnel starts on boot:

    ```bash
    pmabry@sheepsoc:~$ sudo cloudflared service install
    ```

11. Verify outbound port 443 is permitted in UFW (almost certainly already open, but confirm):

    ```bash
    pmabry@sheepsoc:~$ sudo ufw status verbose | grep 443
    ```

12. Test from outside the LAN — a mobile device on cellular data is simplest

### SSH Access via Cloudflare Tunnel

SSH through a Cloudflare Tunnel works differently from web services — it requires installing `cloudflared` on the client machine as well and using it as a ProxyCommand. The client-side setup is:

```bash
# On the client machine (not sheepsoc) — add to ~/.ssh/config
Host sheepsoc.yourdomain.com
    ProxyCommand cloudflared access ssh --hostname %h
```

Then connect normally: `ssh pmabry@sheepsoc.yourdomain.com`. Cloudflare Access will prompt for authentication in the browser on first use, then cache the session token.

### Alternatives Considered

| Alternative | Why Not Chosen (currently) |
|---|---|
| **Tailscale** | Was installed and removed after two months of failure. Worth retrying now that DNS and gateway are fixed (2026-04-18). Simpler client setup than Cloudflare Tunnel for SSH in particular. |
| **WireGuard + VPS relay** | Works but adds ~$5/month VPS cost and requires maintaining a relay node. More operational overhead for the same capability. |
| **ngrok** | Similar architecture to Cloudflare Tunnel but the free tier is more restrictive (connection limits, no custom domains, sessions expire). Cloudflare is strictly better at this use case. |
