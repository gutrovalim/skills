# RDBMS source constraints

Platform facts that disqualify a design. Each carries its source; verify before a spec rests on a number.

MySQL, PostgreSQL and SQL Server, in one file because the same decision has a different answer on each. Two halves: the engine as a **source** — where timezone, collation and extraction consistency are decided before any analytical model exists — and the engine as a **platform** you may also model inside.

## Timezone: the column type is the decision

The day boundary is fixed by the column type at write time and is not recoverable afterwards.

| Engine and type | Carries a zone | Stored as | The trap |
|---|---|---|---|
| **PostgreSQL** `timestamptz` | yes | UTC | "The originally stated or assumed time zone is not retained and cannot be retrieved later" |
| **PostgreSQL** `timestamp` | **no** | as written | conversions to and from `timestamptz` assume the session `TimeZone` |
| **MySQL** `TIMESTAMP` | yes | seconds since epoch, UTC | converted session tz → UTC on write and back on read; range ends **2038-01-19 03:14:07 UTC** |
| **MySQL** `DATETIME` | **no** | as written | never stored in UTC and never converted; range runs to 9999 |
| **SQL Server** `datetimeoffset` | yes | instant, UTC-aware | offset range `-14:00` to `+14:00` |
| **SQL Server** `datetime2` | **no** | as written | "Time zone offset aware and preservation: No"; "Daylight saving aware: No" |

(https://www.postgresql.org/docs/current/datatype-datetime.html, https://dev.mysql.com/doc/refman/8.4/en/time-zone-support.html, https://dev.mysql.com/doc/refman/8.4/en/date-and-time-type-syntax.html, https://learn.microsoft.com/en-us/sql/t-sql/data-types/datetimeoffset-transact-sql, https://learn.microsoft.com/en-us/sql/t-sql/data-types/datetime2-transact-sql)

- **The session timezone is a hidden input.** MySQL's per-session `time_zone` and PostgreSQL's `TimeZone` decide what a zone-naive timestamp means. An extract running with a different session timezone from the application that wrote the row returns a different answer for the same stored value, with no error.
- **MySQL's 2038 ceiling is a one-way door.** A `TIMESTAMP` column cannot hold a date past `2038-01-19`, and a value outside the range is converted to `0` rather than rejected. Any future-dated column — an expiry, a contract end — has to be `DATETIME` before it is populated.
- Consequence: the spec names the column type **and** the session timezone for the connection that reads it. "The source stores UTC" is true of `timestamptz`, false of `timestamp`, and only conditionally true of MySQL's `TIMESTAMP`.

## Isolation: what a batch extract can see

- **MySQL/InnoDB defaults to `REPEATABLE READ`**, where consistent reads inside a transaction all use the snapshot from the first read. `WITH CONSISTENT SNAPSHOT` gives a snapshot **only** at `REPEATABLE READ` — "For all other isolation levels, the `WITH CONSISTENT SNAPSHOT` clause is ignored", with a warning.
- **PostgreSQL defaults to `read committed`** (`default_transaction_isolation`), so each statement takes a new snapshot.
- **SQL Server defaults to `READ COMMITTED`**, with locking.
- Consequence: **an extraction is consistent only if the isolation level says so.** MySQL's default gives it away free; PostgreSQL needs `REPEATABLE READ` for a multi-statement extract; and on any engine, an extract reading many tables in separate statements can straddle a commit. This is the mechanism behind a reconciliation that fails intermittently.

(https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-isolation-levels.html, https://dev.mysql.com/doc/refman/8.4/en/commit.html, https://www.postgresql.org/docs/current/runtime-config-client.html)

## Collation: case and accent are defaults, not choices

- **MySQL's default charset and collation are `utf8mb4` and `utf8mb4_0900_ai_ci`**, so nonbinary string comparisons are **case-insensitive by default** — `col LIKE 'a%'` matches `A` and `a`. The `_ai_ci` suffix means **accent-insensitive as well as case-insensitive**.
- **PostgreSQL compares strings case-sensitively** by default. SQL Server follows the column or database collation, commonly case-insensitive.
- Consequence: **a unique constraint on a name means different things on the two engines.** `ACME` and `acme` collide on MySQL and do not on PostgreSQL, so the same dedup design produces a different grain — and a spec that does not name the collation has left the grain of a name-keyed table to an engine default. This is the mechanism behind "the dedup didn't dedup".

(https://dev.mysql.com/doc/refman/8.4/en/case-sensitivity.html, https://dev.mysql.com/doc/refman/8.4/en/charset-unicode-sets.html)

## NULL ordering: the two engines disagree

- **MySQL** presents `NULL` first for `ASC` and last for `DESC`.
- **PostgreSQL** sorts nulls as if larger than any non-null value, so `NULLS LAST` is the default for `ASC` and `NULLS FIRST` for `DESC` — the opposite of MySQL.
- Both treat all `NULL`s as equal under `DISTINCT`, `GROUP BY` and `ORDER BY`; MySQL's aggregate functions ignore `NULL` except `COUNT(*)`.
- Consequence: a latest-row-per-key query written as `ORDER BY updated_at DESC LIMIT 1` returns a **different row** on each engine when `updated_at` is null for some rows. Any spec depending on the ordering of nulls states `NULLS FIRST`/`NULLS LAST`, or it is engine-specific by accident.

(https://dev.mysql.com/doc/refman/8.4/en/working-with-null.html, https://www.postgresql.org/docs/current/queries-order.html)

## Upsert: three dialects, three failure modes

| Engine | Statement | The failure |
|---|---|---|
| **MySQL** | `INSERT ... ON DUPLICATE KEY UPDATE` | with several unique indexes, "If `a=1 OR b=2` matches several rows, only one row is updated" — the rest stay stale. Affected-rows is 1, 2 or 0, so a counter reads 0 for an unchanged row |
| **PostgreSQL** | `INSERT ... ON CONFLICT DO UPDATE` | atomic upsert; `conflict_target` is **required** for `DO UPDATE`, so the arbiter index is explicit — a conflict on an index not named there raises instead of updating |
| **SQL Server** | `MERGE` | "can't update the same row more than once, or update and delete the same row", and two source rows matching one target row is an **error**. `WHEN NOT MATCHED BY SOURCE` deletes target rows absent from the source |

- Consequence: the upsert **is** the restatement policy at the source. A `MERGE` carrying `WHEN NOT MATCHED BY SOURCE` against a **windowed** extract deletes everything outside the window, and that is a spec decision rather than a detail of the job.

(https://dev.mysql.com/doc/refman/8.4/en/insert-on-duplicate.html, https://www.postgresql.org/docs/current/sql-insert.html, https://learn.microsoft.com/en-us/sql/t-sql/statements/merge-transact-sql)

## CDC and log retention: the replay window

- **MySQL** binlog retention: `binlog_expire_logs_seconds` defaults to **2592000 (30 days)**; `binlog_format` defaults to `ROW` and is deprecated, with row-based becoming the only format.
- **SQL Server** CDC retention defaults to **3 days (4320 minutes)**, maximum 52494800 minutes. Cleanup is a time-based low-water-mark job, and **a single cleanup job applies the same retention to every capture instance in the database**.
- **PostgreSQL** logical replication requires `wal_level = logical`. A slot retains WAL until `max_slot_wal_keep_size` (default `-1`, unlimited) is exceeded, after which `wal_status` goes `unreserved` and then **`lost`**, with `invalidation_reason` of `wal_removed` or `idle_timeout`.
- Consequence: **log retention is the maximum replay and backfill window, and it defaults to days.** A CDC pipeline more than 3 days behind on SQL Server has lost the changes — the capture instance's low endpoint has moved past them and nothing errors. A lost PostgreSQL slot is the same failure with a visible status field. As with a stream, raise it before you need it.

(https://dev.mysql.com/doc/refman/8.4/en/replication-options-binary-log.html, https://learn.microsoft.com/en-us/sql/relational-databases/track-changes/about-change-data-capture-sql-server, https://learn.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sys-sp-cdc-add-job-transact-sql, https://www.postgresql.org/docs/current/logical-replication-config.html, https://www.postgresql.org/docs/current/view-pg-replication-slots.html)

## Modelling inside the engine

### Constraints, and whether DDL can carry the grain

Primary, unique and foreign keys are **enforced** on MySQL/InnoDB, PostgreSQL and SQL Server, so on those three the grain can be declared in DDL and enforced by the engine. On Redshift it cannot — see `redshift-constraints.md`.

### Partitioning

- **PostgreSQL**: to create a unique or primary key on a partitioned table, "the constraint's columns must include all of the partition key columns", and the partition key must contain no expressions or function calls — because each partition's index can only enforce uniqueness within itself.
- Consequence: **partitioning can disqualify a primary key.** A table partitioned by `event_date` whose natural key is `order_id` cannot carry a PK on `order_id` alone, so the grain has to widen to `(order_id, event_date)` or move to a test. That is a design decision made by the partition key, and it belongs in the spec next to the grain.

(https://www.postgresql.org/docs/current/ddl-partitioning.html)

### Row-level security

| Engine | Mechanism |
|---|---|
| **PostgreSQL** | `ALTER TABLE ... ENABLE ROW LEVEL SECURITY` plus `CREATE POLICY`. **Enabling it with no policy is default-deny** — no rows visible or modifiable. The table owner is typically not subject to policies. Policies are **not** applied to internal referential-integrity checks, so existence can be inferred from a duplicate-key failure. `TRUNCATE` and `REFERENCES` are not subject to RLS. Policy expressions run with the **invoker's** rights, so a referenced table the user cannot read yields "permission denied" |
| **SQL Server** | `CREATE SECURITY POLICY` binding an inline table-valued function. `FILTER` predicates silently filter reads; `BLOCK` predicates explicitly block writes. Available since SQL Server 2016 |
| **MySQL** | **no row-level security.** Access is by privilege at global, database, table and column scope, plus views — which execute in **definer** context by default (`SQL SECURITY DEFINER`) |

- Consequence: on MySQL, a per-row rule is a **view per audience**, and a definer-context view runs with the definer's privileges, so the only thing restricting rows is the filter inside the view. Granting the base table separately silently bypasses it.
- Consequence: on PostgreSQL, enabling RLS before writing the policy is a **silent outage**, not an over-share.

(https://www.postgresql.org/docs/current/ddl-rowsecurity.html, https://www.postgresql.org/docs/current/sql-createpolicy.html, https://learn.microsoft.com/en-us/sql/relational-databases/security/row-level-security, https://learn.microsoft.com/en-us/sql/t-sql/statements/create-security-policy-transact-sql, https://dev.mysql.com/doc/refman/8.4/en/privileges-provided.html, https://dev.mysql.com/doc/refman/8.4/en/stored-objects-security.html)

### Materialized views

- **MySQL has none** — a "materialized" result is a table you populate.
- **PostgreSQL** `REFRESH MATERIALIZED VIEW` replaces the contents wholesale. `CONCURRENTLY` is allowed **only if there is at least one `UNIQUE` index on the materialized view which uses only column names and includes all rows** — not an expression index, no `WHERE` clause — and only once the view is already populated. Without `CONCURRENTLY`, a refresh that affects many rows **blocks concurrent reads**.
- **SQL Server** indexed views: the first index must be a **unique clustered** one, the view must be created `WITH SCHEMABINDING`, the definition must be deterministic, the base table must have the **same owner** as the view, and required `SET` options must be in force. DML on a base table degrades as the number of indexed views over it grows.
- Consequence: **"make it a materialized view" is not portable.** On PostgreSQL the `CONCURRENTLY` requirement forces a unique key onto the view, which is a grain decision; on SQL Server the schema-binding and owner rules constrain the definition before anyone considers its contents.

(https://www.postgresql.org/docs/current/sql-refreshmaterializedview.html, https://learn.microsoft.com/en-us/sql/relational-databases/views/create-indexed-views)
