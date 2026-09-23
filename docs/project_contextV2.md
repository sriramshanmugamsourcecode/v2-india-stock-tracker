# India Stock Tracker v2 — Project Context

*This is the current, accurate snapshot of the project. If you're resuming work with a fresh AI session, this file plus `index.html` is what you hand it. Superseded content lives alongside it for history — see "Superseded documents" at the bottom — but this file, not those, is the source of truth as of the date below.*

*Last verified against live code: `APP_VERSION = 'v4.20260921.52'`.*

## What this is

A personal, single-user Indian-equities portfolio tracker and momentum-discovery/rebalancing tool, built and run as **one static HTML file** hosted on GitHub Pages. No backend server — the browser talks directly to Supabase (Postgres + Auth) and to Yahoo Finance (via a Cloudflare Worker CORS proxy). Full environment/tooling setup is [`01-developer-environment-setup.md`](01-developer-environment-setup.md); this file covers architecture, accounts, and current feature/data state.

## Live references

- **App**: served via GitHub Pages from the `v2-india-stock-tracker` repo's `main` branch
- **Repo (active, v2)**: https://github.com/sriramshanmugamsourcecode/v2-india-stock-tracker
- **Repo (frozen, v1 — never push here, see below)**: https://github.com/sriramshanmugamsourcecode/india-stock-tracker
- **Supabase project (active, v2)**: `lixhqzzdutdjpdqfxmgn` — https://lixhqzzdutdjpdqfxmgn.supabase.co
- **Supabase project (frozen, v1)**: `zlpejsixpycewmfpmwrz` — historical only, do not write to it
- **Cloudflare Worker (Yahoo Finance CORS proxy)**: `little-bird-6066` — https://little-bird-6066.sriram-shanmugam.workers.dev/?url=

Real credentials for all of the above: `PROJECT_SECRETS_LOCAL_ONLY.md`, one directory above this repo — see [`01-developer-environment-setup.md`](01-developer-environment-setup.md) §5. Nothing secret is repeated here.

## Why there are two repos (v1 and v2)

v1 was the original project. At some point development continued in a separate `v2-india-stock-tracker` repo/Supabase project so the two could run side-by-side for comparison, with v1 explicitly frozen. **This project's local git remotes have `origin` (v1) push intentionally disabled** — only `v2` is ever pushed to. This is a standing rule, not a preference: never modify the v1 repo or its Supabase project from this codebase.

## Tech stack

- **Frontend**: vanilla HTML/CSS/JS, single file (`index.html`, ~7,900 lines as of this version). No build step, no framework, no bundler.
- **Backend**: Supabase — Postgres tables + Row Level Security + email/password auth. No custom server code anywhere.
- **Market data**: Yahoo Finance's unofficial `query1.finance.yahoo.com` chart/quoteSummary endpoints, fetched client-side through the Cloudflare Worker above (adds CORS headers Yahoo doesn't send). Not an official or paid API — see Doc 1 §6 for the caveat.
- **Broker import**: manual Zerodha Kite `.xlsx` export (tradebook or holdings statement), parsed client-side with SheetJS. No broker API integration anywhere.
- **Hosting**: GitHub Pages, deploys automatically on push to `main` (`.nojekyll` present at repo root, no Actions workflow needed).
- **Local (per-browser) state**: `localStorage` for several feature-specific caches, `IndexedDB` for App Maintenance's backup history — see "Client-side caching" below. None of this is shared across devices or backed up server-side.

## Database — current schema (9 tables, all in the v2 Supabase project)

| Table | Scope | Purpose |
|---|---|---|
| `trades` | Per-user (`user_id`) | Source of truth for every buy/sell. Portfolio, WoW, Rebalance are all derived from this at render time — nothing else stores a "current holding" row. |
| `wow_entries` | Per-user | Weekly closing-price snapshots, captured manually via the WoW tracker's "Fetch & Save Week." Feeds the WoW tab, and (reused, not refetched) the Rebalance tab's 1W/3W/Gain-Lost columns. |
| `user_preferences` | Per-user (PK `user_id`) | `pnl_floor` (WoW target P&L floor, default 40) and `gain_lost_exit_pct` (Exit Signals' Gain Lost threshold, default -20, added 2026-09-21). |
| `watchlist` | Per-user | **Retired 2026-09-21.** Still exists with its original schema (no unique constraint on `(ticker, user_id)`, which meant its one write path errored on every use), but nothing in the app writes to it any more — Discovery's old "+ Watch" button was replaced by the shared `rank_status` tag (see below). |
| `universe` | **Shared across all users**, not per-user | The Nifty-500-ish stock list Discovery/Rank History screen against (722 rows, 717 active as of the last sync). Bare tickers (no exchange prefix), `exchange` in its own column. Editable via the App Maintenance → Universe panel. |
| `rank_status` | Per-user | Invest/Watch/Don't invest status tag per ticker (values still `go`/`wait`/`dont`) — settable from **both** Rank History's Trend view and Discovery's rows; same tag shows in both. |
| `rank_history` | **Shared** (no `user_id`) | Monthly composite-rank snapshots (top ~50 stocks) captured via Rank History's "Capture" button — columns: `month, rank, ticker, stock_name, composite, score52w, ret6m_skip, ret3m_skip, ret1m_skip, adj_ret, universe_size, source, captured_at`. |
| `holding_tags` | Per-user | Long term / Momentum tagging for Rebalance. Only `'momentum'` rows are ever written — untagged = Long term by default, no row needed. |
| `momentum_capital` | Per-user | Append-only log of the user-set momentum-sleeve capital cap — latest row = current, one before = previous week. |

RLS on every per-user table is `auth.uid() = user_id` for select/insert/(update)/(delete), verified via `pg_policies` at the time each table was built — see individual feature notes in memory/git history for exact per-table policy lists if you need to re-verify. `universe` and `rank_history` are shared, not user-scoped — `universe` allows authenticated insert/update/delete (Maintenance panel); `rank_history` is written by whichever user runs a Capture, read by all.

**No `imports` table** — an early plan (visible in the retired v1-era context doc) was superseded by a simpler `import_batch` timestamp column directly on `trades`, which is what's actually shipped.

## Client-side caching (per-browser, not synced, not backed up)

- `localStorage`: `lk_tickers_v1`/`lk_open_v2` (Pattern Lookup), `rh_trend_v1`/`rh_notin50_v1` (Rank History caches), `mnt_override_todo_v1` (Yahoo-override to-do list), `rb_52w_v1` (Rebalance's 52-week-high cache, 24h TTL), `last_import_batch_v2` (24h undo window for the last Kite import).
- `IndexedDB`: `mnt_backups_db` — App Maintenance's Backups feature, last 10 full-table snapshots kept.

Clearing browser data wipes all of the above but never touches Supabase — it's all either a cache (refetches fine) or a convenience (undo window, backup history).

## Current feature map (8 tabs)

Nav order: Portfolio → Trade Log → WoW Tracker → Discovery → Rank History → Exit Signals → Rebalance → Maintenance (Maintenance moved to the end on 2026-09-21 to make room for Exit Signals ahead of Rebalance).

| Tab | Covers |
|---|---|
| Portfolio | Holdings, P&L, per-stock XIRR, Long term/Momentum tag toggle |
| Trade Log | Manual entry, Kite import, delete/undo |
| WoW Tracker | Weekly price capture, pyramiding buy signal, exit signal, buy-qty calculator |
| Discovery (⬡) | Momentum + fundamentals screener over `universe`, Invest/Watch/Don't invest tag per row (shared with Rank History), Pattern Lookup panel (copy-prompt-to-Claude, no in-app AI call) |
| Rank History (📈) | Grid/Trajectory/Trend views over `rank_history` snapshots, Invest/Watch/Don't invest status tags |
| Exit Signals (🚪) | Full-holdings table consolidating every exit rule — WoW's original 5 (unchanged) plus 3 new: Gain Lost breach (user-set threshold), 50-DMA break, Rank Decay (best-vs-current composite rank, red flag after 3+ consecutive months outside top 50). All/Momentum filter, sortable columns. |
| Rebalance (⚖️) | Momentum capital cap, momentum-holdings table, "Rebalance for this week" split calculator (Weighted / Equal / Side-by-side), **Pyramiding** sub-section (staged per-stock entry planner with a cushion floor, deployment amount, and optional max-tranche cap) |
| Maintenance (🛠️) | Universe / Trade Logs / Portfolio (Backups + Reset Portfolio) direct-edit panels |

Each feature beyond the original Phase 1–4 core is built as an isolated IIFE (or sub-section within one) with its own CSS class prefix (`.rh-*`, `.lk-*`, `.mnt-*`, `.rb-*`, `.ex-*`, `.rb-pyr-*`), touching shared code only via one nav button, one `switchTab` array entry, and one hook line. This convention has held for every feature added since Rank History.

## Known-resolved items (carried forward from the retired spec files, now checked against current code)

- **Manual trade action casing** — a prior spec flagged that `submitManualTrade()` might insert lowercase `buy`/`sell` against a DB constraint requiring uppercase. **Verified resolved**: current code does `action: manualAction.toUpperCase()` before insert (`index.html` ~line 4887). Not a live bug.
- **Universe ticker prefix format** — previously mixed (`NSE:ABB` vs bare `ABB`); confirmed fully bare across all rows after a 2026-08-26 migration and the subsequent Nifty-500 sync project.

## Open items worth knowing about

- The Cloudflare Worker's actual source isn't backed up in git anywhere (see Doc 1 §5) — only its behavior is documented.
- `watchlist` table — **retired 2026-09-21**, not fixed. Confirmed via live schema dump (no unique constraint on `(ticker, user_id)`, so the app's `onConflict: 'ticker,user_id'` upsert errored on every use). Rather than add the missing constraint, Discovery's "+ Watch" button was removed and replaced with the shared `rank_status` tag it already had working elsewhere. The table itself still exists, unfixed, unused.
- `rank_status` — **fixed 2026-09-12**, and since 2026-09-21 also writable from Discovery, not just Rank History. Its migration had never actually been run despite the Rank History status-tag feature being fully shipped in code; the table silently didn't exist. Created live (see `03-ai-rebuild-spec.md` §3 for the exact SQL) and verified via `pg_policies`. Labels renamed Good-to-go/Wait-and-watch/Don't-venture → Invest/Watch/Don't-invest at the same time Discovery started using it.
- `getEmaFromCache()` — **fixed 2026-09-21**: called `.find()` (array-only) on the Screener's localStorage cache, which is actually a plain object keyed by ticker — always silently threw, always returned `null`. Means WoW's "100 EMA Break" exit rule had likely never fired since it shipped. Fixed with `Object.values(data).find(...)`, found while wiring a new 50-DMA lookup that copied the same broken pattern.
- The manual-entry stock-search dropdown defaults exchange to `'NSE'` based on ticker format rather than reading the `exchange` column — harmless for the ~717 NSE-listed universe rows, would mistag a manually-added BSE-only stock (flagged, not fixed, per the project's no-silent-logic-changes convention).
- Sell/rotation logic for Rebalance (tax-aware, checkbox-gated) is designed but not built — see memory `feature_rebalance.md`.
- A proper historical factor-validation study for the Rebalance composite weights is backlogged, not built — see memory `backlog_momentum_indicators_validation.md`.
- Exit Signals' Rank Decay trigger (3 consecutive months outside top 50) and its tranche trigger-price spacing in Pyramiding (round +5% price bands) were both explicit, deliberate defaults, not backtested — worth keeping in mind alongside the existing composite-weights validation backlog item above.

## How to resume a session

This project now primarily resumes via Claude Code's own persistent memory (`~/.claude/projects/.../memory/`, indexed by `MEMORY.md`), not by manually re-uploading a context file each time — that was the old (pre-Claude-Code) workflow, kept here as history. If starting genuinely fresh (new machine, memory unavailable), read this file, `01-developer-environment-setup.md`, and `index.html` itself, in that order.

## Superseded documents (kept for history, not maintained, don't treat as current)

- `PROJECT_CONTEXT_V1.md` — the original Phase-1-era context doc (10-May-2026). Wrong repo, wrong Supabase project, wrong table names (`wow_tracker` vs the actual `wow_entries`) relative to today. Kept only because deleting project history felt worse than clearly labeling it.
- `project-overview.md` + `data-layer.spec.md` + `discovery-screener.spec.md` + `portfolio.spec.md` + `ui-dashboard.spec.md` + `wow-tracker.spec.md` — a genuinely well-built spec set from a prior session, frozen at `v4.20260601.6`. Still useful for understanding *how* certain Phase 1–4 mechanics work in detail (e.g. `calcPortfolio()`'s moving-average cost basis, the screener's composite ranking formula) since that core logic hasn't changed — just don't trust anything in them about which tables/features/tabs currently exist.
- `KNOWN_BUGS.md` — a closed changelog (all 6 items resolved at the v1→v2 cutover). Accurate as history, not something to act on.
