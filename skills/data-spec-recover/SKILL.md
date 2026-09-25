---
name: data-spec-recover
description: Recover the as-built spec from an existing data codebase — read the DDL, defining SQL, orchestration, serving config and access policies, and write down what the model actually decided, each row citing its evidence. Use when there is no spec, or the spec and the pipeline disagree: "there's no spec", "reverse-engineer the data model", "document what this pipeline does", "as-built spec", "recover the spec from the code", "we inherited this warehouse". Do NOT use to interview a new spec into existence (use data-spec-grill), to attack a spec (use data-spec-review), or to review application code (use code-review).
---

# Data spec recover

A codebase is a spec with the decisions already made and none of them written down. Recovering them is not documentation for its own sake: it produces the **as-built spec**, which is the *left side of a diff*. Set it against the elicited *as-needed* spec — what the business and the consumers actually want — and the findings live in the gap. Neither artifact produces them alone, and the as-built one reviewed by itself is just the code's assumptions agreeing with themselves.

So the as-built spec is not a review target. `data-spec-review`'s necessity pass finds *missing* rows, and a codebase has no blanks: every row is answered by a `CREATE TABLE`, a `MERGE`, or a cron expression. What it has instead is rows answered four different ways, and only three of them are findings.

## The four classes

| Class | What it means |
|---|---|
| `stated` | the code decides it, and something a human can read says so — a comment, a doc, a ticket |
| `implied` | the code decides it and only the code says so. Readable, undocumented. This is the documentation debt. |
| `absent` | nothing decides it. The behavior is **emergent** — whatever the platform does when nobody chose. |
| `unread` | could not be read: no access, not in the repo, lives in a console. |

Every row resolves to one of these. Two rules keep them honest:

- **`implied` cites its evidence.** File and line, or the exact statement. Without a citation the spec is a paraphrase, and a paraphrase cannot be audited — the reviewer ends up checking your reading of the SQL instead of the model.
- **`absent` names its search scope.** Where you looked and found nothing. Same discipline as the necessity checklist's `n/a - <reason>`: the reason is what makes the omission reviewable. "No mechanism found" is not a finding until it says where you looked.

## Emergent is the trap

The failure this skill exists to prevent is reading a default and writing it down as a decision.

An `event_time` predicate becomes "event time is processing time" in prose. Nobody chose that; it is what happens to late data when no one thought about it. A recovered spec that launders an omission into a decision is worse than no spec, because it now reads as authoritative.

So when you find no mechanism the row is `absent` — and the *observed behavior* is still recorded, so the reader can see what is actually happening:

> **Event time** — `absent`. Rows land in the partition of their processing date. No look-back window in the DAG, the incremental predicate, or `dbt_project.yml`. A source correction older than the current day is not picked up.

That is a finding. "Event time is processing time" would have been a fabrication.

## Record, don't repair

Recovering a model is not changing it. Write down what the code does; the decision to change it belongs after the diff, with the consumer in the room. A pass that starts fixing SQL produces neither a spec nor a review.

## The run

### 1. Scope

Enumerate every serving object from the repo — table, view, materialized view, dataset, graph, store. Look in DDL, migrations, model SQL, catalog exports and IaC.

**Recover what is read, not what exists.** Rank objects by consumer, then work the ones that have one. An object nobody queries cannot mislead anyone, and the full map over every table is a bad trade. Consumers are visible in dashboard and dataset definitions, lineage, and query logs where you have them.

Every object is either in scope or excluded with a reason. Done when the list is exhaustive and each exclusion names why.

### 2. Shape gate

Per object: **table, graph, or key-value store.** The three share the map's universal rows but not their meaning — a graph's grain splits into node and edge, and a key-value store is read in the reverse order, access patterns before keys. Settle the shape first; walking the wrong shape's rows wastes the pass.

### 3. Extract

Walk [references/extraction-map.md](references/extraction-map.md) row by row, per object. Each row resolves to a class, with a citation or a search scope.

The map's rows mirror `data-spec-review`'s necessity checklist row for row, and that alignment is the integration contract: when the as-built spec uses the checklist's rows, the review finds each one already resolved, and its findings become "row X is `absent`" instead of "row X is missing".

Read [data-modeling](../data-modeling/SKILL.md) for the vocabulary and the platform constraints. A `Materialization` row is only meaningful against the options that actually exist, and a `Refresh order` row means nothing until you know the platform does not sequence refreshes for you.

The map marks each row's **recoverability**, because recovery runs out at a predictable point:

- `visible` — the code decides it, and the code says what it is.
- `mechanism-only` — the code shows the mechanism; the promise or the meaning is not in the code. Freshness is the type case: a schedule is recoverable, the agreed number and its timezone are not.
- `human-only` — the code cannot answer it. Who owns a failed refresh, what a metric means to the business, what a consumer does with the number.

A `human-only` row is not a failure to search harder. Record it as `unread` with the question that has to go to a person, and move on.

Done when every in-scope object has every row resolved, each `implied` cited, each `absent` scoped.

### 4. Write the as-built spec

One file per model at `docs/data/<name>.as-built.md` — beside the elicited `docs/data/<name>.md`, so the diff is two files side by side. Format and a worked row: [references/as-built-format.md](references/as-built-format.md).

Sections in the checklist's order. A thin section shows up as a thin section, never as an absence.

### 5. Hand off

The recovered spec goes to [data-spec-review](../data-spec-review/SKILL.md), which dispatches the adversarial pass as a sub-agent that did not write it. Give that pass **the code alongside the spec, never the spec alone** — otherwise it audits your reconstruction instead of the model, and a misreading of the DDL becomes invisible.

Two separations hold the value:

- **The recoverer does not attack.** Whoever read the SQL will defend their reading of it. The adversarial pass runs elsewhere.
- **The recoverer does not elicit.** Reading a schema anchors every requirement conversation after it — "well, the table already has a `region` column". So the as-needed spec comes from someone who has not read the as-built one, or from a pass run first. Otherwise the diff compares the code to itself and comes back clean.

Done when the spec and the code paths are named, the anchoring warning is stated, and the review is handed off rather than run.
