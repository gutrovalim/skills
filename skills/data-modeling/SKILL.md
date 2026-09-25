---
name: data-modeling
description: Reference for analytical table modeling and pipeline architecture — grain, keys, time semantics, history, the materialization choice between an Athena view, a Glue Catalog materialized view, an Iceberg table and a QuickSight SPICE dataset, partitioning, refresh topology, graph shapes where traversal is bounded by the platform, key-value stores where the key is the schema, and the constraints that make each of those one-way. Use when designing tables, datasets, graphs, key-value stores or pipeline layers, when deciding which layer a transformation belongs in, or when another skill needs the modeling vocabulary.
---

# Data modeling

Reference for the shape of an analytical model and the architecture around it. Consulted, not run — the decisions belong to the user, and this holds the vocabulary and the constraints that make them one-way.

Every platform fact here is cached from AWS documentation with its source beside it. **Quotas drift** — the SPICE row limit has been raised twice — so re-verify a number before a spec rests on it.

## Layers

Four layers. The only question that matters about a transformation is which one it belongs in.

- **raw** — a faithful copy of the source, never edited, partitioned by arrival.
- **staging** — typed, deduplicated, renamed. One per source, not one per consumer.
- **curated** — the business model: conformed grains, conformed dimensions, metrics with their definitions.
- **serving** — what a dashboard reads. Thin by design: it exists to be fast and to carry access rules.

A transformation that renames and deduplicates belongs in staging. One that changes grain belongs in curated. A grain change placed in serving gets re-derived by the next consumer, differently.

Nothing in the raw layer is as neutral as it looks: the source engine's timezone, collation and null-ordering defaults are already baked into the bytes. Those are in [rdbms-source-constraints.md](references/rdbms-source-constraints.md).

## The one-way doors

A door is one-way when reversing it costs more than a rewrite, because consumers have already bound to it. These are the ones that do not look like doors.

| Door | Why it is one-way | The tell it is being decided by accident |
|---|---|---|
| **Grain** | every measure and join downstream is defined against it | no comment in the DDL saying what one row is |
| **Key** | consumers join on it; changing it invalidates their joins | a surrogate key chosen by the tool, undocumented |
| **Timezone and day boundary** | periods have already been published and compared | no timezone anywhere in the spec |
| **Partition key** | it is the file layout; changing it rewrites history | partitioned by a column no query filters on |
| **Storage format** | Iceberg, Hive and Delta do not support the same features | format chosen by whichever engine wrote first |
| **Materialization** | a view and a materialized view are different contracts to the consumer | "make it a view for now", with no note of when that stops holding |
| **Refresh topology** | ordering between datasets is a dependency nobody draws | schedules set per dataset and never compared |
| **RLS mode** | it decides whether the dataset can use SPICE at all | security considered after the dataset is built |
| **Metric definition** | two dashboards computing "revenue" differently is a governance failure, not a bug | the metric exists only as a calculated field |

Naming is a door too, and the cheapest one to get right: a name becomes a column, a calculated field, a dataset and a dashboard title, and renaming it after someone builds on it is a migration.

## Materialization: which one

The constraints disqualify more often than the design chooses. Check the disqualifiers first, then read the two constraint files.

| Option | Choose when | Disqualified when |
|---|---|---|
| **Athena view** | the logic is shared and cheap, and freshness must be exact | it is scanned per dashboard and the scan is expensive |
| **Glue Catalog materialized view** | the definition fits the incremental-refresh subset and every base table is Iceberg | a base table is Hive or Delta; the definition needs an outer join, a subquery, `DISTINCT` or a window function; sources are cross-account or cross-Region |
| **Iceberg table written by a job** | the logic fits no MV subset, or needs full SQL | there is no job runner and none is being added |
| **QuickSight SPICE dataset** | the data fits the quota and a dashboard needs it fast and interactive | it is over quota; it must inherit RLS from a parent; or it must be fresher than the refresh floor |
| **QuickSight direct query** | the data is too large for SPICE and latency is acceptable | the dashboard is interactive over a slow source, or the scan cost is unbounded |

Two of these are traps. A materialized view is only chosen once its **exact definition** has been created and refreshed successfully — the documented refresh subset is narrow and internally inconsistent, so a complex definition is a spec risk, not an implementation detail. And a SPICE dataset whose parent has RLS **cannot be SPICE at all**: inherited rules force direct query.

In pipeline order, from source to serving:

- [rdbms-source-constraints.md](references/rdbms-source-constraints.md) — MySQL, PostgreSQL and SQL Server side by side: timezone and collation defaults, extraction isolation, upsert dialects, CDC retention, and what each does for partitioning, RLS and materialized views.
- [streaming-constraints.md](references/streaming-constraints.md) — delivery semantics, buffering as the freshness floor, retention as the replay window, and what a stream leaves in the table.
- [orchestration-constraints.md](references/orchestration-constraints.md) — catchup and backfill, data-aware triggers, and the cross-tool dependency no scheduler draws.
- [dbt-constraints.md](references/dbt-constraints.md) — incremental strategies, the look-back window, snapshots, and the tests that enforce grain.
- [athena-constraints.md](references/athena-constraints.md) — views, materialized views, Iceberg, partitioning, scan cost.
- [redshift-constraints.md](references/redshift-constraints.md) — unenforced constraints, snapshot isolation, best-effort autorefresh, and the vacuum debt.
- [quicksight-constraints.md](references/quicksight-constraints.md) — SPICE quotas, refresh, dataset chaining, RLS and CLS.
- [lakeformation-constraints.md](references/lakeformation-constraints.md) — row filters, cell-level security, and the hybrid mode that silently disables them.
- [graph-constraints.md](references/graph-constraints.md) — traversal bounds, nested types, and what Athena cannot do with a graph.
- [kv-constraints.md](references/kv-constraints.md) — key design and throttling, expiry, and the three paths from a store into Athena.

## When the shape is a graph

A knowledge graph is a different shape with the same doors. Two platform limits bound it, and neither is a design preference:

- **Traversal stops at 10 hops.** Recursive queries work in Athena engine v3 with a maximum recursion depth of 10, so anything deeper is materialized — a closure table, an adjacency array, or precomputed path counts.
- **The serving layer is relational again.** QuickSight has no graph visual, so graph insight reaches a dashboard only as precomputed flat tables: degree, community, centrality, co-occurrence.

There is also no vector search and no graph query language. So a graph engine keeps traversal, and Athena keeps the analytics *about* the graph — the graph sits upstream of the star schema, never in place of it. Details in [graph-constraints.md](references/graph-constraints.md); the levels to interview for are in [graph-frontier.md](../data-spec-grill/references/graph-frontier.md).

## When the shape is a key-value store

A key-value store is the inverse of an analytical model, and the two are complementary rather than alternatives: the store answers *point access by a known key*, the warehouse answers *scan and aggregate across all keys*. So the store is a serving tier and the warehouse is fed from it.

Two consequences follow, and both are design decisions rather than details:

- **The key is the schema.** Key design follows the enumerated access patterns, so it is decided *before* the model rather than derived from it, and changing it is a table rebuild plus a migration — the least reversible door in the data space.
- **The landing path is a choice between three paths that answer different questions.** Export to S3 for recurring analytics, a change stream for history and near-real-time, the federated connector for live ad-hoc queries. The incremental export in particular is a **state sync, not a change log** — it compacts each item to its final state in the window, so no history can be reconstructed from it.

An expiry attribute is also not a retention policy: expiry is best-effort and can lag by days, and an expired item still costs storage and still appears in reads until it is removed. Details in [kv-constraints.md](references/kv-constraints.md); the levels to interview for are in [kv-frontier.md](../data-spec-grill/references/kv-frontier.md).

## Refresh topology

Refresh ordering is the dependency nobody draws, and it fails in the direction that looks fine: a child refreshed before its parent keeps serving the parent's previous contents, and nothing errors.

So the spec states the order, not just the schedules. Where the platform does not sequence the refreshes for you, the pipeline must — and where a child is a QuickSight dataset over a SPICE parent, the platform does not sequence them. The mechanisms that do sequence an order — data-aware triggers, and the schedule offsets that stand in for them — are in [orchestration-constraints.md](references/orchestration-constraints.md).

State freshness as a number the business agreed to ("by 07:00 local, T+1"), never as "daily". A cadence is a mechanism; freshness is the promise.
