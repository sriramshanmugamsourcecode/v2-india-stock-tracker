# Discovery Screener Spec

Covers: the momentum + fundamentals screener that ranks the Nifty-500-ish `universe` table and surfaces buy candidates. This is the one part of the app backed by actual backtesting (§7). See [[data-layer]] §2-3 for how price/fundamentals data is fetched, [[wow-tracker]] §4 for the one place its output (100-day EMA) is consumed outside this tab.

## 1. Indicators computed per stock (`scrCalc`, from a 1-year daily close+volume series)

All computed client-side from the raw daily series fetched per §2 in [[data-layer]] — no indicator is pulled pre-computed from any API.

| Indicator | Definition |
|---|---|
| `hi52` | max close over the last 252 trading days (fixed window, not calendar year) |
| `pct52w` | `(hi52 − price) / hi52 × 100` — gap below 52W high; 0 or negative means at/above it |
| `score52w` | `price / hi52` — used only for ranking (§3), not displayed directly |
| `dma50/100/200` | simple moving average, last N closes |
| `ema100/200` | exponential MA, seeded from the SMA of the first N-back closes then EMA'd forward |
| `ret1m/3m/6m/1y` | plain point-to-point % return over N trading days (21/63/126/252) — **display only** |
| `ret1mSkip/3mSkip/6mSkip` | **ranking** returns: same window, but ending 21 trading days ago instead of today (Jegadeesh-Titman "skip the most recent month" convention, to avoid short-term reversal contaminating a momentum signal) |
| `volatility` | annualised stdev of daily log returns over the last 3 months (×√252) |
| `adjRet` | `ret6mSkip / volatility` — a Sharpe-like, volatility-adjusted momentum score |
| `volRatio` | avg daily volume (last 1M) ÷ avg daily volume (last 6M) — not currently used in ranking or any active filter (computed, displayed nowhere as of this version — legacy/future-use) |
| `adtv` | Average Daily Traded Value, ₹ Cr, last 20 trading days (`avg(close×volume) / 1e7`) — the liquidity gate |
| `updays` | % of up-days over the last 6 months — **removed from all filters after backtesting showed it hurt returns** (§7); the field is still computed but `scrGetFilters()` hardcodes it to `null` |
| `pe, eps, roe, de, rev` | fundamentals, straight from Yahoo `quoteSummary`, `null` if unavailable |

## 2. Composite ranking (`scrRank`) — runs on the *entire* fetched universe, before any filter

1. Each of `score52w`, `ret6mSkip`, `ret3mSkip`, `ret1mSkip`, `adjRet` is independently rank-ordered across all stocks (rank 1 = best/highest). Stocks with a `null` value for a given metric get that metric's **median rank**, not a worst-case penalty — this avoids a single missing data point (e.g. insufficient price history) tanking a stock's overall position.
2. Weighted sum: `compositeRank = rank52w×2 + rank6m×2 + rank3m×2 + rank1m×1 + rankAdjRet×1` (total weight 8×). Lower composite = stronger.
3. `position` = the stock's ordinal place when every stock in the universe is sorted by `compositeRank` — this is the "Rank #N/500" figure shown in results, and it's computed **before** filters are applied, so a stock's rank reflects the whole universe regardless of which filtered view you're looking at.

This composite formula is exactly what backtest Test #1 validated (§7) — changing the weights or the metric set here invalidates that backtest's applicability until re-run.

## 3. Filters — two independent layers

Filters only decide what's **displayed**; they never change `position`/`compositeRank`, which is always universe-wide.

**Liquidity** (applied first, unconditionally if set): `adtv < threshold` → excluded, logged as `layer: 'liquidity'`.

**Momentum** (`scrMomentum`) — all four must pass:
- `w52`: gap to 52W high ≤ threshold (or, if threshold is negative, requires literally being *at* a new high, `pct52w ≤ 0`)
- `dma`: price above a chosen moving average — user picks one of `50`/`100`/`200` DMA, `100ema`, or **`brutal`** (price > DMA50 > DMA100 > DMA200, all three stacked bullish — the strictest option)
- `ema`: optional 100EMA > 200EMA golden-cross filter (`f.emaCross`, on/off toggle)
- `ret`: return over a chosen period (3M/6M/1Y, user's choice) exceeds a threshold — uses the **display** return, not the skip-month one, for this specific filter

**Fundamentals** (`scrFund`) — P/E below threshold, EPS above 0 (default), ROE above threshold, D/E below threshold, revenue growth above threshold. Only enforced as a hard gate in **Strict mode**; in **Advisory mode** (default) they're shown as colored dots but don't disqualify a stock. A `null` fundamental value is neither pass nor fail — it's excluded from the strong/watch signal count (§4) and shown as a neutral dot, and in Strict mode a `null` does not disqualify (only an explicit `false` does).

A stock failing momentum is excluded before fundamentals are even checked (fundamentals are never evaluated on a momentum-failing stock). Every exclusion is logged with a human-readable reason and shown in a collapsible "not in results" panel, separately bucketed by which layer excluded it (`fetch` / `liquidity` / `momentum` / `fundamental`) — this log is a first-class part of the UI, not a debug artifact.

## 4. Signal classification (`scrSignal`) — only for stocks that passed momentum

```
green = count of fundamentals that are explicitly true (nulls excluded from both numerator and denominator)
green ≥ 4 → 'strong'   (rendered "● Strong pick")
green ≥ 2 → 'watch'    (rendered "◑ Watchlist")
otherwise → 'mom'      (rendered "○ Mom. only")
```
This 3-tier signal is independent of Strict/Advisory mode — Strict mode changes whether a fundamental *failure* removes the stock from the results entirely; this classification runs only on survivors either way.

## 5. Presets

Three one-click filter presets (`scrPreset`), each a specific, deliberately different point in the momentum/quality tradeoff:

| Preset | ADTV | 52W gap | MA filter | EMA cross | Return | Intent |
|---|---|---|---|---|---|---|
| 🎯 Pure Rank | none | ≤20% | Price>100EMA | off | 1Y ≥7% | Closest to the raw backtested Test #1 formula — research/reference mode, not a trading signal (high turnover, see §7) |
| 📡 Early | ≥₹5Cr | ≤20% | Price>100EMA | off | 3M ≥5% | Wider net, catches momentum building before it's obvious |
| ✅ Confirmed | ≥₹10Cr | at new high only | Brutal Strength (50>100>200 stacked) | on | 1Y ≥10% | Tightest — high-conviction only |

A fourth, unlabeled "default" filter state (`scrRestoreDefaults`) exists as the initial/reset state: ADTV≥₹5Cr, 52W≤20%, 100EMA, EMA-cross on, 1Y≥7%, plus the fundamental thresholds (P/E<50, EPS>0, ROE>15%, D/E<1.0, Rev>10%) that only bite in Strict mode.

## 6. Streak tracker — separate from the ranking above

Once per screener run, the **top 30 stocks by pure `compositeRank`, filtered only to ADTV ≥ ₹5Cr** (deliberately *not* the user's active filter/preset — this is to avoid a filter change resetting streaks) are snapshotted to `localStorage` under the current calendar month, keyed `YYYY-MM`. Running the screener again in the same month overwrites that month's snapshot rather than adding a second one. History is capped at the most recent 24 months.

A stock's displayed "streak" = consecutive months (counting back from the current month, stopping at the first gap) it appeared in that top-30 snapshot. Rendered as 🔥 (3+ months) or 🔥🔥 (6+ months). This is **entirely local to the browser** — `localStorage`, not Supabase — so it does not sync across devices and is lost if browser storage is cleared. (Explicit standing instruction in the UI: "Never use `localStorage.clear()` — use the ↺ refresh-prices button instead," because that would also wipe streak history, not just the price cache.)

## 7. Backtesting (Colab notebooks, in `Backtest/`)

- **Test #1 — Pure Composite Rank**: 528 stocks, 11.2 years, *no filters at all* except the ranking formula itself. Result: CAGR 31.6% vs. 11.4% benchmark, Sharpe 1.18, max drawdown −36.9%, but 62.5% average monthly portfolio turnover across 465 unique stocks touched — i.e. this is a strong *ranking signal*, not a low-turnover, low-effort strategy as-is. Top-10-by-rank subset: Sharpe 1.25. Top-20: 66.2% win rate.
- **Test #2 — Filter Impact**: tested adding filters on top of the ranking. Volatility cap and "up days %" filters were both **catastrophic** to CAGR (−0.5% and −0.1% respectively) and were removed from the product entirely (not just defaulted off — `scrGetFilters()` hardcodes `updays: null`, and there's no volatility filter UI at all despite `volatility` being computed). 100-day EMA filter: slight positive, kept. 52W-high proximity and plain return filters: neutral, kept for other reasons (usability/conviction, not backtested edge).
- **Practical entry heuristic** (not itself backtested as a rule, just an observation from the data): 3+ consecutive months in the top-20 streak has correlated with "currently active" momentum names.
- **Planned, not run**: Test #2B (ADTV filter's own impact — needs volume data the notebooks don't yet pull in), Test #3 (which ranking dimension contributes most), Test #4 (Early vs. Confirmed preset comparison), Test #6 (exit-signal validation — i.e. actually backtesting [[wow-tracker]] §4's exit rules, which today are designed but unvalidated).

**Implication for this spec**: only the plain composite-rank formula (§2) and the two removed filters are backtest-validated. The three presets (§5), the fundamentals gate, and the exit-signal rules are designed-by-judgment, not backtested — treat them as reasonable defaults subject to revision once Tests #3/#4/#6 exist, not as settled product decisions.

## 8. Caching & performance

Fetching the full universe (500ish stocks × 2 API calls each) is slow and puts real load on the free Yahoo/Cloudflare-Worker path, so results are cached in `localStorage` (`scr_cache_v4`) with a **same-calendar-day TTL** — first run each day fetches fresh (batches of 6, 500ms apart, with retry/backoff and a progress bar), every subsequent run that day reads the cache with zero network calls. The cache key is versioned (`v4`) so a schema change to what's cached (e.g. adding `adjRet`) can force a clean refetch by bumping the constant. Cache is cleared manually via "↺ refresh prices" (§6 note on why not to use `localStorage.clear()`).

The WoW tracker's EMA-Break exit signal ([[wow-tracker]] §4) reads `ema100` out of this same cache with its own, looser freshness check (3 days, not same-day) — meaning Discovery's cache freshness and WoW's exit-signal freshness are governed by two different TTLs on the same underlying data.
