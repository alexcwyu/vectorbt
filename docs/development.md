# vectorbt -- Development Guide

## Setup

### Prerequisites

- Python >= 3.13
- uv (recommended) or pip

### Installation

```bash
cd vectorbt

# Development install with uv
uv sync --active --all-groups --all-extras

# Or with pip
pip install -e ".[full]"

# Without TA-Lib (avoids C dependency)
pip install -e ".[full-no-talib]"
```

### Dependencies

**Core**: numpy, pandas, scipy, matplotlib, plotly, ipywidgets, numba, dill, tqdm, dateparser, imageio, scikit-learn, schedule, requests, pytz

**Optional (full)**:
- Data: yfinance, python-binance, ccxt, alpaca-py
- Indicators: ta, TA-Lib
- Distributed: ray
- Messaging: python-telegram-bot
- Analytics: quantstats

### Running Tests

```bash
# All tests
uv run pytest

# With coverage
uv run pytest --cov=vectorbt

# Single test file
uv run pytest tests/test_portfolio.py

# Single test
uv run pytest -k test_function_name
```

### Linting and Type Checking

```bash
# Ruff linting
uv run ruff check src/

# Ruff formatting
uv run ruff format src/

# Type checking
uv run mypy src/vectorbt/
```

## Creating Custom Indicators

### Using IndicatorFactory

The `IndicatorFactory` provides a structured way to create indicators:

```python
import vectorbt as vbt
import numpy as np
from numba import njit

# 1. Define the computation function (must be Numba-compiled for performance)
@njit(cache=True)
def custom_ma_nb(close, window):
    """Simple moving average using Numba."""
    result = np.full_like(close, np.nan)
    for i in range(window - 1, len(close)):
        result[i] = np.mean(close[i - window + 1:i + 1])
    return result

# 2. Create the indicator class
CustomMA = vbt.IndicatorFactory(
    class_name='CustomMA',
    input_names=['close'],
    param_names=['window'],
    output_names=['ma']
).with_apply_func(custom_ma_nb)

# 3. Use it
price = pd.DataFrame({'a': [1, 2, 3, 4, 5], 'b': [5, 4, 3, 2, 1]})
ma = CustomMA.run(price, window=[2, 3])
```

### Using External Libraries

```python
# From TA-Lib
RSI = vbt.indicators.talib('RSI')
rsi = RSI.run(close, timeperiod=[7, 14])

# From pandas-ta
VWAP = vbt.indicators.pandas_ta('VWAP')

# From ta library
BB = vbt.indicators.ta('BollingerBands')
```

### Indicator with Multiple Outputs

```python
@njit(cache=True)
def bb_nb(close, window, std_mult):
    ma = np.full_like(close, np.nan)
    upper = np.full_like(close, np.nan)
    lower = np.full_like(close, np.nan)
    for i in range(window - 1, len(close)):
        w = close[i - window + 1:i + 1]
        m = np.mean(w)
        s = np.std(w)
        ma[i] = m
        upper[i] = m + std_mult * s
        lower[i] = m - std_mult * s
    return ma, upper, lower

MyBB = vbt.IndicatorFactory(
    class_name='MyBB',
    input_names=['close'],
    param_names=['window', 'std_mult'],
    output_names=['ma', 'upper', 'lower']
).with_apply_func(bb_nb)
```

## Creating Custom Signal Generators

### Using SignalFactory

```python
from vectorbt.signals.factory import SignalFactory

# Define entry choice function
@njit
def entry_choice_nb(from_i, to_i, col, close, threshold):
    """Enter when price drops by threshold from recent high."""
    for i in range(from_i, to_i):
        if i > 0 and close[i] < close[i-1] * (1 - threshold):
            return i  # return the chosen index
    return -1  # no entry

# Define exit choice function
@njit
def exit_choice_nb(from_i, to_i, col, close, threshold):
    """Exit when price rises by threshold from entry."""
    for i in range(from_i, to_i):
        if i > 0 and close[i] > close[i-1] * (1 + threshold):
            return i
    return -1

MySig = SignalFactory(
    class_name='MySig',
    input_names=['close'],
    param_names=['threshold']
).with_entry_exit_func(entry_choice_nb, exit_choice_nb)
```

### Direct Signal Arrays

For simpler strategies, create boolean arrays directly:

```python
# Moving average crossover
fast_ma = close.rolling(10).mean()
slow_ma = close.rolling(30).mean()

entries = fast_ma > slow_ma  # golden cross
exits = fast_ma < slow_ma    # death cross

pf = vbt.Portfolio.from_signals(close, entries, exits)
```

## Data Source Integration

### Creating a Custom Data Source

Extend the `Data` base class:

```python
from vectorbt.data.base import Data

class MyData(Data):
    @classmethod
    def download(cls, symbols, start=None, end=None, **kwargs):
        # Fetch data from your source
        data = {}
        for symbol in symbols:
            df = my_api.get_ohlcv(symbol, start, end)
            data[symbol] = df
        return cls(data, **kwargs)
```

### Using DataUpdater

```python
# Initial download
data = vbt.YFData.download(['AAPL', 'GOOGL'], start='2020')

# Set up periodic updates
updater = vbt.DataUpdater(data)
updater.update_every(minutes=5)  # uses schedule library
```

## Performance Optimization Tips

### 1. Use Numba for Custom Logic

Always use `@njit(cache=True)` for functions passed to the portfolio simulator:

```python
from numba import njit

@njit(cache=True)
def my_order_func(oc):
    # oc is OrderContext - a Numba-compatible named tuple
    if oc.position_now == 0 and should_enter(oc):
        return (oc.close[oc.i, oc.col], 1.0, 0)  # price, size, direction
    return (-1, np.nan, 0)  # no order
```

### 2. Vectorize Parameter Sweeps

Instead of looping over parameters in Python, broadcast them:

```python
# BAD - Python loop
results = []
for window in [10, 20, 30, 50, 100]:
    ma = close.rolling(window).mean()
    entries = close > ma
    pf = vbt.Portfolio.from_signals(close, entries)
    results.append(pf.total_return())

# GOOD - vectorized
windows = [10, 20, 30, 50, 100]
ma = vbt.MA.run(close, window=windows)
entries = close.vbt.broadcast_to(ma.ma) > ma.ma
pf = vbt.Portfolio.from_signals(close, entries)
returns = pf.total_return()  # Series indexed by window
```

### 3. Use Records Instead of Matrices

For sparse data, records use orders of magnitude less memory:

```python
# Dense matrix: N_rows x N_cols x sizeof(float)
# Records: N_events x N_fields x sizeof(field_type)

# Access record fields via MappedArray
pf.orders.size.mean()  # efficient aggregation
```

### 4. Minimize Data Copies

- Use `to_2d_array()` to avoid repeated conversions
- Let broadcasting handle shape alignment instead of manual padding
- Use `flex_select_auto_nb` for flexible array indexing in Numba

### 5. Warm Up Numba Cache

First run compiles Numba functions. Subsequent runs use cached bytecode:

```python
# Warm up with small data
small_pf = vbt.Portfolio.from_signals(close.iloc[:10], entries.iloc[:10])

# Now run with full data - compilation is cached
full_pf = vbt.Portfolio.from_signals(close, entries)
```

### 6. Profile with py-spy

```bash
# Flame graph
py-spy record -o profile.svg -- python my_strategy.py

# Top-like view
py-spy top -- python my_strategy.py
```

## Project Structure

```
vectorbt/
├── src/vectorbt/
│   ├── __init__.py          # Top-level imports
│   ├── _settings.py         # Global settings (SettingsConfig)
│   ├── _typing.py           # Type aliases
│   ├── _version.py          # Version string
│   ├── ohlcv_accessors.py   # OHLCV pandas accessors (.vbt.ohlcv)
│   ├── base/                # ArrayWrapper, reshape, indexing, combine
│   ├── data/                # Data sources (YF, Binance, CCXT, Alpaca, synthetic)
│   ├── generic/             # Drawdowns, ranges, splitters, stats/plots builders
│   ├── indicators/          # IndicatorFactory, built-in indicators
│   ├── labels/              # ML labeling utilities
│   ├── messaging/           # Telegram integration
│   ├── portfolio/           # Portfolio simulator, Numba kernels, enums
│   ├── records/             # Records, MappedArray, ColumnMapper
│   ├── returns/             # Return analysis
│   ├── signals/             # SignalFactory, signal generators
│   ├── templates/           # Plotly templates
│   └── utils/               # Config, caching, decorators, datetime, math
├── tests/                   # pytest test suite
├── docs/                    # Documentation
├── examples/                # Example notebooks
├── apps/                    # Applications
└── pyproject.toml           # Project configuration
```

## Configuration Reference

### Portfolio.from_signals() Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `close` | `ArrayLike` | *(required)* | Close price series or DataFrame. |
| `entries` | `ArrayLike \| None` | `None` | Boolean array of entry signals. |
| `exits` | `ArrayLike \| None` | `None` | Boolean array of exit signals. |
| `short_entries` | `ArrayLike \| None` | `None` | Boolean array of short entry signals. |
| `short_exits` | `ArrayLike \| None` | `None` | Boolean array of short exit signals. |
| `size` | `ArrayLike \| None` | `None` | Order size. Interpretation depends on `size_type`. |
| `size_type` | `ArrayLike \| None` | `None` | How to interpret `size`: `'amount'`, `'value'`, `'percent'`, `'targetamount'`, `'targetvalue'`, `'targetpercent'`. |
| `price` | `ArrayLike \| None` | `None` | Execution price. Defaults to `close`. |
| `fees` | `ArrayLike \| None` | `None` | Fees as a fraction of order value (e.g., `0.001` = 0.1%). |
| `fixed_fees` | `ArrayLike \| None` | `None` | Fixed fee per order in cash units. |
| `slippage` | `ArrayLike \| None` | `None` | Slippage as a fraction of price. |
| `min_size` | `ArrayLike \| None` | `None` | Minimum order size. Orders below this are rejected. |
| `max_size` | `ArrayLike \| None` | `None` | Maximum order size. |
| `size_granularity` | `ArrayLike \| None` | `None` | Round order size to multiples of this value (e.g., `1.0` for integer positions). |
| `reject_prob` | `ArrayLike \| None` | `None` | Probability of random order rejection (for robustness testing). |
| `lock_cash` | `ArrayLike \| None` | `None` | Lock cash after each order to prevent overallocation. |
| `allow_partial` | `ArrayLike \| None` | `None` | Allow partial fills when cash is insufficient. |
| `raise_reject` | `ArrayLike \| None` | `None` | Raise an exception on order rejection instead of silently skipping. |
| `accumulate` | `ArrayLike \| None` | `None` | Allow adding to existing positions on repeated entry signals. |
| `direction` | `ArrayLike \| None` | `None` | Trade direction: `'longonly'`, `'shortonly'`, or `'both'`. |
| `sl_stop` | `ArrayLike \| None` | `None` | Stop-loss as a fraction of entry price (e.g., `0.05` = 5%). |
| `sl_trail` | `ArrayLike \| None` | `None` | Trailing stop-loss. If `True`, the SL level follows the price upward. |
| `tp_stop` | `ArrayLike \| None` | `None` | Take-profit as a fraction of entry price (e.g., `0.10` = 10%). |
| `stop_entry_price` | `ArrayLike \| None` | `None` | Price used for stop calculation: `'close'`, `'open'`, or custom. |
| `stop_exit_price` | `ArrayLike \| None` | `None` | Price used for stop exit: `'close'`, `'stoplimit'`, or custom. |
| `upon_stop_exit` | `ArrayLike \| None` | `None` | Behavior on stop exit: `'close'`, `'closereduce'`, `'reverse'`. |
| `upon_opposite_entry` | `ArrayLike \| None` | `None` | Behavior on opposite signal: `'close'`, `'closereduce'`, `'reverse'`, `'ignore'`. |
| `init_cash` | `ArrayLike \| None` | `None` | Initial cash. Default from `vbt.settings.portfolio['init_cash']` (100). |
| `cash_sharing` | `bool \| None` | `None` | Share cash across columns within groups. |
| `freq` | `str \| None` | `None` | Data frequency for annualization (e.g., `'1D'`, `'1H'`). |
| `seed` | `int \| None` | `None` | Random seed for reproducible results (relevant when `reject_prob` is used). |
| `group_by` | `GroupByLike \| None` | `None` | Group columns for cash sharing or joint analysis. |
| `log` | `ArrayLike \| None` | `None` | Enable order logging for debugging. |

### Global Settings (`vbt.settings`)

| Setting Path | Type | Default | Description |
|-------------|------|---------|-------------|
| `portfolio.init_cash` | `float` | `100` | Default initial cash for portfolios. |
| `portfolio.fees` | `float` | `0.0` | Default fee rate. |
| `portfolio.slippage` | `float` | `0.0` | Default slippage rate. |
| `portfolio.freq` | `str \| None` | `None` | Default data frequency. |
| `data.tz_localize` | `str \| None` | `'UTC'` | Default timezone for data. |
| `returns.year_freq` | `str` | `'365 days'` | Frequency used for annualization. |
| `array_wrapper.freq` | `str \| None` | `None` | Default frequency for ArrayWrapper. |

## Troubleshooting

### 1. `ModuleNotFoundError: No module named 'numba'`
**Cause**: Numba is a core dependency not installed in the environment.
**Fix**: Install numba: `pip install numba` or `pip install vectorbt[full]`. Numba requires a compatible LLVM version; use `conda install numba` if pip fails.

### 2. Extremely slow first run, fast subsequent runs
**Cause**: Numba JIT-compiles functions on first invocation. Compiled code is cached for subsequent runs.
**Fix**: This is normal behavior. Warm up with small data to trigger compilation: `vbt.Portfolio.from_signals(close.iloc[:10], entries.iloc[:10])`. Use `cache=True` in custom `@njit` decorators.

### 3. `ValueError: operands could not be broadcast together`
**Cause**: Shape mismatch between close prices and signals (entries/exits).
**Fix**: Ensure entries and exits have the same shape as close. When using multi-parameter indicator sweeps, broadcast close to match: `close_broadcast = close.vbt.broadcast_to(ma.ma)`.

### 4. Statistics show `NaN` for Sharpe ratio or other metrics
**Cause**: Too few data points, zero returns, or missing `freq` parameter prevents annualization.
**Fix**: Set `freq='1D'` (or appropriate frequency) in `from_signals()`. Ensure the backtest period is long enough for meaningful statistics (at least 30 trading days).

### 5. `MemoryError` with large parameter sweeps
**Cause**: Vectorized broadcasting creates large intermediate arrays (N_rows x N_columns x N_params).
**Fix**: Reduce the parameter space. Process parameters in batches. Use records (sparse storage) instead of dense matrices: `pf.orders.records` instead of full-matrix outputs.

### 6. Plotly figures not displaying in Jupyter
**Cause**: Plotly renderer not configured for the notebook environment.
**Fix**: Run `import plotly.io as pio; pio.renderers.default = 'notebook'` or install `nbformat`: `pip install nbformat>=4.2.0`.

### 7. Entries and exits on the same bar produce unexpected behavior
**Cause**: By default, conflicting signals on the same bar follow the `upon_long_conflict` / `upon_short_conflict` resolution rules.
**Fix**: Set conflict resolution explicitly: `upon_long_conflict='ignore'` or `'exit'`. Review the signal arrays to remove simultaneous entry/exit signals.

### 8. Stop-loss not triggering accurately
**Cause**: Without `open`, `high`, `low` data, stops can only be evaluated against `close`, missing intra-bar breaches.
**Fix**: Pass OHLC data: `vbt.Portfolio.from_signals(close, entries, exits, open=open, high=high, low=low, sl_stop=0.05)`.

## Security Considerations

### API Keys and Data Sources
- Built-in data downloaders (YFData, BinanceData, CCXTData, AlpacaData) require API keys or exchange credentials. Never hardcode these in scripts.
- Use environment variables: `os.environ['ALPACA_API_KEY']` or a `.env` file with `python-dotenv`. Add `.env` to `.gitignore`.

### Data Integrity
- Downloaded market data should be validated for completeness and correctness. Missing bars, stock splits, or dividend adjustments can produce misleading backtest results.
- The `SyntheticData` and `GBMData` generators use pseudo-random numbers. Set `seed` for reproducibility, but do not use synthetic data for production decisions.

### Numba Code Execution
- Numba JIT-compiles Python functions to machine code. Custom `@njit` functions execute at the CPU level without Python's safety checks.
- Only JIT-compile trusted code. Do not accept arbitrary Numba functions from untrusted sources, as they can perform unrestricted memory access.

### Pickle and Caching
- vectorbt caches compiled Numba functions and computed results. Cache files are stored in `__pycache__` directories.
- Portfolio objects can be pickled for persistence. Do not pickle objects containing credentials or sensitive data.
- The `dill` dependency (used for serialization) can deserialize arbitrary Python objects. Only load pickled files from trusted sources.

### Telegram Integration
- The messaging module supports Telegram bot notifications. Bot tokens grant full control over the bot. Store tokens securely and rotate them if compromised.
- Telegram messages may contain portfolio values and trade details. Be cautious about sending sensitive financial data over messaging platforms.

### Look-Ahead Bias
- vectorbt's vectorized design computes indicators over full arrays. When used correctly with `from_signals()`, the simulation engine processes bars sequentially. However, building signals from future data (e.g., using `.shift(-1)`) introduces look-ahead bias.
- Always verify that signal arrays only use data available at or before each bar's timestamp.

---
## See Also
- [README](README.md) — Project overview and quick start
- [Architecture](architecture.md) — System design and components
- [Workflow](workflow.md) — Event flows and processing pipelines
- [State Management](state-management.md) — State lifecycle and data models
