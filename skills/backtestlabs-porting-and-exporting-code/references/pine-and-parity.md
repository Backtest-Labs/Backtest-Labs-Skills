# Pine export and parity

## The Pine export

`export_code` with `language` `pine` answers the signal file (`<strategy>_buysell_signals.pine`, or one per signal type) and `README.txt`. The script is an **indicator**, not a `strategy()`: it computes the strategy's conditions with `ta.*` and `math.*`, plots `buySignal` and `sellSignal` with `plotshape` (buy below the bar, sell above), and declares `alertcondition` for each, titled `<strategy> Buy` and `<strategy> Sell`. `overlay` is true when the strategy's indicators draw on price. It places no orders, so TradingView's strategy tester does not apply; the user adds it to a chart and creates alerts from the two alert conditions, set to **Once Per Bar Close** so they fire on the confirmed bar and not intrabar on the forming one.

The header, warm-up note and usage comments are part of the file; deliver them.

## Refusals

Every language: the portfolio nodes (`portfolio_value`, `cash_balance`, `drawdown`, `exposure`, `position_count`, `risk_per_trade`: generated code has no running portfolio) and a Numeric Value in percent mode. Pine only: `candlestick` and `bars` (multi-pattern bundling not implemented in Pine; Python exports them), a Dividend or Earnings Event whose Offset Days is not 0 (Pine fires on the event bar itself), a Support and Resistance wire from level 2 or higher. Found while generating: a Math formula or Transform setting Pine cannot express, and a Capture node triggered by a series. The tool names each node and the reason.

## Parity: engine, TA-Lib, Pine

The engine's indicator math tracks TA-Lib. The Python export uses TA-Lib (or the engine's own helper where rounding would flip a strict comparison), so it matches the engine on every warm bar. Pine's `ta.*` functions implement the same standard math in nearly all cases; the export labels the exceptions in line.

| Indicator | Divergence | Effect |
|---|---|---|
| ATR, MACD, Supertrend | Pine seeds the Wilder or EMA average one bar earlier than TA-Lib | values differ by a small decaying amount over about the first period bars, then converge |
| Volume Profile | Pine has no tick or lower-timeframe data in an indicator | Point of Control and value-area levels are approximated; Python reproduces the histogram |
| Bollinger Bands | population standard deviation (divides by N) on both sides | none when the user's own script used `ta.stdev` with the default (population) |
| Donchian Channels | the engine compares the close with the previous bar's channel | a script comparing with the current channel never signals on a breakout bar |
| Crossings | engine: `A[t] > B[t]` and `A[t-1] <= B[t-1]` | `ta.crossover` is the same strictness; a script using `>=` or a two-bar confirmation differs |
| Patterns (Candlestick, Bars) | classification follows the TradingView built-in conventions, not TA-Lib's | Pine export refused for these nodes; Python exports them |

## Timing

Signals fire on the bar whose close meets the condition; neither export shifts them. The engine fills on the next bar's open, so a backtest's trade marker sits one bar after the signal. A script that executes `strategy.entry` on the signal bar's close, or uses `calc_on_every_tick`, times fills differently from the backtest.

## Reconciling signal counts

The export's raw signals exceed the backtest's trades: the engine skips a sell with no position, a buy with no cash, a signal on the last bar, and merges consecutive buys into one position. To reconcile, count the script's buy signals that fall while flat and sell signals while long; that count approaches the backtest's closed trades.
