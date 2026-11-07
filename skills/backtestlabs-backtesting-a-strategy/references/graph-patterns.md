# Graph patterns

Eight graphs, each a `strategy_graph` value (`nodes`, `edges`, `groups`) to drop into the document skeleton. Every config carries every key of its node's fields, as the document reference lists them. Positions are left out so the editor lays the graph out. Adapt periods, thresholds and directions to the spec; keep the wiring.

Reading the wiring: an indicator node's `out0` carries both a trigger (its signal) and values (its line). A `signal` input (Buy, Sell, And, Or, Direction Filter top) reads the trigger; a `series` input (Crossover, Relation, Math, Direction Filter bottom, a `data_in`) reads the values. An output takes one wire; the Split node fans one wire out to two.

## 1. Moving-average crossover (long only)

Buy when the fast EMA crosses above the slow one; exit when it crosses below. Two EMA nodes, because one node with direction `both` would wire the same trigger to both orders.

```json
{"nodes":[
 {"id":"0","type":"ema","data":{"type":"ema","label":"EMA","config":{"type":"crossover","periods":[20,50],"direction":"bullish"}}},
 {"id":"1","type":"buy","data":{"type":"buy","label":"Buy","config":{"direction":"long","order_type":"market","amount_type":"percent_cash","amount_value":100}}},
 {"id":"2","type":"ema","data":{"type":"ema","label":"EMA","config":{"type":"crossover","periods":[20,50],"direction":"bearish"}}},
 {"id":"3","type":"sell","data":{"type":"sell","label":"Sell","config":{"direction":"exit_long","order_type":"market","amount_type":"percent_position","amount_value":100}}}],
 "edges":[
 {"source":"0","target":"1","sourceHandle":"out0","targetHandle":"in0"},
 {"source":"2","target":"3","sourceHandle":"out0","targetHandle":"in0"}],
 "groups":[]}
```

The MA node is the same shape with `ma_method` (`sma` or `smma`) added to the config.

## 2. Oscillator mean reversion (RSI)

Buy on the bar RSI crosses back up through the oversold level; sell on the bar it crosses back down through overbought. `trigger` `cross` fires once per crossing; `state` would fire on every bar below the level.

```json
{"nodes":[
 {"id":"0","type":"rsi","data":{"type":"rsi","label":"RSI","config":{"period":14,"overbought":70,"oversold":30,"direction":"bullish","trigger":"cross"}}},
 {"id":"1","type":"buy","data":{"type":"buy","label":"Buy","config":{"direction":"long","order_type":"market","amount_type":"percent_cash","amount_value":100}}},
 {"id":"2","type":"rsi","data":{"type":"rsi","label":"RSI","config":{"period":14,"overbought":70,"oversold":30,"direction":"bearish","trigger":"cross"}}},
 {"id":"3","type":"sell","data":{"type":"sell","label":"Sell","config":{"direction":"exit_long","order_type":"market","amount_type":"percent_position","amount_value":100}}}],
 "edges":[
 {"source":"0","target":"1","sourceHandle":"out0","targetHandle":"in0"},
 {"source":"2","target":"3","sourceHandle":"out0","targetHandle":"in0"}],
 "groups":[]}
```

## 3. Band reversion with a midline exit (Bollinger)

Buy when the close crosses below the lower band (`band_side` `lower`, `direction` `bearish`: the bearish case is price breaking below the lower band). Exit when the close crosses back above the middle band: a Crossover node compares the Price series with the band node's `middle` line, picked by naming it as the wire's `sourceHandle`.

```json
{"nodes":[
 {"id":"0","type":"bollinger_bands","data":{"type":"bollinger_bands","label":"Bollinger Bands","config":{"period":20,"std_dev_up":2,"std_dev_dn":2,"direction":"bearish","band_side":"lower","trigger":"cross"}}},
 {"id":"1","type":"buy","data":{"type":"buy","label":"Buy","config":{"direction":"long","order_type":"market","amount_type":"percent_cash","amount_value":100}}},
 {"id":"2","type":"bollinger_bands","data":{"type":"bollinger_bands","label":"Bollinger Bands","config":{"period":20,"std_dev_up":2,"std_dev_dn":2,"direction":"both","band_side":"upper","trigger":"cross"}}},
 {"id":"3","type":"price","data":{"type":"price","label":"Price","config":{"priceSource":"close","lag":0}}},
 {"id":"4","type":"crossover_detection","data":{"type":"crossover_detection","label":"Crossover","config":{"mode":"cross","direction":"bullish"}}},
 {"id":"5","type":"sell","data":{"type":"sell","label":"Sell","config":{"direction":"exit_long","order_type":"market","amount_type":"percent_position","amount_value":100}}}],
 "edges":[
 {"source":"0","target":"1","sourceHandle":"out0","targetHandle":"in0"},
 {"source":"3","target":"4","sourceHandle":"out0","targetHandle":"in0"},
 {"source":"2","target":"4","sourceHandle":"middle","targetHandle":"in1"},
 {"source":"4","target":"5","sourceHandle":"out0","targetHandle":"in0"}],
 "groups":[]}
```

## 4. Channel breakout (Donchian)

Buy on the first close above the prior bar's upper channel; exit on the first close below the prior bar's lower channel. The channel is compared one bar back because the current bar's high and low are inside the current channel.

```json
{"nodes":[
 {"id":"0","type":"donchian_channels","data":{"type":"donchian_channels","label":"Donchian Channels","config":{"period":20,"direction":"bullish","band_side":"upper","trigger":"cross"}}},
 {"id":"1","type":"buy","data":{"type":"buy","label":"Buy","config":{"direction":"long","order_type":"market","amount_type":"percent_cash","amount_value":100}}},
 {"id":"2","type":"donchian_channels","data":{"type":"donchian_channels","label":"Donchian Channels","config":{"period":10,"direction":"bearish","band_side":"lower","trigger":"cross"}}},
 {"id":"3","type":"sell","data":{"type":"sell","label":"Sell","config":{"direction":"exit_long","order_type":"market","amount_type":"percent_position","amount_value":100}}}],
 "edges":[
 {"source":"0","target":"1","sourceHandle":"out0","targetHandle":"in0"},
 {"source":"2","target":"3","sourceHandle":"out0","targetHandle":"in0"}],
 "groups":[]}
```

## 5. Trend-filtered oscillator entry

The RSI buy passes only while the close is above its 200-bar average. A Crossover node in `state` mode outputs the difference close minus MA as its values; the Direction Filter reads that series on its bottom input and passes the RSI signal (top input) on bars where the difference is positive.

```json
{"nodes":[
 {"id":"0","type":"rsi","data":{"type":"rsi","label":"RSI","config":{"period":14,"overbought":70,"oversold":30,"direction":"bullish","trigger":"cross"}}},
 {"id":"1","type":"price","data":{"type":"price","label":"Price","config":{"priceSource":"close","lag":0}}},
 {"id":"2","type":"ma","data":{"type":"ma","label":"MA","config":{"type":"trend","ma_method":"sma","periods":[200],"direction":"both"}}},
 {"id":"3","type":"crossover_detection","data":{"type":"crossover_detection","label":"Crossover","config":{"mode":"state","direction":"bullish"}}},
 {"id":"4","type":"direction_filter","data":{"type":"direction_filter","label":"Direction Filter","config":{"filter_direction":"bullish"}}},
 {"id":"5","type":"buy","data":{"type":"buy","label":"Buy","config":{"direction":"long","order_type":"market","amount_type":"percent_cash","amount_value":100}}},
 {"id":"6","type":"rsi","data":{"type":"rsi","label":"RSI","config":{"period":14,"overbought":70,"oversold":30,"direction":"bearish","trigger":"cross"}}},
 {"id":"7","type":"sell","data":{"type":"sell","label":"Sell","config":{"direction":"exit_long","order_type":"market","amount_type":"percent_position","amount_value":100}}}],
 "edges":[
 {"source":"1","target":"3","sourceHandle":"out0","targetHandle":"in0"},
 {"source":"2","target":"3","sourceHandle":"out0","targetHandle":"in1"},
 {"source":"0","target":"4","sourceHandle":"out0","targetHandle":"in0"},
 {"source":"3","target":"4","sourceHandle":"out0","targetHandle":"in1"},
 {"source":"4","target":"5","sourceHandle":"out0","targetHandle":"in0"},
 {"source":"6","target":"7","sourceHandle":"out0","targetHandle":"in0"}],
 "groups":[]}
```

## 6. Stop loss and profit target measured from the entry

The Buy node's signal feeds a Capture node, which stores the close of the signal bar (a signal-only wire carries no value, so the ticker's close is captured) and holds it until the next entry. A Split fans the held entry price to the Stop Loss and Profit Target `data_in` inputs; their cross signals are combined with Or and drive the Sell. The fill itself is at the next bar's open, so the captured level sits one bar before the fill price; an ATR-type distance is measured from that captured close.

```json
{"nodes":[
 {"id":"0","type":"ema","data":{"type":"ema","label":"EMA","config":{"type":"crossover","periods":[20,50],"direction":"bullish"}}},
 {"id":"1","type":"buy","data":{"type":"buy","label":"Buy","config":{"direction":"long","order_type":"market","amount_type":"percent_cash","amount_value":100}}},
 {"id":"2","type":"capture","data":{"type":"capture","label":"Capture","config":{"mode":"last","initialValue":null}}},
 {"id":"3","type":"split","data":{"type":"split","label":"Split","config":{}}},
 {"id":"4","type":"stop_loss","data":{"type":"stop_loss","label":"Stop Loss","config":{"stop_type":"percentage","stop_value":3,"atr_period":null,"direction":"long"}}},
 {"id":"5","type":"profit_target","data":{"type":"profit_target","label":"Profit Target","config":{"target_type":"percentage","target_value":6,"atr_period":null,"direction":"long"}}},
 {"id":"6","type":"or","data":{"type":"or","label":"Or","config":{}}},
 {"id":"7","type":"sell","data":{"type":"sell","label":"Sell","config":{"direction":"exit_long","order_type":"market","amount_type":"percent_position","amount_value":100}}}],
 "edges":[
 {"source":"0","target":"1","sourceHandle":"out0","targetHandle":"in0"},
 {"source":"1","target":"2","sourceHandle":"out0","targetHandle":"in0"},
 {"source":"2","target":"3","sourceHandle":"out0","targetHandle":"in0"},
 {"source":"3","target":"4","sourceHandle":"out0","targetHandle":"data_in"},
 {"source":"3","target":"5","sourceHandle":"out1","targetHandle":"data_in"},
 {"source":"4","target":"6","sourceHandle":"out0","targetHandle":"in0"},
 {"source":"5","target":"6","sourceHandle":"out0","targetHandle":"in1"},
 {"source":"6","target":"7","sourceHandle":"out0","targetHandle":"in0"}],
 "groups":[]}
```

To keep a signal exit as well, feed this Or's output and the exit signal into a second Or that drives the Sell. A Trailing Stop node takes no `data_in`: it trails the running high on its own and its signal joins the Or the same way.

## 7. A formula entry (internal bar strength)

IBS = (close - low) / (high - low), computed by a Math node from three Price nodes. The Math output goes through a Split: one branch compares it with 0.2 (buy on the bar it crosses below), the other with 0.8 (sell while it sits above). Numeric Value nodes supply the constants as series.

```json
{"nodes":[
 {"id":"0","type":"price","data":{"type":"price","label":"Price","config":{"priceSource":"close","lag":0}}},
 {"id":"1","type":"price","data":{"type":"price","label":"Price","config":{"priceSource":"low","lag":0}}},
 {"id":"2","type":"price","data":{"type":"price","label":"Price","config":{"priceSource":"high","lag":0}}},
 {"id":"3","type":"math","data":{"type":"math","label":"Math","config":{"formula":"(input1 - input2) / (input3 - input2)","inputCount":3,"display":"hidden"}}},
 {"id":"4","type":"split","data":{"type":"split","label":"Split","config":{}}},
 {"id":"5","type":"numeric_value","data":{"type":"numeric_value","label":"Numeric Value","config":{"value":0.2,"mode":"value","percentSource":"equity"}}},
 {"id":"6","type":"crossover_detection","data":{"type":"crossover_detection","label":"Crossover","config":{"mode":"cross","direction":"bearish"}}},
 {"id":"7","type":"buy","data":{"type":"buy","label":"Buy","config":{"direction":"long","order_type":"market","amount_type":"percent_cash","amount_value":100}}},
 {"id":"8","type":"numeric_value","data":{"type":"numeric_value","label":"Numeric Value","config":{"value":0.8,"mode":"value","percentSource":"equity"}}},
 {"id":"9","type":"crossover_detection","data":{"type":"crossover_detection","label":"Crossover","config":{"mode":"state","direction":"bullish"}}},
 {"id":"10","type":"sell","data":{"type":"sell","label":"Sell","config":{"direction":"exit_long","order_type":"market","amount_type":"percent_position","amount_value":100}}}],
 "edges":[
 {"source":"0","target":"3","sourceHandle":"out0","targetHandle":"in0"},
 {"source":"1","target":"3","sourceHandle":"out0","targetHandle":"in1"},
 {"source":"2","target":"3","sourceHandle":"out0","targetHandle":"in2"},
 {"source":"3","target":"4","sourceHandle":"out","targetHandle":"in0"},
 {"source":"4","target":"6","sourceHandle":"out0","targetHandle":"in0"},
 {"source":"5","target":"6","sourceHandle":"out0","targetHandle":"in1"},
 {"source":"4","target":"9","sourceHandle":"out1","targetHandle":"in0"},
 {"source":"8","target":"9","sourceHandle":"out0","targetHandle":"in1"},
 {"source":"6","target":"7","sourceHandle":"out0","targetHandle":"in0"},
 {"source":"9","target":"10","sourceHandle":"out0","targetHandle":"in0"}],
 "groups":[]}
```

The Math formula reads `input1` for `in0`, `input2` for `in1` and so on; every input the formula names must be wired. Math's output handle is `out`, not `out0`.

## 8. Time-of-day entry and exit (intraday only)

Buy 30 minutes after the open on weekdays and sell 15 minutes before the close. The Schedule node needs an interval below 1d. Its config carries every key of both modes; the keys of the unused mode hold their defaults or null as the reference marks them.

```json
{"nodes":[
 {"id":"0","type":"schedule","data":{"type":"schedule","label":"Schedule","config":{"mode":"event","timezone":"America/New_York","daysOfWeek":[false,true,true,true,true,true,false],"anchorTime":"market_open","manualTime":null,"offsetMinutes":30,"offsetSign":"+","startAnchorTime":"market_open","startManualTime":null,"startOffsetMinutes":0,"startOffsetSign":"+","endAnchorTime":"market_close","endManualTime":null,"endOffsetMinutes":0,"endOffsetSign":"+"}}},
 {"id":"1","type":"buy","data":{"type":"buy","label":"Buy","config":{"direction":"long","order_type":"market","amount_type":"percent_cash","amount_value":100}}},
 {"id":"2","type":"schedule","data":{"type":"schedule","label":"Schedule","config":{"mode":"event","timezone":"America/New_York","daysOfWeek":[false,true,true,true,true,true,false],"anchorTime":"market_close","manualTime":null,"offsetMinutes":15,"offsetSign":"-","startAnchorTime":"market_open","startManualTime":null,"startOffsetMinutes":0,"startOffsetSign":"+","endAnchorTime":"market_close","endManualTime":null,"endOffsetMinutes":0,"endOffsetSign":"+"}}},
 {"id":"3","type":"sell","data":{"type":"sell","label":"Sell","config":{"direction":"exit_long","order_type":"market","amount_type":"percent_position","amount_value":100}}}],
 "edges":[
 {"source":"0","target":"1","sourceHandle":"out0","targetHandle":"in0"},
 {"source":"2","target":"3","sourceHandle":"out0","targetHandle":"in0"}],
 "groups":[]}
```

Combine a schedule with a condition through an And node (two signal inputs) feeding the Buy.
