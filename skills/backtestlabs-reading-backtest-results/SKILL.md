---
name: backtestlabs-reading-backtest-results
description: Reads a completed Backtest Labs backtest in a fixed order, validity first, into a review, or explains one metric, validation, chart or result the user asks about. Use when the user asks how a backtest did, what a number means, whether a result holds up, to compare a strategy with buy and hold, or to look at a chart. Not for building a strategy (backtesting-a-strategy), diagnosing individual trades (trade-log-forensics) or the full validation suite (robustness-audit).
---

# Reading backtest results on Backtest Labs

A completed run answers through four tools: `get_results` (overview, then sections per ticker), `get_trades` (the trade log), `get_chart` (charts as text or images) and `read_docs` (the documentation). Every answer carries a `units` legend defining each value name; read it before interpreting a number. Percentages are percent, drawdowns are positive distances below the peak, money is in the run's quote currency, and a null value comes with a note saying why. Each ticker is its own backtest with its own cash; nothing is combined across tickers.

## Mode 1: review a run

Follow the order. Each step is one call; stop early when the user's question is answered.

1. **Find the run.** `list_runs` lists the ten most recent completed runs, newest first, with per-ticker headline stats. Use the run the user names, else the newest. Older runs are gone.
2. **Validity gate** with `get_results` and the run id only. Before any metric, read `warnings` (dropped tickers, benchmarks that failed or started late, validations refused or skipped and why), each ticker's `coverage` (traded bars against period bars), and under `run.validations` which validations ran for which tickers. A late-starting benchmark makes the excess return a comparison over different periods; say so.
3. **Headline per ticker** from the overview: total return against the benchmark's (`excess_return_pp`), CAGR, max drawdown and its longest stretch, Sharpe, Sortino, Calmar, closed trades, win rate, profit factor, payoff ratio, expectancy, cost drag, average leverage, final equity.
4. **The summary section** (`section` `summary`) for what the overview leaves out: gross against net return and the four drag lines (commission, slippage, borrow, interest); winning, losing and breakeven trades, average and largest win and loss, streaks, SQN, P&L standard deviation; `data_sufficiency`, which says whether enough bars exist for each annualized statistic; the drawdown distribution (mean, median and 95th percentile depth and duration); dividends; turnover; margin calls.
5. **Periodic returns** for concentration: which years and months carried the return, how many lost, whether one month holds most of the P&L.
6. **Drawdowns** for the ten deepest underwater periods with dates, depth, duration and whether one is ongoing.
7. **Trade analysis** for holding periods (hours, first entry to final exit) and capacity (fill size against the ticker's average daily volume, or 24-hour traded value for crypto).
   **Then the trade concentration**, from the summary's `largest_win` against `total_pnl` and, when the largest win is a large share, from `get_trades`: sum the closing fills' `realized_pnl` in descending order and report how many trades make the whole profit. Periodic returns show concentration in time; only the log shows it in trades.
8. **Validations** that ran, each from its overview block first (`references/validations.md` says what each shows and its trap). Fetch a validation's full section only when the question needs its rows (windows, regime cells, sweep points).
9. **Charts** only when a shape answers the question better than the numbers: call `get_chart` with the ticker alone to list pages and chart ids, then one page or one chart. Prefer `visuals` false, which returns the plotted series as text. Each image costs about 1,250 to 1,600 tokens and a page returns up to six.

## Mode 2: explain

"What does Sharpe mean here", "why is profit factor null", "what is walk-forward efficiency": answer from `references/metrics.md` and `references/validations.md`, in the run's own numbers when a run is in the conversation. For a node, a chart's construction or an engine rule (fills, costs, dividends, data), call `read_docs` with the page id from `references/docs-map.md` and answer from the page; the documentation is the reference and this skill does not restate it. Engine facts that change how a result reads are summarized in `references/engine-rules.md`.

## Traps

- **Annualized statistics on short or intraday runs.** `data_sufficiency` in the summary section says which are stable. Volatility and Sharpe on 4h bars are annualized by bars per day times the base; compare only runs on the same interval.
- **Sharpe against the trade count.** Sharpe is computed from bar returns and `data_sufficiency` judges that sample; it can be stable on hundreds of bars while a handful of trades decided the sign. Whether the trades support the expectancy is `sqn`'s question (the t-statistic of the average trade); quote the closed-trade count and SQN beside the Sharpe.
- **Win rate against expectancy.** A 65% win rate with a payoff ratio below 1 can still lose; expectancy (average P&L per closed trade) and profit factor carry the sign.
- **Profit factor carried by one trade.** Compare `largest_win` with `total_pnl`; a run whose largest win is most of its P&L is one trade, not an edge. The 999.99 cap and the null with note mean no losing trade.
- **One year carrying the return.** Periodic returns with one strong year and flat or negative others is a single regime, not a strategy that works across them.
- **A few trades carrying the return.** When the top few closing fills sum to all of `total_pnl` or more, the rest of the trades lost money together; say how many trades the profit rests on.
- **Hold-out halves against the whole.** Each half is a separate backtest from a flat start, so a trade open across the split is cut and never re-entered; the whole run's CAGR can differ from both halves'. Check the log for a position open on `split_date` before reading the gap as degradation.
- **Monte Carlo and Trade Sequence actuals.** Their "actual" figures sit on the closed-trade step path, dividends excluded and open-position swings unseen, so they differ from the summary's equity and drawdown by design.
- **Exposure against time in market.** `average_leverage` of 0.4 means the account was 40% invested on average; a return of 20% on 40% exposure is a different edge than 20% fully invested. The exposure chart shows when.
- **Benchmarks from the same starting cash.** `final_value` under a benchmark is buy and hold from the same cash on the same period; a late-starting benchmark has a shorter period, flagged in `warnings`.
- **Drawdown in money against percent.** `max_drawdown_value` can fall on another bar than `max_drawdown_pct`; quote the one the question asks for.

## Hand-off map

| What the numbers show | Skill to run next |
|---|---|
| Losses cluster in time; a few underwater periods hold most of the drawdown; large MAE on winners | trade-log-forensics |
| Winners give back most of their favorable excursion; exits by signal only | exit-engineering |
| Good headline numbers, few validations ran, or the user asks "does it hold up" | robustness-audit |
| Parameter sensitivity spikes; "which period is best" | walk-forward-parameter-selection |
| The user wants to run it elsewhere | porting-and-exporting-code |
| A metric looks wrong against the trade count or costs | check the summary section's gross against net and the drag lines first |

## Report template (review mode)

```
Run: <strategy name>, <tickers> vs <benchmarks>, <interval>, <period>; <warnings or "no warnings">
Validity: coverage <traded/period bars per ticker>; validations that ran: <list>; skipped: <list with reason>
<ticker>:
  Return <x>% (buy and hold <y>%, <excess> pp); CAGR <x>%; max drawdown <x>% (<days> days, <ongoing?>)
  Sharpe <x> / Sortino <x> / Calmar <x>; volatility <x>%; <data sufficiency notes>
  <n> closed trades: win rate <x>%, profit factor <x>, payoff <x>, expectancy <money>, SQN <x>; streaks <w>/<l>
  Costs: gross <x>% to net <x>%; <largest drag line>
  Concentration: <best year share of P&L>, <losing years>; <top n trades = x% of total P&L>
  Validations: <one line each, from validations.md>
How to read it: <two or three sentences on what these numbers do and do not establish>
Open questions: <what a next skill would answer>
```

Report what the numbers show and how to read them. No verdict on whether to trade it and no advice on size or capital.

## References

- `references/metrics.md`: every summary value, what it shows, how to read it, the trap.
- `references/validations.md`: the ten validations, the question each answers, the fields to read, the thresholds the service uses, the trap.
- `references/charts.md`: which chart answers which question; the token rules.
- `references/engine-rules.md`: engine facts that change how a result reads.
- `references/docs-map.md`: topic to `read_docs` page id.
