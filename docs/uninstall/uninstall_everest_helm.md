# Uninstall OpenEverest using Helm

!!! warning "Developer Preview"
    This is a **developer preview** release. Features are incomplete and subject to change. The `everestctl` installation method is not available for this release, so OpenEverest is uninstalled with Helm.

!!! info "Important"
    If you installed OpenEverest using Helm, make sure to uninstall it exclusively through Helm for a seamless removal.

!!! note alert alert-primary "Warning"
    Uninstalling OpenEverest removes all database clusters and their data from the Kubernetes cluster, including backups. Back up any data you want to keep before you continue.

## Before you begin

Delete your database clusters from the OpenEverest UI first. This lets OpenEverest clean up the underlying database resources (and their backups) while the providers are still running. Removing the Helm releases or namespaces first can leave orphaned resources behind.

## Uninstall OpenEverest

To uninstall OpenEverest, follow these steps:
{.power-number}

1. List the Helm releases in the `everest-system` namespace to identify the OpenEverest core release and any installed providers:

    ```sh
    helm list -n everest-system
    ```

    ??? example "Expected output"
        ```sh
        NAME                              NAMESPACE       REVISION  STATUS    CHART
        everest-core                      everest-system  1         deployed  openeverest-2.0.0-dev.3
        provider-percona-server-mongodb   everest-system  1         deployed  provider-percona-server-mongodb-2.0.0-dev.3
        ```

2. Uninstall each provider release. Repeat this command for every provider listed in the previous step:

    ```sh
    helm uninstall provider-percona-server-mongodb -n everest-system
    ```

3. Uninstall the OpenEverest core release:

    ```sh
    helm uninstall everest-core -n everest-system
    ```

4. Delete the namespaces created by OpenEverest:

    ```sh
    kubectl delete ns everest-system
    kubectl delete ns everest-monitoring
    ```

    !!! note
        The `everest-monitoring` namespace exists only if the monitoring stack was installed. If the namespace is not found, you can safely skip it.

5. Remove the OpenEverest [Custom Resource Definitions (CRDs)](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/).

    !!! caution alert alert-warning "warning"
        Helm does not delete CRDs when you uninstall a release, so this step is required for a complete removal. Deleting a CRD also deletes every custom resource of that type across the cluster. Make sure the CRDs are not used by anything outside OpenEverest before you delete them.

## Remove CRDs

OpenEverest installs the following CRDs. List the ones currently present with `kubectl get crd | grep openeverest.io`, then delete them by name:

```sh
kubectl delete crd \
  providers.core.openeverest.io \
  instances.core.openeverest.io \
  instancepresets.core.openeverest.io \
  backupclasses.backup.openeverest.io \
  backups.backup.openeverest.io \
  backupimports.backup.openeverest.io \
  restores.backup.openeverest.io \
  backupstorages.backup.openeverest.io \
  monitoringconfigs.monitoring.openeverest.io \
  plugins.extensions.openeverest.io \
  installedextensions.extensions.openeverest.io
```

!!! caution alert alert-warning "Provider CRDs"
    Each provider brings its own database operator CRDs (for example, the Percona Server for MongoDB CRDs). These are cluster-scoped and are **not** removed when you uninstall the provider chart. Remove them only if no other workloads in the cluster depend on them. Inspect the remaining CRDs with:

    ```sh
    kubectl get crd
    ```
