# Rebuild Spec — India Stock Tracker v2

*Hand this file, [`04-business-logic.md`](04-business-logic.md), and (if reproducing the exact look) [`ui-dashboard.spec.md`](ui-dashboard.spec.md) §2 to a fresh AI session with an empty codebase, and it should be able to regenerate an equivalent app. This file is the product brief + technical spec + database DDL; `04-business-logic.md` is the exhaustive formula-level detail this file deliberately doesn't repeat.*

## 1. What to build

A personal, single-user (extensible to a handful of users) Indian-equities portfolio tracker with an active-management layer (weekly signal tracking) and a momentum-discovery/rebalancing tool on top. Not a general brokerage app — no order placement, no live broker integration. The user manually logs trades (or imports a broker export) and the app does everything else: valuation, P&L, signals, screening, and — the newest layer — momentum-sleeve capital allocation.

**Non-goals**, stated explicitly so a fresh build doesn't scope-creep: no multi-broker support beyond manual CSV/XLSX upload, no real-time streaming prices (periodic fetch is fine), no mobile app (responsive web is sufficient), no automated trade execution of any kind.

## 2. Technical constraints (this project's actual choices — reconsider only if you have a real reason to)

- **Single static HTML file**, vanilla HTML/CSS/JS. No framework, no build step, no bundler, no npm dependency tree. This was a deliberate simplicity choice, not a limitation discovered the hard way — it's kept the whole project buildable/editable by hand-editing one file for its entire life.
- **Backend**: Supabase (Postgres + Auth + Row Level Security). No custom server code anywhere — the browser talks to Supabase directly via its JS client.
- **Hosting**: GitHub Pages, deployed by pushing to the repo's `main` branch (needs a `.nojekyll` file at repo root, no Actions workflow required).
- **Market data**: Yahoo Finance's public (unofficial) chart/quoteSummary JSON endpoints, fetched through a small CORS-proxy (a Cloudflare Worker in this project, but any thin server-side proxy works) since Yahoo doesn't send CORS headers browsers require for direct fetch. See [`01-developer-environment-setup.md`](01-developer-environment-setup.md) §5 for the accepted-risk framing — this is not a licensed API.
- **Broker import**: manual `.xlsx` upload (tradebook and/or holdings-statement formats), parsed client-side (SheetJS in this project). No broker API integration.
- Full environment/account setup: [`01-developer-environment-setup.md`](01-developer-environment-setup.md).

## 3. Database schema — ready-to-run DDL

Every table below is copied **exactly** from a live schema dump of the real v2 database (`information_schema.columns`, `table_constraints`, `check_constraints`, and `pg_policies`, queried directly on 2026-09-12; `user_preferences.gain_lost_exit_pct` added 2026-09-21) — not reconstructed from memory or old docs. Column types, defaults, nullability, constraints, and RLS policies all match what's actually running. Several genuine findings surfaced by this dump and later work are called out after the SQL, not smoothed over.

```sql
-- ── trades — the source of truth. Everything else is computed from this. ──
create table trades (
  id uuid primary key default gen_random_uuid(),
  date date not null,
  stock_name text not null,
  ticker text not null,
  action text not null check (action in ('BUY','SELL')),
  qty integer not null,
  price numeric not null,
  brokerage numeric default 0,
  created_at timestamp default now(),
  user_id uuid,                              -- no FK to auth.users — see note below
  exchange text not null default 'NSE' check (exchange in ('NSE','BSE')),
  import_batch text                          -- shared timestamp per bulk import, null for manual entries
);
alter table trades enable row level security;
create policy "users read own trades"   on trades for select using (auth.uid() = user_id);
create policy "users insert own trades" on trades for insert with check (auth.uid() = user_id);
create policy "users update own trades" on trades for update using (auth.uid() = user_id);
create policy "users delete own trades" on trades for delete using (auth.uid() = user_id);

-- ── wow_entries — weekly closing-price snapshots ──
create table wow_entries (
  id uuid primary key default gen_random_uuid(),
  week_date date not null,
  ticker text not null,
  close_price numeric not null,
  user_id uuid,                              -- no FK
  created_at timestamp default now()
);
alter table wow_entries enable row level security;
create policy "users read own wow_entries"   on wow_entries for select using (auth.uid() = user_id);
create policy "users insert own wow_entries" on wow_entries for insert with check (auth.uid() = user_id);
create policy "users delete own wow_entries" on wow_entries for delete using (auth.uid() = user_id);
-- no update policy exists — a week's price is deleted and re-inserted, never edited in place

-- ── user_preferences — WoW P&L floor + Exit Signals' Gain Lost threshold ──
create table user_preferences (
  user_id uuid primary key,                  -- no FK
  pnl_floor integer default 40,
  gain_lost_exit_pct integer default -20,    -- added 2026-09-21 for the Exit Signals tab
  created_at timestamp default now(),
  updated_at timestamp default now()
);
alter table user_preferences enable row level security;
create policy "users read own prefs"   on user_preferences for select using (auth.uid() = user_id);
create policy "users insert own prefs" on user_preferences for insert with check (auth.uid() = user_id);
create policy "users update own prefs" on user_preferences for update using (auth.uid() = user_id);
-- no delete policy

-- ── watchlist — RETIRED 2026-09-21 (see finding #2 below) — table still exists, nothing writes to it anymore ──
create table watchlist (
  id uuid primary key default gen_random_uuid(),
  ticker text not null,
  stock_name text not null,
  entry_condition text,
  exit_condition text,
  created_at timestamp default now(),
  user_id uuid,                              -- no FK
  exchange text not null default 'NSE' check (exchange in ('NSE','BSE'))
);
alter table watchlist enable row level security;
create policy "users read own watchlist"   on watchlist for select using (auth.uid() = user_id);
create policy "users insert own watchlist" on watchlist for insert with check (auth.uid() = user_id);
-- no update, no delete policy — and no unique constraint on (ticker, user_id) either, see caveat below

-- ── universe — SHARED across all users, the stock list Discovery screens against ──
create table universe (
  id uuid primary key default gen_random_uuid(),
  ticker text not null unique,               -- bare, no exchange prefix e.g. 'ABB' not 'NSE:ABB'
  stock_name text not null,
  exchange text not null default 'NSE' check (exchange in ('NSE','BSE')),
  active boolean default true,
  created_at timestamp default now()
);
alter table universe enable row level security;
create policy "authenticated users read universe" on universe for select using (auth.role() = 'authenticated');
create policy "authenticated insert" on universe for insert with check (auth.role() = 'authenticated');
create policy "authenticated update" on universe for update using (auth.role() = 'authenticated');
create policy "authenticated delete" on universe for delete using (auth.role() = 'authenticated');

-- ── rank_status — per-user Go/Wait/Don't annotation on a ticker ──
-- (this exact migration was found MISSING from the live DB on 2026-09-12 despite the feature
--  being fully built and shipped — see caveat below — and was run for the first time that day)
create table rank_status (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id),
  ticker text not null,
  status text not null check (status in ('go','wait','dont')),
  updated_at timestamptz not null default now(),
  unique(user_id, ticker)
);
alter table rank_status enable row level security;
create policy "select own rank_status" on rank_status for select using (auth.uid() = user_id);
create policy "insert own rank_status" on rank_status for insert with check (auth.uid() = user_id);
create policy "update own rank_status" on rank_status for update using (auth.uid() = user_id);
create policy "delete own rank_status" on rank_status for delete using (auth.uid() = user_id);

-- ── rank_history — SHARED, monthly composite-rank snapshots ──
create table rank_history (
  id uuid primary key default gen_random_uuid(),
  month text not null,                       -- 'YYYY-MM'
  rank integer not null,
  ticker text not null,
  stock_name text,
  composite numeric,
  score52w numeric,
  ret6m_skip numeric,
  ret3m_skip numeric,
  ret1m_skip numeric,
  adj_ret numeric,
  universe_size integer,
  source text not null default 'live',       -- 'live' | 'colab_backfill'
  captured_at timestamptz not null default now(),
  unique(month, ticker)
);
alter table rank_history enable row level security;
create policy "auth read"   on rank_history for select using (auth.role() = 'authenticated');
create policy "auth insert" on rank_history for insert with check (auth.role() = 'authenticated');
create policy "auth delete" on rank_history for delete using (auth.role() = 'authenticated');
-- no update policy — a month is captured once and frozen; corrections are delete-and-recapture

-- ── holding_tags — Long term / Momentum tagging for Rebalance ──
create table holding_tags (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id),
  ticker text not null,
  tag text not null default 'momentum' check (tag = 'momentum'),
  updated_at timestamptz not null default now(),
  unique(user_id, ticker)
);
alter table holding_tags enable row level security;
create policy "select own tags" on holding_tags for select using (auth.uid() = user_id);
create policy "insert own tags" on holding_tags for insert with check (auth.uid() = user_id);
create policy "delete own tags" on holding_tags for delete using (auth.uid() = user_id);
-- no update policy — the only tag value is 'momentum'; untagging is a delete, not an update

-- ── momentum_capital — append-only log of the user-set momentum sleeve cap ──
create table momentum_capital (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id),
  amount numeric not null,
  updated_at timestamptz not null default now()
);
alter table momentum_capital enable row level security;
create policy "select own capital" on momentum_capital for select using (auth.uid() = user_id);
create policy "insert own capital" on momentum_capital for insert with check (auth.uid() = user_id);
create policy "delete own capital" on momentum_capital for delete using (auth.uid() = user_id);
-- no update policy — append-only by design, a correction is a new row, not an edit
```

**Auth**: Supabase email/password auth, no signup UI in the app itself — accounts are provisioned by hand (Studio → Authentication → Users → Invite). No social login, no magic link, nothing fancier.

### Real findings surfaced by this schema dump and later work (not smoothed over)

1. **`rank_status` didn't exist at all until 2026-09-12.** The Rank History tab's Go/Wait/Don't status-tag feature was fully designed and shipped in code, but its migration was never actually run — the table was silently absent from the live database, meaning every attempt to set a status tag had been failing since the feature shipped. Confirmed via `information_schema.columns` returning zero rows for it, then fixed live during this documentation pass (the `create table rank_status` block above is exactly what was run). If you're building fresh from this spec, this is a non-issue — just don't skip actually running every migration you write.

2. **`watchlist`'s one write path was confirmed broken, then retired rather than fixed (2026-09-21).** The application code did `db.from('watchlist').upsert({...}, { onConflict: 'ticker,user_id' })`, but the live table had **no unique or exclusion constraint on `(ticker, user_id)`** — Postgres requires one matching the `onConflict` columns for an upsert to work at all, so the call errored on every single click (visible to the user as a "✗ Error" on the Discovery tab's "+ Watch" button). Rather than add the missing constraint, the button and its write path were removed entirely — Discovery already has a working, shared per-ticker tag mechanism (`rank_status`, the same Invest/Watch/Don't invest tag Rank History uses, see [`04-business-logic.md`](04-business-logic.md) §4.6) that made a second, broken tagging table redundant. `watchlist` itself was not dropped and still exists with its original (still-flawed) schema — nothing in the app writes to it any more, so the missing constraint is now moot rather than fixed.

3. **A related bug in the *same* investigation: `getEmaFromCache()` always returned `null`.** Found while wiring a new 50-DMA lookup that copied its pattern — the function called `.find()` (an array-only method) on the Screener's localStorage cache, which `scrSaveCache()` actually stores as a **plain object keyed by ticker**, not an array. The call threw every time, silently caught, always returning `null`. This means WoW's existing "100 EMA Break" exit rule had likely never actually fired in production since it shipped. Fixed (`Object.values(data).find(...)`) alongside the new 50-DMA rule it was found while building — see [`04-business-logic.md`](04-business-logic.md) §9.

4. **Only the three next-newest tables (`holding_tags`, `momentum_capital`, `rank_status`) have a real foreign key from `user_id` to `auth.users(id)`.** Every older table (`trades`, `wow_entries`, `user_preferences`, `watchlist`) relies purely on RLS + application code for that relationship, with no DB-level enforcement. Worth being consistent about if you're building fresh — add the FK from the start.

## 4. Recommended build order

This project was actually built in these phases, and each one is independently useful — a rebuild doesn't have to do everything before shipping something usable:

1. **Foundation** — Supabase auth + `trades` table, Portfolio tab (holdings computed from trades, §1 of the business-logic doc), Trade Log tab (manual entry + delete).
2. **Live prices** — Yahoo Finance fetch via the CORS proxy, unrealised P&L, per-stock and portfolio XIRR (**build the 5,000% clamp in from day one** — see the business-logic doc §1.2 for why; this isn't an edge case, it's routine once you top up positions regularly).
3. **WoW tracker** — manual weekly snapshot capture, pyramiding buy signal, exit signal, Gain Lost, buy-qty calculator.
4. **Discovery screener** — `universe` table (seed it — Nifty 500 or whatever index you're targeting), the indicator set and composite ranking formula (§4 of the business-logic doc), filters, presets.
5. **Kite/broker import** — `.xlsx` parsing, `import_batch` tracking, undo window.
6. Everything after this point (Rank History, Pattern Lookup, App Maintenance CRUD panels, the Rebalance tab and its split calculator, the Exit Signals tab, and Pyramiding inside Rebalance) was added well after the core was stable and battle-tested — treat them as genuinely optional layers, not core requirements, unless you specifically want feature parity with this project's current state.

## 5. Gotchas this project actually hit — build these in correctly the first time

These are all things that were bugs in earlier versions of this exact app, fixed after real data got confusing. A fresh build that ignores them will likely rediscover them the same way.

- **Cost basis must be a chronological moving average with reset-on-full-exit**, not a simple lifetime average of all buys. See [`04-business-logic.md`](04-business-logic.md) §1.1 for the precise algorithm. Getting this wrong silently corrupts P&L and XIRR for any stock that's ever been fully exited and re-bought.
- **XIRR needs a sanity clamp.** Annualizing a short holding period (days to a few weeks) mathematically produces extreme numbers even for a completely normal move — and near-simultaneous large cashflows can additionally send an unclamped Newton-Raphson solver to genuinely absurd (not just extreme) values. Clamp the upper bound (this project uses 5,000%/year); don't null it out, or you'll wrongly zero out that signal wherever XIRR feeds a ranking/composite score elsewhere in the app.
- **Ticker grouping needs prefix-stripping but should not auto-strip suffixes.** Group holdings by bare ticker (strip `NSE:`/`BSE:`), but leave broker-quirk suffixes (`-IV`, `-RR`) alone and require a human to confirm a merge — auto-stripping those caused real distinct-instrument merges to go wrong in this project's history (see [`04-business-logic.md`](04-business-logic.md) §8.1).
- **Momentum ranking should skip the most recent month** (Jegadeesh-Titman convention) for the *ranking* return calculation specifically, while still showing the plain, non-skipped return for display — using the same number for both understates recent reversal risk.
- **A hand-maintained ticker→Yahoo-symbol override map is unavoidable** — Yahoo's symbol doesn't always match the exchange's real trading symbol (renames, delistings, exchange-suffix quirks). Never guess a replacement symbol without confirming it resolves to the *correct company* — this project shipped a real mislabeling bug once (a ticker pointed at a similarly-named but different company) from an unconfirmed guess.
- **Universe search-matching must not silently default to the wrong exchange.** If your ticker format ever becomes ambiguous about which exchange a stock trades on, don't infer it from the ticker string's shape (e.g. "has no prefix → assume NSE") — read the actual `exchange` column, or you'll mis-fetch prices for anything BSE-only.

## 6. UI/UX baseline (optional — only needed for visual parity, not functional parity)

Dark theme, no light mode. This project's current v2 theme uses a violet accent (`#a78bfa`) specifically chosen to be visually distinct from a predecessor version's green — pick whatever accent you like if you're not trying to distinguish from anything. Numeric/tabular values consistently set in a monospace face; a display face for headings; a body face for everything else. Sortable table columns everywhere (click header, click again to reverse, nulls always sort last). See [`ui-dashboard.spec.md`](ui-dashboard.spec.md) for the full (though dated on tab count) shell conventions.

## 7. What this document deliberately leaves out

Exact pixel-level CSS, the specific wording of every UI label, and anything already covered in depth elsewhere — for those, read [`04-business-logic.md`](04-business-logic.md) (exact formulas, every feature's rules) and the actual `index.html` (the real, current implementation, which is always more authoritative than any doc describing it).
