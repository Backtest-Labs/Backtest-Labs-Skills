---
name: backtestlabs-backtesting-a-strategy
description: Takes a trading strategy the user describes, names, or picks from the Backtest Labs Strategy Hub, builds it as a Backtest Labs strategy document, saves it, runs the backtest and reports a first summary. Use when the user wants to backtest, test, try, build or run a strategy on Backtest Labs, or to copy a Strategy Hub strategy and run it. Not for reading a finished run in depth (reading-backtest-results), diagnosing trades (trade-log-forensics) or exporting code (porting-and-exporting-code).
---

# Backtesting a strategy on Backtest Labs

The Backtest Labs MCP server turns a strategy document into a backtest. This skill is the fixed order of calls from a description to a first summary, the defaults to choose when the user does not say, and how to recover when a save is refused. Each backtest runs every ticker on its own; there is no combined portfolio result.

## Workflow

1. **Read the account first.** Call `get_account`. It gives the plan, the most tickers and benchmarks a strategy may hold, the intraday span cap per interval, the validations the plan includes, and `daily_backtest_runtime` (cap, used, remaining, reset time). Every backtest this skill starts counts against that cap. If `remaining_seconds` is small, say so before starting.
2. **Fill the spec card** (below) from the user's words. Every value the user did not give is chosen by the defaults in `references/settings-and-defaults.md` and stated back in the summary, never left implicit. Ask only when a wrong guess would make the run useless: the interval when the strategy's wording implies one and none is given, the direction when a short sale is possible.
3. **Confirm tickers** with `search_tickers`. A stock is its symbol, a crypto pair is `BASE/QUOTE` (composite across exchanges) or `BASE/QUOTE@EXCHANGE`. A ticker with `daily_only` true runs on 1d, 1w and 1mo only. One asset class per strategy, benchmarks included.
4. **Read the node types you will use** with `get_document_reference`, at most ten types per call. It gives each field with its kind, default and bounds, the inputs with `required_to_run`, the output handles and lines, and `tunable_keys`. Do not write a config from memory; the fields differ by node and a wrong key is refused. The call with no types returns the whole skeleton, every global setting and the node catalog, a large payload: skip it when the globals you set are in `references/settings-and-defaults.md` and you leave the other sections out of the document, and make it only when you need a setting or rule that file does not carry.
5. **Build the graph** from a pattern in `references/graph-patterns.md`, adapted to the spec. The rules that bite: node ids are digit strings; a wire's `sourceHandle` is `out0` (`out` on Math, `out0` or `out1` on Split) or a line name of a multi-line node; an output takes one wire and an input takes one wire, so fan out through a Split node; inputs marked `required_to_run` must be wired; the graph needs a Buy node and a Sell node to run; no cycles. Positions may be left out and the whole graph is laid out.
6. **Pre-run checks** before saving:
   - Interval support: `vwap` and `schedule` need an interval below 1d; `dividend`, `earnings_event` and `split_event` need 1d or longer; `relative_volume` in `tod` mode needs intraday. A short sale (`sell` with direction `short`) needs `execution.margin.enabled` true.
   - Enough bars for signals: indicators are warm from the first bar because the engine fetches history before the start date, so the question is sample size, not warm-up. A 200-bar lookback on 100 daily bars gives a handful of signals at best.
   - Signal frequency: a `state` trigger fires on every bar the condition holds, which turns an entry into an every-bar buy. Use `cross` for entries unless the user wants accumulation.
   - Name: Latin letters (accents allowed), digits, spaces and standard ASCII punctuation, at most 100 characters. No em dash, no emoji. Description at most 500 characters.
7. **Save** with `create_strategy`. Read `errors` (the save was refused; fix and resend) and `run_problems` (saved as work in progress; the run is refused until fixed). `references/save-refusals.md` lists the refusals seen and the fix for each. Each error names its path, like `strategy_graph.nodes[2].data.config.period`.
8. **Run** with `run_backtest` on the saved id. The server keeps one backtest in flight per user and refuses a start within 30 seconds of the last one. The answer carries `validations_removed_by_plan`; mention any. Then loop `get_run_status` with `wait_seconds` 60 until the status is no longer `queued` or `running`. `failed` carries `failure.reason`; `cancelled` with a `stop_reason` means the daily cap stopped it; `stalled` means the process stopped answering.
9. **First pass** with `get_results` and only the run id. Check, in this order: `warnings` (dropped tickers, late-starting benchmarks, refused or skipped validations); each ticker's `coverage` (traded bars against period bars); `closed_trades` (zero trades on a legitimate setup is data, not a bug; hundreds of trades on a daily strategy means a state trigger or an unfiltered signal); `average_leverage` against the sizing you set; `cost_drag_pp` against the return; `benchmarks.<ticker>.excess_return_pp`. Then the headline values, then one line per validation that ran from its headline block under `validations`: hold-out's out-of-sample change in Sharpe and drawdown with the out-of-sample trade count; rolling window's profitable windows of total with `windows_under_trade_floor` (a window under the floor weakens the count); Monte Carlo's drawdown ladder; trade sequence's drawdown rank; cost stress's survival flags; market regime's trend spread. Monte Carlo's and trade sequence's "actual" figures sit on the closed-trade path, dividends excluded and open positions unseen, so they differ from the summary's equity and drawdown by design; do not report that as a conflict.
10. **Report** with the template below. Hand off for anything deeper: reading-backtest-results for the sections and validations, trade-log-forensics when losses cluster, robustness-audit for the full validation suite.

## Spec card

Fill every row before step 7. State the chosen value and whether the user gave it.

| Row | Source |
|---|---|
| Tickers and benchmarks | user; benchmark defaults to the ticker itself (buy and hold) |
| Asset class | from the tickers; one per strategy |
| Interval | user; see the defaults file when absent |
| Period | user; defaults by interval, within the plan's intraday span |
| Starting cash | user; 100000 by default |
| Costs | user; the skeleton's defaults otherwise (stated) |
| Sizing | user; Buy 100% of cash, Sell 100% of position by default |
| Direction | long unless the user says short; short needs margin |
| Entry, exit, filters | the strategy's own rules, one node each |
| Risk exits | stop, target, trailing only when the user names them (exit-engineering sets them from data) |
| Validations | the skeleton's six on; the four plan-gated ones off unless the user asks |

## Starting from the Strategy Hub

When the user wants a ready-made strategy: `search_hub` with the filters (query words, `nodes`, `categories`, `tickers`, `interval`, the metric bounds, `sort`); `get_hub_strategy` with the `hub_id` for the document; `create_strategy` with that document and `source_hub_id` set to the hub id, which records the copy as a fork. Change the tickers, interval or period in the copy before saving when the user asks. The Hub document carries all ten validation sections; keep them, the save switches off what the plan lacks. Hub strategies are built at the Professional plan's limits, so a copy on a smaller plan is trimmed on save: report what was trimmed.

## Changing a saved strategy

`get_strategy` for the current document and `revision`; edit the document; `update_strategy` with that revision. A refusal naming a newer revision means read again and redo the change. Change one thing per run and compare the two runs on the same tickers, period and costs. For "which period is best" questions do not hand-sweep values run by run: Walk Forward and Parameter Sensitivity do it in one run (walk-forward-parameter-selection).

## Report template

```
Strategy: <name> (id <id>, link)
Spec: <tickers> vs <benchmarks>, <interval>, <period>, cash <n>, costs <summary>, sizing <summary>
Chosen for you: <each default the user did not give>
Run: <status>; <warnings or "no warnings">; <validations removed by plan, if any>
Per ticker:
  <ticker>: return <x>% vs buy-and-hold <y>% (<excess> pp); CAGR; max drawdown; Sharpe; <n> closed trades, win rate, profit factor; cost drag <pp>
  Validations: <one line each that ran: hold-out OOS Sharpe/DD change with OOS trades; rolling <k>/<n> profitable, <m> under the trade floor; Monte Carlo P(DD > 20%); sequence DD rank; cost stress survives <flags>; regime trend spread>
Read next: <which section or skill answers the obvious question>
```

Report what the numbers show and how to read them. No verdict on whether to trade it, no advice on position size or capital.

## References

- `references/settings-and-defaults.md`: every global setting, its default and when it is checked; the period definitions; what the plan trims.
- `references/graph-patterns.md`: eight node-and-wire graphs as JSON, each built from the document reference and the node documentation.
- `references/save-refusals.md`: errors against run problems, the refusals seen in testing and the fix for each.
