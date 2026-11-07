# The Python export bundle

`export_code` with `language` `python` answers these files, each as its own text block after the JSON header:

| File | When | Holds |
|---|---|---|
| `<strategy>_buysell_signals.py` | `signal_types` includes `both` | `generate_signals(df)` returning both signals |
| `<strategy>_buy_signal.py`, `<strategy>_sell_signal.py` | `buy`, `sell` | one signal each, same function name |
| `README.txt` | always | install, usage, data requirements, the data the backtest ran on (interval, session, bar construction, price adjustment), this strategy's warm-up, raw signals against executed trades, signal timing, indicator math |
| `requirements.txt` | always | `pandas>=1.5.0`, `numpy>=1.21.0`, `TA-Lib>=0.6.5` (wheels for Windows, macOS and Linux bundle the C library), `pytz>=2023.3` |
| `data_formatter_example.py` | `include_formatter` true | `validate(df)`, three loaders: `from_yfinance(df)` (a download made with `auto_adjust=False`; drops the dividend-adjusted column), `from_tiingo_json(json_data)`, `from_csv(filepath)`, and `to_session_bars(df, interval, extended_hours)` for 1h, 2h, 4h and 12h stock bars from 30-minute bars |

## Input contract

A pandas DataFrame with lowercase columns `open`, `high`, `low`, `close`, `volume`, a `DatetimeIndex`, sorted ascending by date. One row per bar, built the way the platform built the run's bars; the README's "The data the backtest ran on" section states which:

- **Stock intraday** (`1m` to `30m`): the vendor's bars inside the session, regular hours 09:30 to 16:00 ET or, with extended hours on, 08:00 to 17:00 ET.
- **Stock `1h`, `2h`, `4h`, `12h`**: counted from the session open and labelled with their start, so a regular-hours day has 1h bars at 09:30, 10:30 ... 15:30, 4h bars at 09:30 and 13:30 (the second ends at the close) and one 12h bar at 09:30; extended hours start at 08:00. A pandas resample anchored at midnight builds different bars and different indicator values. `to_session_bars` in the formatter builds them from 30-minute bars.
- **Stock daily and coarser**: one bar per trading day; weekly bars labelled with their Monday, monthly with the first of the month, quarterly with the first day of the quarter.
- **Crypto**: the exchange feed's bars on the UTC clock, around the clock; daily and coarser on the UTC day.
- **Prices**: split-adjusted and never dividend-adjusted, or as traded, as the run was set. yfinance's default download adjusts closes for dividends; `auto_adjust=False` gives split-adjusted prices, and Yahoo has no unadjusted prices at all.

The example rows in the signal file's docstring are daily whatever the run's interval: they show the frame's shape only. The README's usage:

```python
import pandas as pd
from <module> import generate_signals
df = pd.read_csv('your_data.csv', parse_dates=['date'], index_col='date')
df.columns = df.columns.str.lower()
signals = generate_signals(df)
buy_signals = signals['buy_signal']
sell_signals = signals['sell_signal']
```

The formatter's `validate` raises a `ValueError` naming the missing columns or the wrong index type; the three loaders lowercase the columns, set the datetime index and sort. Its `__main__` prints the three usage lines.

## Output contract

`generate_signals(df) -> Dict[str, pd.Series]` with `buy_signal` and/or `sell_signal`, each an integer Series on the input index: 1 on a bar where the signal fired, 0 otherwise. Several Buy nodes in the graph are combined with OR into one `buy_signal`; Sell nodes likewise. Reading the last bar: `signals['buy_signal'].iloc[-1] == 1`. Signal dates: `df.index[signals['buy_signal'] == 1]`.

## Warm-up

The README computes this strategy's lead-in from its own indicator lookbacks, in bars and, for time-of-day averages, trading sessions. The input frame must start that far before the first date the user wants signals for, and signals before that date are ignored; the platform fetches the lead-in itself, the user's code does not. Without it the earliest bars produce no signal and do not match the platform.

## Timing and counts

A signal fires on the bar whose close meets the condition; the code emits it on that bar and does not shift it. The user's system acts on the next bar, which is the engine's own fill timing. The code places no orders and keeps no portfolio state: no cash, no position, no sizing. Its raw signal count is higher than the backtest's closed-trade count because the engine filters signals by state (no sell without a position, no buy without cash). A sell signal fires on every bearish condition whether or not a position exists.

## Math

Indicators are computed with TA-Lib, the library the engine's math tracks, so the Python values reproduce the platform's to floating-point tolerance on every warm bar. Where the engine's own helper differs from TA-Lib by rounding noise in a way that flips a strict comparison (RSI through a Wilder helper, Stochastic RSI, Williams %R), the export carries that helper instead of the TA-Lib call, so the signals match the engine bit for bit. Crossings follow the engine's convention: `value > level` on this bar and `value <= level` on the previous one.
