# Supported operators and Kubernetes versions

OpenEverest is designed to run on any CNCF-conformant Kubernetes cluster — cloud-managed or self-hosted. This page lists the database operators bundled with OpenEverest and the Kubernetes versions that are tested and supported.

## Operators

* [Percona Operator for MySQL Based on Percona XtraDB Cluster (PXC)](https://docs.percona.com/percona-operator-for-mysql/pxc/) 1.18.0, 1.19.0
* [Percona Operator for MongoDB (PSMDB)](https://docs.percona.com/percona-operator-for-mongodb/) 1.21.2, 1.22.0
* [Percona Operator for PostgreSQL (PG)](https://docs.percona.com/percona-operator-for-postgresql/2.0/) 2.7.0, 2.8.2

## Supported Kubernetes versions

Each minor release of OpenEverest is tested against a range of Kubernetes versions — generally the minor versions actively supported by the CNCF at the time of the OpenEverest release.

The currently supported Kubernetes versions are **1.33 – 1.35**.

For reference, the CNCF support window and the list of actively maintained Kubernetes minor versions can be found on the [Kubernetes releases page :octicons-link-external-16:](https://kubernetes.io/releases/).

## Supported platforms

OpenEverest runs on any CNCF-conformant Kubernetes distribution. The following platforms are regularly used by the community and the maintainers:

- Cloud-managed Kubernetes (EKS, GKE, AKS, and equivalents)
- OpenShift
- Vanilla Kubernetes (kubeadm)
- On-premises distributions

!!! note
    Air-gapped environments are not currently supported. Support is planned for a future release.

!!! note
    Local development clusters (minikube, kind, k3d) may work for evaluation purposes but are not tested for production use cases due to networking and storage constraints.
