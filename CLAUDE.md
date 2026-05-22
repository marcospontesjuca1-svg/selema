# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Architecture

The entire application is a **single self-contained HTML file** (`index.html`, ~336KB). There is no build step, no bundler, and no server. Everything — HTML, CSS, JavaScript, and hardcoded financial data — lives in one file.

### Data layer (lines ~940–2420)

Six `const` global arrays/objects hold all portfolio data. They are declared in this order:

| Variable | Type | Content |
|---|---|---|
| `BRADESCO_ACCRUED` | object | `{ KEY: { total, bonds: [...] } }` — accrued interest per bond per month |
| `SANTANDER_ACCRUED` | object | same structure as above |
| `SANTANDER_DATA` | array | `[{ key, year, month, label, net_worth, variation, mtd_return, ytd_return, ... }]` |
| `BRADESCO_DATA` | array | same shape, adds `bonds`, `mutual_funds`, `structured_notes` fields |
| `BRADESCO_INCOME` | object | `{ KEY: number }` — coupon/dividend income per month |
| `SANTANDER_EXPENSES` / `BRADESCO_EXPENSES` | objects | `{ KEY: { integra, amex, total } }` |

Month keys follow the pattern `MMM_YY` (e.g. `JAN_26`, `DEZ_24`). `const` arrays can be mutated (push/sort) but not reassigned.

### Persistence (lines ~4912–5330)

Three persistence mechanisms run in sequence at startup:

```
var IMPORT_STORE_KEY = 'selema_imports';  // must be declared BEFORE the calls below
init();
loadPersistedImports();   // reads localStorage key 'selema_imports'
loadFromRemote();         // fetches ./data.json from same origin (GitHub Pages)
```

- **`loadPersistedImports()`** — reads a full snapshot from `localStorage` and calls `applyImportData()`.
- **`loadFromRemote()`** — fetches `data.json` from `./data.json?_=<timestamp>` (no-cache). Used so GitHub Pages visitors all see the same data without needing to import.
- **`applyImportData(json, mode)`** — merges data into the global arrays via upsert-by-key; always calls `init()` at the end.
- **`saveImportToStorage()`** — snapshots all six globals to `localStorage['selema_imports']`.

**Critical:** `IMPORT_STORE_KEY` is a `var` (not `const`) and must appear **before** the `init()` / `loadPersistedImports()` calls, otherwise `localStorage.getItem(undefined)` silently returns null at startup.

### PDF import pipeline (lines ~5333–5530)

- `loadPDFJS()` — lazy-loads pdfjs-dist 3.11.174 from CDN.
- `extractPDFLines(pdfjsLib, buffer)` — groups text items by y-coordinate, sorts by x, returns array of line strings.
- `parseSantanderPDF(pages)` — US number format (`parseUSNum`), English→Portuguese month mapping.
- `parseBradescoPDF(pages)` — EU number format (`parseEUNum`, `1.234,56` → `1234.56`).
- Both parsers return `{ data: [...], accrued: {...} | undefined, expenses: {...}, income?: {...} }`.

### GitHub Pages publishing (`publishToGitHub`, line ~5134)

Uploads **only `data.json`** to the repo via the GitHub Contents API. Does NOT modify `index.html`. On the next page load, `loadFromRemote()` picks up the new `data.json`. The token is stored in `localStorage['selema_gh_token']`.

### Auth (lines ~631–697)

Login gate uses `window._sHash()` (simple hash). Credentials stored in `localStorage`:
- `selema_pwd_hash` — hashed password (default: hash of `'7680'`)
- `selema_login` — username (default: `'selema'`)

Session unlocked via `sessionStorage['selema_auth']`.

### Rendering

- `renderSantander(monthKey)` / `renderBradesco(monthKey)` — KPI cards, alloc bar.
- `renderSantAnnual(year)` / `renderBradAnnual(year)` — annual performance tables.
- `renderAnalytics()` / `renderAnual(year)` — cross-institution analytics tab.
- `renderPosicoes(monthKey)` — bond positions table with accrued interest.
- `renderConsolidated()` — consolidated PL across both institutions.
- All charts are rendered inline as SVG strings written to `innerHTML`.

### localStorage keys reference

| Key | Purpose |
|---|---|
| `selema_imports` | Full data snapshot (all six globals) |
| `selema_pwd_hash` | Hashed password |
| `selema_login` | Username |
| `selema_ai_key` | Anthropic API key |
| `selema_gh_token` | GitHub Personal Access Token |
| `selema_gh_owner/repo/branch` | GitHub repo config for publishing |

## Development

No build step. Open `index.html` directly in a browser. `pdfjs-dist` is listed in `package.json` only for local Node.js testing of the PDF parsers.

To test PDF parsing locally:
```bash
node -e "
const pdfjs = require('./node_modules/pdfjs-dist/legacy/build/pdf.js');
// ... test parseSantanderPDF / parseBradescoPDF logic
"
```

## Key constraints

- Number formats: Santander PDFs use US format (`1,234.56`); Bradesco PDFs use EU format (`1.234,56`).
- When adding a new month's hardcoded data, add entries to all relevant globals (`SANTANDER_DATA`, `SANTANDER_ACCRUED`, `SANTANDER_EXPENSES`, etc.) using the same `MMM_YY` key.
- `embedData(html)` strips previous `<!--SELEMA_DATA_START-->…<!--SELEMA_DATA_END-->` markers before injecting — used only by `backupHTML()` for local file downloads.
- Git push requires token: `git remote set-url origin "https://${TOKEN}@github.com/marcospontesjuca1-svg/selema.git"` — the local proxy blocks direct push.
