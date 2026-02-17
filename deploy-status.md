# WebApp deploy status

**Deploying updates:** see [deployment.md](deployment.md) for the full guide to deploying changes to Production and Beta Docker stacks.

## Current status

| Item | Status |
|------|--------|
| Shared DB stack (WebAppDb: MariaDB + phpMyAdmin) | ✅ Configured |
| Production stack (app, nginx, td-trust-worker) | ✅ Configured |
| Beta stack (app, nginx, td-trust-worker) | ✅ Configured |
| Composer / Laravel (migrate, passport, storage:link) | Per-environment |
| Vite frontend build (`public/build/`) | Per-environment |
| App Production on port 8008 | Expect HTTP 200 |
| App Beta on port 8009 | Expect HTTP 200 |
| Reverse proxy (Caddy) | Caddy on host; Caddyfile in this docs repo at [caddy/Caddyfile](caddy/Caddyfile), copy to /etc/caddy/Caddyfile |
| SSL (Let's Encrypt via Caddy) | Automatic for all configured hostnames |

## Access and URLs

| Environment | Directory | Branch | URL |
|-------------|-----------|--------|-----|
| Production | `/root/WebApp` | main | https://app.josh.me.uk, https://app.jb-vpn.uk |
| Beta | `/root/WebAppBeta` | develop (or beta branch) | https://app-beta.josh.me.uk |
| Database admin | `/root/WebAppDb` | — | https://app-db.josh.me.uk (phpMyAdmin) |

**Database names:** Production uses `laravel` when reusing the existing volume, or `laravel_production` on a fresh WebAppDb; Beta uses `laravel_beta`.

**Local checks:**
- Production: `curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:8008/` → 200
- Beta: `curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:8009/` → 200

## Useful commands

```bash
# Startup order
cd /root/WebAppDb && docker compose up -d
cd /root/WebApp && docker compose up -d
cd /root/WebAppBeta && docker compose up -d

# Production
cd /root/WebApp
docker compose ps
docker compose logs -f app
docker compose exec app php artisan migrate

# Beta
cd /root/WebAppBeta
docker compose ps
docker compose exec app php artisan migrate

# Rebuild frontend (from app directory)
docker run --rm -v "$(pwd):/app" -w /app node:20-bookworm-slim sh -c "npm ci && npm run build"
```
