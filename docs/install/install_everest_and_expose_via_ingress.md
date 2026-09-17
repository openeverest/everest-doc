# Install OpenEverest and expose via Ingress controller

This section explains how to install OpenEverest using [Helm](https://helm.sh/){:target="_blank"} and expose it using an [Ingress controller](https://kubernetes.io/docs/concepts/services-networking/ingress/#what-is-ingress){:target="_blank"}.

An Ingress controller is a Kubernetes component that manages external access to services within a cluster, usually over HTTP and HTTPS. It processes Ingress resources — rules that define how traffic is routed to services in the cluster.

!!! warning "Developer Preview"
    This is a **developer preview** release. Features are incomplete and subject to change. The `everestctl` installation method is not available for this release; install OpenEverest with Helm.

## Prerequisites

- A `kubeconfig` file at `~/.kube/config`. If your file is located elsewhere, set the `KUBECONFIG` environment variable:

    ```sh
    export KUBECONFIG=~/.kube/config
    ```

- An [Ingress controller](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/){:target="_blank"} (for example, NGINX) installed on your cluster.
- (Optional but recommended for production) A TLS certificate stored in a Kubernetes Secret.

## Install OpenEverest
{.power-number}

1. Add the OpenEverest Helm repository:

    ```sh
    helm repo add openeverest https://openeverest.github.io/helm-charts/
    helm repo update
    ```

2. Install the OpenEverest core with **Ingress enabled**:

    ```sh
    helm install everest-core openeverest/openeverest \
      --devel \
      --namespace everest-system \
      --create-namespace \
      --set ingress.enabled=true \
      --set ingress.ingressClassName="nginx" \
      --set ingress.hosts[0].host=everest.example.com \
      --set ingress.hosts[0].paths[0].path=/ \
      --set ingress.hosts[0].paths[0].pathType=ImplementationSpecific
    ```

    !!! note
        Replace `everest.example.com` with your own domain, and `nginx` with your Ingress class name.

    The `--devel` flag installs the latest developer preview. To pin an exact release, add `--version`, for example `--version "2.0.0-dev.3"`.

    ??? info "What's happening under the hood"
        This deploys the OpenEverest core components in the `everest-system` namespace. Specifying a different namespace for the core is not currently supported. Database technologies are added separately by installing Providers (see the next step).

3. Install the MongoDB Provider:

    ```sh
    helm install provider-percona-server-mongodb \
      oci://ghcr.io/openeverest/charts/provider-percona-server-mongodb \
      --namespace everest-system
    ```

    !!! tip "Find more providers and plugins in the Plugin Hub"
        OpenEverest ships with the **Plugin Hub**, an in-product catalog for discovering additional providers and plugins. Open it in the OpenEverest UI at `/plugins/plugin-hub`, or find it in the left-hand menu. Learn more in the [Extension Hub docs](../extend/hub.md), browse the online catalog at [openeverest.io/extensions :octicons-link-external-16:](https://openeverest.io/extensions/), or read the [Plugin Hub introduction blog post](https://openeverest.io/blog/the-hub-introduction/).

4. Verify the Ingress resource:

    ```sh
    kubectl get ingress -n everest-system
    ```

    Ensure the address is valid and routes to the `everest` service.

    ??? example "Example: using a Helm values file"

        ```yaml
        ingress:
          # -- Enable ingress for the Everest server
          enabled: true
          # -- Ingress class name — which ingress controller handles this ingress.
          ingressClassName: "nginx"
          # -- Additional annotations for the ingress resource.
          annotations: {}
          # -- Hosts and their paths for the ingress resource.
          hosts:
            - host: everest.example.com
              paths:
                - path: /
                  pathType: ImplementationSpecific
          # -- TLS configuration for the ingress resource.
          tls: []
          #  - secretName: everest-tls
          #    hosts:
          #      - everest.example.com
        ```

        Install OpenEverest using this file:

        ```sh
        helm install everest-core openeverest/openeverest \
          --devel \
          -n everest-system \
          --create-namespace \
          -f everest-values.yaml
        ```

    ??? info "🔒 Install OpenEverest with TLS enabled"

        ```sh
        helm install everest-core openeverest/openeverest \
          --devel \
          --namespace everest-system \
          --create-namespace \
          --set server.tls.enabled=true
        ```

        For comprehensive instructions on enabling TLS for OpenEverest, see [TLS setup with OpenEverest](../security/tls_setup.md#tls-setup-with-percona-everest).

5. Once the installation is complete, retrieve the `admin` password:

    ```sh
    kubectl get secret everest-accounts -n everest-system -o jsonpath='{.data.users\.yaml}' | base64 --decode | yq '.admin.passwordHash'
    ```

    The default username for the OpenEverest UI is `admin`. You can set a different initial admin password with the `server.initialAdminPassword` parameter during installation.

    !!! info "Important"
        The default `admin` password is stored in plain text. It is highly recommended to update it so the password is hashed. See [manage users in OpenEverest](../administer/manage_users.md#update-the-password).

6. To access the OpenEverest UI/API, open your browser and go to `https://everest.example.com`.

    !!! note
        Replace `everest.example.com` with your own domain.

## Next steps

[Provision a database :material-arrow-right:](../use/db_provision.md){.md-button}
