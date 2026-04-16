# Architecture

OpenEverest is an open-source platform for automated database provisioning and management on Kubernetes. This page explains the key architectural decisions behind OpenEverest and describes its main components.

## Kubernetes Operators

At the heart of OpenEverest is a deliberate bet on **Kubernetes Operators** as the primary building block for deploying and managing stateful workloads.

![!image](images/operator-architecture.png)

A Kubernetes Operator is a software extension that uses custom resources to manage applications and their components. Operators encode operational knowledge — the same knowledge a human expert would use to deploy, configure, scale, and recover a specific piece of software.

### Why Operators?

Unlike Helm charts, which excel at initial deployment, Operators provide full day-2 operational capabilities out of the box:

- **Scaling** — horizontal and vertical, with proper coordination across cluster members
- **Backups and restores** — scheduled or on-demand, with point-in-time recovery
- **Maintenance** — rolling restarts, minor and major version upgrades
- **Self-healing** — automatic recovery from pod failures and configuration drift

Beyond automation, database Operators carry built-in domain expertise. They don't just start database pods in the right order — they also configure replication topologies, deploy read/write proxies and connection poolers, and tune engine-level settings based on the resources available. This expertise would otherwise require years of operational experience to develop in-house.

## Main components

OpenEverest is composed of three main components that work together to give users a single pane of glass over their database fleet.

![!image](images/openeverest-architecture.png)

### OpenEverest Server

The **OpenEverest Server** is the control plane that users interact with directly. It exposes:

- A **web UI** for managing database clusters, backups, monitoring endpoints, and access control.
- A **REST API** for programmatic access to all OpenEverest capabilities.

### OpenEverest Operator

The **OpenEverest Operator** acts as an *operator of operators*. When a user creates or modifies a database cluster through the UI or API, the OpenEverest Operator translates that intent into the native `CustomResource` object of the appropriate database operator. This abstraction layer means users work with a single, consistent API regardless of which database technology is running underneath.

### Database Operators

**Database Operators** are Kubernetes Operators created by various open-source communities and vendors. They manage the lifecycle of a specific database engine — handling replication, failover, configuration, backups, and more according to each engine's requirements.

OpenEverest is designed to be modular: new database technologies can be added by integrating a new Operator, without changes to the core platform. The currently supported engines are listed on the [supported operators](install/supported_operators_k8s.md) page.
