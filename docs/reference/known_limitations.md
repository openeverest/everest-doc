# Known limitations in OpenEverest

This section describes the known limitations associated with OpenEverest:


## Passwords

Refrain from changing the password of administrative users (e.g., root, monitor, or operator) manually in the database. This action may cause inconsistencies with the secrets stored in Kubernetes, which are crucial for the proper functioning of the cluster. Such modifications have the potential to disrupt your cluster.

We are developing a new feature that will allow you to modify these settings directly from the user interface (UI).


## Split-Horizon DNS

Here are the limitations for OpenEverest:

- Split-Horizon DNS feature is supported only by Percona Server for MongoDB engine. 

- You cannot enable the Split-Horizon DNS feature for existing Percona Server for MongoDB clusters. You can only enable when creating a new Percona Server for MongoDB cluster. 

- You cannot enable Split-Horizon DNS if **Sharding is enabled** for a specific Percona Server for MongoDB cluster. Support for sharded clusters will be included in future releases.

- You cannot disable Split-Horizon DNS feature once you enable it.

- You cannot change the Split-Horizon domains or certificates for an existing cluster.

- Only one Split-Horizon DNS configuration can be applied to a Percona Server for MongoDB cluster.

- OpenEverest does not allow you to manually set custom domain names for each Pod in a ReplicaSet. Instead, you must provide a base domain (e.g., mycompany.com). OpenEverest will automatically generate domain names and TLS certificates for each Pod in the ReplicaSet following this pattern: 

    `<DB cluster name>-rs-0-<pod number>-<namespace>.<base domain>`  
   
    ??? example "Example"  
        - my-db-cluster-rs0-0-default.mycompany.com  
        - my-db-cluster-rs0-1-default.mycompany.com 


- OpenEverest is not responsible for managing DNS configuration (i.e., the resolution of domain names). The OpenEverest Admin must properly configure their DNS server to bind the ReplicaSet's private/public IPs to the generated domain names, allowing external applications to access the ReplicaSet pods via these domain names.

    OpenEverest will provide the Admin with a list of domain names and the corresponding private/public IPs allocated by Kubernetes or the cloud provider.

## Load balancer configuration

- Once annotations are added to a service by the database operators, they cannot be entirely removed.

- After applying a Load Balancer Configuration (LBC), you cannot revert to the `-No Configuration-` option.

- If a service is changed from **LoadBalancer** to **ClusterIP**, the previously applied annotations remain on the service.

- When you apply a new Load Balancer configuration, all existing annotations in the service are replaced by those defined in the selected Load Balancer configuration.



## MongoDB sharding

- Once MongoDB sharding is enabled, it cannot be disabled.

- After you enable sharding for a cluster, you need to take another backup to ensure you can restore. 

## Manual storage scaling

- When manually scaling storage in OpenEverest, resource quotas are not automatically validated during the volume expansion process. If the requested storage exceeds the defined quota, the PVC resize operation will fail, leaving the database in the **Resizing Volumes** state.

    To avoid such issues, ensure you verify your namespace's resource quotas before initiating a resize:

    ```sh
    kubectl describe quota <resource-quota-name> -n <namespace>
    ```

- If scaling fails for any reason, the database will remain in the **Resizing Volumes** state, and the operation will be continuously retried.

- Consult your CSI driver documentation for important details regarding volume resizing restrictions or limitations. This is essential for ensuring smooth operations.

- In previous versions of OpenEverest, users were able to create clusters with decimal storage sizes (for example, 1.2 GiB). However, database operators rounded these values up when provisioning PersistentVolumeClaims (PVCs)—meaning that 1.2 GiB would be provisioned as 2 GiB. OpenEverest would then display the original value in a different unit (such as **1.2 GiB shown as 1288490188800 m**), which caused confusion.

    In version 1.6.0, OpenEverest introduces support for scaling up storage. However, when attempting to scale a cluster to a size that does not exceed the larger rounded PVC size (for instance, trying to scale from 1.2 GiB to 2 GiB, which still rounds to 2 GiB), the cluster may become stuck in the **Resizing Volumes** state. This issue affects MySQL and MongoDB clusters; however, **PostgreSQL** clusters are **not impacted**.

    **Workaround** 

    When scaling storage on clusters created with decimal sizes, make sure the new size exceeds the next whole GiB value (for example, scale from **1.2 GiB to 3 GiB**).

## Databases

- MongoDB 4.4 will no longer be supported, preventing users from upgrading the PSMDB operator if any database is running version 4.4.

- Do not perform a full backup of an empty MongoDB database. Instead, ensure that you add some data to the database before creating a full backup.

- If you attempt to delete a MongoDB cluster stuck in the **initializing** state due to insufficient resources, the cluster will remain in the deleting state indefinitely. This issue is caused by a [bug](https://perconadev.atlassian.net/browse/K8SPSMDB-1208){:target="_blank"} in the PSMDB operator.


- If you attempt to delete a MongoDB and MySQL cluster stuck in the **initializing** state due to insufficient resources, the cluster will remain in the deleting state indefinitely. This issue is caused by a [bug](https://perconadev.atlassian.net/browse/K8SPSMDB-1208){:target="_blank"} in the PSMDB operator.

    **Workaround**

    1. Run the command:

        `kubectl edit psmdb/<DBName> -n <Namespace>`

    2. Delete the finalizer called `delete-pods-in-order`.

- When you delete a PostgreSQL (PG) cluster, the associated backup files will not be automatically removed from S3. Instead, these files will remain in S3 storage. Therefore, you need to manually delete any backup files that you no longer need.


## Upgrading operators

- When you upgrade PostgreSQL operators to version 2.4.1, the database transitions to the **Initializing** state as part of the upgrade process. However, this **Initializing** state does not cause any downtime.
- Since the PostgreSQL operator does not support major version upgrades, you cannot directly upgrade PostgreSQL 12.19 to 13.16. This limitation also prevents upgrading the PG operator itself. Morever, there is no built-in mechanism for transferring data from PostgreSQL 12.19 to 13.16, making migration a challenge.
- When you upgrade PXC operators to version 1.15.0, single node MySQL databases will be restarted, resulting in downtime. However, it is worth noting that single node databases should not be used in production environments.

## Backups storage

The backup storage you choose for your initial backup schedule will be used for all subsequent schedules and point-in-time recovery (PITR).

## On-demand backups

Let's delve into the limitations of on-demand backups in OpenEverest.

### PostgreSQL limitations for on-demand backups

- When attempting to delete a PostgreSQL database that contains backups created with OpenEverest versions older than 1.0.0, the database may become stuck in the **Deleting** state. 

    **Workaround**: To prevent this, manually delete any backups created with versions prior to 1.0.0 by using the **Delete** action on the **Backups** page before deleting the database.

- You cannot change the bucket name and region for a specific backup storage. Doing so will make any backups taken for that bucket unusable.

- You can use any of the existing backup storages across on-demand backups and schedules, as long as the total number of storages in use (by existing on-demand backups and schedules) does not exceed three.

    If you have created two schedules using backup storage `bucket-1` and `bucket-2`, and an on-demand backup using backup storage `bucket-3`, you can only utilize one of these three backup storages to create the next on-demand backup or a schedule.

### MongoDB backup failures

There maybe instances when your MongoDB backups may encounter unexpected failures. 

Let's check the reason for these failures by running the following command:

    kubectl get psmdb-backup <BACKUP_NAME> -n <YOUR_NAMESPACE> -o yaml | grep error

Here are some potential errors you could encounter:

- `starting deadline exceeded`
- `'couldn''t get response from all shards: convergeClusterWithTimeout:`

#### Workarounds for MongoDB backup failures

In the following table, we have compiled a list of workarounds to ensure that your backups function properly again.
{.power-number}


|**Workaround**|**Impact on the cluster**|**Action**|**Procedure**|
|----------|-----------|-------------|--------------------|
| **1**|**No downtime**  |Delete the locks |**1.** Connect to your MongoDB database. </br></br> **2.** Run the command: `db.getSiblingDB("admin").pbmLock.find()` to see the list of database locks. If the list is empty, the scenario is not applicable. </br></br> **3.** If the list was not empty, run the command`db.pbmLock.deleteMany({})`. </br></br> **4.** Run another backup. If the backup still fails, check the workaround 2.|
| **2**|**Shorter downtime**  |Restart config server (sharded clusters only) |**1.** Get the list of config server pods:`kubectl get po -n <YOUR_NAMESPACE> -l app.kubernetes.io/component=cfg,app.kubernetes.io/instance=<YOUR_DB_CLUSTER_NAME>`. </br></br> **2.** For each pod name in the list, run `kubectl delete pod <POD_NAME> -n <YOUR_NAMESPACE>` </br></br> **3.** Wait until your database cluster is up. </br></br> **4.** Run another bakckup. If the backup still fails, check workaround 3.|
| **3**|**Longer downtime**|Restart the DB server |**1.** On OpenEverest UI, click **Actions >> Restart**. </br></br> **2.** When the database cluster is up, take another backup.|


## Scheduled backups

Let's explore the constraints of scheduled backups in OpenEverest.

### PostgreSQL Limitations for schedules

Due to PostgreSQL limitations, the following functionality is unavailable for PostgreSQL:

- Modifying the storage location in existing schedules
- Using the same backup storage for different schedules
- Creating more than three schedules for PostgreSQL
- Using more than three different backup storages in total, including those used in existing on-demand backups.
 
Everest does not allow these actions to be performed because they could corrupt previously taken backups, making it impossible to restore from them.


## Point-in-time-recovery (PITR)

Point-In-Time Recovery (PITR) can only be used after a certain time has elapsed since the latest backup. This interval of time is referred to as the **PITR upload interval**. During this interval, PITR restore won't be available.

The default **uploadInterval** values for different databases are as follows:  

- MySQL = 1 minute
- MongoDB = 10 minutes
- PostgreSQL = 1 minute


### PostgreSQL limitation for PITR

When performing point-in-time recovery (PITR) for PostgreSQL, it is important to consider the following limitations:

- The Edit button for PITR in the UI is always disabled for PostgreSQL. However, PITR is automatically enabled once a backup or backup schedule is created.

- You may encounter issues with point-in-time recovery (PITR) when attempting to recover the database after the last transaction. PITR can get stuck in the **Restoring** state.

**Check the timestamp of the last transaction**

Connect to your database and run the following command:

`select pg_last_committed_xact();`

??? example "Expected output"
    ```
    (768,“2024-03-13 15:52:25.122746+00”,0);
    ```
    It contains the global transaction identifier (`GTID`), `timestamp` and the status of the last transaction.
    
                
!!! warning  
    You can only recover data for the dates prior to this specific date.


**Workaround**

You can follow these steps if your database cluster is stuck in the **Restoring** state:
{.power-number}


1. Check if your database cluster has been stuck because you have used a date after the last transaction:

    a. Find the recovery pod:

        
	    kubectl get pod -n your-namespace
        		
    
    The format of the recovery pod name is `<cluster_name>-pgbackrest-restore-<XYZ>`. The status of the recovery pod should be **Running**.

    b. Check the logs for the recovery pod:
       
        kubectl logs <restore_pod_name> -n your-namespace

    ??? example "Example"
        ```        
        kubectl logs postgresql-kbi-pgbackrest-restore-8b95v -n your-namespace
        ```

    Check whether the log contains the following:
        
        FATAL: recovery ended before configured recovery target was reached
        
    In this case, the cluster is stuck during restoration because you used a date that was after the last transaction.

2. Start an interactive bash shell inside the recovery pod:

    ```
	kubectl -n your-namespace exec <restore_pod_name> -it -- bash
	```

    ??? example "Example"
        ```        
	    kubectl -n your-namespace exec postgresql-kbi-pgbackrest-restore-8b95v -it -- bash
        ```

    Delete the `recovery.signal `file:

    ```sh
	rm pgdata/pg16/recovery.signal
    ```

    After a certain period, the recovery pod will self-destruct. The database cluster status will change from **Restoring** to **Initializing** and eventually to **Up**.


## OIDC integration with Microsoft Entra

When integrating Microsoft Entra ID as your OIDC provider for OpenEverest, it's essential to ensure that the access tokens issued are compatible with OpenEverest's token validation logic.

**Problem**

By default, Microsoft Entra issues access tokens intended for use with Microsoft's own APIs (e.g., Microsoft Graph). These tokens have the following characteristics:

- **Audience (aud):** "`00000003-0000-0000-c000-000000000000`" (Microsoft Graph)

- **Signature:** Uses a proprietary mechanism that cannot be validated by OpenEverest

Microsoft Entra generates access tokens using a proprietary signature mechanism, which OpenEverest cannot validate. This results in signature verification failures when integrating OpenEverest with Entra-generated tokens.

**Solution**

To obtain access tokens that OpenEverest can validate, request tokens explicitly scoped for the registered application in Microsoft Entra using the `<application-client-id>/.default scope`. This ensures:

- **Audience (aud):** Set to your application's client ID

- **Signature:** Standard JWT, verifiable using the issuer's public keys

**Everest Configuration**

When configuring Everest's OIDC settings via `everestctl`, ensure you specify the correct scope:

```sh
everestctl settings oidc configure \
--issuer-url=http://url.com \
--client-id=<your-app-client-id> \
--scopes=openid,profile,email,<your-app-client-id>/.default
```

!!! note
    Note: Replace `<your-app-client-id>` with your actual Microsoft Entra application (client) ID, and ensure the issuer-url points to the correct tenant.

With this configuration, the access token will include `"aud": "<your-app-client-id>"`, and it will have a valid signature that OpenEverest can verify.

