# Install OpenEverest using Helm

!!! warning "Developer Preview"
    This is a **developer preview** release (v2.0.0-dev.1). Features are incomplete and subject to change. The `everestctl` installation method is not available for this release.

This section explains how to install OpenEverest using [Helm](https://helm.sh/){:target="_blank"}. Helm charts simplify the deployment process by packaging all necessary resources and configurations, making them ideal for automating and managing installations in Kubernetes environments.

!!! info "Important"
    If you installed OpenEverest using Helm, make sure to uninstall it exclusively through Helm for a seamless removal.


## Install OpenEverest

Here are the steps to install OpenEverest:
{.power-number}

1. Add the OpenEverest Helm repository:

    ```sh
    helm repo add openeverest https://openeverest.github.io/helm-charts/
    helm repo update
    ```

2. Install OpenEverest:

    ```sh
    helm install everest-core openeverest/openeverest \
      --devel \
      --version "2.0.0-dev.1" \
      --namespace everest-system \
      --create-namespace
    ```

3. Install the MongoDB Provider:

    ```sh
    helm repo add provider-percona-server-mongodb https://openeverest.github.io/provider-percona-server-mongodb/
    helm repo update
    helm install provider-percona-server-mongodb provider-percona-server-mongodb/provider-percona-server-mongodb \
      --namespace everest-system
    ```

    !!! note
        Additional providers will be available in future releases. See [Providers](../extend/providers.md) for more details.

4. Once the installation is complete, retrieve the `admin` password. 

    ```sh
    kubectl get secret everest-accounts -n everest-system -o jsonpath='{.data.users\.yaml}' | base64 --decode  | yq '.admin.passwordHash'
    ```

    - The default username for logging into the OpenEverest UI is `admin`. You can set a different default admin password by using the `server.initialAdminPassword` parameter during installation.

    - The default `admin` password is stored in plain text. It is highly recommended to update the password after installation.

        To access detailed information on user management, see the [manage users in OpenEverest](../administer/manage_users.md#update-the-password) section.

4. Access the OpenEverest UI/API using one of the following options for exposing it, as OpenEverest is not exposed with an external IP by default:

        
    === "Load Balancer"
        Use the following commands to change the Everest service type to `LoadBalancer`:
        {.power-number}

        1. Run the following command:
                    
            ```sh
            helm upgrade everest-core openeverest/openeverest \
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

                ```sh
                NAME      TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)          AGE
                everest   LoadBalancer   10.43.172.194   34.175.201.246       443:8080/TCP    10s
                ```


    === "Node Port"
        A NodePort is a service that makes a specific port accessible on all nodes within the cluster. It enables external traffic to reach services running within the Kubernetes cluster by assigning a static port to each node's IP address.
        {.power-number}

        1. Run the following command to change the Everest service type to `NodePort`:

            ```sh
            helm upgrade everest-core openeverest/openeverest \
            --namespace everest-system \
            --reuse-values \
            --set server.service.type=NodePort
            ```
            The following output displays the port assigned by Kubernetes to the everest service, which is `32349` in this case.

            ```sh
            kubectl get svc/everest -n everest-system
            NAME      TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
            everest   NodePort   10.43.139.191   <none>        8080:32349/TCP   28m
            ```

            ??? example "When TLS is enabled"
                ```
                kubectl get svc/everest -n everest-system
                NAME      TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
                everest   NodePort   10.43.139.191   <none>        443:32349/TCP   28m
                ```
        
        2. Retrieve the external IP addresses for the kubernetes cluster nodes.

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
        
        3. To launch the OpenEverest UI and create your first database cluster, go to the IP address/port found in step 1 and 3 (if TLS is enabled). In this example, the external IP address used is `http://34.175.155.135:32349`. Nevertheless, you have the option to use any node IP specified in the above steps.

    === "Port Forwarding"

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


## Configure parameters

You can customize various parameters in the OpenEverest Helm charts for your deployment to meet your specific needs. Refer to the [Helm documentation](https://helm.sh/docs/chart_best_practices/values/){:target="_blank"} to discover how to configure these parameters.

A few parameters are listed in the following table. For a detailed list of the parameters, see the [README](https://github.com/openeverest/helm-charts/blob/main/charts/everest/README.md#configuration){:target="_blank"}.


**openeverest/openeverest chart**

|**Key**|**Type**|**Default**|**Description**|
|------|---------|-----------|---------------|
|`server.initialAdminPassword`|string|""|Initial password configured for admin user.</br></br> If it is not set, a random password is generated. It is recommended to reset the admin password after installation.|
|`server.oidc`|object|{}|OIDC configuration for Everest.</br></br> These settings are applied only during installation. To modify the settings after installation, you have to manually update the everest-settings `ConfigMap`.|


## Next steps

[Provision a database :material-arrow-right:](../use/db_provision.md){.md-button}
