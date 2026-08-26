# Data Layer Spec

Covers: Supabase schema, RLS, market-data fetching (Yahoo Finance via proxy), ticker/symbol mapping, and the Kite tradebook import format. See [[project-overview]] for how this fits into the whole app.

## 1. Supabase

- Project URL: `https://zlpejsixpycewmfpmwrz.supabase.co`
- Client: `@supabase/supabase-js@2` loaded from CDN, initialized once as `db = createClient(SUPA_URL, SUPA_KEY)` (`index.html:1513-1514`).
- Auth: Supabase email/password only (`db.auth.signInWithPassword`). No signup flow in the app — accounts are provisioned manually. Single user today; RLS is already per-`user_id`, so a second account is additive, not a redesign (see Phase 5 backlog in [[project-overview]]).
- All data tables except `universe` are scoped by `user_id` and RLS-protected to `auth.uid() = user_id`.

### 1.1 `trades` — source of truth for all holdings/P&L

Every BUY/SELL entry, whether typed manually or imported from a Kite tradebook.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | PK, auto |
| `date` | date | trade date, `YYYY-MM-DD` |
| `stock_name` | text | display name, e.g. "Apar Industries" |
| `ticker` | text | e.g. `NSE:APARINDS` (**with** exchange prefix — this table's ticker format, unlike `universe`, see §3) |
| `action` | text | `'BUY'` or `'SELL'` |
| `qty` | integer | shares |
| `price` | numeric | price per share, ₹ |
| `brokerage` | numeric | default 0 |
| `exchange` | text | `'NSE'` or `'BSE'` |
| `user_id` | uuid | RLS key |
| `import_batch` | text | ISO timestamp shared by every row from one import; `NULL` for manual entries. This is the mechanism the 24-hour "Undo Last Import" feature deletes by (see [[portfolio]] §3) |
| `created_at` | timestamp | auto |

**⚠ Open question (see [[project-overview]] Open Questions #1 and #2)**: `PROJECT_CONTEXT.md` claims a check constraint enforces `action` uppercase. The import path (`confirmImport`) uppercases explicitly; the manual-entry path (`submitManualTrade`) inserts `manualAction` as typed (`'buy'`/`'sell'`, lowercase) with no uppercase step. This spec section describes what the client code does, not a verified DB constraint — **do not treat the uppercase requirement as confirmed until checked against the live schema.**

RLS (per `PROJECT_CONTEXT.md`, not independently re-verified against live DB):
- SELECT/INSERT/UPDATE/DELETE all scoped to `auth.uid() = user_id`.
- DELETE policy was added later (27-May) — was missing initially, causing silent delete failures. Historical note only; current code assumes it's present (`deleteTrade`, `undoLastImport` both filter `.eq('user_id', user.id)` client-side in addition to relying on RLS).

### 1.2 `wow_entries` — week-over-week closing prices

One row per (ticker, week) — a manually-triggered weekly price snapshot for every currently-held stock.

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | PK, auto |
| `week_date` | date | the date the snapshot was fetched for (defaults to most recent Friday, but user-editable) |
| `ticker` | text | matches `trades.ticker` format (with exchange prefix) |
| `close_price` | numeric | ₹, fetched from Yahoo Finance for that specific date |
| `user_id` | uuid | RLS key |
| `created_at` | timestamp | auto (assumed — not read by client code) |

Retention: capped at the most recent **12 weeks** per user (`WOW_DISPLAY_WEEKS = 12`). `cleanupOldWowEntries()` runs after every `fetchAndSaveWow()` and hard-deletes anything older than the 12 most recent distinct `week_date`s for that user. There is no user-facing way to keep more than 12 weeks of history.

Re-fetching for a `week_date` that already has rows deletes the old rows for that date first (full overwrite, not merge).

### 1.3 `watchlist`

Stocks added from the Discovery screener's "+ Watch" button, or (per the original Phase-4 plan) meant to carry entry/exit conditions — that half of the feature has no UI yet (Phase 5 backlog).

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | PK, auto |
| `ticker` | text | |
| `stock_name` | text | |
| `entry_condition` | text | written by schema design, never populated by any current code path |
| `exit_condition` | text | same — unused today |
| `exchange` | text | |
| `user_id` | uuid | RLS key |

Unique constraint on `(ticker, user_id)` — inferred from the app's `upsert(..., { onConflict: 'ticker,user_id' })` call in `scrAddWatch()`; not confirmed against a DDL dump.

### 1.4 `user_preferences`

Single-row-per-user settings table. Currently holds exactly one setting.

| Column | Type | Notes |
|---|---|---|
| `user_id` | uuid | PK |
| `pnl_floor` | integer | default 40 — the P&L% floor used by the WoW pyramiding/exit logic (see [[wow-tracker]]) |
| `updated_at` | timestamp | set on every `savePnlFloor()` call |

### 1.5 `universe` — the stock list Discovery screens against

The only table that is **not** per-user. Same list for every account.

```sql
CREATE TABLE universe (
  id         uuid DEFAULT gen_random_uuid() PRIMARY KEY,
  ticker     text NOT NULL UNIQUE,
  stock_name text NOT NULL,
  exchange   text NOT NULL DEFAULT 'NSE' CHECK (exchange IN ('NSE','BSE')),
  active     boolean DEFAULT true,
  created_at timestamp DEFAULT now()
);

ALTER TABLE universe ENABLE ROW LEVEL SECURITY;

CREATE POLICY "authenticated users read universe"
  ON universe FOR SELECT
  USING (auth.role() = 'authenticated');
-- No insert/update policy for regular users — maintained via Supabase table editor only.

CREATE INDEX idx_universe_ticker ON universe(ticker);
CREATE INDEX idx_universe_active ON universe(active);
```
(Verbatim from `Starting Point/nifty500_universe.sql`, dated 16-May.)

**Ticker format discrepancy, resolved**: this seed script's `INSERT` statements use exchange-prefixed tickers (`'NSE:POLYCAB'`), but the more recent `PROJECT_CONTEXT.md` (27-May) and the app's own defensive parsing (`ticker.includes(':') ? ticker.split(':')[1] : ticker`, used in `filterStockSearch`, `handleImportFile`'s universe map, `filterImportPicker`) both point to the *live* table storing un-prefixed tickers with `exchange` as a separate column. Treat **un-prefixed** as the current, correct format for `universe.ticker` — the seed SQL is a historical artifact from before that reformat, and the `.split(':')` defensiveness in the code is legacy safety, not evidence the format is still mixed.

Only 41 tickers are confirmed populated as of the last narrative snapshot (the current holdings); `PROJECT_CONTEXT.md` claims 555 (Nifty 500) total rows, matching the seed file's intent. Not independently re-verified against the live table (see [[project-overview]] Open Question #2) — an unauthenticated read returns zero rows by RLS design, so this needs an authenticated check to confirm the actual row count.

`active` flag: every query filters `.eq('active', true)` — inactive rows (delisted, etc.) are presumably kept for history but excluded from screening/search. No UI exists to toggle this (Phase 5 backlog: Universe management UI).

## 2. Market data — Yahoo Finance via Cloudflare Worker proxy

Yahoo Finance has no official public API and blocks CORS from browsers, so all price fetches go through a Cloudflare Worker that just forwards the request and adds CORS headers:

```
PROXY = 'https://little-bird-6066.sriram-shanmugam.workers.dev/?url='
YF_BASE = 'https://query1.finance.yahoo.com/v8/finance/chart/'
```

Every fetch is `PROXY + encodeURIComponent(YF_BASE + symbol + '?...')`.

Three distinct fetch shapes are used:

1. **Live price** (`fetchPrice`, portfolio tab) — `?interval=1d&range=1d`, reads `chart.result[0].meta.regularMarketPrice`. Batched 8 at a time with a 300ms gap between batches to avoid rate-limiting.
2. **Historical price for one date** (`fetchHistoricalPrice`, WoW tracker) — `?interval=1d&period1=<date-1day>&period2=<date+1day>`, reads the last non-null value in `chart.result[0].indicators.quote[0].close`. Same 8-per-batch/300ms pattern.
3. **1-year daily series** (`scrFetch1Y`, Discovery screener) — `?interval=1d&period1=<~380 days ago>&period2=now`, reads the full `close[]`/`volume[]`/`timestamp[]` arrays. Batched 6 at a time with a 500ms gap, plus retry-with-backoff (`scrFetchWithRetry`: up to 2 attempts, 1s then 2s backoff) and a 20-second per-request timeout via `AbortController`.

A separate endpoint, `v10/finance/quoteSummary`, is used once per stock in the screener for fundamentals (`scrFetchFund`): P/E (`summaryDetail.trailingPE`), EPS (`defaultKeyStatistics.trailingEps`), ROE and Debt/Equity and revenue growth (all from `financialData`). Missing fields resolve to `null`, not 0 — downstream filter logic treats `null` as "no data" (advisory-mode pass, not fail — see [[discovery-screener]] §3).

No rate-limit or API-key handling exists beyond the batching/backoff above, because this is an unofficial, undocumented endpoint — there is no published rate limit to code against. This is a known fragility, not a gap to fix casually.

## 3. Ticker format — two conventions coexist, by design

- **`universe.ticker`**: un-prefixed (`'ABB'`), exchange in a separate `exchange` column. This is what stock-search dropdowns match against.
- **`trades.ticker`, `wow_entries.ticker`, `watchlist.ticker`**: exchange-prefixed (`'NSE:ABB'`). This is what the portfolio/WoW/screener calculations key on internally.

Any code that needs to go from one to the other does it explicitly (e.g. `filterStockSearch` strips the prefix to search against `universe`, `selectManualStock` re-attaches the exchange from the matched `universe` row). There is no shared conversion utility — each call site repeats `ticker.includes(':') ? ticker.split(':')[1] : ticker` or the reverse. This duplication is existing debt, not a design decision to preserve.

### 3.1 Yahoo Finance symbol overrides

Two **separate, independently-maintained** override maps exist because Yahoo's own ticker often differs from the NSE/BSE ticker (renames, delistings, BSE-only listings, symbols containing `&`):

- `YF_OVERRIDES` (`index.html:1509`) — used only by the Portfolio/WoW live-price paths. Currently one entry (`ARTSON-X`).
- `SCR_YF_OVERRIDES` (`index.html:2764`) — used only by the Discovery screener. ~35 entries, covering renames (`TATAMOTORS`→`TMPV.NS` post-demerger, `ZOMATO`→`ETERNAL.NS`), symbol quirks (`AMARAJABAT`→`ARE&M.NS`), and explicit `null` for delisted/unmatched tickers (skipped entirely rather than fetched).

These two maps can and do diverge — a ticker fixed in one context isn't automatically fixed in the other. New mismatches surface as fetch failures (screener's exclusion log tags them `layer: 'fetch'`) and are fixed by manually adding an entry. There is no automated Yahoo-symbol-resolution step.

## 4. Kite tradebook import

Source: a `.xlsx` file exported from Zerodha Console (console.zerodha.com), not the Kite Connect API — this is a **manual, on-demand upload**, not a live sync (Kite Connect's ₹2000/month API tier was considered and passed over for this; see `PROJECT_CONTEXT.md`'s "Kite Integration Plan").

Parse steps (`handleImportFile`, using SheetJS):
1. Locate the header row by scanning for one containing both `'Symbol'` and `'Trade Date'` (Kite's export format isn't fixed-row).
2. Read columns: `Symbol`, `Trade Date`, `Exchange`, `Trade Type`, `Quantity`, `Price`, `Order ID`. (`Auction` column is read into `colIdx` but never used downstream.)
3. **Group by `orderId|symbol|action|date`** — Kite can split one order into multiple fill rows; these are summed (`totalQty`) and averaged by value-weighted price (`totalValue / totalQty`).
4. Match each grouped row's symbol against the `universe` table (case-insensitive, un-prefixed) to get a display name and confirm it's a known ticker; unmatched rows are flagged `notInUniverse: true` and shown in the preview with an inline stock picker so the user can manually resolve them before confirming.
5. On confirm: `brokerage` defaults to `0` for every imported row (Kite's export doesn't carry brokerage per fill in this flow), `action` is uppercased, and every row in the batch gets the same `import_batch` value (`new Date().toISOString()` at confirm time).

Undo: the batch ID + row count + a 24-hour expiry are cached in `localStorage` (`last_import_batch`) right after a successful import. "Undo Last Import" deletes every `trades` row matching that `import_batch` + `user_id`, then clears the `localStorage` key **only after** the delete succeeds. Past the 24-hour window, the button won't reappear on reload and the only recourse is deleting trades individually from the Trade Log.
