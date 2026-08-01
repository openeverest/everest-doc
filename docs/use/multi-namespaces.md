# Multiple namespaces

In Kubernetes, the concept of namespaces enables you to create isolated groups of resources within a single cluster. These namespaces provide a way to organize and manage resources without interfering with other resources within the same cluster.


!!! info "Important"
    Resource names must be unique within a specific namespace but not across different namespaces.


## Use case for multiple namespaces

If you are dealing with complex environments comprising Kubernetes clusters that need to be used with OpenEverest, you can leverage the multiple namespace feature. This feature enables logical partitioning within the cluster.

For example, you can deploy different environments like production, development, and QA within a single cluster by using multiple namespaces. This approach enables you to efficiently manage the clusters.

## Default namespaces in OpenEverest

!!! info "Important"
    The following namespaces are restricted and cannot be used for deploying databases.

OpenEverest will create the following namespaces by default. You can see these default namespaces while OpenEverest is being installed.

- everest-olm: hosts the [Operator Lifecycle Manager](https://olm.operatorframework.io/) that manages all operators that are part of everest
- everest-system: hosts Everest 
- everest-monitoring: hosts the monitoring stack for kubernetes metrics ([VictoriaMetrics](https://victoriametrics.com/) and [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics))

To set up the namespaces that OpenEverest will manage and where you can deploy your databases, see the [Installation](../install/installEverest.md#installation) section.

## Configure multiple namespaces

The following holds true for multiple namespaces:

- You can [configure multiple namespaces](../administer/manage_namespaces.md#add-new-namespaces) in OpenEverest using the `everestctl namespaces add [NAMESPACE]` command.

- You can [install different operators](../administer/manage_namespaces.md#update-namespaces) in various namespaces using the `everestctl namespaces update [NAMESPACE]` command.


??? example "Example: Configuring multiple namespaces and installing various operators within those namespaces"
    To install various operators in different namespaces, such as MongoDB and MySQL operator in namespace production, and PostgreSQL operator in namespace development, run the following commands:
    {.power-number}

    1. `everestctl namespaces add development`
                
        ??? example "Expected output"
            ```sh
            ? Which operators do you want to install? PostgreSQL
            ✓ Installing namespace 'development'
            ```

    2. `everestctl namespaces add production`.

        ??? example "Expected output"
            ```sh
            ? Which operators do you want to install? MySQL, MongoDB
            ✓ Installing namespace 'production'
            ```


Go to OpenEverest UI and navigate to <i class="uil uil-cog"></i> **Settings > Namespaces**. A list of all the namespaces that you have created will appear here.

![!image](../images/everest_multinamespaces.png)

## Deploy the database in your namespace

Once you have configured your namespaces, you can choose the namespace where you want to deploy your new database cluster.

 ![!image](../images/everest_multi-namespaces.png)

For information on deploying a new database cluster in the namespace, see the [Provision a database](../use/db_provision.md) section.















