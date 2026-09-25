# Graph frontier

The levels a knowledge graph adds, and the ones whose meaning changes. Reached from `SKILL.md` when the shape is a graph. The same rule binds every row: it resolves to a decision, to something that already behaves that way, or to an explicit `n/a - <reason>`.

## New levels

| Level | The decision | What it costs when it is skipped |
|---|---|---|
| **Ontology** | is the predicate vocabulary closed or open? | an open vocabulary physically forbids one table per relation, and discovering that after consumers bind is a rewrite |
| **Identity and resolution** | how a merge is recorded, and how a past answer is reproduced | no historical answer is reproducible, and merged entities cannot be un-merged |
| **Traversal** | the depth bound, and what is materialized as closure versus computed per query | a query that cannot exist, discovered at build time |
| **Directionality** | one row per edge with a direction, two rows, or a reverse view | every symmetric query pays a `UNION`, or the fat table doubles |

**Ontology first, always.** A closed vocabulary allows one table per relation type: typed columns, best compression, best statistics. An open vocabulary — extraction invented the predicates, which is the usual case — forbids that and forces one table with `relation` as a data column. This decides the physical model, so it cannot be deferred behind a later level.

**Identity is the level people skip.** Extraction merges entities that turn out to be the same thing. A warehouse cannot rewrite history, so the merge has to be recorded — a `merged_into` edge plus a resolution view — or the graph cannot answer "what did we believe last quarter", which is usually why the graph was landed there at all.

**Traversal is a bound, not a design.** Ask for the deepest question anyone needs answered, in hops, then check it against the platform limit. Anything deeper is materialized, and the materialization is named explicitly: a closure table, an adjacency array, or precomputed path counts.

## Levels whose meaning changes

| Level | Under a graph |
|---|---|
| **Grain** | becomes two sentences: what is a node, and what is an edge. An edge is not a row in a fact table — it carries text, provenance and validity |
| **Time** | becomes three axes that disagree: world truth (`valid_at`/`invalid_at`), observation (`reference_time`), and system belief (`created_at`/`expired_at`). Iceberg snapshots add a fourth. Every temporal question names the axis it means |
| **Measures** | almost nothing is additive across a path. Degree, centrality, community and path counts are computed, never summed, and a path count through a supernode is a cost decision |
| **Materialization** | adds closure tables and adjacency-with-arrays beside the view / materialized view / table / SPICE choice |
| **Layout** | partition by graph or tenant when queries are scoped. The extraction pipeline's partition column is usually already that key |
| **Cost** | supernodes, recursive-CTE depth multiplication, and embedding columns that are scanned but never aggregated |

## What stays the same

Consumer, Keys, Refresh, Access, Lifecycle and Reconciliation are unchanged: a graph still has an owner, a freshness promise, an access model, and a number you compare against a source of truth to prove it right. Reconciliation is both harder and more necessary for a graph, because there is no natural row count to sanity-check against — so state what you compare, or nothing can prove the graph correct.
