# Bus departures (bustimes.org)

Staff departure boards and a token-gated public display, powered by [bustimes.org](https://bustimes.org/) stop metadata, HTML departure boards, and JSON APIs for live journeys.

## URLs

| Surface | URL | Auth |
|---------|-----|------|
| Staff settings + preview | `/bus-departures` | Staff session + 2FA |
| Staff poll (JSON) | `/bus-departures/poll` | Staff |
| Service detail + display rules | `/bus-departures/services/{slug}` (JSON for sheet) | Staff |
| Display rules admin | `/bus-departures/display-rules` | Staff |
| Public wall board | `/d/bus-departures` | None |
| Public poll (JSON) | `/public/bus-departures/{BUSTIMES_PUBLIC_TOKEN}/poll` | Token in path |

Sidebar: **Buses → Bus departures** (staff). The public wall board is not linked from the sidebar.

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
| Destination filter (display scenes) | `app/Services/BusDepartureDestinationFilter.php` |
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
- `GET /bus-departures/display-rules` — Global display rules index
- `GET /bus-departures/services/{slug}?journey={id}` — Service detail JSON (includes `displayRules`, optional sample PTP stops)
- `POST /bus-departures/services/{slug}/display-rules` — Create PTP display rule
- `PUT /bus-departures/display-rules/{rule}` — Update rule
- `DELETE /bus-departures/display-rules/{rule}` — Delete rule
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

Board via/destination/notes text is stored in **`bus_service_display_rules`** (migration `2026_07_04_140000_create_bus_service_display_rules_table.php`, plus `notes` / destination ATCO / operator / `match_destination` column migrations):

| Column | Purpose |
|--------|---------|
| `service_slug` | Optional bustimes.org service slug (case-insensitive). When set, matches departures for that service. Can be combined with other match criteria. |
| `ptp_atco_codes` | JSON array of PTP stop ATCO codes that must appear **in order** on the upcoming journey (after the board stop, excluding terminus). Empty when matching by destination ATCO, destination text, service slug, or operator only. |
| `destination_atco_code` | Optional stop or station ATCO for the journey **terminus**. When set (and `ptp_atco_codes` is empty), matches any service terminating at that stop; station codes match any bay at the station. |
| `match_destination` | Optional exact board destination text (case-insensitive). Use to rewrite labels such as `Chellaston - Derby` → `Chellaston` without needing a terminus ATCO. |
| `operator_noc` | Optional National Operator Code (NOC). When set (and the other match fields are empty), matches any journey or trip operated by that operator. |
| `via` | Custom via label shown on the board (e.g. `University`) |
| `destination` | Custom destination label shown on the board (e.g. `Chellaston`) |
| `service` | Custom service line name shown on the board (e.g. `Allestree`). Does not change the bustimes.org service link slug. |
| `notes` | Custom notes shown on the board and in staff journey detail |

Rules are **global** — they match any service when upcoming PTP stops match in order, the journey terminus matches `destination_atco_code`, the board destination text matches `match_destination`, the departure service slug matches `service_slug`, and/or the service operator matches `operator_noc`. When multiple rules match, their destination, via, notes, and service outputs are **merged**; if several rules would set the same field, priority is PTP → destination ATCO → destination text → service slug → operator. Within a tier, the longest PTP match wins, then the highest rule `id`. Destination-text and service-slug rules also apply when journey/trip detail cannot be loaded. Rules are managed on **Buses → Bus departures → Display rules** (global list) and from the **journey dialog** or **service sheet** on the departures board.

Staff UI supports **create**, **edit**, and **delete**. Journey/trip detail shows **source** and **display** values separately for destination, via, and notes.

**Data loss note:** Rules live only in this table. They are not in bustimes.org or the `settings` table. Back up or export `bus_service_display_rules` when migrating environments.

## Board enrichment (live journeys)

For departures with a `journeyId` or `tripId`, `BustimesService` loads trip times and adds:

- **`boardDestination`** — scraped board destination before display rules
- **`destination`** — from matched display rule, else scraped/terminus name
- **`via`** — from matched display rule only (not auto-generated from stop names)
- **`displayNotes`** — from matched display rule
- **`sourceNotes`** — bustimes trip notes (staff poll only; stripped from public display API)

Destination-text rules (`match_destination`) also apply to departures without journey/trip detail, matching against the scraped board destination.

Departures cache key version: **`v7`** (live fetch without date/time; includes `operatingDate` with post-midnight timetable-day normalisation; header-aware column mapping).

## Display scene destination filter

Bus departure **display scenes** (playlists / wall displays) can optionally filter which services appear on the board by destination. This is configured per scene in **Displays → Scenes** and stored on `display_scenes.config`:

```json
{
  "stops": [
    { "atco_code": "109000008742", "label": "Park Farm" },
    { "atco_code": "109000009000", "label": "University" }
  ],
  "destinationFilter": {
    "mode": "include",
    "entries": [
      { "value": "Derby", "stops": null },
      { "value": "Nottingham", "stops": ["109000008742"] }
    ]
  }
}
```

| Field | Purpose |
|-------|---------|
| `mode` | `include` (only matching departures), `exclude` (hide matching departures), or `null` (no filter) |
| `entries` | One or more filter rules. Each entry is an object with `value` (destination name or stop/station ATCO) and optional `stops` (scene stop ATCO codes this rule applies to). Legacy string entries are still accepted. |
| `entries[].value` | Destination name or stop/station ATCO code |
| `entries[].stops` | When `null` or omitted, the entry applies to **all** stops on the scene. When set, the entry only applies to departures from those board sections. Sections with no applicable entries are not filtered. |
| `disabled` | When `true`, the filter is not applied; departures that would be hidden are shown with a `destinationFilterNote` instead |

Matching rules (final destination only — pass-through services are not matched):

- **Text entries** — case-insensitive match against the departure’s `destination` field only (substring or exact).
- **Stop ATCO codes** — match when the journey/trip **terminus** is that stop (requires `journeyId` or `tripId`).
- **Station ATCO codes** — match when the terminus is any bay at that station (requires journey/trip detail).

The filter runs **after** PTP display rule enrichment in `BusDepartureDestinationFilter::applyToBoard()`. It applies to display player and scene preview APIs only — the staff `/bus-departures` board does not use scene config.

| Layer | Path |
|-------|------|
| Filter service | `app/Services/BusDepartureDestinationFilter.php` |
| Scene validation | `app/Services/DisplaySceneService.php` |
| Scene config UI | `resources/js/components/displays/DisplaySceneConfigFields.vue` |
| Player / preview API | `PublicDisplayController`, `DisplayScenePreviewController` |

## Display scene board text

Bus scenes can override how destination, via, and notes appear on wall displays. Stored on `display_scenes.config.display`:

| Field | Purpose |
|-------|---------|
| `destinationSource` | Bustimes field used before display rules: `board`, `headsign`, `terminus`, or `journey` |
| `customDestination` | When set, shown for every departure and overrides source + display rules |
| `ignoreDestinationRules` | Use the selected source without applying global destination display rules |
| `customVia` | When set, shown for every departure |
| `ignoreViaRules` | Hide via text from display rules |
| `customNotes` | When set, shown for every departure |
| `ignoreNotesRules` | Hide notes from display rules |
| `stopColumn` | `label` (scene stop label on combined boards) or `member` (station bay/stop number, stop-group member label, or stop indicator) |

Applied in `BusDepartureSceneDisplay::applyToBoard()` after destination filtering. Staff `/bus-departures` board is unaffected.

### Non-breaking spaces in stop labels

Scene stop labels (and stop-group member labels shown in the Stop column) may include a non-breaking space via:

| Token | Result |
|-------|--------|
| `* *` | U+00A0 (preferred) |
| `&nbsp;` | U+00A0 |

Example: `S2, Midland* *Rd` renders as `S2, Midland Rd` without wrapping between Midland and Rd. Tokens are expanded at render time in `formatDisplayText()` / `stopDisplayLabel()` (`resources/js/lib/displayText.ts`); storage keeps the literal token.

When display text soft-wraps so a comma would be the last character on a line (for example `S1,` then `Midland Rd` on the next line), `DisplayWrapText` hides that comma so the lines read `S1` / `Midland Rd`.

## Custom stop groups

Named groups of stop ATCO codes are stored in `bus_stop_groups` / `bus_stop_group_members` and referenced by a system-generated code starting with **`CG`** (for example `CGPARKFARMAREA` for “Park Farm area”). They behave like bustimes stations:

- **Boards / scenes** — departures from all member stops are merged and sorted
- **Location lookup** — shows group name and member count
- **Destination filters & display rules** — matching a group code matches any member stop terminus

Manage groups under **Buses → Bus departures → Stop groups**.

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
| Destination filter | `tests/Unit/BusDepartureDestinationFilterTest.php` |
| Station member stops | `tests/Unit/BustimesStationTest.php` |
| Scene filter validation | `tests/Feature/Web/DisplayScenesTest.php` |
| Fixture | `tests/fixtures/bustimes-departures.html` |

Run inside Docker:

```bash
docker compose exec app php artisan test --filter=BusDepartures
docker compose exec app php artisan test --filter=Bustimes
```

`phpunit.xml` sets `BUSTIMES_PUBLIC_TOKEN=test-public-token` for feature tests.

## Troubleshooting

See [troubleshooting.md](troubleshooting.md#bus-departures-bustimesorg) for common issues (missing Expected times, empty via/destination, public poll 404, lost display rules).
