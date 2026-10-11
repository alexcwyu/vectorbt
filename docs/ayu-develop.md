# VectorBT: workflow and architecture

Updated 2026-10-12. Source baseline: `0a6e1a134b5124600e4e367f8dd3a7233d15bab5` on `feature/ayu_develop`. See [development commands](development.md) and [validation evidence](validation.md) for the tested environment and current limitations.

## Purpose and features

Runs array-oriented strategy research by broadcasting prices and parameters into indicator, signal, portfolio-simulation, and record-analysis pipelines.

- IndicatorFactory and SignalFactory
- Order, signal, and custom callback simulations
- Column grouping and parameter broadcasting
- Sparse orders, trades, and drawdown records
- Numba execution and a local Rust companion

## Component architecture

ArrayWrapper preserves index, columns, and grouping metadata across NumPy computations. Portfolio constructors express different simulation contracts: orders, signals, or a callback-driven order function. Numba kernels handle numerical loops and return sparse event records. Records and MappedArray support analysis without expanding every event into a dense per-bar table. Broadcasting is a research convenience, but the simulation still needs explicit cash, fees, order sizing, and grouping semantics.

```mermaid
flowchart LR
    C0["Data and array metadata"]
    C1["Indicators and signals"]
    C2["Portfolio simulation"]
    C3["Sparse records and mapped arrays"]
    C4["Statistics and plots"]
    C0 --> C1 --> C2 --> C3 --> C4
```

The diagram shows responsibility boundaries in the main research or execution path. Concrete entry points and implementation files are linked below.

## Core types

The types below belong to the public or native boundary described by the linked implementation. Arrows describe the dependency or ownership relationship used in the documented workflow. Listed Python methods are present in the linked source; native array types represent the binding boundary.

```mermaid
classDiagram
    class ArrayWrapper {
        +indexing_func_meta()
        +indexing_func()
        +from_obj()
    }
    class IndicatorFactory {
        +from_custom_func()
        +from_apply_func()
        +get_talib_indicators()
    }
    class Portfolio {
        +engine()
        +indexing_func()
        +from_orders()
    }
    class Records {
        +field_config()
        +engine()
        +replace()
    }
    class MappedArray {
        +engine()
        +replace()
        +indexing_func_meta()
    }
    Portfolio *-- ArrayWrapper : indexed simulation metadata
    Records *-- ArrayWrapper : indexed event metadata
    Records --> MappedArray : maps event columns
    IndicatorFactory --> ArrayWrapper : creates wrapped outputs
```

## End-to-end workflow

1. Fetch or prepare indexed price arrays.
2. Broadcast indicator parameters and generate signals.
3. Choose portfolio constructor and execution backend.
4. Simulate orders and retain event records.
5. Analyze returns, trades, drawdowns, and plots.

```mermaid
flowchart TD
    W0["Fetch or prepare indexed price arrays"]
    W1["Broadcast indicator parameters and generate signals"]
    W2["Choose portfolio constructor and execution backend"]
    W3["Simulate orders and retain event records"]
    W4["Analyze returns, trades, drawdowns, and plots"]
    W0 --> W1 --> W2 --> W3 --> W4
    W4 --> R{"Review evidence"}
    R -->|Revise configuration| W0
    R -->|Retain result| A["Store reproducible artifacts"]
```

## Interaction sequence

The sequence follows one unit of work. Results return across the same boundaries that accepted the configuration and data. External I/O and numerical execution have different failure modes and should retain separate evidence.

```mermaid
sequenceDiagram
    actor User as Researcher or operator
    participant C0 as Data and array metadata
    participant C1 as Indicators and signals
    participant C2 as Portfolio simulation
    participant C3 as Sparse records and mapped arrays
    participant C4 as Statistics and plots
    User->>C0: Supply configuration and inputs
    C0->>C1: Broadcast indicator parameters and generate signals
    C1->>C2: Choose portfolio constructor and execution backend
    C2->>C3: Simulate orders and retain event records
    C3->>C4: Analyze returns, trades, drawdowns, and plots
    C4-->>C3: Results and retained state
    C3-->>C2: Execution or evaluation evidence
    C2-->>C1: Computed outputs
    C1-->>C0: Research or runtime result
    C0-->>User: Inspect outputs and provenance
```

## Data and artifact lifecycle

```mermaid
flowchart LR
    Inputs["Configuration and source data"] --> Runtime["Validated runtime inputs"]
    Runtime --> Work["Portfolio simulation"]
    Work --> Outputs["Statistics and plots"]
    Outputs --> Review["Metrics, logs, and retained artifacts"]
    Review --> Inputs
```

Inputs include explicit strategy or model settings, data selection, and execution assumptions. Retaining these with outputs makes a rerun reviewable. Temporary environments and caches remain development artifacts, while source-controlled configuration documents intended behavior.

## Execution and failure boundaries

Data provider integrations are optional external services. Their API versions can affect import and network tests. The rust/ project supplies the matching vectorbt-rust package through a uv path source. A Python wheel and a Rust extension build are separate artifacts and both are recorded in validation.

```mermaid
stateDiagram-v2
    [*] --> Configured
    Configured --> Prepared: Inputs and dependencies available
    Prepared --> Running: Start unit of work
    Running --> Completed: Retain result
    Running --> Failed: Record exception and inputs
    Failed --> Configured: Correct configuration or implementation
    Completed --> [*]
```

This state diagram describes the documented workflow, not a claim that these names are an implementation enum. Failure evidence should identify whether input preparation, external services, native compilation, or application execution failed.

## Development layout

Python code remains under src/vectorbt with Hatchling packaging. Tests import from src. The native companion remains in rust/ with its maturin backend and Cargo manifest.

## Source map

| Existing source | Responsibility evidence |
|---|---|
| [src/vectorbt/base/array_wrapper.py](../src/vectorbt/base/array_wrapper.py#L109) | Implementation and module/type documentation, beginning at line 109 |
| [src/vectorbt/base/reshape_fns.py](../src/vectorbt/base/reshape_fns.py#L40) | Implementation and module/type documentation, beginning at line 40 |
| [src/vectorbt/indicators/factory.py](../src/vectorbt/indicators/factory.py#L1206) | Implementation and module/type documentation, beginning at line 1206 |
| [src/vectorbt/indicators/basic.py](../src/vectorbt/indicators/basic.py#L93) | Implementation and module/type documentation, beginning at line 93 |
| [src/vectorbt/signals/factory.py](../src/vectorbt/signals/factory.py#L30) | Implementation and module/type documentation, beginning at line 30 |
| [src/vectorbt/signals/nb.py](../src/vectorbt/signals/nb.py#L42) | Implementation and module/type documentation, beginning at line 42 |
| [src/vectorbt/portfolio/base.py](../src/vectorbt/portfolio/base.py#L1500) | Implementation and module/type documentation, beginning at line 1500 |
| [src/vectorbt/portfolio/nb.py](../src/vectorbt/portfolio/nb.py#L66) | Implementation and module/type documentation, beginning at line 66 |
| [src/vectorbt/records/base.py](../src/vectorbt/records/base.py#L417) | Implementation and module/type documentation, beginning at line 417 |
| [src/vectorbt/records/mapped_array.py](../src/vectorbt/records/mapped_array.py#L379) | Implementation and module/type documentation, beginning at line 379 |
| [src/vectorbt/data/custom.py](../src/vectorbt/data/custom.py#L38) | Implementation and module/type documentation, beginning at line 38 |
| [src/vectorbt/_engine.py](../src/vectorbt/_engine.py#L19) | Implementation and module/type documentation, beginning at line 19 |
| [rust/Cargo.toml](../rust/Cargo.toml#L1) | Implementation and module/type documentation, beginning at line 1 |

The links target files present in this checkout. Follow their imports and adjacent tests when changing a boundary. The validation document distinguishes successful builds, complete test runs, inherited skips, and unresolved prerequisites.
