# Key-value frontier

The levels a key-value store adds, and the one that inverts. Reached from `SKILL.md` when the shape is a key-value store. The same rule binds every row: it resolves to a decision, to something that already behaves that way, or to an explicit `n/a - <reason>`.

## The inversion

Every other shape is modelled from the questions. A key-value store is modelled from the **access patterns**, and that reverses the interview: Consumer stops validating the model and starts generating it.

So Access patterns is the first level, not a later one, and it is asked as an enumeration — every read and every write the store must serve, each with the key that serves it. A key design produced before that list exists is a design that will be rebuilt.

The reason is that the key **is** the schema. In a warehouse, changing the grain is a re-derivation; here it is a table rebuild and a migration, with the data copied and every client repointed. It is the least reversible door in the data space.

## New levels

| Level | The decision | What it costs when it is skipped |
|---|---|---|
| **Access patterns** | the enumerated reads and writes, each with its key, before any key is designed | a table rebuilt after launch, with a migration |
| **Key design** | partition key, sort key, and the item collection they define | a hot partition throttling under load |
| **Cardinality** | how many distinct partition key values the real traffic hits | throttling at the per-key-value ceiling, invisible in testing |
| **Secondary access** | which extra patterns a global or local secondary index serves | an index that cannot be added without a backfill, or one that blocks partition splitting |
| **Consistency** | where a strongly consistent read is actually required | a read that is occasionally stale, found in production |
| **Landing path** | how this store's data reaches the analytical tier, and at what latency | an analytics tier that cannot see the store's data at all |
| **Retention mechanism** | what actually deletes, as opposed to what is meant to expire | data retained well past its stated life |

**Cardinality is the level that hides.** A partition key can look sensible — a date, a status, a region — and still funnel traffic into one physical partition. That failure never appears in a functional test, only under load, so it has to be reasoned about at spec time or it is discovered by users.

**Secondary access is a door, not a detail.** A local secondary index prevents the store from splitting a hot item collection, which caps the recovery available later. Prefer a global secondary index, even when it carries the same partition key.

**Retention is a claim about machinery.** A per-item expiry attribute is a hint to a background process, not a deletion guarantee: expiry is best-effort and can lag by days, and an expired item still occupies storage and still appears in reads until it is removed. If the spec says "retained 90 days", name what enforces that — a filter, a job, or the analytics tier — or the store does not deliver it.

## Levels whose meaning changes

| Level | Under a key-value store |
|---|---|
| **Grain** | becomes the item, and the item collection it belongs to. Both are fixed by the key, not chosen separately |
| **Keys** | merges into key design — the key is no longer what makes a row unique, it is what makes the store fast |
| **History** | usually absent by design. If history is needed it lives in the change stream or the analytics tier, never in the store |
| **Cost** | becomes per request and per provisioned capacity, not per byte scanned. A spec spanning a store and a warehouse carries **two cost models**, and must name both |
| **Materialization** | the store is itself a serving tier, so this level asks how the store is *served to* — direct access, a cache, or an analytical copy |
| **Reconciliation** | points at a truth that is itself eventual: expiry lags, streams lag, and exports are point-in-time snapshots. Name which snapshot is compared |

## What stays the same

Consumer, Time, Access, Lifecycle and Layout still apply. Access does not disappear — a store with per-request authorization needs its access model stated as much as a table does.
