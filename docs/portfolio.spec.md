# Portfolio & Trade Log Spec

Covers: the Portfolio tab (holdings, P&L, XIRR) and the Trade Log tab (manual entry, import, delete/undo). See [[data-layer]] for the `trades` schema this is all computed from, [[project-overview]] for how this fits the whole app.

## 1. Portfolio calculation (`calcPortfolio()`)

Everything is derived fresh from the full `trades` table on every load — there is no stored "current holdings" row anywhere; `trades` is the only source of truth.

**Per-stock aggregation** (grouped by `ticker`):
- `buyQty` / `sellQty` — summed quantities.
- `totalCost` = Σ(BUY qty × price) + Σ(BUY brokerage). `totalSell` = Σ(SELL qty × price) − Σ(SELL brokerage).
- `netQty = buyQty − sellQty`. A stock only appears as an active holding if `netQty > 0`.
- `avgBuy = totalCost / buyQty` — a simple weighted average across **all** BUY trades ever made for that ticker, not FIFO/LIFO lot tracking. Selling some shares does not change `avgBuy` for the remainder.
- **Realised P&L** (only if `sellQty > 0`): `totalSell − (sellQty × avgBuy)` — i.e. sells are costed at the *overall* average buy price, not at the price of any specific matched lot.

**Per-holding figures** (only for `netQty > 0` stocks):
- `invested = netQty × avgBuy`
- `currentValue = livePrice × netQty` (null until live price arrives)
- `unrealisedPnl = currentValue − invested`, `unrealisedPct = unrealisedPnl / invested × 100`
- `stockXirr` — see §2 below.

**Portfolio totals**: sums of the above across all active holdings, plus a portfolio-level XIRR (§2) computed across *every* trade ever made (not just active holdings — closed positions' historical cashflows still count).

## 2. XIRR (Newton-Raphson)

Cashflow convention: BUY = negative (money out) = `-(qty×price + brokerage)`; SELL = positive (money in) = `qty×price − brokerage`. For any stock still held, one additional synthetic positive cashflow is appended **today**, valued at `livePrice × netQty` — this is what turns "unrealised" gain into an XIRR-computable series (marking the open position to market at the valuation date).

Solve: `xirr(cashflows)` finds the rate where NPV = 0, using derivative-based Newton-Raphson (`npv`/`dnpv` helpers), starting guess 10%, up to 100 iterations, converging when the rate changes by < 1e-7 between iterations. Returns `null` (displayed as `—`) if:
- fewer than 2 cashflows exist,
- the derivative underflows (< 1e-10) before convergence,
- the rate diverges below -100% (`rate <= -1`),
- or the result isn't finite.

Both a **per-stock XIRR** (that stock's trades + its own current value) and a **portfolio XIRR** (all trades across all stocks + total current value, one combined series) are computed independently — the portfolio figure is not an aggregate of the per-stock figures, it's its own XIRR solve over the merged cashflow list.

XIRR requires live prices — until `fetchAllPrices()` resolves, both figures show `—` with a "needs live prices" / "fetching..." label rather than a stale or zero value.

## 3. Trade Log

Three ways rows enter `trades`, all writing the same schema (see [[data-layer]] §1.1):

1. **Manual entry** (`submitManualTrade`) — a modal with a BUY/SELL toggle, stock search (autocomplete against `universe`, prefix-stripped), date (defaults to today), exchange (auto-filled from the matched `universe` row, user can still override via a dropdown in the holdings table — `updateExchange()`), qty, price, brokerage (defaults ₹0), with a live running total shown as the user types. Client-side validation only: qty > 0, price > 0, a stock must be selected, a date must be set. **Note the open casing question in [[data-layer]] §1.1** — this path does not uppercase `action` before insert.
2. **Kite import** (`confirmImport`, full parse rules in [[data-layer]] §4) — bulk insert, one shared `import_batch` timestamp per batch, `action` uppercased, `brokerage` forced to 0.
3. Nothing else — no CSV-generic importer, no other broker format.

**Delete**: any single trade row can be deleted (🗑 button, confirm dialog), scoped to `id` + `user_id`. This is a hard delete with no soft-delete/audit trail — deleting a trade permanently changes historical P&L calculations retroactively (there's no "as of" snapshot).

**Undo Last Import**: see [[data-layer]] §4 for the mechanism. UI-visible only within 24 hours of an import, showing a live countdown (`"↩ Undo Last Import (3 trades · 23h left)"`), re-appearing correctly across page reloads (state lives in `localStorage`, checked on every `loadAll()`).

**Sorting**: both the Holdings table and the Trade Log table support click-to-sort on any column, with nulls always sorted last regardless of direction, and string comparisons case-insensitive. Sort state is in-memory only (`holdingsSort`, `tradesSort` module-level objects) — resets on page reload.

**Exchange correction**: a holding's exchange can be changed post-hoc via a dropdown directly in the Holdings table (`updateExchange`) — this is a live edit against `trades.exchange` for that ticker, used when a stock was imported/entered under the wrong exchange (which also affects which Yahoo Finance suffix — `.NS` vs `.BO` — gets used for its price lookups, see [[data-layer]] §2).
