# Bus departures (bustimes.org)

Staff departure boards and a token-gated public display, powered by [bustimes.org](https://bustimes.org/) stop metadata, HTML departure boards, and JSON APIs for live journeys.

## URLs

| Surface | URL | Auth |
|---------|-----|------|
| Staff settings + preview | `/bus-departures` | Staff session + 2FA |
| Staff poll (JSON) | `/bus-departures/poll` | Staff |
| Service detail + display rules | `/bus-departures/services/{slug}` | Staff |
| Public wall board | `/d/bus-departures` | None |
| Public poll (JSON) | `/public/bus-departures/{BUSTIMES_PUBLIC_TOKEN}/poll` | Token in path |

Sidebar: **Transport → Bus departures** (staff), **Transport → Bus departures** display link (public board).

## Configuration

Set in `.env` (see `.env.example`):

| Variable | Purpose |
|----------|---------|
| `BUSTIMES_PUBLIC_TOKEN` | Non-empty token registers the public poll route and meta tag for the wall board. If unset, public routes are not registered. |
| `BUSTIMES_BASE_URL` | API/HTML base (default `https://bustimes.org`) |
| `BUSTIMES_DEPARTURES_CACHE_SECONDS` | Live departures cache TTL (default `45`) |
| `BUSTIMES_STOP_CACHE_SECONDS` | Stop metadata cache TTL (default `86400`) |

Laravel reads these via `config/services.php` → `services.bustimes.*`.

Board stop(s) are stored in the **`settings`** table under key `bus_departures`:

```json
{
  "stops": [
    { "atco_code": "109000008742", "label": "Park Farm" }
  ]
}
```

Legacy single-stop config (`bus_departures.stop`) is still read if the new key is absent.

## Architecture

| Layer | Path |
|-------|------|
| Bustimes API + HTML + enrichment | `app/Services/BustimesService.php` |
| PTP display rule matching | `app/Services/BusServiceDisplayRuleMatcher.php` |
| Board stop settings | `app/Support/BusDeparturesConfig.php` |
| Display rules model | `app/Models/BusServiceDisplayRule.php` |
| Staff controller | `app/Http/Controllers/Web/BusDeparturesController.php` |
| Public poll | `app/Http/Controllers/Web/PublicBusDeparturesController.php` |
| Staff page | `resources/js/pages/bus-departures/Index.vue` |
| Service modal | `resources/js/components/bus-departures/ServiceDetailSheet.vue` |
| PTP display rules UI | `resources/js/components/bus-departures/ServiceDisplayRulesSection.vue` |
| Public board | `resources/js/components/BusDeparturesBoard.vue` |
| Public entry | `resources/js/bus-departures-board.ts` |
| Shared helpers | `resources/js/lib/busDepartures.ts` |
| Staff routes | `routes/staff.php` |
| Public routes | `routes/web.php` |

### Staff routes

- `GET /bus-departures` — Inertia index (settings, multi-stop preview, 45s poll)
- `GET /bus-departures/poll` — JSON board snapshot
- `GET /bus-departures/services/{slug}?journey={id}` — Service detail JSON (includes `displayRules`)
- `POST /bus-departures/services/{slug}/display-rules` — Create PTP display rule
- `PUT /bus-departures/services/{slug}/display-rules/{rule}` — Update rule
- `DELETE /bus-departures/services/{slug}/display-rules/{rule}` — Delete rule
- `POST /bus-departures/settings` — Save board stops

### Data sources (bustimes.org)

| Data | Source |
|------|--------|
| Stop metadata | `GET /api/stops/{atco}/` |
| Live departures | `GET /stops/{atco}/departures` (HTML scrape) |
| Service detail | `GET /api/services/?slug=…`, `/api/services/{id}/`, `/api/stops/?service=`, `/api/operators/{noc}/` |
| Live journey | `GET /api/vehiclejourneys/{id}/details/` |

**Important:** Do **not** pass `date` and `time` query params when fetching live departures. bustimes.org returns a schedule-only table without the **Expected** column when those params are present.

## PTP display rules

Per-service via/destination text is stored in **`bus_service_display_rules`** (migration `2026_07_04_140000_create_bus_service_display_rules_table.php`):

| Column | Purpose |
|--------|---------|
| `service_slug` | bustimes service slug (e.g. `ta-derby-allestree`) |
| `ptp_atco_codes` | JSON array of PTP stop ATCO codes that must all appear on the upcoming journey |
| `via` | Custom via label shown on the board (e.g. `University`) |
| `destination` | Custom destination label (e.g. `Derby`) |

Rules are managed in the **service modal** (click a departure row → **Board display rules**). When a live journey’s upcoming PTP stops (excluding the terminus) contain all ATCO codes in a rule, that rule’s via/destination override the scraped text. If multiple rules match, the rule with the **most PTP stops** wins.

The staff UI supports **create** and **delete**; the `PUT …/display-rules/{rule}` route exists for updates but there is no edit form yet — delete and re-create to change a rule.

**Data loss note:** Rules live only in this table. They are not in bustimes.org or the `settings` table. Back up or export `bus_service_display_rules` when migrating environments.

## Board enrichment (live journeys)

For departures with a `journeyId`, `BustimesService` loads trip times and adds:

- **`destination`** — from matched display rule, else scraped/terminus name
- **`via`** — from matched display rule only (not auto-generated from stop names)
- **`futureStops`** — upcoming stops with expected times; PTP stops highlighted on the public board

Departures cache key version: **`v3`** (live fetch without date/time).

## Adding or changing bus departures — integration checklist

After any bus-departures work, verify **all** of:

1. `config/services.php` — `bustimes` block
2. `routes/staff.php` — seven bus-departures routes (including three display-rule routes)
3. `routes/web.php` — `/d/bus-departures` and conditional public poll prefix
4. `resources/js/components/AppSidebar.vue` — staff + display nav items
5. `vite.config.ts` — `resources/js/bus-departures-board.ts` entry
6. `.env.example` — `BUSTIMES_PUBLIC_TOKEN` (and optional bustimes vars)
7. Migration applied: `bus_service_display_rules`
8. `php artisan route:list --name=bus-departures`
9. Frontend built: `npm run build` (staff page + public board chunks)

Symptoms of missing wiring: 404 on `/bus-departures`, public board shows “Public API is not configured”, empty Expected column (stale cache or date/time fetch regression).

## Deploy steps (feature-specific)

After pulling bus-departures changes:

```bash
docker compose exec -T app php artisan migrate --force
docker compose exec -u root app sh -c "npm ci && npm run build && chown -R www:www /var/www/public/build"
docker compose exec -T app php artisan cache:clear
```

Confirm `.env` has `BUSTIMES_PUBLIC_TOKEN` set on the target environment.

Re-create **PTP display rules** in the service modal if the `bus_service_display_rules` table was truncated or restored without that table.

## Tests

| Test | Path |
|------|------|
| Feature | `tests/Feature/Web/BusDeparturesTest.php` |
| HTML parser | `tests/Unit/BustimesDeparturesParserTest.php` |
| Journey enrichment | `tests/Unit/BustimesJourneyBoardContextTest.php` |
| Fixture | `tests/fixtures/bustimes-departures.html` |

Run inside Docker:

```bash
docker compose exec app php artisan test --filter=BusDepartures
docker compose exec app php artisan test --filter=Bustimes
```

`phpunit.xml` sets `BUSTIMES_PUBLIC_TOKEN=test-public-token` for feature tests.

## Troubleshooting

See [troubleshooting.md](troubleshooting.md#bus-departures-bustimesorg) for common issues (missing Expected times, empty via/destination, public poll 404, lost display rules).
