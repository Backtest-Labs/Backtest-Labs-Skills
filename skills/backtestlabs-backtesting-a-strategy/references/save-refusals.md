# Save refusals and run problems

`create_strategy` and `update_strategy` answer two lists. `errors` means the document was not saved: fix each named path and resend. `run_problems` means it was saved as work in progress, as the editor saves it, and `run_backtest` refuses it until each is fixed. Both name the path of the value they concern.

## Errors (the save is refused)

| Refusal | Fix |
|---|---|
| A character outside Latin letters, digits, spaces and ASCII punctuation in `strategy_name` (seen: an em dash) | Replace it with a hyphen or a space. |
| A validation section missing a validation (seen: `validation.parameterSensitivity is required` after the four plan-gated blocks were dropped from a Hub document) | A present `validation` section carries all ten validations with every key, enabled or not. Leave the section out entirely to take the skeleton's, or send it complete. |
| A global section missing a key, or carrying one the reference does not list | Send the section with exactly the skeleton's keys. `execution_mode` and `execution_interval` may be left out of `data_source`; they are filled in. |
| `execution_mode` not matching the interval (`intraday` below 1d, `eod` at 1d and above) or `execution_interval` differing from `interval` | Leave both out. |
| `years_value` not null when the definition is not `last_n_years`; a key other than `definition` and `years_value` under `last_n_years` | Follow the period definition's keys exactly. |
| A value of the wrong kind, outside its enum, or null where not allowed; an int written with a decimal point (12.0) | Match the field's `kind` from the document reference. |
| The same ticker listed twice; a stock beside a crypto pair; crypto pairs of two quote families | One asset class, each ticker once, one quote family (USD, USDT, USDC, DAI and FDUSD count as one). |
| `walkForward.selectedParams` with more than two entries or a malformed entry; `parameterSensitivity.surfaceAxes` without a valid mode | See the format in the document reference's `validation_settings`. |
| Margin values out of the asset class's range; `call_threshold` below `maintenance_requirement`; maintenance at or above 100 / leverage | Checked only while margin is enabled; keep the order call >= maintenance. |

## Run problems (saved, refused to run)

| Problem | Fix |
|---|---|
| No ticker; a ticker the ticker lists do not hold; a `daily_only` ticker on an intraday interval | `search_tickers`. |
| A `required_to_run` input left unwired (Buy and Sell `in0`, Crossover and Relation both inputs, Direction Filter both, Capture top, a Math input its formula reads) | Wire it. |
| No Buy node or no Sell node | Add the missing order node. |
| A node the interval does not support (`vwap`, `schedule` below 1d only; `dividend`, `earnings_event`, `split_event` at 1d and above only) | Change the interval or drop the node. |
| A short sale (`sell` direction `short`) without `execution.margin.enabled` | Enable margin or make it `exit_long`. |
| A number outside its range, a fraction in a whole-number field, a required number left blank | The field's bounds in the document reference. |
| A Walk Forward parameter whose range leaves its `tunable_keys` range, or whose sweep would break a node's own rules (a fast period at or above the slow one, oversold at or above overbought) | Narrow the range. |
| A stop, target or trailing percentage of 100 or more on the long side | Lower it; the level would be at or below zero. |

## Reading a refusal

The path tells the layer: `strategy_graph.nodes[i].data.config.<key>` is a node field, `data_source.*`, `backtest_period.*`, `portfolio.*`, `execution.*` are global settings, `validation.<name>.<key>` a validation setting. Fix only the named value and resend the whole document; the save is all or nothing.
