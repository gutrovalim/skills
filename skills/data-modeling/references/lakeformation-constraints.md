# Lake Formation constraints

Platform facts that disqualify a design. Each carries its source; verify before a spec rests on a number.

Access is the checklist group most often decided after the model. These are the ways the decision fails at build time, or worse, appears to succeed.

## A row filter is a small language, not SQL

- A row filter expression is a **subset of PartiQL**. Lake Formation "does not allow any user defined or standard partiQL functions in the filter expression". (https://docs.aws.amazon.com/lake-formation/latest/dg/partiql-support.html)
- You can compare a column to a **constant**, but **"you can't compare columns with other columns"** (same source).
- Supported operators: `=`, `>`, `<`, `>=`, `<=`, `<>`, `!=`, `BETWEEN`, `IN`, `LIKE`, `NOT`, `IS [NOT] NULL`. (same source)
- **All string comparisons and `LIKE` pattern matches are case-sensitive**, and `IS [NOT] NULL` cannot be used on partition columns (same source).
- Consequence: a rule the business states as "users see their own region" is expressible only if region is a column compared to a constant per principal. A rule needing a join, a function, or a column-to-column comparison **cannot be expressed at all** — and that is a design change, not a filter tweak.

## Types the filter cannot carry

- The `array` and `map` data types are **not supported in row filter expressions**; `struct` is. (https://docs.aws.amazon.com/lake-formation/latest/dg/data-filtering-notes.html)
- **Cell-level security is not supported on nested columns, views, or resource links** (same source).
- Consequence: a restricted field modelled as an array — a list of permitted tenants, a set of tags — has no row-filter expression, so the model changes shape or the access rule moves elsewhere.

## The quotas and the precedence that flip a design

- **100 data filters per principal per table**, though there is no limit on total filters defined on a table (same source).
- Applying a data filter with a row expression requires **`SELECT` with grant option on all table columns** (same source).
- If an **all-rows** filter expression is applied concurrently with predicated ones, **the all-rows expression prevails** (same source).
- Column names `ctid`, `oid`, `xmin`, `xmax`, `cmax`, `tableoid`, `insertxid`, `deletexid`, `importoid`, `redcatuniqueid` are restricted for filtering (same source).
- Cell-level security is column filtering and row filtering applied together (https://docs.aws.amazon.com/lake-formation/latest/dg/data-filtering.html).
- Consequence: an all-rows grant anywhere in the path **widens** access rather than narrowing it, and the filter count bounds how far per-tenant rules can be enumerated — past 100, the rule has to move to groups.

## Hybrid mode is how permissions silently do not apply

This is the one that looks like success and is not.

- If a table carries `IAMAllowedPrincipals` group permissions and the principal has **not** opted in to hybrid access mode, **Lake Formation permissions are not enforced** — "all principals in the account gets `Super` or `All` permissions on the table". (https://docs.aws.amazon.com/lake-formation/latest/dg/hybrid-access-workflow.html)
- If the table location is **not registered with Lake Formation**, only IAM applies. Lake Formation credential vending is unavailable unless the location is registered (same source).
- In hybrid mode, opted-in principals need **both** Lake Formation and IAM permissions; non-opted-in principals continue on IAM alone (https://docs.aws.amazon.com/lake-formation/latest/dg/hybrid-access-mode.html).
- Consequence: a spec that says "RLS is enforced by Lake Formation" is only true for principals that opted in, on tables that carry no `IAMAllowedPrincipals` grant, at registered locations. The spec states all three, or it states a claim that the first unregistered table in the account falsifies.

## Where the effective rule comes from

- A principal that is also in a group gets the **union** of its own and the group's row permissions (https://docs.aws.amazon.com/lake-formation/latest/dg/data-filtering-notes.html).
- For a cross-account grant, the effective predicate is the **intersection** of the account's predicate and any predicate granted directly to the principal (same source).
- Consequence: access is not readable from any single grant. The spec names the **principal**, not the group, when the two disagree — and cross-account rules narrow, so they cannot be used to widen a shared dataset.
