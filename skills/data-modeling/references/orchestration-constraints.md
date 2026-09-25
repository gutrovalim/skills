# Orchestration constraints

Platform facts that disqualify a design. Each carries its source; verify before a spec rests on a number.

Refresh order is a one-way door in `data-modeling`. This is where it is enforced — and where it silently is not.

## Catchup is a backfill you did not ask for

- Airflow's `catchup_by_default` is **`False`**. A DAG with `catchup=True` makes the scheduler "kick off a Dag Run for any data interval that has not been run since the last data interval". (https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/dag-run.html)
- **Re-enabling a paused DAG triggers catchup** for the intervals it missed (same source).
- Consequence: a DAG is either **interval-addressed** or **now-addressed**, and catchup is safe only for the first. `catchup=True` on a DAG whose tasks read `Now` produces one run per missed interval, each recomputing the same window over the same data.

## depends_on_past serialises, it does not order

- `depends_on_past=True` makes a task wait for the **previous run of itself** to succeed (https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/dags.html).
- Consequence: it is a **same-task, previous-interval** dependency and cannot express "child after parent". On a DAG with a backlog it converts parallel work into a queue, which lengthens the very refresh it was added to protect.

## Data-aware triggers are the only real cross-object edge

- In Airflow 3 the concept is **Assets**, renamed from Datasets in 3.0 (https://airflow.apache.org/docs/apache-airflow/3.0.0/release_notes.html).
- A downstream DAG is scheduled when an upstream task **successfully** updates an asset: "If the task fails or if it is skipped, no update occurs, and Airflow doesn't schedule the consumer Dag." (https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/asset-scheduling.html)
- A DAG consuming several assets runs **once, after all of them have updated**; conditions combine with `&` and `|` (same source).
- A **partition-aware** DAG (`PartitionedAssetTimetable`) is not triggered by an event carrying no `partition_key` (https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/assets.html).
- A security consequence worth a line in the spec: `can_create` on Assets is equivalent to trigger permission on **every** downstream DAG, because data-aware scheduling uses an implicit trust model (same source).
- Consequence: this is the mechanism that actually draws the parent-child edge, so a spec that states refresh order **without naming the trigger** has stated a wish, not a dependency.

## The dependency nobody draws

- Airflow orders tasks within a DAG. dbt orders models within a project. **Neither orders across the boundary between them**, and a BI dataset's schedule lives in the BI tool, where no DAG can see it.
- Consequence: every **cross-tool** pair — DAG → dbt, dbt → dataset, dataset → child dataset — is ordered by schedule offsets that nobody compares. That is exactly the failure `data-modeling` names: the child serves the parent's previous contents and nothing errors.
- So the spec states the order **and** the mechanism that enforces it. Where the platform does not sequence, the pipeline must, and a cron offset is a mechanism — a fragile one, which is the argument for writing it down where it can be reviewed rather than leaving it in a schedule nobody reads.
