# Extraction map

Where each row of `data-spec-review`'s [necessity checklist](../../data-spec-review/references/necessity-checklist.md) is decided in a codebase, what `implied` and `absent` look like there, and how far recovery can go.

Rows mirror the checklist row for row and in its order. That alignment is the contract: an as-built spec using these rows lets the review walk its own checklist and find every row already resolved.

## Where to read

Six artifact types. The map names them per row; the platform specifics are examples, not requirements.

| Type | What it is |
|---|---|
| **schema** | DDL, migrations, table definitions in IaC, catalog exports, `schema.yml` |
| **query** | the SQL that defines or populates the object: model SQL, view and MV definitions, notebooks |
| **orchestration** | DAGs, schedules, triggers, workflow definitions, dbt schedule |
| **serving** | BI dataset definitions, calculated fields, refresh schedules, RLS and CLS policies, workgroup settings |
| **application** | the code that reads and writes a store |
| **policy** | IAM, Lake Formation grants, row filters, column masks, tags |

The constraint files in [data-modeling](../../data-modeling/SKILL.md) decide which mechanisms exist at all — `streaming-`, `orchestration-`, `dbt-` and `lakeformation-constraints.md` for the rows below. **A mechanism the platform disqualifies is not a mechanism:** a look-back window outside the retention period, a row filter over a column type PartiQL cannot carry, a dedup claim on an at-least-once feed. Those rows are `absent`, not `implied`.

## Recoverability

Recovery runs out at a predictable point, and knowing where saves a wasted search.

- **`visible`** — the code decides it, and the code says what it is.
- **`mechanism-only`** — the code shows the mechanism; the promise or the meaning is not in the code. Record what the mechanism is and mark the promise `unread`.
- **`human-only`** — the code cannot answer it. Mark it `unread` with the question that has to go to a person.

Recoverability maps onto the classes like this: a `visible` row resolves to `stated` where something a human can read says so and `implied` otherwise; a `mechanism-only` row resolves to `implied` for the mechanism and `unread` for the promise; a `human-only` row is `unread` outright.

The map carries **one row fewer than the checklist**. The checklist states grain twice — in the invalidating three, and again as the first Identity and time row ("grain, and what enforces uniqueness"). It is kept once, here, in the invalidating table. Every other checklist row appears exactly once.

## The three that invalidate

| Row | Where to look | `implied` | `absent` | Recoverability |
|---|---|---|---|---|
| **Grain** | schema: PK or unique constraint. query: `GROUP BY` / `DISTINCT`. tests: a `unique` test | a `GROUP BY` exists and no sentence states it | no PK, no unique test, no `GROUP BY` — grain is whatever the join fan-out produces | `visible` |
| **Freshness** | orchestration: schedule or cron. serving: dataset refresh schedule. alerts | a cron exists; no agreed number and no timezone | no schedule at all — manual runs | `mechanism-only` |
| **Reconciliation** | tests, data-quality checks, assertions, monitoring, alerts | a row-count test exists; nothing compared to a source of truth | no tests, no alert — the usual answer, and a major finding | `mechanism-only` |

## Identity and time

| Row | Where to look | `implied` | `absent` | Recoverability |
|---|---|---|---|---|
| **Join key, and its stability** | schema: PK, FKs. query: join predicates. query: surrogate key generation | a hash surrogate key; stability unstated | consumers join on a mutable attribute — an email, a name | `visible` |
| **Event vs processing time** | schema: timestamp columns. query: which one the predicate filters. ingest columns (`ingested_at`, `load_date`) | both columns exist and the code uses one — which one is an unrecorded decision | one timestamp only, and the predicate uses it | `visible` |
| **Timezone, day boundary** | query: `AT TIME ZONE`, `date_trunc`, session config. serving: dataset timezone | `date_trunc('day', x)` with no timezone, so the boundary is the engine default | no timezone anywhere in the pipeline | `visible` |
| **Corrected past period** | query: write mode — `INSERT OVERWRITE` / `MERGE` / append. dbt incremental strategy + `unique_key` | merge with a `unique_key`; the window unstated | append-only, so a correction duplicates | `visible` |
| **History model** | schema: SCD columns, snapshot tables, `valid_from` / `valid_to`. table format snapshot and CDC settings. streams | SCD columns exist; the type is unstated | last-write-wins merge | `visible` |

## Measures

| Row | Where to look | `implied` | `absent` | Recoverability |
|---|---|---|---|---|
| **Additivity** | query: aggregates. serving: calculated fields | `SUM` or `AVG` with no additivity note | an average summed across a dimension | `mechanism-only` |
| **Denominator** | query: ratio expressions. serving: calculated fields | a ratio exists; its base is unstated | two objects computing the same ratio over different bases | `mechanism-only` |
| **Null policy** | query: `COALESCE` / `NULLIF`. the join type feeding an aggregate | a `COALESCE(x, 0)` is present | `SUM` over an outer join with no null handling, so nulls read as zero | `visible` |
| **Metric definition, in one place** | serving: calculated fields. a metrics layer, if one exists | the metric exists only as a calculated field, duplicated per dashboard | two dashboards computing "revenue" differently | `human-only` |

## Materialization and layout

| Row | Where to look | `implied` | `absent` | Recoverability |
|---|---|---|---|---|
| **Materialization per serving object** | query: `CREATE VIEW` / `CREATE MATERIALIZED VIEW`. dbt `materialized:`. serving: cached dataset vs direct query. IaC | a view, with no note of when that stops holding | — structurally always present | `visible` |
| **MV proven to create and refresh** | orchestration: refresh logs and history. IaC | an MV exists; whether its definition is inside the refresh subset is a build-time question | — | `mechanism-only` |
| **Partition key, and the filter that uses it** | schema: `PARTITIONED BY`, partition spec, path layout. dbt partition config | partitioned by a column; whether consumers filter on it needs the serving layer too | no partition on a large object | `visible` |
| **Storage format, and why** | schema: table properties, `write.format.default`, catalog parameters, metadata files | the format is visible; the reason is not recorded | mixed formats across one pipeline | `visible` |
| **Compaction and snapshot retention** | schema: table properties. maintenance jobs | a retention setting exists; who runs compaction is unstated | unbounded small files, or time travel quietly disabled | `mechanism-only` |

## Pipeline and refresh

| Row | Where to look | `implied` | `absent` | Recoverability |
|---|---|---|---|---|
| **Which job writes each object** | orchestration: the job or schedule per target | a DAG writes it; ownership unstated | an object nobody maintains | `visible` |
| **Refresh order between dependents** | orchestration: DAG edges, `ref()` graph, workflow triggers, cron offsets | a DAG encodes it; nothing encodes it between a cached dataset and its parent | per-dataset schedules never compared | `visible` |
| **Look-back window, and what it misses** | query: the incremental predicate, `is_incremental()` blocks, export windows | a three-day window; what it misses is unstated | a window covering only "yesterday" | `visible` |
| **Backfill, and idempotency** | query: merge key and full-refresh handling. runbooks in the repo | idempotent by construction; the procedure is unstated | append-only with no backfill path | `mechanism-only` |
| **Owner of a failed refresh** | orchestration: alerting, `owner` fields. `CODEOWNERS`, on-call config | a DAG `owner` string | nothing at all | `human-only` |

## Access and cost

| Row | Where to look | `implied` | `absent` | Recoverability |
|---|---|---|---|---|
| **Who sees which rows, in which mode** | policy: row filters, dataset RLS rules, IAM, RLS policies | rules exist; the mode decides whether caching is possible at all | no access control on an object with PII | `visible` |
| **Who sees which columns** | policy: column masks, column-level grants, tags | a grant list | nothing, on an object with restricted columns | `visible` |
| **Scan budget per query** | serving and infra: bytes-scanned cutoff, workgroup limits | a cutoff exists | no budget, so the worst query is unbounded | `visible` |
| **Size against the cache quota** | serving: dataset size and refresh history against the platform quota | a size near a quota | ingestions failing for unrelated datasets | `mechanism-only` |
| **Sensitivity, and retention** | policy: tags, classifications, retention config, snapshot retention | a tag exists | PII with no tag | `human-only` for the lawful life; `visible` for the tag |

## When the shape is a graph

Added to every row above. Grain splits in two: what is a node, and what is an edge.

| Row | Where to look | `implied` | `absent` | Recoverability |
|---|---|---|---|---|
| **Closed or open predicate vocabulary** | schema: a constraint or enum on the relation column. extraction output schema | an enum in code, with no note of what happens on a new value | a free-text predicate column | `visible` |
| **Node vs edge, as two sentences** | schema: two table families, or one table with a type discriminator | a discriminator column; nothing states which values are nodes | edges modelled as fact rows, losing text, provenance and validity | `visible` |
| **Entity merge, and reproducing a past answer** | query: entity resolution code. schema: `merged_into` columns, canonical ID maps | merge columns exist; un-merge is not addressed | merges happen in a job with no record | `mechanism-only` |
| **Deepest question, in hops** | query: recursive CTEs, depth columns, closure tables | a recursion bound exists in code | nothing bounds the traversal | `human-only` for the question; `visible` for the bound |
| **One row per edge or two** | schema: the edge table's unique constraint | a unique constraint on the ordered pair | no constraint, and no note of which direction is stored | `visible` |
| **Which time axis a temporal question means** | schema: `valid_from` / `observed_at` / `ingested_at` on edges | several time columns, no note of which answers "as of" | one timestamp, used for both | `mechanism-only` |
| **Highest-degree node, and its cost** | query: a precomputed degree or centrality table, if one exists | a degree table exists, refresh cadence unstated | nothing precomputed | `mechanism-only` |
| **Where embeddings live, and whether the engine can search them** | schema: vector columns. query: any similarity search | vector columns exist; the search path is unstated | embeddings stored where no query can reach them | `visible` |

## When the shape is a key-value store

Added to every row above. The order matters here: the access patterns are recovered before the key, not after.

| Row | Where to look | `implied` | `absent` | Recoverability |
|---|---|---|---|---|
| **Every read and write, each with its key** | application: every call site — get, query, put, update — and the attributes each one passes | call sites exist; the pattern list was never written down | no enumeration exists anywhere | `visible` |
| **Partition key, and traffic cardinality** | schema: the key attributes. metrics: per-key traffic distribution | the key attribute is visible; its distribution is not | a key chosen before any pattern was listed | `mechanism-only` |
| **Secondary indexes, global or local** | schema: index definitions and their key schemas. application: which calls use them | indexes exist; which pattern each serves is unstated | a pattern served by a scan | `visible` |
| **What actually deletes** | schema: TTL attribute and table TTL config. lifecycle policies | a TTL attribute exists | an expiry attribute the platform does not honour, so nothing deletes | `visible` |
| **Landing path, and its latency** | orchestration and infra: export jobs, streams, federated connectors | a path exists; its latency and its history semantics are unstated | no path into the analytics tier | `visible` |
| **Whether history is needed, and where it lives** | infra: stream configuration, PITR setting, export windows | a stream exists; what it retains is unstated | state-sync only, so no history can be reconstructed | `mechanism-only` |
| **Cost per request, and per byte scanned** | schema: capacity mode, provisioned throughput. serving: scan budget | a capacity mode is visible | no cost model at all | `mechanism-only` |
| **The point in time reconciled against** | orchestration: export and stream lag. the analytics copy's watermark | a watermark column exists | nothing reconciles the copy to the store | `mechanism-only` |
