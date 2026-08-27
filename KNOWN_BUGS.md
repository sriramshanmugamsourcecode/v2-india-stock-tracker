# Known Bugs — v1 (India Stock Tracker)

Compiled 2026-08-27 from a full systematic read-through of `index.html` (4,122 lines), cross-checked against the actual live Supabase schema. Nothing in this list has been fixed yet — logged here to fix on a future pass, per the project's no-logic-changes-without-sign-off rule.

## High priority — real, user-facing breakage

### 1. Manual "Add Trade" likely fails outright
- **Where**: `submitManualTrade()`, inserts `action: manualAction` where `manualAction` is lowercase `'buy'`/`'sell'` (set by the BUY/SELL toggle buttons via `setManualAction()`), with no `.toUpperCase()` anywhere in this path.
- **Why it breaks**: the live DB has a real, confirmed constraint — `trades_action_check CHECK (action = ANY (ARRAY['BUY','SELL']))`. A lowercase insert should violate it and error.
- **Fix shape**: uppercase `manualAction` before insert (one-line change), matching what the Kite-import path already does (`confirmImport()`'s `action: t.action.toUpperCase()`).
- **Verify by**: adding a manual trade via the "Add Trade" button and confirming whether it errors.

### 2. WoW tracker's "100 EMA Break" exit signal is permanently dead
- **Where**: `scrSaveCache()` writes `{ date: scrToday(), data }` to localStorage; `scrCacheIsFresh()` and `getEmaFromCache()` both destructure a `ts` field that was never written (`{ ts, data } = JSON.parse(raw)`).
- **Effect**: `Date.now() - undefined` = `NaN`; any comparison with `NaN` is `false`, so `scrCacheIsFresh()` always returns `false`. Downstream, `ema100` in `renderWow()` is always `null` (`cacheOk ? getEmaFromCache(...) : null`), so the "🔴 EMA Break" exit signal in `getExitSignal()` can never fire. The "⚡ 100 EMA exit signals unavailable — run Screener first" banner shows permanently, even right after running the screener.
- **Fix shape**: either rename the field consistently (`ts` everywhere, written as `Date.now()`), or switch the freshness check to parse `date` as a calendar day instead of a timestamp.

### 3. Kite import preview shows wrong buy/sell counts and colors
- **Where**: `handleImportFile()` uppercases `action` at parse time (`.toUpperCase()`, so values are `'BUY'`/`'SELL'`), but `showImportPreview()` compares against lowercase `'buy'`/`'sell'` (the buy/sell count filters, and the row text color `t.action==='buy' ? green : red`).
- **Effect**: the preview's summary pills always show "0 buys, 0 sells" regardless of actual counts, and every row's action text renders in the red "sell" color even for BUY orders.
- **Important**: this is **display-only** — `confirmImport()` re-uppercases (`t.action.toUpperCase()`, a no-op here) before inserting, so the data actually saved to the database is correct.
- **Fix shape**: compare against `'BUY'`/`'SELL'` (uppercase) in `showImportPreview()` instead.

## Low priority — cosmetic / dead code, no functional impact

### 4. `--font-sans` CSS variable is used but never defined
- **Where**: `.mt-input` (manual trade entry fields) and `.scr-row select` (Discovery filter dropdowns) both reference `var(--font-sans)`.
- **Effect**: only `--font-body`, `--font-display`, `--font-mono` exist in `:root` — `--font-sans` resolves to nothing, so these specific elements silently fall back to the browser's default font instead of DM Sans.
- **Likely fix**: change to `var(--font-body)`.

### 5. Duplicate key in `SCR_YF_OVERRIDES`
- **Where**: `'NSE:L&TFH'` is defined twice — once as `'LTFH.NS'`, again later as `'LTF.NS'`.
- **Effect**: none functionally (JS object literals silently let the later key win — `'LTF.NS'` is what's actually used), but confusing/redundant and the first entry is unreachable dead code.

### 6. Stale top-of-file doc comment
- **Where**: lines 8-34, a comment block from the Phase 2 era.
- **Effect**: describes "Phase 2 (current)", lists `wow_tracker` (not `wow_entries`) as the live table, and never mentions `universe` or `user_preferences` at all. No functional impact, but misleading to anyone (including a future AI session) who trusts it as current documentation.

## Resolved (kept here for history, not action items)

- **Discovery screener total failure (2026-08-27)**: caused by stripping exchange prefixes from `universe.ticker` for a Kite-import-matching fix, which broke `scrToYF()`'s assumption that `ticker` always contains `':'`. Rolled back (`universe.ticker` re-prefixed) same day. Root cause and full incident notes are in the conversation history / `project-overview.md`'s "Verified-against-real-source findings" section.
- **`universe.ticker` format**: confirmed via real DB export to be exchange-prefixed (`NSE:XXXX`/`BSE:XXXX`) as of the 2026-08-27 rollback — `project-overview.md` and `data-layer.spec.md` were corrected to reflect this (a prior version of those docs incorrectly claimed the opposite based on inference, never a real query).
