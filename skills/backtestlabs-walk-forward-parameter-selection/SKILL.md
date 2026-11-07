---
name: backtestlabs-walk-forward-parameter-selection
description: Chooses parameter values for a Backtest Labs strategy with one Walk Forward and Parameter Sensitivity run instead of re-running the backtest value by value, picks the plateau rather than the best single cell, updates the strategy once and re-runs with hold-out. Use when the user asks which period, threshold or level is best, wants to optimize or tune a strategy, or asks how stable a parameter is. Not for the full validation suite (robustness-audit) or for stop and target levels from trade excursions (exit-engineering).
---

# Walk-forward parameter selection on Backtest Labs

Testing values one run at a time picks the value that fit the one period tested. Walk Forward re-optimizes on each in-sample window and scores the winners on the out-of-sample window after it; Parameter Sensitivity scores every neighbour of a value. Together they show whether a region of values works, and where its centre is. This skill runs both once, reads the region, and changes the strategy once.

## Workflow

1. **Account and strategy.** `get_account` for the daily runtime remaining and whether `walkForward` and `parameterSensitivity` are in `validations.available` (both are plan-gated; a free account with `free_runs_left` above 0 keeps them). `get_strategy` for the document and `revision`.
2. **Find the parameters.** `get_document_reference` with the node types in the graph. `tunable_keys` lists what can be swept and its range. A parameter resolves only when it shapes an output the graph reads on the way to a trade. Choose one or two: the ones the user asked about, else the entry indicator's period or threshold. List keys are swept one item at a time as `periods[0]`.
3. **Set the sweep** per `references/sweep-format.md`: Walk Forward with `selectedParams` (min, max, steps up to 5 each, inside the tunable range, never breaking the node's own rules), `mode` rolling, `numRuns` 10, `oosPercent` 20 to 30, the `objective` the user cares about (`car_mdd`, `sharpe`, `profit_factor`, `cagr`); Parameter Sensitivity with `parameterScope` `selected`, the same parameters in `selectedParams`, `variationPercent` 25, `variationsPerParam` 6, `surfaceGridSize` 5, `surfaceAxes` manual on the two parameters (or auto with one), `rankMetric` matching the objective where the enum allows; `holdOut` on. Say what the sweep costs: combinations are the product of the steps, times the windows.
4. **Save and run.** `create_strategy` with the document as a copy named `<name> sweep` (the baseline stays), or `update_strategy` with the revision when the user prefers. `run_backtest`, `get_run_status` with `wait_seconds` 60 until done.
5. **Read the region** with `references/plateau-reading.md`: the `walk_forward` section's per-window `chosen_params`, `param_stability` (values, `spread`, `max_step`, `jumpy`), `walk_forward_efficiency_pct`, `stitched_oos` against buy and hold; the `parameter_sensitivity` section's per-parameter `points` (realized value, metric, closed trades, status) and the `surface` cells. A plateau is a run of adjacent values whose metric stays within a band of the best; its centre is the value to take. The single best cell is the peak and is not the value to take.
6. **Decide once.** The value at the plateau's centre, consistent with the windows' choices (the modal or median chosen value when `jumpy` is false). When the windows disagree with the surface, the windows win: they were scored out of sample.
7. **Update and confirm.** `update_strategy` with the chosen values and the revision, Walk Forward and Sensitivity switched off, `holdOut` on. Run. Compare the new run's hold-out block (in-sample against out-of-sample Sharpe, drawdown, closed trades) with the baseline's, and quote the sweep run's WFE as the honest expectation of what re-tuning would keep.
8. **Report** with the template. One update, one confirmation run. A second round of changes is more trials that the overfitting numbers do not know about; say so if the user asks for another.

## Report template

```
Strategy: <name>; parameters swept: <node type>.<key> <min>..<max> in <steps>, <...>
Sweep run: <windows> windows, <oos>% OOS, objective <name>; <combinations> combinations
Walk forward: WFE <x>% (service passing line 50%); stitched OOS <x>% vs buy and hold <x>%; <k>/<n> OOS windows profitable
  chosen per window: <values in order>; stability: spread <x>, largest step <x>, jumpy <yes|no>
Sensitivity (<metric>): baseline <x> at <value>; points: <value: metric> ...; plateau <from>..<to> (within <band> of best <x> at <peak>)
Decision: <key> = <value> (plateau centre; windows chose <modal value> in <m> of <n>)
Confirmation run: hold-out Sharpe IS <x> / OOS <x> (baseline <x> / <x>); max DD OOS <x> (baseline <x>); OOS trades <n>
How to read it: <two sentences: what the region shows and what one confirmation cannot>
```

Report what the sweep shows. No verdict on whether to trade it, no sizing advice.

## References

- `references/sweep-format.md`: the exact `selectedParams` and `surfaceAxes` JSON for both validations, the resolution rule, the run problems a sweep can raise, and the cost in backtests.
- `references/plateau-reading.md`: how to read the walk-forward windows, the stability rows, the sensitivity points and the surface into one value.
