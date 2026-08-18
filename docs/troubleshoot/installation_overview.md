# OpenEverest installation and workflow

This page provides an overview of how OpenEverest is installed, the components involved, and the workflow, from operator installation to database provisioning, backups, and restores.


## Installation workflow

Starting with OpenEverest v1.4.0, the [everestctl](../install/installEverest.md) is a wrapper around two helm charts:

- [everest-core](https://github.com/openeverest/helm-charts/tree/v1.x/charts/everest){:target="_blank"} 

- [everest-db-namespace](https://github.com/openeverest/helm-charts/tree/v1.x/charts/everest/charts/everest-db-namespace){:target="_blank"}. 

The installation flow is as follows:
{.power-number}

1. The `everest-core` helm chart is installed in the `everest-system` namespace:

    This step deploys all the core components required for OpenEverest to function.

    | Component category   | Namespace            | Components                                                  |
    |----------------------|----------------------|-------------------------------------------------------------|
    | OLM components       | `everest-olm`        | `olm-operator`, `catalog-operator`, `package-server`, `everest-catalog` |
    | Monitoring stack     | `everest-monitoring` | `vm-operator`, `kube-state-metrics`                         |
    | Everest components   | `everest-system`     | `everest-server`, `everest-operator`                        |


2. The `everest-db-namespace` chart is installed in the `everest` namespace. This chart handles the deployment of database operators using the following workflow:

    1. Creates OLM subscriptions for the selected database operators (by default, all supported operators are selected).
    2. OLM reconciles the subscriptions and creates an install plan for each operator.
    3. A Helm hook creates the `everest-operators-installer` job that waits for the OLM install plan to be created and approves the plan once created.
    4. OLM reconciles the approved install plan, creates a `cluster-service-version` (CSV) for each database operator, and applies the manifests from the CSV to install the operator. The CSV includes all the manifests needed to install the operator.


### Configurable installation options 

OpenEverest supports several configuration flags for customizing the installation.

**Namespace configuration**

| **Flag**                  | **Description**                                     |
| --------------------- | ----------------------------------------------- |
| `--namespaces everest`  | Uses the default `everest` namespace and installs all supported operators (PXC, PSMDB, and PG).|
| `--skip-db-namespace` | Skips creation of database namespaces.           |

??? example "Example"
    You can provision multiple database namespaces simultaneously by providing a comma-separated list with the `--namespaces` flag.

    ```sh
    everestctl install --namespaces n1,n2
    ```


**Operator installation flags**

If you want to skip the installation of certain operators, you can use the following flags:

| **Flag** | **Description**|
| --------------------------------- | ------------------------------------ |
| `--operator.mongodb=false`        | Skip installing the MongoDB operator.    |
| `--operator.postgresql=false`     | Skip installing PostgreSQL operator. |
| `--operator.mysql=false`          | Skip installing PXC operator.        |

### Manage namespaces

You can manage database namespaces after the initial installation using the following commands:

```sh
everestctl namespaces add [NAMESPACE]
everestctl namespaces update [NAMESPACE]
everestctl namespaces remove [NAMESPACE]
```

For detailed information on managing namespaces, see the [Namespaces management](https://github.com/openeverest/helm-charts/tree/v1.x/charts/everest#4-deploy-additional-database-namespaces){:target="_blank"} section.

The [helm installation method](../install/install_everest_helm_charts.md) provides an identical flow to the one described above, with similar configuration options. Refer to the [helm chart documentation](https://github.com/openeverest/helm-charts/tree/v1.x/charts/everest){:target="_blank"} for a complete list of available [configuration options](https://github.com/openeverest/helm-charts/tree/v1.x/charts/everest#configuration){:target="_blank"}.

## Server and operator workflow

### Database creation workflow

Here's the database creation workflow in OpenEverest:
{.power-number}

1. The OpenEverest user is authenticated and logged in, and a `JWT Token` is provided.
2. The user creates a database via the OpenEverest UI or API.
3. The OpenEverest API is invoked to create a new database. The OpenEverest server then creates a `DatabaseCluster` custom resource on the Kubernetes cluster.
4. In the `everest-operator` reconciliation loop, once the `DatabaseCluster` object is recognized, an appropriate custom resource for the database is created, for example, if it’s MySQL, `PerconaXtraDBCluster` is created.
5. The database operator takes over the task of creating the database and the necessary objects to manage it.


    !!! note
        Similar workflows apply to **backups, restores, monitoring, and deletions,** although the object types may vary.
                
        All custom resources in OpenEverest follow this flow, **except for** the `DatabaseEngine`.

### Database engine workflow

Here’s the workflow for the database engine in OpenEverest:
{.power-number}

1. You can install the `everest-db-namespace` chart either as part of the initial installation or as a [separate step](https://github.com/openeverest/helm-charts/tree/v1.x/charts/everest#4-deploy-additional-database-namespaces).
2. OLM subscriptions are created for the operators chosen while installing the Helm chart.
3. OLM **reconciles the Subscriptions** and creates an **InstallPlan**.
4. The Helm chart creates a Kubernetes job called `everest-operators-installer` that waits for the `InstallPlan` to be created and approves it.
5. OLM detects that the **InstallPlan** has been approved and creates a `ClusterServiceVersion`, deploying all components that comprise  the database operator.
6. The OpenEverest operator detects the database operator's deployment resource and reconciles the **DatabaseEngine CR** of the corresponding type. 

7. During reconciliation, the OpenEverest operator detects the installed database operator version and queries [Percona’s Version Service](https://github.com/Percona-Lab/percona-version-service){:target="_blank"} to fetch supported engine versions.
 
    ??? example "Example"
        Check [Percona's Version Service](http://check.percona.com/){:target="_blank"} to get the engine versions supported by that operator. 
        
        To check which MongoDB versions are supported by PSMDB operator v1.19.0, visit:
        
        🔗 [Supported MongoDB versions – PSMDB Operator v1.19.0](https://github.com/Percona-Lab/percona-version-service/blob/09867dc07b553e452df2330e50185d98b68ed90a/sources/operator.1.19.0.psmdb-operator.json#L7-L73){:target="_blank"}.
