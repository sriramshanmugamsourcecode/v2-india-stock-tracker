# WoW (Week-over-Week) Tracker Spec

Covers: the manually-triggered weekly price snapshot system, the pyramiding buy signal, the exit signal, and the buy-quantity calculator. This is the app's active-management layer — it turns "how is this holding doing" into a discrete add/hold/exit decision. See [[data-layer]] §1.2 for the `wow_entries` schema, [[portfolio]] for how holdings/avgBuy feed into this.

## 1. Mechanics

Nothing here is automatic. The user picks a date (defaults to the most recent Friday, `setDefaultWowDate()`) and clicks "Fetch & Save Week," which:
1. Takes the *current* list of active holdings from `calcPortfolio()`.
2. Fetches each one's Yahoo Finance closing price for that specific date (§2 in [[data-layer]]).
3. Overwrites (deletes-then-inserts) any existing `wow_entries` rows for that date.
4. Trims to the most recent 12 weeks (`cleanupOldWowEntries`).

Because holdings are re-derived at fetch time, a stock bought *after* an old week's snapshot will simply have no entry for that older week (shown as `—` in its trend arrows), not a backfilled/estimated value.

Each week renders as a collapsible card, newest first, newest expanded by default. A card can be deleted outright (`deleteWowWeek`, with confirm) — this removes every ticker's entry for that date, not just one stock's.

## 2. Per-holding, per-week figures

For each holding, for each week that has data:
- `pnlPct = (weekPrice − avgBuy) / avgBuy × 100` — **not** live price; this is P&L as of that week's snapshot price, so historical weeks show historical P&L, and today's actual unrealised P&L (Portfolio tab) can differ from the latest WoW week if prices moved since the last snapshot.
- `wowChgPct` — % change vs. the immediately preceding tracked week's price for the same stock (`null` if there's no prior week, e.g. a newly-added holding).
- `vsFloor = pnlPct − pnlFloor` (the user-configurable floor, default 40%, stored in `user_preferences`).
- A rolling 3-week arrow trend (`wowStreakArrows`) — newest→oldest, ⬆/⬇ with the % magnitude, padded with `—` if fewer than 3 prior weeks exist yet.

## 3. Pyramiding ("Buy Signal") — `getPyramidSignal(pnlPct, wowChgPct, floor)`

Evaluated independently for each holding, each week. Not priority-ordered — these four conditions are mutually exclusive by construction:

| Condition | Signal |
|---|---|
| P&L ≥ floor **and** price up (or flat) this week | ✅ **Strong Add** |
| P&L ≥ floor **and** price down this week | 🟡 **Hold** |
| P&L within 5 points below floor (`floor−5 ≤ pnlPct < floor`) | 🟡 **Watch** |
| Otherwise (more than 5 points below floor) | ❌ **Don't Add** |

`wowChgPct == null` (no prior week yet) counts as "up" for this purpose — a brand-new holding above floor defaults to Strong Add rather than Hold.

Rows are sortable by signal severity (Strong Add → Hold → Watch → Don't Add), and Strong Add rows get a "🧮 Qty?" button (§5).

## 4. Exit Signal — `getExitSignal(pnlPct, wowChgPct, history, floor, ema100)`

Unlike the buy signal, this **is** priority-ordered — only the single most severe triggered condition is shown, checked in this order:

1. **🔴 Hard Exit** — `pnlPct < floor − 10` (a "hard floor" 10 points below the pyramiding floor).
2. **🔴 Trend Break** — 3 consecutive down weeks (needs ≥4 data points to evaluate; looks at the last 3 weeks and requires all 2 week-over-week gaps to be negative).
3. **🔴 EMA Break** — last known price below the 100-day EMA. **This check is only available if the Discovery screener has been run within the last 3 days** — `ema100` comes from the screener's `localStorage` cache (`SCR_CACHE_KEY`), not a live calculation. If the cache is stale or absent, this rule is silently skipped (not a "pass," just not evaluated) and a banner tells the user to run the screener. This is a real cross-feature dependency: **WoW's exit-signal accuracy depends on how recently Discovery was run.**
4. **🟡 Near Floor** — between hard floor and floor (`floor−10 ≤ pnlPct < floor`) — early warning before Hard Exit triggers.
5. **🟡 Sharp Drop** — single-week drop worse than −5%, independent of overall P&L position.
6. **✅ Holding** — none of the above triggered.

`pnlPct == null` short-circuits to a bare `—` (order 99, sorts last) before any of the above run.

## 5. Buy-Qty Calculator (`calcBuyQty`)

Shown only on Strong Add rows. Answers: "how many more shares can I buy at this week's price without dropping my post-buy average-cost P&L% below the floor?"

```
floorMult = 1 + floor/100
qtyNew = qtyHeld × (weekPrice − floorMult × avgBuy) / (weekPrice × (floorMult − 1))
```
This solves for the added quantity that makes the *new* blended average cost exactly equal to `weekPrice / floorMult` (i.e. exactly at the floor after buying). It is a sizing suggestion only — clicking it does not create a trade; the user still enters it manually via Add Trade.

## 6. P&L floor preference

Single number (`pnlFloor`, default 40%), stored per-user in `user_preferences.pnl_floor`, editable via a dropdown in the WoW tab. Changing it re-renders every week's signals/exit evaluations immediately (no cache to invalidate — signals are computed at render time, not stored).
