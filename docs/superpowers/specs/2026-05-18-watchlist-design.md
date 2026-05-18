# Watchlist Feature Design

**Date:** 2026-05-18
**Status:** Approved

## Overview

Allow users to define a personal stock watchlist in a separate `watchlist.json` file. Watched symbols are fetched via Yahoo Finance (no API key required), displayed in the dashboard, and included in the LLM context for AI-generated trade ideas.

## Components

### 1. `watchlist.json` (new, project root)

User-editable file defining the watchlist. Loaded at runtime — no restart needed to pick up changes between sweep cycles.

```json
{
  "watchlist": [
    { "symbol": "AAPL", "name": "Apple" },
    { "symbol": "NVDA", "name": "Nvidia" },
    { "symbol": "TSLA", "name": "Tesla" }
  ]
}
```

- `symbol`: any Yahoo Finance-compatible symbol (stocks, ETFs, crypto, futures)
- `name`: optional display name; falls back to Yahoo Finance's `shortName` if omitted
- File missing or empty → watchlist feature silently disabled, no errors

### 2. `apis/sources/yfinance.mjs` (modified)

- On `collect()`, read `watchlist.json` from project root using `fs.readFileSync`
- Fetch quotes for watchlist symbols via the existing `fetchQuote()` function, in parallel with existing symbols
- Return watchlist quotes in a new `watchlist` array on the result object, same shape as `indexes`/`crypto` arrays: `{ symbol, name, price, change, changePct, history }`
- Watchlist symbols that fail to fetch are excluded silently (same error handling as existing symbols)

### 3. `dashboard/inject.mjs` (modified)

- Read `yfData.watchlist` and map it into `markets.watchlist` using the same projection as `markets.indexes`
- Pass `markets.watchlist` through in the `V2` object

### 4. `dashboard/public/jarvis.html` (modified)

- In `renderLower()`, build `watchlistCards` from `mkt.watchlist` using the existing `mktCard()` helper
- Each card: name, price, up/down arrow + % change (green/red), 5-day sparkline via `mkSparkSvg()`
- Render as a new "WATCHLIST" subsection in the Macro + Markets panel, between CRYPTO and ENERGY + METALS + MACRO
- Section hidden if `mkt.watchlist` is empty or absent

### 5. `lib/llm/ideas.mjs` (modified)

- In `compactSweepForLLM()`, after the METALS section, add a WATCHLIST section if `data.markets?.watchlist?.length`
- Format: `WATCHLIST: AAPL=$213.50 (+1.2%), NVDA=$875.00 (-0.8%)`
- Capped to first 10 symbols to stay within token budget

## Data Flow

```
watchlist.json
      ↓
apis/sources/yfinance.mjs (collect)
      ↓  [watchlist quotes array]
dashboard/inject.mjs
      ↓  [markets.watchlist]
      ├── jarvis.html renderLower() → WATCHLIST section in Macro + Markets panel
      └── lib/llm/ideas.mjs compactSweepForLLM() → WATCHLIST: line in LLM context
```

## Error Handling

- `watchlist.json` missing → skip silently, no watchlist rendered
- `watchlist.json` malformed JSON → log warning, skip silently
- Individual symbol fetch fails → exclude that symbol, others still shown
- Empty watchlist array → WATCHLIST section not rendered in dashboard or LLM context

## Out of Scope

- Price alerts / notifications for watchlist stocks
- Editing the watchlist from the dashboard UI
- Historical data beyond the 5-day window Yahoo Finance already returns
