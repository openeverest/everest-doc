# Uninstall OpenEverest using everestctl

!!! warning "Developer Preview"
    The `everestctl` uninstall method is **not available** in the v2.0.0-dev.1 developer preview. Uninstall OpenEverest with [Helm](uninstall_everest_helm.md) instead. The steps below describe the `everestctl` workflow for reference.

!!! info "Important"
    If you installed OpenEverest using `everestctl`, make sure to uninstall it exclusively through `everestctl` for a seamless removal.

!!! note alert alert-primary "Warning"
    Uninstalling OpenEverest removes all database clusters and their data from the Kubernetes cluster, including backups. Back up any data you want to keep before you continue.

## Uninstall OpenEverest

Run the following command and confirm the prompt:

```sh
everestctl uninstall
```

`everestctl uninstall` removes everything OpenEverest created, including:

- The OpenEverest Helm release and the `everest-system` namespace
- The monitoring stack and the `everest-monitoring` namespace
- All database namespaces managed by OpenEverest, along with their database clusters and data
- All OpenEverest CRDs

By default, the command asks for confirmation before proceeding. Use the following flags to change its behavior:

| Flag | Description |
|------|-------------|
| `-y`, `--yes` | Assume yes to all prompts and skip confirmation. |
| `-f`, `--force` | Proceed even if there are database clusters still running. |

!!! caution alert alert-warning "Provider CRDs"
    Database operator CRDs installed by providers (for example, the Percona Server for MongoDB CRDs) are cluster-scoped and are **not** removed by `everestctl uninstall`. Remove them manually only if no other workloads in the cluster depend on them. See [Remove CRDs](uninstall_everest_helm.md#remove-crds) for details.
