# dbt constraints

Platform facts that disqualify a design. Each carries its source; verify before a spec rests on a number.

Where the transformation layer decides grain, key and history — often without the spec recording it.

## The incremental strategy is the restatement policy

- The strategies are `append`, `merge`, `delete+insert`, `insert_overwrite` and `microbatch`, and **the adapter decides which exist**: dbt-postgres has no `insert_overwrite`, dbt-bigquery has no `delete+insert`. (https://docs.getdbt.com/docs/build/incremental-strategy)
- `merge` with a `unique_key` **overwrites matched rows entirely by default**; `merge_update_columns` narrows it to the named columns (same source).
- `insert_overwrite` **does not use `unique_key`** — it replaces whole partitions, and requires the model to carry a partition config. (same source; https://docs.getdbt.com/reference/resource-configs/bigquery-configs)
- Consequence: the strategy *is* the answer to "what happens to a corrected past period", so it belongs in the spec. `append` on a self-correcting source duplicates; `merge` on a correction outside the window misses it with no error.

## The look-back window is a number, or it is one day

- The late-arriving pattern is a look-back on the incremental filter — `where date_day >= (select max(date_day) - 3 days from {{ this }})` — guarded by `{% if is_incremental() %}`. (https://docs.getdbt.com/docs/build/incremental-microbatch)
- `microbatch` replaces that hand-written filter with declared config: `event_time`, `batch_size`, `lookback`, `begin`, `full_refresh=false`. The mechanism **varies by adapter** — dbt-postgres microbatch uses `merge` and therefore **requires** a `unique_key`. (same source)
- Consequence: the window's **width is a spec number** and what it misses is a spec sentence. A correction older than the window is not late, it is lost.

## A snapshot is not a history model by default

- `strategy` is **required and has no default**: `timestamp` (needs a reliable `updated_at`) or `check` (needs `check_cols`). (https://docs.getdbt.com/reference/snapshot-configs)
- **`hard_deletes` defaults to `ignore`**, under which "Deleted rows will not be tracked and their `dbt_valid_to` column remains `NULL`". The alternatives are `invalidate` and `new_record`. (https://docs.getdbt.com/reference/resource-configs/hard-deletes)
- `dbt_valid_to_current` puts a sentinel in `dbt_valid_to` instead of `NULL`, which changes every downstream `is_current` test (same source).
- Consequence: **the default snapshot silently loses deletes.** A spec claiming SCD2 history without naming a `hard_deletes` mode has a model that cannot answer "when did this customer leave" — the row simply stops changing and reads as current forever.

## Tests are where reconciliation lives

- Four generic tests ship built in: `unique`, `not_null`, `accepted_values`, `relationships`. (https://docs.getdbt.com/faqs/Tests/available-tests)
- A data test is a `select` returning **failing** rows, so zero failing rows is a pass. (https://docs.getdbt.com/docs/build/data-tests)
- `store_failures` writes failing rows to a table named after the test, in `<schema>_dbt_test__audit`, replacing the previous failures even when there are none. `severity`, `error_if` and `warn_if` decide whether a failure stops the build. (https://docs.getdbt.com/reference/resource-configs/store_failures, https://docs.getdbt.com/reference/data-test-configs)
- Consequence: `unique` on the key **is the mechanism that enforces grain**, so a model whose grain is asserted in prose with no `unique` test has no grain mechanism at all — the row `data-spec-recover` resolves to `absent`. A `relationships` test is the nearest thing dbt has to a reconciliation row, because it names the source of truth it checks against.
- A test set to `severity: warn` records the disagreement and continues, which is a decision to serve a wrong number knowingly. That decision belongs in the spec.

## The graph stops at the project boundary

- dbt orders models by `ref()`, so ordering **within** a project is derived rather than declared.
- Consequence: that guarantee does not cross into a DAG, a snapshot, or a BI dataset schedule — see `orchestration-constraints.md`.
