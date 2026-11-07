# MAE and MFE method

From each closing fill (SELL or SHORT_COVER row of `get_trades`):

```
entry_cost = realized_pnl / (pnl_pct / 100)
MAE_pct    = |mae_value| / entry_cost * 100        (mae_value is stored negative; 0 when the trade never went against)
MFE_pct    = mfe_value / entry_cost * 100
outcome    = winner if realized_pnl > 0, loser if < 0
```

MAE and MFE are measured in money against the weighted average entry of the units closed, from every held bar's high and low with the exit fill as the last point, so the realized move always lies inside the excursion: an exit on a favorable gap at the open sets MFE to the exit itself, and an adverse gap sets MAE. A chain with several entry legs has one excursion pair per closing fill.

## Percentiles

Sort each group's values; the p-th percentile is the value at rank ceil(p/100 x n). Report with n.

| Group | 50 | 75 | 90 | max | n |
|---|---|---|---|---|---|
| winners MAE% | | | | | |
| losers MAE% | | | | | |
| winners MFE% | | | | | |
| losers MFE% | | | | | |
| winners give-back (MFE% - pnl%) | | | | | |

The gap to look for: winners' 90th percentile MAE% below losers' 50th. A stop between them cuts over half the losers early and touches a tenth of the winners. When the two overlap, no stop separates them and the finding belongs to entries.

## Estimating a stop at s%

For each closed trade:

```
new_pnl_pct = -s            if MAE_pct > s
            = pnl_pct       otherwise
new_pnl     = new_pnl_pct / 100 * entry_cost
```

Sum `new_pnl - realized_pnl` over all trades. Count winners with MAE_pct > s (winners lost) and losers with MAE_pct > s (losers cut). Each stopped trade also pays one more fill's slippage, approximated by the mean slippage of the log's exit fills.

## Estimating a target at t%

```
new_pnl_pct = +t            if MFE_pct >= t
            = pnl_pct       otherwise
```

Count losers with MFE_pct >= t (losers turned) and winners with pnl_pct > t (winners capped, their excess `pnl_pct - t` forgone).

## Estimating a trailing stop at d%

```
new_pnl_pct = MFE_pct - d   if MFE_pct > d and (MFE_pct - d) > pnl_pct
            = pnl_pct       otherwise
```

This is an upper bound, not an estimate like the other two: it assumes the best price came before any pull-back of d, and it leaves untouched the trades that a trailing level would have stopped before they reached their best. The log holds each trade's extremes, not its path, so the order of the high and the low is unknown. Quote it as "at most", and let the run set the number.

## Example (daily MSFT mean reversion, 146 closed trades, 96 winners, 50 losers)

Computed from the run's own log with the formulas above.

| Group | p50 | p75 | p90 | max | n |
|---|---|---|---|---|---|
| winners MAE% | 0.86 | 1.51 | 2.56 | 7.39 | 96 |
| losers MAE% | 3.92 | 7.03 | 9.92 | 14.35 | 50 |
| winners MFE% | 2.26 | 3.30 | 4.40 | 21.94 | 96 |
| losers MFE% | 0.39 | 0.96 | 1.99 | 7.75 | 50 |
| winners give-back | 0.31 | 0.66 | 1.24 | 3.23 | 96 |

The gap: winners' 90th percentile MAE is 2.56%, losers' median is 3.92%. Winners' median MFE captured is 0.87 (25th percentile 0.57), so exits are not the leak; losers held twice as long as winners (median 144 against 72 hours). Eleven of fifty losers had been up more than 1%.

Stop estimates (closed-trade P&L 11,086):

| stop | new P&L | change | winners lost | losers cut |
|---|---|---|---|---|
| 2.0% | 8,910 | -2,175 | 17 | 39 |
| 3.0% | 12,733 | +1,647 | 4 | 30 |
| 3.5% | 12,932 | +1,846 | 2 | 27 |
| 4.0% | 12,164 | +1,078 | 2 | 22 |
| 5.0% | 10,931 | -154 | 1 | 18 |

The 3.5% stop, between the winners' 90th percentile and the losers' median, is the candidate: two winners lost, 27 losers cut early, the largest estimated gain. The 2% stop sits inside the winners' distribution and costs more than it saves.

Target estimates: 2.0% caps 39 winners and turns 5 losers, change -5,425; 3.0% change -2,101; 4.0% change -7 (10 capped, 3 turned). On a strategy whose winners are small and frequent, and whose best exits land on gaps at the open, a target below the winners' 90th percentile MFE costs more than it saves; the estimate says so before a run is spent.

Trailing upper bounds: 1% at most +9,383 (21 trades changed), 2% at most +5,879 (8), 3% at most +4,499 (4). These are ceilings; the path is unknown.

The order to run: the 3.5% stop first (one change), then the hold-out comparison, then the trailing stop as a second variant only if the user wants it.
