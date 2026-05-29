# OpenEverest quick install guide

!!! warning "Developer Preview"
    This is a **developer preview** release (v2.0.0-dev.1). Features are incomplete and subject to change. The `everestctl` installation method is not available for this release.

With this guide, you'll be up and running with OpenEverest in no time. For more details, see the comprehensive [installation guide](install/install_everest_helm_charts.md).

## Prerequisites

Before getting started with OpenEverest, do the following:
{.power-number}

1. Install [Helm v3  :octicons-link-external-16:](https://docs.helm.sh/using_helm/#installing-helm){:target="_blank"}.

2. Set up a Kubernetes cluster.
     
    !!! note alert alert-primary "Note"

        OpenEverest assists with installing all the necessary operators and required packages, but does not deploy a Kubernetes cluster.

       We recommend setting up OpenEverest on the Amazon Elastic Kubernetes Service (EKS) or Google Kubernetes Engine (GKE).
         
      [Create EKS cluster :material-arrow-right:](install/eks.md){.md-button}  [Create GKE cluster :material-arrow-right:](install/gke.md){.md-button}

3. Verify that you have access to the Kubernetes cluster that you want to use with OpenEverest. By default, OpenEverest uses the kubeconfig file available under `~/.kube/config`. 

     If your file is located elsewhere, use the export command below to set the `KUBECONFIG` environment variable:
    
     ```sh
     export KUBECONFIG=~/.kube/config
     ```

     To verify access to the Kubernetes cluster, run the following command:
   
     ```sh
     kubectl get nodes
     ```

    ??? example "Expected output"
        
        ```{.text .no-copy}
        NAME                                    STATUS   ROLES    AGE   VERSION
        gke-<name>-default-pool-75d48bfc-bx8g   Ready    <none>   11h   v1.26.7-gke.500
        gke-<name>-default-pool-75d48bfc-c2df   Ready    <none>   11h   v1.26.7-gke.500
        gke-<name>-default-pool-75d48bfc-zl7k   Ready    <none>   11h   v1.26.7-gke.500
        ```


## Install OpenEverest

To install OpenEverest using Helm follow these steps:
{.power-number}

1. Add the OpenEverest Helm repository and install the core platform.

    ```sh
    helm repo add openeverest https://openeverest.github.io/helm-charts/
    helm repo update
    helm install everest-core openeverest/openeverest \
      --devel \
      --version "2.0.0-dev.1" \
      --namespace everest-system \
      --create-namespace
    ```

2. Install the MongoDB Provider.

    ```sh
    helm repo add provider-percona-server-mongodb https://openeverest.github.io/provider-percona-server-mongodb/
    helm repo update
    helm install provider-percona-server-mongodb provider-percona-server-mongodb/provider-percona-server-mongodb \
      --namespace everest-system
    ```

    !!! note
        Additional providers will be available in future releases. See [Providers](extend/providers.md) for more details.


    ??? info " 🌐 Install OpenEverest and access it using Ingress"

        **Prerequisite**

        - An Ingress controller (e.g., Nginx) installed on your Kubernetes cluster

        - If TLS is required on your Ingress endpoint, a **Secret** containing the TLS certificates

        **Example**

        To install OpenEverest and access using [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/#what-is-ingress){:target="_blank"}, here are the steps:
        {.power-number}


        1. Install OpenEverest:

            ```sh
            helm install everest openeverest/openeverest \
              -n everest-system \
              --set ingress.enabled=true \
              --set ingress.ingressClassName="" \
              --set ingress.hosts[0].host=everest.example.com \
              --set ingress.hosts[0].paths[0].path=/ \
              --set  ingress.hosts[0].paths[0].pathType=ImplementationSpecific
            ```
            Replace `everest.example.com` with your own domain.
             
        3. Verify Ingress:

            ```sh
            kubectl get ingress -n everest-system
            ```

            Make sure the address provided is valid and that it correctly routes to the `everest` service.

        ??? example "Example: Custom YAML configuration file"


            ```sh
            ingress:
            # -- Enable ingress for Everest server
            enabled: true
            # -- Ingress class name. This is used to specify which ingress controller should handle this ingress.
            ingressClassName: "nginx"
            # -- Additional annotations for the ingress resource.
            annotations: {}
            # -- List of hosts and their paths for the ingress resource.
            hosts:
                - host: everest.example.com
                  paths:
                    - path: /
                      pathType: ImplementationSpecific
            # -- TLS configuration for the ingress resource.
            # -- Each entry in the list specifies a TLS certificate and the hosts it applies to.
            tls: []
            #  - secretName: everest-tls
            #    hosts:
            #      - everest.example.com
            ```
            Install OpenEverest using this file:

            ```sh
            helm install everest openeverest/openeverest \
            -n everest-system \
            -f everest-values.yaml
            ```

    ??? info "🔒 Install OpenEverest with TLS enabled"

        Install OpenEverest with TLS enabled:

        ```sh
        helm install everest openeverest/openeverest \
        --namespace everest-system \
        --create-namespace
        --set server.tls.enabled=true
        ```


        For comprehensive instructions on enabling TLS for OpenEverest, see the section [TLS setup with OpenEverest](../security/tls_setup.md#tls-setup-with-percona-everest).

    Once OpenEverest is running successfully, you can install additional Providers from the [Providers](extend/providers.md) page.

    !!! note
        - If you installed OpenEverest using `helm` and need to uninstall it, make sure to uninstall it exclusively through `helm` for seamless removal.

## Post-installation steps

Once you have successfully installed OpenEverest, proceed with the following steps:
{.power-number}

1. Retrieve the `admin` password.

    ```sh
    kubectl get secret everest-accounts -n everest-system -o jsonpath='{.data.users\.yaml}' | base64 --decode  | yq '.admin.passwordHash'
    ```

    !!! note
        The default admin password is stored in plain text. It is highly recommended to update the password after installation.

    For information on user management, see the section [manage users in OpenEverest](administer/manage_users.md).

2.  Access the OpenEverest UI/API using one of the following options, as the `everest` Service is not exposed with an external IP by default:

    === "Load Balancer"

        1. Use the following command to change the Everest service type to `LoadBalancer`:
                                                
            ```sh
            helm install everest openeverest/openeverest \
            --set service.type=LoadBalancer
            ```
            
            ??? example "When TLS is enabled"

                ```sh
                NAME      TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)          AGE
                everest   LoadBalancer   10.43.172.194   34.175.201.246       443:8080/TCP    10s
                ```
                    
        2. Retrieve the external IP address for the `everest` service. This is the address where you can then launch OpenEverest at the end of the installation procedure. In this example, the external IP address used is `http://34.175.201.246`.
                
            ```sh 
            kubectl get svc/everest -n everest-system
            ```
    === "Ingress"

        To access OpenEverest, open your browser and go to: `https://everest.example.com`.

        !!! note
            Replace `everest.example.com` with your own domain.


    === "Node Port"       

        1. Run the following command to change the Everest service type to `NodePort`:

            ```sh
            kubectl patch svc/everest -n everest-system -p '{"spec": {"type": "NodePort"}}'
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

        3. Retrieve the external IP addresses for the kubernetes cluster nodes.

            ??? example "Expected output"
                ```{.text .no-copy}
                NAME                                          STATUS   ROLES    AGE   VERSION               INTERNAL-IP     EXTERNAL-IP      OS-IMAGE                             KERNEL-VERSION CONTAINER-RUNTIME
                gke-everest-test-default-pool-8bbed860-65gx   Ready    <none>   3m35s v1.30.3-gke.1969001   10.204.15.199   34.175.155.135   Container-Optimized OS from Google   6.1.100+       containerd://1.7.19
                gke-everest-test-default-pool-8bbed860-pqzb   Ready    <none>   3m35s v1.30.3-gke.1969001   10.204.15.200   34.175.120.50    Container-Optimized OS from Google   6.1.100+       containerd://1.7.19
                gke-everest-test-default-pool-8bbed860-s0hg   Ready    <none>   3m35s v1.30.3-gke.1969001   10.204.15.201   34.175.201.246   Container-Optimized OS from Google   6.1.100+       containerd://1.7.19
                ```
        
        4. To launch the OpenEverest UI and create your first database cluster, go to the IP address/port found in steps 2 and 3. In this example, the external IP address used is `http://34.175.155.135:32349`. Nevertheless, you have the option to use any node IP specified in the above steps.

    === "Port forwarding"
        The `kubectl port-forward` command in Kubernetes is used to create a temporary connection between your local machine and a specific Kubernetes resource (e.g., a Pod, Service, or Deployment) by forwarding traffic from a local port to a port on the resource. 
        
        1. Run the following command to setup a port-forward to the OpenEverest server service:

                
            ```sh
            kubectl port-forward svc/everest 8080:8080 -n everest-system
            ``` 

            OpenEverest will be available at `http://127.0.0.1:8080`. This method is mostly useful for testing purposes. 

            ??? example "When TLS is enabled"

                ```sh
                kubectl port-forward svc/everest 8443:443 -n everest-system
                ```

                OpenEverest will be available at `https://127.0.0.1:8443`.

## Next steps

[Provision a database :material-arrow-right:](use/db_provision.md){.md-button}

<!-- 
[Monitor the database with PMM :material-arrow-right:](../use/monitorDBs.md){.md-button}
-->
