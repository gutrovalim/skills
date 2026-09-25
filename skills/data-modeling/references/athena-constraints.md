# Athena constraints

Platform facts that disqualify a design. Each carries its source; verify before a spec rests on a number.

## Scan cost

- **$5 per TB scanned** for SQL over S3, rounded up to the nearest MB with a **10 MB minimum per query**. (https://aws.amazon.com/athena/pricing/)
- DDL — `CREATE`/`ALTER`/`DROP TABLE`, partition management — is **not charged**. Failed queries are not charged. **Cancelled queries are charged** for what they scanned.
- A workgroup carries one **per-query data limit**, which cancels any query exceeding it, and multiple **per-workgroup limits**, which notify an SNS topic or disable the workgroup. (https://docs.aws.amazon.com/athena/latest/ug/workgroups-setting-control-limits-cloudwatch.html)
- Consequence for a spec: a partition key no query filters on is not a partition key, it is storage. Columnar format plus partition filtering *is* the cost model, and a view that hides the filter from the optimizer breaks it.

## Views

An Athena view is logical: **the defining query runs every time the view is referenced.** It stores nothing and accelerates nothing. (https://docs.aws.amazon.com/athena/latest/ug/views.html)

- **Nested views are allowed** (a view over a view); recursive views are blocked. A stale view — a referenced table dropped, recreated with a different schema, or missing — errors at query time.
- View names take no special characters other than underscore.
- **No geospatial functions** in a view.
- Views do not work with external Hive metastores or UDFs.
- **A view does not manage access to the underlying S3 data.** Permission to query the view is not permission to read the data. (https://docs.aws.amazon.com/athena/latest/ug/considerations-limitations-views.html)
- Hidden metadata columns `$bucket`, `$file_modified_time`, `$file_size` and `$partition` are **not supported in views**; `$path` is.
- **Partition projection configured on a view is ignored.** Projection is a table property, so a view over a projected table needs projection configured on the underlying table. This is the most common "the view returns nothing" bug.
- Cross-account *querying* works in engine v3, but a view cannot be created over a cross-account Data Catalog.
- **Data Catalog views** — the Glue kind, for cross-service use with definer semantics — are stricter: they **cannot reference other views**, database or table resource links; `UNPROTECTED` is unsupported; federated sources are unsupported. The two view kinds are not interchangeable, so choose deliberately.

## Partitioning

Hive-style partitions only for `MSCK REPAIR TABLE` (`key=value` paths).

- **`MSCK REPAIR TABLE` does not remove stale partitions.** Deleting a partition in S3 and re-running it leaves the metadata behind; drop those with `ALTER TABLE DROP PARTITION`. (https://docs.aws.amazon.com/athena/latest/ug/msck-repair-table.html)
- It **fails above roughly 100,000 partitions** on a table (memory), and can time out mid-run leaving a partial state. Use `ALTER TABLE ADD PARTITION` instead.
- S3 paths must be **lower case** or partitions are not added. It scans a folder *and its subfolders*, so two tables under one prefix absorb each other's partitions — keep separate hierarchies.
- Partition locations must use the `s3://` protocol; `s3a://` fails.

**Partition projection** removes the repair step, at a cost in truthfulness:

- Athena **ignores catalog partition metadata** for a projected table.
- A projected partition that does not exist in S3 is still projected: **the query succeeds and returns zero rows, with no error.** A missing data drop is indistinguishable from a legitimately empty period.
- Queries outside the projected range also return zero rows without error.
- If **more than half the projected partitions are empty**, performance is worse than traditional partitions.
- Projection is **Athena-only**: Redshift Spectrum, EMR and Athena for Spark fall back to catalog metadata, which projection never populated.
- `SHOW PARTITIONS` does not list projected partitions.
- Time-format projections need `yyyy-MM-dd HH:00:00`; a `range` separates values with commas, not hyphens.

(https://docs.aws.amazon.com/athena/latest/ug/partition-projection.html, https://docs.aws.amazon.com/athena/latest/ug/troubleshooting-athena.html)

## Materialized views

A Glue Data Catalog materialized view is a **managed Iceberg table holding precomputed results**. Creation and refresh are Spark-only — **Spark 3.5.6+ in Athena Spark, EMR 7.12.0+, or Glue 5.1**.

- **Athena can read a materialized view but cannot manage one.** Supported: `SELECT`, `DESCRIBE`, `SHOW TABLES`, joins, filtering, aggregation. **Not supported in Athena**: `CREATE MATERIALIZED VIEW`, `REFRESH MATERIALIZED VIEW`, `ALTER`, `DROP`, `INSERT`, `UPDATE`, `MERGE`, `DELETE`, `OPTIMIZE`, `VACUUM`. (https://docs.aws.amazon.com/athena/latest/ug/querying-iceberg-gdc-mv.html)
- **Base tables must be Iceberg tables registered in the Glue Data Catalog.** Hive, Hudi and Delta sources are unsupported.
- A materialized view **cannot reference a Glue Catalog view, a multi-dialect view, or another materialized view.**
- All source tables must be **governed by Lake Formation**; IAM-only and hybrid access are unsupported. The definer role needs `SELECT`/`ALL` on sources **without row, column or cell filters**, plus `CREATE_TABLE` on the target database. Lose access and **refreshes fail** — the view does not degrade, it stops.
- **Cross-Region and cross-account source tables are unsupported.**
- **The minimum automatic refresh interval is one hour.** Manual `REFRESH MATERIALIZED VIEW` is the only way to go faster.
- A materialized view is **eventually consistent** with its base tables: during the refresh window a query returns stale data with no error.
- **A full refresh makes previous snapshots unavailable.**
- Columns prefixed `__ivm` are reserved for system use.
- Automatic query rewrite only considers views whose definitions fall inside the restricted subset below. A stale or non-matching view is **silently bypassed** and the base tables are scanned instead — so the cost saving is not guaranteed by creating the view.

**The incremental-refresh subset** (https://docs.aws.amazon.com/emr/latest/ReleaseGuide/emr-spark-materialized-views.html):

- The definition must be a single `SELECT`-`FROM`-`WHERE`-`GROUP BY`-`HAVING` block.
- No set operations, no subqueries, no `DISTINCT` in `SELECT`, no window functions, no joins other than `INNER JOIN`.
- The same bullet list also says "no aggregate functions", while the same page's examples use `COUNT(*)` and `SUM()` under `GROUP BY`. **The documentation contradicts itself**, so treat the subset as unproven for any definition that aggregates, and prove it by creating and refreshing the view *before* the spec depends on it.
- No UDFs, and only a subset of built-in functions.
- `SORT BY`, `LIMIT`, `OFFSET`, `CLUSTER BY` and `ORDER BY` are unsupported.
- Non-deterministic functions (`rand()`, `current_timestamp()`) are unsupported.
- Identifiers with characters other than alphanumerics and underscores are unsupported.

## Iceberg

- Athena supports Iceberg v1.4.2 and **creates and operates on v2 tables only**; file formats Parquet, ORC, Avro. (https://docs.aws.amazon.com/athena/latest/ug/querying-iceberg.html)
- Glue catalog only, created through the open-source Glue catalog implementation.
- `tinyint`, `smallint`, `char` and `fixed(L)` are unsupported for Iceberg tables in Athena; `time` is DDL-unsupported but queryable.
- Iceberg is what makes `UPDATE`/`DELETE`/`MERGE` available in Athena at all. On Hive tables they are not.
