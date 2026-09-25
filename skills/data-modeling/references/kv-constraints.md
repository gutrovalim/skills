# Key-value constraints

Platform facts that bound a key-value store and its landing in Athena. Written against DynamoDB as the AWS reference store; each carries its source. Verify before a spec rests on a number.

## Key design and throttling

- Items are placed by **hash of the partition key**, so a low-cardinality partition key funnels traffic into one physical partition. (https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/bp-partition-key-uniform-load.html)
- A single partition key value throttles beyond **3,000 RCU and 1,000 WCU**. (https://aws.amazon.com/blogs/database/choosing-the-right-dynamodb-partition-key/)
- Adaptive capacity ("split for heat") can split a busy partition, but **a local secondary index prevents splitting within an item collection** — so an LSI caps the recovery available to you later. Prefer a GSI, even when it carries the same partition key. (https://aws.amazon.com/blogs/database/part-2-scaling-dynamodb-how-partitions-hot-keys-and-split-for-heat-impact-performance/)
- Loading data already sorted by partition key creates a **rolling hot partition**, so a backfill can throttle production even when the steady-state pattern is fine.
- Composite partition keys (`customer#product#region`) are the standard mitigation for a naturally low-cardinality dimension.

## Expiry is not deletion

- TTL deletes on a **best-effort background process**, typically within a few days — not at the expiry timestamp. (https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/TTL.html)
- **Expired items still count toward storage and read cost**, and still appear in reads and scans until they are actually removed, unless a filter expression excludes them. (https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/ttl-expired-items.html)
- TTL deletions appear in the change stream **only in the region where the deletion happened** — not in regions the deletion replicated to.
- Enabling TTL takes about an hour to reach all partitions; after disabling, deletions continue for roughly 30 minutes. Renaming the TTL attribute requires disabling and re-enabling, and TTL must be reconfigured on restored tables.

So an expiry attribute is not a retention policy. When a spec promises a retention period, it names the mechanism that enforces it.

## Landing in Athena

Three paths, and they answer different questions.

**Export to S3** — the path for recurring analytics. Requires point-in-time recovery. Asynchronous, **consumes no read capacity**, no impact on table performance or availability. (https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/S3DataExport.HowItWorks.html)

- A **full export** snapshots the table at a point in time within the PITR window.
- An **incremental export** covers changes in a window that must be **at least 15 minutes and at most 24 hours**, start inclusive and end exclusive.
- An incremental export is **compacted to each item's final state for the window** — an item updated five times appears once. It is a state sync, **not a change log**, so history cannot be reconstructed from it. (https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/S3DataExport_Requesting.html)
- **Full and incremental exports have different structures**: incremental records are wrapped in `NewImage` / `OldImage`. A pipeline seeded from a full export and then fed incrementals has to reconcile two shapes — a design step, not a detail.
- Formats are DynamoDB JSON or Amazon Ion. Billed by table size (full) or data processed (incremental, 10 MB minimum).

**Change stream** — the path for near-real-time and for history. It is the only place a deletion is observable, including a TTL deletion, and the only path that preserves intermediate states.

**Federated connector** — the path for live, ad-hoc and join queries. Athena invokes a Lambda connector in your account, and predicate pushdown **does** apply to simple predicates (validation shows a pushed-down query scanning ~25 KB against ~25 MB for a full scan), but **complex and case-insensitive predicates are not pushed down**. (https://github.com/awslabs/aws-athena-query-federation/pull/1810)

- You pay Lambda invocation plus read capacity per query. AWS's own guidance warns that tables above a few gigabytes can incur high cost, and recommends always bounding a query with `LIMIT` and considering S3 instead for large tables. (https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/access-query-and-join-amazon-dynamodb-tables-using-athena.html)
- **It is not a point-lookup API.** Use it for exploration and joining; use exports for anything recurring.
- Connectors without predicate pushdown average around two minutes on small datasets and can time out on large ones. (https://docs.aws.amazon.com/athena/latest/ug/connectors-available.html)

## Two cost models

A store bills per request and per provisioned capacity; Athena bills per byte scanned. A spec spanning both carries two cost models, and a single cost line cannot hold them — state each against the tier that incurs it.
