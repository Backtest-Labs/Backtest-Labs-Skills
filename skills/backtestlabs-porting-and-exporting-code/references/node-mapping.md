# Mapping a script to nodes

Confirm every field with `get_document_reference` for the node type before writing the config; the names below are the config keys as the reference lists them, and the reference wins on any difference.

## Indicators

| Script | Node type | Fields |
|---|---|---|
| `ta.ema(src, n)`, `talib.EMA` | `ema` | `type` `trend` with `periods` `[n]` for one line; `crossover` with `[fast, slow]` for two; `direction` |
| `ta.sma`, `talib.SMA`; `ta.rma` | `ma` | as EMA with `ma_method` `sma` (`smma` for RMA) |
| `ta.rsi(src, n)`, `talib.RSI` | `rsi` | `period`, `oversold`, `overbought`, `direction`, `trigger` |
| `ta.macd(src, f, s, sig)`, `talib.MACD` | `macd` | `fastPeriod`, `slowPeriod`, `signalPeriod`, `direction`, `trigger` |
| `ta.bb(src, n, k)`, `talib.BBANDS` | `bollinger_bands` | `period`, `std_dev_up`, `std_dev_dn`, `band_side`, `direction`, `trigger`; lines `upper`, `middle`, `lower` |
| `ta.highest`/`ta.lowest` channel | `donchian_channels` | `period`, `band_side`, `direction`, `trigger`; lines `upper`, `middle`, `lower` |
| `ta.atr(n)`, `talib.ATR` | `atr` | `period`, `smoothing_method` (`wilder` is `ta.atr`), `multiplier`, `threshold`, `direction`, `trigger` |
| `ta.supertrend(f, n)` | `supertrend` | `atrPeriod`, `multiplier`, `direction` |
| `ta.stoch`, `ta.cci`, `ta.wpr`, `ta.mfi`, `ta.cmf`, `ta.obv`, `ta.dmi`, `ta.sar`, Ichimoku, Aroon, Keltner, VWAP, pivots | the node of that name | read the reference for the fields; the node's documentation page (`read_docs`) gives the formula to compare with the script's |
| `close`, `open`, `high`, `low`, `hl2`, `hlc3`, `ohlc4` | `price` | `priceSource`; `lag` for `close[1]` |
| a constant | `numeric_value` | `value`, `mode` `value` |
| arithmetic on series | `math` | `formula` over `input1..input5`, `inputCount`; output handle `out` |
| `ta.change`, `ta.roc`, `ta.stdev`, `ta.linreg`, `ta.cum`, a shift | `transform` | `transformType` and its active keys |

An indicator node's `direction` and `trigger` turn its own threshold or crossing into a signal: RSI bullish cross is the bar RSI crosses back up through `oversold`; Bollinger bearish on the lower side is the close crossing below the lower band; EMA crossover bullish is the fast line crossing above the slow one. When the script's condition is the node's own signal, use it; when it compares two series the node does not pair, use the operator nodes.

## Conditions

| Script | Node | Notes |
|---|---|---|
| `ta.crossover(a, b)` | `crossover_detection` `mode` `cross`, `direction` `bullish` | inputs `in0` (A) and `in1` (B), both series |
| `ta.crossunder(a, b)` | same, `direction` `bearish` | |
| `a > b` held every bar | `crossover_detection` `mode` `state` | the output values are A minus B |
| `a > b`, `>=`, `<`, `<=`, `==`, `!=` as a signal | `relation` | `relation` |
| `cond1 and cond2` | `and` | two signal inputs |
| `cond1 or cond2` | `or` | |
| `cond1 and close > ta.sma(close, 200)` | `direction_filter` | signal on `in0`, a series whose sign is the direction on `in1` (a `state` Crossover of close against the MA gives that series) |
| `cond2` within N bars after `cond1` | `sequence` | `candles` |
| wait N bars after a signal | `wait` | `periods` |
| a value frozen at entry (`var entry = close`) | `capture` | top input the trigger, the ticker's close is captured for a signal-only wire |
| time-of-day or weekday rule | `schedule` | intraday only |

## Orders and exits

| Script | Node | Notes |
|---|---|---|
| `strategy.entry("L", strategy.long)` | `buy` `direction` `long` | `amount_type` and `amount_value` from the script's sizing (`percent_cash` 100 for all-in) |
| `strategy.close("L")` | `sell` `direction` `exit_long` | `percent_position` 100 |
| `strategy.entry("S", strategy.short)` | `sell` `direction` `short` | needs `execution.margin.enabled` |
| `strategy.close("S")` | `buy` `direction` `cover_short` | |
| `strategy.exit(stop=...)` | `stop_loss` from a captured entry | `stop_type` `percentage`, `fixed` or `atr` |
| `strategy.exit(limit=...)` | `profit_target` | |
| `strategy.exit(trail_points=...)` | `trailing_stop` | no reference input |
| `pyramiding=N` | merged entries | consecutive buys merge into one position; N is not enforced |
| commission, slippage | `execution.commission`, `execution.slippage` | document settings, not nodes |
| `default_qty_type=strategy.percent_of_equity` | `buy` `amount_type` `percent_portfolio` | |

## No node

State these as divergences: `request.security` (a second timeframe), intrabar logic (`calc_on_every_tick`, `barstate.isrealtime`), loops and arrays carried across bars, `varip`, equity-curve-dependent sizing beyond `percent_portfolio`, limit orders that rest for several bars (the Buy and Sell nodes' `limit` order type evaluates against the next bar), partial exits in a sequence other than one Sell per signal, and any indicator without a node (`read_docs` `nodes-overview` lists them all).
