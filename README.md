# skills

Agent skills for designing and reviewing **data models** — datasets, tables, graphs, key-value stores, and the pipelines that fill them.

## The goal

A wrong data model produces numbers that look plausible. By the time a dashboard is built on them they have been read and believed, so the cost of the mistake climbs at every step: a bad spec is cheap to catch, the query it produces is expensive, and the dashboard built on the query is more expensive still.

These skills push those decisions to the front — what one row means, where the day boundary falls, what happens when a source corrects a past period — and make them explicit enough to argue with before anyone writes SQL.

## The skills

| Skill | What it does |
|---|---|
| [`data-modeling`](skills/data-modeling/SKILL.md) | The reference. Modelling vocabulary, the one-way doors, and ten platform and shape constraint files sourced from primary vendor docs. |
| [`data-spec-grill`](skills/data-spec-grill/SKILL.md) | Interviews you into a spec for a dataset that does not exist yet. |
| [`data-spec-review`](skills/data-spec-review/SKILL.md) | Attacks a spec before it is built — a necessity checklist, then an adversarial pass. |
| [`data-spec-recover`](skills/data-spec-recover/SKILL.md) | Recovers the as-built spec from code that already exists, with a citation per row. |

`data-modeling` is consulted by the other three. It holds the platform constraints that decide which designs are possible at all, so it is the one to read when a decision looks free but is not.

## Vendored skills

The engineering and productivity skills from [mattpocock/skills](https://github.com/mattpocock/skills) (commit `c55ee46`) are copied unmodified into `skills/`, including `grilling` and `code-review`, which the data skills refer to. They are MIT licensed; see `licenses/mattpocock-skills-LICENSE`.
