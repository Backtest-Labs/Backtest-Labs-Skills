---
name: backtestlabs-porting-and-exporting-code
description: Exports a completed Backtest Labs backtest as Pine Script or Python signal code with the export tool, never hand-written, and ports a Pine Script or Python strategy the user already has into a Backtest Labs node graph, runs it, and checks parity by exporting it back. Use when the user asks for Pine Script, a TradingView indicator or alert, Python code, to run a strategy outside Backtest Labs, or to bring an existing script into Backtest Labs. Not for building a strategy from prose (backtesting-a-strategy).
---

# Porting and exporting code on Backtest Labs

Two directions, one parity rule. Out: `export_code` reproduces the engine's signal logic for a completed backtest, so the code is never written by hand. In: a script's indicators and conditions map to nodes, the graph is run, and the export of that graph is diffed against the original script to find what the port changed.

## Exporting

1. **Prerequisites.** `get_account.code_export` says whether the plan includes export. The run must be a completed backtest (`list_runs`; the ten most recent are kept). The strategy may not use the portfolio nodes (`portfolio_value`, `cash_balance`, `drawdown`, `exposure`, `position_count`, `risk_per_trade`) in either language, or a percent-mode Numeric Value; Pine also refuses `candlestick` and `bars`, a Dividend or Earnings Event with an Offset Days other than 0, and a Support and Resistance wire from level 2 or higher. The tool refuses with each node and its reason; relay them and stop.
2. **Call** `export_code` with `run_id`, `language` (`pine` or `python`), `signal_types` (`both` by default: one file with both signals; `buy` and `sell` each give their own file), and for Python `include_formatter` true when the user has to shape their own data. The answer is a JSON header (strategy name, options, each file's name and length) followed by each file's full text in that order, each file its own text block. A client that joins the blocks into one string shows no marker between files: split them by the header's `characters` counts, in order.
3. **Read the data the run used** before describing the input: the `get_results` overview's `run.interval`, `run.extended_hours`, `run.adjustment_type` and `run.asset_class`, which the README's section "The data the backtest ran on" states the same way. The DataFrame answer comes from them, with `references/python-bundle.md`: the interval; the session (stock regular hours 09:30 to 16:00 ET or extended hours 08:00 to 17:00 ET; crypto on the UTC clock); for a stock run on 1h, 2h, 4h or 12h bars, that the platform counts bars from the session open and labels each with its start (a regular-hours 4h day has two bars, 09:30 and 13:30), so a resample anchored at midnight builds different bars; the price adjustment (split-adjusted, never dividend-adjusted, or as traded). The example rows in the signal file's docstring and its `__main__` demo show the frame's shape, not the interval: say so when the run is not daily.
4. **Deliver the files verbatim**, each in its own code block under its file name. Do not trim headers, comments, alert titles or the warm-up note; a test pass showed a model rewriting alert messages and dropping the usage comments. If length forces a cut, say what was left out.
5. **Relay the README's notes** in your own words: the data the run used (step 3); signals fire on the bar whose close meets the condition and the export does not shift them; the user's system places the order on the next bar; the raw signal count is higher than the backtest's trade count because the engine filters by portfolio state; the warm-up lead-in in bars or sessions the README computes for this strategy; for Pine, the script is an indicator, not a strategy, alerts are set to Once Per Bar Close, and the chart's timeframe, extended hours setting and dividend adjustment are set to match the run. With the formatter: `from_yfinance` takes a download made with `auto_adjust=False` (yfinance's default adjusts closes for dividends) and `to_session_bars` builds the 1h, 2h, 4h and 12h stock bars from 30-minute bars, which yfinance serves where it has no 4h interval. `references/python-bundle.md` and `references/pine-and-parity.md` hold the contracts.
6. **Readiness, when the user means to trade it.** Read beside the export: the run's `cost_stress` and `execution_robustness` blocks (a one-bar delay is what their system adds), the `trade_analysis` capacity figures (fill size against daily volume), and the warm-up lead-in. Say what each number implies for running the signals live; no verdict.

## Porting a script in

1. **Read the script** and list, in order: the data it reads (timeframe, source price, any second timeframe), each indicator with its parameters and source, each entry and exit condition as a comparison or crossing, position rules (long only, pyramiding, sizing), stops and targets, and anything stateful (variables carried between bars, loops, arrays).
2. **Map to nodes** with `references/node-mapping.md` and `get_document_reference` for the exact fields. Each indicator becomes a node whose output lines carry its values; each comparison becomes a Crossover (cross or state) or Relation node; `and`/`or` become And and Or nodes; a stop or target becomes a Stop Loss, Profit Target or Trailing Stop node from a captured entry. List what has no node: a second timeframe, pyramiding beyond merged entries, intrabar execution, custom loops and arrays, `varip`, order sizing by equity curve. State each as a divergence before building.
3. **Build and run** with backtesting-a-strategy: the spec card from the script's settings (timeframe, costs, sizing), the graph from the mapping, `create_strategy`, `run_backtest`, `get_results`.
4. **Export back** in the script's language and diff against the original: indicator parameters and sources; the operator and strictness of each condition (the engine's crossing is `A[t] > B[t]` and `A[t-1] <= B[t-1]`); which bar a signal fires on; the exits. Known math divergences are in `references/pine-and-parity.md`: ATR, MACD and Supertrend seed one bar earlier in Pine than TA-Lib, Volume Profile is approximated in Pine, Bollinger uses the population standard deviation, Donchian compares one bar back.
5. **Report**: the mapping table (script element, node, field values), the divergences with their effect, the run's headline, and the diff. The user's original script stays the reference for what they meant; the engine is the reference for what the export does.

## Report template

```
Direction: <export | port>
Run: <strategy>, <ticker>, <interval>, <period>
Files: <name (characters)> ...                          (export)
Mapping: <script element> -> <node type> {<fields>} ...  (port)
Divergences: <construct>: <what the port does instead, effect on signals>
Parity: <what the export-back diff matched and what it did not>
Readiness (if asked): cost stress <survives flags>; execution tolerance <x>; capacity p95 <x>% of ADV; warm-up <n bars>
How to read it: <two sentences>
```

Report what the code does and where it differs. No verdict on whether to trade it.

## References

- `references/python-bundle.md`: the Python export's files and contracts: input frame, `generate_signals` output, the formatter, requirements, warm-up, timing.
- `references/pine-and-parity.md`: the Pine export's form, alerts, the refusals per language, the divergences between TA-Lib, the engine and Pine.
- `references/node-mapping.md`: Pine `ta.*` and TA-Lib calls to node types and fields; conditions to operator nodes; what has no node.
