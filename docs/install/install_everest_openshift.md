# Install OpenEverest on OpenShift

This section explains how to install OpenEverest on OpenShift using [Helm](https://helm.sh/){:target="_blank"}.

!!! warning "Developer Preview"
    This is a **developer preview** release. Features are incomplete and subject to change. The `everestctl` installation method is not available for this release; install OpenEverest with Helm.

## Install OpenEverest
{.power-number}

1. Add the OpenEverest Helm repository:

    ```sh
    helm repo add openeverest https://openeverest.github.io/helm-charts/
    helm repo update
    ```

2. Install the OpenEverest core with **OpenShift compatibility enabled**:

    ```sh
    helm install everest-core openeverest/openeverest \
      --devel \
      --namespace everest-system \
      --create-namespace \
      --set compatibility.openshift=true
    ```

    The `--devel` flag installs the latest developer preview. To pin an exact release, add `--version`, for example `--version "2.0.0-dev.3"`.

3. Install the MongoDB Provider:

    ```sh
    helm install provider-percona-server-mongodb \
      oci://ghcr.io/openeverest/charts/provider-percona-server-mongodb \
      --namespace everest-system
    ```

    !!! tip "Find more providers and plugins in the Plugin Hub"
        OpenEverest ships with the **Plugin Hub**, an in-product catalog for discovering additional providers and plugins. Open it in the OpenEverest UI at `/plugins/plugin-hub`, or find it in the left-hand menu. Learn more in the [Extension Hub docs](../extend/hub.md), browse the online catalog at [openeverest.io/extensions :octicons-link-external-16:](https://openeverest.io/extensions/), or read the [Plugin Hub introduction blog post](https://openeverest.io/blog/the-hub-introduction/).

4. Once the installation is complete, retrieve the `admin` password:

    ```sh
    kubectl get secret everest-accounts -n everest-system -o jsonpath='{.data.users\.yaml}' | base64 --decode | yq '.admin.passwordHash'
    ```

    The default username for the OpenEverest UI is `admin`. You can set a different initial admin password with the `server.initialAdminPassword` parameter during installation.

    !!! info "Important"
        The default `admin` password is stored in plain text. It is highly recommended to update it so the password is hashed. See [manage users in OpenEverest](../administer/manage_users.md#update-the-password).

5. Access the OpenEverest UI/API. OpenEverest is not exposed with an external IP by default, so use one of the following options:

    === "Load Balancer"

        1. Change the `everest` service type to `LoadBalancer`:

            ```sh
            kubectl patch svc/everest -n everest-system -p '{"spec": {"type": "LoadBalancer"}}'
            ```

        2. Retrieve the external IP address for the `everest` service:

            ```sh
            kubectl get svc/everest -n everest-system
            ```

            ??? example "Expected output"
                ```
                NAME      TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)         AGE
                everest   LoadBalancer   10.43.172.194   34.175.201.246   8080:8080/TCP   10s
                ```

        Open the OpenEverest UI at the external IP address, for example `http://34.175.201.246:8080`.

    === "Node Port"

        A NodePort service makes a specific port accessible on all nodes within the cluster, assigning a static port on each node's IP address.

        1. Change the `everest` service type to `NodePort`:

            ```sh
            kubectl patch svc/everest -n everest-system -p '{"spec": {"type": "NodePort"}}'
            ```

        2. Find the port Kubernetes assigned to the `everest` service (here, `32349`):

            ```sh
            kubectl get svc/everest -n everest-system
            ```

            ```{.text .no-copy}
            NAME      TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
            everest   NodePort   10.43.139.191   <none>        8080:32349/TCP   28m
            ```

        3. Retrieve the external IP addresses of the cluster nodes:

            ```sh
            kubectl get nodes -o wide
            ```

        4. Open the OpenEverest UI at any node IP address on the assigned port, for example `http://34.175.155.135:32349`.

    === "Port Forwarding"

        1. Set up a port-forward to the `everest` service:

            ```sh
            kubectl port-forward svc/everest 8080:8080 -n everest-system
            ```

            Open the OpenEverest UI at `http://127.0.0.1:8080`.

        2. (**Recommended**) When **TLS is enabled**, forward the TLS port instead:

            ```sh
            kubectl port-forward svc/everest 8443:443 -n everest-system
            ```

            OpenEverest is then available at `https://127.0.0.1:8443`.

            For comprehensive instructions on enabling TLS for OpenEverest, see [TLS setup with OpenEverest](../security/tls_setup.md#tls-setup-with-percona-everest).

## Next steps

[Provision a database :material-arrow-right:](../use/db_provision.md){.md-button}
