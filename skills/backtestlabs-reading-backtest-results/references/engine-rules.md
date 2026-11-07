# Engine rules that change how a result reads

- **Fills are at the next bar's open.** A signal computed on a bar's close fills at the following bar's open. A signal on the last bar of the range is dropped. Under volume-based slippage an order whose fill bar has no volume rests until the first bar with volume.
- **Each ticker is its own backtest.** Cash, positions and trade history are isolated per ticker, each from the same `starting_cash`. There is no combined equity, no cross-ticker sizing, no portfolio-level drawdown.
- **Consecutive buys merge.** Buy signals without an intervening sell aggregate into one position with a weighted average entry; the trade log shows the legs under one `chain_id`.
- **Costs.** Commission per fill by type (fixed, percent of value, per share). Slippage `volume_based` is a half-spread always paid plus an impact term that grows with the fill's share of the bar's volume, capped. Margin interest and short borrow cost accrue as separate log rows.
- **Holding periods.** Min Hold refuses an exit that would fill before it has elapsed (a margin call or Max Hold still closes); Max Hold closes at the open of the first bar on which the time is up.
- **Annualization.** `annualization_base` is periods per year: 252 for stocks, 365 for crypto on daily runs; intraday multiplies by bars per day; weekly 52, monthly 12. Sharpe and Sortino subtract `risk_free_rate_pct` (auto: 2% stocks, 0% crypto).
- **Risk anchor.** R multiples, MAE and MFE are measured in units of the initial risk set by `portfolio.risk_anchor` (a percent, a money distance, or an ATR multiple from entry). It is analytics only; nothing closes a trade on it.
- **Warm-up.** History is fetched before the start date so indicators are warm from the first bar of the period; the period is not shortened.
- **Crosses need a previous value.** A cross is `A[t] > B[t]` with `A[t-1] <= B[t-1]`; while either side is undefined on the previous bar there is no cross, so the first bar an indicator exists never signals. A Stop Loss, Profit Target or Trailing Stop is the exception: its level is born at the entry, and a close already beyond the level on the level's first bar fires, then only a cross does.
- **Stock intraday bars** start at 09:30 ET and are labelled by their start time: seven 1h bars a session, two 4h bars (09:30 and 13:30), one 12h bar. No bars on holidays or after an early close. **Crypto** bars start on the hour UTC, every day. Crypto composite bars are volume-weighted averages across exchanges; `BASE/QUOTE@EXCHANGE` runs one exchange's own bars.
- **Dividends** are tracked apart from the price (split-adjusted prices, dividends as rows); DRIP buys shares when on. Short positions pay in lieu.
- **Delisting** follows `risk_management.delisting_behavior`: hold at the last price, sell at it, or write off at zero.
- **Short sales** need `execution.margin.enabled`; a margin account borrows up to `leverage_ratio` and is called at `call_threshold`, liquidated at `maintenance_requirement`.
- **Minimum trade amount** refuses orders below `minimum_trade_amount`; `reserved_cash` is held back from new positions.
