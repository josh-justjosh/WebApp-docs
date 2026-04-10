# AgentContext – Environment and conventions for AI agents

This document summarizes the project stack, runtime environment, and learned patterns so future agents can work effectively without re-discovering them. For deployment runbooks and troubleshooting, see [README.md](README.md) and the linked docs.

---

## 1. Stack overview

- **Backend:** Laravel 12, **PHP 8.4**, Laravel Passport (API auth). The app image uses `php:8.4-fpm-bookworm` in [docker/Dockerfile](../docker/Dockerfile).
- **PHP version on the host:** `composer.lock` resolves dependencies such that Composer’s platform check expects **PHP ≥ 8.4** (not 8.2). Debian Bookworm’s default packages are 8.2; if you run `php artisan` or Composer on the VPS **outside** Docker, install PHP 8.4 (e.g. [Sury PHP packages](https://packages.sury.org/php/)) or use `docker compose exec app …` so the runtime matches the container.
- **Frontend:** Vue (Laravel UI), Axios, SPA; built assets live in `public/build`.
- **Runtime:** The app runs in Docker: PHP-FPM in container `webapp-beta-php`, project root mounted at `/var/www`. Nginx serves the app. Network Rail TD/TRUST ingest runs in the separate [`network-rail-data`](../../network-rail-data) stack (`feed-worker`, `nr-maintenance` on `webapp-shared`), not inside the beta app compose.
- **Docs:** [docs/README.md](README.md) indexes deployment, troubleshooting, and [td-trust.md](td-trust.md) (Network Rail STOMP feed / `network-rail-data`). This file is the agent-facing context.

---

## 2. Docker and paths

- **Compose:** [docker-compose.yml](../docker-compose.yml) – app service builds from [docker/Dockerfile](../docker/Dockerfile), mounts `.` to `/var/www`, and uses named volumes for `storage`, `bootstrap/cache`, and shared storage.
- **Vendor and build:** The Dockerfile does **not** run `composer install`. It copies the repo (including `vendor/`) and runs `npm ci && npm run build`. So `vendor` is whatever exists on the host at build/copy time.
- **Invoice PDFs:** Generated with [Spatie Browsershot](https://github.com/spatie/browsershot) (Puppeteer/headless Chrome). Requires Node, `puppeteer` in `package.json`, and Chromium system libraries (installed in the Dockerfile). The Blade view `resources/views/invoices/pdf.blade.php` is rendered to HTML and passed to Browsershot for full CSS support.
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

- **Default:** MySQL via `DB_*` env; connection name `mysql` (or `mysql_shared` when a shared DB is used).
- **Shared DB:** If `DB_SHARED_DATABASE` is set (e.g. to a database name), many models and migrations use connection `mysql_shared` (see [config/database.php](../config/database.php)). Tables such as `users`, `clients`, `invoices`, `invoice_lines`, `client_contacts` then live on that shared DB. Check `getConnectionName()` on models ([app/Models/Invoice.php](../app/Models/Invoice.php), [app/Models/Client.php](../app/Models/Client.php), etc.) and use the same connection in controllers and migrations. For example, [app/Http/Controllers/API/V1/InvoiceController.php](../app/Http/Controllers/API/V1/InvoiceController.php) uses `env('DB_SHARED_DATABASE') ? 'mysql_shared' : config('database.default')` for transactions.

---

## 4. API and auth

- **Base:** Routes are under [routes/api.php](../routes/api.php); namespace `App\Http\Controllers\API\V1`. Most read/write routes use `middleware('auth:api')` (Passport).
- **Exception handling:** [bootstrap/app.php](../bootstrap/app.php) – for `api/*` requests, exceptions render as JSON 500; if `app.debug` is true, the body includes message, error class, file, and line; otherwise a generic "Server Error" (or DB exception message when applicable).

---

## 5. Invoice PDF import (learned behavior)

- **Flow:** The user selects a PDF and a client in the UI; the frontend POSTs to `POST /api/invoices/import` with only `file` and `client_id`. The backend runs [InvoicePdfExtractionService](../app/Services/InvoicePdfExtractionService.php) to extract number, dates, total, subtotal, currency, line items, and notes; then creates the invoice and lines and stores the PDF under `invoices/{id}.pdf` (Storage).
- **Parser:** Uses `smalot/pdfparser` (`Smalot\PdfParser\Parser`). The extraction service is **not** injected in the controller constructor; it is resolved lazily via `app(InvoicePdfExtractionService::class)` only inside `extract()` and `import()`. That way `GET /api/invoices` (and other routes that do not need PDF parsing) never load the Parser class, avoiding 500s when the package is missing or autoload has not been run yet.
- **Supported PDF formats:** App-generated (from [resources/views/invoices/pdf.blade.php](../resources/views/invoices/pdf.blade.php)) and external/CB-0004-style (Invoice number, Date of issue, Due by, Qty/Description/Unit price/Total price, Total £). Extracted data is merged from both parsers where possible.

---

## 6. Frontend (relevant to agents)

- **Entry:** Laravel serves the SPA; the Vue app and routes live under `resources/js/` (e.g. [resources/js/App.vue](../resources/js/App.vue), [resources/js/routes.js](../resources/js/routes.js)). Built with Vite (or Mix) into `public/build`.
- **Invoice import UI:** [resources/js/components/InvoiceImportModal.vue](../resources/js/components/InvoiceImportModal.vue) – only a PDF file input and client selector; no manual fields (all from PDF). [resources/js/components/InvoicesList.vue](../resources/js/components/InvoicesList.vue) uses it and calls `POST /api/invoices/import` with FormData (file + client_id).

---

## 7. Conventions and gotchas

- **Rule/validation with shared DB:** When using `Rule::exists()` for tables that may be on the shared connection, pass the **model class** (e.g. `Rule::exists(Client::class, 'id')`) so Laravel’s rule uses the model’s connection. Do not chain `->connection(...)` on `Rule::exists()` — that method does not exist on the Exists rule.
- **Storage:** The default disk is used for invoice PDFs and similar; the named volume `webapp-beta-storage` is mounted at `/var/www/storage` so persisted files survive container recreation. Shared storage is mounted at `storage/app/public/shared` for cross-environment files.
- **Existing docs:** For deployment, Caddy, rollback, and runbooks, see [deployment.md](deployment.md) and [deploy-status.md](deploy-status.md). For Passport keys, Composer, PHP, and tests, see [troubleshooting.md](troubleshooting.md).

---

## 8. File locations quick reference

| Area | Path |
|------|------|
| API controllers | `app/Http/Controllers/API/V1/` |
| Models | `app/Models/` (Invoice, InvoiceLine, Client, ClientContact, User) |
| Services | `app/Services/` (InvoicePdfService, InvoicePdfExtractionService) |
| Migrations | `database/migrations/` (connection chosen via `env('DB_SHARED_DATABASE')` in many) |
| Vue components | `resources/js/components/` |
| Invoice PDF view | `resources/views/invoices/pdf.blade.php` |
| Config | `config/` |
| Env | `.env` (gitignored), `.env.example` for reference |
