# Troubleshooting

## `composer install` (Windows / local PHP)

If `composer install` fails with missing extensions or PHP version errors:

1. **Enable required PHP extensions**  
   Open your `php.ini` (run `php --ini` to see its path, e.g. `C:\tools\php85\php.ini`). Ensure these lines are present and uncommented:
   ```ini
   extension=fileinfo
   extension=sodium
   ```
   Restart the terminal and run `composer install` again.

2. **PHP 8.5 and "does not satisfy" version errors**  
   The lock file was generated for PHP 8.0–8.4. If you use PHP 8.5:
   - Prefer switching to **PHP 8.2 or 8.4** for this project (e.g. via your version manager), or
   - Run: `composer install --ignore-platform-reqs`  
   Then still enable `fileinfo` and `sodium` (step 1) so the app works at runtime (uploads and Fortify 2FA need them).

3. **PHP 8.2+ required**  
   This project uses Laravel 12 and requires PHP 8.2 or higher.

## 413 when uploading profile photo (not using Docker)

If you use `php artisan serve` and get "413 Request Entity Too Large", increase PHP limits in your `php.ini`: `upload_max_filesize` and `post_max_size` (e.g. 10M). Run `php --ini` to find the loaded `php.ini`.

## Unit Test

### run PHPUnit

```bash
# run PHPUnit all test cases
vendor/bin/phpunit
# or Feature test only
vendor/bin/phpunit --testsuite Feature
```

### Running tests (Docker)

With the app stack running (`docker compose up -d`), run PHPUnit inside the app container:

```bash
docker compose exec app php vendor/bin/phpunit
```

Use this when PHP is not on your PATH (e.g. on Windows) or to match the container environment.

### Code Coverage Report

```bash
# reports is a directory name
vendor/bin/phpunit --coverage-html reports/
```
A `reports` directory has been created for code coverage report. Open the dashboard.html.

## Realtime Trains (RTT)

See [rtt-api.md](rtt-api.md) for full architecture. Common issues:

### `/rtt` returns 404 or “API key not configured”

Implementation files may exist without wiring. Confirm all of:

- `config/services.php` has a `rtt` block
- `routes/staff.php` registers four `/rtt` routes and imports `RttController`
- `AppSidebar.vue` includes **RTT API** under Rail
- `RTT_API_KEY` is set in `.env` (refresh token from [api-portal.rtt.io](https://api-portal.rtt.io))

```bash
docker compose exec app php artisan route:list --path=rtt
docker compose exec app php artisan tinker --execute="echo config('services.rtt.api_key') ? 'key ok' : 'missing';"
```

### Reference data empty or truncated

Tables live on **`mysql_shared`** when `DB_SHARED_DATABASE` is set. Re-sync:

```bash
docker compose exec app php artisan rtt:sync-reference-data --all
```

Check counts (shared connection):

```bash
docker compose exec app php artisan tinker --execute="
\$c = DB::connection('mysql_shared');
echo 'locations: '.\$c->table('rtt_ref_locations')->count().PHP_EOL;
echo 'stops: '.\$c->table('rtt_ref_stops')->count().PHP_EOL;
"
```

Expect ~12k locations and ~2.6k stops after a successful full sync.

### RTT feature tests fail

Tests use sqlite in-memory (`APP_ENV=testing`); RTT shared-DB migrations target `mysql` or `mysql_shared` depending on `DB_SHARED_DATABASE` at migrate time. Run:

```bash
docker compose exec app php artisan test --filter=Rtt
```

Failures mentioning `jb.app_beta.rtt_ref_*` or duplicate keys on `mysql_shared` usually mean the test DB setup does not match production shared-DB layout — see [rtt-api.md](rtt-api.md#tests). Do not run destructive sync or manual inserts against production shared DB while debugging tests.

## Bus departures (bustimes.org)

See [bus-departures.md](bus-departures.md) for full architecture. Common issues:

### Missing Expected times on the board

bustimes.org omits the **Expected** column when departures HTML is fetched with `date` and `time` query params. Live fetches must use `/stops/{atco}/departures` with no params (`BustimesService::fetchDeparturesHtml`). If Expected was missing after a deploy, clear departures cache:

```bash
docker compose exec app php artisan cache:clear
```

Cache version for departures is **`v3`** (`DEPARTURES_CACHE_VERSION` in `BustimesService`).

### Via / destination not showing on public board

Custom via and destination come from **`bus_service_display_rules`**, matched by service slug and upcoming PTP stop ATCO codes. They are configured in the staff service modal (**Board display rules**), not in board stop settings.

If rules were lost after a DB restore:

```bash
docker compose exec app php artisan tinker --execute="echo App\Models\BusServiceDisplayRule::count();"
```

Re-create rules in the UI when count is zero. Board stop config (`settings` key `bus_departures`) is separate and may still be present.

### Public poll 404 or “Public API is not configured”

Confirm wiring and env:

```bash
docker compose exec app php artisan route:list --name=bus-departures
docker compose exec app php artisan tinker --execute="echo config('services.bustimes.public_token') ?: 'missing';"
```

Public routes register only when `BUSTIMES_PUBLIC_TOKEN` is non-empty. The wall board reads `<meta name="bus-departures-public-api-base">` from `resources/views/layouts/bus-departures-board.blade.php`.

### Staff page or service modal looks stale

Rebuild frontend after Vue/TS changes:

```bash
docker compose exec -u root app sh -c "npm ci && npm run build && chown -R www:www /var/www/public/build"
```

### Tests

```bash
docker compose exec app php artisan test --filter=BusDepartures
docker compose exec app php artisan test --filter=Bustimes
```

## Invoicing and quotes

See [invoicing-quotes.md](invoicing-quotes.md) for architecture. Common issues:

### `/quotes/*` returns 404 but invoice pages work

Quote routes may be missing from `routes/staff.php` after a partial restore while controllers remain. Confirm:

```bash
docker compose exec app php artisan route:list --name=quotes
```

Expect ~13 routes. Restore `QuoteController` import and the quote block in [routes/staff.php](../routes/staff.php) (see invoicing-quotes doc).

### Section date or line notes not saving

Schema migrations may not have run. **`npm run build` does not migrate.**

```bash
docker compose exec app php artisan migrate --force
```

Relevant columns: `invoice_sections.date`, `invoice_sections.show_date`, `invoice_lines.notes`.

### PDF footer split across pages or wrong page numbers

Layout is controlled in [pdf.blade.php](../resources/views/invoices/pdf.blade.php) and [InvoicePdfService.php](../app/Services/InvoicePdfService.php) (Browsershot footer for `Page X of Y`). Regenerate a preview after Blade/service changes; no frontend rebuild required unless Vue pages changed.

### TypeScript errors on quote/invoice pages

Ensure [resources/js/types/domain/index.ts](../resources/js/types/domain/index.ts) includes `InvoiceSection`, `QuoteRow`, and line `notes` / `show_date` fields — this file was reverted during a prior restore.

### Tests

```bash
docker compose exec app php artisan test --filter=QuotesAndSections
docker compose exec app php artisan test --filter=InvoiceLineShowFlags
```

## Docker and frontend build

### `vite build` fails with `php: not found`

You ran the build in a Node-only container. Wayfinder invokes `php artisan wayfinder:generate` during the build. Use the app container:

```bash
docker compose exec -u root app sh -c "npm ci && npm run build && chown -R www:www /var/www/node_modules /var/www/public/build"
```

Do **not** use `docker run … node:20-bookworm-slim` for this project.

### Docker image rebuild hangs or freezes the VPS

The slim [docker/Dockerfile](../docker/Dockerfile) no longer runs `npm ci`, `npm run build`, or `npx puppeteer browsers install chrome` at image build time. If you still see long stalls:

- Prefer `docker compose build app` over `docker compose build --no-cache app`.
- `--no-cache` recompiles PHP extensions and reinstalls ~250 Chromium/GTK apt packages; on a ~4 GB RAM host with no swap this can make SSH feel frozen for several minutes.

### Invoice PDF preview returns null or 500

Browsershot needs headless Chrome in the puppeteer cache volume:

```bash
docker compose exec app test -x /opt/puppeteer-cache/chrome/linux-*/chrome-linux64/chrome && echo ok || echo 'install chrome'
docker compose exec -u root app sh -c "npx puppeteer browsers install chrome && chown -R www:www /opt/puppeteer-cache"
```

Ensure `PUPPETEER_CACHE_DIR=/opt/puppeteer-cache` is set in `docker-compose.yml` and the `webapp-*-puppeteer-cache` volume is mounted. See [deployment.md](deployment.md#6b-browsershot--puppeteer-chrome-one-time-per-environment).
