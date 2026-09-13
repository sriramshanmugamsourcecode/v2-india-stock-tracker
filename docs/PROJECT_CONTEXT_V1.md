# India Stock Tracker — Project Context
*Last updated: 10-May-2026*

---

## Overview
A web-based Indian stock portfolio tracker, built to replace an Excel file (India_Stock_Tracker_v2.xlsx). Accessible on desktop and mobile via browser.

---

## Infrastructure

### Supabase
- **Project URL:** https://zlpejsixpycewmfpmwrz.supabase.co
- **Publishable (anon) key:** sb_publishable_4CmwdTrj7CndRqTZ4MGYvg_TXmP8uqv
- **Note:** RLS is enabled with temporary public read+insert policies. Auth lockdown (email login, only-me access) is planned for after Phase 1 is finalised.

---

## Database Schema

### `trades` (source of truth)
| Column | Type | Notes |
|---|---|---|
| id | uuid | auto |
| date | date | trade date |
| stock_name | text | e.g. "APARINDS" |
| ticker | text | e.g. "NSE:APARINDS" |
| action | text | BUY or SELL |
| qty | integer | shares |
| price | numeric | price per share ₹ |
| brokerage | numeric | default 0 |
| created_at | timestamp | auto |

### `wow_tracker`
| Column | Type | Notes |
|---|---|---|
| id | uuid | auto |
| ticker | text | e.g. "NSE:APARINDS" |
| week_date | date | Friday date |
| close_price | numeric | closing price ₹ |
| created_at | timestamp | auto |

### `watchlist` *(ready for Phase 4)*
| Column | Type | Notes |
|---|---|---|
| id | uuid | auto |
| ticker | text | |
| stock_name | text | |
| entry_condition | text | |
| exit_condition | text | |
| created_at | timestamp | auto |

### `imports` *(planned — not yet created)*
- Will track every CSV import: timestamp, trade count, date range covered
- Purpose: help user identify delta (trades not yet imported) on next upload

---

## Current Data State
- **41 real holdings** imported from Zerodha holdings statement (holdings-WED962__2_.xlsx) dated 09-May-2026
- Imported as synthetic single BUY entries dated 2026-05-09 with brokerage = 0
- This is a snapshot, not a trade-by-trade history — real tradebook import planned via Kite CSV
- `wow_tracker` is currently empty (truncated during cleanup)

---

## Web App
- **File:** india_stock_tracker.html (standalone HTML, no framework needed)
- **Stack:** Vanilla HTML + CSS + JS, connects directly to Supabase REST API
- **Tabs:** Portfolio | Trade Log | WoW Tracker
- **Known limitation:** Must be opened via http:// (not file://) — CORS blocks local file access
- **Hosting:** Not yet hosted — running locally via browser for now. Netlify/GitHub Pages planned.

---

## Build Phases

### ✅ Phase 1 — Foundation (in progress)
- [x] Supabase DB setup (trades, wow_tracker, watchlist tables)
- [x] RLS enabled with temporary public policies
- [x] Basic web UI: Portfolio + Trade Log + WoW Tracker tabs
- [x] 41 real holdings imported from Zerodha holdings statement
- [ ] `imports` table creation (tracking CSV import history)
- [ ] Kite tradebook CSV importer with duplicate detection
- [ ] Auth lockdown (Supabase email login, only-me access)
- [ ] Host on Netlify or GitHub Pages

### 🔲 Phase 2 — Live NSE Prices
- Fetch live prices via Yahoo Finance API (ticker format: APARINDS.NS)
- Auto-update Portfolio current value, P&L, return %
- Fill the "Live Price" placeholder currently shown in Portfolio tab

### 🔲 Phase 3 — WoW Tracker (web-native)
- Multi-stock WoW tracking (currently hardcoded to APARINDS in Excel)
- Weekly price entry UI in the browser
- Chart improvements

### 🔲 Phase 4 — Momentum + Fundamentals Screener
- Universe: Nifty 500
- Momentum filter: price vs 52W high, relative strength
- Fundamental filter: PE, ROE, Debt-to-Equity, Sales growth
- Entry/exit conditions per stock (stored in `watchlist` table)

---

## Kite Integration Plan
- **Short term:** Manual CSV download from Zerodha Console (console.zerodha.com)
  - Supports custom date range download (not just daily)
  - Import via CSV uploader in the app
  - `imports` table will track last import date to identify delta
  - Duplicate detection to prevent double-counting overlapping uploads
- **Long term:** Kite Connect API (₹2000/month) for auto-sync
  - Note: Kite Connect only provides today's tradebook via API; historical requires Console CSV
  - Daily token expiry is a known pain point to handle

---

## Key Decisions Made
| Decision | Choice | Reason |
|---|---|---|
| Backend | Supabase (free tier) | Simple, free, no server to manage |
| Price data | Yahoo Finance API (free) | Free, covers NSE with .NS suffix |
| Fundamentals | Yahoo Finance / Screener.in | Free |
| Screener universe | Nifty 500 | Broader than a fixed watchlist |
| Auth | Supabase email auth (only-me) | Single user, simplest option |
| Kite sync | CSV import first, API later | Free, covers 90% of the need |
| Portfolio calc | Calculated from trades table | No duplication, always accurate |
| Trade insert access | Authenticated users only | Security, single user |

---

## Original Excel File Summary (India_Stock_Tracker_v2.xlsx)
- **Sheet 1 - Portfolio:** Summary dashboard, calculated from Trade Log
- **Sheet 2 - Trade Log:** Source of truth, manual BUY/SELL entries
- **Sheet 3 - WoW Tracker:** Week-on-week price tracker for APARINDS, 52 weeks pre-filled
- Sample data: 2 BUYs + 1 SELL for APARINDS
