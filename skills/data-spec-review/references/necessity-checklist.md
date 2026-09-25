# Necessity checklist

What a dataset, table or pipeline spec must state to be buildable. Walk every row. A row resolves to `stated`, `n/a - <reason>`, or missing. An `n/a` without a reason is itself a finding: the reason is what makes the omission reviewable later.

## The three that invalidate

Report these first. A spec missing any of them cannot be built, only guessed at.

1. **Grain** — the sentence "one row per ___", plus the mechanism that enforces it. Without it every measure downstream is wrong in a way that still looks plausible.
2. **Freshness** — a number the business agreed to, with its timezone ("by 07:00 local, T+1"). "Daily" is a mechanism, not a promise.
3. **Reconciliation** — the number you compare against a source of truth to prove the dataset is right, and who owns the disagreement when they differ.

## Identity and time

| Must state | Fails as |
|---|---|
| grain, and what enforces uniqueness | retries and backfills append duplicates |
| the key consumers join on, and whether it is stable | a join silently drops rows when the key is reissued |
| event time and processing time, named separately | late data dropped or double-counted, silently |
| timezone, and where the day boundary falls | two regions disagree about yesterday |
| what happens to a corrected past period | restatements either duplicate or vanish |
| the history model — snapshot, delta, or SCD type | last-write-wins destroys history on the first question asked |

## Measures

| Must state | Fails as |
|---|---|
| additive, semi-additive or ratio, per measure | an average is summed across a dimension |
| the denominator, for every ratio | a rate moves because its base changed |
| the null policy — excluded, zero, or unknown | nulls silently read as zero |
| the metric's definition, in one place | two dashboards disagree and neither is wrong |

## Materialization and layout

| Must state | Fails as |
|---|---|
| view / materialized view / Iceberg table / SPICE / direct query, per serving object | a view re-scanned per dashboard, or an MV that cannot express the query |
| that the materialized-view definition was proven to create and refresh | discovered at build time, after the design is committed |
| partition key, and the query filter that uses it | full scans billed per query |
| storage format, and why | a feature the chosen format does not support |
| who owns compaction and snapshot retention | unbounded small files, or time travel that quietly stops working |

## Pipeline and refresh

| Must state | Fails as |
|---|---|
| which job or schedule writes each table | a table nobody maintains |
| the refresh order between dependent objects | a child serving its parent's previous contents, with no error |
| the look-back window, and what it misses | a correction outside the window is never picked up |
| the backfill procedure, and whether it is idempotent | a backfill that doubles the data |
| who owns a failed refresh | a stale dashboard nobody notices until a decision is made on it |

## Access and cost

| Must state | Fails as |
|---|---|
| who may see which rows, and in which mode | RLS rules that cannot be expressed over numeric columns |
| who may see which columns | a visual that renders "Not Authorized", or disappears |
| the scan budget per query, and the workgroup limit | cost discovered on the bill |
| dataset size against the SPICE quota, and account-level capacity | ingestions failing for unrelated datasets |
| the sensitivity of each field, and the retention period | PII kept past its lawful life |

## When the shape is a graph

Added to every row above, which still apply. Grain splits in two: what is a node, and what is an edge.

| Must state | Fails as |
|---|---|
| whether the predicate vocabulary is closed or open | one table per relation while the vocabulary is open — a rewrite after consumers bind |
| what is a node and what is an edge, as two sentences | an edge modelled as a fact row, losing its text, provenance and validity |
| how an entity merge is recorded, and how a past answer is reproduced | merged entities cannot be un-merged, and history is unreproducible |
| the deepest question asked, in hops, and whether it is materialized | a query that cannot exist, discovered at build time |
| one row per edge or two, per symmetric relation | every symmetric query pays a `UNION`, or the fat table doubles |
| which time axis each temporal question means | "as of T" silently answers a different question than intended |
| the highest-degree node, and what expanding it costs | a traversal that explodes rather than slows |
| where embeddings live, and that Athena cannot search them | embedding columns scanned at $5/TB and never used |

## When the shape is a key-value store

Added to every row above. The order matters here: the access-pattern rows are answered before the schema rows, not after.

| Must state | Fails as |
|---|---|
| every read and write the store serves, each with its key | a table rebuilt after launch, with a migration |
| the partition key, and the cardinality of real traffic across its values | a hot partition that throttles only under load |
| which access patterns a secondary index serves, and whether it is global or local | an index that cannot be added without a backfill, or one that blocks partition splitting |
| what actually deletes, as opposed to what is meant to expire | data retained well past its stated life |
| the landing path into the analytics tier, and its latency | an analytics tier that cannot see the store's data |
| whether history is needed, and where it lives if so | a change stream added after the fact, when the events are already gone |
| cost per request for the store, and per byte scanned for the warehouse | two cost models collapsed into one line |
| which point in time the analytics copy is reconciled against | a reconciliation that can never pass, because the store moved |

## Done when

Every row is `stated` or carries a reasoned `n/a`, and the three invalidating rows are stated explicitly rather than implied by the SQL.
