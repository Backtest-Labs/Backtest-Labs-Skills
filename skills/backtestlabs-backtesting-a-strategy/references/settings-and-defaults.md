# Settings and defaults

The document skeleton from `get_document_reference` is what the editor writes for a new strategy. A global section left out of a document takes the skeleton's values; a section that is present must carry every key the skeleton shows for it, and only those keys. The values below are the skeleton's unless marked as this skill's choice.

## Interval and period

| Setting | Default | Notes |
|---|---|---|
| `data_source.interval` | `1d` in the skeleton | Use the user's interval. When none is given and the wording implies none, use `1d` and say so. `execution_mode` and `execution_interval` follow the interval and may be left out. |
| `backtest_period` | `last_n_units`, 100 days in the skeleton | This skill's choice when the user gives no period: `last_n_years` 3 for 1d, 1w and 1mo; 1 for 1h to 12h; `last_n_units` 3 months for 1m to 30m. |
| Intraday span | plan cap | `get_account.limits.intraday_max_span_months` per interval. A longer period is trimmed on save. |
| `date_range` | user dates | `start_date` before `end_date`, `YYYY-MM-DD`. |

Period definitions and the keys each reads: `last_n_units` reads `value` and `unit` (`days` or `months`); `last_n_years` holds only `definition` and `years_value` (a fraction is allowed); `date_range` reads `start_date` and `end_date`. `years_value` must be null unless the definition is `last_n_years`.

Stock intraday bars are counted from the 09:30 ET open: a 1h interval has seven bars a session, 4h has two (09:30 and 13:30), 12h one. Crypto bars start on the hour UTC and trade every day. `extended_hours` true includes pre-market and after-hours stock bars.

## Portfolio

| Setting | Default | Meaning |
|---|---|---|
| `starting_cash` | 100000 | each ticker's starting equity |
| `reserved_cash_type` / `reserved_cash_value` | `percentage` / 0 | cash held back from new positions |
| `minimum_trade_amount` | 10 | smallest order value that fills |
| `fractional_shares` | true | quantities may be fractional |
| `risk_anchor` | `fixed_percentage`, percent 2 | the unit of R multiples, MAE and MFE; analytics only, it never closes a trade |
| `risk_free_rate` | `auto` (0% crypto, 2% stocks) | what Sharpe and Sortino subtract |
| `annualization_base` | `auto` (365 crypto, 252 stocks) | periods a year; intraday multiplies by bars per day |

## Execution

| Setting | Default | Meaning |
|---|---|---|
| `commission` | `fixed`, 0 | per fill; `percentage` is percent of fill value; `per_share` money per unit |
| `slippage` | `volume_based`, 0.05 | 0.05% half-spread always paid, plus impact: 0.03% at 5% participation, logarithmic past it, capped at 0.75% |
| `margin` | disabled | enabling allows borrowing up to `leverage_ratio` and short sales; `call_threshold` at or above `maintenance_requirement`; maintenance below 100 / leverage |
| `holding_periods` | 0 (none) | Min Hold refuses earlier exits; Max Hold closes at the open of the first bar it has elapsed |
| `risk_management.delisting_behavior` | `hold_last_price` | what happens to a position whose ticker stops trading |
| `features.drip_enabled` | true | dividends buy more shares (stocks only) |

Fills are at the next bar's open after the signal bar. A signal on the last bar is dropped. Consecutive Buy signals without a Sell merge into one position with a weighted average entry.

## Sizing (on the Buy and Sell nodes)

- Buy: `direction` `long`, `amount_type` `percent_cash`, `amount_value` 100, `order_type` `market`. Other amount types: `max_cash`, `custom_dollar`, `percent_portfolio`, `shares`.
- Sell: `direction` `exit_long`, `amount_type` `percent_position`, `amount_value` 100, `order_type` `market`. A `short` direction opens a short and needs margin.
- Order types: `market`, `limit` (needs `limit_price`), `stop` (needs `stop_price`), `stop_limit` (both), `trailing_stop` (`trail_type` `$` or `%`, `trail_value`). The price keys are left out of the config unless the order type uses them.

## Validations in the skeleton

On by default: `monteCarlo` (1000 simulations, 0.95), `tradeSequence` (1000 permutations), `marketRegime`, `holdOut` (split 0.5), `rollingWindow` (5 rolling windows), `costStress`. Off by default: `parameterSensitivity`, `overfitting`, `walkForward`, `executionRobustness`. A validation the plan lacks is switched off on save; `get_account.validations.available` lists what runs. A free account with `free_runs_left` above 0 keeps every validation.

## What the plan trims on save

Tickers and benchmarks beyond `max_tickers` and `max_benchmarks`, an intraday period beyond the span cap, and validations the plan lacks. The saved document in the answer shows what survived; report the difference.
