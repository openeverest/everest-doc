# Manage database clusters with CRD

At the core of OpenEverest is the `DatabaseCluster` CRD, which defines and manages database clusters.


??? example "Example: Basic database cluster"
    ```yaml
    apiVersion: everest.percona.com/v1alpha1
    kind: DatabaseCluster
    metadata:
      labels:
        clusterName: my-database-cluster
      name: my-database-cluster
    spec:
      backup:
        pitr:
          enabled: false
      engine:
        replicas: 1
        resources:
          cpu: "1"
          memory: 2G
        storage:
          class: standard-rwo
          size: 25Gi
        type: postgresql # Can be pxc, psmdb, postgresql
        userSecretsName: everest-secrets-my-database-cluster
        version: "17.4"
      monitoring:
        resources: {}
      proxy:
        expose:
          type: ClusterIP
        replicas: 1
        resources:
          cpu: "1"
          memory: 30M
        type: pgbouncer
    ```


    

## Supported database engine types

OpenEverest supports three database engines:

- Percona Distribution for PostgreSQL

- Percona XtraDB Cluster (MySQL)

- Percona Server for MongoDB


### DatabaseEngine CRD

The DatabaseEngine Custom Resource Definition (CRD) manages the lifecycle of database engines in your Kubernetes cluster. 

It handles the following:

- Operator installation
- Upgrades
- Version management


??? example "Example: DatabaseEngine CRD"
    ```
    apiVersion: everest.percona.com/v1alpha1
    kind: DatabaseEngine
    metadata:
      name: percona-postgresql-operator
    spec:
      type: postgresql
    ```

### DatabaseEngine status

The DatabaseEngine status provides the following information:


- Operator state: not installed, installing, installed, upgrading
- Current operator version
- Available database versions and their components
- Pending operator upgrades


Each version can have one of these statuses:

- **recommended**: Preferred version for production use

- **available**: Supported, but not the recommended version

- **unavailable**: Version exists but currently can’t be used

- **unsupported**: Version is no longer supported


To check the available versions and their status:

```
kubectl get dbengine percona-postgresql-operator -n <your namespace> -o jsonpath='{.status.availableVersions}'
```

??? example "Example: DatabaseEngine with both spec and status"
    ```
    apiVersion: everest.percona.com/v1alpha1
    kind: DatabaseEngine
    metadata:
      name: percona-postgresql-operator
      namespace: everest
    spec:
      type: postgresql
    status:
      state: installed
      operatorVersion: 1.4.0
      availableVersions:
        "14.9.0": recommended
        "14.8.0": available
        "13.11.0": available
        "12.14.0": unsupported
        "15.0.0": unavailable
      pendingUpgrades:
        - from: 1.3.0
          to: 1.4.0
    ```

!!! note
    Operator upgrades must be performed through the UI or API, not directly via Kubernetes.


### Pausing a database cluster

Setting `spec.paused: true` stops all engine and proxy pods while keeping storage volumes, credentials, and configuration intact. The cluster can be resumed at any time by setting the field back to `false`.

```yaml
spec:
  paused: true  # Set to false to resume
```

For the full workflow including UI and API methods, see [Pause and resume databases](../use/pause_resume.md).

### Exposing your database

You can expose your database service either through a LoadBalancer or NodePort service type.

```sh
spec:
  proxy:
    type: haproxy  # or: pgbouncer, proxysql, mongos
    expose:
      type: LoadBalancer # or: NodePort
      ipSourceRanges:  # Optional IP whitelist
        - "10.0.0.0/24"
    replicas: 2
```








