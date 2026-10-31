# About backups

## Why it’s important to create backups

Databases (DBs) may get damaged due to a variety of reasons, including disk crashes, power outages, software errors, or even virus attacks.  

To ensure that you can recover data in such events, it is critical to back up your databases. With Everest, you can do this with zero downtime and minimal performance impact.

## Supported setups

Everest enables you to create and restore on-demand, scheduled and Point-in-Time-Recovery (PITR) backups.

## Where backups are stored

Backups are written to a backup storage: a bucket in an object store, optionally with a prefix within it. Each backup storage names a backup storage class, the object storage technology. OpenEverest ships the `s3` class, for Amazon S3 and S3-compatible stores such as MinIO, Ceph RGW and SeaweedFS.

Each instance writes its backups under its own path in the bucket, so many instances and namespaces can share one bucket. An instance can use the backup storages of its namespace that its backup class can write to.

## Start here

[Prepare a backup storage location :material-arrow-right:](../backups_and_restore/createBackups/backup_storage/CreateStorage.md){.md-button}