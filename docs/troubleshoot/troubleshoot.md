# OpenEverest: Troubleshooting and diagnostics

This page is your go-to resource for tackling common issues and finding solutions. 

For limitations, see the [known limitations](../reference/known_limitations.md) section.


## General troubleshooting guidelines

Before troubleshooting, it's important to understand how OpenEverest works at a high level. Understanding how it works can help narrow down potential areas to investigate. You can refer to the [Kubernetes concepts guide](https://kubernetes.io/docs/concepts/)
 for background. This context will help you interpret Everest’s behavior when debugging.

- Refer to the [kubectl quick reference](https://kubernetes.io/docs/reference/kubectl/quick-reference/){:target="_blank"} for commonly used commands to inspect and troubleshoot Kubernetes resources when working with OpenEverest.

- Use logs and events for debugging.


### Key logs and commands

You can review different logs for additional information depending on the specific issue.
{.power-number}

1. **Everest Core Components**

    | Logs        | Command                                                     |
    | ---------------- | ----------------------------------------------------------- |
    | OpenEverest operator| `kubectl logs -f deploy/everest-operator -n everest-system` |
    | OpenEverest server | `kubectl logs -f deploy/everest-server -n everest-system`   |

2. **Database Operators (in Namespaces)**

    | Operator type | Namespace | Command                                                             |
    | ------------- | --------- | ------------------------------------------------------------------- |
    | PostgreSQL    | `everest` | `kubectl logs -f deploy/percona-postgresql-operator -n everest`     |
    | MongoDB       | `everest` | `kubectl logs -f deploy/percona-server-mongodb-operator -n everest` |
    | PXC           | `everest` | `kubectl logs -f deploy/percona-xtradb-cluster-operator -n everest` |

    !!! note
        Update the namespace (`-n <namespace>`) in your commands if your database clusters are deployed in a namespace other than the default one.

3. **Monitoring**

    | Component                | Command                                                                   |
    | ------------------------ | ------------------------------------------------------------------------- |
    | VictoriaMetrics Operator | `kubectl logs -f deploy/vm-operator -n everest-monitoring`                |
    | VM Agent                 | `kubectl logs -f deploy/vmagent-everest-monitoring -n everest-monitoring` |


4. **Database and Proxy Pods**

    ```sh
    kubectl logs -f <pod-name of database or proxy> -c <database-container>
    ```

5. **Kubernetes Events**

    !!! info "Important"
        Events (Events are stored only for 60 mins; if there are any events older than 60 minutes, they will be lost).

    ```sh
    kubectl get events --sort-by=".lastTimestamp"
    ```

## Troubleshooting key areas


### Installation issues

For troubleshooting OpenEverest installation issues using **everestctl** or the **Helm chart**, the following steps may be helpful:
{.power-number}

1. **Permissions and privileges**

    Installing database operators and their dependencies may require appropriate privileges. 

     - OpenEverest installation may require `cluster-admin` privileges for:
        - **Operator Lifecycle Manager (OLM)**
        - **CustomResourceDefinitions (CRDs)**

    If you encounter failures during installation, ensure your user account has the appropriate permissions. 

    !!! note
	    Application teams frequently encounter problems when installing operators due to **insufficient privileges**.

    Run the following command to check if the required privileges are granted:

    ```sh
    kubectl auth can-i <verb> <resource> --namespace <namespace>
    ```

    To verify if you can create CRDs, run the following command:

    ```
    kubectl auth can-i create crd
    ```

    Ensure that your cluster administrator grants the required privileges before you **retry** the installation.


2. **Helm Chart validation**

    - Verify the installation status of the Helm chart. A properly functioning chart should be in a **Deployed** status. 

        To verify the values used during the chart installation, run the following command:

        ```sh
        helm list -n everest-system
        NAME            NAMESPACE         REVISION    UPDATED                                 STATUS      CHART            APP VERSION
        everest-core    everest-system    1           2025-01-16 16:24:56.577713 +0530 IST    deployed    everest-1.4.0    1.4.0
        ```

    - The OpenEverest installation has many components, so it will fail if any **subcomponent installations fail**. Check the relevant namespace where components are installed, along with the logs and events.

3. **Resource availability**

    When a job is created to approve the installation plan for operators, if the cluster has no available resources to run pods, the Helm installation will wait for the specified `--timeout` or the default 5 minutes before failing. 
    
    In such cases, check if any pod is stuck in the **Pending** state due to insufficient resources:

    ```sh
    kubectl get pods -A --field-selector=status.phase=Pending
    ```

    !!! note
        This behavior is not confined to jobs alone. If the cluster lacks sufficient resources, any component may remain in a **Pending** state.

4. **Local kind setup initialization timing**

    When using local Kubernetes environments such as `kind`, Helm installation may initially time out while cluster components are still stabilizing. This behavior may be more noticeable in local environments such as WSL2.

    Example:

    ```sh
    Error: INSTALLATION FAILED: failed post-install:
    timed out waiting for the condition
    ```

    During startup, `everest-server` may temporarily restart if Kubernetes API connectivity is not yet available. This does not always indicate an installation failure.

    Before retrying the installation, verify whether OpenEverest components are still progressing:

    ```sh
    kubectl get pods -n everest-system
    ```

    Review logs if required:

    ```sh
    kubectl logs -f deploy/everest-server -n everest-system
    ```

    !!! note
        Temporary pod restarts during local kind setup may occur while Kubernetes components initialize. Allow the cluster to stabilize before reinstalling OpenEverest.

For detailed information on the installation process, see [Installation overview](../troubleshoot/installation_overview.md)


### UI, API and authorization issues

To troubleshoot issues with the OpenEverest UI, API, or authorization, check the everest-server deployment. 
{.power-number}

1. **Check everest-server Pod health**

    If the OpenEverest API is not working, check the status of the everest-server pod, specifically its **Status** and **Restarts**.

    ```sh
    kubectl get po -l app.kubernetes.io/name=everest-server -n everest-system

    NAME                            READY STATUS  RESTARTS AGE
    everest-server-78699679d4-kgqk5 1/1   Running  0       4d23h
    ```

2. **Check `everest-server` logs**

    ```sh
    kubectl logs -f deploy/everest-server -n everest-system
    ```

3. **RBAC validation**

    To resolve authorization and access issues, check the OpenEverest server logs. If [Role-Based Access Control (RBAC)](../administer/rbac.md) is enabled, [validate](../administer/administer/rbac.md#validate-your-rbac-policy) or [check the permissions](../administer/rbac.md#test-your-rbac-policy) using `everestctl`.

    ```sh
    kubectl get configmap everest-rbac -n everest-system
    ```

5. **Local access via Port Forwarding**

    If you experience any access issues or lag in the OpenEverest UI or API, try port-forwarding to the service and check the latency compared to accessing it via a **LoadBalancer** or **NodePort**. 

    ```sh
    kubectl port-forward svc/everest 8080:8080
    ```
    Once you have set up the port-forward, access the webpage using [http://localhost:8080](http://localhost:8080).

### Database operation issues

Here are the common issues related to the database operations:
{.power-number}

1. **Check the `everest-operator` logs**

    Check the `everest-operator` logs if the `DatabaseCluster` object has not been created or if there are any issues. 

    ```sh
    kubectl logs -f deploy/everest-operator -n everest-system
    ```

2. **Check the `DatabaseCluster` object**

    Verify the status of the `DatabaseCluster` object and events. The status should be **Ready**. If it is anything other than **Ready**, further investigation is required. Describing the `DatabaseCluster` object provides details about the database configuration.

    ```sh
    kubectl get DatabaseCluster <DatabaseCluster-Name>
    kubectl describe DatabaseCluster <DatabaseCluster-Name>
    kubectl get DatabaseCluster <DatabaseCluster-Name> -oyaml
    ```

3. **Database objects**

    Check the operator logs and the relevant database objects, such as PXC, PSMDB, and PG. For instance, check the PXC object followed by the operator logs.

    ```sh
    kubectl get pxc <database-name>
    kubectl describe pxc <database-name>
    kubectl get pxc <database-name> -oyaml 
    kubectl logs -f deploy/percona-xtradb-cluster-operator
    
    # Change to pxc, psmdb, pg for respective database
    ```

4. **Check the database pod logs**

    ```sh
    kubectl logs -f <database-pod-name> -c <database-container-name>  
    
    # (container name could be database, pxc, mongo)
    ```




























