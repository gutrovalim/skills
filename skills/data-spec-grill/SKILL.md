---
name: data-spec-grill
description: Grill the user into a spec for a dataset, table, graph, key-value store, or pipeline before anything is built. Use when the user says "spec this dataset", "grill the data model", "dataset spec", "table spec", "pipeline spec", "knowledge graph", or "DynamoDB", or names a QuickSight dataset, an Athena view, or a materialized table as the thing to design. Do NOT use to review a spec that already exists (use data-spec-review) or to design application code.
---

# Data spec grill

Interview the user until the data model is decided. Same design tree as `grilling`, with one difference: the frontier starts **seeded**.

An application's frontier grows out of the codebase, so you discover it. A dataset's frontier is knowable before you start, because every dataset carries the same decisions. Seeding it is the enclosure: a dimension nobody asks about is a dimension the SQL decides by accident, and a grain decided by accident is wrong data that still looks plausible.

Work in **rounds**. Ask the whole frontier at once, numbered, each with your recommended answer. Wait for the answers, then recompute the frontier.

## Facts are yours, decisions are theirs

Resolve every fact yourself: what the source table's grain actually is, whether a column is numeric, what the existing pipeline refreshes, which partition key is in use. Read the DDL, the catalog, the existing SQL. Never ask for a fact you could have looked up, and never invent one — a fabricated column or limit propagates into the spec, then into a query that returns a wrong number confidently.

Put to the user only what is genuinely theirs: what the number is for, who may see it, what freshness is worth paying for, which of two defensible grains is the business truth.

Read [data-modeling](../data-modeling/SKILL.md) before asking about materialization, layout, or refresh. It carries the platform constraints that decide which options exist at all, and a round that offers an impossible option burns the turn.

## Shape first

One question gates the whole frontier: **is this a table, a graph, or a key-value store?** The three share the levels below but not their meaning.

- **A graph** adds four levels a table never has, and changes what grain, time and measures mean. Read [graph-frontier.md](references/graph-frontier.md).
- **A key-value store inverts the order.** It is modelled from its access patterns rather than from its questions, so the enumeration comes before any key is designed, and the key *is* the schema. Read [kv-frontier.md](references/kv-frontier.md).

Settle the shape before anything else. A round that asks the wrong shape's questions burns the turn — and for a key-value store it asks them in the wrong order, which is worse.

## The seeded frontier

Levels settle top-down: nothing below grain means anything until grain is fixed.

| Level | The decision | What it costs when it is skipped |
|---|---|---|
| **Consumer** | who reads this, and which decision it drives | a dataset nobody queries; effort spent on a question nobody asks |
| **Grain** | what exactly is one row | silent duplication; every measure downstream is wrong and still looks plausible |
| **Keys** | what makes a row unique, and what *enforces* it | a retry or a backfill appends duplicates |
| **Time** | event or processing time; timezone; where a day starts | two reports disagree; month boundaries move by an hour |
| **Measures** | additive, semi-additive, or ratio; the denominator; the null rule | summing an average; a rate whose denominator changed silently |
| **History** | snapshot or delta; correction or restatement; SCD | last-write-wins destroys the history someone asks for next quarter |
| **Materialization** | view, materialized view, table, or SPICE dataset | a view re-scanned per dashboard; an MV that cannot express the query |
| **Layout** | partition key, file format, compaction | a full scan per query; a scheme that cannot be backfilled |
| **Refresh** | who refreshes what, in what order, to what freshness | a child dataset built on a parent that refreshes after it |
| **Access** | RLS, CLS, definer or invoker semantics | a rule that cannot be expressed, found after the dataset is shared |
| **Cost** | scan budget per query; dataset size against its quota | a dashboard costing more than the decision it informs |
| **Lifecycle** | retention, deletion, PII | data kept past its lawful life |

Ask them in that order, in rounds: Consumer and Grain together, then Keys, Time and Measures, then the rest once the shape is stable.

**Every level resolves** — to a decision, to something that already behaves that way, or to an explicit `n/a - <reason>`. The `n/a` escape is mandatory, and it is what keeps the list from inventing work: a fully public dataset has no access decision, and writing that down costs one line. A level left blank is the failure this skill exists to prevent.

## The four questions that come back wrong

Most of the frontier is answerable in one line. These four are not, and they are where a model goes wrong while every individual answer given was reasonable.

**Grain is stated, never assumed.** "One row per order" and "one row per order line" produce the same-looking dashboard and different numbers. Ask for the sentence, then ask what happens when the source sends the same order twice. If the answer is "it shouldn't", grain has no mechanism and is a key decision, not an observation.

**Time is two questions, not one.** Which timestamp the business means (event time), and which one the pipeline can see (processing time). Late-arriving rows are exactly the gap between them, so a spec naming one timestamp has quietly decided to ignore late data. Then timezone, and where the day boundary falls — a dashboard read from Sydney and from Seattle must not disagree about yesterday.

**A guarantee needs a mechanism.** "No duplicates", "one row per customer", "always current" are claims about machinery. Point at what enforces it — a merge key, a dedup window, a snapshot, a unique constraint — or it is a decision still to make, not a property to record.

**Cost is a spec input, not an outcome.** Athena bills per byte scanned, with a 10 MB minimum per query. A spec that states no scan budget has left the decision to whoever builds the dashboard, and they will make it badly, at whatever interval the dashboard refreshes.

## Output

Write the spec as you go rather than at the end, at `docs/data/<name>.md` or the path the user names. One section per level in the frontier's order, so a thin level shows up as a short section instead of an absence, plus:

- `## Constraints` — every platform limit the spec depends on, each with its source link. These are load-bearing: a materialized view chosen for a query outside the refresh subset, or an RLS rule over a numeric column, fails at build time and invalidates the design.
- `## Delegated` — decisions the user handed back, recorded with the default you chose. Discretion on the record, never inferred from silence.

The session ends when the frontier is empty and every level carries a decision or an `n/a`. Hand the spec to [data-spec-review](../data-spec-review/SKILL.md) before anyone writes SQL: a bad spec is cheaper to catch than the query it produces, and the query is cheaper to catch than the dashboard built on it.
