# As-built spec format

One file per model, at `docs/data/<name>.as-built.md` — beside the elicited `docs/data/<name>.md`, so the diff is two files side by side and neither has to be re-explained.

The header states the object, its shape, and the revision the code was read at. An as-built spec is a snapshot, and a snapshot without a revision is a claim about the past.

```markdown
# orders_daily — as-built

| | |
|---|---|
| Object | `curated.orders_daily` |
| Shape | table |
| Read at | `a1b2c3d` (2026-09-25) |
| Read | `models/curated/orders_daily.sql`, `dags/orders_pipeline.py`, `terraform/athena.tf` |
```

Then a **findings index**: one line per non-`stated` row, pointing at its section. It is a table of contents, not a summary — the detail lives once, under its row.

```markdown
## Findings index

- `absent` — Grain, Freshness, Reconciliation, Look-back window
- `unread` — Owner of a failed refresh, Metric definition
- `implied` — Timezone, Materialization, Partition key, Access
```

Then the rows, in the necessity checklist's order and groupings. Each carries its class, its evidence, and — where the behavior is emergent — what actually happens:

```markdown
## Identity and time

**Grain** — `absent`. One row per order, intended, but nothing enforces it: no primary key in the DDL, no `unique` test in `schema.yml:14-22`, and `orders_daily.sql:31` joins `dim_customer` without deduplicating it, so a customer with two current rows fans out. → searched: DDL, `schema.yml`, `orders_daily.sql`

**Event time** — `implied`. `orders_daily.sql:18` filters `where ingested_at >= date_add('day', -1, current_date)`. → `models/curated/orders_daily.sql:18`

**Timezone** — `absent`. `date_trunc('day', ingested_at)` at `orders_daily.sql:12` carries no timezone, so the day boundary is the engine default. → searched: all model SQL, `dbt_project.yml`, `athena.tf`
```

Close with the three lists a reader wants without walking the spec:

```markdown
## Absent

Decisions no one has made. Each names what was searched.

## Unread

What could not be read, and the question that has to go to a person.

## Implied

Decided by the code, undocumented. The documentation debt.
```

## Rules

- **One row per checklist row.** Never merge two, never add one. The alignment is what lets `data-spec-review` walk its own checklist and find every row already resolved.
- **`implied` cites, `absent` scopes.** A row with neither is not resolved.
- **An `absent` row records observed behavior.** Never phrase it as a decision.
- **No recommendations.** The spec records the present. Every departure belongs in the diff, and the diff is the next step.
- **An `unread` row names the question.** "Not in the repo" is where you stopped looking; the question is what a person has to answer.
