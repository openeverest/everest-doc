# Upgrade OpenEverest using Helm

!!! warning "ACTION REQUIRED: OpenEverest and Bitnami Container Catalog changes"
    Bitnami is **restructuring** its container catalog on **September 29, 2025**. To avoid potential failures in OpenEverest operations, follow the steps outlined in this [post](https://github.com/openeverest/openeverest/discussions/1663).

OpenEverest consistently delivers updates that includes bug fixes, security enhancements, and various improvements designed to optimize the overall performance of your database.

## Before you upgrade

!!! Info "Important"
    - Upgrade one minor release at a time to avoid unexpected issues. Start by updating to the latest patch release before moving on to the next minor release.
    - For a safe upgrade, we perform a pre-upgrade hook that runs a series of checks. You can disable this by setting `upgrade.preflightChecks=false`, but doing so means a safe upgrade cannot be assured.


### Prepare for upgrading to version 1.7.0


**Before you upgrade** to OpenEverest version 1.7.0, run the following command:

```sh
kubectl label namespaces everest-system app.kubernetes.io/managed-by-
```

## Migrate from the Percona Helm repository

If you originally installed OpenEverest using the `percona` Helm repository, you need to perform a one-time migration to switch your existing releases to the new `openeverest` repository. This does not reinstall OpenEverest — it simply re-associates your existing Helm releases with the new chart source while preserving all your current configuration.

To migrate, run the following commands:
{.power-number}

1. Add the new OpenEverest Helm repository:

    ```sh
    helm repo add openeverest https://openeverest.github.io/helm-charts/
    ```

2. Update your local Helm repository cache:

    ```sh
    helm repo update
    ```

3. Upgrade each release to use the chart from the new repository.

    Upgrade the core components release:

    ```sh
    helm upgrade everest-core openeverest/openeverest \
      --namespace everest-system \
      --reuse-values
    ```

    If you have one or more database namespace releases, upgrade each one:

    ```sh
    helm upgrade <RELEASE_NAME> openeverest/everest-db-namespace \
      --namespace <DB_NAMESPACE> \
      --reuse-values
    ```

    Replace `<RELEASE_NAME>` and `<DB_NAMESPACE>` with your actual release name and namespace. To list all your current releases, run `helm list --all-namespaces`.

4. (Optional) Remove the old Percona Helm repository once all releases have been migrated:

    ```sh
    helm repo remove percona
    ```

## Upgrade CRDs

In Helm v3, CRDs are not updated automatically during a Helm upgrade. You need to upgrade the CRDs manually.

To update the CRDs, run the following command:

```sh
helm repo update
helm upgrade --install everest-crds \
    openeverest/everest-crds \
    --namespace everest-system \
    --take-ownership
```

### Upgrading with Helm versions prior to 3.17.0
    
If you upgrade from **OpenEverest 1.8.0** and use a Helm version **older than 3.17.0**, the `--take-ownership` flag will not be available. Without this flag, you may encounter the following validation errors related to missing ownership metadata:

```
invalid ownership metadata; label validation error: missing key "app.kubernetes.io/managed-by": must be set to "Helm";
annotation validation error: missing key "meta.helm.sh/release-name": must be set to "everest-crds";
annotation validation error: missing key "meta.helm.sh/release-namespace": must be set to "everest-system"
```

#### Workaround for Helm versions older than 3.17.0

If you **must** use a Helm version **older than 3.17.0**, you can manually simulate the behavior of ``--take-ownership`` by adding the required labels and annotations to the Everest CRDs:

```
CRDS=(databaseclusters.everest.percona.com              databaseengines.everest.percona.com databaseclusterbackups.everest.percona.com databaseclusterrestores.everest.percona.com backupstorages.everest.percona.com monitoringconfigs.everest.percona.com)
kubectl label crds "${CRDS[@]}" app.kubernetes.io/managed-by=Helm --overwrite
kubectl annotate crds "${CRDS[@]}" meta.helm.sh/release-name=everest-crds
kubectl annotate crds "${CRDS[@]}" meta.helm.sh/release-namespace=everest-system
```

This ensures the CRDs are correctly recognized as managed by Helm, avoiding validation issues during the upgrade.

## Upgrade Helm releases

!!! warning
    When using `helm upgrade`, specifying `--set` (or other equivalent flags) causes Helm to revert all other values to the defaults defined in the chart. To avoid this issue, either include the `--reuse-values` flag or provide the complete set of values, including those used during the installation.

To upgrade OpenEverest using Helm, run the following commands:
{.power-number}

1. Upgrade the Helm release for Everest (core components).

    ```sh
    helm upgrade everest-core openeverest/openeverest --namespace everest-system --version "$VERSION"       
    ```

    where,

    **VERSION** is the OpenEverest version you wish to upgrade to.

2. Upgrade the Helm release for the database namespace (if applicable):

    ```sh
    helm upgrade everest openeverest/everest-db-namespace --namespace <DB namespace> --version "$VERSION"
    ```

    where,

    **VERSION** is the OpenEverest version you wish to upgrade to.

    **DB namespace** is the namespace you wish to upgrade.
