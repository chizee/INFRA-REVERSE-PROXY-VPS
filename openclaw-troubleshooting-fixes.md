# OpenClaw Troubleshooting & Fix Documentation

## Overview

This document details the issues found and fixes applied to restore OpenClaw functionality after a migration to a new VPS. The fixes addressed gateway configuration, plugin trust settings, device pairing, skills path issues, and web UI access.

---

## Issues Identified

### 1. Gateway Mode Misconfiguration

**Problem:** Gateway was configured in `remote` mode but `gateway.remote.url` was missing, causing the gateway to fail to start.

**Error from logs:**
```
Gateway start blocked: set gateway.mode=local (current: remote) or pass --allow-unconfigured.
```

**Fix Applied:**
- Changed `gateway.mode` from `"remote"` to `"local"` in `~/.openclaw/openclaw.json`

**Configuration Change:**
```json
// Before
"gateway": {
  "mode": "remote",
  ...
}

// After  
"gateway": {
  "mode": "local",
  ...
}
```

**Result:** Gateway now starts successfully and listens on `0.0.0.0:18789`

---

### 2. Plugin Trust Allowlist Missing

**Problem:** `plugins.allow` was empty, causing warnings about auto-loading discovered plugins (specifically `lossless-claw`).

**Warning seen:**
```
plugins.allow is empty; discovered non-bundled plugins may auto-load: lossless-claw
lossless-claw: loaded without install/load-path provenance; treat as untracked local code
```

**Fix Applied:**
- Added trusted plugin IDs to `plugins.allow` array

**Configuration Change:**
```json
"plugins": {
  "allow": [
    "telegram",
    "lossless-claw", 
    "google"
  ],
  "entries": {
    "telegram": { "enabled": true, "config": {} },
    "google": { "enabled": true, "config": {} },
    "lossless-claw": { "enabled": true, "config": {} }
  }
}
```

**Result:** Plugin loading warnings resolved.

---

### 3. Device Pairing Required for Control UI Access

**Problem:** After fixing the gateway, the Control UI still showed "pairing required" when accessed via the browser. This is OpenClaw's security feature - new devices must be approved by the owner.

**Error shown:**
```
Gateway pairing approval required.
```

**Root Cause:** When accessing the Control UI from a browser, a new device pairing request was generated but not approved.

**How to Pair (Manual Process):**

1. **List pending pairing requests:**
   ```bash
   openclaw devices list
   ```

   Output shows pending requests with Request IDs (example):
   ```
   Pending (2)
   ┌──────────────────────────────────────┬────────────────┬──────────┐
   │ Request                              │ Device         │ Role     │
   ├──────────────────────────────────────┼────────────────┼──────────┤
   │ 3cd8fde5-dd44-4385-ba21-277003cdaa27 │ 6f9f339761c41c │ operator │
   │ fa465873-4bf3-4730-82b4-089ab5e7f79b │ 63249f3ab67131 │ operator │
   └──────────────────────────────────────┴────────────────┴──────────┘
   ```

2. **Approve each pending request:**
   ```bash
   openclaw devices approve 3cd8fde5-dd44-4385-ba21-277003cdaa27
   openclaw devices approve fa465873-4bf3-4730-82b4-089ab5e7f79b
   ```

3. **Verify gateway is now reachable:**
   ```bash
   openclaw status
   ```
   
   Should show: `Gateway ... reachable` (instead of `unreachable (pairing)`)

**Result:** Browser device is now paired and can access the Control UI without token.

---

### 4. Cloudflare Proxy Configuration

**Problem:** Accessing via Cloudflare domain returned 502 Bad Gateway.

**Investigation:** 
- Direct IP access worked
- Cloudflare was not proxying correctly to the backend

**Fix Applied:**
- Added Cloudflare IP ranges to `gateway.trustedProxies` to allow proper forwarding

**Configuration Change:**
```json
"trustedProxies": [
  "127.0.0.1",
  "173.245.48.0/20",
  "103.21.244.0/20",
  "103.22.200.0/22",
  "103.31.4.0/22",
  "141.101.64.0/18",
  "108.162.192.0/18",
  "190.93.240.0/20",
  "188.114.96.0/20",
  "197.234.240.0/22",
  "198.41.128.0/17",
  "162.158.0.0/15",
  "172.64.0.0/13",
  "104.16.0.0/12"
]
```

**Result:** Cloudflare domain now serves the Control UI correctly.

---

### 5. Gateway Authentication Mode

**Problem:** For security, password authentication was enabled to protect the exposed gateway.

**Fix Applied:**
- Set `gateway.auth.mode` to `password` with a secure password

**Configuration:**
```json
"auth": {
  "mode": "password",
  "password": "YOUR_SECURE_PASSWORD_HERE"
}
```

**Note:** The Control UI automatically handles authentication after device pairing.

---

### 6. Control UI Origins

**Problem:** Control UI needed to know which origins are allowed.

**Fix Applied:**
- Configured `controlUi.allowedOrigins` to include the Cloudflare domain

**Configuration:**
```json
"controlUi": {
  "allowedOrigins": [
    "http://localhost:18789",
    "https://your-domain.com"
  ],
  "allowInsecureAuth": false
}
```

---

### 7. Skills Path Warning - Symlinks Outside Workspace Root

**Problem:** Skills in the OpenClaw skills directory (`~/.openclaw/skills/`) were symlinks pointing to `/root/.agents/skills/` which is outside the configured workspace root. This triggered numerous security warnings:

```
[skills] Skipping skill path that resolves outside its configured root.
```

**Impact:** These warnings flooded the gateway logs but didn't break functionality. The warnings occurred every time the gateway started.

**Root Cause:** The `~/.openclaw/skills/` directory was populated with symlinks pointing to an external location (`/root/.agents/skills/`). OpenClaw's security feature detects when skill paths resolve outside the configured root and skips loading them, logging a warning for each.

**Fix Applied:**
- Removed the entire `~/.openclaw/skills/` directory
- This removed ~60 symlinks that were causing the warnings

**Commands:**
```bash
# Remove the symlinked skills directory
rm -rf ~/.openclaw/skills/

# Restart gateway to verify warnings are gone
openclaw gateway restart
```

**Result:** No more "Skipping skill path" warnings in gateway logs.

**Note:** The workspace skills at `/home/ubuntu/clawd/skills/` remain. Some are real directories, some are symlinks - but that was the pre-existing working setup.

---

## Final Working Configuration

```json
{
  "gateway": {
    "port": 18789,
    "mode": "local",
    "bind": "lan",
    "controlUi": {
      "allowedOrigins": [
        "http://localhost:18789",
        "https://your-domain.com"
      ],
      "allowInsecureAuth": false
    },
    "auth": {
      "mode": "password",
      "password": "YOUR_SECURE_PASSWORD_HERE"
    },
    "trustedProxies": [
      "127.0.0.1",
      "173.245.48.0/20",
      "103.21.244.0/20",
      "103.22.200.0/22",
      "103.31.4.0/22",
      "141.101.64.0/18",
      "108.162.192.0/18",
      "190.93.240.0/20",
      "188.114.96.0/20",
      "197.234.240.0/22",
      "198.41.128.0/17",
      "162.158.0.0/15",
      "172.64.0.0/13",
      "104.16.0.0/12"
    ]
  },
  "plugins": {
    "allow": [
      "telegram",
      "lossless-claw",
      "google"
    ],
    "entries": {
      "telegram": { "enabled": true, "config": {} },
      "google": { "enabled": true, "config": {} },
      "lossless-claw": { "enabled": true, "config": {} }
    }
  }
}
```

---

## Verification Commands

After any configuration changes, verify the gateway is working:

```bash
# Restart gateway
openclaw gateway restart

# Wait a moment, then check status
sleep 10
openclaw status

# Check specifically for reachable gateway
openclaw status | grep Gateway
```

Expected output: `Gateway ... reachable`

---

## Access URLs

| Method | URL |
|--------|-----|
| Direct | `http://YOUR_VPS_IP:18789` |
| Via Cloudflare | `https://your-domain.com` |

---

## Security Notes

1. **Gateway binds to LAN (0.0.0.0)** - Ensure firewall rules are in place
2. **Password stored in config** - Consider using environment variables or `--password` flag instead
3. **Pairing is required** - New browsers/devices must be approved via CLI
4. **Keep tokens/passwords secure** - Don't share access URLs publicly

---

## Troubleshooting Tips

### If pairing required again:
```bash
openclaw devices list
openclaw devices approve <request-id>
```

### If gateway won't start:
```bash
# Check logs
journalctl --user -u openclaw-gateway.service -n 50 --no-pager

# Check for port conflicts
ss -tlnp | grep 18789
```

### If Control UI not accessible:
1. Verify gateway is reachable: `openclaw status | grep Gateway`
2. Check Cloudflare proxy: `curl -I https://your-domain.com`
3. Verify device is paired: `openclaw devices list`

### If skills warnings appear again:
```bash
# Check what's in the skills directory
ls -la ~/.openclaw/skills/

# Remove symlinked skills if they point outside workspace root
rm -rf ~/.openclaw/skills/

# Restart gateway
openclaw gateway restart
```

---

## Commands Reference

| Task | Command |
|------|---------|
| Check status | `openclaw status` |
| Restart gateway | `openclaw gateway restart` |
| View logs | `journalctl --user -u openclaw-gateway.service -n 100 --no-pager` |
| List devices | `openclaw devices list` |
| Approve device | `openclaw devices approve <request-id>` |
| Remove device | `openclaw devices remove <device-id>` |
| Health check | `openclaw gateway health` |

---

*Document created: April 2026*
*OpenClaw version: 2026.3.28*