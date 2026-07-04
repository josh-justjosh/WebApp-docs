# AgentContext – Environment and conventions for AI agents

This document summarizes the project stack, runtime environment, and learned patterns so future agents can work effectively without re-discovering them. For deployment runbooks and troubleshooting, see [README.md](README.md) and the linked docs.

---

## 1. Stack overview

- **Backend:** Laravel 13, **Fortify** (session auth + 2FA), **Inertia** + Vue for staff UI. The app image uses **PHP 8.4** (`php:8.4-fpm-bookworm` in [docker/Dockerfile](../docker/Dockerfile)).
- **PHP vs Composer on the host:** [`composer.json`](../composer.json) and [`composer.lock`](../composer.lock) declare **`php: ^8.2`** (Composer platform `^8.2`), so a host PHP 8.2+ can satisfy Composer. **Runtime in Docker is still 8.4**; prefer `docker compose exec app …` for Artisan and Composer so behaviour matches the image. If you run PHP on the VPS host, use **8.4** to match the container (e.g. [Sury PHP packages](https://packages.sury.org/php/) on Debian), not only “whatever satisfies ^8.2”.
- **Frontend:** Vue 3 + Inertia (staff app via `resources/js/app.ts`), Vite; legacy standalone scoreboard pages use `resources/js/scoreboard.ts`; the public bus departures wall board uses `resources/js/bus-departures-board.ts`. Built assets live in `public/build`.
- **Runtime:** The app runs in Docker: PHP-FPM in container `webapp-beta-php` (see [docker-compose.yml](../docker-compose.yml); production may use different `container_name` values), project root mounted at `/var/www`. Nginx serves the app. Network Rail TD/TRUST ingest runs in the separate [`network-rail-data`](../../network-rail-data) stack (`feed-worker`, `nr-maintenance` on `webapp-shared`), not inside the beta app compose. That sibling path assumes the usual VPS layout (`app.jb/beta` next to `app.jb/network-rail-data`; the stack is not inside the WebApp git tree). It writes to MySQL schema `` `jb.app_network_rail` ``; Laravel reads/writes NR data via connection `mysql_network_rail` (`DB_NETWORK_RAIL_*` in `.env`).
- **Docs:** [docs/README.md](README.md) indexes deployment, troubleshooting, and [td-trust.md](td-trust.md) (Network Rail STOMP feed / `network-rail-data`). This file is the agent-facing context.

---

## 2. Docker and paths

- **Compose:** [docker-compose.yml](../docker-compose.yml) – app service builds from [docker/Dockerfile](../docker/Dockerfile), bind-mounts `.` → `/var/www`, and uses named volumes for `storage`, `bootstrap/cache`, shared storage, and **`webapp-*-puppeteer-cache`** → `/opt/puppeteer-cache` (Beta: `webapp-beta-puppeteer-cache`; Production: `webapp-puppeteer-cache`). `PUPPETEER_CACHE_DIR=/opt/puppeteer-cache` is set in compose.
- **Image build (what the Dockerfile does *not* do):** The Dockerfile does **not** run `composer install`, `npm ci`, `npm run build`, or `npx puppeteer browsers install chrome`. Older images baked those in at build time (~3.5 GB, multi-minute `--no-cache` rebuilds that could hang a ~4 GB RAM VPS with no swap). The slim image (~2 GB) only installs PHP extensions, Node 20, Composer, and Chromium **system libraries**; app code is copied once (`COPY --chown=www:www`). `vendor/`, `node_modules/`, and `public/build/` are excluded via [`.dockerignore`](../.dockerignore) and come from the bind mount at runtime. Run Composer and the frontend build at **deploy** time (see [deployment.md](deployment.md)).
- **Rebuild images:** Prefer `docker compose build app` (uses layer cache). Use `--no-cache` only when debugging a broken layer — it recompiles PHP extensions and reinstalls the full GTK/Chromium apt stack and can make the host unresponsive on low-memory VPSes.
- **Frontend build (Vite):** Always build **inside the app container** — **not** `docker run node:20-bookworm-slim`. [`@laravel/vite-plugin-wayfinder`](../vite.config.ts) runs `php artisan wayfinder:generate --with-form` during `vite build`; a Node-only container fails with `php: not found`.
  ```bash
  docker compose exec -u root app sh -c "npm ci && npm run build && chown -R www:www /var/www/node_modules /var/www/public/build"
  ```
- **Invoice PDFs (Browsershot):** Generated with [Spatie Browsershot](https://github.com/spatie/browsershot) (Puppeteer/headless Chrome) for full CSS — see [invoicing-quotes.md](invoicing-quotes.md#pdf-generation). [InvoicePdfService.php](../app/Services/InvoicePdfService.php) renders [pdf.blade.php](../resources/views/invoices/pdf.blade.php) → HTML → PDF. **Why Chrome?** Browsershot drives a real headless Chrome binary via Puppeteer; PHP-only PDF libraries cannot reproduce the Blade/CSS layout (fonts, backgrounds, page footers). Runtime needs: Node in the app image, `puppeteer` in `package.json`, Chromium libs in the Dockerfile, and a **Chrome binary** in the puppeteer cache volume (installed once per environment, not baked into the image):
  ```bash
  docker compose exec -u root app sh -c "npx puppeteer browsers install chrome && chown -R www:www /opt/puppeteer-cache"
  ```
  Re-run after upgrading the `puppeteer` npm package or recreating the puppeteer volume. **Page numbers** (`Page X of Y`) use Browsershot’s Chrome footer template (`showBrowserHeaderAndFooter()`), not inline body HTML.
- **Adding PHP packages / running Composer:** Composer is often **not installed on the host** (only inside the app container). Running `composer update` inside the app container (`docker compose exec app composer ...`) can also hit **permission denied** when writing `composer.lock` (container user `www` vs mount). Use a one-off Composer image with the project mounted so the lock and vendor are updated on the host:
  ```bash
  docker run --rm -v "$(pwd):/app" -w /app composer:latest composer update <package> --no-interaction
  ```
  For dependency resolution (e.g. new package requires different transitive versions), add `--with-all-dependencies` (-W). Then rebuild or restart the app so it sees the new vendor (or rely on the bind mount if already running).
- **Logging from PHP:** Prefer `storage_path('logs/...')` for any debug or custom log files so writes go to the writable storage volume in the container. Paths under the project root that are not in the storage volume may not be writable by the app user (`www`) or may not persist as expected.
- **Debugging (agents):** When debugging server-side behavior (e.g. 500s), use runtime evidence: add minimal instrumentation (e.g. writing NDJSON or plain lines to `storage_path('logs/...')`) at entry, before/after suspected code, and on error paths. In Docker, only paths under storage (or other writable volumes) are reliable for app-written logs; clear the log file between runs for a clean trace. Prefer fixing with evidence from logs (which code path ran, what failed) rather than code-only guesses. After a fix, verify with a fresh run and log comparison before removing instrumentation.
- **When instrumentation never writes:** If debug logs never appear, the exception may occur before the instrumented code runs (e.g. during dependency injection, middleware, or service-provider registration). To get the real exception when `APP_DEBUG` is false: run the failing code path inside the app container via a one-off PHP script (e.g. `docker compose exec app php -r "require '...'; \$app->bootstrap(); ..."`) so the exception message and class are printed to stdout.
- **Laravel container “Target class [x] does not exist”:** This usually means a binding (e.g. from a package service provider) was never registered—common when the `bootstrap/cache` volume is stale or the provider list differs from the running app. Methods to identify: run the same code path in the container (see above) to confirm the exception. A defensive fix is to ensure the binding exists before use (e.g. if the app does not have the binding, register the package’s service provider on demand), so the feature works even when the provider was not loaded at boot.

---

## 3. Database

- **Default:** Connection `mysql` uses `DB_*` (per-environment schema such as `laravel_beta` / `laravel_production` per `.env`). This is also where Laravel stores the **`migrations` table** for `php artisan migrate`.
- **Shared DB:** If `DB_SHARED_DATABASE` is set, connection `mysql_shared` points at that schema (see [config/database.php](../config/database.php)). Tables such as `users`, `clients`, `invoices`, `invoice_lines`, **`invoice_sections`**, **`quote_groups`**, `client_contacts`, `signalling_diagrams`, `admin_audit_logs`, **`rtt_access_tokens`**, and **`rtt_ref_*`** live there when models use `getConnectionName()` returning `mysql_shared` or the [UsesSharedDatabaseWhenConfigured](../app/Models/Concerns/UsesSharedDatabaseWhenConfigured.php) trait ([app/Models/Invoice.php](../app/Models/Invoice.php), [app/Models/RttAccessToken.php](../app/Models/RttAccessToken.php), [app/Models/RttRef/](../app/Models/RttRef/), etc.). Code that writes audit rows or spans connections must pick the connection explicitly, e.g. `env('DB_SHARED_DATABASE') ? 'mysql_shared' : config('database.default')` (see [UserManagementController](../app/Http/Controllers/UserManagementController.php)).
- **Migrations vs live DB:** `migrate` replays any migration missing from **`migrations` on the default DB**—not from `mysql_shared`. If shared tables already exist but those rows are missing (restore, clone, hand-edited DB), plain `migrate` can try to recreate tables or columns and fail. Many shared migrations use `Schema::hasTable` / `hasColumn` guards so a replay no-ops safely; Network Rail migrations are different (always use `--database=mysql_network_rail` **and** `--path` per `*network_rail*.php` file). Moving diagrams to shared: [`2026_04_12_100000_move_signalling_diagrams_to_mysql_shared.php`](../database/migrations/2026_04_12_100000_move_signalling_diagrams_to_mysql_shared.php) runs on the default migrate batch (not the NR `--path` loop); it copies or drops the per-app `signalling_diagrams` copy as needed.
- **Network Rail (dedicated schema):** Connection `mysql_network_rail` points at database `` `jb.app_network_rail` `` (see `DB_NETWORK_RAIL_*` in `.env`). Models live under [app/Models/NetworkRail/](../app/Models/NetworkRail/) with `$connection = 'mysql_network_rail'`; tables use **short names** (`feed_messages`, `berths`, `berth_meanings`, `area_path_configs`, `signal_states`, etc.). STOMP ingest and retention/SMART jobs run only in [`network-rail-data`](../../network-rail-data); Laravel exposes read APIs and CRUD for berth meanings / area path configs—**no** HTTP feed ingest in PHP. NR migrations: run with `--database=mysql_network_rail` **and** `--path` per file matching `database/migrations/*network_rail*.php` (see [network-rail-data/README.md](../../network-rail-data/README.md)); do not run bare `migrate` on that connection without `--path` or Laravel may run unrelated migrations.

---

## 4. Auth and HTTP routes

- **Staff (session):** Fortify login/logout/2FA; authenticated pages under [routes/staff.php](../routes/staff.php) and [routes/settings.php](../routes/settings.php) with Inertia controllers in `app/Http/Controllers/Web/`. **Realtime Trains API playground** at `/rtt` ([RttController](../app/Http/Controllers/Web/RttController.php)) — see [rtt-api.md](rtt-api.md).
- **Public OpenTrack scoreboard:** Token-gated JSON at `/public/opentrack/{OPENTRACK_PUBLIC_TOKEN}/…` ([PublicOpenTrackController](../app/Http/Controllers/Web/PublicOpenTrackController.php)); standalone scoreboard blades read the base URL from `<meta name="opentrack-public-api-base">`.
- **Bus departures:** Staff UI at `/bus-departures`; public wall board at `/d/bus-departures` with token-gated poll at `/public/bus-departures/{BUSTIMES_PUBLIC_TOKEN}/poll`. See [bus-departures.md](bus-departures.md).
- **Projects (invoices + quotes):** Unified index at `/invoices` (UI title **Projects**); quotes under `/quotes/*`. See [invoicing-quotes.md](invoicing-quotes.md).
- **API:** [routes/api.php](../routes/api.php) exposes only `GET /api/version` for health checks. Legacy `API/V1` controllers were removed; staff features use Inertia web routes.
- **Exception handling:** [bootstrap/app.php](../bootstrap/app.php) – for `api/*` requests, exceptions render as JSON 500. If `app.debug` is true, the body includes message, error class, file, and line. If `app.debug` is false, `GET api/invoices/*/pdf/preview` still returns message, class, file, and line (to debug PDF rendering); other routes return a generic `"Server Error"` unless the exception is a `QueryException` / `PDOException` (then the DB message is included).

---

## 5. Quotes and invoicing (learned behavior)

Full reference: [invoicing-quotes.md](invoicing-quotes.md).

- **Single model:** Quotes and invoices are both `Invoice` rows (`document_type`). Versioned quotes use `quote_groups` + `version` / `is_current_version`. Sections live in `invoice_sections`; lines in `invoice_lines` with optional `notes`, per-line and per-section dates, and PDF show flags.
- **Editor:** [SectionedLineEditor.vue](../resources/js/components/invoices/SectionedLineEditor.vue) is shared by invoice and quote forms. Line notes use a sticky-note modal (not an inline table field). Section date defaults flow to new lines with line `show_date` off when the section has a date.
- **PDF labels:** Supply date header is **Project Date**; section subtotals use **`{section title} total`**. Line notes render below the description.
- **Deploy:** `php artisan migrate` and `npm run build` are **independent** — rebuilding assets does not apply migrations (e.g. `invoice_sections.date`, `invoice_lines.notes`).
- **Restore gotcha:** After partial data loss, **PHP controllers/services may exist while `routes/staff.php` lost quote routes** — verify with `php artisan route:list --name=quotes`. Also restore `resources/js/types/domain/index.ts` (`QuoteRow`, `InvoiceSection`, line `notes`) if TypeScript types were reverted.

---

## 6. Invoice PDF import (learned behavior)

- **Flow:** The user selects a PDF and a client in the UI; the frontend POSTs to `POST /invoices/import (Inertia web route `invoices.import`)` with only `file` and `client_id`. The backend runs [InvoicePdfExtractionService](../app/Services/InvoicePdfExtractionService.php) to extract number, dates, total, subtotal, currency, line items, and notes; then creates the invoice and lines and stores the PDF under `invoices/{id}.pdf` (Storage).
- **Parser:** Uses `smalot/pdfparser` (`Smalot\PdfParser\Parser`). The extraction service is **not** injected in the controller constructor; it is resolved lazily via `app(InvoicePdfExtractionService::class)` only inside `extract()` and `import()`. That way `GET /api/invoices` (and other routes that do not need PDF parsing) never load the Parser class, avoiding 500s when the package is missing or autoload has not been run yet.
- **Supported PDF formats:** App-generated (from [resources/views/invoices/pdf.blade.php](../resources/views/invoices/pdf.blade.php)) and external/CB-0004-style (Invoice number, Date of issue, Due by, Qty/Description/Unit price/Total price, Total £). Extracted data is merged from both parsers where possible.

---

## 7. Frontend (relevant to agents)

- **Entry:** Inertia app bootstrapped from [resources/js/app.ts](../resources/js/app.ts); pages under `resources/js/pages/`. Built with **Vite** into `public/build`.
- **Wayfinder:** [vite.config.ts](../vite.config.ts) uses `@laravel/vite-plugin-wayfinder` (`formVariants: true`). `npm run build` invokes `php artisan wayfinder:generate` — build in the **app container** (see §2), not a standalone Node image. Generated route/action TS lives under `resources/js/routes/` and `resources/js/actions/`.
- **Tables:** `@tanstack/vue-table` is retained as a headless wrapper composing shadcn `Table` primitives — see [decisions/tanstack-vue-table.md](decisions/tanstack-vue-table.md).
- **Invoice import UI:** [resources/js/components/invoices/InvoiceImportModal.vue](../resources/js/components/invoices/InvoiceImportModal.vue) – only a PDF file input and client selector; no manual fields (all from PDF). The Projects index page uses it and calls `POST /invoices/import` with FormData (file + client_id).
- **Projects table:** [resources/js/pages/invoices/Index.vue](../resources/js/pages/invoices/Index.vue) uses [DataTable.vue](../resources/js/components/DataTable.vue) with `getRowHref` for row navigation and server pagination via `:paginator` + `pagination-only`.

---

## 8. Conventions and gotchas

- **New staff features — wire everything:** Adding a page is not done when the controller and Vue file exist. Also update **`routes/staff.php`**, **`config/services.php`** (if env-backed), **`AppSidebar.vue`** (or relevant nav), **`.env.example`**, and **`resources/js/types/domain/index.ts`** when API shapes change. After deploy or restore, run `php artisan route:list --path=…` to confirm routes. RTT lost wiring (routes/config/sidebar) while service files remained — see [rtt-api.md](rtt-api.md) checklist. Bus departures uses the same pattern — see [bus-departures.md](bus-departures.md#adding-or-changing-bus-departures--integration-checklist). **Quotes** lost all `/quotes/*` routes while controllers remained — see [invoicing-quotes.md](invoicing-quotes.md#routes-routesstaffphp).
- **Shared DB data vs code:** Migrations can be applied while **row data** in `mysql_shared` is empty or truncated (restore, failed sync, test pollution). Re-seed domain-specific data via the feature’s sync command (e.g. `php artisan rtt:sync-reference-data --all` for RTT reference tables). **Bus departures PTP display rules** live in `bus_service_display_rules` on the default DB — re-enter via the service modal after a restore that omitted that table.
- **Git branches:** App code: develop on **`beta`**, merge **`beta` → `main`** for production. **`docs/`** is a **submodule** ([WebApp-docs](https://github.com/josh-justjosh/WebApp-docs)); commit and push documentation changes from the `docs` checkout, then bump the submodule pointer on `beta`/`main` in the WebApp repo. Relative links in this file (e.g. `../docker/Dockerfile`) assume this markdown lives at **`WebApp/docs/…`** inside the WebApp working tree; they do not resolve when browsing the docs repo alone on GitHub.
- **Rule/validation with shared DB:** When using `Rule::exists()` for tables that may be on the shared connection, pass the **model class** (e.g. `Rule::exists(Client::class, 'id')`) so Laravel’s rule uses the model’s connection. Do not chain `->connection(...)` on `Rule::exists()` — that method does not exist on the Exists rule.
- **Storage:** The default disk is used for invoice PDFs and similar; the named volume `webapp-beta-storage` is mounted at `/var/www/storage` so persisted files survive container recreation. Shared storage is mounted at `storage/app/public/shared` for cross-environment files. Headless Chrome for Browsershot lives in the **`webapp-*-puppeteer-cache`** volume (not under the bind mount).
- **Existing docs:** For deployment, Caddy, rollback, and runbooks, see [deployment.md](deployment.md) and [deploy-status.md](deploy-status.md). For Composer, PHP, and tests, see [troubleshooting.md](troubleshooting.md).

---

## 9. File locations quick reference

| Area | Path |
|------|------|
| Web controllers | `app/Http/Controllers/Web/` (staff pages), `app/Http/Controllers/` (domain CRUD) |
| Models | `app/Models/` (Invoice, InvoiceLine, InvoiceSection, QuoteGroup, Client, …); Network Rail: `app/Models/NetworkRail/`; RTT ref: `app/Models/RttRef/` |
| Services | `app/Services/` (InvoiceService, QuoteService, InvoicePdfService, InvoicePdfExtractionService, RttService, BustimesService, …) |
| Projects / quotes UI | `/invoices`, `/quotes/*` — [invoicing-quotes.md](invoicing-quotes.md); `resources/js/pages/invoices/`, `resources/js/pages/quotes/`, `SectionedLineEditor.vue` |
| RTT API playground | `/rtt` — [rtt-api.md](rtt-api.md); `resources/js/pages/rtt/`, `resources/js/lib/rttApi.ts` |
| Bus departures | `/bus-departures`, `/d/bus-departures` — [bus-departures.md](bus-departures.md) |
| Migrations | `database/migrations/` (connection chosen via `env('DB_SHARED_DATABASE')` in many) |
| Vue components | `resources/js/components/` |
| Domain TS types | `resources/js/types/domain/index.ts` |
| Invoice PDF view | `resources/views/invoices/pdf.blade.php` |
| Puppeteer Chrome cache | `/opt/puppeteer-cache` (named volume; env `PUPPETEER_CACHE_DIR`) |
| Config | `config/` |
| Env | `.env` (gitignored), `.env.example` for reference |
