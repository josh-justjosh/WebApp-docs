# Deploying Updates to WebApp Production and Beta (Docker)

This guide describes how to deploy code changes and updates to the Production and Beta Docker stacks for WebApp (Laravel + Vue, served via Nginx in Docker; public HTTPS via Caddy on the host).

For current status and URLs, see [deploy-status.md](deploy-status.md).

## Overview

Production and Beta each run as separate Docker Compose stacks **without** a local `db` service. The shared MariaDB 11 server and phpMyAdmin run in the **`db/`** stack. Both app stacks connect to it over the Docker network `webapp-shared`. Production uses database **`laravel_production`** (or **`laravel`** when reusing the existing DB volume); Beta uses **`laravel_beta`**.

- **Production** (`production/`): **app** (container `webapp-php`), **nginx** (port 8008), **td-trust-worker**
- **Beta** (`beta/`): **app** (container `webapp-beta-php`), **nginx** (port 8009), **td-trust-worker**
- **Shared DB** (`db/`): **db** (MariaDB 11, container `webapp-db`), **phpMyAdmin** (port 8080)

App code is mounted from the host in each directory, so many updates only need pull + migrations + frontend build, without rebuilding images.

## Prerequisites

- SSH (or direct) access to the server where WebApp runs
- `docker` and `docker compose` available
- **WebAppDb stack running first** so the `webapp-shared` network and `webapp-db` container exist (see [Shared database stack (WebAppDb)](#shared-database-stack-webappdb))
- This repo (or a clone) with **production/** and **beta/** app stacks and **db/** for the shared database
- `.env` configured in each app directory (database credentials, `APP_KEY`, `APP_URL`, etc.)

## Docker Compose and .env

- **`docker-compose.yml`** is **committed** to the repo in each stack directory (`production/`, `beta/`, `db/`). It defines services, container names, ports, and volume names; it does not contain secrets. Do **not** add `docker-compose.yml` to `.gitignore`.
- **App and DB config** (e.g. `APP_ENV`, `APP_URL`, `DB_DATABASE`, `DB_PASSWORD`) belong in **`.env`**. Copy from `.env.example` and set values per environment. Each stack has its own example: **production/.env.example** uses production defaults (`APP_ENV=production`, `APP_URL=https://app.josh.me.uk`, `DB_DATABASE=laravel_production`); **beta/.env.example** uses beta defaults (`APP_ENV=beta`, `APP_URL=https://app-beta.josh.me.uk`, `DB_DATABASE=laravel_beta`). Docker Compose reads `.env` and passes these into the containers; the defaults in `docker-compose.yml` are fallbacks only.
- **`docker-compose.example.yml`** in each stack directory is a reference/template shown on GitHub. Use it to see the canonical layout or to create a local override; the real `docker-compose.yml` is the one in the repo.

## Startup order

Start stacks in this order so the database is available:

1. **DB:** `cd db && docker compose up -d`
2. **Production:** `cd production && docker compose up -d`
3. **Beta:** `cd beta && docker compose up -d`

## Deployment Steps (Production)

### 1. Go to the app directory

```bash
cd production
```

### 2. Ensure branch and pull latest code

Production must run the **main** branch. Then pull:

```bash
git checkout main
git pull
```

Resolve any merge conflicts before continuing.

### 3. Rebuild Docker images (only when needed)

Rebuild **only** if you changed:

- `docker/Dockerfile`
- `docker-compose.yml` (service build context, Dockerfile path, or image name)
- Base PHP/extensions or system packages

```bash
docker compose build --no-cache app
docker compose up -d
```

If you did **not** change the above, skip this step and keep the stack running.

### 4. Install or update PHP dependencies

```bash
docker compose exec -T -u root app composer install --no-interaction
docker compose exec -T -u root app chown -R www:www /var/www/vendor /var/www/bootstrap/cache
```

For major upgrades (e.g. Laravel version bump) you may use `composer update` instead of `composer install` after reviewing dependencies.

### 5. Run database migrations

```bash
docker compose exec -T app php artisan migrate --force
```

Use `--force` in production so the command does not prompt.

### 6. Build frontend assets (Vite)

Required after any change to JS, CSS, or Vite config so `public/build/` is up to date:

```bash
docker run --rm -v "$(pwd):/app" -w /app node:20-bookworm-slim sh -c "npm ci && npm run build"
```

Alternatively, build inside the app container and fix ownership:

```bash
docker compose exec -u root app sh -c "npm ci && npm run build && chown -R www:www /var/www/node_modules /var/www/public/build"
```

### 7. Clear application caches

```bash
docker compose exec -T app php artisan config:clear
docker compose exec -T app php artisan cache:clear
docker compose exec -T app php artisan view:clear
```

Optional after config changes: `php artisan config:cache`.

### 8. Restart app (and optionally Nginx)

If you only changed code or env (no image rebuild):

```bash
docker compose restart app
```

If you changed Nginx config (`docker/nginx/conf.d/app.conf`):

```bash
docker compose restart nginx
```

For a full stack restart:

```bash
docker compose up -d
```

## Deploy Beta

Deploy the Beta stack from **beta/** using the same steps as Production, but against the Beta branch and database:

1. `cd beta`
2. `git checkout develop` (or your beta branch name), then `git pull`
3. Rebuild images only if Dockerfile/compose changed; otherwise skip.
4. `docker compose exec -T -u root app composer install --no-interaction` and `chown -R www:www ...`
5. `docker compose exec -T app php artisan migrate --force` (runs against `laravel_beta` on `webapp-db`)
6. Build frontend: `docker run --rm -v "$(pwd):/app" -w /app node:20-bookworm-slim sh -c "npm ci && npm run build"`
7. Clear caches and restart: `docker compose exec -T app php artisan config:clear && ... cache:clear` then `docker compose restart app`

Verification: `curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:8009/` → expect `200`. Public URL: https://app-beta.josh.me.uk

## Post-deploy verification

- **Local HTTP checks:**  
  - Production: `curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:8008/` → expect `200`  
  - Beta: `curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:8009/` → expect `200`

- **Public URLs (Caddy):**  
  - Production: https://app.josh.me.uk, https://app.jb-vpn.uk  
  - Beta: https://app-beta.josh.me.uk  
  - Database admin (phpMyAdmin): https://app-db.josh.me.uk  

- **Logs:**  
  `docker compose logs -f app` and `docker compose logs -f nginx` (run from `production/` or `beta/` as appropriate)

## Laravel scheduler (cron) for OpenTrack scoreboard refresh

The app schedules `opentrack:refresh-scoreboard` every 2 minutes (see `bootstrap/app.php`). That only runs if **Laravel's scheduler** is invoked every minute. The app container runs only PHP-FPM and has no cron; you must run the scheduler from the **host**.

### One-time setup on the server

1. From the host (where Docker runs), add **two** cron entries so both Production and Beta run the Laravel scheduler every minute:

   **Production:** from the production directory, e.g.  
   `* * * * * cd production && ./scripts/run-schedule.sh >> schedule.log 2>&1`

   **Beta:** from the beta directory, e.g.  
   `* * * * * cd beta && ./scripts/run-schedule-beta.sh >> schedule-beta.log 2>&1`

   Or with explicit `docker compose exec`: `cd production && docker compose exec -T app php artisan schedule:run ...` and for Beta: `cd beta && docker compose exec -T app php artisan schedule:run ...`.

2. Use a log path of your choice or redirect to `/dev/null` if you don't need logs.

### Verify the scheduler is running

- **Refresh log in the UI:** Open **OpenTrack → Refresh log**. If the cron is working, you should see entries with trigger "Scheduler (cron)" every few minutes when the scoreboard has been viewed recently. If you never see "Scheduler (cron)" entries, the host cron is not running or not calling `schedule:run`.
- **Run manually once:**  
  `docker compose exec -T app php artisan schedule:run`  
  Then check the refresh log again for a new "scheduled" entry (may be "skipped" if no one has viewed the scoreboard in the last 10 minutes).

### When the scheduled refresh actually runs

The command performs a real refresh from OpenTrack only when:

1. Someone has loaded the **public scoreboard** in the last **10 minutes** (the scoreboard page pings the API and sets a cache key).
2. A **competition** is set (OpenTrack config: year, country, competition).
3. **Scoring is enabled** for that competition.

Otherwise the scheduler still runs but logs "skipped" (e.g. "No recent scoreboard view" or "Scoring disabled for competition").

**If the refresh log shows "OpenTrack returned 403":** OpenTrack often returns 403 for requests from server/datacenter IPs. The scheduled job runs on your server, so it may see 403 while manual refresh from a browser works. Use manual refresh from the OpenTrack page when needed, or ask OpenTrack whether they can whitelist your server's outbound IP.

## Transferring the berth meanings table to production

The **berth meanings** table (`network_rail_berth_meanings`) holds berth-code tooltips for the Network Rail area. The table is created by migrations; the **data** is not seeded. To copy data from another environment (e.g. local/dev) to production:

### 1. Ensure production has the table

Run migrations on production so the table exists (see [Deployment Steps](#deployment-steps-production) step 5):

```bash
docker compose exec -T app php artisan migrate --force
```

### 2. Export from the source environment

On the machine/environment where the berth meanings data lives (e.g. your local app or staging):

```bash
# Export to a file (recommended)
php artisan berth-meanings:export --output=berth-meanings.json
```

If running in Docker locally:

```bash
docker compose exec -T app php artisan berth-meanings:export --output=/var/www/berth-meanings.json
```

Then copy the file from the container or project dir (e.g. `berth-meanings.json` in the project root).

### 3. Copy the file to the production server

Copy `berth-meanings.json` into the production directory (e.g. project root) using `scp`, rsync, or any other method.

### 4. Import on production

From the production directory:

```bash
cd production
docker compose exec -T app php artisan berth-meanings:import berth-meanings.json
```

If you placed the file elsewhere, use the path as seen from the host (the app container mounts the project as `/var/www`, so a file at `production/berth-meanings.json` is available in the container as `/var/www/berth-meanings.json`). So you can run:

```bash
docker compose exec -T app php artisan berth-meanings:import /var/www/berth-meanings.json
```

Import is idempotent: existing rows are updated by `berth_code`, new rows are inserted. You can re-run the import after adding more meanings in the source environment.

### Alternative: direct database copy

If you have access to both databases, you can dump only the berth meanings table from the source and load it into production. From the **source** host (replace with your source DB credentials and host):

```bash
mysqldump -h SOURCE_HOST -u USER -p DATABASE network_rail_berth_meanings > berth_meanings.sql
```

On the **production** server, import into the shared DB (from the WebAppDb stack; use `laravel_production`, `laravel` (when reusing existing volume), or `laravel_beta` as needed):

```bash
docker compose -f db/docker-compose.yml exec -T db mariadb -u root -p"${DB_PASSWORD}" laravel_production < berth_meanings.sql
```

(Use `laravel` instead of `laravel_production` if Production is using the existing DB volume. Or run from `db/` and use the db container name `webapp-db` if connecting from the host.) This copies IDs and timestamps; the Artisan import uses upsert by `berth_code` and does not preserve IDs.

## When to run one-time setup again

The script `docker-setup.sh` is for **initial** setup (key:generate, migrate, db:seed, passport:install, storage:link). You do **not** need to run it on every deploy. Only re-run (or run its steps manually) if:

- You are deploying to a new environment, or
- You have added a step that is not yet in the normal deploy flow (e.g. a new artisan command that must run once).

## Summary: minimal deploy (code-only change)

For typical code-only updates (no Dockerfile/compose changes):

```bash
cd production
git checkout main && git pull
docker compose exec -T -u root app composer install --no-interaction
docker compose exec -T -u root app chown -R www:www /var/www/vendor /var/www/bootstrap/cache
docker compose exec -T app php artisan migrate --force
docker run --rm -v "$(pwd):/app" -w /app node:20-bookworm-slim sh -c "npm ci && npm run build"
docker compose exec -T app php artisan config:clear && docker compose exec -T app php artisan cache:clear
docker compose restart app
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:8008/
```

## Rollback

Production and Beta can be rolled back **independently**. Database rollbacks (migrations) are per database (`laravel_production` or `laravel` for Production, `laravel_beta` for Beta).

If a deploy causes issues:

1. Revert code in that directory: `git checkout <previous-commit>` (or `git revert` and then pull).
2. Re-run the same steps (composer install, migrate if you need to reverse migrations separately, npm run build, clear caches, restart app).
3. If you had run new migrations, consider `php artisan migrate:rollback` if appropriate, then redeploy the previous code.

Keep a note of the last known-good commit for each environment so you can return to it quickly.

---

## Reverse proxy and SSL (Caddy)

SSL is handled by **Caddy** with **Let's Encrypt** (automatic issuance and renewal). The hostname → port mapping is:

| Hostname | Backend port |
|----------|--------------|
| app.josh.me.uk, app.jb-vpn.uk | 127.0.0.1:8008 (Production) |
| app-beta.josh.me.uk | 127.0.0.1:8009 (Beta) |
| app-db.josh.me.uk | 127.0.0.1:8080 (phpMyAdmin) |

The Caddyfile lives in this docs repo at [caddy/Caddyfile](caddy/Caddyfile). To use it:

1. Install Caddy on the host (package manager or [caddy.com](https://caddy.com)).
2. Copy the Caddyfile from this repo to the host: copy [caddy/Caddyfile](caddy/Caddyfile) to your Caddy config path on the host.
3. Reload Caddy so it listens on 80/443 and proxies the hostnames to the ports above. Caddy will obtain and renew certificates automatically. The Caddyfile also includes blocks for dsm.jb-vpn.uk, plex.jb-vpn.uk, vps.jb-vpn.uk, wiki.jb-vpn.uk, werbs-wiki.jb-vpn.uk, and a default server.

**Securing phpMyAdmin:** Restrict access to https://app-db.josh.me.uk (e.g. Caddy `basicauth`, IP allowlist, or VPN-only). See the commented `basicauth` block in [caddy/Caddyfile](caddy/Caddyfile).

---

## Shared database stack (WebAppDb)

The shared MariaDB 11 and phpMyAdmin run in **db/**. Both Production and Beta connect to the `webapp-db` container over the Docker network `webapp-shared`.

### First-time setup

1. `cd db`
2. Copy `.env.example` to `.env` and set `DB_ROOT_PASSWORD` (and optionally `DB_DATABASE_PRODUCTION`).
3. `docker compose up -d`. The init script creates the `laravel_beta` database automatically on first run; `laravel_production` is created by `MYSQL_DATABASE` (or use an external volume and existing database name `laravel` for Production).
4. **When reusing the existing volume** (`webapp_webapp-mysql-data`): set Production app `DB_DATABASE=laravel`. Create `laravel_beta` manually if the volume already had data before the init script existed:  
   `docker compose exec -T db mariadb -u root -p"$DB_ROOT_PASSWORD" -e "CREATE DATABASE IF NOT EXISTS laravel_beta;"`

### Securing phpMyAdmin

Protect https://app-db.josh.me.uk so it is not open to the whole internet. Use Caddy `basicauth` (see [caddy/Caddyfile](caddy/Caddyfile)), an IP allowlist, or serve it only over a VPN. Do not expose MySQL port 3306 publicly; it is bound to `127.0.0.1` for host-side tools only.
