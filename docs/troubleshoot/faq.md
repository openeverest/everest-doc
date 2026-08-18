# OpenEverest: Frequently asked questions (FAQ)

This page outlines the most frequently asked questions (FAQs) about OpenEverest.


## What is a DatabaseEngine?

The `DatabaseEngine` is a resource that represents the installed engines (MySQL, MongoDB, PostgreSQL). 

Each `DatabaseEngine` indicates:

- The operator version currently installed.
- Compatible engine versions supported by that operator. 

Refer to our [API documentation](https://openeverest.io/docs/api/#/operations/getKubernetesClusterResources){:target="_blank"} for usage information.

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

The API is responsible for updating Everest resources, while the Everest operator continues to create resources for the corresponding database operators. 

## Recommended troubleshooting flow

When debugging, start with the API, proceed to the Everest operator, and examine the individual database operators.


## Does OpenEverest deploy PMM servers?

OpenEverest doesn't deploy PMM (Percona Monitoring and Management). However, you can deploy a PMM server while installing OpenEverest. You need to set `pmm.enabled=true`. 

We configure PMM agents in each DB deployment to communicate with an existing PMM server.

The following table shows the [configurable parameters](https://github.com/openeverest/helm-charts/tree/v1.x/charts/everest#configuration){:target="_blank"} of OpenEverest chart and their default values.

### Monitoring configuration highlights:

- `MonitoringConfig` contains all necessary details to connect to the PMM Server, including URL and API key.

-  An API key is generated to facilitate data transmission. This API key allows us to configure monitoring endpoints for individual database operators, enabling them to send data to PMM.

!!! note
    PMM is an external service supported by OpenEverest.