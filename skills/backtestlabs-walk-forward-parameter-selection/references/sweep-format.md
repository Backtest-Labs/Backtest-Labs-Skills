# Sweep format

Both validations name a parameter by the node's `id` in the graph and a key from the node type's `tunable_keys` (`get_document_reference` with the type). A list key names one item as `key[index]`, like `periods[0]` for the first EMA period.

## Walk Forward

```json
"walkForward": {
  "enabled": true,
  "mode": "rolling",
  "numRuns": 10,
  "oosPercent": 25,
  "objective": "car_mdd",
  "selectedParams": [
    {"node_id": "0", "param": "periods[0]", "min": 10, "max": 30, "steps": 5},
    {"node_id": "0", "param": "periods[1]", "min": 40, "max": 80, "steps": 5}
  ],
  "exportTrades": false
}
```

- `selectedParams`: at least one, at most two, each parameter once. `min` no greater than `max`, both inside the key's tunable range. `steps` 1 to 5: the values tried from min to max (1 tries min alone). The check fills in a `label`.
- Every value tried, with the node's other settings, must keep the node's rules between its own fields: a crossover's fast period below its slow one for every pair of swept values, oversold below overbought. A sweep that breaks one is a run problem naming it.
- `mode` `rolling`: `numRuns` 10 to 25; `anchored`: 10 to 12 (the in-sample always starts at the beginning). `oosPercent` 10 to 40.
- `objective`: `car_mdd` (CAGR over max drawdown), `sharpe`, `profit_factor`, `cagr`.
- Backtests: combinations (product of the steps) x `numRuns` in-sample fits, plus the out-of-sample runs.

## Parameter Sensitivity

```json
"parameterSensitivity": {
  "enabled": true,
  "variationPercent": 25,
  "variationsPerParam": 6,
  "parameterScope": "selected",
  "selectedParams": [
    {"node_id": "0", "param": "periods[0]"},
    {"node_id": "0", "param": "periods[1]"}
  ],
  "surfaceGridSize": 5,
  "surfaceAxes": {"mode": "manual", "params": [{"node_id": "0", "param": "periods[0]"}, {"node_id": "0", "param": "periods[1]"}]},
  "rankMetric": "sharpe",
  "exportTrades": false
}
```

- `parameterScope` `selected` needs at least one entry; `all` sweeps every parameter that resolves and needs an empty list.
- `variationPercent` 5 to 50: how far each parameter is moved above and below its set value; above 25 the service calls it a broad stress rather than a neighbourhood. `variationsPerParam` 2, 4 or 6 levels.
- `surfaceAxes` `manual` names two different swept parameters; `auto` takes the two most sensitive after ranking. `surfaceGridSize` 3 or 5.
- `rankMetric`: `sharpe`, `sortino`, `profit_factor`, `total_return`. The Walk Forward objective `car_mdd` has no Sensitivity counterpart; use `sharpe` beside it.
- A requested step is snapped to the nearest legal value (whole numbers for periods, inside the node's range); two steps landing on one value are tested once; a step landing on the set value or outside the range is reported and not run.

## Resolution

A parameter resolves only while its key takes part in the node's current settings and the graph reads, on the way to a trade, an output the key shapes. An RSI's `overbought` while the node's direction is `bullish` does not resolve (nothing reads it); an indicator's threshold whose values feed only a Math node does not. A parameter that does not resolve is a run problem for both validations.

## Run problems a sweep raises

- A parameter entry that does not resolve.
- A Walk Forward range leaving the key's tunable range.
- A sweep that would try values breaking the node's own rules (fast at or above slow; oversold at or above overbought; a stop percentage of 100 or more).
- `surfaceAxes` params that are not among the swept parameters, or the same parameter twice.

`run_backtest` refuses the strategy and names each; fix the range and `update_strategy` with the current revision.
