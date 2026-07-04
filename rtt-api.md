# Realtime Trains (RTT) API playground

Staff-only tool at **`/rtt`** (sidebar: **Rail → RTT API**) for exploring the [Realtime Trains API](https://realtimetrains.github.io/api-specification/) through a server-side proxy. The refresh token never reaches the browser.

## Configuration

Set in `.env` (see `.env.example`):

| Variable | Purpose |
|----------|---------|
| `RTT_API_KEY` | Long-lived refresh token from [api-portal.rtt.io](https://api-portal.rtt.io) |
| `RTT_BASE_URL` | API base (default `https://data.rtt.io`) |
| `RTT_CREDENTIAL_KEY` | Key for stored access-token row when multiple credentials share one DB (default `default`) |
| `RTT_REFERENCE_DATA_GITHUB_URL` | Base URL for NR reference CSVs (default: `realtimetrains/nr-reference-data` on GitHub) |

Laravel reads these via `config/services.php` → `services.rtt.*`.

## Architecture

| Layer | Path |
|-------|------|
| Controller | `app/Http/Controllers/Web/RttController.php` |
| API proxy + token refresh | `app/Services/RttService.php` |
| Reference data sync/browse | `app/Services/RttReferenceDataService.php` |
| Access token model | `app/Models/RttAccessToken.php` |
| Reference models | `app/Models/RttRef/*` |
| Frontend page | `resources/js/pages/rtt/Index.vue` |
| Endpoint definitions | `resources/js/lib/rttApi.ts` |
| Browse modal | `resources/js/components/rtt/RttReferenceDataBrowseDialog.vue` |
| Routes | `routes/staff.php` (4 routes under `/rtt`) |
| Sidebar | `resources/js/components/AppSidebar.vue` (`railNavItems`) |
| Artisan sync | `php artisan rtt:sync-reference-data` |

### Routes

- `GET /rtt` — Inertia playground
- `POST /rtt/query` — proxied GET to whitelisted RTT paths
- `POST /rtt/reference-data/sync` — sync GitHub CSVs and/or API location/stop lists
- `GET /rtt/reference-data/{dataset}` — paginated browse (modal)

### Token flow

1. `RTT_API_KEY` in `.env` is the **refresh token**.
2. `RttService` exchanges it for a short-lived access token via `/api/get_access_token`.
3. Access tokens are stored in **`rtt_access_tokens`** on the shared DB (when `DB_SHARED_DATABASE` is set).
4. Subsequent API calls use the stored token; refresh is automatic (with cache lock) before expiry.

### Whitelisted API paths

Defined in `RttService::ALLOWED_PATHS`: `/api/info`, `/api/get_access_token`, `/rtt/location`, `/rtt/service`, `/gb-nr/location`, `/gb-nr/service`, allocation endpoints, `/data/locations_ungrouped`, `/data/stops`.

## Shared database tables

When `DB_SHARED_DATABASE` is set, migrations create these on **`mysql_shared`**:

| Table | Purpose |
|-------|---------|
| `rtt_access_tokens` | Cached short-lived access tokens |
| `rtt_ref_operators`, `rtt_ref_attribution`, `rtt_ref_categories`, `rtt_ref_areas` | GitHub CSV reference data |
| `rtt_ref_vehicle_branding`, `rtt_ref_vehicle_mapping`, `rtt_ref_activities` | GitHub CSV reference data |
| `rtt_ref_locations`, `rtt_ref_stops` | RTT API reference lists (Version header `2026-04-09`) |
| `rtt_ref_sync_logs` | Last sync time and row counts per dataset |

Models use `UsesSharedDatabaseWhenConfigured` (via `RttRef\Model` and `RttAccessToken`).

Run migrations on each environment after deploy:

```bash
docker compose exec app php artisan migrate
```

## Reference data sync

**UI:** Reference data card on `/rtt` — Sync GitHub / Sync API / Sync all. Click a row to open the browse modal.

**CLI:**

```bash
docker compose exec app php artisan rtt:sync-reference-data --all
docker compose exec app php artisan rtt:sync-reference-data --github
docker compose exec app php artisan rtt:sync-reference-data --api
```

Expected row counts after a full sync (approximate): ~12k locations, ~2.6k stops, ~45 operators, plus smaller GitHub datasets.

GitHub sync strips UTF-8 BOM from CSVs and deduplicates rows before insert (e.g. duplicate delay codes in attribution).

## Train formation / diagram data

The website train diagram on service pages (e.g. `#allox_id=0`) is **not** a separate image API. It is built from JSON returned by **`GET /gb-nr/service?detailed=true`**:

- `service.allocationData[]` — formations (`allocationItems`, TOPS numbers, units/carriages)
- `knowYourTrainData` — coach letters, facilities, `graphic` paths for stock artwork

This data is **entitlement-gated** (`allowAllocations`, Know Your Train, etc.). Tokens with empty `entitlements` from `/api/info` return schedule/location data only. Request additional entitlements from RTT if needed.

## Adding or changing RTT features — integration checklist

A partial deploy or restore can leave **implementation files** on disk while **wiring** is missing. After any RTT work, verify **all** of:

1. `config/services.php` — `rtt` block present
2. `routes/staff.php` — `RttController` import and four routes
3. `resources/js/components/AppSidebar.vue` — RTT API nav item
4. `.env.example` — RTT env comments
5. `php artisan route:list --path=rtt` — four routes registered
6. Migrations applied; reference data synced if tables are empty

Symptoms of missing wiring: 404 on `/rtt`, “API key not configured” despite `RTT_API_KEY` in `.env`, no sidebar link.

## Tests

Feature tests: `tests/Feature/Web/RttTest.php`, `tests/Feature/Web/RttReferenceDataTest.php`.

Run inside Docker:

```bash
docker compose exec app php artisan test --filter=Rtt
```

RTT models use `UsesSharedDatabaseWhenConfigured`, which returns the default connection during `APP_ENV=testing` (sqlite in-memory). Shared-DB migrations choose connection via `env('DB_SHARED_DATABASE')` at migration time; feature tests HTTP-fake external APIs. See [troubleshooting.md](troubleshooting.md#realtime-trains-rtt) if tests fail on connection or missing tables.

## External links

- [API specification](https://realtimetrains.github.io/api-specification/)
- [NR reference data (GitHub)](https://github.com/realtimetrains/nr-reference-data)
- [API portal](https://api-portal.rtt.io)
