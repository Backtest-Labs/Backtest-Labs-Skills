# Charts

`get_chart` with run id and ticker alone lists the pages and each chart's id, title and what it shows. With `page` it returns every chart of the page; with `chart` as well, that one chart. Each image costs about 1,250 to 1,600 tokens and a page returns up to six. `visuals` false returns each chart's data as text (the series, downsampled, and the annotated values with units) and no image: use it by default, and fetch an image only when the user wants to see the shape or asks for the picture.

## Which chart answers which question

| Question | Page | Chart id |
|---|---|---|
| Where did it buy and sell on the price | `price` | `price_fills` |
| How did equity track the benchmark | `returns` | `equity_curve` |
| Are bar returns fat-tailed or skewed | `returns` | `returns_distribution` |
| Which months made or lost | `returns` | `periodic_returns` |
| How long and how deep underwater | `drawdown` | `underwater` |
| How much time at each depth | `drawdown` | `drawdown_distribution` |
| Steady edge or a few big trades | `trade_distribution` | `net_pnl_cumulative` |
| Entry and exit quality per trade | `trade_distribution` | `mae_mfe` (percent of entry, winners and losers apart) |
| Shape of trade P&L | `trade_distribution` | `pnl_histogram` |
| Does the exit weekday matter | `trade_distribution` | `day_of_week` (average R by exit weekday) |
| Did holding longer pay | `trade_distribution` | `duration_vs_payoff` |
| How long are winners held against losers | `trade_distribution` | `holding_period_distribution` |
| Is the Sharpe stable over time | `rolling_metrics` | `rolling_sharpe` |
| Is volatility stable | `rolling_metrics` | `rolling_volatility` |
| How much of it is the benchmark | `rolling_metrics` | `rolling_beta_correlation_<benchmark>` |
| When was the account invested | `exposure` | `net_gross_exposure` |
| Spread of Monte Carlo paths | `validation_monte_carlo` | `equity_bands`, `final_equity_distribution`, `max_drawdown_distribution` |
| Odds of the drawdown, streak and time underwater | `validation_trade_sequence` | `drawdown_exceedance`, `losing_streak_odds`, `time_underwater_odds` |
| In-sample against out-of-sample equity | `validation_hold_out` | `is_oos_equity` |
| Returns across regimes | `validation_market_regime` | `regime_timeline` |
| Each window's equity | `validation_rolling_window` | `window_equity_curves` |
| Equity at each cost level | `validation_cost_stress` | `equity_curves_under_stress` |

Pages for execution robustness, parameter sensitivity, walk forward and overfitting appear in the list only when that validation ran on the ticker; read their chart ids from the list.

## Reading the text form

With `visuals` false each chart comes as its series (downsampled) and the values the image would annotate, with units. Quote the annotated values (the max drawdown point, the percentile, the breakeven multiplier) rather than reading points off the series.
