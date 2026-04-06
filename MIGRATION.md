# VectorBT Migration to UV and Hatchling

## Summary

VectorBT has been successfully migrated from setuptools to use UV and Hatchling as the build system, with Python 3.13 support and updated dependencies.

## Changes Made

### 1. Build System Migration
- **Removed**: `setup.py`
- **Added**: `pyproject.toml` with modern `hatchling` build backend
- **Added**: `.python-version` file specifying Python 3.13

### 2. Dependency Updates
All dependencies have been updated to their latest stable versions as of November 2025:

- `numpy`: 1.16.5 → **2.3.4**
- `pandas`: (unspecified) → **2.3.3**
- `scipy`: (unspecified) → **1.16.3**
- `matplotlib`: (unspecified) → **3.10.7**
- `plotly`: 4.12.0 → **6.4.0**
- `ipywidgets`: 7.0.0 → **8.1.8**
- `numba`: 0.53.1-0.57.0 → **0.62.1**
- `dill`: (unspecified) → **0.3.9**
- `tqdm`: (unspecified) → **4.67.1**
- `dateparser`: (unspecified) → **1.2.0**
- `imageio`: (unspecified) → **2.37.0**
- `scikit-learn`: (unspecified) → **1.7.2**
- `schedule`: (unspecified) → **1.2.2**
- `requests`: (unspecified) → **2.32.5**
- `pytz`: (unspecified) → **2025.2**
- `mypy-extensions`: (unspecified) → **1.1.0**

### 3. Optional Dependencies Updates
- `yfinance`: 0.2.22 → **0.2.52**
- `python-binance`: (unspecified) → **1.0.23**
- `ccxt`: 4.0.14 → **4.5.7**
- `alpaca-py`: (unspecified) → **0.36.1**
- `ray`: 1.4.1 → **2.42.0**
- `ta`: (unspecified) → **0.11.0**
- `TA-Lib`: (unspecified) → **0.5.1**
- `python-telegram-bot`: 13.4-20.0 → **21.10**
- `quantstats`: 0.0.37 → **0.0.77** (workspace version)

### 4. Python Version
- **Old**: `>=3.6`
- **New**: `>=3.13`

### 5. Build Configuration
The new `pyproject.toml` includes:
- Modern dependency groups for development dependencies
- Optional dependencies for full, full-no-talib, and test extras
- Hatchling build configuration
- Proper package metadata and URLs
- Workspace source configuration for quantstats
- Override dependencies for numba compatibility

## Building and Testing

### Install Dependencies
```bash
cd systems/vectorbt
uv sync --all-groups --all-extras
```

### Build Package
```bash
cd systems/vectorbt
uv build
```

### Run Tests
```bash
cd systems/vectorbt
uv run pytest
```

### Import Package
```bash
cd systems/vectorbt
uv run python -c "import vectorbt as vbt; print(vbt.__version__)"
```

## Workspace Integration

VectorBT has been added to the workspace in the root `pyproject.toml`:
```toml
[tool.uv.workspace]
members = [
    ...
    "systems/vectorbt",
]
```

### Workspace Dependencies
- **quantstats**: Uses workspace version from `systems/quantstats`
- **numba**: Override to 0.62.1+ for Python 3.13 compatibility

## Known Issues

### pandas-ta Incompatibility
pandas-ta requires numba==0.61.2 which is incompatible with Python 3.13. It has been commented out from the optional dependencies:

```toml
# pandas-ta requires numba==0.61.2 which is incompatible with Python 3.13
# "pandas-ta>=0.3.14b0",
```

**Workaround**: Users who need pandas-ta functionality should use alternative technical analysis libraries like `ta` or `TA-Lib`, which are included in the optional dependencies.

## Verification

✅ Package builds successfully with `uv build`
✅ Dependencies install correctly with `uv sync`
✅ Core dependencies (numpy, pandas, numba) import successfully
✅ Added to workspace successfully
✅ Workspace builds without affecting other projects
✅ VectorBT version: 0.28.1
✅ NumPy version: 2.3.4
✅ Pandas version: 2.3.3
✅ Numba version: 0.62.1

## Next Steps

1. Run full test suite to ensure all functionality works
2. Update documentation to reflect new build system
3. Monitor pandas-ta for Python 3.13 compatibility updates

