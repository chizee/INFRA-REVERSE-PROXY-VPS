# Reverse Proxy Architecture Guide
**Omenabyte Intelligence — VPS Infrastructure Standard**

---

## Overview

This document defines the standard reverse proxy architecture for any new Omenabyte VPS instance. It uses a **hybrid Caddy + Coolify** pattern where Caddy is the single public edge, and Coolify manages internal routing for its own services.

---

## Architecture

```
Internet (80/443)
        │
   Caddy (standalone)
        │
        ├── coolify.domain.com     → Coolify dashboard     (localhost:8443)
        │
        ├── app1.domain.com        → Coolify's proxy        (localhost:8080)
        ├── app2.domain.com        ↑ (host-based routing,  Coolify handles internally)
        │
        └── other.domain.com       → Standalone service     (localhost:PORT)
```

**Rule:** Only Caddy owns ports 80 and 443. Everything else is internal.

---

## Components

### 1. Caddy (Standalone — System Service)

- Installed directly on the host, **not** in Docker
- Owns ports `80`, `443`, and `2019` (admin API, localhost only)
- Handles all public TLS (via Let's Encrypt or Cloudflare DNS challenge)
- Acts as the edge proxy for **both** Coolify-managed and standalone services

### 2. Coolify

- Runs in Docker, managed by its own compose stack
- Dashboard exposed on `localhost:8443` (not public directly)
- Runs its own internal proxy (Traefik or Caddy container) on a non-public port (e.g., `8080`)
- Coolify's proxy handles **host-based routing** for all services deployed through it
- Coolify's proxy must **not** bind to ports 80 or 443

### 3. Standalone Services (outside Coolify)

- Installed manually or via Docker, exposed on `localhost:PORT` only
- Routed publicly via a dedicated block in the Caddy config
- Completely independent of Coolify's proxy layer

---

## Port Allocation

| Port  | Bound To    | Owner              | Purpose                        |
|-------|-------------|--------------------|--------------------------------|
| 80    | 0.0.0.0     | Caddy (standalone) | HTTP → HTTPS redirect          |
| 443   | 0.0.0.0     | Caddy (standalone) | Public HTTPS (TLS termination) |
| 2019  | 127.0.0.1   | Caddy (standalone) | Caddy admin API (local only)   |
| 8443  | 127.0.0.1   | Coolify            | Dashboard HTTPS                |
| 8080  | 127.0.0.1   | Coolify's proxy    | Internal app routing           |
| 9000  | 127.0.0.1   | Coolify agent      | Agent communication            |
| XXXX  | 127.0.0.1   | Standalone service | Custom per-service             |

---

## Caddy Configuration Pattern

### `/etc/caddy/Caddyfile`

```caddyfile
# -----------------------------------------------
# Global Options
# -----------------------------------------------
{
    email admin@yourdomain.com
    admin localhost:2019
}

# -----------------------------------------------
# Coolify Dashboard
# -----------------------------------------------
coolify.yourdomain.com {
    reverse_proxy localhost:8443 {
        transport http {
            tls_insecure_skip_verify  # if Coolify uses self-signed cert internally
        }
    }
}

# -----------------------------------------------
# Coolify-managed apps (single upstream block)
# Caddy passes Host header → Coolify's proxy routes internally
# -----------------------------------------------
app1.yourdomain.com {
    reverse_proxy localhost:8080
}

app2.yourdomain.com {
    reverse_proxy localhost:8080
}

# -----------------------------------------------
# Standalone services (outside Coolify)
# -----------------------------------------------
othertool.yourdomain.com {
    reverse_proxy localhost:3000
}
```

> **Key principle for Coolify apps:** Pass the `Host` header through to `localhost:8080`. Coolify's proxy reads it and routes to the correct container. Do not bypass Coolify's proxy for services it manages.

---

## Coolify Setup Checklist

When deploying Coolify on a new VPS:

- [ ] Disable Coolify's proxy from binding to ports 80/443
- [ ] Set Coolify's proxy to listen on an internal port only (e.g., `8080`)
- [ ] Expose Coolify dashboard on `localhost:8443` (not public)
- [ ] Confirm no Coolify container has `ports: ["80:80"]` or `ports: ["443:443"]` in its compose config
- [ ] Add Coolify subdomain block to Caddyfile before going live

---

## Caddy Setup Checklist

When setting up Caddy on a new VPS:

- [ ] Install Caddy as a system service (`apt` or official install script)
- [ ] Enable and start: `systemctl enable caddy && systemctl start caddy`
- [ ] Confirm Caddy owns 80/443: `ss -tlnp | grep -E '80|443'`
- [ ] Place config at `/etc/caddy/Caddyfile`
- [ ] Reload after changes: `systemctl reload caddy`
- [ ] Check logs: `journalctl -u caddy -f`

---

## Adding a New Service

### If deployed via Coolify:
1. Deploy the service in Coolify as normal
2. Set its domain in Coolify's service settings
3. Add a Caddy block pointing to `localhost:8080` with the same domain
4. Reload Caddy — done

### If deployed standalone (outside Coolify):
1. Install and run the service, bind it to `localhost:PORT`
2. Add a new Caddyfile block:
   ```caddyfile
   newservice.yourdomain.com {
       reverse_proxy localhost:PORT
   }
   ```
3. Reload Caddy: `systemctl reload caddy`

---

## TLS Notes

- Caddy auto-provisions and renews Let's Encrypt certs by default
- For wildcard certs, use the Cloudflare DNS challenge plugin (`caddy-dns/cloudflare`)
- When using Cloudflare proxy (orange cloud), set SSL mode to **Full (Strict)** and use a Cloudflare Origin Certificate in Caddy instead of Let's Encrypt
- Coolify's internal proxy does **not** need valid public certs — Caddy handles TLS at the edge

---

## What NOT to Do

- Do not let Coolify bind to 80/443 — it will conflict with Caddy
- Do not install Caddy as a Docker container — it creates networking complexity at the edge
- Do not expose container ports directly to `0.0.0.0` — always bind to `127.0.0.1:PORT`
- Do not add a Coolify service to Caddy by proxying directly to its container port — route through Coolify's proxy so dynamic routing works

---

*Maintained by Omenabyte Intelligence — update this doc when infrastructure patterns change.*
