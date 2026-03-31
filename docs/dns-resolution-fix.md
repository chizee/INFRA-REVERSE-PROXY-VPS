# VPS DNS Resolution Issue - Troubleshooting & Fix

> **Date**: April 1, 2026  
> **Issue**: GitHub/npm/Cloudflare blocked while other sites work  

---

## Problem Description

### Symptoms
- `npx skills add` and `git clone` from GitHub **failed**
- `curl https://github.com` **timed out**
- `dig @PROVIDER_DNS github.com` **timed out**
- **Ping to GitHub worked** (low latency)
- **Google DNS (8.8.8.8) resolved GitHub correctly**

### Error Messages
```
fatal: unable to access 'https://github.com/vercel-labs/skills.git/': 
Could not resolve host: github.com

curl: (28) Resolving timed out after 10000 milliseconds
```

---

## Root Cause Analysis

### DNS Server Failure

The VPS hosting provider assigns DNS servers that **cannot resolve external domains properly**:

| DNS Server | Status |
|------------|--------|
| `PROVIDER_DNS_1` | ❌ Timeouts on GitHub queries |
| `PROVIDER_DNS_2` | ❌ Timeouts on GitHub queries |
| `PROVIDER_DNS_IPV6` | ❌ IPv6 also failing |
| `1.1.1.1` (Cloudflare) | ✅ Works |
| `8.8.8.8` (Google) | ✅ Works |

### Why Ping Worked But HTTPS Failed
- **Ping** uses ICMP and bypasses DNS (hosts file or direct IP)
- **HTTPS/curl/npm** requires DNS resolution first, which timed out

### Technical Details
```
# DNS resolution test - FAILS
$ dig @PROVIDER_DNS github.com
;; communications error to PROVIDER_DNS#53: timed out

# DNS resolution test - WORKS  
$ dig @1.1.1.1 github.com
RESOLVED_IP

# HTTPS test - FAILS without fix
$ curl https://github.com
curl: (28) Resolving timed out after 10000 milliseconds

# HTTPS test - WORKS after fix
$ curl https://github.com
Connected to github.com (RESOLVED_IP) port 443
```

---

## Solution Applied

### Files Modified

#### 1. Netplan Configuration
**File**: `/etc/netplan/50-cloud-init.yaml`

**Before**:
```yaml
nameservers:
  addresses:
  - PROVIDER_DNS_1
  - PROVIDER_DNS_2
  - PROVIDER_DNS_IPV6
```

**After**:
```yaml
nameservers:
  addresses:
  - 1.1.1.1      # Cloudflare Primary
  - 1.0.0.1      # Cloudflare Secondary
  - 8.8.8.8      # Google Primary
  - 8.8.4.4      # Google Secondary
```

#### 2. Systemd-Resolved Fallback
**File**: `/etc/systemd/resolved.conf.d/cloudflare-dns.conf` (created)

```ini
# Cloudflare DNS configuration
# Primary: Cloudflare (1.1.1.1, 1.0.0.1) - fastest, privacy-focused
# Fallback: Google (8.8.8.8, 8.8.4.4) - reliable backup
[Resolve]
DNS=1.1.1.1 1.0.0.1
FallbackDNS=8.8.8.8 8.8.4.4
```

### Commands Executed

```bash
# 1. Create systemd-resolved fallback config
sudo mkdir -p /etc/systemd/resolved.conf.d
sudo tee /etc/systemd/resolved.conf.d/cloudflare-dns.conf << 'EOF'
[Resolve]
DNS=1.1.1.1 1.0.0.1
FallbackDNS=8.8.8.8 8.8.4.4
EOF

# 2. Update netplan (replace bad DNS with Cloudflare + Google)
sudo vim /etc/netplan/50-cloud-init.yaml

# 3. Apply netplan changes
sudo netplan apply

# 4. Restart systemd-resolved
sudo systemctl restart systemd-resolved
```

---

## Verification

### Pre-Fix Tests (Failed)
```bash
$ curl --max-time 10 https://github.com
curl: (28) Resolving timed out...

$ npx skills add https://github.com/vercel-labs/skills --skill find-skills
fatal: unable to access 'https://github.com/...': Could not resolve host

$ git clone https://github.com/...
fatal: unable to access 'https://github.com/...': Could not resolve host
```

### Post-Fix Tests (Passed)
```bash
$ curl --max-time 10 https://github.com
Connected to github.com (RESOLVED_IP) port 443
[Success]

$ npx --yes skills add https://github.com/vercel-labs/skills --skill find-skills
✓ Installed 1 skill

$ git clone https://github.com/user/repo.git
Cloning into 'repo'...
[Success]
```

---

## Permanent Fix vs Temporary Workaround

### Temporary Workaround (No Fix Applied)
If you need to bypass DNS issues without modifying system config:

```bash
# For single command
curl --noproxy '*' https://github.com

# For entire session
export no_proxy='*'
npx skills add https://github.com/...

# For permanent npm/git bypass
git config --global url."https://github.com".insteadOf "git@github.com:"
```

### Permanent Fix (Recommended)
1. Modify `/etc/netplan/50-cloud-init.yaml` to use public DNS
2. Create `/etc/systemd/resolved.conf.d/cloudflare-dns.conf`
3. Run `sudo netplan apply`

---

## Related Infrastructure

### Current Server Setup
| Component | Configuration |
|-----------|---------------|
| **Host** | VPS (Provider) |
| **Reverse Proxy** | Caddy |
| **VPN** | TailScale |
| **Containers** | Docker + Coolify |
| **SSH Access** | Port 22 |
| **Firewall** | UFW + iptables |

### Relevant Files
```
/etc/netplan/50-cloud-init.yaml    # Network config with DNS
/etc/systemd/resolved.conf.d/      # DNS resolver config
/etc/caddy/Caddyfile               # Reverse proxy config
/root/.ssh/                        # SSH keys
~/.agents/skills/                  # Global skills
```

---

## Troubleshooting Commands

```bash
# Check current DNS servers
resolvectl status

# Test specific DNS server
dig @1.1.1.1 github.com
dig @8.8.8.8 github.com

# Test HTTPS connectivity
curl -v --max-time 10 https://github.com

# Check DNS resolution
nslookup github.com
dig github.com

# Verify netplan config
cat /etc/netplan/50-cloud-init.yaml

# Apply netplan changes
sudo netplan apply

# Restart DNS resolver
sudo systemctl restart systemd-resolved
```

---

## Prevention

### For Future VPS Deployments
1. Always test DNS resolution with multiple providers after setup
2. Verify `curl https://github.com` works before installing packages
3. Document custom DNS configurations in infrastructure docs
4. Consider using `1.1.1.1` (Cloudflare) as default over provider DNS

### Monitoring
Add to monitoring checklist:
- [ ] GitHub connectivity
- [ ] npm registry access
- [ ] Docker Hub access
- [ ] DNS resolution for common domains

---

## Credits
- DNS fix inspired by Cloudflare's public DNS service (1.1.1.1)
- Troubleshooting methodology: ICMP vs DNS packet flow analysis

---

*Last updated: April 1, 2026*
