# OpenEverest: Frequently asked questions (FAQ)

This page outlines the most frequently asked questions (FAQs) about OpenEverest.


## What is a Provider?

A `Provider` is a resource that represents an installed database provider (for example, Percona Server for MongoDB). Each provider bundles a database operator and declares:

- The database engine it manages (MySQL, MongoDB, or PostgreSQL).
- The engine versions it supports.

Users create databases as `Instance` resources, which reference a provider and one of its supported engine versions.

## Does OpenEverest provide logs for API calls?

Yes, the OpenEverest backend has logs. It runs as an `everest-server` deployment within the `everest-system` namespace.

- Accessing these logs helps us verify whether an API call was initiated for a specific user operation.

- Identify any errors that may have occurred during that process.

!!! note
    OpenEverest logs are essential for troubleshooting issues when an operation is **completed**, but the corresponding Everest operator resource **fails to be created**.


Run the following command to retrieve the logs from the pods associated with the deployment:

```sh
kubectl logs -f deploy/everest-server -n everest-system
```


## How to troubleshoot issues between the UI and the OpenEverest API? 

You can view the logs by running the following command:

```sh
kubectl logs -f deploy/everest-server -n everest-system
```

## How to identify which component is failing?

All communication with OpenEverest resources begins with the API.

The API (`everest-server`) updates OpenEverest resources, while the `everest-controller` reconciles them and hands off to the provider's database operator.

## Recommended troubleshooting flow

When debugging, start with the API (`everest-server`), proceed to the `everest-controller`, and then examine the provider's database operator.


## Does OpenEverest deploy PMM servers?

No. OpenEverest does not deploy PMM (Percona Monitoring and Management). PMM v3 is still supported as an external monitoring target, but only by some providers — monitoring support is provider-dependent. When a provider supports it, OpenEverest configures the database's monitoring agents to send metrics to your existing PMM server.

- A `MonitoringConfig` holds the details required to connect to the PMM server, such as its URL and API key.
- Point the database at that configuration to start sending metrics.

For cluster-level Kubernetes metrics, OpenEverest also ships a VictoriaMetrics-based stack in the `everest-monitoring` namespace.

!!! note
    PMM is an external service; you must run your own PMM server for the providers that support it.