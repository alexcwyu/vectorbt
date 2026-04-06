# vectorbt -- Workflow

## Backtesting Pipeline

```mermaid
flowchart TD
    A[1. Load Data] --> B[2. Compute Indicators]
    B --> C[3. Generate Signals]
    C --> D[4. Simulate Portfolio]
    D --> E[5. Analyze Results]
    E --> F[6. Visualize]

    subgraph "Phase 1: Data"
        A1[YFData.download] --> A2[Data.get<br/>extract OHLCV]
    end

    subgraph "Phase 2: Indicators"
        B1[IndicatorFactory.run] --> B2[Parameter Broadcasting]
        B2 --> B3[Numba Computation]
        B3 --> B4[Output Concatenation]
    end

    subgraph "Phase 3: Signals"
        C1[Boolean Entry Arrays] --> C2[Boolean Exit Arrays]
        C2 --> C3[Stop-Loss / Take-Profit]
    end

    subgraph "Phase 4: Simulation"
        D1[Preparation<br/>resolve defaults, broadcast, validate]
        D2[Numba Simulation<br/>traverse time x assets]
        D3[Construction<br/>create Portfolio from records]
    end

    subgraph "Phase 5: Analysis"
        E1[Portfolio.stats]
        E2[Portfolio.orders / trades / drawdowns]
    end

    subgraph "Phase 6: Visualization"
        F1[Portfolio.plot]
        F2[Custom Plotly figures]
    end

    A --> A1 --> A2
    B --> B1 --> B2 --> B3 --> B4
    C --> C1 --> C2 --> C3
    D --> D1 --> D2 --> D3
    E --> E1 & E2
    F --> F1 & F2
```

## Signal Generation to Portfolio Simulation Flow

```mermaid
sequenceDiagram
    participant User
    participant Portfolio
    participant Preparer as Preparation
    participant Numba as Numba Kernel
    participant Records as Record Arrays

    User->>Portfolio: from_signals(price, entries, exits, ...)
    Portfolio->>Preparer: Resolve defaults from settings
    Preparer->>Preparer: Broadcast inputs to common shape
    Preparer->>Preparer: Convert Pandas to NumPy
    Preparer->>Preparer: Validate inputs
    Preparer->>Numba: Pass arrays to simulate_from_signals_nb

    loop For each row (time step)
        loop For each column (asset)
            Numba->>Numba: Check entry/exit signals
            Numba->>Numba: Apply stop-loss / take-profit
            Numba->>Numba: Resolve conflicts (entry + exit same bar)
            Numba->>Numba: Generate Order tuple
            Numba->>Numba: Process order (buy_nb / sell_nb)
            alt Order filled
                Numba->>Records: Append order record
                Numba->>Numba: Update cash & position state
            else Order rejected/ignored
                Numba->>Numba: Continue
            end
        end
    end

    Numba-->>Portfolio: Return order records array
    Portfolio->>Portfolio: Construct Portfolio object
    Portfolio-->>User: Portfolio instance
```

## Data Loading and Preprocessing

```mermaid
flowchart LR
    subgraph "Data Sources"
        YF[Yahoo Finance<br/>YFData]
        BN[Binance<br/>BinanceData]
        CX[CCXT<br/>CCXTData]
        AL[Alpaca<br/>AlpacaData]
        SN[Synthetic<br/>GBMData]
    end

    subgraph "Data Base Class"
        DL[Data.download<br/>symbol_dict mapping]
        UP[DataUpdater<br/>scheduled updates]
    end

    subgraph "Processing"
        GET[Data.get<br/>extract field]
        WR[ArrayWrapper<br/>attach metadata]
        BC[broadcast<br/>align shapes]
    end

    YF & BN & CX & AL & SN --> DL
    DL --> UP
    DL --> GET
    GET --> WR --> BC
```

**Data flow:**

1. **Fetch**: `Data.download('SYMBOL', start, end)` calls the source API (yfinance, ccxt, etc.)
2. **Store**: Returns are stored per-symbol in a `symbol_dict` mapping
3. **Extract**: `Data.get('Close')` extracts a specific field as a DataFrame
4. **Update**: `DataUpdater` supports scheduled re-fetching via the `schedule` library

## Indicator Computation Pipeline

```mermaid
flowchart TD
    INPUT[Input Arrays<br/>e.g. close prices] --> PARAMS[Parameter Grid<br/>e.g. window=2,3,5,10,20]

    PARAMS --> BC[Broadcast<br/>inputs x params]
    BC --> COMPUTE[Compute per combo<br/>Numba @njit or NumPy]
    COMPUTE --> CONCAT[Concatenate outputs<br/>MultiIndex columns]
    CONCAT --> CACHE[Cache results]
    CACHE --> OUTPUT[IndicatorBase instance<br/>with .output, .run() etc.]

    subgraph "IndicatorFactory Pipeline"
        BC
        COMPUTE
        CONCAT
    end

    subgraph "Example: MA"
        MA_IN["close prices (2 assets)"]
        MA_P["window = [10, 20]"]
        MA_OUT["4-column output<br/>(2 assets x 2 windows)"]
        MA_IN --> MA_P --> MA_OUT
    end
```

**Key steps:**

1. `IndicatorFactory` creates a class with `run()` and `run_combs()` methods
2. `run()` broadcasts inputs against parameter arrays
3. For each parameter combination, the custom function is called
4. Results are concatenated with a MultiIndex encoding the parameter values
5. Crossover/signal methods are auto-generated

## Portfolio Optimization Flow

vectorbt supports parameter sweep optimization through broadcasting:

```mermaid
flowchart TD
    PRICE[Price Data] --> SWEEP[Parameter Sweep<br/>e.g. SL/TP grid]
    SWEEP --> SIM[Portfolio.from_signals<br/>vectorized across all combos]
    SIM --> METRICS[Extract metrics<br/>total_return, sharpe, etc.]
    METRICS --> RANK[Rank / filter results]
    RANK --> BEST[Best parameter combination]

    subgraph "Vectorized Sweep"
        direction LR
        P1["SL=1%, TP=2%"] --> C1[Column 1]
        P2["SL=2%, TP=3%"] --> C2[Column 2]
        P3["SL=3%, TP=5%"] --> C3[Column 3]
        PN["..."] --> CN["Column N"]
    end
```

All parameter combinations run in a single simulation call. No Python-level loop required.

## Plotting and Visualization Flow

```mermaid
flowchart TD
    PF[Portfolio] --> PB[PlotsBuilder]
    PF --> SB[StatsBuilder]

    PB --> |"plot()"| FIG[Plotly Figure]
    PB --> |"plot() subplots"| MULTI[Multi-subplot Figure]

    SB --> |"stats()"| SR[Pandas Series<br/>metric summary]

    PF --> |"orders.plot()"| ORD_FIG[Orders Plot]
    PF --> |"trades.plot()"| TRD_FIG[Trades Plot]
    PF --> |"drawdowns.plot()"| DD_FIG[Drawdowns Plot]

    FIG --> SHOW[fig.show]
    FIG --> SAVE[fig.write_image<br/>via imageio]
    FIG --> HTML[fig.write_html]
```

**Visualization features:**
- `Portfolio.plot()` -- equity curve with trade markers
- `Portfolio.drawdowns.plot()` -- drawdown periods
- `Portfolio.orders.plot()` -- order execution on price chart
- Custom subplots via `PlotsBuilder`
- Export to PNG, HTML, or interactive display

---
## See Also
- [README](README.md) — Project overview and quick start
- [Architecture](architecture.md) — System design and components
- [State Management](state-management.md) — State lifecycle and data models
- [Development](development.md) — Development guide and best practices
