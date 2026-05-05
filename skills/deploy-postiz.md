---
name: deploy-postiz
description: Deployment and configuration of Postiz using Docker Compose and Caddy reverse proxy.
---

# Deploying Postiz

This guide documents the successful deployment of Postiz on a Linux VPS, resolving critical networking and authentication pitfalls.

## 🚀 Deployment Workflow

### 1. Environment Setup
Create a directory (e.g., `/opt/postiz`) and an `.env` file containing:
- `POSTGRES_PASSWORD`
- `REDIS_PASSWORD`
- `JWT_SECRET`

### 2. Docker Compose Configuration
Use a `docker-compose.yml` that explicitly separates the Backend and Temporal ports to avoid "Port Already Allocated" errors.

**Critical Config Points:**
- **Backend Port:** Map a unique host port (e.g., `3001:3000`).
- **Temporal Port:** Only map the necessary Temporal port (e.g., `7233:7233`).
- **DB URL Expansion:** Ensure the `DATABASE_URL` uses the variable for the password to avoid `P1000` auth failures:
  `DATABASE_URL=postgresql://postgres:${POSTGRES_PASSWORD}@postiz-db:5432/postiz-db`

### 3. Caddy Reverse Proxy Setup
Postiz's backend handles requests without the `/api` prefix. You MUST use `handle_path` in Caddy to strip this prefix before forwarding.

**Correct Caddy Block:**
```caddy
your-subdomain.omenabyte.com {
    handle_path /api* {
        reverse_proxy 127.0.0.1:3001
    }
    handle {
        reverse_proxy 127.0.0.1:5000
    }
    tls /etc/caddy/certs/origin.crt /etc/caddy/certs/origin.key
}
```

## ⚠️ Pitfalls & Fixes

### The "Port War" (Temporal vs Backend)
**Issue:** The `temporalio/auto-setup` image may attempt to bind to the same host ports as the Postiz backend if not carefully defined.
**Fix:** Explicitly define unique host ports in `docker-compose.yml` and avoid using `sed` replacements that might accidentally apply the same port to multiple services.

### P1000: Authentication Failed
**Issue:** Prisma fails to connect to PostgreSQL because `${POSTGRES_PASSWORD}` isn't expanded in the `DATABASE_URL` string.
**Fix:** Use the `${VARIABLE}` syntax directly inside the `environment:` section of the `postiz` service in `docker-compose.yml`.

### 404 Not Found on `/api/auth/register`
**Issue:** The backend receives `/api/auth/register` but expects `/auth/register`.
**Fix:** Use `handle_path /api*` in Caddy instead of a simple `reverse_proxy /api*`.

## ✅ Verification
1. Run `docker ps` to ensure all containers (postiz, postiz-db, postiz-redis, temporal, temporal-postgresql, temporal-elasticsearch) are `Up`.
2. Check logs: `docker logs postiz` should show the frontend and backend as `online` via PM2.
3. Test registration via the browser or `curl -X POST http://localhost:3001/auth/register`.
