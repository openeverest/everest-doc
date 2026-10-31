# Manage backup storage

Managing backup storage allows you to edit or remove existing backup storage configurations used for database backups, helping you keep your backup environment organized, secure, and up-to-date.

## Edit backup storage

You can change the connection settings, the authentication and the TLS certificate verification of a backup storage. The name, the namespace, the storage class, the bucket and the prefix can't be changed: database engines treat the location as their backup catalog. To write backups to another bucket or prefix, create a new backup storage, switch your instances to it, and then delete the old one.

To edit a backup storage configuration:
{.power-number}

1. Go to Backup storage settings:

    Navigate to OpenEverest home page and select <i class="uil uil-cog"></i> **Settings > Backup storages**. Then click on the ellipsis (three dots) next to the storage you want to modify.

2. Click **Edit**. The **Edit Backup storage** pop-up will open. **Name**, **Namespace**, **Class**, **Bucket** and **Prefix** are read-only.

3. Make your changes:

      - **Region**, **Endpoint URL**, **Implementation** and **Force path-style URLs**: Update the connection settings.
      - **Credentials**: To rotate the credentials, select **Enter credentials** and enter the new keys. OpenEverest saves them in a new Secret and switches the storage to it. Leave the fields blank to keep the current Secret. To switch to another Secret in the namespace, select **Use an existing Secret**.
      - **Advanced**: Change the **TLS certificate verification**. A new CA bundle is saved in a new Secret too.

4. Click **Edit** to save your configuration.

The Secret the storage used before isn't deleted. Delete it once nothing else uses it.

Instances that use the backup storage pick up the change. If the new settings can't be used by an instance's database engine, for example an implementation the engine doesn't support, the instance keeps its last working configuration and reports the reason in its status. See [Instance status](../../../crd/backup_storage_config.md#instance-status).

### Edit backup storage with everestctl

`everestctl backup-storage update` changes the parameters, the authentication and the TLS settings of a backup storage. Parameters merge key by key: the ones you don't name keep their value.

```sh
# Rotate credentials: point the storage at a Secret holding the new keys
everestctl backup-storage update --name s3-prod -n team-a \
  --credentials-secret s3-prod-credentials-2

# Move to another endpoint and region
everestctl backup-storage update --name s3-prod -n team-a \
  --endpoint-url https://s3.eu-west-1.amazonaws.com --region eu-west-1

# Trust a private CA kept in an existing Secret
everestctl backup-storage update --name minio -n team-a --ca-secret minio-ca:ca.crt
```

The `update` command takes `--parameter key=value` and the `s3` shorthands `--region`, `--endpoint-url`, `--implementation` and `--force-path-style`, plus `--auth-mode`, `--credentials-secret`, `--ca-secret NAME[:KEY]` and `--insecure-skip-tls-verify`. Setting `--ca-secret` clears `--insecure-skip-tls-verify`, and the other way round. The Secret named by `--credentials-secret` or `--ca-secret` must already exist in the namespace.

## View backup storages

The **Backup storages** settings page lists every backup storage with its **Class**, **Bucket** (and prefix), **Auth mode** and **Status**. The status is **Ready** when the backup storage is valid for its storage class, **Deleting** while it waits to be removed, or the reason it isn't accepted.

With `everestctl`:

```sh
everestctl backup-storage list -n team-a
```

`--class` lists only the backup storages of a storage class, and `--backup-class` only those a backup class can write to. `--json` shows the parameters, the TLS settings and the status conditions.

With `kubectl`:

```sh
kubectl get backupstorageclasses
kubectl get backupstorages -n team-a
```

## Delete backup storage

To delete a backup storage:
{.power-number}

1. Go to Backup storage settings:

    Navigate to OpenEverest home page and select <i class="uil uil-cog"></i> **Settings > Backup storages**. Then click on the ellipsis (three dots) next to the storage you want to delete.

2. Click **Delete**. The **Delete storage** confirmation pop-up will open.

3. Click **Delete**. The backup storage will be deleted.

With `everestctl`:

```sh
everestctl backup-storage delete --name s3-prod -n team-a --wait
```

`--wait` blocks until the backup storage is gone and reports what it's waiting for. Add `--yes` to skip the confirmation prompt.

OpenEverest keeps a deleted backup storage until no instance, backup, restore or backup import references it. Meanwhile its status is **Deleting**. The Secrets the backup storage uses aren't deleted with it: delete them once nothing else uses them.

