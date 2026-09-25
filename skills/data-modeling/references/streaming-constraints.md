# Streaming constraints

Platform facts that disqualify a design. Each carries its source; verify before a spec rests on a number.

The ingest end: a stream into a table. Serving is `athena-constraints.md`; what a stream leaves behind in the table is in `data-modeling`.

## Delivery is at-least-once, so a stream is not a grain

- Firehose uses **at-least-once** delivery: a timed-out delivery that later succeeds **duplicates the record**. The documented exceptions are Amazon S3 destinations, Iceberg Tables, and Snowflake destinations. (https://docs.aws.amazon.com/firehose/latest/dev/basic-deliver.html)
- **Producer retries duplicate regardless of destination.** `PutRecord` and `PutRecordBatch` are retried automatically on `ServiceUnavailableException`, and re-invoking either "can result in data duplicates" — AWS's own guidance is to "handle any duplicates at the destination". (https://docs.aws.amazon.com/firehose/latest/APIReference/API_PutRecordBatch.html)
- Consequence for a spec: **a stream is not a grain.** Every design over a stream needs a named dedup mechanism — a merge key, a unique constraint, an idempotent overwrite — and "the source sends each event once" is not one.

## Buffering is the freshness floor

- Firehose buffers before delivering: **size 1–128 MiB, interval 60–900 s**, and the **first condition satisfied** triggers delivery. Defaults are 5 MiB and 300 s. (https://docs.aws.amazon.com/firehose/latest/APIReference/API_BufferingHints.html)
- They are **hints** — "Firehose might choose to use different values when it is optimal" (same source).
- With **dynamic partitioning**, multi-stage buffering puts end-to-end delay at up to **1.5× the configured interval** (https://docs.aws.amazon.com/firehose/latest/dev/buffering.html).
- Consequence: the buffer interval is a **lower bound on freshness**, so it is a spec input rather than a tuning detail. A dataset promised "by 07:00" over a Firehose feed has to price the buffer and the 1.5× multiplier into that promise.

## Dynamic partitioning has a ceiling that diverts data

- **500 active partitions per stream.** Past that, records go to the S3 error prefix `activePartitionExceeded` — **not into the table**. Raisable to 2,500 by quota request. A partition is dropped once its buffer is delivered. (https://docs.aws.amazon.com/firehose/latest/dev/buffering.html)
- Consequence: a high-cardinality partition key — an id, a session — can exceed the ceiling and **silently divert records to an error prefix**, where no query sees them. Partition keys stay low-cardinality.

## Retention is the replay window

- Kinesis Data Streams retention: **24 hours default and minimum, 8,760 hours (365 days) maximum**, with extra charges above 24 h. (https://docs.aws.amazon.com/streams/latest/dev/kinesis-extended-retention.html)
- **Raising retention does not resurrect expired data** — records older than the previous period "remain inaccessible". (https://docs.aws.amazon.com/kinesis/latest/APIReference/API_IncreaseStreamRetentionPeriod.html)
- **Lowering it loses data immediately.** (https://docs.aws.amazon.com/kinesis/latest/APIReference/API_DecreaseStreamRetentionPeriod.html)
- Consequence: retention bounds every **backfill and reprocess**, which makes it a one-way door — raise it before the first incident, because raising it afterwards recovers nothing. A spec whose backfill needs a week of stream data needs the retention raised *now*, not when the backfill runs.

## What a stream does to the table

- Iceberg commits under **optimistic concurrency**: a writer that loses the atomic swap re-applies its changes to the new state, or fails validation if its files are gone. (https://iceberg.apache.org/docs/latest/reliability/)
- **Streaming writes produce small files.** Iceberg's own guidance is a **minimum 1-minute trigger interval**, regular compaction via `rewrite_data_files`, and snapshot expiration — because every batch creates a snapshot and snapshots accumulate. Default snapshot expiration is 5 days. (https://iceberg.apache.org/docs/latest/spark-structured-streaming/)
- Iceberg's Spark streaming reader **reads append snapshots only**: overwrite and delete snapshots raise an exception unless explicitly skipped (same source).
- Consequence: compaction and snapshot retention are **part of the spec, not maintenance**. A streaming table with no named compaction owner is a table whose query cost climbs while its data volume does not.
