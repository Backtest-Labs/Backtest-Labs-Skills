# Audit settings

The `validation` section as saved for the audit. A present section carries all ten validations with every key, enabled or not. Values keep to the ranges in the document reference's `validation_settings`.

```json
"validation": {
  "monteCarlo":           {"enabled": true,  "simulations": 1000, "confidenceLevel": 0.95},
  "tradeSequence":        {"enabled": true,  "permutations": 1000},
  "marketRegime":         {"enabled": true},
  "holdOut":              {"enabled": true,  "splitRatio": 0.5, "exportTrades": false},
  "rollingWindow":        {"enabled": true,  "numWindows": 5, "mode": "non_overlapping", "exportTrades": false},
  "costStress":           {"enabled": true},
  "executionRobustness":  {"enabled": true,  "exportTrades": false},
  "parameterSensitivity": {"enabled": true,  "variationPercent": 15, "variationsPerParam": 4, "parameterScope": "all",
                           "selectedParams": [], "surfaceGridSize": 3, "surfaceAxes": {"mode": "auto"},
                           "rankMetric": "sharpe", "exportTrades": false},
  "overfitting":          {"enabled": true,  "nTrials": 50, "spreadPercent": 20, "cscvSlices": 16,
                           "rankMetric": "sharpe", "exportTrades": false},
  "walkForward":          {"enabled": true,  "mode": "rolling", "numRuns": 10, "oosPercent": 20, "objective": "car_mdd",
                           "selectedParams": [ {"node_id": "<id>", "param": "<key>", "min": <v>, "max": <v>, "steps": 5} ],
                           "exportTrades": false}
}
```

## Choosing the Walk Forward parameters

- One or two parameters, each a `tunable_keys` entry of its node (`get_document_reference` with the type). A list key is named one item at a time: `periods[0]`, `periods[1]`.
- The key must shape an output the graph reads on the way to a trade; a key that only changes a drawn line does not resolve and is a run problem.
- `min` and `max` inside the key's tunable range; `steps` 1 to 5. Every value tried, with the node's other settings, must keep the node's own rules: a crossover's fast period stays below its slow one, oversold below overbought. A sweep that would break one is a run problem.
- Prefer the parameter the strategy's logic turns on: the period of the entry indicator, the threshold of the entry oscillator, a stop or target value.
- Combinations are the product of the steps; two parameters at five steps are 25 combinations per window, on 10 windows 250 backtests. Mention the count with the daily cap; the wall-clock estimate is `run_backtest`'s own `estimate` label, not a figure of yours.

## Overfitting and Sensitivity

Overfitting varies every nonzero tunable parameter together within `spreadPercent` and runs `nTrials` variants over the full period; 50 trials at 20% is the audit default. Sensitivity moves each parameter on its own; `parameterScope` `all` tests every parameter that resolves. Leave it at `all`: a parameter whose set value is 0 (a Price node's lag, an offset) is reported as invalid with the reason that a relative variation of zero is undefined, it costs no backtest, and switching to `selected` only to skip it narrows the test and makes the result a selected-parameter sensitivity. Use `selected` only when the user names the parameters.

The ranking metric is Sharpe for Sensitivity and Overfitting, and the Walk Forward objective is `car_mdd` (annual return over max drawdown), because `car_mdd` is not offered as an Overfitting ranking metric: drawdown depends on the bar order the cross-validation splits do not keep. So the three cannot share one metric; say so in the scorecard. When the user names a metric that all three offer (`sharpe`, `profit_factor`), use it for all three. A saved strategy's own choices win over these defaults.

## Plan gating

`get_account.validations.available` lists what runs. A validation the plan lacks is switched off on save and named in `run_backtest.validations_removed_by_plan`; a free account with `free_runs_left` above 0 keeps every validation. The scorecard lists the validations that did not run with the reason.

## Rolling window mode

`non_overlapping` is the stricter reading (one good stretch lifts one window only); `rolling` is the editor's default and what a saved strategy usually carries. Keep the saved mode unless the user asks for the stricter one, and say which one the scorecard used.

## Saved settings win

The JSON above is what to write when a validation has no saved settings. A saved strategy's `validation` section is the starting point: switch each validation on, keep its saved values, and report every value the audit changed and why.
