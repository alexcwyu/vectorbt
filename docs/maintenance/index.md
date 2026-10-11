# vectorbt: portable verification records

Full raw logs are retained in the workspace maintenance evidence, not distributed here. Hashes identify those original logs. A successful build or selected test subset does not imply a full test-suite pass. Commands containing <workspace> describe the original execution environment.

[Machine-readable records](verification.json) · [Repository validation summary](../validation.md)

## Executed commands

| Check | Exit | Duration | Command | Outcomes |
| --- | ---: | ---: | --- | --- |
| build | 0 | unrecordeds | `uv build` | Successfully built dist/vectorbt-1.1.2.tar.gz<br>Successfully built dist/vectorbt-1.1.2-py3-none-any.whl |
| build-final | 0 | unrecordeds | `uv build` | Successfully built dist/vectorbt-1.1.2.tar.gz<br>Successfully built dist/vectorbt-1.1.2-py3-none-any.whl |
| cargo-tests | 0 | unrecordeds | `cargo test --manifest-path rust/Cargo.toml` | test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s |
| data-warning-contracts | 0 | unrecordeds | `uv run --no-sync pytest tests/test_data.py::TestData::test_download -x` | 1 passed in 0.38s |
| data-warning-current | 1 | unrecordeds | `uv run --no-sync pytest tests/test_data.py::TestData::test_download -x` | See exit status; no test-count assertion. |
| engine-parity-current | 0 | unrecordeds | `uv run --no-sync pytest tests/test_engine.py::TestIndicatorRustParity -x` | 2 passed in 51.10s |
| python314-sync | 0 | unrecordeds | `uv sync --all-groups --all-extras --python 3.14.8` | See exit status; no test-count assertion. |
| python315-lock | 0 | unrecordeds | `uv lock --upgrade --python 3.15` | See exit status; no test-count assertion. |
| python315-sync | 2 | unrecordeds | `uv sync --python 3.15 --all-groups --all-extras` | See exit status; no test-count assertion. |
| tests | 4 | unrecordeds | `uv run --no-sync pytest` | See exit status; no test-count assertion. |
| tests-all-final | 0 | unrecordeds | `uv run --no-sync pytest` | 1113 passed in 227.09s (0:03:47) |
| tests-asserted-warnings | 1 | unrecordeds | `uv run --no-sync pytest` | 10 failed, 1103 passed in 728.26s (0:12:08) |
| tests-current-contracts-final | 1 | unrecordeds | `uv run --no-sync pytest` | 2 failed, 1111 passed in 259.96s (0:04:19) |
| tests-current-warning-contracts | 2 | unrecordeds | `uv run --no-sync pytest` | See exit status; no test-count assertion. |
| tests-explicit-time-units | 1 | unrecordeds | `uv run --no-sync pytest` | 37 failed, 1076 passed in 687.44s (0:11:27) |
| tests-lazy-provider-import | 2 | unrecordeds | `uv run --no-sync pytest` | See exit status; no test-count assertion. |
| tests-warning-assertions | 1 | unrecordeds | `uv run --no-sync pytest` | 10 failed, 1103 passed in 718.02s (0:11:58) |
