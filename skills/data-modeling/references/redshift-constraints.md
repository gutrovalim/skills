# Redshift constraints

Platform facts that disqualify a design. Each carries its source; verify before a spec rests on a number.

Redshift is the warehouse where **the constraints you declare are not enforced**, which moves where grain lives. Everything else here follows from that.

## Nothing is enforced, so grain lives in the pipeline

- "Uniqueness, primary key, and foreign key constraints are informational only; they are not enforced by Amazon Redshift when you populate a table." An insert that violates a constraint **succeeds**. (https://docs.aws.amazon.com/redshift/latest/dg/t_Defining_constraints.html)
- They remain **planner hints**, and AWS names the consequence: "If your application allows invalid foreign keys or primary keys, some queries could return incorrect results. For example, a `SELECT DISTINCT` query might return duplicate rows if the primary key is not unique." (same source)
- The guidance cuts both ways: declare keys when you know they are valid, and "do not define key constraints for your tables if you doubt their validity" (same source; https://docs.aws.amazon.com/redshift/latest/dg/c_best-practices-defining-constraints.html).
- Consequence: **a primary key in Redshift DDL is a claim, not a mechanism.** It is worse than declaring nothing, because the planner will *trust* it — so a non-unique "primary key" produces duplicate rows out of `SELECT DISTINCT`, which is a wrong number that looks like a query bug. The mechanism that enforces grain is a `MERGE` or a test, and the spec names which.

## Isolation: snapshot by default, and it is not the strict one

- **SNAPSHOT isolation is the default** on provisioned clusters and serverless workgroups. SERIALIZABLE is available, takes longer, and aborts concurrent transactions with a serialization violation rather than permitting write skew.
- The snapshot is taken at the first `SELECT`, DML (`COPY`, `DELETE`, `INSERT`, `UPDATE`, `TRUNCATE`) or certain DDL in the transaction, and "no other transaction is able to change this snapshot".
- Consequence: a refresh reads a consistent point in time, which is what makes it reproducible — but two refreshes of the same object at different times can legitimately disagree, so a reconciliation compares a **timestamp** as well as a number.

(https://docs.aws.amazon.com/redshift/latest/dg/c_serial_isolation.html)

## Materialized views: autorefresh is best-effort and the subset is narrow

- `AUTO REFRESH` **defaults to `NO`.** Without it, "the data in the materialized view remains unchanged, even when applications make changes to the data in the underlying tables."
- Incremental refresh is unsupported for a definition using **`OUTER JOIN`**, the set operations **`UNION`, `INTERSECT`, `EXCEPT`, `MINUS`**, **`DISTINCT` aggregates**, **window functions**, **subqueries**, or the aggregates `MEDIAN`, `PERCENTILE_CONT`, `LISTAGG`, `STDDEV_SAMP`, `STDDEV_POP`, `APPROXIMATE COUNT`, `APPROXIMATE PERCENTILE` and bitwise aggregates. `COUNT`, `SUM` and `AVG` **are** supported.
- Where incremental refresh is unavailable the view silently falls back to a **full** refresh and is still created; the notice "may or may not be displayed, depending on the SQL client application". `STV_MV_INFO` reports the refresh type actually in use.
- Autorefresh is **deprioritised against the user workload**: "Amazon Redshift prioritizes your workloads over autorefresh and could stop autorefreshing to preserve the performance of the user workload", which "can delay the refreshes of some materialized views". Status is in `SVL_MV_REFRESH_STATUS`.
- A **full recompute** is triggered by operations such as `VACUUM` or `TRUNCATE` on base tables, and needs `CREATE` on the schema as well as `SELECT` on the bases. Incremental refresh sees only **already-committed** base rows, so a refresh in the same transaction as a DML statement does not see that statement's changes.
- **Behaviour change, 27 February 2026:** Auto REFRESH queries execute **as user queries rather than background autonomic processes**, so they run at the same priority as other user queries, improving freshness. Enabled on provisioned clusters on the CURRENT track from patch P198, and **disabled on Serverless**.

(https://docs.aws.amazon.com/redshift/latest/dg/materialized-view-refresh.html, https://docs.aws.amazon.com/redshift/latest/dg/materialized-view-create-sql-command.html, https://docs.aws.amazon.com/redshift/latest/dg/materialized-view-refresh-sql-command.html, https://docs.aws.amazon.com/prescriptive-guidance/latest/materialized-views-redshift/refreshing-materialized-views.html, https://docs.aws.amazon.com/redshift/latest/mgmt/behavior-changes.html)

- Consequence: **a materialized view's freshness is a range, not a number.** With autorefresh off it is unbounded; with it on, it is "as soon as possible unless the cluster is busy". A spec promising freshness from a materialized view names the refresh trigger and the view it is monitored through, or it promises nothing. The incremental subset is checkable **before** the design commits: a definition carrying an outer join or a window function is a full refresh forever.

## Layout: sort keys, distribution, and the vacuum debt

- Sort keys are `COMPOUND` by default, or `INTERLEAVED` with a **maximum of eight columns**. `SORTKEY AUTO` delegates the choice to automatic table optimization.
- `DELETE` marks rows for deletion and the space is reclaimed by `VACUUM`. `VACUUM RECLUSTER` "doesn't merge the newly sorted data with the sorted region. It also doesn't reclaim all space that is marked for deletion", and when it completes the table "might not appear fully sorted". It is unsupported on tables with interleaved sort keys or `ALL` distribution.
- For interleaved sort keys, performance degrades as the distribution of key values shifts; an `interleaved_skew` above **1.4** usually means `VACUUM REINDEX` will help.
- Consequence: **deletes are deferred**, so a delete-and-reload leaves the old rows occupying storage until a vacuum runs. A spec that states a retention policy but names no vacuum owner has a retention policy that reclaims nothing, and a scan cost that grows on data no query can see.

(https://docs.aws.amazon.com/redshift/latest/dg/t_Sorting_data.html, https://docs.aws.amazon.com/redshift/latest/dg/r_VACUUM_command.html, https://docs.aws.amazon.com/redshift/latest/dg/r_vacuum-decide-whether-to-reindex.html)

## What this moves in the spec

Two rows land differently from every other platform here. **Grain** has no DDL mechanism, so it is a pipeline and test responsibility. **Compaction and snapshot retention** becomes **vacuum and reindex ownership**. Both are quiet failures: the first surfaces as a duplicate row in a `DISTINCT`, the second as storage and scan cost that no change in data volume explains.
