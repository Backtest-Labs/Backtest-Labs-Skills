---
name: backtestlabs-exit-engineering
description: Sets stop loss, profit target and trailing stop levels for a Backtest Labs strategy from its own trades' maximum adverse and favorable excursions, builds the exit nodes into a copy of the strategy, runs it and compares the two runs. Use when the user asks where to put a stop, how wide a target should be, whether a trailing stop helps, or how to cut losers without clipping winners. Not for diagnosing where losses come from (trade-log-forensics) or sweeping a parameter (walk-forward-parameter-selection).
---

# Exit engineering on Backtest Labs

Every closed trade in the log carries its maximum adverse excursion (the worst it went against the position) and maximum favorable excursion (the best it went in its favour). Winners and losers have different excursion distributions, and that gap is where a stop or a target belongs: a stop sits where losers go but winners rarely do; a target sits where winners get to but rarely go past. This skill measures the gap, estimates the effect, builds the exits and lets the run decide.

## Workflow

1. **Baseline.** `list_runs` for the run; when more than one run matches the strategy the user named, take the newest and say which run and date you used. `get_results` overview and `summary` section; `get_strategy` for the document and `revision` (the exits are added to a copy). Fewer than about 30 closed trades gives percentiles too coarse to set a level; say so and stop at a description.
2. **Trade table.** `get_trades`, every page, closing fills only (SELL for longs, SHORT_COVER for shorts). Save each page's CSV to a file and compute from the files where the session can. For each fill: entry cost from the row itself (`quantity` times `price` minus `realized_pnl` minus `commission`, never `realized_pnl` over `pnl_pct`, which is rounded to two decimals and is zero on a breakeven trade), MAE% and MFE% of that cost, return in percent, `holding_period_hours`, outcome (`references/mae-mfe-method.md`, formulas).
3. **Stop from MAE.** Percentiles (50, 75, 90, max) of MAE% for winners and for losers. Candidate stop levels: the winners' 75th and 90th percentile MAE%. For each candidate s, estimate the change: every trade with MAE% above s closes at -s% of entry cost instead of its return (a winner that dipped past s becomes a loss of s; a loser that fell past s stops at s). Sum the new P&L against the old; count winners lost and losers cut. Then the fragility checks, before any run: how much of the estimated gain the three largest changed trades carry; the estimate on the first and second half of the trades separately; and the same estimate in percent of entry cost, so position size does not dress up the money figure. A gain that sits in a few trades or in one half is a feature of this sample, not a level to trust. Pick the s with the best net estimate that loses the fewest winners and survives the checks; report both candidates and the checks.
   The log's MAE is the worst intraday low; the Stop Loss node fires when a **close** crosses the level and the fill is the next bar's open. So the estimate counts as stopped some winners whose low pierced the level but whose close did not, and a stopped loser exits below the level by the overnight gap. The estimate is an upper bound on the trades touched and a lower bound on the stopped loss.
4. **Target from MFE.** Percentiles of MFE% for winners. Candidate targets: the winners' 50th and 75th percentile MFE%. Estimate: every trade with MFE% at or above t closes at +t% instead of its `pnl_pct` (a loser that reached t becomes a win of t; a winner that went past t is capped at t). Sum and count.
5. **Trailing stop from give-back.** For winners, give-back = MFE% minus `pnl_pct`. A trailing distance d% closes a trade that reached MFE% above d at about MFE% minus d. Candidates: the winners' median give-back and the losers' median MFE% (the point where a loser that had been up is taken out with a gain). Estimate the same way.
6. **Choose one exit first.** The stop, unless the forensics finding was give-back, in which case the trailing stop. Adding stop, target and trail together is three changes in one run and nothing can be attributed.
7. **Build the copy.** Wire the exit per `references/exit-graphs.md`: the Buy signal into a Capture node (it stores the signal bar's close as the entry reference), a Split to feed Stop Loss and Profit Target `data_in`, Or nodes to merge with the existing signal exit, the Sell at the end. A Trailing Stop takes no reference and trails the running high on its own. Use `stop_type` / `target_type` / `trail_type` `percentage` with the level in percent of entry; the log's excursions are in the same unit. `create_strategy` with the copy named `<name> + stop <s>%` so the baseline document and run stay.
8. **Run and compare.** `run_backtest`, `get_run_status` until done, `get_results` overview and `summary` for both runs. The comparison table below. A stop replaces the signal exit rather than adding a fill, so costs rise only where the entry fires again after a stop (an IBS or RSI entry can re-enter the next bar in a falling market); read `closed_trades` and `cost_drag_pp` beside the return.
9. **Confirm it is not fitted to these trades.** The level was chosen on the same trades it is judged on. Before calling it an improvement: the copy's `hold_out` block (out-of-sample change in Sharpe and drawdown against the baseline's), and `stop_value`, `target_value` and `trail_value` are tunable keys, so walk-forward-parameter-selection can sweep the level across windows.

## Comparison table

```
                      baseline     with <exit>     change
closed trades
win rate %
payoff ratio
expectancy
profit factor
total return %        (gross %)    (gross %)
cost drag pp
max drawdown %
Sharpe / Sortino
median MFE captured (winners)
hold-out OOS Sharpe change %
```

State the estimate from step 3 to 5 beside the run's actual change; the difference is the part the estimate could not see (next-open fills, gaps, re-entries after a stop, aggregated chains).

## What the estimate ignores

- A stop fills at the next bar's open after the close that crossed the level, so a gap takes the loss past the level; a low that pierced the level without a close below it does not trigger at all.
- The exit fill is the excursion's last point, so a trade that exited on a gap at the open has its MFE or MAE at the exit price itself; a stop or target cannot reproduce such an exit.
- The Capture reference is the signal bar's close, one bar before the fill; a percentage from it differs from a percentage from the fill price by the overnight move.
- After a stop the entry condition may fire again and re-enter; the estimate keeps the original trades.
- Merged entries (several legs in a chain) share one excursion against the average entry.
- A Max Hold (`execution.holding_periods`) is a time exit that needs no node; consider it when losers are held far longer than winners.

Report what the distributions show and what the re-run changed. No verdict on whether to trade it, no sizing advice.

## References

- `references/mae-mfe-method.md`: the formulas, the percentile method, the estimate arithmetic, worked on an example.
- `references/exit-graphs.md`: the node wiring for stop, target, trailing and combined exits, with the saved JSON shape.
