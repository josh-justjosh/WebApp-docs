# WebApp deploy status

**Deploying updates:** see [deployment.md](deployment.md) for the full guide to deploying changes to Production and Beta Docker stacks.

## Current status

| Item | Status |
|------|--------|
| Shared DB stack (WebAppDb: MariaDB + phpMyAdmin) | ✅ Configured |
| Production stack (app, nginx) | ✅ Configured |
| Beta stack (app, nginx) | ✅ Configured |
| Network Rail data stack (`network-rail-data`: feed-worker, nr-maintenance on `webapp-shared`) | Per-host; see [network-rail-data/README.md](../../network-rail-data/README.md) |
| Composer / Laravel (migrate, storage:link) | Per-environment |
| Vite frontend build (`public/build/`) | Per-environment |
| App Production on port 8008 | Expect HTTP 200 |
| App Beta on port 8009 | Expect HTTP 200 |
| Reverse proxy (Caddy) | Caddy on host; Caddyfile in this repo at [caddy/Caddyfile](caddy/Caddyfile), copy to your Caddy config on the host |
| SSL (Let's Encrypt via Caddy) | Automatic for all configured hostnames |

## Access and URLs

| Environment | Directory | Branch | URL |
|-------------|-----------|--------|-----|
| Production | `production/` | main | https://app.josh.me.uk, https://app.jb-vpn.uk |
| Beta | `beta/` | develop (or beta branch) | https://app-beta.josh.me.uk |
| Database admin | `db/` | — | https://app-db.josh.me.uk (phpMyAdmin) |

**Database names:** Production uses `laravel` when reusing the existing volume, or `laravel_production` on a fresh WebAppDb; Beta uses `laravel_beta`.

**Verify branches:** From `production/` run `git branch --show-current` (expect `main`). From `beta/` run `git branch --show-current` (expect `develop` or your beta branch).

**Local checks:**
- Production: `curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:8008/` → 200
- Beta: `curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:8009/` → 200

**Feature URLs (Beta example):** Staff bus departures `/bus-departures`; public wall board `/d/bus-departures` (requires `BUSTIMES_PUBLIC_TOKEN` in `.env`). See [bus-departures.md](bus-departures.md).

## Useful commands

```bash
# Startup order
cd db && docker compose up -d
cd production && docker compose up -d
cd beta && docker compose up -d

# Production
cd production
docker compose ps
docker compose logs -f app
docker compose exec app php artisan migrate

# Beta
cd beta
docker compose ps
docker compose exec app php artisan migrate

# Rebuild frontend (from app directory; requires PHP for Wayfinder)
docker compose exec -u root app sh -c "npm ci && npm run build && chown -R www:www /var/www/node_modules /var/www/public/build"
```
