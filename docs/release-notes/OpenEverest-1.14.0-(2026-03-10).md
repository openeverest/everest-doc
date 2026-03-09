# What's new in OpenEverest 1.14.0

➡️ **New to OpenEverest?** Get started with our [Quickstart Guide](../quick-install.md).

??? info "Expand to unleash the key updates"

    ## 📋 Release summary

    |**#**|**Category**|**Description**|
    |---------|---------------------|---------|
    | **1.**|[ARM support](#arm-support)|OpenEverest now supports ARM (arm64) architecture, enabling deployment on ARM-based servers and Apple Silicon Macs.|
    | **2.**|[Homebrew installation for everestctl](#homebrew-installation-for-everestctl)|Install `everestctl` on macOS and Linux with a single Homebrew command.|
    | **3.**|[Community Helm chart migration](#community-helm-chart-migration)|OpenEverest is now installed via the community-driven Helm chart hosted at `openeverest/helm-charts`.|
    | **4.**|[Operator upgrades](#operator-updates)|Support for Percona Operator for MongoDB v1.22.0.|
    | **5.**|[New features](#new-features)|Check out the new features introduced in OpenEverest 1.14.0|


## Release highlights

=== "💪 ARM support"
    ### OpenEverest now runs on ARM
    OpenEverest 1.14.0 adds full support for ARM (arm64) architecture. All OpenEverest Docker images are now published as multi-architecture manifests, supporting both `amd64` and `arm64`. This enables you to run OpenEverest on ARM-based servers (such as Ampere or AWS Graviton instances) as well as Apple Silicon Macs.

    **What this means for you:**

    - Install and run OpenEverest on ARM servers without workarounds
    - Use `everestctl` and all OpenEverest components natively on ARM hardware
    - `everestctl` Homebrew tap works on both Apple Silicon and Intel CPUs

    📘 Related GitHub issue: [#784](https://github.com/openeverest/openeverest/issues/784)

=== "🍺 Homebrew for everestctl"
    ### Install everestctl via Homebrew
    `everestctl` is now available through Homebrew, making installation on macOS and Linux as simple as two commands. This works on both Apple Silicon (arm64) and Intel (amd64) hardware.

    ```sh
    brew tap openeverest/tap
    brew install everestctl
    ```

    📘 For the full installation guide, see the [everestctl installation documentation](../install/install_everestctl.md).

    📘 Related GitHub issue: [#1728](https://github.com/openeverest/openeverest/issues/1728)

=== "⛵ Community Helm chart"
    ### Helm chart migrated to openeverest/helm-charts
    OpenEverest is now installed using the community-driven Helm chart hosted at [openeverest/helm-charts](https://github.com/openeverest/helm-charts). This replaces the previous Percona Helm chart and reinforces OpenEverest's commitment to vendor-neutral, community-owned infrastructure.

    Update your Helm repository reference to the new chart location when upgrading to OpenEverest 1.14.0.

    📘 See the [Helm installation guide](../install/install_everest_helm_charts.md) for updated commands.

=== "⚙️ Operators"
    ### Operator updates

    OpenEverest now supports:

    - Percona Operator for MongoDB [v1.22.0](https://docs.percona.com/percona-operator-for-mongodb/RN/Kubernetes-Operator-for-PSMONGODB-RN1.22.0.html)

    **Supported Percona Server for MongoDB versions:**

    - 6.0.27-21
    - 7.0.30-16
    - 8.0.19-7

## New features

- [#784](https://github.com/openeverest/openeverest/issues/784): OpenEverest Docker images are now published as multi-architecture manifests, adding native support for ARM (arm64) alongside AMD64.

- [#1728](https://github.com/openeverest/openeverest/issues/1728): `everestctl` is now available via Homebrew (`brew tap openeverest/tap && brew install everestctl`), supporting both macOS and Linux on ARM and AMD64.

- OpenEverest now supports Percona Operator for MongoDB v1.22.0 with Percona Server for MongoDB 6.0.27-21, 7.0.30-16, and 8.0.19-7.

- OpenEverest Helm chart has been migrated to the community-driven [openeverest/helm-charts](https://github.com/openeverest/helm-charts) repository.

## Improvements

- [#1910](https://github.com/openeverest/openeverest/pull/1910): Increased the RSA key size used for signing authentication JWT tokens from 2048 to 4096 bits for enhanced security.

## 🚀 Upgrade to OpenEverest 1.14.0

Because this release migrates the OpenEverest Helm chart from the `percona` repository to the community-driven `openeverest` repository, the upgrade process requires an additional one-time migration step.

Choose one of the following upgrade paths:

=== "Using everestctl"

    Upgrade using the latest version of `everestctl`. The CLI handles the Helm chart migration automatically.

    Make sure you are running the latest version of `everestctl` before starting the upgrade.

    📖 See the [Upgrade section](../upgrade/upgrade_with_cli.md) for the full steps.

=== "Using Helm directly"

    If you prefer to upgrade using Helm, follow these steps to migrate your existing releases from the `percona` Helm repository to the `openeverest` repository and upgrade them.
    {.power-number}

    1. Add the new OpenEverest Helm repository and update your local cache:

        ```sh
        helm repo add openeverest https://openeverest.github.io/helm-charts/
        helm repo update
        ```

    2. Upgrade the CRDs:

        ```sh
        helm upgrade --install everest-crds \
            openeverest/everest-crds \
            --namespace everest-system \
            --take-ownership
        ```

    3. Upgrade each release to re-associate it with the new `openeverest` chart:

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

    4. (Optional) Remove the old Percona Helm repository:

        ```sh
        helm repo remove percona
        ```
