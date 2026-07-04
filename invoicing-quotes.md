# Invoicing and quotes

Staff UI for **Projects** (`/invoices`) covers both invoices and versioned quotes. Quotes and invoices share one document model (`invoices` table with `document_type`), sectioned line items, and the same PDF Blade template with quote/invoice-specific blocks.

## UI entry points

| URL | Purpose |
|-----|---------|
| `/invoices` | **Projects** index — All / Invoices / Quotes tabs; row click opens detail |
| `/invoices/new` | New direct invoice |
| `/quotes/new` | New quote |
| `/quotes/{id}` | Quote show (version history, accept, mark sent) |
| `/quotes/{id}/edit` | Quote edit (creates new version if current is sent) |
| `/invoices/{id}` | Invoice show |
| `/invoices/{id}/edit` | Draft invoice edit |

Currency amounts display as **`£`** (not `GBP`) via `resources/js/lib/currency.ts`.

## Document model

- **`document_type`:** `invoice` | `quote` on `invoices`.
- **`quote_groups`:** Base reference (`T4S-0001`), PO number, acceptance, link to converted invoice.
- **Quote versions:** Stored as separate `invoices` rows with `version`, `is_current_version`, full display number (e.g. `T4S-0001.12`). Saving after **Mark as sent** creates a new version.
- **Sections:** `invoice_sections` + `invoice_lines.section_id`. Both quotes and direct invoices use sections in the editor ([SectionedLineEditor.vue](../resources/js/components/invoices/SectionedLineEditor.vue)).
- **Accept flow:** Mark sent → Accept (optional PO) → draft invoice at base number; re-accept updates draft invoice content and PO.

### Section fields

| Field | Purpose |
|-------|---------|
| `title` | Section heading; PDF subtotal label is **`{title} total`** |
| `date` | Optional section date |
| `show_date` | When true, section date appears next to title on PDF |

When a section has a date, **new lines** inherit that date and **`show_date` defaults to off** on lines (date shown at section level).

### Line fields

| Field | Purpose |
|-------|---------|
| `description` | Required; main line text on PDF |
| `notes` | Optional; rendered **below** description on PDF (smaller text) |
| `date`, `show_date` | Per-line date visibility on PDF |
| `unit`, `show_unit` | Unit label; optional `(Day rate)` / `(per …)` suffix on PDF |
| `quantity`, `unit_price`, `amount` | Pricing |

Line **notes** are edited via the sticky-note button beside the description in the section editor (modal with description + notes).

## PDF generation

- **Service:** [InvoicePdfService.php](../app/Services/InvoicePdfService.php) renders [pdf.blade.php](../resources/views/invoices/pdf.blade.php) → HTML → **Browsershot** (headless Chrome via Puppeteer).
- **Why Chrome:** Full CSS fidelity (fonts, backgrounds, multi-page layout). The Chrome binary lives in `/opt/puppeteer-cache` (named Docker volume); install once per environment — see [deployment.md](deployment.md#6b-browsershot--puppeteer-chrome-one-time-per-environment).
- **Quote vs invoice:** Heading (“Quote” / “Invoice”), meta labels, due/bank block (invoices only), PO on issued invoices, grand total placement.
- **Project Date:** Header label for supply dates (not “Date of supply”).
- **Single section:** Section titles hidden on PDF when only one section exists.
- **Multi-page layout:**
  - Business footer (address, contact, footer note) uses `page-break-inside: avoid` and must not split across pages; positioned at bottom of last page via layout script.
  - **Page numbers** (`Page X of Y`) come from Browsershot’s Chrome footer template (`showBrowserHeaderAndFooter()`), not from body HTML — always at the bottom of every page.
- **Stored PDFs:** Generated on quote mark-sent and invoice issue; path `invoices/{id}.pdf` in storage.

Preview: `GET /quotes/{id}/pdf/preview` or `GET /invoices/{id}/pdf/preview`.

## Architecture

| Layer | Path |
|-------|------|
| Invoice controller | `app/Http/Controllers/Web/InvoiceController.php` |
| Quote controller | `app/Http/Controllers/Web/QuoteController.php` |
| Invoice service | `app/Services/InvoiceService.php` (sections sync, list scoping, format for API) |
| Quote service | `app/Services/QuoteService.php` (versioning, accept, copy) |
| PDF service | `app/Services/InvoicePdfService.php` |
| Section editor | `resources/js/components/invoices/SectionedLineEditor.vue` |
| Forms | `resources/js/pages/invoices/Form.vue`, `resources/js/pages/quotes/Form.vue` |
| Show pages | `resources/js/pages/invoices/Show.vue`, `resources/js/pages/quotes/Show.vue` |
| Index | `resources/js/pages/invoices/Index.vue` (Projects) |
| Domain types | `resources/js/types/domain/index.ts` (`InvoiceRow`, `QuoteRow`, `InvoiceSection`, `InvoiceLine`) |
| Tests | `tests/Feature/Web/QuotesAndSectionsTest.php`, `InvoiceLineShowFlagsTest.php` |

### Shared DB tables (when `DB_SHARED_DATABASE` is set)

| Table | Purpose |
|-------|---------|
| `invoices` | Invoices and quote versions |
| `invoice_lines` | Line items (`notes`, `show_date`, `show_unit`, `section_id`) |
| `invoice_sections` | Sections (`date`, `show_date`) |
| `quote_groups` | Quote grouping, PO, conversion |

Recent migrations agents should not forget after restore:

- `2026_06_25_*` — quote groups, sections, document_type extensions
- `2026_06_26_100000_add_date_to_invoice_sections_table.php`
- `2026_06_27_100000_add_notes_to_invoice_lines_table.php`

## Routes (`routes/staff.php`)

Quote and convert routes **must** be registered alongside invoice routes. After a partial restore, controllers may exist while routes are missing (404 on `/quotes/*`).

**Invoices (extra):**

- `POST invoices/{invoice}/convert-to-quote` → `invoices.convert-to-quote` (draft invoice with client, not from quote)
- `POST invoices/{invoice}/copy-to-new-quote` → `invoices.copy-to-new-quote`

**Quotes (full set):**

```bash
docker compose exec app php artisan route:list --name=quotes
docker compose exec app php artisan route:list --name=invoices.convert
```

Expect **13** named `quotes.*` routes including `store`, `show`, `edit`, `update`, `destroy`, `mark-sent`, `cancel`, `accept`, `copy-to-new-quote`, `restore-version`, `pdf`, `pdf.preview`.

Ensure `use App\Http\Controllers\Web\QuoteController;` is present at the top of `staff.php`.

## Deploy checklist

**Migrations and frontend build are separate steps.** `npm ci && npm run build` does **not** run migrations.

```bash
# Schema (including section date + line notes columns)
docker compose exec -u root app php artisan migrate --force

# Frontend (section editor, Projects index, sticky-note modal, etc.)
docker compose exec -u root app sh -c "npm ci && npm run build && chown -R www:www /var/www/node_modules /var/www/public/build"
```

PDF changes are server-side (Blade + Browsershot); they do not require a frontend rebuild unless you also changed Vue pages.

## Product actions

| Action | Where | Notes |
|--------|-------|-------|
| Convert to quote | Invoice show/edit | Draft only; must have client; not already linked to a quote group |
| Copy to new quote | Quote or invoice show/edit | New quote group; no link to source |
| Mark sent | Quote show | Locks version; stores PDF |
| Accept | Quote show | Requires client + sent; creates/updates draft invoice |
| Restore version | Quote show history | Creates new version from historical row |

## Adding or changing invoicing — integration checklist

1. **Routes** — `routes/staff.php` (quote + convert/copy routes).
2. **Validation** — `InvoiceStoreRequest`, `InvoiceUpdateRequest`, `QuoteStoreRequest`, `QuoteUpdateRequest` (`sections.*`, line `notes`, section `date`/`show_date`).
3. **Services** — `InvoiceService::syncSections`, `exportSectionsPayload`, `formatInvoice`; `QuoteService` for version/accept flows.
4. **PDF** — `InvoicePdfService` + `pdf.blade.php` (test multi-page footer if layout changes).
5. **Frontend** — `SectionedLineEditor.vue`, forms, show pages, `resources/js/types/domain/index.ts`.
6. **Migrations** — idempotent `hasColumn` guards when using shared DB.
7. **Tests** — `php artisan test --filter=QuotesAndSections`.
