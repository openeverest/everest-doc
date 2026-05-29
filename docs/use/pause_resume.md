# Pause and resume databases

Pausing a database stops all engine and proxy pods while preserving everything else: storage volumes, credentials, configuration, and backup metadata. The cluster can be fully resumed at any time.

!!! note
    The OpenEverest UI labels this action **Suspend**. The API and CRD use the term **paused**. They refer to the same feature.

## Use cases

- **Development and test environments** — shut down non-production clusters overnight or over weekends to free up compute resources
- **Batch-driven workloads** — keep a database offline between scheduled batch jobs and bring it up only when the process runs
- **Cost optimization** — reduce infrastructure costs for databases that are used infrequently without losing data or configuration

## Before you pause

!!! warning
    - All active connections to the database are dropped immediately when pausing.
    - Scheduled backups do not run while the cluster is paused.
    - PITR log archiving is suspended for the duration of the pause. WAL segments are preserved on existing storage but no new segments are archived until the cluster resumes.
    - Storage volumes are retained and continue to incur storage costs.

## Pause a database

=== "UI"

    1. From the OpenEverest homepage, locate the database you want to pause.
    2. Click the ellipsis menu (**...**) next to the database name.
    3. Select **Suspend**.

    The database status changes to **Paused**. All engine and proxy pods are stopped.

=== "API"

    Send a `PATCH` request to the [`updateDatabaseCluster`](https://openeverest.io/docs/api/#/operations/updateDatabaseCluster){:target="_blank"} endpoint and set `paused` to `true`:

    ```json
    {
      "spec": {
        "paused": true
      }
    }
    ```

=== "CRD"

    Set `spec.paused` to `true` in the `DatabaseCluster` manifest and apply it:

    ```yaml
    apiVersion: everest.percona.com/v1alpha1
    kind: DatabaseCluster
    metadata:
      name: my-database-cluster
    spec:
      paused: true
    ```

    ```sh
    kubectl apply -f my-database-cluster.yaml
    ```

## Resume a database

=== "UI"

    1. From the OpenEverest homepage, locate the paused database.
    2. Click **Resume** in the database row, or open the ellipsis menu (**...**) and select **Resume**.

    The database status transitions from **Paused** back to **Ready** once all pods are running.

=== "API"

    Send a `PATCH` request to the [`updateDatabaseCluster`](https://openeverest.io/docs/api/#/operations/updateDatabaseCluster){:target="_blank"} endpoint and set `paused` to `false`:

    ```json
    {
      "spec": {
        "paused": false
      }
    }
    ```

=== "CRD"

    Set `spec.paused` to `false` (or remove the field) and apply:

    ```yaml
    apiVersion: everest.percona.com/v1alpha1
    kind: DatabaseCluster
    metadata:
      name: my-database-cluster
    spec:
      paused: false
    ```

    ```sh
    kubectl apply -f my-database-cluster.yaml
    ```

## Monitor pause status

To check the current status of your database cluster:

```sh
kubectl get databasecluster <cluster-name> -o jsonpath='{.status.state}'
```

While paused, the status field returns `paused`. Once resumed and fully operational, it returns `ready`.

For the full list of possible status values, see [Status monitoring](../crd/status_monitoring.md).

## See also

- [Database view](database_view.md) — full list of actions available from the database menu
- [Scale database deployment](scaling.md) — adjust resources when resuming after a long pause
- [Manage database clusters with CRD](../crd/database_cluster_management.md) — full `DatabaseCluster` spec reference
