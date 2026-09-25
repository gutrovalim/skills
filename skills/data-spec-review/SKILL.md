---
name: data-spec-review
description: Review a dataset, table, graph, key-value store, or pipeline spec before anyone builds it — what a buildable spec must state, plus an adversarial pass that attacks the model's own assumptions. Use when the user says "review this spec", "is this dataset spec complete", "attack this model", or before implementing Athena or QuickSight data work. Do NOT use to interview a spec into existence (use data-spec-grill) or to review the code a spec produced (use code-review).
---

# Data spec review

Review the spec, not the code. A bad spec is cheap to catch; the query it produces is expensive; the dashboard built on the query is more expensive still, because by then the numbers have been read and believed.

Two passes, and they are different work. The **necessity** pass checks a fixed list, so it finds what is *missing*. The **adversarial** pass attacks what is *present*, so it finds what is stated and wrong. A spec can pass either and fail the other: a complete spec can be confidently incorrect, and a correct spec can omit the one thing that makes it buildable.

## Necessity

Read [necessity-checklist.md](references/necessity-checklist.md) and walk it. Every row resolves to `stated`, `n/a - <reason>`, or **missing**. A blank is a finding; so is an `n/a` with no reason, because the reason is what makes the omission reviewable.

Report the three invalidating rows first — a spec that cannot name its grain, its freshness promise, or its reconciliation cannot be built, only guessed at. When the shape is a graph, the graph rows apply too, and grain splits in two: what is a node, and what is an edge.

## Adversarial

The author of a spec cannot find its faults, so the attack runs as a **sub-agent** that did not write it. Dispatch one sub-agent with the spec and the brief below. Do not run the attack yourself, and do not soften it: the value is in the disagreement, and a reviewer who agrees with the author has added nothing.

The brief names the axes, because "find problems" finds none. Each axis is a way a data model passes review and still fails in production:

| Axis | The attack |
|---|---|
| **Grain collision** | Find two rows the spec would produce for one business event, or one row where the business sees two. |
| **Time** | Where does late-arriving data land? What does a report read from another timezone show? Which day does a row at 23:30 belong to? |
| **Restatement** | A source corrects a past period. Does the model overwrite, duplicate, or silently ignore it? |
| **Refresh order** | Name every pair where one object must refresh after another, and check the spec states that order. |
| **Materialization** | Is every materialized-view definition inside the documented refresh subset? Is every SPICE dataset inside its quota, and does any inherited RLS force direct query? |
| **Access** | Can every RLS rule be expressed in the types available — are any restricted fields numeric or date? Does any user have no rule, and therefore no data? |
| **Cost** | What is the worst query this model permits, in bytes scanned, and does the spec bound it? |
| **Silent emptiness** | Where can this return zero rows without erroring — partition projection, a look-back window that misses a correction, a filter that excludes everything? |
| **Reconciliation** | What number would you compare against a source of truth to prove the dataset is right? If the spec names none, nothing can prove it. |

Report per axis: the spec line, the scenario that breaks it, and the smallest change that fixes it. **Under 400 words**, and a clean axis is reported clean — padding the list with maybes buries the real finding.

When the spec's shape is a graph, add these axes — the graph-specific ways a model passes review and still fails:

| Axis | The attack |
|---|---|
| **Ontology drift** | Does the model assume a closed relation vocabulary? What happens when extraction invents a predicate — a new column, or a new value? |
| **Identity** | Was any entity merged with another? Is the merge recorded, and can a past answer be reproduced from it? |
| **Traversal depth** | What is the deepest question asked, in hops, and is it inside the platform's recursion bound, or materialized? |
| **Directionality** | For a symmetric relation, is there one row or two, and what does each query pay for that choice? |
| **Temporal axis** | For every "as of" question, does it mean world truth, observation, or system belief — and do any two of those disagree in this model? |
| **Supernode** | What is the highest-degree node, and what does expanding it cost? |

When the spec's shape is a key-value store, add these — and check the ordering, since a key designed before the access patterns were enumerated is the failure this shape invites:

| Axis | The attack |
|---|---|
| **Access patterns** | Is every read and write the store serves enumerated, each with its key, and was the design derived from that list rather than preceding it? |
| **Cardinality** | What is the traffic distribution across partition key values — is any single value a hot partition? |
| **Expiry** | Does the spec treat an expiry attribute as deletion? What actually enforces the stated retention? |
| **Landing path** | Which path moves this data to the analytics tier, at what latency, and can it answer the questions asked of it? A state sync cannot answer a history question |
| **Cost model** | Does the spec state cost per request for the store and per byte scanned for the warehouse, or has it collapsed two models into one line? |
| **Reconciliation snapshot** | When the analytics copy is compared to the store, which point in time is it compared against, given that expiry and streams both lag? |

## Verdict

Report the two passes separately, never merged or reranked: necessity finds absences, adversarial finds errors, and one severity list lets either mask the other.

End with the smallest set of changes that makes the spec buildable, and name the single finding that invalidates the design rather than merely weakening it.
