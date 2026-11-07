# Scorecard rubric

For each validation: the fields to quote, the thresholds the service applies (quoted as the service's), the trap, and what it leaves open. Fields are from the `get_results` overview block; the full section is named where it adds rows.

## Sample

`closed_trades` and each validation's `confidence`. Below about 30 closed trades, the trade-based tests (Monte Carlo, Trade Sequence, the per-trade parts of Hold-Out) describe a handful of outcomes; Trade Sequence needs at least five. `data_sufficiency` in the summary section flags each annualized statistic. Leaves open: nothing can be concluded from a thin sample; the honest row is the count.

## Hold-Out

Quote `split_date`, `sharpe_ratio` and `max_drawdown_pct` in-sample against out-of-sample with `change_pct`, `closed_trades` per half, `buy_hold_cagr_pct` per half, `average_degradation_pct`, `risk_assessment`. The service infers risk from how far out-of-sample fell short. Trap: one split; the out-of-sample half is also one period's market. Leaves open: whether the later half was a friendlier market (compare with its buy-and-hold CAGR) and whether the result repeats across more splits (Rolling Window, Walk Forward).

## Rolling Window

Quote `profitable_windows`/`total_windows`, `windows_beating_buy_and_hold`, `average_excess_return_pp`, `worst_excess_return` with dates, `window_return_std_pct`, `trend`, `windows_under_trade_floor`, `mode`. Trap: rolling windows overlap. Leaves open: why the worst window lost (Market Regime, trade-log-forensics on that span).

## Walk Forward

Quote `walk_forward_efficiency_pct` (the service marks 50% as passing and flags above 100% as an anomaly to investigate), `stitched_oos` against `buy_and_hold_total_return_pct` on the same bars, `oos_profitable_windows`/`oos_total_windows`, each `battery` check (`combined_oos_profitable`, `wfe_pass`, `consistency`, `worst_window_drawdown`), `parameters_jumpy`, `risk_assessment`. The section's `param_stability` rows (values chosen per window, `spread`, `max_step`, `jumpy`) say whether the optimizer kept choosing the same region. Trap: a WFE above 100% means out-of-sample beat in-sample, which is luck or a trend the optimizer could not see, not strength. Leaves open: the parameter value to use (walk-forward-parameter-selection reads the same section).

## Parameter Sensitivity

Quote `metric`, `baseline_metric`, `median_metric`, `best_metric`, `worst_metric`, `most_sensitive_parameter` with `max_adverse_change`, `variations_completed`/`_requested`/`_failed`. The service ranks parameters by the largest drop any single change produced, in the metric's units. Trap: one-at-a-time; interactions are Overfitting's and Walk Forward's business. A `most_sensitive_parameter` of null with the note that nothing scored below baseline means the set values are a local minimum of the metric, which is its own warning. Leaves open: whether the set value sits on a plateau or a peak (the section's per-parameter points and surface cells).

## Overfitting

Quote `pbo_pct` (robust to 20, borderline 20 to 50, likely overfit from 50), `probability_oos_loss_pct`, `deflated_sharpe_ratio_pct` (survives deflation from 95, inconclusive 50 to 95, plausibly luck below 50), `probabilistic_sharpe_ratio_pct`, `baseline_sharpe_annualized` against `expected_max_sharpe_annualized`, `degradation_slope`, `trials_executed`, `min_track_record_met`, `min_backtest_length_met`, `risk_assessment`. Trap: the luck benchmark is this pool only; every change the user makes after reading it is a trial the number does not count. Leaves open: nothing about path risk or costs.

## Monte Carlo

Quote `final_equity_percentile_rank`, the CI bounds, `actual_max_drawdown_pct` against `median_max_drawdown_pct` and `max_drawdown_p5_pct`, the `probability_max_drawdown_over_N_pct` ladder. Trap: resampling the run's own trades; it cannot produce a loss the strategy never took. Leaves open: whether the trades themselves are representative (the time-based tests).

## Trade Sequence

Quote `max_drawdown_percentile_rank`, `longest_losing_streak` and `longest_time_underwater` (actual, median, p95, rank). Trap: same trades, different order; an actual drawdown at a low rank means a kind order and the p95 is the plan-for number. Leaves open: sizing, which is the user's.

## Cost Stress

Quote `survives_1_5x`, `_2x`, `_3x`, `breakeven_cost_multiplier` (or its note), `net_profit_at_2x` and `_3x` against `baseline_net_profit`, `profit_change_per_1x_cost_pct`, `cost_sensitivity`, `stress_tolerance`. Trap: survival at 3x with a steep `profit_change_per_1x_cost_pct` is survival on a shrinking margin. Leaves open: whether the base costs are realistic for the user's venue (the trade analysis section's capacity figures).

## Execution Robustness

Quote `tolerance_score` (100 means no effect; null with a note when the baseline Sharpe is about 0), `baseline_sharpe`, `avg_delayed_sharpe`, `worst_sharpe` with `worst_run_label`, `entry_sensitivity` against `exit_sensitivity`, `significance` and its p-value, `risk_level`. Trap: the delay is in bars; read it against the holding time (the section's `avg_hold_bars`). A strategy holding two bars with a one-bar delay has lost half its trade. Leaves open: whether the user's own execution is one bar late or two.

## Market Regime

Quote `annualized_return_pct_by_regime` for the five regimes and `annualized_return_spread_pp` per axis; the section adds per-cell Sharpe, closed-trade win rates and the benchmarks in the same buckets. Trap: a long-only strategy losing in the downtrend regime is the market; the question is the benchmark's loss in the same cell. Leaves open: whether a gate on the losing regime would have kept the gains (build the gated variant, one change).

## Cross-reads

| Pair | What the combination says |
|---|---|
| Hold-Out fine, PBO high | The set values held in the later half, but so would many neighbours; the result is a region, not these numbers. Read Sensitivity's plateau. |
| WFE passing, `parameters_jumpy` true | Re-optimizing worked by chasing; the user cannot pick one value from it. Walk-forward-parameter-selection or a wider plateau. |
| Cost Stress survives 3x, Execution tolerance low, short holds | The profit is in the first bar after the signal; costs do not kill it but a late fill does. Execution is the risk, not fees. |
| Monte Carlo rank near 50, Trade Sequence drawdown rank low | The outcome is typical of its trades; the drawdown was a kind order. Plan for the p95 drawdown. |
| Rolling `trend` weakening, Walk Forward later windows losing | The edge is fading through the period, in both tests. Regime timeline shows where. |
| Regime spread large, Rolling consistency low | The strategy works in one condition and the windows without it lose. A gate is the test, not a verdict. |
| Sensitivity worst far below baseline, Overfitting DSR high | The Sharpe survives the luck benchmark but a nearby setting breaks it; the pool was built around these values. |
