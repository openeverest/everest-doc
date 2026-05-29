# Install OpenEverest using everestctl

!!! warning "Not available in this release"
    The `everestctl` installation method is **not available** in the v2.0.0-dev.1 developer preview. Please use the [Helm-based installation](install_everest_helm_charts.md) instead.

## Before you start

Before running the commands in the **Installation** section, note that OpenEverest will search for the kubeconfig file in the `~/.kube/config` path. If your file is located elsewhere, use the export command below to set the `KUBECONFIG` environment variable: 
    
```sh
export KUBECONFIG=~/.kube/config
```

!!! info "Important"
    If you installed OpenEverest using `everestctl`, make sure to uninstall it exclusively through `everestctl` for a seamless removal.


## Install OpenEverest

!!! info "Important"
    Starting from version 1.4.0, `everestctl` now uses the [Helm chart](https://github.com/openeverest/helm-charts/tree/main/charts/everest){:target="_blank"} to install OpenEverest. To configure chart parameters during installation through `everestctl`, you can:

    * Use the `--helm.set` flag to specify individual parameter values.
    * Provide a values file with the `--helm.values` flag for bulk configuration.

To install and provision OpenEverest to Kubernetes:
{.power-number}

1. Download the latest release of [everestctl](https://github.com/openeverest/openeverest/releases/latest){:target="_blank"} to provision OpenEverest. For detailed installation instructions, see [everestctl installation documentation](../install/install_everestctl.md).

2. You can install OpenEverest using either the wizard or the headless mode.

    !!! note        
        * If you do not specify a namespace, the `everest` namespace gets provisioned by default.
        * You can skip provisioning the database namespace during initial installation by using the flag `--skip-db-namespace`.

            ```
            everestctl install --skip-db-namespace
            ```
            To explore namespaces management in details, refer to the section on [namespace management](../administer/manage_namespaces.md).


    - **Install OpenEverest using the wizard**
        {.power-number}

        1. Run the following command.
            ```sh
            everestctl install
            ```

        2. Enter the specific names for the namespaces you want OpenEverest to manage, separating each name with a comma. [These](../use/multi-namespaces.md#default-namespaces-in-openeverest) namespaces are restricted and cannot be used for deploying databases.

        3.  If you skip adding the namespaces while installing OpenEverest, you can add them later using the following command.

            ```sh
            everestctl namespaces add <NAMESPACE>
            ``` 

    - **Install OpenEverest using the headless mode**
        {.power-number}

        1. Run the following command. You can set multiple namepaces in the headless mode. Replace `<namespace-name>` with the desired name for your namespace.
            ```sh
            everestctl install --namespaces <namespace-name1>,<namespace-name2> --operator.mongodb=true --operator.postgresql=true --operator.mysql=true --skip-wizard
            ```
            
            !!! note
                The flag `--operator.xtradb-cluster` has been **deprecated** and will be removed in the subsequent releases. While it will continue to function for now,  users will receive a warning message asking them to use `--operator.mysql` instead.


            ??? example "Example"
                ```
                everestctl install --namespaces dev,prod --operator.mongodb=true --operator.postgresql=true --operator.mysql=true --skip-wizard
                ```

            **Optional installation flags**
    
            | **Flags**          | **Description**                                                                                      |**Helm flag**                       |
        |------------------|--------------------------------------------------------------------------------------------------|----------------------------------|
        | PMM deployment | Deploy Percona Monitoring and Management (PMM) as a sub-chart. PMM will be automatically deployed within the `everest-system` namespace. | `--helm.set pmm.enabled=true ` |
        | TLS enabled      | Enable TLS encryption for secure communication between OpenEverest components.| `--helm.set server.tls.enabled=true`  |


            ??? example  "Examples"
                Install with PMM enabled  
        

                ```sh
                everestctl install --namespaces <namespace-name1>,<namespace-name2> --operator.mongodb=true --operator.postgresql=true --operator.mysql=true --helm.set pmm.enabled=true --skip-wizard
                ```


                Install OpenEverest with TLS enabled:

                ```sh
                everestctl install --namespaces <namespace-name1>,<namespace-name2> --operator.mongodb=true --operator.postgresql=true --operator.mysql=true --helm.set server.tls.enabled=true --skip-wizard
                ```

                For comprehensive instructions on enabling TLS for OpenEverest, see the section [TLS setup with OpenEverest](../security/tls_setup.md#tls-setup-with-percona-everest).


        2. If you skip adding the namespaces while installing OpenEverest, you can add them later using the following command.

            ```sh
            everestctl namespaces add <NAMESPACE>
            ```

3. Update the password for the `admin` user:

    ```sh
    everestctl accounts set-password --username admin
    ```

    !!! info "Important"
        You can retrieve the automatically generated password by running the `everestctl accounts initial-admin-password` command. However, this password isn't stored securely.

    To access detailed information about user management, see the [Manage users in OpenEverest](../administer/manage_users.md) section.


4. Access the OpenEverest UI/API using one of the following options for exposing it, as OpenEverest is not exposed with an external IP by default:

    === "Load Balancer"

        1. Use the following command to change the Everest service type to `LoadBalancer`:
                    
            ```sh
            helm upgrade everest openeverest/openeverest \
            --namespace everest-system \
            --reuse-values \
            --set server.service.type=LoadBalancer
            ```
                    
        2. Retrieve the external IP address for the `everest` service. This is the address where you can then launch OpenEverest at the end of the installation procedure. In this example, the external IP address used is `http://34.175.201.246`.
                
            ```sh 
            kubectl get svc/everest -n everest-system
            ```
                    
            ??? example "Expected output"
                ```
                NAME      TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)          AGE
                everest   LoadBalancer   10.43.172.194   34.175.201.246       8080:8080/TCP    10s
                ```

            ??? example "When TLS is enabled"
                
                ```
                NAME      TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)          AGE
                everest   LoadBalancer   10.43.172.194   34.175.201.246       443:8080/TCP    10s
                ```

    === "Node Port"
        A `NodePort` is a service that makes a specific port accessible on all nodes within the cluster. It enables external traffic to reach services running within the Kubernetes cluster by assigning a static port to each node's IP address.

        1. Run the following command to change the Everest service type to `NodePort`:

                    
            ```sh
            helm upgrade everest openeverest/openeverest \
            --namespace everest-system \
            --reuse-values \
            --set server.service.type=NodePort
            ```

        2. The following command displays the port assigned by Kubernetes to the everest service, which is `32349` in this case.

            ```sh
            kubectl get svc/everest -n everest-system
            NAME      TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
            everest   NodePort   10.43.139.191   <none>        8080:32349/TCP   28m
            ```
            ??? example "When TLS is enabled"
            
                ```sh
                kubectl get svc/everest -n everest-system
                NAME      TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
                everest   NodePort   10.43.139.191   <none>        443:32349/TCP   28m
                ```            

        4. Retrieve the external IP addresses for the kubernetes cluster nodes.

            ??? example "Expected output"

                ```sh
                kubectl get nodes -o wide
                NAME                   STATUS   ROLES    AGE   VERSION             
                INTERNAL-IPEXTERNAL-IP  OS-IMAGE                        KERNEL-VERSION   
                CONTAINER-RUNTIME
                gke-everest-test-default-pool-8bbed860-65gx   Ready    <none>   3m35s   
                v1.30.3-gke.1969001   10.204.15.199   34.175.155.135   Container- 
                Optimized OS from Google   6.1.100+         containerd://1.7.19
                gke-everest-test-default-pool-8bbed860-pqzb   Ready    <none>   3m35s   
                v1.30.3-gke.1969001   10.204.15.200   34.175.120.50    Container- 
                Optimized OS from Google   6.1.100+         containerd://1.7.19
                gke-everest-test-default-pool-8bbed860-s0hg   Ready    <none>   3m35s   
                v1.30.3-gke.1969001   10.204.15.201   34.175.201.246   Container- 
                Optimized OS from Google   6.1.100+         containerd://1.7.19
                ```
        
        5. To launch the OpenEverest UI and create your first database cluster, go to the IP address/port found in steps 2 and 3. In this example, the external IP address used is `http://34.175.155.135:32349`. Nevertheless, you have the option to use any node IP specified in the above steps.

            !!! note
                If TLS is enabled, the external IP address will be `https://34.175.155.135:32349`.


    === "Port forwarding"

        Run the following command to setup a port-forward to the OpenEverest server service:
                
        ```sh
        kubectl port-forward svc/everest 8080:8080 -n everest-system
        ``` 

        To launch the OpenEverest UI and create your first database cluster, go to your localhost IP address `http://127.0.0.1:8080`.


        ??? example "When TLS is enabled"

            ```sh
            kubectl port-forward svc/everest 8443:443 -n everest-system
            ```

            OpenEverest will be available at `https://127.0.0.1:8443`.                    



## Next steps

[Provision a database :material-arrow-right:](../use/db_provision.md){.md-button}