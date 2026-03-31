# VPS Reverse Proxy Infrastructure Setup

**Omenabyte Intelligence — VPS Infrastructure Standard**

---

## Overview

This document details the complete setup of a reverse proxy architecture using **Caddy + Docker + Coolify** on a VPS (Ubuntu 24.04). The architecture ensures zero port conflicts while providing a unified entry point for all services.

---

## Architecture Diagram

```
                            Internet
                               │
                    ┌──────────┴──────────┐
                    │   Cloudflare DNS    │
                    │  (omenabyte.com)    │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │  Caddy (Ports 80,443)│ ← Public HTTPS endpoint
                    │  System Service      │
                    └──────────┬──────────┘
                               │
    ┌──────────────────────────┼──────────────────────────┐
    │                          │                          │
    ▼                          ▼                          ▼
┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐
│ Coolify Dashboard│   │ Coolify-managed │   │  Direct Routes  │
│ coolify.xxx     │   │   apps via      │   │ (Portainer,     │
│ localhost:8000  │   │ localhost:8000  │   │   Agent Zero)   │
│                 │   │                 │   │ localhost:5080  │
│ WS: :6001, :6002│   │                 │   │ 10.0.2.2:9000   │
└─────────────────┘   └─────────────────┘   └─────────────────┘
    │                          │                          │
    └──────────────────────────┼──────────────────────────┘
                               │
                    ┌──────────┴──────────┐
                    │  Docker Bridge Network│
                    │  Coolify Services     │
                    │  - coolify (8000)     │
                    │  - coolify-db         │
                    │  - coolify-redis      │
                    │  - coolify-realtime   │
                    │  - coolify-sentinel   │
                    │  - portainer (9000)   │
                    │  - agent-zero (5080)  │
                    └───────────────────────┘
```

---

## Port Allocation Table

| Port | Bound To   | Owner        | Purpose                                    |
|------|------------|--------------|--------------------------------------------|
| 80   | 0.0.0.0    | Caddy        | HTTP → HTTPS redirect                      |
| 443  | 0.0.0.0    | Caddy        | Public HTTPS (TLS termination)            |
| 2019 | 127.0.0.1  | Caddy        | Admin API (localhost only)                 |
| 8000 | 0.0.0.0    | Coolify      | HTTP web interface / Coolify proxy        |
| 5080 | 0.0.0.0    | Agent Zero   | Web UI                                     |
| 9000 | 0.0.0.0    | Portainer    | Web UI                                     |
| 6001 | 0.0.0.0    | Coolify      | WebSocket realtime (Pusher)                |
| 6002 | 0.0.0.0    | Coolify      | WebSocket terminal                         |

**Note:** The official Coolify installer binds ports to 0.0.0.0 by default. This is acceptable when behind Caddy and Cloudflare.

---

## Domain Configuration Summary

| Service         | Domain                        | Routing Method      | Upstream        |
|-----------------|-------------------------------|--------------------|-----------------|
| Coolify         | coolify.omenabyte.com         | Caddy → localhost | localhost:8000 |
| Portainer       | portainer.omenabyte.com       | Direct (bypass)   | 10.0.2.2:9000   |
| Agent Zero      | agent0.omenabyte.com          | Direct (bypass)   | localhost:5080  |
| Coolify Apps    | app.yourdomain.com           | Caddy → Coolify   | localhost:8000  |

---

## Installation Steps

### 1. Prerequisites

```bash
# Update system
apt-get update && apt-get upgrade -y

# Install required packages
apt-get install -y curl wget git jq openssl
```

### 2. Install Docker

Using the official get.docker.com script:

```bash
curl -fsSL https://get.docker.com -o /tmp/get-docker.sh
sh /tmp/get-docker.sh

# Enable and start Docker
systemctl enable docker
systemctl start docker
```

### 3. Install Caddy (Reverse Proxy)

Add the official Caddy repository and install:

```bash
# Install prerequisites
apt-get install -y debian-keyring debian-archive-keyring apt-transport-https curl

# Add Caddy repository
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | tee /etc/apt/sources.list.d/caddy-stable.list

# Update and install
apt-get update
apt-get install -y caddy

# Enable and start
systemctl enable caddy
systemctl start caddy
```

### 4. Install Coolify

**Method: Official Installer (Recommended)**

```bash
# Use official installer (add --resolve if DNS issues)
curl --resolve cdn.coollabs.io:443:79.127.134.229 -fsSL https://cdn.coollabs.io/coolify/install.sh | bash
```

This automatically:
- Creates required Docker networks
- Sets up PostgreSQL, Redis, and Coolify containers
- Generates encryption keys
- Configures initial admin credentials

### 5. Configure Caddy

Create `/etc/caddy/Caddyfile`:

```caddyfile
# -----------------------------------------------
# Global Options
# -----------------------------------------------
{
    email admin@omenabyte.com
    admin localhost:2019
}

# -----------------------------------------------
# HTTP to HTTPS redirect
# -----------------------------------------------
:80 {
    redir https://{host}{uri}
}

# -----------------------------------------------
# Coolify Dashboard
# -----------------------------------------------
coolify.omenabyte.com {
    reverse_proxy localhost:8000

    # WebSocket proxy for realtime
    @ws_pusher {
        path /app /app/*
    }
    reverse_proxy @ws_pusher localhost:6001

    @ws_terminal {
        path /terminal /terminal/*
    }
    reverse_proxy @ws_terminal localhost:6002
}

# -----------------------------------------------
# Portainer (Direct route - bypass Coolify)
# -----------------------------------------------
portainer.omenabyte.com {
    reverse_proxy 10.0.2.2:9000
}

# -----------------------------------------------
# Agent Zero (Direct route - bypass Coolify)
# -----------------------------------------------
agent0.omenabyte.com {
    reverse_proxy 127.0.0.1:5080 {
        header_up X-Forwarded-Proto https
    }
}

# -----------------------------------------------
# Catch-all (all other domains go to Coolify)
# -----------------------------------------------
:443 {
    reverse_proxy localhost:8000 {
        header_up X-Forwarded-Proto https
    }
}
```

Validate and reload:

```bash
caddy validate --config /etc/caddy/Caddyfile
systemctl reload caddy
```

---

## DNS Configuration

### Cloudflare Setup

1. **Create A Records** for main services:
   - Type: `A`, Name: `coolify`, Content: `<VPS_IP>`, Proxy: `Proxied`
   - Type: `A`, Name: `portainer`, Content: `<VPS_IP>`, Proxy: `Proxied`
   - Type: `A`, Name: `agent0`, Content: `<VPS_IP>`, Proxy: `Proxied`

2. **Create CNAME** for Coolify-deployed apps:
   - Type: `CNAME`, Name: `<app-name>`, Content: `coolify.yourdomain.com`, Proxy: `Proxied`

### Important: TLS Setting

In Cloudflare SSL/TLS settings:
- **Mode**: Full (Strict)
- This ensures Cloudflare connects to your server using HTTPS, and Caddy provides valid certificates

---

## Adding Applications

### Type 1: Coolify-Managed Apps

For apps deployed through Coolify UI (managed by Coolify's proxy):

1. **Deploy in Coolify**:
   - Add Resource → Choose template
   - Configure and deploy
   - Set domain: `https://appname.yourdomain.com` (NO port!)

2. **No Caddy config needed** - Catch-all handles automatically

3. **Add DNS**: CNAME record pointing to VPS IP

### Type 2: Direct Route (Bypass Coolify)

For standalone containers not managed by Coolify:

1. **Find container IP or port**:
   ```bash
   docker ps
   docker port <container-name>
   ```

2. **Add Caddy block** (see examples below)

3. **Add DNS**: A record pointing to VPS IP

---

## Deployment Examples

### Portainer (via Coolify)

1. **In Coolify UI**:
   - Add Resource → Search "Portainer"
   - Deploy
   - **IMPORTANT**: Set domain to `https://portainer.yourdomain.com` (NO port!)
   - When warned about port, choose "Remove port anyway"

2. **Find Portainer IP**:
   ```bash
   docker inspect <portainer-container> --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
   ```

3. **Add to Caddy** (get container IP):
   ```
   portainer.omenabyte.com {
       reverse_proxy 10.0.2.2:9000
   }
   ```

4. **Access**: `https://portainer.omenabyte.com`

### Agent Zero (Standalone Docker)

1. **Container already running**:
   - Port: 5080 (mapped to container port 80)

2. **Add to Caddy**:
   ```
   agent0.omenabyte.com {
       reverse_proxy 127.0.0.1:5080 {
           header_up X-Forwarded-Proto https
       }
   }
   ```

3. **Access**: `https://agent0.omenabyte.com`

---

## Troubleshooting

### Check Caddy Status

```bash
systemctl status caddy
journalctl -u caddy -f
```

### Check Coolify Status

```bash
docker ps
docker logs coolify
```

### Find Container IP

```bash
docker inspect <container-name> --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
```

### Verify Port Allocation

```bash
ss -tlnp | grep -E '80|443|8000|5080|9000|6001|6002'
```

### Common Issues

| Error | Solution |
|-------|----------|
| 502 Bad Gateway | Check upstream service is running: `docker ps` |
| SSL handshake failed (525) | Add explicit domain block in Caddy |
| Routes to wrong app | Verify container IP or use direct port |
| Coolify redirect loop | Use HTTPS in domain: `https://app.domain.com` |
| Connection reset | Check container's internal service is listening |

---

## Maintenance

### Update Coolify

```bash
curl --resolve cdn.coollabs.io:443:79.127.134.229 -fsSL https://cdn.coollabs.io/coolify/install.sh | bash
```

### Backup Configuration

```bash
# Backup Coolify .env
cp /data/coolify/source/.env ~/coolify-backup.env

# Backup Caddy config
cp /etc/caddy/Caddyfile ~/caddy-backup.conf
```

### Restart Services

```bash
# Restart Coolify
cd /data/coolify/source && docker compose restart

# Restart Caddy
systemctl reload caddy

# Restart specific container
docker restart <container-name>
```

---

## Security Considerations

1. **Keep ports internal**: Coolify binds to 0.0.0.0 by default - acceptable behind Caddy/Cloudflare
2. **Firewall**: Only ports 80/443 need to be publicly accessible
3. **Cloudflare**: Proxy all traffic through Cloudflare for additional security
4. **Updates**: Keep Docker, Caddy, and Coolify updated
5. **Direct routes**: Only use for standalone apps; Coolify-managed apps should go through Coolify proxy

---

## File Locations

| Component | Location |
|-----------|----------|
| Caddy config | `/etc/caddy/Caddyfile` |
| Coolify compose | `/data/coolify/source/docker-compose.yml` |
| Coolify env | `/data/coolify/source/.env` |
| Coolify data | `/data/coolify/` |
| Caddy certificates | `/var/lib/caddy/.local/share/caddy` |

---

## Container Network Reference

| Container | Network | IP Address |
|-----------|---------|-------------|
| coolify | coolify | (internal) |
| coolify-db | coolify | (internal) |
| coolify-redis | coolify | (internal) |
| coolify-realtime | coolify | (internal) |
| portainer | root_default | 10.0.2.2 |
| agent-zero | bridge | 10.0.0.3 |

*Maintained by Omenabyte Intelligence*
*Last updated: 2026-03-31*