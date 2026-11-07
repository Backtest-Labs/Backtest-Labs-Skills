# Documentation map for `read_docs`

`read_docs` with no page lists every page; with a page id it returns the page's full text with formulas and parameters. Answer node, chart-construction and engine questions from the page, in its terms.

## Nodes (page id = node type unless noted)

`nodes-overview`; trend: `ema`, `ma`, `dmi`, `ichimoku`, `supertrend`, `parabolic-sar`, `trendline`, `aroon`; oscillators: `rsi`, `macd`, `stochastic`, `cci`, `williams-r`, `stochastic-rsi`; volatility and range: `bollinger-bands`, `donchian-channels`, `keltner-channels`, `atr`, `volatility-regime`, `pivot-points`, `support-resistance`, `swing-pivots`, `zigzag`; volume: `volume`, `relative-volume`, `obv`, `mfi`, `cmf`, `volume-profile`, `vwap`; price action: `price`, `candlestick`, `bars-node`, `heikin-ashi`; events: `dividend`, `earnings-event`, `split-event`, `schedule`; orders: `buy`, `sell`, `wait`; operators: `and-operator`, `or-operator`, `split-operator`, `relation-operator`, `sequence-operator`, `transform-operator`, `math-operator`, `numeric-value`, `capture`; trade management: `stop-loss`, `profit-target`, `trailing-stop`; utilities: `divergence`, `direction-filter`, `crossover`; portfolio: `portfolio-value`, `drawdown`, `exposure`, `position-count`, `cash-balance`, `risk-per-trade`.

## Charts and validations

`charts-overview`, `charts-price`, `charts-returns`, `charts-drawdown`, `trade-distribution-overview`, `trade-distribution-distribution`, `trade-distribution-trades`, `trade-distribution-timing`, `trade-distribution-execution`, `rolling-performance`, `rolling-benchmark`, `charts-calendar`, `charts-exposure`, `charts-multi`; `validations`, `monte-carlo`, `trade-sequence`, `hold-out`, `market-regime`, `rolling-window`, `cost-stress`, `robustness` (execution robustness), `sensitivity` (parameter sensitivity), `walk-forward`, `overfitting`.

## Engine

`execution-overview`, `graph-resolution`, `data-pipeline`, `trade-execution`, `cost-modeling`, `dividends-corporate-actions`, `validation-engine`, `code-export`, `strategy-import-export`.

## Other

`getting-started`, `faq`, `mcp-overview`, `mcp-connect`, `mcp-tools`, `mcp-connected-apps`, `mcp-troubleshooting`.
