# PrimoPets Playwright Scraper

Scrapes all products from [primopets.store](https://primopets.store) (Wix-based) using **Playwright** (headless Chromium), maps each product and its variants into the provided Excel template, and outputs a filled workbook while preserving the original template formatting.

## Prerequisites
- Node.js 18+
- Template file: `speadsheet.xlsx` (in project root)
- EAN list: `ean.xlsx` (in project root)

## Install & Run
```bash
npm install
npx playwright install chromium
npm run scrape
```

## Configuration (optional `.env`)
| Variable | Default | Description |
|---|---|---|
| `TEMPLATE_PATH` | `speadsheet.xlsx` (absolute path) | Path to template workbook |
| `EAN_PATH` | `ean.xlsx` (absolute path) | Path to EAN pool file |
| `OUTPUT_PATH` | `speadsheet-filled.xlsx` | Output path (separate from template) |
| `CONCURRENCY` | `4` | Number of parallel browser pages |
| `MAX_RETRIES` | `1` | Retry count for failed page loads |
| `PAGE_TIMEOUT` | `20000` | Navigation timeout in ms |

## How It Works

1. **Sitemap discovery** — Loads `/sitemap.xml` via Playwright, finds `store-products-sitemap.xml`, and extracts all `/product-page/` URLs.

2. **Product extraction** — Visits each page headlessly, extracting data from:
   - **JSON-LD** `Product` schema (title, price, SKU, image, brand)
   - **`data-hook`** attributes (product title, description)
   - **`og:image`** meta tag (fallback image)

3. **Variant detection** — Interacts with Wix UI to find variant options:
   - **Dropdowns** (sizes, flavors): clicks `[data-hook="dropdown-base"]`, reads `[role="option"]` items from opened listbox
   - **Color pickers**: reads hex values from `[data-hook="color-picker-item"]` radio inputs, converts to Hebrew color names
   - Each option combination produces a **separate row** in the output

4. **Brand detection** — If JSON-LD doesn't provide a brand, matches title/description against 19 keyword rules (e.g., "פרו פלאן" → Pro Plan, "קונג" → Kong). Unidentified brands are set to `null`.

5. **Classification** — Automatically classifies products into:
   - Lifecycle stage (גור / בוגר / מבוגר)
   - Animal size (קטן / בינוני / גדול / ענק)
   - Animal type (כלב / חתול)

6. **SKU generation** — Deterministic format: `{brand-code}-{product-slug}-{variant-options}`. Example: `kong-kong-classic-toy-medium`.

7. **EAN handling**:
   - Uses barcode from JSON-LD if available (GTIN-13)
   - Otherwise, assigns the next unused EAN from `ean.xlsx`
   - Used EANs are logged to `used-eans.json`

8. **Excel output** — Uses `xlsx-populate` to read the template and write data starting at row 3, preserving all formatting (colors, fonts, validations) in rows 1–2. Dropdown columns only receive values from the allowed ReferenceData lists.

## Project Structure
```
src/
  config.js       — Configuration constants & utility functions
  classifiers.js  — Brand rules, lifecycle/size/type detection
  scraper.js      — Playwright: sitemap, product & variant extraction
  excel.js        — EAN pool, column loading, row mapping, Excel writing
  index.js        — Main orchestrator with retry logic
```

## Assumptions
- The site is **Wix-based**; product data comes from JSON-LD schema and `data-hook` elements.
- Variant data is extracted from Wix's custom dropdown and color picker UI components. Products without visible option selectors are treated as single-variant.
- `product-id-type` is set to `SKU` (allowed values: SHOP_SKU, SKU, ean).
- `basePriceUnit` defaults to `יחידות`.
- **Original template is never modified** — output goes to a separate file (`speadsheet-filled.xlsx`).
