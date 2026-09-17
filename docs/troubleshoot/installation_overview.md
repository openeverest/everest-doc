# OpenEverest installation and workflow

This page provides an overview of how OpenEverest is installed, the components involved, and the workflow, from provider installation to database provisioning, backups, and restores.

!!! warning "Developer Preview"
    This is a **developer preview** release. Features are incomplete and subject to change. OpenEverest is installed with [Helm](../install/install_everest_helm_charts.md); the `everestctl` installation method is not available for this release.

## Installation workflow

OpenEverest is installed in two stages: the core platform, followed by one or more providers.
{.power-number}

1. The `openeverest` (core) chart is installed in the `everest-system` namespace as the `everest-core` release. This deploys all the core components required for OpenEverest to function.

    | Component category | Namespace            | Components                             |
    |--------------------|----------------------|----------------------------------------|
    | Everest components | `everest-system`     | `everest-server`, `everest-controller` |
    | Monitoring stack   | `everest-monitoring` | `vm-operator`, `kube-state-metrics`    |

    - `everest-server` serves the OpenEverest UI and API.
    - `everest-controller` is the OpenEverest operator that reconciles OpenEverest custom resources.

2. One or more **provider** charts are installed into `everest-system`. Each provider is a self-contained Helm release that bundles a database operator and registers a `Provider` resource describing the database engines and versions it supports.

    ```sh
    helm install provider-percona-server-mongodb \
      oci://ghcr.io/openeverest/charts/provider-percona-server-mongodb \
      --namespace everest-system
    ```

    Discover additional providers and plugins in the in-product **Plugin Hub** or in the [Extension Hub docs](../extend/hub.md).

!!! note
    OpenEverest no longer uses the Operator Lifecycle Manager (OLM). Providers are installed and upgraded directly through Helm, so `helm install` / `helm upgrade` is the apply step — there is no install-plan approval job.

## Server and controller workflow

### Database creation workflow

Here is the database creation workflow in OpenEverest:
{.power-number}

1. The OpenEverest user is authenticated and logged in, and a `JWT Token` is provided.
2. The user creates a database via the OpenEverest UI or API.
3. The OpenEverest API is invoked to create a new database. The OpenEverest server then creates an `Instance` custom resource on the Kubernetes cluster.
4. In the `everest-controller` reconciliation loop, once the `Instance` object is recognized, the responsible provider creates the database operator's own custom resource (for example, `PerconaServerMongoDB` for MongoDB).
5. The database operator takes over the task of creating the database and the necessary objects to manage it.

    !!! note
        Similar workflows apply to **backups, restores, monitoring, and deletions**, although the object types may vary.

### Provider workflow

Here is how a provider surfaces its supported database engines:
{.power-number}

1. You install a provider chart, either alongside the core chart or as a separate step.
2. The provider deploys its database operator into `everest-system` and registers a `Provider` resource.
3. The `everest-controller` reconciles the `Provider` resource and reads the engine versions the provider declares as supported.

    !!! note
        Supported engine versions are bundled with the provider release itself — OpenEverest does not query an external version service. Upgrading the provider is how you change the set of available engine versions.

4. The resolved engine versions become available to users when they create or edit an `Instance`.
