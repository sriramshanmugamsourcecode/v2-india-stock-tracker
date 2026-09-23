# Business Logic — India Stock Tracker v2

*Written fresh against `APP_VERSION = 'v4.20260912.35'`, verified directly against `index.html` rather than carried from the older spec files in this folder; updated for the Exit Signals tab and Pyramiding as of `v4.20260921.52`. Where a claim below corrects something those older specs got wrong (they were frozen months ago and missed at least one real behavior change), it's called out explicitly — see "Corrections vs. the old specs" at the end.*

This document is organized by tab, in the order they appear in the nav, followed by cross-cutting conventions that apply everywhere.

---

## 1. Portfolio

### 1.1 Cost basis — `calcPortfolio()`

Nothing is stored as "current holdings" anywhere — every render recomputes from the full `trades` table, grouped by bare ticker (exchange prefix stripped, so `NSE:ABB` and bare `ABB` merge into one holding — see §8.1).

The cost basis is a **chronological moving average**, not a simple lifetime average of all buys:

- Trades for a stock are sorted chronologically (by `date`, then `created_at`, then `id` as tiebreakers).
- A running `qty`/`pool` (total cost) is walked forward trade by trade.
- A **BUY** adds to both: `qty += t.qty`, `pool += t.qty*price + brokerage`.
- A **SELL** is booked against the running average **at that moment**: `avgNow = pool/qty`, realised P&L on the sold units = `(sell price − avgNow) × units − brokerage`, and the pool is reduced by `units × avgNow` (not by the units' original purchase cost) before quantity is reduced.
- **A full exit resets the basis**: if quantity reaches ~0, both `qty` and `pool` are zeroed out, so a later re-buy of the same stock starts a clean average rather than inheriting anything from the closed-out position.

This means a partial sell **does** change the story for what's left, in the sense that the remaining shares' average cost is whatever the running pool says at that point — it does not retroactively change, but it's also not a "some buys are more special than others" FIFO/LIFO lot system. There is no per-lot tracking anywhere in this app.

**Per-holding figures** (only for `netQty > 0`): `invested = netQty × avgBuy`, `currentValue = livePrice × netQty` (null until live price arrives), `unrealisedPnl = currentValue − invested`, `unrealisedPct = unrealisedPnl / invested × 100`.

**Portfolio totals**: sums of the above across active holdings, plus `realisedPnl` and `sellProceeds` summed across every stock's chronological walk (so a fully-closed position still contributes its realised P&L to the total even though it no longer appears as a holding).

### 1.2 XIRR — `xirr()`, Newton-Raphson

Cashflow convention: BUY = `-(qty×price + brokerage)`, SELL = `qty×price − brokerage`. Any stock still held gets one extra synthetic positive cashflow **dated today**, valued at `livePrice × netQty` — this is what lets an *unrealised* gain be expressed as a rate.

Solved by Newton-Raphson (`npv`/`dnpv`), starting guess 10%, up to 100 iterations, converging at <1e-7 change between iterations. Returns `null` (shown as `—`) if fewer than 2 cashflows exist, the derivative underflows before convergence, or the rate diverges below −100%.

**Clamped at +5,000%/year** (added this session — see `docs/../feature_rebalance.md`-adjacent memory, and the live incident that motivated it): a large cashflow very close in time to "today" — a recent top-up, or a backdated correction trade — can make the solved rate explode, either through genuine Newton-Raphson divergence (near-cancelling cashflows flatten the derivative near zero) or simply because annualising a multi-day return is inherently extreme. Rather than null it out (which would wrongly zero out that factor's contribution wherever XIRR feeds the Rebalance composite score), the result is capped at 5,000% and displayed as **"5,000%+ (maxed)"** (`fmtXirr()`) so it's visibly a ceiling, not a precise measurement.

**A high XIRR next to a negative current Returns % is not a contradiction** — it happens whenever an earlier profitable sell (which resets the cost pool per §1.1) is still counted in the full cashflow history even though the shares currently held are a separately-rebuilt, currently-underwater position. Verified concretely this session on a real holding (SUZLON: a round-trip sale returning +25% in 58 days legitimately annualizes past +250%, while the small recent re-buy sits at a current −4% — both numbers are correct, they answer different questions).

Both a **per-stock XIRR** and an independent **portfolio-wide XIRR** (one combined cashflow series across every trade ever made, not an average of the per-stock figures) are computed. The portfolio-wide figure is computed but **no longer displayed anywhere** — its stat card was removed from the Portfolio tab; per-stock XIRR is now shown instead, on the Rebalance tab's Momentum holdings table.

### 1.3 Holding tags (Long term / Momentum)

A two-way toggle per holding, Portfolio tab only. Backed by `holding_tags` — **only Momentum rows are ever written**; untagged = Long term by default, so toggling back to Long term deletes the row rather than writing an explicit `'long_term'` value. Drives which holdings appear on the Rebalance tab (§6).

---

## 2. Trade Log

Three ways a row enters `trades`:

1. **Manual entry** (`submitManualTrade`) — BUY/SELL toggle, autocomplete stock search against `universe`, date (defaults today), exchange auto-filled from the matched universe row, qty/price/brokerage (brokerage defaults ₹0), live running total. `action` is explicitly `.toUpperCase()`'d before insert (`manualAction.toUpperCase()`) — **verified this session**, since an older spec had flagged this as a possible unresolved casing bug; it is not one in current code.
2. **Kite import** (`confirmImport`) — bulk insert from a parsed `.xlsx` tradebook, one shared `import_batch` ISO-timestamp per batch, action uppercased, brokerage forced to ₹0. Header row auto-detected by locating `Symbol` + `Trade Date` columns.
3. Nothing else — no generic CSV importer.

**Delete**: any single row, hard delete, scoped to `id` + `user_id` — no soft-delete or audit trail. Deleting a trade retroactively changes every downstream calculation for that stock (cost basis, XIRR, WoW P&L history) since nothing is cached.

**Undo Last Import**: a 24-hour window tracked via `localStorage` (`last_import_batch_v2`), deletes by `import_batch` + `user_id`. Re-appears correctly across page reloads (`showUndoButton()` runs on every `loadAll()`).

**Exchange correction**: a holding's exchange can be edited post-hoc directly from the Portfolio Holdings table (`updateExchange`) — this is a live write to every `trades` row for that ticker, and matters because exchange determines the `.NS`/`.BO` suffix used for Yahoo Finance lookups (§8.2).

---

## 3. WoW (Week-over-Week) Tracker

Manually triggered, nothing automatic. Picking a date (defaults to the most recent Friday) and clicking "Fetch & Save Week":
1. Takes the *current* active holdings from `calcPortfolio()`.
2. Fetches each one's Yahoo Finance closing price for that date.
3. Deletes-then-inserts `wow_entries` rows for that date (idempotent re-run).
4. Trims to the most recent 12 weeks.

A stock bought after an older week's snapshot simply has no entry for that week (shown as `—`), never a backfilled estimate.

### 3.1 Per-holding, per-week figures

- `pnlPct = (weekPrice − avgBuy) / avgBuy × 100` — the week's **snapshot** price, not live price, so this can legitimately differ from the Portfolio tab's current unrealised P&L if the price has moved since the last capture.
- `wowChgPct` — % change vs. the immediately preceding tracked week (`null` if no prior week).
- `vsFloor = pnlPct − pnlFloor` (user-configurable, default 40%, `user_preferences.pnl_floor`).
- **Gain Lost** — `gainLost = pnlPct − peakPnlPct`, where `peakPnlPct` is derived from the highest weekly close price seen up to and including the current week. Always ≤ 0; 0 means the current week is a fresh high. Displayed as "▲ at peak" rather than a percentage when it rounds to ~0. This is the metric reused, unmodified, by the Rebalance tab (§6.2).
- A rolling 3-week arrow trend (`wowStreakArrows`) — newest→oldest, padded with `—` for missing history. Also reused by Rebalance.

### 3.2 Pyramiding ("Buy Signal") — `getPyramidSignal`

Four mutually-exclusive conditions, not priority-ordered:

| Condition | Signal |
|---|---|
| P&L ≥ floor and price up/flat this week | ✅ Strong Add |
| P&L ≥ floor and price down this week | 🟡 Hold |
| P&L within 5 points below floor | 🟡 Watch |
| More than 5 points below floor | ❌ Don't Add |

A brand-new holding with no prior week (`wowChgPct == null`) counts as "up" — defaults to Strong Add if above floor, not Hold.

### 3.3 Exit Signal — `getExitSignal`, priority-ordered (first match wins)

1. **🔴 Hard Exit** — `pnlPct < floor − 10`.
2. **🔴 Trend Break** — 3 consecutive down weeks.
3. **🔴 EMA Break** — price below the 100-day EMA, sourced from the **Discovery screener's own cache**, only trusted if that cache is ≤3 days old. If stale/absent, this check is silently skipped (not treated as pass) and a banner prompts running the screener. **Real cross-feature dependency**: WoW's exit-signal accuracy depends on how recently Discovery was run.
4. **🟡 Near Floor** — between hard floor and floor.
5. **🟡 Sharp Drop** — single-week drop worse than −5%, independent of overall P&L.
6. **✅ Holding** — none of the above.

This exact function is also called **completely unchanged** by the Exit Signals tab (§9) — same inputs WoW itself would use, so its Exit Signal column always matches what WoW shows for that stock.

### 3.4 Buy-Qty Calculator

Shown only on Strong Add rows. Solves for the additional quantity that brings the post-buy blended average cost exactly to `weekPrice / (1 + floor/100)` — i.e. exactly at the floor after buying. A sizing suggestion only; does not create a trade.

---

## 4. Discovery Screener

### 4.1 Indicators (`scrCalc`, from a 1-year daily close+volume series, all computed client-side)

`hi52` (252-trading-day max close), `pct52w` (gap below it), `score52w` (`price/hi52`, ranking only), `dma50/100/200`, `ema100/200`, plain `ret1m/3m/6m/1y` (display), **skip-month** `ret1mSkip/3mSkip/6mSkip` (ranking — same window but ending 21 trading days ago, the Jegadeesh-Titman convention avoiding short-term reversal contamination — see the paper reference discussion earlier in this project), `volatility` (annualised stdev of daily log returns, 3M), `adjRet = ret6mSkip / volatility`, `adtv` (avg daily traded value, ₹Cr, liquidity gate), fundamentals straight from Yahoo (`pe, eps, roe, de, rev`).

### 4.2 Composite ranking (`scrRank`) — runs on the whole fetched universe, before any filter

Each of `score52w, ret6mSkip, ret3mSkip, ret1mSkip, adjRet` is rank-ordered across all stocks (rank 1 = best). A stock missing a given metric gets that metric's **median rank**, not a worst-case penalty. Weighted sum: `compositeRank = rank52w×2 + rank6m×2 + rank3m×2 + rank1m×1 + rankAdjRet×1` — lower is stronger. The displayed "Rank #N" is this stock's ordinal position **universe-wide**, computed before filters, so it doesn't change based on the active filter view. This exact formula is what backtest Test #1 validated (§4.4) — changing the weights invalidates that backtest until re-run. This is the same formula reused (unchanged) by Rank History's monthly `rank_history` captures.

### 4.3 Filters (display-only — never affect rank/position)

**Liquidity** gate first (ADTV threshold). **Momentum** (all four must pass): 52W-gap threshold, a moving-average filter (50/100/200 DMA, 100EMA, or "brutal" = price>DMA50>DMA100>DMA200 stacked), an optional EMA golden-cross toggle, and a return-over-period threshold (uses the plain, non-skip return). **Fundamentals** are a hard gate only in Strict mode; in Advisory mode (default) they're colour-coded but non-disqualifying, and a `null` fundamental never counts as a fail either way.

Signal classification for survivors: count of true fundamentals — ≥4 → "Strong pick," ≥2 → "Watchlist," else "Mom. only."

### 4.4 Backtesting (Colab notebooks, `Starting Point/Backtest/`)

Test #1 (pure composite rank, no filters, 528 stocks, 11.2 years): CAGR 31.6% vs 11.4% benchmark, Sharpe 1.18, max drawdown −36.9%, 62.5% average monthly turnover. Test #2 (filter impact): a volatility cap and an "up-days %" filter were both **catastrophic** and removed from the product entirely; 100-day EMA filter kept (slight positive); 52W-proximity and plain-return filters kept for usability, not backtested edge. **Only the plain composite formula and the two removed filters are backtest-validated** — the three presets, the fundamentals gate, and WoW's exit-signal rules are designed-by-judgment, not backtested. See the separate backlog item on validating the newer Rebalance composite weights the same way, in memory (`backlog_momentum_indicators_validation.md`).

### 4.5 Streak tracker

Independent of the ranking display: once per run, the top-30-by-pure-compositeRank (ADTV≥₹5Cr only, deliberately ignoring the user's active filter so a filter change doesn't reset streaks) is snapshotted to `localStorage`, keyed by calendar month, 24-month retention. A stock's streak = consecutive months present, rendered 🔥 (3+) / 🔥🔥 (6+). Entirely browser-local, not in Supabase.

### 4.6 Invest / Watch / Don't invest tag

Each Discovery row carries the same 3-button tag widget Rank History's Trend view uses (`rank_status` table, `go`/`wait`/`dont` values) — tagging a stock in one tab shows the same tag in the other, since both read/write the identical per-user `(ticker, status)` row. This **replaced** an earlier "+ Watch" button that upserted into a `watchlist` table with no unique constraint on `(ticker, user_id)` — the upsert's `onConflict` clause had no matching constraint to satisfy, so it errored on every click. Rather than add the missing constraint, the button and its `watchlist` write path were retired entirely in favour of reusing the already-working `rank_status` mechanism. `watchlist` still exists in the DB (nothing dropped it) but nothing in the app writes to it any more.

### 4.7 Pattern Lookup panel

A stocks-only input inside the Discovery tab: enter tickers, get a stacked card per stock with rule-based signal chips derived from the same indicators above, plus a **copy-to-clipboard prompt** formatted for pasting into a separate Claude conversation for a qualitative read. Deliberately **does not call any AI API from inside the app** — no Worker, no API cost, no automated response — the user explicitly chose the copy-prompt-only design over an earlier in-app AI-read prototype.

---

## 5. Rank History

Monthly composite-rank snapshots, stored in the **shared** (not per-user) `rank_history` table, captured manually via a "Capture" button using the exact same composite formula as Discovery (§4.2) — top ~50 stocks, columns `month, rank, ticker, stock_name, composite, score52w, ret6m_skip, ret3m_skip, ret1m_skip, adj_ret, universe_size, source, captured_at`.

Three views: **Grid** (spreadsheet-style month×rank table, §10.4 on its mobile-width fix), **Trajectory** (a stock's rank over time), **Trend** (sortable by streak-of-months-in-top-50, with a "Not in Top 50" box for recent dropouts and a "Holdings on top" toggle). Per-user **status tags** — labelled **Invest / Watch / Don't invest** (`rank_status` table, values `go`/`wait`/`dont`) — let the user annotate a ticker's actionability independent of its rank, and are **shared with Discovery** (§4.6): the same tag shows in both tabs. This table's migration was found never to have actually been run when this documentation set was first verified against the live database (2026-09-12), meaning the feature had silently never persisted a tag since it shipped; fixed the same day (see [`03-ai-rebuild-spec.md`](03-ai-rebuild-spec.md) §3 for the exact migration).

---

## 6. Rebalance

### 6.1 Momentum capital

A user-set **cap** on sleeve size (not a weekly top-up amount) — "the most I'll ever have deployed in momentum, total." Persisted as an **append-only log** (`momentum_capital`: `user_id, amount, updated_at`) — the latest row is "current," the one before is "previous week," by design (no update, only insert). "% of cap" for a holding = `currentValue / latest cap amount × 100`; "headroom" = `cap − total deployed value`, can go negative (shown as "X over cap").

### 6.2 Momentum holdings table

Columns beyond the obvious (qty/avg cost/current/P&L): **Total cost** (`invested`), **Profit contrib %** (this stock's `unrealisedPnl` ÷ the whole momentum sleeve's total `unrealisedPnl` — *sleeve-scoped*, not whole-portfolio-scoped), **Returns %** (reuses `calcPortfolio()`'s `unrealisedPct`), **XIRR** (reuses `stockXirr`, §1.2, including the 5,000% clamp), **1W chg / 3W trend / Gain Lost** (all reused verbatim from WoW's own weekly data — §3.1 — nothing new fetched), **vs 52W high** (the one genuinely new calculation, via the Discovery screener's existing `scrFetch1Y`/`scrHigh52`/`scrToYF` plumbing so `SCR_YF_OVERRIDES` mismatches are already handled, cached client-side 24h in `localStorage['rb_52w_v1']`). A **sleeve XIRR** in the footer is a real combined-cashflow XIRR across every momentum holding's trades concatenated together — not an average of the per-stock XIRRs, which wouldn't be valid.

### 6.3 "Rebalance for this week" — the split calculator

Top-up only — **no sells**. Candidate pool = only currently Momentum-tagged holdings (not new Discovery candidates). Three modes, toggled by the user:

**Weighted** (default): each candidate gets a **composite score** —

```
composite = 0.40×XIRR_pts + 0.25×vs52W_pts + 0.25×trend_pts + 0.10×GainLost_pts
```

— where each `_pts` is that factor's value rank-normalised 0–100 within the candidate pool (tie-averaged), same convention as Discovery's own ranking. **3W trend's points** = ratio of positive-vs-negative WoW weeks among however many are actually available (up to the last 3), not a fixed denominator. **Missing-factor rule** (generalized from an explicit user instruction that originally covered only the 3-week-trend case): any factor that can't be computed for a candidate contributes a flat neutral 50 points **for that factor only**, rather than excluding the stock or nulling the whole composite — 50 pts × that factor's weight exactly equals "half that factor's weight," matching the original spec precisely (e.g. 50×0.25=12.5, the "12.5% default" the user asked for). Allocation = the deploy amount split proportional to composite score; a zero-composite stock gets **exactly ₹0, silently, with no visual flag** (explicit user call).

**Equal split**: every tagged holding gets an equal ₹ share regardless of score — **the one behavioral difference from Weighted**: a zero-composite stock still gets funded here.

**Side by side**: both of the above shown together, compact columns (target ₹ with weight % beside it, share count as its own column).

**Round-off sweep — two different algorithms, deliberately**: after rounding each candidate's target down to whole shares, the leftover cash is redeployed. In **Weighted** mode it's spent greedily on the single richest-composite affordable stock, repeatedly, before moving to the next rank — reinforcing "keep betting on the leader," and explicitly excluding zero-composite stocks from ever receiving swept money either. In **Equal** mode it's round-robined — at most one extra share per stock per pass, cheapest-first, cycling passes until nothing is affordable — specifically because the Weighted-style "exhaust the top pick first" approach would let whichever stock happens to be cheapest-per-share silently swallow most of the leftover and defeat the entire point of "equal."

Deliberately **not built**: any sell/rotation logic (checkbox-gated, tax-aware, opt-in — designed in conversation, not implemented) and any historical validation of the 40/25/25/10 weighting itself (separate backlog item, to avoid overfitting a scheme to this account's own tiny, weeks-old momentum basket).

### 6.4 Pyramiding

A staged-entry planner living inside the Rebalance tab (own sub-namespace, `.rb-pyr-*`), self-contained: its own `universe` fetch (type-ahead search over the whole active universe, not just tagged/held stocks) and its own live-price fetch (`scrFetch1Y`, same path the 52W-high in §6.2 already uses) — reads `calcPortfolio()` for a real existing position, nothing else shared.

Inputs: a stock (search), a **cushion target** (4/6/10/15/20/25%, default 6% — a **floor, not an exact target**: the guarantee is "never let a planned addition drop blended cushion below this," not "converge to exactly this"), an **amount to deploy** (default ₹3,00,000, explicitly **per stock**, not portfolio-wide — and **includes** any existing position's real cost, not new-money-on-top-of-it), a manually-typed **first tranche amount** (default ₹1,20,000, no formula — the cushion math only applies to tranches *after* some price gain already exists), and an optional **max tranche** cap (blank = uncapped) limiting every single tranche's size, including the first.

If already held, the plan starts from the real `avgBuy`/`netQty` (shown as a highlighted "Already owned" row) and stages further tranches from there. Tranches 2+ trigger at round **+5% price bands** from today's live price (actual tradeable GTT/limit levels) — at each, the tool adds the **maximum** shares the cushion formula allows (same formula as `calcBuyQty`, §3.4, generalised) without dropping blended cushion below the floor, capped by whichever binds first: remaining deployment budget, or the max-tranche cap. A tranche is skipped entirely (no row) if either cap allows zero shares at that price. Built from a clickable Artifact mockup reviewed and approved before any production code was written, per this project's standing "mock before building" workflow.

---

## 7. App Maintenance

Direct CRUD panels over the database, for fixing data drift without hand-writing SQL — three sub-tabs:

- **Universe** — search-to-load (never renders all ~720 rows as inputs at once), inline edit/add/soft-delete, batched "Save changes (N)" button. Ticker-field blur triggers a **live Yahoo verification** (tries the row's exchange first, then the other, honours any existing `SCR_YF_OVERRIDES` entry) — Save is blocked on any unverified new/edited ticker unless the user explicitly checks "Save anyway — needs a Yahoo override," which logs the ticker to a shared to-do list (`localStorage['mnt_override_todo_v1']`) for later.
- **Trade Logs** — same edit/add/delete/batched-save pattern over `trades`, plus autocomplete (sourced from `universe`, free-text not locked) and the same shared Yahoo-check/override-list machinery.
- **Portfolio** — **Backups** (its own always-visible card: downloads a versioned JSON snapshot of `trades`/`wow_entries`/`watchlist`/`user_preferences`/`universe` to **IndexedDB**, not localStorage, last 10 kept) and **Reset Portfolio** (a progressive-reveal wizard: auto-backup on start → upload a holdings-statement CSV → before/after comparison with hyphenated-ticker flagging (`-IV`/`-RR` style broker quirks, never auto-stripped) and an editable ticker preview → type-the-exact-trade-count confirm → commit, which deletes all the user's `trades` and inserts one clean BUY per reviewed row). Deliberately loses individual lot history on reset — an accepted trade-off for walking away from tangled data, not an oversight.

Both save flows are **best-effort**: every pending row is attempted independently regardless of earlier failures, reporting "Saved X of Y — Z failed: reason" rather than an all-or-nothing rollback; successful rows clear from local state, failed ones stay staged for retry.

Saving in either Universe or Trade Logs auto-invalidates the relevant caches elsewhere in the app (`scrUniverse`, the Trade Logs autocomplete list, and a full `loadAll()` re-render) so Portfolio/WoW/Rank History/Rebalance reflect the edit without a page reload.

---

## 9. Exit Signals

A dedicated tab (sits before Rebalance in the nav; Maintenance moved to the very end to make room) that consolidates every exit rule into one full-holdings table — read-only, changes nothing in WoW/Portfolio/Rank History. Defaults to a Momentum-only filter (toggle to All, reuses `holding_tags`).

Reuses `getExitSignal()` (§3.3) completely unchanged for its Exit Signal column. Layered on top, three new independent checks, each its own column (not folded into `getExitSignal()`'s single-winner label):

- **Gain Lost breach** — reuses WoW's own Gain Lost formula (§3.1: P&L points given back from a position's own weekly-tracked peak P&L%) against a user-configurable threshold — a dropdown (-5/-10/-15/-20/-25/-30%, default -20%) persisted to a new `user_preferences.gain_lost_exit_pct` column.
- **50-DMA break** — price below the 50-day SMA, read from the same Discovery-screener cache the 100-EMA check already uses, via `dma50` (already computed by `scrCalc`, just not previously read anywhere).
- **Rank Decay** — a holding's best-ever vs current composite rank from `rank_history`. Shown in amber while still inside the top 50 but sliding ("▼N from best #X"); flips red once outside the top 50 for **3+ consecutive captured months** (fixed, not user-tunable). Calendar gaps in capture history (months nobody ran Capture for) are detected and flagged separately, and **pause** rather than break or continue the streak count — a gap month counts neither for nor against a stock's "outside top 50" streak, since there's no data for it either way.

**A real pre-existing bug found and fixed while wiring the 50-DMA lookup**: `getEmaFromCache()` called `.find()` — an array-only method — on the Screener's localStorage cache, which is actually stored as a **plain object keyed by ticker** (`scrSaveCache()`), not an array. The call always silently threw and was swallowed by a `catch`, returning `null` every time. This means the existing 100-EMA-Break exit rule in WoW (§3.3, item 3) had likely never actually fired in production since it shipped. Fixed with `Object.values(data).find(...)`, which works regardless of the cache's actual shape.

---

## 10. Cross-cutting conventions

### 10.1 Ticker grouping — `bareSym()`

Every feature that aggregates by "stock" strips the exchange prefix (`NSE:`/`BSE:`) before grouping, so a stock imported once with a prefix and once without still merges into one holding. This does **not** strip suffixes (`-IV`, `-RR` broker quirks) — several real bugs this project fixed (PGINVIT, BIRET-RR, INDUSINVIT) were exactly this: a suffixed and a bare version of the same underlying instrument silently living as two separate "holdings" until manually reconciled and renamed in `trades`/`universe`.

### 10.2 Yahoo Finance symbol mapping — `SCR_YF_OVERRIDES` (screener/Rebalance) and `YF_OVERRIDES` (portfolio live-price fetch)

Two separate override maps exist for the same underlying problem (NSE ticker ≠ Yahoo's symbol for that instrument) because they evolved independently — `scrToYF()`/`SCR_YF_OVERRIDES` is the actively-maintained one (dozens of entries, each commented with the reason and confirmation status). A `null` entry means "confirmed dead on Yahoo, skip rather than guess." New mismatches surface as fetch failures and get added by hand after manual verification — guessing a replacement symbol without confirming caused at least one real mislabeling bug this project fixed (KTIL pointed at the wrong company entirely).

### 10.3 Isolated-feature convention

Every feature added since Rank History (Rank History, Pattern Lookup, App Maintenance, Rebalance, Exit Signals, and Pyramiding as a sub-namespace inside Rebalance) is a self-contained IIFE (or sub-section within one) with its own CSS class prefix (`.rh-*`, `.lk-*`, `.mnt-*`, `.rb-*`, `.ex-*`, `.rb-pyr-*`), touching the rest of the app only via: one nav `<button>`, one entry in `switchTab()`'s tab-list array, and one hook line (`if (tab === 'x' && window.xEnter) window.xEnter();`). Cross-feature data access happens through JS closures (all IIFEs are nested in the same outer `<script>` block, so they read outer-scope globals like `trades`, `wowEntries`, `holdingTags`, and call outer functions like `calcPortfolio()`/`loadAll()` directly) rather than any event bus or shared store.

### 10.4 Table width on mobile — `width:100%`, not `width:max-content; min-width:100%`

Every scrollable table (`.table-wrap`, Discovery's results table, Rank History's Grid and Trend) originally used `width:max-content; min-width:100%` — the theoretically-correct CSS for "fill the container, but grow wider and trigger horizontal scroll if content needs more." In practice this doesn't reliably fill the container for `<table>` elements specifically (a `<div>` with the same rule behaves correctly) — on mobile, tables built this way visibly stopped short of full width while everything else on the page reached the edge. Fixed by switching every affected table to plain `width:100%`; cells already had `white-space:nowrap`, so a table can still grow past 100% and trigger the wrapper's `overflow-x:auto` when content genuinely needs more room — the fix only removes the under-fill case, not the intended overflow behavior.

### 10.5 Versioning

Single source of truth: `const APP_VERSION` near the top of the script, format `v<phase>.<YYYYMMDD>.<build>` — hand-incremented, no build tooling stamps it. Bumped on every shipped change in this project's history.

---

## Corrections vs. the old specs in this folder

Two concrete, verified drifts found while writing this document — a demonstration of why the older `*.spec.md` files (frozen `v4.20260601.6`) shouldn't be trusted for current mechanics even where they look plausible:

1. **Cost basis** — `portfolio.spec.md` describes a simple lifetime-average `avgBuy` where "selling some shares does not change avgBuy for the remainder." The actual current code (§1.1 above) is a chronological moving average where a sell *is* booked against the running average and a full exit resets the basis entirely. This was a real bug fix earlier in this project's history, and the old spec still describes the pre-fix behavior.
2. **Theme/shell** — `ui-dashboard.spec.md` describes a green accent color and 4 tabs. Current code uses violet (deliberately, "distinct from v1's green, for side-by-side comparison") and has 8 tabs.

Treat any other specific numeric claim in those files (thresholds, formulas not touched by this project's session history) as *plausible but unverified* rather than confirmed current.
