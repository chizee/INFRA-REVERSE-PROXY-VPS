# Portainer, Hermes & Services Reverse Proxy Fix Documentation

**Date:** April 14-15, 2026  
**Author:** Generated through troubleshooting session  
**Context:** VPS infrastructure with Caddy + Coolify proxy + Docker reverse proxy setup

---

## Executive Summary

This document details the issues encountered when the reverse proxy infrastructure became unavailable after a container restart, and how each service (Portainer, Hermes, Coolify, Agent Zero, OpenClaw) was restored to working condition. The issues stemmed from port conflicts, TLS certificate problems, and network routing changes.

---

## Initial Problem Statement

After restarting the container stack, all services became inaccessible:

- `https://coolify.omenabyte.com` → Error 521 (Web server down)
- `https://agent0.omenabyte.com/login` → Error 521 (Web server down)
- `https://vmi3108871.tail9f9e73.ts.net/portainer` → ERR_CONNECTION_REFUSED

---

## Root Cause Analysis

### Issue 1: Port Conflict - Coolify Proxy Grabbed Ports 80/443

**Symptom:** Caddy couldn't bind to ports 80 and 443, causing all HTTPS traffic to fail.

**Investigation:**
```bash
ss -tlnp | grep -E "(80|443)"
# Output showed docker-proxy holding ports:
# 0.0.0.0:80 -> docker-proxy (coolify-proxy)
# 0.0.0.0:443 -> docker-proxy (coolify-proxy)
```

**Root Cause:** Coolify deployed its own Traefik proxy container (`coolify-proxy`) that was configured to bind to public ports 80 and 443, conflicting with the standalone Caddy service.

**Solution:** Reconfigure Coolify proxy to use localhost-only ports.

---

### Issue 2: Portainer Not on Coolify Network

**Symptom:** Portainer responded with 404 errors when routed through Caddy.

**Investigation:**
```bash
docker inspect portainer --format '{{json .NetworkSettings.Networks}}'
# Output showed portainer only on 'bridge' network, not on 'coolify' network
```

**Root Cause:** Portainer was deployed outside Coolify's Docker network, making it unreachable from the Coolify proxy.

**Solution:** Connect Portainer to Coolify network:
```bash
docker network connect coolify portainer-xxxx
```

---

### Issue 3: Caddy TLS Certificate Storage Failure

**Symptom:** TLS handshake failures with "permission denied" errors.

**Investigation:**
```bash
journalctl -u caddy | grep "permission denied"
# Output: "unable to create folder for config autosave /var/lib/caddy: permission denied"
```

**Root Cause:** The `/var/lib/caddy` directory was owned by root instead of the `caddy` user.

**Solution:** Fix directory ownership:
```bash
rm -rf /var/lib/caddy /var/log/caddy
mkdir -p /var/lib/caddy/.local/share/caddy /var/lib/caddy/.config/caddy
chown -R caddy:caddy /var/lib/caddy /var/log/caddy
chmod 755 /var/lib/caddy
```

---

### Issue 4: Hermes Dashboard Path Issues

**Symptom:** Hermes dashboard shows blank white screen.

**Root Cause:** 
1. Hermes serves absolute paths (`/assets/`) but accessing via proxy subdirectory (`/hermes`) breaks asset loading
2. Tailscale Magic DNS doesn't support custom subdomains like `subdomain.tail9f9e73.ts.net`

**Solution:** Access Hermes directly via its Tailscale IP (simpler and more reliable):
```bash
curl http://100.86.x.x:9119
```

---

## Solutions Applied (Step by Step)

### Fix 1: Reconfigure Coolify Proxy to Internal Ports

**File:** `/data/coolify/proxy/docker-compose.yml`

**Before:**
```yaml
ports:
  - '80:80'
  - '443:443'
  - '443:443/udp'
  - '8080:8080'
```

**After:**
```yaml
# Bind to localhost only - Caddy owns public ports 80/443
ports:
  - '127.0.0.1:8001:80'
  - '127.0.0.1:8443:443'
  - '127.0.0.1:8080:8080'
```

**Apply:**
```bash
cd /data/coolify/proxy && docker compose up -d --force-recreate
```

---

### Fix 2: Connect Portainer to Coolify Network

```bash
docker network connect coolify portainer-xxxx
```

---

### Fix 3: Generate Self-Signed Origin Certificate

Since Let's Encrypt failed due to the port conflicts, create a self-signed certificate:

```bash
# Generate origin certificate
openssl genrsa 2048 > /tmp/origin.key
openssl req -new -key /tmp/origin.key -subj "/CN=yourdomain.com" -out /tmp/origin.csr
openssl x509 -req -in /tmp/origin.csr -signkey /tmp/origin.key -out /tmp/origin.crt -days 365

# Install to Caddy
cp /tmp/origin.crt /etc/caddy/certs/origin.crt
cp /tmp/origin.key /etc/caddy/certs/origin.key
chown caddy:caddy /etc/caddy/certs/*
```

---

## Current Working Configuration

### Port Allocation Table

| Port | Owner | Use |
|------|-------|-----|
| 80 | Caddy | HTTP → HTTPS redirect |
| 443 | Caddy | HTTPS TLS termination |
| 8000 | Coolify | App backend |
| 8001 | Coolify proxy (internal) | Internal HTTP |
| 8443 | Coolify proxy (internal) | Internal HTTPS |
| 5080 | Agent Zero | Web UI |
| 9119 | Hermes | Web dashboard (native) |
| 18789 | OpenClaw | Gateway/Control UI |

### Working Service URLs

| Service | URL | Access Via |
|--------|-----|------------|
| Coolify | `https://coolify.yourdomain.com` | Domain |
| Agent Zero | `https://agent0.yourdomain.com/login` | Domain |
| OpenClaw | `https://openclaw.yourdomain.com/` | Domain |
| Portainer | `https://vmiXXXXXX.tailXXXXXX.ts.net/` | Tailscale |
| Hermes | `http://100.86.x.x:9119` | Tailscale (direct IP) |

---

## Caddy Configuration

**File:** `/etc/caddy/Caddyfile`

```caddyfile
{
    email admin@yourdomain.com
    admin localhost:2019
}

coolify.yourdomain.com {
    reverse_proxy 127.0.0.1:8000
    tls /etc/caddy/certs/origin.crt /etc/caddy/certs/origin.key
}

portainer.yourdomain.com {
    reverse_proxy CONTAINER_IP:9000
    tls /etc/caddy/certs/origin.crt /etc/caddy/certs/origin.key
}

agent0.yourdomain.com {
    reverse_proxy 127.0.0.1:5080
    tls /etc/caddy/certs/origin.crt /etc/caddy/certs/origin.key
}

openclaw.yourdomain.com {
    reverse_proxy 127.0.0.1:18789
    tls /etc/caddy/certs/origin.crt /etc/caddy/certs/origin.key
}

# Tailscale URL for Portainer
vmiXXXXXX.tailXXXXXX.ts.net {
    reverse_proxy CONTAINER_IP:9000
    tls /etc/caddy/certs/origin.crt /etc/caddy/certs/origin.key
}
```

**Note:** Replace `yourdomain.com`, `CONTAINER_IP`, and `vmiXXXXXX.tailXXXXXX.ts.net` with your actual values.

---

## How to Find Container IPs

```bash
# For Docker containers
docker inspect CONTAINER_NAME --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'

# For native services
ss -tlnp | grep SERVICE_PORT
```

---

## Troubleshooting Commands

### Check What's Using Ports 80/443
```bash
ss -tlnp | grep -E "(80|443)"
```

### Check Caddy Status
```bash
systemctl status caddy
journalctl -u caddy -f
```

### Test Services Locally
```bash
# Test Coolify
curl -I http://127.0.0.1:8000/

# Test Portainer
curl -I http://CONTAINER_IP:9000/

# Test Hermes
curl -I http://100.86.x.x:9119/
```

### Verify Caddy Routing
```bash
curl -IH "Host: coolify.yourdomain.com" https://127.0.0.1:443/
```

---

## Key Lessons Learned

1. **Never let Coolify bind to ports 80/443** - Always configure Coolify proxy to use localhost-only ports to avoid conflicts with the main reverse proxy.

2. **Services on Coolify network must be connected** - Any service that needs to be routed through Coolify must be on the `coolify` Docker network.

3. **Directory permissions matter** - Caddy needs `/var/lib/caddy` owned by `caddy:caddy` to store TLS certificates.

4. **Tailscale Magic DNS limitations** - Custom subdomains don't resolve; use path-based routing or direct IP access.

5. **Path-based proxy routing complications** - Applications serving absolute paths can break when accessed via proxy subdirectories. Direct IP access is simpler when possible.

---

## Cloudflare Configuration (If Using)

If using Cloudflare in front of this VPS:

1. Set SSL/TLS mode to **Full (Strict)**
2. Use a Cloudflare Origin Certificate on Caddy (or the self-signed approach above)
3. Ensure DNS A records point to your VPS IP

For production, consider setting up Let's Encrypt with the Cloudflare DNS challenge for automatic certificate renewal.

---

## Maintenance Notes

### Restart Order
```bash
# 1. Restart Coolify proxy first
cd /data/coolify/proxy && docker compose restart

# 2. Then restart Caddy
systemctl restart caddy
```

### Updating Coolify
```bash
curl --resolve cdn.coollabs.io:443:ORIGIN_IP -fsSL https://cdn.coollabs.io/coolify/install.sh | bash
```

---

*Document generated: April 2026*
*For infrastructure reference - Omenabyte Intelligence*