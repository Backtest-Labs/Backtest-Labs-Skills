# Exit graphs

Node shapes as the document reference lists them. Every config carries every key; `atr_period` is null unless the type is `atr`. Ids are digit strings; replace the entry and signal-exit nodes with the strategy's own and keep the ids consistent in the edges.

## Reference price for a stop or target

Stop Loss and Profit Target measure their level from `data_in`; unwired, they use the close, which makes the level drift with price. Wire the entry: the Buy node's output into a Capture node (`mode` `last`, `initialValue` null). A signal-only wire carries no value, so Capture stores the close of the bar the buy signal fired on and holds it until the next capture; the value is available from the bar after. Fan the captured series out with a Split to each level node's `data_in`.

```
Buy.out0 -> Capture.in0 -> Split.in0
Split.out0 -> StopLoss.data_in
Split.out1 -> ProfitTarget.data_in
StopLoss.out0 -> Or.in0 ; ProfitTarget.out0 -> Or.in1 ; Or.out0 -> Sell.in0
```

An output takes one wire: a Buy that already feeds something else needs a Split after it.

## Stop loss only, keeping the signal exit

```json
{"nodes":[
 {"id":"E","type":"<entry node>","data":{...}},
 {"id":"1","type":"buy","data":{"type":"buy","label":"Buy","config":{"direction":"long","order_type":"market","amount_type":"percent_cash","amount_value":100}}},
 {"id":"2","type":"capture","data":{"type":"capture","label":"Capture","config":{"mode":"last","initialValue":null}}},
 {"id":"3","type":"stop_loss","data":{"type":"stop_loss","label":"Stop Loss","config":{"stop_type":"percentage","stop_value":3.5,"atr_period":null,"direction":"long"}}},
 {"id":"X","type":"<signal exit node>","data":{...}},
 {"id":"4","type":"or","data":{"type":"or","label":"Or","config":{}}},
 {"id":"5","type":"sell","data":{"type":"sell","label":"Sell","config":{"direction":"exit_long","order_type":"market","amount_type":"percent_position","amount_value":100}}}],
 "edges":[
 {"source":"E","target":"1","sourceHandle":"out0","targetHandle":"in0"},
 {"source":"1","target":"2","sourceHandle":"out0","targetHandle":"in0"},
 {"source":"2","target":"3","sourceHandle":"out0","targetHandle":"data_in"},
 {"source":"3","target":"4","sourceHandle":"out0","targetHandle":"in0"},
 {"source":"X","target":"4","sourceHandle":"out0","targetHandle":"in1"},
 {"source":"4","target":"5","sourceHandle":"out0","targetHandle":"in0"}],
 "groups":[]}
```

Capture's output is a single wire, so with only one level node no Split is needed.

## Stop and target, keeping the signal exit

Two Or nodes: the first merges stop and target, the second merges that with the signal exit.

```
Capture.out0 -> Split.in0 ; Split.out0 -> StopLoss.data_in ; Split.out1 -> ProfitTarget.data_in
StopLoss.out0 -> Or1.in0 ; ProfitTarget.out0 -> Or1.in1
Or1.out0 -> Or2.in0 ; SignalExit.out0 -> Or2.in1 ; Or2.out0 -> Sell.in0
```

Profit Target config: `{"target_type":"percentage","target_value":<t>,"atr_period":null,"direction":"long"}`.

## Trailing stop

No reference input. `{"trail_type":"percentage","trail_value":<d>,"atr_period":null,"direction":"long"}`; its `out0` joins the Or like a stop. The level is the running high minus the distance and only tightens; direction `long` tracks the highest high.

## ATR distances

`stop_type` `atr` with `atr_period` set measures the distance as a multiple of the ATR (simple average of true range over the period) from the reference. The log's excursions are in percent of entry; to set an ATR multiple from them, divide the chosen percent by the ticker's typical ATR as a percent of price, read from the ATR node's values (`get_chart` is per run, so build the ATR node, run once, or use the percentage type).

## Short positions

Direction `short` on each level node sits the level above the reference (stop) or below it (target); a Trailing Stop with direction `short` tracks the lowest low. The Sell node that opens the short needs `direction` `short` and margin enabled; the Buy that covers uses `direction` `cover_short`.

## Saving the copy

`create_strategy` with the baseline document, the exit nodes added, and `strategy_name` changed (Latin letters, digits, spaces, ASCII punctuation; no em dash). Keep every other setting identical so the comparison is one change. The answer's `run_problems` must be empty before `run_backtest`.
