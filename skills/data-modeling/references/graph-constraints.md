# Graph constraints

Platform facts that bound a knowledge graph in Athena. Each carries its source; verify before a spec rests on a number.

## Traversal

- **Athena engine v3 supports recursive queries via `WITH RECURSIVE`, with a maximum recursion depth of 10.** The session property `max_recursion_depth` raises it, but the work is unbounded and Athena bills the bytes scanned. (https://docs.aws.amazon.com/athena/latest/ug/select.html)
- Consequence: **any traversal deeper than 10 hops is impossible in one query.** Hierarchy closure (is-a, part-of), multi-hop reachability and path finding must be materialized, not computed per query.
- A recursive CTE re-reads its own working set each iteration, so cost multiplies with depth. A hub node with a large degree makes an expansion explode rather than slow down.
- Always carry a termination condition. Without one the query fails on the depth limit or exhausts working buffers.

## Types

- `array<T>` and `map<K,V>` are supported, as are `struct` and nested combinations. `UNNEST` explodes an array into rows. (https://docs.aws.amazon.com/athena/latest/ug/querying-iceberg-supported-data-types.html)
- Iceberg supports `list<E>` → `array`, `map<K,V>` → `map`, `struct<...>`; `fixed(L)`, `char`, `tinyint` and `smallint` are not supported for Iceberg tables in Athena.
- **`map` requires one key type and one value type.** A heterogeneous attribute bag therefore stringifies, and every consumer casts forever. A `struct` keeps the types but needs a closed attribute set.
- Iceberg is what makes `UPDATE`/`DELETE`/`MERGE` available in Athena at all — on Hive tables the mechanism behind slowly-changing graph updates does not exist.

## Not available

- **No vector search.** No distance functions over embeddings and no ANN index. An embedding column in Athena is storage and scan cost with no query value; vector search belongs to Neptune Analytics or OpenSearch.
- **No graph query language.** No Cypher, no Gremlin, no pattern matching. Every traversal is a hand-written recursive CTE or a precomputed table.
- **No graph visual in QuickSight.** There is no network or node-link visual type, so graph insight reaches a dashboard only as precomputed flat tables — degree, community membership, centrality, co-occurrence, path counts.

## Where the boundary sits

Athena is the analytical mirror of a graph, not the graph. Traversal, neighborhood lookup and vector search belong to a graph engine — Neptune, or the store the extraction pipeline already writes to. Land a graph in Athena to answer questions *about* it: degree, centrality, community, co-occurrence, point-in-time belief, and joins between the graph and relational facts.

So the serving layer stays relational. The graph is upstream of the star schema, never a replacement for it.
