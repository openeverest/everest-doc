# Install OpenEverest and expose via Ingress controller

This section explains how to install OpenEverest using [Helm](https://helm.sh/){:target="_blank"} or  `everestctl` and expose OpenEverest using [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/#what-is-ingress){:target="_blank"}.


An Ingress Controller is a Kubernetes component that manages external access to services within a cluster, usually over HTTP and HTTPS. It is responsible for processing Ingress resources, which are rules that define how traffic should be routed to different services within the cluster.

## Prerequisites

- A `kubeconfig` file in the `~/.kube/config` path. If your file is located elsewhere, use the export command below to set the `KUBECONFIG` environment variable:            
        
    ```sh
    export KUBECONFIG=~/.kube/config
    ```

- An [Ingress controller](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/){:target="_blank"} (e.g., Nginx) installed on your Kubernetes cluster

- (Optional but recommended for production) A TLS certificate stored in a Kubernetes Secret.

=== "Install OpenEverest using Helm"

    OpenEverest Helm charts are in the [openeverest/helm-charts](https://github.com/openeverest/helm-charts/tree/v1.x/charts/everest){:target="_blank"} repository on GitHub.

    Here are the steps to install OpenEverest and deploy additional database namespaces:
    {.power-number}

    1. Add the Percona Helm repository:

        ```sh
        helm repo add openeverest https://openeverest.github.io/helm-charts/
        helm repo update
        ```

    2. Install OpenEverest with **Ingress enabled**:

        ```sh
        helm install everest openeverest/openeverest \
          -n everest-system \
          --set ingress.enabled=true \
          --set ingress.ingressClassName="" \
          --set ingress.hosts[0].host=everest.example.com \
          --set ingress.hosts[0].paths[0].path=/ \
          --set  ingress.hosts[0].paths[0].pathType=ImplementationSpecific
        ```  
                
        !!! note
            Replace `everest.example.com` with your own domain.

        ??? info "What's happening under the hood"
            The command does the following:
            {.power-number}

            1. Deploys the OpenEverest components in the `everest-system` namespace. Currently, specifying a different namespace for OpenEverest is not supported.

            2. Deploys a new namespace called `everest` for your databases and the database operators.

                You can override the name of the database namespace by using the `dbNamespace.namespaceOverride` parameter. If you prefer to deploy just the core components, set `dbNamespace.enabled=false`


    3. Verify the Ingress resource:

        ```sh
        kubectl get ingress -n everest-system
        ```

        Ensure the address provided is valid and correctly routes to the `everest` service.

        ??? example "Example: Using a Helm values file"


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
            
            Install OpenEverest using this `YAML` file:

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


    4. Once the installation is complete, retrieve the `admin` password. 

        ```sh
        kubectl get secret everest-accounts -n everest-system -o jsonpath='{.data.users\.yaml}' | base64 --decode  | yq '.admin.passwordHash'
        ```

        - The default username for logging into the OpenEverest UI is `admin`. You can set a different default admin password by using the `server.initialAdminPassword` parameter during installation.

            !!! info "Important"
                The default `admin` password is stored in plain text. It is highly recommended that the password be updated using `everestctl` to ensure that the passwords are hashed. Instructions for installing `everestctl` can be found in [everestctl installation guide](../install/installEverestCLI.html#__tabbed_1_1).

            To access detailed information on user management, see the [manage users in OpenEverest](../administer/manage_users.md#update-the-password) section.

    5. To access the OpenEverest UI/API, open your browser and go to `https://everest.example.com`.

        !!! note
            Replace `everest.example.com` with your own domain.
    

    6. Deploy additional database namespaces:

        Once OpenEverest runs successfully, you can create additional database namespaces using the `everest-db-namespace` Helm chart. 

        If you set `dbNamespaces.enabled=false` in **step 2**, you can deploy a database namespace with the following command:

        ```sh
        helm install everest \
        openeverest/everest-db-namespace \
        --create-namespace \
        --namespace <DB namespace>
        ```

        !!! note
            -  All database operators are installed in your database namespace by default. You can override this by specifying one or more of the following options: `[dbNamespace.pxc=false, dbNamespace.pg=false, dbNamespace.psmdb=false]`.
            - Installation without chart hooks (i.e, the use of `--no-hooks`) is currently not supported.

=== "Install OpenEverest using everesctl"

    !!! info "Important"
        Starting from version 1.4.0, `everestctl` now uses the [Helm chart](https://github.com/openeverest/helm-charts/tree/v1.x/charts/everest){:target="_blank"} to install OpenEverest. To configure chart parameters during installation through `everestctl`, you can:
        
        * Use the `--helm.set` flag to specify individual parameter values.
        * Provide a values file with the `--helm.values` flag for bulk configuration.
        
    To install and provision OpenEverest to Kubernetes:
    {.power-number}
        
    1. Download the latest release of [everestctl](https://github.com/openeverest/openeverest/releases/latest){:target="_blank"} to provision OpenEverest. For detailed installation instructions, see [everestctl installation documentation](../install/install_everestctl.md).
        
        
    2. Install OpenEverest:
        
        ```sh
        everestctl install \
        --helm.set ingress.enabled=true \
        --helm.set ingress.ingressClassName="" \
        --helm.set ingress.hosts[0].host=everest.example.com \
        --helm.set ingress.hosts[0].paths[0].path=/ \
        --helm.set ingress.hosts[0].paths[0].pathType=ImplementationSpecific
        ```

        
        Replace `everest.example.com` with your own domain.

    3. Enter the specific names for the namespaces you want OpenEverest to manage, separating each name with a comma. [These](../use/multi-namespaces.md#default-namespaces-in-openeverest) namespaces are restricted and cannot be used for deploying databases.   
            
    4. Verify Ingress:

        ```sh
        kubectl get ingress -n everest-system
        ```

        Make sure the address provided is valid and that it correctly routes to the `everest` service.

        ??? example "Example: Custom YAML configuration file"

                
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
                    # -- Each entry in the list specifies a TLS certificate and the hosts to which it applies.
                tls: []
                        #  - secretName: everest-tls
                        #    hosts:
                        #      - everest.example.com
        

            Install OpenEverest using this file:

            ```sh
            everestctl install --helm.values everest-values.yaml
            ```

    5. Once the installation is complete, retrieve the `admin` password. 

        ```sh
        everestctl accounts initial-admin-password
        ```

        - The default username for logging into the OpenEverest UI is `admin`. You can set a different default admin password by using the `server.initialAdminPassword` parameter during installation.

        - The default `admin` password is stored in plain text.
        
            !!! info "Important"
                It is highly recommended that the password be updated using `everestctl` to ensure that the passwords are hashed.  Instructions for installing `everestctl` can be found at [everestctl installation guide](../install/installEverestCLI.html#__tabbed_1_1).

            To access detailed information on user management, see the [manage users in OpenEverest](../administer/manage_users.md#update-the-password) section.

    6. To access the OpenEverest UI/API, open your browser and go to  `https://everest.example.com`.

        !!! note
            Replace `everest.example.com` with your own domain.
        
    7.  If you skip adding the namespaces while installing OpenEverest, you can add them later using the following command.
        
        ```sh
        everestctl namespaces add <NAMESPACE>
        ``` 
                

## Next steps

[Provision a database :material-arrow-right:](../use/db_provision.md){.md-button}
