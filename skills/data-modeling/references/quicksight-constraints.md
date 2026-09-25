# QuickSight constraints

Platform facts that disqualify a design. Each carries its source; **quotas drift** — the SPICE row limit has been raised twice — so re-verify before a spec rests on a number.

## SPICE quotas

Per dataset (https://docs.aws.amazon.com/quick/latest/userguide/data-source-limits.html):

- **Standard edition: 25 million rows or 25 GB per dataset. Enterprise edition: 2 billion rows or 2 TB per dataset.** Whichever limit is reached first applies.
- 2,000 columns per file; column names up to 127 characters; a field up to 2,047 Unicode characters (65,534 with the new data preparation experience).
- A manifest may specify up to 1,000 files.
- **Up to 32 tables** may be joined into one dataset.
- **SPICE capacity is account-level, not per dataset**: one oversized dataset can exhaust the pool and cause *other* datasets' ingestions to fail. Auto-purchase exists, but a spec that ignores total capacity is a spec that breaks someone else's dashboard.

## Refresh

- **Direct query** datasets refresh when opened. Filter controls refresh automatically every 24 hours.
- **SPICE** refresh requires stored credentials, and **manually uploaded data cannot be refreshed** — including from S3, because the connection metadata is not retained. To refresh from S3, build the dataset from the **S3 data source card** rather than an upload.
- On-demand limits **per dataset per 24 hours**: **32 full refreshes**, **100 incremental refreshes**. Once hit, further on-demand refreshes are refused — **scheduled refreshes keep running** and are unaffected. (https://aws.amazon.com/blogs/business-intelligence/best-practices-for-amazon-quicksight-spice-and-direct-query-mode/)
- **Incremental refresh is Enterprise-only and SQL-sources-only.** It takes a **look-back window**, deletes the SPICE rows inside that window, and replaces them with the freshly queried rows. Anything changed *outside* the window is never picked up. (https://docs.aws.amazon.com/quicksight/latest/user/refreshing-data.html)
- Incremental refresh schedules can run as often as **every 15 minutes**.
- **Custom SQL may defeat incremental refresh.** If the query is complex the source cannot push the look-back filter down, and the "incremental" refresh costs more than a full one. That makes it a spec decision, not a tuning detail: if a dataset needs incremental refresh, its SQL must be shaped so the window is optimizable.

## Datasets over datasets

- A **child dataset inherits the parent's preparation** — joins, calculated fields — and its security settings. (https://docs.aws.amazon.com/quick/latest/userguide/create-a-dataset-existing-dataset.html)
- **Refresh schedules are not synchronized between parent and child.** A child over a SPICE parent needs its own schedule, and if the child runs first it keeps serving the parent's previous contents with no error. Order the schedules explicitly, or the dashboard is silently one period behind.
- **Joins are same-source or cross-source.** A dataset is cross-source if any logical table is a SPICE parent, or if the parents are not all from one data source. A parent used in a join must be **direct query** and on the same source for the join to stay same-source.
- **A child of an RLS-enabled parent can only be direct query**, and inherited RLS rules **are not supported in SPICE**. So RLS at the parent level removes SPICE from the design entirely.

## Row-level security

Enterprise edition only. (https://docs.aws.amazon.com/quicksight/latest/user/row-level-security.html, https://repost.aws/knowledge-center/quicksight-fix-row-level-security-issues)

- Enforced at **analysis and dashboard** level, for every user. The data preparation page shows all rows, but only to the dataset owner.
- The **rules dataset** is a separate dataset flagged `UseAs = RLS_RULES`: one identity column (`UserName`/`GroupName`/`UserARN`/`GroupARN`) plus **one column per restricted field, whose name must exactly match the main dataset**, and **every column must be string type**.
- **RLS supports only textual fields.** A rule cannot be written over a date or a numeric column — so a design that restricts access by a numeric `region_id` cannot use RLS as specified, and needs the field cast to string or the rule moved.
- **NULL in a rule means all values.** A user or group with **no rule sees no data** — deny by default, which makes a missing rules row a silent outage rather than an over-share.
- Multiple fields in one rule combine with **AND**; **OR is not supported**. "Region A *or* Region B" must be expressed as two rules.
- **At most 999 rule records per user.** Beyond that, RLS may fail to apply and *restricted users can still see everything* — a security failure that looks like success. Use groups rather than enumerating users.
- Rules datasets take no duplicate rows.
- Rows with **null in the restricted field are not restricted** by the rule.
- **Tag-based rules** apply only to embedded dashboards for anonymous users via `GenerateEmbedUrlForAnonymousUser`; registered-user embedding uses user-based rules.

## Column-level security

Enterprise edition only. (https://docs.aws.amazon.com/quick/latest/userguide/restrict-access-to-a-data-set-using-column-level-security.html)

- Restricts named columns per user or group; by default all groups and users have access to all columns.
- A table or pivot table with a restricted column in the **Rows** or **Columns** well cannot be seen at all by a user without access. In the **Values** well the visual renders and shows **"Not Authorized"** for that column — so a CLS change alters what an existing dashboard shows, not only who can open it.
- Enabling CLS requires administrator access.
