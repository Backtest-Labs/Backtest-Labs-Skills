# Reading the region

## Walk Forward section (`get_results`, section `walk_forward`)

Rows under `windows`, one per cycle: `is_start`, `is_end`, `oos_start`, `oos_end`, `chosen_params` ("label=value; label=value"), `is_return_pct`, `is_sharpe`, `oos_return_pct`, `oos_sharpe`, `oos_max_drawdown_pct`, `oos_fills`, `wfe_pct` (undefined when the window's in-sample return is not positive), `is_carried_forward`.

`param_stability`, one row per parameter: `label`, `sweep_min`, `sweep_max`, `values` (the value chosen in each window, in order), `spread`, `max_step` (the largest jump between consecutive windows), `jumpy`.

Reading:

1. List the chosen values in window order. A value chosen in most windows, with neighbours chosen in the rest, is a region. Values spread across the whole sweep range with `jumpy` true mean the optimizer found nothing stable; no single value is supported and the honest answer is that.
2. Per-window `wfe_pct` sliding downhill across windows means the edge is fading, whatever the overall WFE.
3. `oos_fills` low in a window makes that window's return a coin flip; weight it accordingly.
4. The overview's `walk_forward_efficiency_pct` is annualized out-of-sample return over annualized in-sample return across the stitched track; the service's passing line is 50%, and above 100% is flagged as an anomaly. `stitched_oos` against `buy_and_hold_total_return_pct` on the same bars is the live-like comparison.

## Parameter Sensitivity section (section `parameter_sensitivity`)

`parameters`, one per swept parameter: `rank`, `baseline_value`, `variation_statistics` (`realized_min`, `realized_max`, `median`, `best`, `worst`, `range`, `median_absolute_deviation`, `max_adverse_delta`, `max_favorable_delta`, `completed_points`), and `points`: each with `requested_variation_pct`, `realized_value`, `metric_value`, `closed_trades`, `status` (completed, unchanged value, invalid, no trades, failed) and `reason`.

`surface`: `param1` and `param2` with their `values`, and `cells` with `p1`, `p2`, `metric_value`, `status`.

Reading a parameter's points:

1. Order the completed points by `realized_value` with the baseline in place. Write the metric beside each.
2. Set the band: within 10% of the best metric for Sharpe-like metrics, or within the metric's own noise when `closed_trades` is small (a point with far fewer closed trades than its neighbours is a different strategy, not a better setting).
3. The plateau is the longest run of adjacent values inside the band. Its centre (the median value of the run) is the candidate. A single point above the band with both neighbours far below is a peak: not the candidate.
4. A curve that rises to the edge of the tested range means the region may continue past it; say so rather than extrapolating.

Reading the surface:

1. Lay the cells out as a grid, p1 across, p2 down, metric in each.
2. The region is the largest connected block of cells inside the band; its centre cell gives both values at once. Diagonal structure (the metric depends on the ratio of the two) means the two parameters are not independent; choose on the ratio.
3. Cells with a status other than completed are holes, not zeros.

## Combining the two

| Walk forward windows | Sensitivity plateau | Decision |
|---|---|---|
| Stable, modal value inside the plateau | Present | The plateau centre; it agrees with what was chosen out of sample. |
| Stable, modal value outside the plateau | Present | The windows' value: it was scored out of sample. Note that the full-period surface disagrees. |
| Jumpy | Present | The plateau centre, with the note that re-tuning would not have found it reliably; expect less than the in-sample metric. |
| Jumpy | Absent (a peak or a slope) | No supported value. Report the result; the strategy's logic, not its number, is the question. |

The decision is one value per parameter, written into the document once, then confirmed with a hold-out run. The sweep run's WFE is the expectation to quote for how much of the in-sample performance survives.
