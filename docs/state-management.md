# vectorbt -- State Management

## Portfolio State Tracking

During simulation, vectorbt maintains state per asset column using Numba-compiled named tuples. State is updated in-place as the simulation traverses the time x assets matrix.

### Simulation State Hierarchy

```mermaid
stateDiagram-v2
    [*] --> SimulationContext
    SimulationContext --> GroupContext : per group
    GroupContext --> RowContext : per time step
    GroupContext --> SegmentContext : per segment
    SegmentContext --> OrderContext : per order
    OrderContext --> PostOrderContext : after fill

    state SimulationContext {
        target_shape: "(n_rows, n_cols)"
        close: "price array"
        init_cash: "starting capital"
        order_records: "output array"
        log_records: "output array"
    }

    state GroupContext {
        group: "column group index"
        cash_now: "current cash balance"
        position_now: "current position per col"
        value_now: "current portfolio value"
    }

    state OrderContext {
        col: "asset column"
        i: "time index"
        signal: "entry/exit/size"
        price: "current price"
        fees: "fee rate"
        slippage: "slippage rate"
    }
```

### State Types (from `portfolio.enums`)

| Type | Fields | Purpose |
|------|--------|---------|
| `SimulationContext` | target_shape, close, init_cash, order_records, log_records | Top-level simulation context |
| `GroupContext` | group, group_len, cash_sharing, ... | Per-group state (for cash sharing) |
| `RowContext` | i (row index) | Current time step |
| `SegmentContext` | extends GroupContext with call_seq | Segment within a group |
| `OrderContext` | col, i, position_now, exec_state, signal info | Context for order decision |
| `PostOrderContext` | extends OrderContext with OrderResult | Context after order processing |
| `FlexOrderContext` | col, i, position_now, exec_state | Flexible order function context |
| `ExecuteOrderState` | cash, position, debt, locked_cash, free_cash | Execution state for buy/sell |
| `ProcessOrderState` | cash, position, debt, locked_cash, free_cash, val_price, value | Full process state |

### State Update Flow

```mermaid
flowchart TD
    INIT[Initialize State<br/>cash=init_cash<br/>position=0] --> LOOP

    LOOP[For each time step i] --> GROUP[For each group g]
    GROUP --> SEG[Create SegmentContext]
    SEG --> COL[For each column in call_seq]

    COL --> ORDER{Generate Order?}
    ORDER -->|Yes| PROCESS[process_order_nb]
    ORDER -->|No/Skip| NEXT[Next column]

    PROCESS --> FILL{Order Result?}
    FILL -->|Filled| UPDATE[Update State<br/>cash += / -=<br/>position += / -=]
    FILL -->|Rejected| LOG_REJ[Log rejection reason]
    FILL -->|Ignored| NEXT

    UPDATE --> RECORD[Append to order_records]
    RECORD --> NEXT
    LOG_REJ --> NEXT

    NEXT --> |More columns| COL
    NEXT --> |Done| LOOP2[Next time step]
    LOOP2 --> |More rows| LOOP
    LOOP2 --> |Done| DONE[Return Records]
```

## Order and Trade Records Management

### Record Data Types

Records are NumPy structured arrays with fixed schemas:

**Order Record (`order_dt`)**:
- `id`: Unique order identifier
- `col`: Column (asset) index
- `idx`: Row (time) index
- `size`: Executed size
- `price`: Execution price
- `fees`: Total fees paid
- `side`: Buy (0) or Sell (1)

**Trade Record (`trade_dt`)**:
- `id`, `col`, `size`, `entry_idx`, `entry_price`, `entry_fees`
- `exit_idx`, `exit_price`, `exit_fees`
- `pnl`, `return_`, `direction`, `status` (open/closed)

**Log Record (`log_dt`)**:
- Complete snapshot of state before and after each order attempt
- Includes cash, position, debt, value, order details, and result

### Records Architecture

```mermaid
classDiagram
    class Records {
        +records_arr: np.ndarray
        +wrapper: ArrayWrapper
        +col_mapper: ColumnMapper
        +records_readable: pd.DataFrame
        +map_field(field) MappedArray
        +count() int
        +filter_by_mask(mask) Records
    }

    class MappedArray {
        +mapped_arr: np.ndarray
        +col_arr: np.ndarray
        +idx_arr: np.ndarray
        +mean() float/Series
        +sum() float/Series
        +min() float/Series
        +max() float/Series
        +std() float/Series
        +to_matrix() pd.DataFrame
    }

    class ColumnMapper {
        +col_arr: np.ndarray
        +get_col_map() tuple
    }

    class Orders {
        +size: MappedArray
        +price: MappedArray
        +fees: MappedArray
        +side: MappedArray
        +buy_records: Orders
        +sell_records: Orders
    }

    class Trades {
        +pnl: MappedArray
        +returns: MappedArray
        +duration: MappedArray
        +win_rate: float
        +profit_factor: float
        +expectancy: float
    }

    class Logs {
        "Full state snapshots"
    }

    Records <|-- Orders
    Records <|-- Trades
    Records <|-- Logs
    Records --> ColumnMapper
    Records --> MappedArray
```

## Signal State Machine

Signal processing in `from_signals` follows a state machine pattern:

```mermaid
stateDiagram-v2
    [*] --> NoPosition

    NoPosition --> Long : entry signal (longonly/both)
    NoPosition --> Short : entry signal (shortonly)

    Long --> NoPosition : exit signal
    Long --> NoPosition : stop-loss triggered
    Long --> NoPosition : take-profit triggered
    Long --> Long : accumulate=True + entry

    Short --> NoPosition : exit signal
    Short --> NoPosition : stop-loss triggered
    Short --> NoPosition : take-profit triggered
    Short --> Short : accumulate=True + entry

    Long --> Short : direction=Both + opposite entry
    Short --> Long : direction=Both + opposite entry

    state "Conflict Resolution" as CR {
        EntryAndExit --> ConflictMode
        ConflictMode --> KeepEntry : "Entry"
        ConflictMode --> KeepExit : "Exit"
        ConflictMode --> KeepOpposite : "Opposite"
    }
```

**Key enums controlling signal behavior:**

| Enum | Values | Purpose |
|------|--------|---------|
| `Direction` | Both, LongOnly, ShortOnly | Allowed position directions |
| `ConflictMode` | Entry, Exit, Opposite | When entry + exit on same bar |
| `AccumulationMode` | Disabled, Both, AddOnly, RemoveOnly | Whether to add to positions |
| `OppositeEntryMode` | Ignore, Close, Reverse | Entry in opposite direction |
| `StopExitMode` | Close, CloseReduce, Reverse | How stops exit positions |
| `StopUpdateMode` | Keep, Override, OverrideExit | How stop levels update |

## Cache Management

### Settings-Based Caching

vectorbt uses a decorator-based caching system controlled via `settings['caching']`:

```python
# Enable/disable caching globally
vbt.settings['caching']['enabled'] = True

# Cache condition controls when caching applies
# CacheCondition from utils.decorators
```

### Caching Levels

1. **Numba JIT Cache** (`cache=True` in `@njit`): Compiled function bytecode cached to disk. Persists across sessions.

2. **Property Cache**: Computed properties (equity curves, metrics) are cached on the Portfolio instance. Invalidated when the object is replaced.

3. **Indicator Cache**: `IndicatorFactory` caches intermediate computation results during `run_pipeline`.

4. **Settings Cache**: The `settings` Config object supports save/load to disk for persistence.

### Cache Lifecycle

```mermaid
flowchart TD
    REQ[Request property<br/>e.g. pf.total_return] --> CHECK{Cached?}
    CHECK -->|Yes| RET[Return cached value]
    CHECK -->|No| COMPUTE[Compute from records]
    COMPUTE --> STORE[Store in cache]
    STORE --> RET

    REPLACE[pf.replace<br/>new parameters] --> INVALIDATE[New Portfolio instance<br/>fresh cache]
    
    GLOBAL[settings change] --> NOTE[Only affects future instances<br/>existing cache untouched]
```

### Settings Persistence

```python
# Save settings to disk
vbt.settings.save('my_settings')

# Load and update in-place
vbt.settings.load_update('my_settings')

# Sub-config save/load also supported
vbt.settings.plotting.save('my_plot_settings')
```

Settings are hierarchical `Config` objects with frozen keys (cannot add/remove sub-configs, only modify values). Each sub-config can be either `dict`-inheriting or its own `Config` instance.

---
## See Also
- [README](README.md) — Project overview and quick start
- [Architecture](architecture.md) — System design and components
- [Workflow](workflow.md) — Event flows and processing pipelines
- [Development](development.md) — Development guide and best practices
