---
name: backtestlabs-trade-log-forensics
description: Diagnoses where a Backtest Labs strategy's edge leaks by computing, from the full trade log and the drawdown table, what no tool reports directly: exit quality from favorable excursion captured, entry quality from adverse excursion on winners against losers, the cost share of profit, loss streaks, losses by holding time and the trades behind each underwater period. Use when the user asks why a strategy lost money, what went wrong in a period, where the losses come from, whether entries or exits are the problem, or which trades caused the drawdown. Not for setting stop and target levels (exit-engineering) or for the validation suite (robustness-audit).
---

# Trade-log forensics on Backtest Labs

The trade log is the one artifact that shows every fill with its adverse and favorable excursion, R multiple, holding time and costs. Reading it in aggregate localizes a problem to entries, exits, costs, regime or sequence. The arithmetic is yours to do on the rows; nothing below is estimated from a chart.

## Workflow

1. **Get the run and its context.** `list_runs` for the run id if not given. `get_results` with the run id for the overview (closed trades, win rate, profit factor, expectancy, cost drag, max drawdown) and the `risk_anchor` method under `run` is not shown; read `portfolio.risk_anchor` from `get_strategy` when R units matter. Then the `summary` section (gross against net return, the drag lines, streaks, largest win and loss, `total_pnl`) and the `drawdowns` section (the ten deepest underwater periods with dates).
2. **Fetch the whole trade log.** `get_trades` returns 200 rows a page, ordered by position chain then time, with dividend and carrying-cost rows first. Read `pages` from page 1 and fetch every page before computing anything. Parse the CSV by the `columns` list.
3. **Build the closed-trade table** from the rows whose `action` is SELL or SHORT_COVER (each is a closing fill with `realized_pnl`, `pnl_pct`, `r_multiple`, `holding_period_hours`, `mae_value`, `mfe_value`). Pair each with the entry fills of its `chain_id` for the costs. `references/derivations.md` gives each formula; the ones used most:
   - entry cost = `realized_pnl` / (`pnl_pct` / 100); average entry price = `price` minus `realized_pnl` / `quantity` on a long
   - MAE% = |`mae_value`| / entry cost x 100; MFE% = `mfe_value` / entry cost x 100
   - initial risk = `realized_pnl` / `r_multiple`; MAE in R = |`mae_value`| / initial risk
   - MFE captured = `realized_pnl` / `mfe_value` when `mfe_value` is above 0
   - chain cost = sum of `commission` + `slippage` over every fill of the chain
4. **Compute the diagnostics** in `references/diagnostics.md`: the five groups (exits, entries, costs, time and regime, sequence), each with the numbers that define it and the reading that follows. Show the working: counts, sums and percentiles, not adjectives.
5. **Attribute the drawdowns.** For each of the deepest underwater periods, sum `realized_pnl` of the closing fills whose `timestamp` falls inside it and list the largest contributors with their MAE, MFE and holding time. A drawdown made by one chain is a sizing or a stop question; one made by twenty small losses is a regime or frequency question.
6. **Cross-check with what the run already computed.** `market_regime` under the overview's validations gives annualized return by trend and volatility regime; `trade_sequence` gives the odds of the loss streak; the `trade_distribution` charts (`get_chart`, `visuals` false) give the MAE/MFE scatter and the duration-against-payoff series as text if the user wants the shape.
7. **Report** with the template: the finding per group, the evidence, and the node-level change each finding points to. Hand off: exit levels to exit-engineering, a gate on regime to backtesting-a-strategy (Direction Filter, Volatility Regime), parameter questions to walk-forward-parameter-selection.

## What the groups mean

| Group | The symptom in numbers | Points to |
|---|---|---|
| Exits | median MFE captured below about 0.5 on winners; losers with MFE% above their final loss (they were up, then closed down) | a target or trailing exit; an earlier signal exit |
| Entries | winners' MAE% distribution close to losers'; most winners dip deep before recovering | entries early or at the wrong place; a confirmation (And with a second condition) or a cross instead of state trigger |
| Costs | chain cost over gross profit above about 0.3; slippage drag a large share of gross return; many trades near zero P&L | fewer trades (a filter), larger average move per trade, a lower participation |
| Time and regime | losses concentrated in a few months or in one regime cell; losers held much longer than winners | a regime gate; a Max Hold; a time stop |
| Sequence | the loss streak at or above the Trade Sequence p95; a drawdown made by consecutive losers of similar size | sizing against the streak, not a signal change |

A legitimate quiet stretch with no trades is data, not a bug. A zero-trade window is reported as such.

## Report template

```
Run: <strategy>, <ticker>, <interval>, <period>; <n> closed trades, <w> winners / <l> losers
Headline: return <x>% (gross <x>%), profit factor <x>, expectancy <money>, max drawdown <x>%
Exits: median MFE captured <x> (winners), <k> losers were up more than <t>% before closing down; <reading>
Entries: MAE% median winners <x> / losers <x>, 90th pct winners <x>; <reading>
Costs: chain cost <money> = <x>% of gross profit; slippage <x>% of starting equity; <n> trades within +/-<x>% of zero
Time and regime: worst months <list with P&L>; regime returns <uptrend/downtrend/vol cells>; winners held <h>h median vs losers <h>h
Sequence: longest loss streak <n> (Trade Sequence median <m>, p95 <p>); largest drawdown <x>% from <date> to <date> made by <k> chains: <top three with P&L>
Where the edge leaks: <one or two groups, with the evidence that singles them out>
Change that targets it: <node or setting, one per finding>
Not established: <what this log cannot tell>
```

Report what the rows show. No verdict on whether to trade it, no sizing advice.

## References

- `references/derivations.md`: every formula from the log's columns, with the row semantics.
- `references/diagnostics.md`: the five groups, the exact statistics for each and how to read them.
