# India Stock Tracker — Project Overview

*This file is the single source of truth for architecture, current state, and open questions. Check this first when resuming this project.*

*Specs frozen against live code at `APP_VERSION = 'v4.20260601.6'` (commit `3801eeb`), reverse-engineered from `index.html` on 2026-08-26.*

## What this is

A personal, single-user Indian-equities portfolio tracker and momentum-discovery tool, built and run as **one static HTML file** hosted on GitHub Pages. No backend server — the browser talks directly to Supabase (Postgres + auth) and to Yahoo Finance (via a Cloudflare Worker CORS proxy).

- **Live app**: https://sriramshanmugamsourcecode.github.io/india-stock-tracker
- **Source repo**: https://github.com/sriramshanmugamsourcecode/india-stock-tracker (cloned into this folder as `repo/`)
- **Supabase project**: https://zlpejsixpycewmfpmwrz.supabase.co
- **Cloudflare Worker (Yahoo Finance CORS proxy)**: https://little-bird-6066.sriram-shanmugam.workers.dev/?url=

This is a **different, unrelated project** from `E:\Projects\AI\Investment Tracker\` (an older momentum-screener notebook project) — do not conflate the two, per explicit user instruction (2026-08-26).

## Why specs now

The app was built iteratively across four AI-assisted sessions (Phase 1 → Phase 4) with no formal spec — each session's handoff was a `PROJECT_CONTEXT.md` narrative snapshot pasted into a fresh chat. Two stale copies of that narrative exist (GitHub's, dated 10-May; the local `Starting Point/` bundle's, dated 27-May) and neither matches the actual shipped code (`v4.20260601.6`, newer than both). Going forward, this repo's specs + the code are authoritative; `PROJECT_CONTEXT.md` is retired as a source of truth (kept in git history only).

## Tech stack (as built, not chosen fresh)

- **Frontend**: vanilla HTML/CSS/JS, single file (`index.html`, ~4100 lines). No build step, no framework.
- **Backend**: Supabase — Postgres tables + Row Level Security + email/password auth. No custom server code.
- **Market data**: Yahoo Finance's unofficial `query1.finance.yahoo.com` chart/quoteSummary endpoints, fetched client-side through a Cloudflare Worker (`little-bird-6066`) that exists solely to add CORS headers Yahoo doesn't send.
- **Broker import**: manual Kite (Zerodha) tradebook `.xlsx` export, parsed client-side with SheetJS (`xlsx.full.min.js` from a CDN).
- **Hosting**: GitHub Pages, deployed by committing `index.html` to `main` (`.nojekyll` present, no Actions workflow — Pages deploys automatically on push).
- **Local state**: `localStorage` used for the screener's daily indicator cache, the monthly top-30 streak history, and the 24-hour import-undo token. None of this is server-side.

See [[data-layer]] for exact schema and [[discovery-screener]] for why Yahoo Finance (free, unofficial) was accepted over paid data.

## Feature map → spec files

| Spec file | Covers |
|---|---|
| [[data-layer]] | Supabase schema, RLS, Yahoo Finance fetch plumbing, ticker/symbol mapping, Kite import format |
| [[portfolio]] | Holdings calculation, P&L, XIRR, Trade Log (manual entry + import + delete/undo) |
| [[wow-tracker]] | Week-over-week price tracking, pyramiding buy signal, exit signal, buy-qty calculator |
| [[discovery-screener]] | Momentum/fundamental screener, composite ranking, presets, streak tracker, backtest results |
| [[ui-dashboard]] | Tab structure, general UI conventions, versioning, auth screen |

## Current build state — Phase 4 complete

- ✅ Phase 1 — Auth, Portfolio/Trade Log/WoW tabs, Supabase foundation
- ✅ Phase 2 — Live NSE/BSE prices (Yahoo Finance), unrealised P&L, XIRR
- ✅ Phase 3 — Web-native multi-stock WoW tracker (replaced the old single-stock Excel version)
- ✅ Phase 4 — Discovery screener (momentum + fundamentals, Nifty 500 universe), backtested
- 🔄 Phase 5A — Backtesting: Test #1 (pure composite rank) and Test #2 (filter impact) complete; Tests #2B/#3/#4/#6 planned but not run
- 🔲 Phase 5 — not yet built, **not yet specced**: Watchlist tab UI, Universe management UI, AI-assisted picks, market-cap filter, transaction-cost backtest, second user account ("wife's account"), Netlify hosting

## Phase 5 backlog (unspecced — needs requirements discussion before any spec or code)

Per the working agreements below (no silent assumptions; UI mocks before implementation), none of these should go straight to implementation — each needs a requirements discussion, and anything UI-facing needs a mockup reviewed before code:

1. **Watchlist tab UI** — the `watchlist` table exists and is written to (Discovery's "+ Watch" button), but there is no tab/screen to view or manage it yet.
2. **Universe management UI** — `universe` table is currently maintained by hand via the Supabase table editor; no in-app CRUD.
3. **AI-assisted picks** — undefined scope; needs a requirements conversation before anything else.
4. **Market cap filter** — would need a market-cap data source; Yahoo's `quoteSummary` may or may not expose it cleanly (unverified).
5. **Transaction-cost backtest (Phase 5B)** — apply real brokerage/STT/slippage to the Colab backtest notebooks.
6. **Second user account** — multi-user isn't designed; RLS is per-`user_id` already, so this may be low-effort, but auth/UI flow for a second login isn't decided.
7. **Netlify hosting** — alternative to GitHub Pages; no stated reason to switch, low priority.
8. **Ticker overrides maintenance (ongoing)** — `YF_OVERRIDES` and `SCR_YF_OVERRIDES` in `index.html` are hand-maintained maps for tickers where Yahoo's symbol differs from the DB ticker (renames, delistings, exchange quirks). New mismatches surface as fetch failures in the screener's exclusion log and get added manually.

## Verified-against-real-source findings (don't re-litigate — checked against actual code/DB, not recalled)

- **Universe ticker format**: the original seed SQL (`Starting Point/nifty500_universe.sql`) stores tickers *with* exchange prefix (`'NSE:POLYCAB'`), but the 27-May `PROJECT_CONTEXT.md` states the live table now stores tickers *without* the prefix (`'ABB'`, exchange in a separate column) — and the current `index.html` code defensively handles both (`ticker.includes(':') ? ticker.split(':')[1] : ticker'` in several places). Conclusion: the seed script is a historical artifact: the live table's actual format is un-prefixed, per the more recent doc and the code's own read paths (`filterStockSearch`, `handleImportFile`). Treat un-prefixed as current truth; the defensive `.split(':')` handling is legacy safety, not a sign the format is still mixed.
- **RLS on `universe`**: confirmed empirically (2026-08-26) — an unauthenticated REST call with only the anon/publishable key returns zero rows (not an error), consistent with the SQL policy `auth.role() = 'authenticated'`. This is expected RLS behavior, not a bug.
- **Supabase anon key is intentionally public**: `sb_publishable_...` keys are Supabase's client-side publishable key type, designed to be shipped in browser code and rely on RLS (not secrecy) for protection. It appearing in `index.html`/docs is not a leak.
- **`trades.action` casing**: Kite's tradebook export uses lowercase `buy`/`sell`; the DB constraint requires uppercase. The import path (`confirmImport`) uppercases it; the manual-entry path (`submitManualTrade`) does *not* explicitly uppercase — it relies on `manualAction` already being lowercase (`'buy'`/`'sell'`) from the Buy/Sell toggle, which **contradicts** the stated DB constraint (`action: 'BUY' or 'SELL' (uppercase enforced by check constraint)`) in `PROJECT_CONTEXT.md`. **This is a real, unresolved discrepancy** — see Open Questions below.

## Open questions

*(Struck through once answered — this is the running audit log, not deleted.)*

1. **Manual trade action casing bug?** `submitManualTrade()` inserts `action: manualAction` where `manualAction` is `'buy'`/`'sell'` (lowercase), but `PROJECT_CONTEXT.md` documents a DB check constraint requiring uppercase `BUY`/`SELL`. Either the constraint doesn't actually exist as documented, or every manual trade entry has been failing/erroring, or Postgres is doing something case-insensitive we haven't verified. **Needs verification against the live DB schema before this is written into portfolio.spec.md as settled behavior.**
2. **Exact current Supabase schema for `trades`, `wow_entries`, `watchlist`, `user_preferences`** — only inferred from client code + narrative docs, not from a DDL dump (unlike `universe`, which has a SQL file). Should pull the real schema (via Supabase dashboard or `pg_dump`) to confirm column types/constraints/defaults before treating data-layer.spec.md's schema section as fully verified.
3. **Whether `imports` table (mentioned as "planned" in the stale May-10 PROJECT_CONTEXT) was ever created** — the current import flow uses an `import_batch` timestamp column directly on `trades` instead, which looks like the shipped solution superseding that plan. Treated as superseded, not open, unless the live DB shows otherwise.
4. All seven Phase 5 backlog items above — need requirements discussion before spec-writing, not just a stub in this file.

## Working agreements for this project (carried over from prior Claude Code project conventions)

- **No assumptions**: don't silently assume an unstated value, data source, or design choice — surface it as a question. Applies to Phase 5 backlog items above.
- **UI mocks before implementation**: any new or changed UI (Watchlist tab, Universe management UI, etc.) gets a requirements discussion, then a mockup for review, before real code is written.
