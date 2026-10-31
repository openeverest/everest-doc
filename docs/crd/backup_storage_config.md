# Configure backup storage

Before configuring backups, you must set up a backup storage location.

A `BackupStorage` is a namespaced resource: a bucket, optionally with a prefix within it, plus the connection parameters, the credentials and the TLS settings needed to reach it. It names a cluster-scoped `BackupStorageClass`, the object storage technology. OpenEverest ships one class, `s3`, which covers Amazon S3 and S3-compatible stores such as MinIO, Ceph RGW, SeaweedFS and the Google Cloud Storage (GCS) interoperability endpoint.

The storage class is the technology, the API spoken to the store. The `implementation` parameter of the `s3` class is a different thing: it says which product serves the S3 endpoint, so that each database engine can pick the right client settings.

## Backup storage classes

List the backup storage classes available in the cluster:

```sh
kubectl get backupstorageclasses
```

??? example "Expected output"

    ```{.text .no-copy}
    NAME   DISPLAY NAME                   AGE
    s3     S3-compatible object storage   5d
    ```

`bsc` is the short name. A class defines the connection parameters a backup storage of that class accepts (`spec.parametersSchema`) and its authentication modes (`spec.authModes`). The `s3` class has one authentication mode, `AccessKey`, which reads the `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` keys from a Secret.

Each `BackupClass` declares the storage classes, and the authentication modes of each, that it can write to:

```yaml
apiVersion: backup.openeverest.io/v1alpha1
kind: BackupClass
metadata:
  name: percona-backup-mongodb
spec:
  backupStorageClasses:
    - name: s3
      authModes:
        - AccessKey
  # ...
```

A backup storage can be used with a backup class only when both its storage class and its authentication mode are listed. The UI offers only the backup storages the instance's backup class can write to.

## Create a backup storage

To create a backup storage:
{.power-number}

1. Create a Secret holding the credentials, in the namespace of the backup storage:

    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: backup-storage-s3-prod-credentials
      namespace: team-a
    type: Opaque
    stringData:
      AWS_ACCESS_KEY_ID: <access key id>
      AWS_SECRET_ACCESS_KEY: <secret access key>
    ```

2. Create the `BackupStorage` that references this Secret:

    ```yaml
    apiVersion: backup.openeverest.io/v1alpha1
    kind: BackupStorage
    metadata:
      name: s3-prod
      namespace: team-a
    spec:
      backupStorageClassRef:
        name: s3                 # immutable
      location:                  # immutable: engines treat it as their backup catalog
        objectStore:
          bucket: my-backups
          prefix: team-a         # optional, no leading/trailing slash
      parameters:                # validated by the class's parametersSchema
        region: us-east-1
        endpointURL: https://s3.us-east-1.amazonaws.com   # an http:// URL means plain HTTP
        forcePathStyle: false
        implementation: aws      # aws | minio | ceph | gcs | other: which software serves the endpoint
      authentication:
        mode: AccessKey
        credentialsSecretRef:
          name: backup-storage-s3-prod-credentials
      tls:                       # optional
        ca:
          secretKeyRef: {name: backup-storage-s3-prod-ca, key: ca.crt}
        # or: insecureSkipVerify: true   (mutually exclusive with ca)
    ```

3. Check that the backup storage is accepted:

    ```sh
    kubectl get backupstorages -n team-a
    ```

    ??? example "Expected output"

        ```{.text .no-copy}
        NAME      TYPE   BUCKET       AUTH        ACCEPTED   AGE
        s3-prod   s3     my-backups   AccessKey   Accepted   12s
        ```

The fields of a `BackupStorage` are:

| Field | Description |
| ----- | ----------- |
| `spec.backupStorageClassRef.name` | The backup storage class. Can't be changed. |
| `spec.location.objectStore.bucket` | The bucket. Can't be changed. |
| `spec.location.objectStore.prefix` | Optional path within the bucket to write backups under, without a leading or trailing slash and without `.` or `..` segments. Can't be changed. |
| `spec.parameters` | Connection parameters, validated by the class's `parametersSchema`. For the `s3` class: `region` and `endpointURL` (required), `forcePathStyle`, and `implementation` (`aws`, `minio`, `ceph`, `gcs` or `other`; unset means `aws`). |
| `spec.authentication.mode` | One of the class's authentication modes, `AccessKey` for `s3`. |
| `spec.authentication.credentialsSecretRef.name` | The Secret, in the same namespace, holding the keys the authentication mode requires. |
| `spec.tls` | Optional. How the certificate of the store is verified: `ca.secretKeyRef` names a Secret key holding a PEM CA bundle, and `insecureSkipVerify: true` turns verification off. The two are mutually exclusive. Without `spec.tls`, the system trust store is used. |

To write backups to another bucket or prefix, create a new backup storage, point your instances at it, and delete the old one.

!!! note alert alert-primary "Permissions"
    When you create or change a backup storage through the OpenEverest API, as the UI and `everestctl` do, you need `read` on its backup storage class and on every Secret it names. The API server doesn't create Secrets for backup storages: the UI and `everestctl` create them through the Secrets API first. See [Role-based access control](../administer/rbac.md#backup-storages-and-secrets).

## Status

The `Accepted` condition reports whether the backup storage is valid for its storage class. It checks the location, the parameters, the authentication mode and the Secrets, not whether the store is reachable.

| Reason | Meaning |
| ------ | ------- |
| `Accepted` | The backup storage is valid for its class. |
| `ClassNotFound` | The backup storage class doesn't exist. |
| `InvalidLocation` | The bucket or the prefix breaks a naming rule of the technology. |
| `InvalidParameters` | The class's `parametersSchema` rejects `spec.parameters`. |
| `AuthModeUnsupported` | The class has no authentication mode with that name. |
| `CredentialsNotFound` | The credentials Secret, or a key the authentication mode requires, is missing. |
| `InvalidCABundle` | The CA Secret or key is missing, or holds no PEM certificate. |

A backup class can write to the backup storage when its `spec.backupStorageClasses` lists the storage's class with the storage's authentication mode. To list the backup storages a backup class can write to, run `everestctl backup-storage list --backup-class <name>`.

A deleted backup storage stays until no instance, backup, restore or backup import references it. Meanwhile the `Deleting` condition is `True` with the reason `WaitingForReferrers`, and its message names the referrers.

## Use the backup storage in an instance

An instance names its backup storages in `spec.backup.storages`, each with its own schedules and point-in-time recovery (PITR) settings:

```yaml
apiVersion: core.openeverest.io/v1alpha1
kind: Instance
metadata:
  name: orders-db
  namespace: team-a
spec:
  # ...
  backup:
    enabled: true
    classRef:
      name: percona-backup-mongodb
    storages:
      - storageRef:
          name: s3-prod
        schedules:
          - name: daily
            enabled: true
            cron: "0 0 * * *"
        pitr:
          enabled: true
```

The backup storage must be in the instance's namespace, and the instance's backup class must be able to write to it.

## How backups are laid out in the bucket

Database engines write each instance's backups under its own path in the bucket:

```text
<prefix>/backups/<namespace>/<instance>/<instance-uid>/<storage>
```

Many instances and namespaces can therefore share one bucket, and an instance that is deleted and recreated with the same name never writes into its predecessor's backups. Every backup sits under `<prefix>/backups/`, so a single lifecycle or replication rule on that path covers them all.

### Instance status

Each instance reports where it writes, and any problem with a backup storage entry, in `status.backup.storages`:

```yaml
status:
  backup:
    storages:
      - name: s3-prod
        location: team-a/backups/team-a/orders-db/0b9f6a2e-4c1d-4f7e-9a51-2d8c3e7b6f10/s3-prod
```

When an entry can't be used, it carries a `reason` and a `message`:

| Reason | Meaning |
| ------ | ------- |
| `BackupStorageNotFound` | The backup storage doesn't exist in the instance's namespace. |
| `BackupStorageNotAccepted` | The backup storage isn't accepted by its storage class. |
| `BackupStorageIncompatible` | The instance's backup class can't write to the backup storage. |
| `UnsupportedParameter` | The engine doesn't know or can't honour a parameter of the backup storage. |
| `ImplementationUnsupported` | The engine doesn't support the backup storage's `implementation`. |
| `TLSOptionUnsupported` | The engine can't honour the backup storage's TLS settings. |
| `BackupStorageLocationChanged` | The path the entry resolves to differs from the recorded `location`, for example because the backup storage was recreated with another bucket or prefix. |

An entry that stops being usable keeps its last working configuration on the engine, so a mistake in an edit doesn't interrupt backups that were already running.

## Engine support

Each database engine maps the backup storage to its own backup tool, and not every tool supports every implementation or TLS option:

| Engine | `implementation` | Custom CA | Skip verification |
| ------ | ---------------- | --------- | ----------------- |
| Percona Server for MongoDB | `aws` uses PBM's `s3` storage type. `minio`, `ceph` and `other`, and `aws` with a custom CA, use PBM's `minio` type. `gcs` isn't supported. | Yes | Yes |
| Percona XtraDB Cluster | Every value is treated as plain S3. | Yes | Yes |
| PostgreSQL (pgBackRest) | `gcs` isn't supported. Plain `http://` endpoints aren't supported. | Yes | Yes |
| Cassandra (Medusa) | All | No | Yes |
| Milvus | All | No | No: TLS is always verified against the system trust store. |

When an engine can't use a backup storage, the instance reports it in `status.backup.storages` with the `ImplementationUnsupported`, `UnsupportedParameter` or `TLSOptionUnsupported` reason.

## Upgrading from an earlier developer preview

In OpenEverest 2.0.0 Developer Preview 4 and earlier, a `BackupStorage` had `spec.type: s3` and its settings under `spec.s3`. There is no conversion to the new shape: developer previews are installed fresh, so recreate your backup storages.
{.power-number}

1. Before you upgrade, save your backup storages:

    ```sh
    kubectl get backupstorages -A -o yaml > backupstorages-old.yaml
    ```

2. After the upgrade, delete each old backup storage:

    ```sh
    kubectl delete backupstorage s3-prod -n team-a
    ```

    The credentials Secret it used isn't deleted. It already holds the `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` keys, so the new backup storage can reuse it by name.

3. Create the new backup storage, mapping the old fields as follows, and apply it with `kubectl apply -f`:

    | Old field | New field |
    | --------- | --------- |
    | `spec.type: s3` | `spec.backupStorageClassRef.name: s3` |
    | `spec.s3.bucket` | `spec.location.objectStore.bucket` |
    | `spec.s3.region` | `spec.parameters.region` |
    | `spec.s3.endpointURL` | `spec.parameters.endpointURL` |
    | `spec.s3.forcePathStyle` | `spec.parameters.forcePathStyle` |
    | `spec.s3.verifyTLS: false` | `spec.tls.insecureSkipVerify: true` |
    | `spec.s3.credentialsSecretRef.name` | `spec.authentication.credentialsSecretRef.name`, with `spec.authentication.mode: AccessKey` |

    ```yaml
    apiVersion: backup.openeverest.io/v1alpha1
    kind: BackupStorage
    metadata:
      name: s3-prod
      namespace: team-a
    spec:
      backupStorageClassRef:
        name: s3
      location:
        objectStore:
          bucket: my-backups
      parameters:
        region: us-east-1
        endpointURL: https://s3.us-east-1.amazonaws.com
      authentication:
        mode: AccessKey
        credentialsSecretRef:
          name: <old credentials Secret>
    ```

New backups are written under the [bucket layout](#how-backups-are-laid-out-in-the-bucket) described above. Backups written by an earlier developer preview stay where they are.

## Manual backups and restores

In addition to scheduled backups, you can create manual backups and perform restores using the `DatabaseClusterBackup` and `DatabaseClusterRestore` CRDs.


### Create a manual backup

Define a `DatabaseClusterBackup` object:


```sh
apiVersion: everest.percona.com/v1alpha1
kind: DatabaseClusterBackup
metadata:
  labels:
    clusterName: my-database-cluster
  name: my-database-cluster-backup
spec:
  backupStorageName: my-s3-backup-storage
  dbClusterName: my-database-cluster
```

Monitor the backup status:

```sh
kubectl get dbbackup manual-backup-2024-04-11 -o jsonpath='{.status}'
```

A backup typically progresses through these states:

- **Starting**: Initial backup preparation

- **Running**: Backup in progress

- **Succeeded**: Backup completed successfully

- **Failed**: Backup failed


### Restore from a backup

You can restore from a backup in two ways:
{.power-number} 

1. Restore from a `DatabaseClusterBackup`.

    ```sh
    apiVersion: everest.percona.com/v1alpha1
    kind: DatabaseClusterRestore
    metadata:
      name: restore-from-backup
    spec:
      dbClusterName: my-database-cluster
      dataSource:
        dbClusterBackupName: my-database-cluster-backup
    ```

2. Restore to a **new database cluster**.

    To restore a backup to a new database cluster, create a new `DatabaseCluster` with the `dataSource` field that references the backup:

    ```sh
    apiVersion: everest.percona.com/v1alpha1
    kind: DatabaseCluster
    metadata:
      name: restored-database
      namespace: default
    spec:
      # .. hidden
      dataSource:
        dbClusterBackupName: backup-name-here  # Name of the backup to restore from
    ```

!!! info "Important"
    - The new cluster must use the same type and version of database engine as the backup. 
    - The storage size of the new cluster should be at least as large as that of the original cluster. 
    - The new cluster will be created in the same namespace as the backup. 
    - Other configurations, such as proxy settings and monitoring, may differ from those of the original cluster.


### Perform Point-in-Time Recovery (PITR)

You can perform a point-in-time recovery to restore your database to a specific moment in time.

```sh
apiVersion: everest.percona.com/v1alpha1
kind: DatabaseClusterRestore
metadata:
  name: pitr-restore
spec:
  dbClusterName: my-database
  dataSource:
    dbClusterBackupName: base-backup
    pitr:
      type: date
      date: "2024-04-11T15:30:00Z"  # UTC timestamp
```

Monitor the restore status:

```sh
kubectl get dbrestore restore-from-backup -o jsonpath='{.status}'
```

The restore will typically go through these states:

- **Starting**: Initial restore preparation
- **Restoring**: Restore in progress
- **Succeeded**: Restore completed successfully
- **Failed**: Restore failed

