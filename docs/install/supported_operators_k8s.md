# Supported technologies and Kubernetes versions

OpenEverest is designed to run on any CNCF-conformant Kubernetes cluster — cloud-managed or self-hosted. This page lists the database technologies available through OpenEverest Providers and the Kubernetes versions that are tested and supported.

## Supported technologies

In OpenEverest v2, database technologies are added through **Providers** — self-contained extensions installed independently of the core. Each Provider teaches OpenEverest how to manage a specific database or storage technology, so the list of supported technologies grows with the available Providers rather than being fixed in the core.

To discover and install Providers:

- Browse the Extension Hub inside the OpenEverest UI. See the [Extension Hub docs](../extend/hub.md) for details.
- Explore the online catalog at [openeverest.io/extensions :octicons-link-external-16:](https://openeverest.io/extensions/).

Want to add a technology that isn't listed? Anyone can build one — see [Extend OpenEverest](../extend/index.md) and the [Providers guide](../extend/providers.md).

## Supported Kubernetes versions

Each minor release of OpenEverest is tested against a range of Kubernetes versions — generally the minor versions actively supported by the CNCF at the time of the OpenEverest release.

The currently supported Kubernetes versions are **1.33 – 1.37**.

For reference, the CNCF support window and the list of actively maintained Kubernetes minor versions can be found on the [Kubernetes releases page :octicons-link-external-16:](https://kubernetes.io/releases/).

## Supported platforms

OpenEverest runs on any CNCF-conformant Kubernetes distribution. The following platforms are regularly used by the community and the maintainers:

- Cloud-managed Kubernetes (EKS, GKE, AKS, and equivalents)
- OpenShift
- Vanilla Kubernetes (kubeadm)
- On-premises distributions

!!! note
    Local development clusters (minikube, kind, k3d) may work for evaluation purposes but are not tested for production use cases due to networking and storage constraints.
