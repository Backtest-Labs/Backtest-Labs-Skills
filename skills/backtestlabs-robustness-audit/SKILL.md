---
name: backtestlabs-robustness-audit
description: Runs a Backtest Labs strategy once with every validation the plan includes switched on, then reads the ten results in a dependency order into a scorecard that states what each test shows and the question it leaves open, with no verdict. Use when the user asks whether a strategy holds up, is overfit, is robust, would survive live trading or higher costs, or wants the full validation suite. Not for a single validation's meaning (reading-backtest-results) or for choosing parameter values (walk-forward-parameter-selection).
---

# Robustness audit on Backtest Labs

Ten validations, each answering one question, run on the same backtest. Read alone, each is a number with a service rating beside it. Read in order, they separate four different reasons a backtest can look better than the strategy is: too little data, a result made in one period, a parameter choice that was the best of many, and frictions the simulation did not charge. This skill is that order.

## Workflow

1. **Plan and budget.** `get_account`: `validations.available` (the ones the plan includes; a free account with `free_runs_left` above 0 keeps every validation for that run) and `daily_backtest_runtime.remaining_seconds`. The full suite re-runs the backtest many times: Walk Forward runs every parameter combination on every window, Overfitting runs the trial pool over the full period, Sensitivity runs every variation, Cost Stress six levels, Execution Robustness ten delays. The only wall-clock figure is the server's: `run_backtest` answers an `estimate` label (such as "under 5 min") when it starts the run, and `get_run_status` carries it after. Before the start, give the backtest count from the settings (combinations times windows for Walk Forward, trials for Overfitting, variations for Sensitivity, six and ten for the two fixed ones) and the remaining daily time; do not invent minutes.
2. **Read the strategy.** `get_strategy` for the document and `revision`. Note the node types and their `tunable_keys` (`get_document_reference` with the types) because Walk Forward needs at least one swept parameter and the sweep must respect the node's own rules (a fast period below the slow one, oversold below overbought).
3. **Set the validation section** per `references/audit-settings.md`. Start from the strategy's own `validation` section, not from the skeleton: a saved strategy usually carries settings the owner chose with every validation switched off, and the audit keeps them (its Walk Forward parameters, objective, ranking metrics, spread and grid) and only switches each validation on. Fill a setting from the audit defaults only where the saved section has none, and list in the report every setting the audit changed from the saved value, with the old and new value. Walk Forward needs at least one swept parameter: keep the saved ones, else one or two keys that shape the entry or exit, at most five steps each. Keep everything else in the document unchanged. Save as a copy with `create_strategy` named `<name> audit` so the baseline run and document stay, or `update_strategy` with the revision when the user wants the audit on the strategy itself.
4. **Run.** `run_backtest`; note `validations_removed_by_plan`; `get_run_status` with `wait_seconds` 60 until done. A `cancelled` status with a `stop_reason` means the daily cap stopped it; report the time used and the reset time.
5. **Read in order** from the `get_results` overview (every validation's headline block sits under `tickers.<ticker>.validations`); fetch a full section only for a row-level question:
   1. Sample: `closed_trades`, `coverage`, `data_sufficiency` (summary section), each validation's `confidence`. A low confidence is read before the value it qualifies.
   2. Across time: `hold_out`, `rolling_window`, `walk_forward`.
   3. Selection: `parameter_sensitivity`, `overfitting`.
   4. Path: `monte_carlo`, `trade_sequence`.
   5. Frictions: `cost_stress`, `execution_robustness`.
   6. Conditions: `market_regime`.
6. **Cross-read** with `references/scorecard-rubric.md`: the pairs that explain each other (a good hold-out with a high PBO, a passing WFE with jumpy parameters, cost survival with short holds and high execution sensitivity).
7. **Report** the scorecard. Each row: the test, the numbers that matter, the service's own rating (`risk_assessment`, `risk_level`, `stress_tolerance`, `confidence`) named as the service's, and the question the test leaves open. End with what the tests together do and do not establish, and the skill that would answer the largest open question.

## Scorecard

```
Strategy: <name>, <ticker>, <interval>, <period>; <n> closed trades; coverage <bars>
Validations that ran: <list>; not run: <list with reason>
Settings changed from the saved strategy: <setting: old to new>, or "none"

Sample        closed trades <n>; data sufficiency <flags>; lowest confidence: <validation> (<level>)
Hold-out      split <date>; Sharpe IS <x> / OOS <x> (<change>%); max DD IS <x> / OOS <x>; OOS trades <n>; service risk <level>
Rolling       <k>/<n> windows profitable; <m> beat buy and hold; worst excess <x> pp (<window dates>); trend <weakening|stable|strengthening>
Walk forward  WFE <x>% (service passing line 50%); stitched OOS <x>% vs buy and hold <x>%; <k>/<n> OOS windows profitable; parameters jumpy <yes|no>; battery <checks failed>
Sensitivity   metric <name>; baseline <x>; worst <x>; most sensitive <param> (<max adverse change>); <completed>/<requested> variations
Overfitting   PBO <x>% (service bands: robust to 20, borderline to 50); DSR <x>% (survives from 95); OOS loss <x>%; trials <n>; track record met <yes|no>
Monte Carlo   final equity rank <x>; CI <lower> to <upper>; max DD actual <x>% vs median <x>% / p5 <x>%; P(DD > 20%) <x>%
Sequence      drawdown rank <x>; loss streak actual <n> vs median <m> / p95 <p>; time underwater actual <n> vs p95 <p>
Cost stress   survives 1.5x/2x/3x <flags>; breakeven <x>x or "beyond 3x"; profit change per 1x <x>%; service tolerance <level>
Execution     tolerance <x>; baseline Sharpe <x>, avg delayed <x>, worst <x> (<run>); entry sens <x> / exit sens <x>; significance <label>
Regime        annualized return uptrend <x>% / downtrend <x>%; low vol <x>% / high vol <x>%; spread trend <x> pp, vol <x> pp

What holds together: <two or three sentences from the cross-read>
Open questions: <each with the skill or run that answers it>
```

Report what each test shows. The service's ratings are quoted as its ratings. No verdict on whether to trade it, no sizing advice.

## References

- `references/audit-settings.md`: the validation section for the audit, with the Walk Forward parameter format and the runtime it implies.
- `references/scorecard-rubric.md`: per test, the numbers that matter, the service's thresholds, the trap, and the cross-reads.
