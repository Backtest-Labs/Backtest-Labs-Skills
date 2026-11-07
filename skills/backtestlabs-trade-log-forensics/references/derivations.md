# Derivations from the trade log

`get_trades` columns: `chain_id`, `trade_id`, `ticker`, `action`, `timestamp`, `quantity`, `price`, `order_type`, `commission`, `slippage`, `realized_pnl`, `pnl_pct`, `r_multiple`, `holding_period_hours`, `mae_value`, `mfe_value`. Money is in the run's quote currency.

## Row kinds

| `action` | Row | Fields that are set |
|---|---|---|
| BUY, SHORT_ENTRY | entry fill | quantity, price, commission, slippage; P&L fields empty |
| SELL, SHORT_COVER | closing fill | all fields; `realized_pnl` is the P&L of the units closed, before the fill's own commission and slippage, which sit in their own columns |
| DRIP | dividend reinvested into shares, on its position's chain | quantity (shares bought), price; `realized_pnl` 0 |
| DIVIDEND_CASH, DIVIDEND_IN_LIEU | dividend received, or paid on a short (negative); `chain_id` `<TICKER>_DIVIDENDS` | quantity (shares held or short), price (per share), `realized_pnl` |
| MARGIN_INTEREST, SHORT_BORROW_COST | carrying cost charged; empty `chain_id` | `realized_pnl` negative |
| MARGIN_CALL | the position fell below the call level, no fill | price |

Rows are ordered by chain, then time; rows outside a numbered chain come first. Rows are paginated 200 at a time: fetch every page.

A chain `<TICKER>_n` is one position: consecutive entries in the same direction merge, so a chain can hold several entry fills and one or more closing fills. The closing fill's `realized_pnl` and excursions are against the weighted average entry of the units it closes.

## Per closing fill

```
entry_cost      = realized_pnl / (pnl_pct / 100)            (pnl_pct != 0)
avg_entry_price = price - realized_pnl / quantity            (long)
                = price + realized_pnl / quantity            (short cover)
MAE_pct         = |mae_value| / entry_cost * 100             (mae_value is stored negative)
MFE_pct         = mfe_value / entry_cost * 100
initial_risk    = realized_pnl / r_multiple                  (r_multiple != 0)
MAE_R           = |mae_value| / initial_risk
MFE_R           = mfe_value / initial_risk
MFE_captured    = realized_pnl / mfe_value                   (mfe_value > 0)
give_back_pct   = MFE_pct - pnl_pct                          (how far it fell from its best)
```

When `pnl_pct` is 0 (a breakeven trade) use `quantity * avg_entry_price` from a neighbouring formula or skip the row for ratios. When `r_multiple` is 0 or empty, skip the R forms.

`initial_risk` is the run's risk anchor applied to the entry (`portfolio.risk_anchor` in the strategy document: a percent of entry, a money distance, or an ATR multiple). It is analytics only; it never closed a trade.

## Per chain

```
chain_cost      = sum(commission + slippage) over every fill in the chain
chain_pnl       = sum(realized_pnl) over the chain's closing fills
net_chain_pnl   = chain_pnl - chain_cost
cost_share      = chain_cost / |chain_pnl|                   (chain_pnl != 0)
legs            = count of entry fills
```

## Aggregates

```
gross_profit    = sum(realized_pnl where realized_pnl > 0)
gross_loss      = |sum(realized_pnl where realized_pnl < 0)|
total_cost      = sum(commission + slippage) over all fills + |carrying-cost rows|
cost_over_gross = total_cost / gross_profit
loss_streak     = longest run of consecutive closing fills with realized_pnl < 0, in trade_id order
```

Percentiles: sort the values, take the value at rank ceil(p/100 x n). Report the 50th, 75th and 90th with n.

## Windows

A closing fill belongs to an underwater period when its `timestamp` is at or after the period's `start` and at or before its `end` (from the `drawdowns` section). Its month is the `timestamp`'s calendar month, which matches `periodic_returns`.

Weekday of exit: from `timestamp`. Stock intraday timestamps are the bar's start time in New York; daily bars carry midnight.
