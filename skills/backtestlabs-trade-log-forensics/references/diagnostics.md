# Diagnostics

Compute each statistic on the closed-trade table from `derivations.md`. Winners: `realized_pnl` above 0. Losers: below 0. Report n beside every percentile.

## 1. Exits (did the trade keep what it had)

| Statistic | How | Reading |
|---|---|---|
| MFE captured, winners: median, 25th pct | `realized_pnl / mfe_value` | median below about 0.5: winners gave back half their best open profit; the exit is late or absent |
| Give-back, winners: median `MFE_pct - pnl_pct` | | in percent of entry, what a trailing exit would have to recover |
| Losers that were up: count and share with `mfe_value` above 0.5 x `initial_risk` (or MFE_pct above 1% when no R) | | these were winners for a while; a target or trailing stop turns some into wins |
| Losers' final loss against their MFE: sum of `mfe_value` over losers | | the open profit the strategy held and lost |

A target set from these numbers is exit-engineering's job; here the finding is that the exit, not the entry, leaks.

## 2. Entries (was the trade right at the start)

| Statistic | How | Reading |
|---|---|---|
| MAE_pct percentiles, winners against losers (50, 75, 90) | | when winners' 90th pct is far below losers' median, a stop between them cuts losers without clipping winners (exit-engineering); when the two distributions overlap, entries do not separate good trades from bad and the signal itself is the problem |
| Share of winners with MAE_R above 1 | | winners that first lost a full R: the entry fires early |
| Immediate losers: losers with `mfe_value` 0 or near 0 | | the trade never went in its favour: wrong direction or wrong moment |
| Winners' MAE at their best: compare MAE_pct of the top quartile of winners by `pnl_pct` with the rest | | the biggest winners also dipping deep means the entry timing is loose everywhere |

## 3. Costs (is the edge paying for itself)

| Statistic | How | Reading |
|---|---|---|
| `cost_over_gross` | total cost over gross profit | above about 0.3 the strategy works for the broker; the summary's `slippage_drag_pct` and `commission_drag_pct` give the same in share of starting equity |
| Near-zero trades: count with abs(`pnl_pct`) below 2 x (chain cost / entry cost x 100) | | trades whose move was smaller than twice their cost: frequency without edge |
| Cost per chain against `expectancy` | mean chain cost against the summary's `expectancy` | cost a large fraction of expectancy means a small shift in slippage flips the sign; Cost Stress's `profit_change_per_1x_cost_pct` says by how much |
| Legs per chain | | many legs with small quantities means repeated entries paying the half-spread each time |

## 4. Time and regime (when does it lose)

| Statistic | How | Reading |
|---|---|---|
| P&L by calendar month and year | sum `realized_pnl` by month of exit | with `periodic_returns`: the months that hold most of the loss |
| P&L by exit weekday | | a weekday effect on daily data is usually noise unless the strategy is calendar-based |
| Holding time: median `holding_period_hours`, winners against losers | | losers held much longer than winners means the strategy waits for a signal that comes late when wrong: a Max Hold or a stop |
| Loss by holding bucket | bucket by hours (one bar, up to 5 bars, up to 20, longer, in the run's interval) and sum losses | which duration carries the losses |
| Regime | overview `market_regime.annualized_return_pct_by_regime` | a loss concentrated in one cell is a gate question (Direction Filter on a trend series, Volatility Regime) |
| Underwater periods | `drawdowns` section; sum `realized_pnl` of closing fills inside each | few chains making the drawdown: sizing or stop; many: regime or frequency |

## 5. Sequence (was it the order)

| Statistic | How | Reading |
|---|---|---|
| Longest loss streak | consecutive losers in `trade_id` order | against `trade_sequence.longest_losing_streak` (actual, median, p95): a streak at the p95 was unlucky ordering; a streak at the median is the strategy's habit |
| Drawdown rank | `trade_sequence.max_drawdown_percentile_rank` | low rank means most orderings drew down less: order luck, not structure |
| Streak P&L | sum of the streak's losses against `max_drawdown_value` | whether the worst streak is the worst drawdown |

## Writing the finding

One group leads. State the statistic, its value with n, the comparison that singles it out, and the node or setting it points to:

- Exits leak: Profit Target or Trailing Stop from MFE (exit-engineering), or an earlier signal exit.
- Entries loose: a confirming condition through And, a `cross` trigger in place of `state`, a Direction Filter on a trend series.
- Costs: fewer, larger trades; a filter that removes the near-zero trades; a lower `volume_participation_threshold` is a model setting, not a fix.
- Time and regime: a regime gate; `holding_periods` Max Hold for the long losers.
- Sequence: sizing (`amount_value`, `reserved_cash_value`), not signal changes.
