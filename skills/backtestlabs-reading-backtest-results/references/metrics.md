# Metrics

Values from `get_results`, overview and `summary` section. Names ending `_pct` are percent, `_pp` percentage points, money in the run's quote currency. A null comes with a note under `notes`.

## Returns

| Value | Shows | How to read | Trap |
|---|---|---|---|
| `total_return_pct` | compounded return over the period | against the benchmark's and against `gross_return_pct` | on 40% average exposure it is a return on a partly idle account |
| `cagr_pct` | compound annual growth rate | the per-year rate the period's return implies | meaningless under about a year of data; `data_sufficiency.cagr_sufficient` |
| `gross_return_pct` | return with every cost added back | gross minus net is `cost_drag_pp` | a strategy whose gross is high and net low is a cost problem, not a signal problem |
| `cost_drag_pp` | commission, slippage, borrow and interest as points of return | with `commission_drag_pct`, `slippage_drag_pct`, `borrow_drag_pct`, `interest_drag_pct` (each a share of starting equity) | slippage under `volume_based` grows with fill size against bar volume |
| `final_equity`, `final_cash` | equity and cash at the last bar | equity includes an open position | |
| `excess_return_pp` (per benchmark) | ticker return minus the benchmark's | the number the "beat buy and hold" question wants | a late-starting benchmark covers a shorter period (`warnings`) |
| `dividends.impact_on_return_pp` | dividends' share of the return | `received`, `reinvested` (DRIP), `in_lieu_paid` on shorts | stocks only |

## Risk

| Value | Shows | How to read | Trap |
|---|---|---|---|
| `annualized_volatility_pct` | standard deviation of bar returns, annualized | against the benchmark's | intraday runs annualize by bars per day times the base |
| `sharpe_ratio` | annualized Sharpe, net of the risk-free rate | with closed trades and `sharpe_sufficient` | a flat equity line gives a Sharpe near 0 with a note; few trades make it unstable |
| `sortino_ratio` | downside deviation measured from zero | above Sharpe when losses are small and gains lumpy | |
| `calmar_ratio` | CAGR over max drawdown | a return-per-unit-of-pain figure | needs a drawdown and enough data (`calmar_sufficient`) |
| `max_drawdown_pct` | deepest fall from a running peak | with `longest_drawdown_days` and `max_drawdown_date` | `max_drawdown_value` (money) can be on another bar |
| `drawdown_at_end_pct` | drawdown at the last bar | above 0 means the run ended underwater | |
| `avg_drawdown_pct`, `drawdown_distribution` | the typical underwater period | mean, median and 95th percentile depth and duration (`duration_unit` days, or bars intraday) | the max is one event; the distribution is the habit |
| `recovery_factor` | net profit over the max drawdown in money | how many times the worst loss was earned back | |
| `best_day_pct`, `worst_day_pct`, `worst_month_pct` | the extremes | with `positive_days_pct` | weekly and monthly runs report the best and worst bar |
| `tail_ratio` | 95th percentile bar return over the magnitude of the 5th | above 1: the good tail is longer | |
| `skewness` | skewness of bar returns | positive: rare large gains; negative: rare large losses | |
| `average_leverage`, `max_leverage` | gross exposure over equity | 1.0 is fully invested; above 1 means borrowing | mean exposure, not time in market |
| `margin_calls` | times a position fell below the call level | any value above 0 with margin on is a sizing problem | |

## Trades

`closed_trades` counts closing fills (SELL or SHORT_COVER rows); a position exited in parts counts once per exit. `fills` counts every execution.

| Value | Shows | How to read | Trap |
|---|---|---|---|
| `win_rate_pct` | share of closed trades that made money | with `payoff_ratio` | high win rate with payoff below 1 can lose |
| `profit_factor` | gross profit over gross loss | above 1 is net positive; 1.5 to 2 is typical of a working edge | capped at 999.99; null with note when no trade lost |
| `payoff_ratio` | mean win over mean loss, as magnitudes | with win rate: expectancy = win rate x mean win minus loss rate x mean loss | |
| `expectancy` | average P&L per closed trade | the sign of the edge in money | small against costs per trade means fragile |
| `average_win`, `average_loss`, `largest_win`, `largest_loss` | the trade P&L shape | `largest_win` against `total_pnl` | one trade holding most of the P&L |
| `pnl_std`, `sqn` | dispersion; System Quality Number = sqrt(n) x mean / std | SQN grows with trade count | |
| `max_win_streak`, `max_loss_streak` | longest runs | Trade Sequence gives the odds of the loss streak | |
| `annual_turnover_pct` | SEC turnover, annualized | 2400% means the account turned over 24 times a year | |
| `total_commissions`, `total_slippage`, `total_borrow_cost`, `total_margin_interest` | costs paid | against `total_pnl` | |

## Data sufficiency

`data_sufficiency` in the summary section: `data_points` (bar returns), `bars_per_year`, and a boolean per annualized statistic. Quote a false flag beside the statistic it concerns.

## Trade analysis section

`holding_period_hours` (mean, median, min, max; first entry to final exit) and `capacity` (`basis` `adv_shares` for stocks: shares over the mean daily volume of the prior 20 sessions; `volume_24h_notional` for crypto; `median_pct`, `p95_pct`, `max_pct`). A p95 above a few percent of daily volume means the backtest's fills would move a real market more than the slippage model assumes.
