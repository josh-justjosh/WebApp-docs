# Realtime Trains (RTT) API playground

Staff-only tool at **`/rtt`** (sidebar: **Rail → RTT → API**) for exploring the [Realtime Trains API](https://realtimetrains.github.io/api-specification/) through a server-side proxy. The refresh token never reaches the browser; access tokens in playground responses are redacted.

## Configuration

Set in `.env` (see `.env.example`):

| Variable | Purpose |
|----------|---------|
| `RTT_API_KEY` | Long-lived refresh token from [api-portal.rtt.io](https://api-portal.rtt.io) |
| `RTT_SECONDARY_API_KEY` | Optional temporary refresh token. **When set, it is used instead of** `RTT_API_KEY` (separate stored access-token row) |
| `RTT_BASE_URL` | API base (default `https://data.rtt.io`) |
| `RTT_CREDENTIAL_KEY` | Key for stored access-token row when multiple credentials share one DB (default `default`) |
| `RTT_REFERENCE_DATA_GITHUB_URL` | Base URL for NR reference CSVs (default: `realtimetrains/nr-reference-data` on GitHub) |
| `RTT_STATIONS_GITHUB_URL` | Station CRS CSV (default: `davwheat/uk-railway-stations` `stations.csv`) |

Laravel reads these via `config/services.php` → `services.rtt.*`.

## Architecture

| Layer | Path |
|-------|------|
| Controller | `app/Http/Controllers/Web/RttController.php` |
| API proxy + token refresh | `app/Services/RttService.php` |
| Rate-limit guard | `app/Services/RttRateLimitGuard.php` |
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
- `POST /rtt/query` — proxied GET to whitelisted RTT paths (tokens redacted)
- `POST /rtt/reference-data/sync` — sync GitHub CSVs (`source=github`), including the station CRS list; clears any persisted API location rows
- `GET /rtt/reference-data/{dataset}` — paginated browse (modal)

### Token flow

1. Active refresh token is `RTT_SECONDARY_API_KEY` if set, otherwise `RTT_API_KEY`.
2. `RttService` exchanges it for a short-lived access token via `/api/get_access_token`.
3. Access tokens are stored in **`rtt_access_tokens`** on the shared DB (when `DB_SHARED_DATABASE` is set). Secondary uses credential key `{RTT_CREDENTIAL_KEY}:secondary`.
4. Subsequent API calls use the stored token; refresh is automatic (with cache lock) before expiry.

### Rate limiting

`RttRateLimitGuard` reads `X-RateLimit-Limit-*` / `X-RateLimit-Remaining-*` (Minute, Hour, Day, Week) on every response and:

- Optimistically decrements remaining before each outbound call
- **Blocks locally** (no HTTP) when remaining would go below zero, with message *Blocked by this server to stay within Realtime Trains rate limits*
- On remote `429`, waits for the full `Retry-After` (no early probe)
- Surfaces status to `/rtt` and display boards (`blockedLocally`, resume time, remaining quotas)

### Local pacing (hourly spread)

RTT Hobbyist-style ceilings are typically **10/min, 100/hour**. Using the full 10/min burns the hour budget in about 10 minutes. With `RTT_PACE_ENABLED` (default on), the guard only paces against the **hour** window (not day/week):

`pacedMinuteCap = min(Minute limit, ceil(Hour remaining / minutes left in this hour))`

Examples: at `:00` with 100 remaining → **2/min**; at `:50` with 80 remaining → **8/min**. Unused hour budget raises the cap so later minutes can catch up, still never above the RTT minute limit.

Essential board calls wait until the next minute when paced out; optional/preferred enrichment is soft-skipped. Access-token refresh is **not** paced — it still counts against RTT remaining, but does not consume or get blocked by the local board slot. Disable with `RTT_PACE_ENABLED=false` if needed.

### Request priority (secondary / tight quotas)

Outbound calls are tagged so essentials keep working when the minute budget is small (e.g. Hobbyist secondary ≈ 10/min):

| Priority | Used for | Behaviour when quota is low |
|----------|----------|-----------------------------|
| **essential** | Station `/gb-nr/location` for display boards; access-token refresh | Uses last remaining slots |
| **preferred** | Calling-point / arrival enrichment | Soft-skipped once Minute remaining ≤ 1 |
| **optional** | Formation (`detailed` service), change-expansion hubs, `/rtt` playground | Soft-skipped once Minute remaining ≤ 3 |

Soft skips do **not** set a global block — later essential location fetches can still proceed. Enrichment lookup caps also shrink from remaining quota (`enrichmentAllowance`). Within a departures board: location → arrivals (`filterTo`) → formation.

### Whitelisted API paths

`/api/info`, `/api/get_access_token`, `/rtt/location`, `/rtt/service`, `/gb-nr/location`, `/gb-nr/service`, allocation endpoints.

Bulk catalogue paths (`/data/locations_ungrouped`, `/data/stops`) are **not** allowed — Hobbyist terms prohibit bulk extraction and durable storage of API data.

## Shared database tables

When `DB_SHARED_DATABASE` is set, migrations create these on **`mysql_shared`**:

| Table | Purpose |
|-------|---------|
| `rtt_access_tokens` | Cached short-lived access tokens |
| `rtt_ref_operators`, `rtt_ref_attribution`, `rtt_ref_categories`, `rtt_ref_areas` | GitHub CSV reference data (CC-BY 4.0) |
| `rtt_ref_vehicle_branding`, `rtt_ref_vehicle_mapping`, `rtt_ref_activities` | GitHub CSV reference data |
| `rtt_ref_stops` | Station CRS codes from GitHub (`davwheat/uk-railway-stations`) |
| `rtt_ref_locations` | Legacy table; kept empty (API catalogue sync removed) |
| `rtt_ref_sync_logs` | Last sync time and row counts per dataset |

Models use `UsesSharedDatabaseWhenConfigured` (via `RttRef\Model` and `RttAccessToken`).

## Reference data sync

**UI:** Reference data card on `/rtt` — Sync GitHub only.

**CLI:**

```bash
docker compose exec app php artisan rtt:sync-reference-data
docker compose exec app php artisan rtt:sync-reference-data --clear-api
```

GitHub sync loads station CRS codes into `rtt_ref_stops` and clears any rows in `rtt_ref_locations`. CRS lookups (`GET /rtt/stops/lookup`) use that table, and fetch the GitHub stations CSV on demand if it is empty.

## Train formation / diagram data

Display scenes request `/gb-nr/location` and `/gb-nr/service?detailed=true` and surface branding / length on rail boards. This data is entitlement-gated. Hobbyist plans may include detailed mode and passenger allocations.

## Adding or changing RTT features — integration checklist

1. `config/services.php` — `rtt` block present (including `secondary_api_key`)
2. `routes/staff.php` — `RttController` import and four routes
3. `resources/js/components/AppSidebar.vue` — RTT API nav item
4. `.env.example` — RTT env comments
5. `php artisan route:list --path=rtt` — four routes registered
6. Migrations applied; GitHub reference data synced if tables are empty

## Tests

Feature tests: `tests/Feature/Web/RttTest.php`, `tests/Feature/Web/RttReferenceDataTest.php`.  
Unit: `tests/Unit/RttServiceRateLimitTest.php`.

```bash
docker compose exec app php artisan test --filter=Rtt
```

## External links

- [API specification](https://realtimetrains.github.io/api-specification/)
- [NR reference data (GitHub)](https://github.com/realtimetrains/nr-reference-data)
- [UK railway stations (GitHub)](https://github.com/davwheat/uk-railway-stations)
- [API portal](https://api-portal.rtt.io)
