# Known Bugs — v1 (India Stock Tracker)

Compiled 2026-08-27 from a full systematic read-through of `index.html` (4,122 lines), cross-checked against the actual live Supabase schema.

**Status: all six items resolved in v2 `APP_VERSION = 'v4.20260827.7'` on 2026-08-27.** Details of each fix are kept below for history.

## Fixed — v4.20260827.7 (2026-08-27)

### 1. Manual "Add Trade" failed outright — FIXED
- **Where**: `submitManualTrade()` inserted `action: manualAction` where `manualAction` is lowercase `'buy'`/`'sell'` (set by the BUY/SELL toggle via `setManualAction()`), with no `.toUpperCase()` in this path.
- **Why it broke**: the live DB has a real, confirmed constraint — `trades_action_check CHECK (action = ANY (ARRAY['BUY','SELL']))`. A lowercase insert violates it and errors.
- **Fix applied**: insert `action: manualAction.toUpperCase()`, matching what the Kite-import path already does (`confirmImport()`'s `action: t.action.toUpperCase()`).
- **Verify by**: adding a manual trade via the "Add Trade" button and confirming it saves without error.

### 2. WoW tracker's "100 EMA Break" exit signal was permanently dead — FIXED
- **Where**: `scrSaveCache()` wrote `{ date: scrToday(), data }` to localStorage; `scrCacheIsFresh()` and `getEmaFromCache()` both destructure a `ts` field that was never written (`{ ts, data } = JSON.parse(raw)`).
- **Effect**: `Date.now() - undefined` = `NaN`; any comparison with `NaN` is `false`, so `scrCacheIsFresh()` always returned `false`. Downstream, `ema100` in `renderWow()` was always `null` (`cacheOk ? getEmaFromCache(...) : null`), so the "🔴 EMA Break" exit signal in `getExitSignal()` could never fire and the "⚡ 100 EMA exit signals unavailable — run Screener first" banner showed permanently.
- **Fix applied**: `scrSaveCache()` now also writes `ts: Date.now()` into the cached object, so the existing 3-day freshness math in `scrCacheIsFresh()` / `getEmaFromCache()` works. `scrLoadCache()` still reads `date` and is unaffected.

### 3. Kite import preview showed wrong buy/sell counts and colors — FIXED
- **Where**: `handleImportFile()` uppercases `action` at parse time (`.toUpperCase()`, so values are `'BUY'`/`'SELL'`), but `showImportPreview()` compared against lowercase `'buy'`/`'sell'` (the buy/sell count filters, and the row text color `t.action==='buy' ? green : red`).
- **Effect**: the preview's summary pills always showed "0 buys, 0 sells" regardless of actual counts, and every row's action text rendered in the red "sell" color even for BUY orders.
- **Note**: this was **display-only** — `confirmImport()` re-uppercases (`t.action.toUpperCase()`) before inserting, so data saved to the DB was always correct.
- **Fix applied**: `showImportPreview()` now compares against `'BUY'`/`'SELL'` (uppercase) in both the count filters and the row color expression.

### 4. `--font-sans` CSS variable was used but never defined — FIXED
- **Where**: `.mt-input` (manual trade entry fields) and `.scr-row select` (Discovery filter dropdowns) both referenced `var(--font-sans)`.
- **Effect**: only `--font-body`, `--font-display`, `--font-mono` exist in `:root` — `--font-sans` resolved to nothing, so these elements silently fell back to the browser default font instead of DM Sans.
- **Fix applied**: both now use `var(--font-body)`.

### 5. Duplicate key in `SCR_YF_OVERRIDES` — ALREADY FIXED IN v2
- **Original (v1)**: `'NSE:L&TFH'` was defined twice — once as `'LTFH.NS'`, again later as `'LTF.NS'`.
- **v2 state**: only one entry remains — `'L&TFH': 'LTF.NS'` (fixed earlier in the v2 ticker-format cleanup, commit `5f7ce99`). No further change needed on this pass; recorded here for completeness.

### 6. Stale top-of-file doc comment — FIXED
- **Where**: the header comment block, from the Phase 2 era.
- **Effect**: described "Phase 2 (current)", listed `wow_tracker` (not `wow_entries`) as the live table, and never mentioned `universe` or `user_preferences`. No functional impact, but misleading to anyone (including a future AI session) trusting it as current documentation.
- **Fix applied**: rewritten to reflect Phase 4 (current), the v2 repo/backend, `Last updated: 2026-08-27`, and the real table set (`trades`, `wow_entries`, `universe`, `user_preferences`, `watchlist`).

## Resolved earlier (kept here for history, not action items)

- **Discovery screener total failure (2026-08-27)**: caused by stripping exchange prefixes from `universe.ticker` for a Kite-import-matching fix, which broke `scrToYF()`'s assumption that `ticker` always contains `':'`. Rolled back (`universe.ticker` re-prefixed) same day. Root cause and full incident notes are in the conversation history / `project-overview.md`'s "Verified-against-real-source findings" section.
- **`universe.ticker` format**: confirmed via real DB export to be exchange-prefixed (`NSE:XXXX`/`BSE:XXXX`) as of the 2026-08-27 rollback — `project-overview.md` and `data-layer.spec.md` were corrected to reflect this (a prior version of those docs incorrectly claimed the opposite based on inference, never a real query).
