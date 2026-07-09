# Brunch Shelf Availability Dashboard — Handoff

> **Update instructions**: When asked to update this document, replace each section fully with current state. Keep it factual and terse. No section should grow unboundedly — prune stale entries. Always save this file before closing a session.
>
> **New session instructions**: Read this file first. Do not ask the user to re-explain context. Pick up from "Next Steps."

---

## 1. Goal

Two fully separate dashboards:

1. **Original (`generate_report.py` → `index.html` / `tool.html`)** — bounding-box annotation data, different schema. Do not touch.
2. **OSA (`generate_report_osa.py` → `index_osa.html` / `tool_osa.html`)** — On Shelf Availability data, generic upload portal. This is the active dashboard.

Both are self-contained HTML files. No server needed; share by sending the file.

**As of 2026-07-08, `index_osa.html` and `tool_osa.html` are identical generic shells** — neither embeds any project's data. Both always open straight to the upload modal and only show data after the user uploads files. This was a deliberate behavior change (previously `index_osa.html` shipped with the Brunch Before/After dataset baked in and auto-loaded it on open, which caused confusion when someone uploaded a different client's files: if the upload silently failed validation, the stale Brunch numbers stayed on screen with no clear signal that the new upload hadn't taken effect). `generate_report_osa.py` still reads `Dump Tables/Before|After` and prints Brunch KPI stats to the console for sanity-checking, but no longer embeds that payload into the HTML output.

---

## 2. OSA Dashboard — Current State

### Data model

- **One row per (store, audit_date) visit** — no pooling across dates. If a store was audited twice, there are 2 rows, each showing only that date's SKUs and images.
- **KPI cards** use unique-store counts (% stores with ≥1 self SKU on any visit).
- **SKU availability table** counts unique stores where each SKU appeared on any visit.
- **Histogram** reflects per-visit SKU count distribution.

### Key identifiers

| Field | OSA file | Gallery file |
|---|---|---|
| Store name | `shop_name` | `store_name` |
| Client ID | `client_shop_id` | `client_store_id` |
| SKU | `class_name` | — |
| Brand type | `competition` (`"self"` / `"competitor"`) | — |
| Presence | `On_Shelf_Availability` (1=present, 0=absent) | — |
| Date | `audit_date` | `audit_date` |
| Image link | — | `image_link` |

### Gallery join

Images are joined per (store, date) — not pooled across dates:
1. Primary: `client_shop_id` (OSA) ↔ `client_store_id` (gallery) + matching `audit_date`
2. Fallback: `shop_name` ↔ `store_name` + matching `audit_date`

This is necessary because After OSA `shop_name` has garbled Arabic encoding — only client_id join works for ~80% of After stores.

### Date normalisation (`_normalize_date`)

Handles three formats from openpyxl:
- Python `datetime` object → `strftime("%Y-%m-%d")`
- ISO string `"2026-04-15T00:00:00.000Z"` → first 10 chars
- Excel serial integer (40000–60000) → `datetime(1899,12,30) + timedelta(days=int(d))`

### Sheet selection (`load_xlsx` / JS `_bestSheet`)

Both Python and JS pick the sheet with the **most non-None/non-empty header columns**. This avoids the pivot summary sheet that appears first in the After OSA xlsx.

### ShelfWatch image viewer

All image links are wrapped with `https://view.shelfwatch.io/?url=` (idempotent — never double-added). Both Python `_viewer_url()` and JS `_vUrl()` handle this.

### QC system

- Keyed by `store_name + "||" + audit_date` (per-visit, not per-store).
- States: `unchecked → opened (auto) → done (manual)`.
- Flag with text note. Flag is independent of done status.
- Progress bar counts against `total_visits` (visit rows), not `total_stores`.
- Export/import JSON; import merges (does not replace) existing state.

---

## 3. Files

| File | Role |
|---|---|
| `generate_report_osa.py` | **Primary.** Python pipeline + full HTML/JS template. All edits here; regenerate outputs by running it. |
| `index_osa.html` | Generated — embedded Before + After data. Do not edit directly. |
| `tool_osa.html` | Generated — generic upload shell (no embedded data). |

### Input data files (`Dump Tables/`)

| File | Purpose |
|---|---|
| `Before/kpi_927_onshelfavailability - Before.xlsx` | Before OSA data (186,666 rows) |
| `Before/kpi_927_gallery_dump.xlsx` | Before image gallery (4,299 rows) |
| `After/kpi_927_onshelfavailability - After.xlsx` | After OSA data (186,666 rows) |
| `After/kpi_927_gallerydump - After.xlsx` | After image gallery (4,299 rows) |

### Brunch reference stats (console-only sanity check, not embedded — see §1)

Before: 1,371 unique stores → 1,987 visit rows (616 stores have ≥2 visits), 78.0% availability, 31 own-brand SKUs, dates 2026-04-15 – 2026-05-15.
After: 80.7% availability, Δ = +2.7pp.

### Oetker reference stats (2nd client, verified via headless-browser upload test, 2026-07-08)

`1215_K_138_April.xlsx`: 238,998 rows → 5,013 stores, 5,017 visits, 76.2% availability.
`1215_K_138_Feb_1st_to_March_8th.xlsx`: 258,378 rows → 5,147 stores, 5,248 visits, 76.6% availability.

---

## 4. Python Pipeline (`generate_report_osa.py`)

### `compute_kpis(osa_records, gallery_records)`

Returns one store entry per (store, audit_date) pair. Key structures:
- `store_date_self[store][date]` — set of self SKUs present on that visit
- `store_date_comp[store][date]` — set of comp SKUs present on that visit
- `gal_date_links_by_cid[cid][date]` — image links for that visit (by client_id)
- `gal_date_images_by_cid[cid][date]` — image IDs (for count)
- `gal_date_links_by_name[name][date]` — fallback by store name

Returns dict with: `total_stores` (unique), `total_visits` (rows), `stores`, `self_skus`, `comp_skus`, `histogram`, `date_trend`, `overall_avail_rate`, etc.

### `compute_delta(before, after)`

Aggregates per store by **max `n_self_skus` across all visits** before computing the delta. This avoids the problem of multiple rows per store colliding in a dict.

---

## 5. JavaScript (`HTML` template string in `generate_report_osa.py`)

### Key globals

| Variable | Purpose |
|---|---|
| `B`, `A`, `D` | Before KPIs, After KPIs, delta object |
| `HAS_AFTER` | whether After data is loaded |
| `viewMode` | `"both"` / `"before"` / `"after"` |
| `afterStoreMap` | `name → most-recent After visit` for Both-mode comparison |
| `afterSkuMap` | `sku → After {stores, rate}` for Both-mode SKU table |
| `histBucket`, `histBucketView` | active histogram bucket filter |
| `skuF`, `rangeF`, `qcF`, `q` | sidebar filter state |
| `page`, `PER=50` | store table pagination |

### `visitKey(s)`

`s.name + "||" + (s.date || "")` — unique key per visit row. Used for all QC operations (open, done, flag, visited links). Replaced old `s.name`-keyed QC.

### `_bestSheet(wb)`

Iterates all sheets in a SheetJS workbook and returns the sheet with the most non-empty header-row values. Mirrors Python `load_xlsx`. Used in both `parseXlsxHeaders` (header detection for column mapper) and `readFullFile` (full data read).

### `computeKpisJS(records, colMap, galleryRecords=[])`

Full rewrite to match Python model:
- Per-visit rows (one per store+date pair)
- Gallery join via `client_shop_id` auto-detected from OSA rows (tries `r['client_shop_id']` first, then `r[colMap.storeId]`)
- Gallery lookups: `galLinksByCid[cid][date]`, `galLinksByName[name][date]`
- `Object.entries(skuStores)` — was incorrectly `skuStores.entries()` (plain object, not Map)
- Returns `total_visits` alongside `total_stores`

### `computeDeltaJS(before, after)`

Aggregates by max `n_self_skus` per store across all visit rows before computing delta.

### Export to Excel (`exportExcel()`)

Toolbar button "⬇ Export Excel" (teal), next to Load Files / Export QC. Downloads a real `.xlsx` via the bundled SheetJS (`XLSX.utils.json_to_sheet` + `XLSX.writeFile` — no CDN needed).

- Exports `filtered()` — the same rows currently shown in the store table, respecting all active sidebar filters (search, SKU, range/histogram bucket, QC status).
- **One row per image.** Each visit (store+date) explodes into one row per entry in `s.all_links`; visits with zero images still emit a single row with blank image fields so no filtered store is silently dropped.
- Columns: Store Name, Client ID, Shop ID, Audit Date, Total Visits, Self SKUs Count/list, Competitor SKUs Count/list (top 10), Total Images, QC Status, Flagged, Flag Note, Image #, Image URL. In Both-mode (`viewMode==="both"` and `HAS_AFTER`) two extra columns are added: After Self SKUs Count and Delta.
- Filename: `osa_export_<viewMode>_<date>.xlsx`.
- Verified via headless Chrome (puppeteer-core) in both "both" view (4,296 image rows from 1,987 visits) and "before" view with the "0 SKUs" filter (978 rows from 522 visits) — column set and row explosion confirmed correct.

### Upload modal

4 file zones in a 2×2 grid:
```
Before                  After
[OSA Data *]            [OSA Data (optional)]
[Gallery Dump (opt)]    [Gallery Dump (opt)]
```
Column mapping is auto-detected from the Before OSA file and applied identically to After. Gallery columns are auto-detected from the gallery file (tries `client_store_id`, `store_name`, `image_link`, `image_id`, `audit_date`).

**Column auto-detect (`showColMapper` / `pick()`)** does an exact-match pass across all keywords first, then a substring-match pass — this avoids e.g. `category_id` beating `category_name` just because it happens to sort earlier in the header list. Keyword lists are intentionally broad since this is a generic portal expected to ingest other clients' exports with different column names, not just the Brunch schema (`shop_name`/`class_name`). Verified against a real second client (Oetker) whose OSA export uses `store_name`, `Display_Name` (SKU-equivalent — 47 distinct self-brand product names vs. `category_name`'s 11 broad categories, so `display_name` is prioritized over `category_name`), `competition`, `audit_date`, `On_Shelf_Availability`, `client_store_id` — none of which are literally `class_name`/`shop_name`, so the old keyword lists auto-mapped nothing for SKU and blocked upload with "Please map all required columns." Extend the keyword lists in `pick()` calls if a future client's headers still don't auto-map.

### Background parsing worker (`WORKER_SRC` / `WORKER_LOGIC_JS`)

All large-file work (`XLSX.read`, `sheet_to_json`, `computeKpisJS`) runs inside a Web Worker, not the main thread — this is what makes the upload modal's progress bar animate smoothly and keeps the tab from looking frozen during a multi-second synchronous parse.

- `WORKER_LOGIC_JS` is a Python string in `generate_report_osa.py` (message handler + `_bestSheet` + `readHeaders` + `parseCsvW` + `readFullFileW` + `computeKpisJS` — the last is the single source of truth; it no longer exists on the main thread at all).
- At generation time, `main()` extracts the SheetJS source already inline in `HTML` (between the `<script>/*! xlsx.js` markers — single source of truth, still used as-is by `exportExcel()` on the main thread) and concatenates it with `WORKER_LOGIC_JS` into one string, JSON-encoded into `const WORKER_SRC = ...;` near the top of the page script.
- `makeWorker()` does `new Worker(URL.createObjectURL(new Blob([WORKER_SRC], {type:"text/javascript"})))` — fully self-contained, no `importScripts`, no separate file to host. Works fine from a `file://` page in Chrome (verified).
- `getHeadersViaWorker(file)` and `processFileViaWorker(file, colMap, galleryFile, onPhaseCb, onProgressCb)` spin up a fresh Worker per call and terminate it when done; each posts `{type:"phase"|"progress"|"done"|"error"}` messages back.
- **Before and After are processed sequentially, not in parallel** — see the Known Issues log entry below for why (`Promise.all` measured 3x slower due to memory contention).
- This roughly doubles the SheetJS footprint in the output HTML (once inline for `exportExcel()`, once inside the `WORKER_SRC` string) — file size went from 355 KB to 645 KB. Acceptable for a local single-file tool.

### SheetJS

`xlsx.mini.min.js` (~280 KB) is **bundled inline** in both output HTML files. No CDN dependency — works in the claude.ai artifact and offline.

**`dense:true` is required on every `XLSX.read(buffer, {type:"array", ...})` call.** Without it, SheetJS parses a sheet into a sparse object keyed by cell address (`{A1:cell, B1:cell, ...}`). For large sheets (rows × cols in the millions of cells — e.g. ~239k rows × 38 cols ≈ 9M cells, seen with a real client file) this blows past V8's practical property-count limit. In Node's full `xlsx` package this throws `RangeError: Too many properties to enumerate`; in the bundled mini build in-browser it fails silently instead — `_bestSheet` returns a sheet object with `ws['!ref']` undefined and `sheet_to_json` returns `[]`, with no exception anywhere in the chain. This reads as "upload succeeded, 0 rows of data" with no error message, which is very hard to diagnose without instrumenting `readFullFile` directly. `dense:true` stores rows as arrays instead, which `XLSX.utils.sheet_to_json` handles transparently and avoids the property-count blowup. Both `parseXlsxHeaders` and `readFullFile` pass `dense:true` now.

---

## 6. Known Issues / Failures Log

| Attempt | What happened | Resolution |
|---|---|---|
| After images "No images" | After OSA `shop_name` garbled Arabic → name-based gallery join failed for 80%+ of stores | Dual lookup: `client_shop_id` primary, `shop_name` fallback |
| After availability = 0% | After xlsx has pivot sheet as first sheet → wrong sheet loaded | `load_xlsx` (Python) and `_bestSheet` (JS) pick sheet with most header columns |
| 5-digit date serial numbers | openpyxl returned raw Excel integers (e.g. 46128) | `_normalize_date` converts via `datetime(1899,12,30) + timedelta(days=d)` |
| Multi-visit data mismatch (Obour_0529) | Pooling visits → wrong image count and SKUs for a specific date | Switched to per-visit row model: one row per (store, date) |
| SheetJS CDN blocked in artifact | Artifact CSP blocks external scripts → xlsx upload silently failed | Bundled `xlsx.mini.min.js` inline |
| Column mapper showing 1 column | `parseXlsxHeaders` used `SheetNames[0]` → pivot sheet | Fixed with `_bestSheet()` |
| `skuStores.entries is not a function` | `skuStores` was plain object `{}`, not a `Map` | Changed to `Object.entries(skuStores)` |
| No gallery upload in artifact | Upload modal only had Before/After OSA zones | Added Before Gallery + After Gallery zones; gallery join wired into `computeKpisJS` |
| Export QC button not downloading | Anchor element not appended to DOM before `.click()` + immediate `revokeObjectURL` → fails in Safari | Append to body, click, remove, revoke in `setTimeout(100)` |
| Import QC showing "Error: invalid file" | `try-catch` wrapped both JSON parse and UI updates (`renderStoreTable` etc.) — any UI error masked as invalid file | Split: only JSON parse in `try-catch`; UI updates outside |
| Browser tab hangs during upload processing | `XLSX.read()` + `computeKpisJS` loop (186k rows) block main thread synchronously | `computeKpisJS` made async with `setTimeout(0)` yield every 10k rows; progress shown in upload note |
| Excel export undercounted images | Gallery `links` list was capped at 3 per visit (UI preview only) — ~12% of visits (234/1989) have 4–7 images | Added uncapped `all_links` field (Python `compute_kpis` + JS `computeKpisJS`) alongside the capped `links`; export uses `all_links` |
| Oetker (2nd client) upload showed "0 stores" / no data | Two compounding bugs: (1) `XLSX.read` missing `dense:true` — a large sheet (238,998 rows × 38 cols ≈ 9M cells) silently parsed to an empty sheet in-browser (Node's full `xlsx` package throws `RangeError: Too many properties to enumerate` for the same file/options, confirming it's a sparse-object cell-count limit, not a data problem); (2) column auto-detect keyword lists only matched the Brunch schema (`class_name`, `shop_name`) so Oetker's `Display_Name`/`category_name` SKU column never auto-mapped, blocking upload on "Please map all required columns." Root-caused by loading the raw files with headless Chrome + puppeteer-core and instrumenting `readFullFile` directly — the failure was silent (no exception, no error toast) both times. | Added `dense:true` to both `XLSX.read` calls; broadened `pick()` keyword lists (see §5 Upload modal); verified full pipeline against both real Oetker files end-to-end (5,013 / 5,147 stores, 76.2% / 76.6% availability, matching an independent Python cross-check) |
| Live dashboard showed stale Brunch data after a failed/no upload | `index_osa.html` auto-loaded embedded Brunch data on `DOMContentLoaded` before any upload; if a new upload then failed validation, the old Brunch numbers stayed on screen with no indication the new file never loaded | Dashboard is now a fully generic, stateless-on-load portal — `index_osa.html` and `tool_osa.html` are both blank shells that always open to the upload modal (see §1) |
| Upload "stuck" for 5+ minutes with no feedback, tab felt frozen | `XLSX.read` + `sheet_to_json` + `computeKpisJS` all ran synchronously on the main thread with only a plain text note (no visual progress bar) — for a ~239k-row file this blocks the tab for tens of seconds and, if Before+After are both uploaded, the second file's read doesn't even start until the first fully finishes, with zero indication anything is happening. Confirmed via headless-Chrome timing that a single ~239k-row file takes ~17-22s to parse+compute even standalone. | Moved all file reading/parsing/KPI-computation into a Web Worker (`WORKER_SRC`, built from `WORKER_LOGIC_JS` + the same inline SheetJS source, Blob-ed at runtime — see §5). Added a real animated progress bar per file (indeterminate stripe during read, real % during the compute phase). **Important**: initially ran Before+After workers concurrently (`Promise.all`) expecting a ~2x speedup — measured the opposite: two ~9M-cell parses at once thrashed memory on an 8GB test machine and took 118s total vs. 51s running them one at a time. Switched to sequential Worker calls (still off the main thread, so the UI stays responsive) — see §5 "Background parsing worker." |

---

## 7. Lint Status

Run: `python3 -m flake8 generate_report_osa.py --max-line-length=120 --extend-ignore=E501,W503,E302,E303,E305`

- **E221** (multiple spaces before operator): intentional alignment in constants and `compute_kpis` variable declarations. Not bugs.
- **E2513** (invalid unescaped ESC char): false positive from pylint scanning ESC bytes inside the inline SheetJS JS string (line ~1754). Not a Python bug.
- **Syntax**: clean (`python3 -m py_compile` passes).

---

## 8. Hosting

| | |
|---|---|
| **GitHub Repo** | https://github.com/Kartik-15/osa-dashboard |
| **Live Dashboard** | https://kartik-15.github.io/osa-dashboard/ |
| **Claude Artifact** | https://claude.ai/code/artifact/cec219db-840a-4f57-ab2e-bba6b1dbfc95 (preview only, not shareable) |

### Deployment

Repo contains two committed files: `index.html` (redirects to `index_osa.html`) and `index_osa.html` (full embedded dashboard). GitHub Pages serves from `main` branch root.

To deploy an update:
1. Run `python3 generate_report_osa.py`
2. `git add index_osa.html && git commit -m "Regenerate" && git push`

GitHub Pages redeploys automatically within ~1 minute.

**Latest deploy**: commit `a53abf0` (2026-07-01) — Excel export feature. Pushed to `origin/main`, live at the URL above.

**⚠️ Not yet deployed (as of 2026-07-08)**: `generate_report_osa.py` and the regenerated `index_osa.html`/`tool_osa.html` on disk now contain the `dense:true` parse fix, broadened column auto-detect, and the "always start fresh, no embedded data" behavior change (see §1, §5, §6). These are working-tree changes only — not committed or pushed. The live dashboard at the URL above still has the old embedded-Brunch-data behavior and the silent large-file parse failure. Commit and push `index_osa.html` (and optionally `tool_osa.html` + `generate_report_osa.py`) to ship the fix — confirm with the user first since this updates a shared live URL.

**⚠️ Local `index.html` is NOT the committed redirect.** Working-tree `index.html` currently holds a full ~1.8MB dashboard build instead of the 1-line `<meta http-equiv="refresh">` stub committed at `e278dd1`. This predates this session (not caused by any change made here) and was deliberately left uncommitted/unpushed so the live redirect isn't broken. Only `index_osa.html` has been pushed. Needs investigation next session — see Next Steps.

---

## 9. Next Steps

1. **Deploy the 2026-07-08 fix** (see §8): commit + push the regenerated `index_osa.html` (and `tool_osa.html`) so the live dashboard gets the `dense:true` parse fix, broadened column auto-detect, and fresh-start-on-load behavior. Not yet done — confirm with user before pushing.
2. **Resolve local `index.html` discrepancy** (see §8): decide whether to discard the local build and restore the committed redirect stub, or something else — then commit/push if a change is needed.
3. **Upload mode — gallery column mapper**: If gallery files use non-standard column names, auto-detection may fail silently (images show as 0). Consider adding a small gallery column mapper UI step.
4. **Both-mode drill panel — After visit images**: Currently shows the most-recent After visit's images for any Before row. Ideally match by (store, date) when possible.
5. **Mobile layout**: Sidebar hidden on narrow screens; consider collapsible drawer.

---

*Last updated: 2026-07-08 (session 4) — fixed silent large-file parse failure (missing `dense:true` in `XLSX.read`, hit a V8 property-count limit on a ~9M-cell client file), broadened column auto-detect keywords beyond the Brunch schema, and made `index_osa.html`/`tool_osa.html` both start blank/upload-first instead of auto-loading embedded Brunch data. Verified end-to-end against two real Oetker files via headless-Chrome + puppeteer-core. Not yet committed/pushed — see §8 and Next Steps #1.*
