# Validations

Each validation that ran appears as an overview block under `tickers.<ticker>.validations` in `get_results`, and in full as a section (`monte_carlo`, `trade_sequence`, `hold_out`, `market_regime`, `rolling_window`, `cost_stress`, `execution_robustness`, `parameter_sensitivity`, `walk_forward`, `overfitting`). A validation that did not run shows its status and reason instead. `confidence` (high, medium, low) is the service's reliability rating from sample size; `risk_assessment` or `risk_level` is its own banding. Quote both as the service's ratings, not as yours.

## Monte Carlo (`monte_carlo`)

Resamples the closed trades with replacement into many equity paths. Answers: how wide is the range of outcomes these trades could have produced, and how deep could drawdowns get.

Read: `actual_final_equity` against `median_final_equity` and the CI (`final_equity_ci_lower`, `_upper`, `ci_level_pct`); `final_equity_percentile_rank` (50 means the run sits at the median of its own resamples); `actual_max_drawdown_pct` against `median_max_drawdown_pct` and `max_drawdown_p5_pct` (5% of paths drew down deeper); the `probability_max_drawdown_over_N_pct` ladder (10 to 50).

Trap: the paths are built from the run's own closed-trade P&L; they say nothing about trades the strategy never took. Few trades give a wide CI by construction.

The "actual" figures are on that same closed-trade path, not the run's bar-by-bar equity: `actual_final_equity` is starting cash plus the sum of closed-trade P&L, with dividends left out (the section's `excluded_dividends` gives their count and total), and `actual_max_drawdown_pct` is the deepest dip of that step path, which never sees an open position's swings. So they differ from the summary's `final_equity` (by the dividends) and `max_drawdown_pct` (which is usually deeper). That is the method, not a discrepancy; compare the actual figures with the simulated ones beside them, not with the summary.

## Trade Sequence (`trade_sequence`)

Reorders the same closed trades without replacement, so every path ends at the same equity and only the order differs. Answers: how much of the drawdown, the losing streak and the time underwater was the luck of the order. Needs at least five closing trades.

Read: `max_drawdown_percentile_rank` (higher is milder than most orderings), `longest_losing_streak` and `longest_time_underwater` each with actual, median, p95 and percentile rank.

Trap: a run whose actual drawdown ranks near the p95 of its own orderings got a kind order; expect worse. The actual values are measured on the closed-trade step path, as in Monte Carlo (dividends excluded, open-position swings unseen), so they differ from the summary's drawdown and streaks measured on bar equity.

## Hold-Out (`hold_out`)

Splits the period into an earlier in-sample and a later out-of-sample part (split 0.5, 0.6 or 0.7) and runs each on its own. Answers: did it keep working in the later part.

Read: `split_date`; for Sharpe, CAGR, max drawdown and win rate the `in_sample`, `out_of_sample` and `change_pct` (relative to in-sample; negative is worse out of sample, the drawdown's too); `closed_trades` in each part (a thin out-of-sample count makes every out-of-sample figure tentative); `buy_hold_cagr_pct` per part; `average_degradation_pct` (mean size of the changes that went worse) drives `risk_assessment`.

Trap: a better out-of-sample half can be a friendlier market, not a better strategy; read it beside `buy_hold_cagr_pct` for the same halves.

Each half is its own backtest from a flat start with the full starting cash: the in-sample run stops at the split date and the out-of-sample run begins there with no position. A trade open across the split is cut in the first half and never entered in the second, so the two halves do not add up to the full run; the full run's CAGR can sit above or below both halves', and a large trade straddling the split moves every out-of-sample figure. When the halves disagree with the whole, check the trade log for a position open on `split_date` before reading anything else into it.

## Market Regime (`market_regime`)

Classifies every bar of the ticker's own price by trend (moving-average slope rising or flat against falling) and volatility (terciles of the run's rolling volatility), with no re-run. Answers: does the edge show up across conditions or in one of them. A regime under 20 bars is absent, not zero.

Read: `annualized_return_pct_by_regime` for `uptrend`, `downtrend`, `low_vol`, `medium_vol`, `high_vol`; `annualized_return_spread_pp` per axis (best minus worst). The overview carries the strategy's returns only. Fetch the `market_regime` section whenever a benchmark ran: it holds the per-cell Sharpe, the closed-trade win rates (trades counted by exit bar) and the benchmarks bucketed by the same regimes, which the trap below needs.

Trap: a long-only strategy that earns in uptrends and loses in downtrends is describing the market; the question is whether it lost less than buy and hold in the downtrend cell, which only the section's benchmark rows answer.

## Rolling Window (`rolling_window`)

Runs the strategy on sequential windows (3 to 12, rolling or non-overlapping). Answers: is the result consistent across sub-periods or made in one.

Read: `profitable_windows` of `total_windows` (`consistency_pct`), `windows_beating_buy_and_hold`, `average_excess_return_pp`, `worst_excess_return` with its window dates, best and worst window returns, `deepest_window_drawdown`, `window_return_std_pct`, `trend` (direction of window returns over time: a `weakening` trend means later windows earned less), `windows_under_trade_floor`.

Trap: rolling windows overlap, so one good stretch lifts several windows; non-overlapping is the stricter mode.

## Cost Stress (`cost_stress`)

Scales commission and slippage from 1x to 3x and re-runs. Answers: how much cost assumption the profit can absorb.

Read: `breakeven_cost_multiplier` (null with a note when still profitable at 3x, or when unprofitable at 1x), `baseline_net_profit`, `net_profit_at_2x` and `_3x`, `sharpe_ratio_at_2x`, `max_drawdown_pct_at_2x`, `profit_change_per_1x_cost_pct` (least-squares change of net profit per 1x of costs, relative to baseline), the `survives_1_5x`, `_2x`, `_3x` flags, `cost_sensitivity` and `stress_tolerance`.

Trap: `survives_3x` true with `profit_change_per_1x_cost_pct` near -25 means a quarter of the profit per cost step; the strategy survives, thinner each step.

## Execution Robustness (`execution_robustness`)

Re-runs with entry and exit fills delayed one to three bars, separately and together. Answers: does the edge depend on getting filled on the bar after the signal.

Read: `baseline_sharpe`, `avg_delayed_sharpe`, `worst_sharpe` with `worst_run_label`, `best_sharpe`, baseline and worst-run total return, `entry_sensitivity` against `exit_sensitivity` (which side of the trade is fragile), `significance` with its p-value, `tolerance_score` (100 means the delays had no effect; null with a note when the baseline Sharpe is about 0), `risk_level`. The full section adds the per-run rows and the worst single trade's impact.

Trap: a short average holding period (`avg_hold_bars` in the section) makes a one-bar delay a large fraction of the trade; read the delay against the holding time.

## Parameter Sensitivity (`parameter_sensitivity`)

Moves each tunable parameter on its own above and below its set value and re-runs, ranked by the chosen metric in its own units. Answers: does the result hold at the values next to the set ones.

Read: `metric` (key, unit, direction), `baseline_metric`, `parameters_tested`, `median_metric`, `best_metric`, `worst_metric`, `most_sensitive_parameter` with `max_adverse_change` (the largest drop from baseline any single change produced; null with a note when nothing scored below baseline), `variations_completed` of `_requested` and `_failed`. The full section adds per-parameter points (requested percent, realized value, metric, closed trades, status) and the two-parameter surface.

Trap: the test is one at a time; it does not see interactions. A broad variation (above 25%) is a stress test, not a neighborhood.

## Walk Forward (`walk_forward`)

Re-optimizes the selected parameters on each in-sample window and applies the winners to the following out-of-sample window; stitches the out-of-sample segments. Answers: would it have kept working traded live with periodic re-tuning.

Read: `walk_forward_efficiency_pct` (annualized out-of-sample return over annualized in-sample; the service marks 50% as the passing line and flags above 100% as an anomaly to look into), `stitched_oos` (total return, CAGR, Sharpe, max drawdown of the stitched track), `buy_and_hold_total_return_pct` over the same stitched bars, `oos_profitable_windows` of `oos_total_windows`, the `battery` checks (`combined_oos_profitable`, `wfe_pass`, `consistency`, `worst_window_drawdown`), `parameters_jumpy` (the chosen values jumped between windows), `risk_assessment`, `confidence`. The full section adds per-window rows (chosen parameters, in-sample and out-of-sample return, Sharpe, drawdown, fills, WFE) and `param_stability` per parameter (values chosen, spread, largest step).

Trap: a stitched track that trails buy and hold on the same bars means the re-optimization is not earning its keep, whatever the WFE.

## Overfitting (`overfitting`)

Builds a pool of variants with every tunable parameter varied together, runs each over the full period, and judges the pool by combinatorially symmetric cross-validation. Answers: is the result an edge or the best of many settings.

Read: `pbo_pct` (Probability of Backtest Overfitting: share of splits where the in-sample winner finished in the bottom half out of sample; the service bands it robust to 20, borderline 20 to 50, likely overfit from 50), `probability_oos_loss_pct`, `deflated_sharpe_ratio_pct` (probability the Sharpe beats the best luck would produce in this pool; survives deflation from 95, inconclusive 50 to 95, plausibly luck below 50), `probabilistic_sharpe_ratio_pct` (beats zero), `baseline_sharpe_annualized` against `expected_max_sharpe_annualized`, `degradation_slope`, `trials_executed`, `min_track_record_met`, `min_backtest_length_met`.

Trap: the luck benchmark comes from this pool only, not from everything the user has tried; a second round of changes after reading it is more trials the number does not know about.
