# Role-based access control (RBAC)


Role-based access control (RBAC) restricts access to resources within OpenEverest. It establishes a framework that defines access permissions and privileges according to individual users' roles. With RBAC, only authorized individuals can access specific information or perform certain actions based on their assigned roles. This method improves security by minimizing the risk of unauthorized access and helps manage permissions more efficiently across OpenEverest.

!!! warning
    RBAC will not work if you have configured Single sign-on (SSO) and your identity provider (IdP) is Microsoft Entra.


## How to enable RBAC

To enable or disable RBAC in OpenEverest, you can use a configuration flag that allows switching between RBAC-enabled and RBAC-disabled modes. By default, RBAC is disabled.

!!! info "Important"
    The RBAC configuration is stored in a `ConfigMap` named `everest-rbac` within the `everest-system` namespace.

Here's how you can enable RBAC:

```sh
apiVersion: v1
data:
  enabled: "true"
  policy.csv: |
    g, admin, role:admin
kind: ConfigMap
metadata:
  name: everest-rbac
  namespace: everest-system
```

## Policy definition in RBAC

RBAC policies are the rules and guidelines that define how roles, permissions, and users are managed within RBAC. These policies ensure that users have appropriate access to resources based on their roles within OpenEverest.


The policy definition in OpenEverest is:


```sh
p, <subject>, <resource-type>, <action>, <resource-name>
```

Where:

**subject**: Refers to the name of the role or user. For example, `role:admin` or `admin`

**resource-type**: Refers to the type of OpenEverest resource, such as `namespaces`, `database-clusters`, `database-engines`, etc.

For in-depth information on the actions that a subject can perform, see the [resources and permissions](#rbac-resources-and-permissions) section.

**action**: Refers to the action the subject can perform. For example, `read`, `update`, `delete`, `create`, or `*`


**resource-name**: Refers to a specific instance of the given resource-type. The argument should be prefixed with the namespace in which the resource is present `<namespace>/<resource-name>`. For example, `my-namespace/my-cluster-1`, `my-namespace-2/my-backup-1`, etc. You may also use a wildcard, such as `*`, `*/*`, or `my-namespace/*`

!!! info "Important"
    If you have permission for specific namespaces or resources, you can perform read, update, create, or delete actions **only** on those resources or **only** within those namespaces. However, if you have permission for **all** the resources or namespaces, you can carry out these actions across all the resources and namespaces.

Example:

`p, example-user, database-clusters, read, example-namespace/example-db`

In this case, the `example-user` user has permissions to read a database called `example-db` in the `example-namespace `namespace.

## RBAC resources and permissions

Below is a comprehensive table outlining the permissions available for various **resources**:

??? info "Permissions for resources"


    !!! warning "Important"
        :x: Represents an action that's not supported by the OpenEverest API.


    **Table: Permissions for the various resources in OpenEverest**

    |**Resource**| **Read**| **Create**| **Update**|**Delete**|
    |------------|---------|-----------|-----------|----------|
    | namespaces|You can view namespaces| :x:|:x: | :x:|
    | database-engines </br> (MySQL, MongoDB, PostgreSQL)|You can view database engines when you create databases</br>  </br>**Note**: This policy must at least be **read all** so the users can create databases.| :x: |Modify database engines| :x:|
    | database-clusters|You can view databases| You can create databases|You can modify databases| You can delete databases|
    | database-cluster-backups|You can view database cluster backups| You can create database cluster backups|:x:| You can delete database cluster backups|
    | database-cluster-restores|You can view database cluster restores| You can create database cluster restores|You can modify database cluster restores| You can delete database cluster restores|
    | backup-storages and monitoring-instances |You can view backups and monitoring endpoints| You can create backups and monitoring endpoints|You can modify backups and monitoring endpoints| You can delete backups and monitoring endpoints|
    | database-cluster-credentials|View database data (credentials) </br> </br>**Note**: If no policy is defined:</br> * You cannot see the credentials and the connection string.</br> * You also cannot create a database from any backup.| :x: |:x:| :x:|
    | load-balancer-configs |You can view load balancer configs| You can create load balancer configs|You can modify load balancer configs| You can delete load balancer configs|
    | pod-scheduling-policies |You can view pod scheduling policies| You can create pod scheduling policies|You can modify pod scheduling policies| You can delete pod scheduling policies|


    ### Key considerations for RBAC

    Before you start defining the different roles, there are some important things to consider when it comes to Role-Based Access Control (RBAC).
    {.power-number}

    === "Namespaces"

        Read `namespaces` permissions are required for **all** the **roles**. 

    === " DB clusters"
    
        To create, update and delete database clusters, backups, schedules, monitoring instances, and restores it is recommended to **explicitly** grant `read` permissions as well for the `database clusters`. Without these permissions, you would not be able to view these resources, which would not be practical.

    === "Resources"
    
        To create, update and delete the resources, it is recommended to **explicitly** grant `read` permissions for these resources as well. Without these permissions, you would not be able to view these resources, which would not be practical.

        Example: To manage `backup schedules` (create, update, delete a schedule), it is recommended to **explicitly** grant `read` permissions for the `backup schedules` as well as `backup storages`.

    === "Backups"
    
        For on-demand backups and schedules, you should grant `read` permissions for `backup storages` as well.

    === "Load balancer configs"

        If a database cluster uses a load balancer config, you must have `read` permissions for `load-balancer-configs` to view that database cluster.

    === "Restores"

        For restores, to new and existing databases, you should grant the following permisssions as well:

        - Read `backups` (of the old DB)
        - Read `MonitoringConfig` (if the old DB has monitoring enabled)
        - Read `database cluster credentials` (of the old DB)
        - Create `backups` (if backup schedules are enabled)
   
    === "Upgrades"

        For upgrades, the following permissions must be granted:

        - Read Namespaces
        - Read all `database engines` in that namespace
        - Read all `database clusters` in that namespace
        - Update `database clusters` in that namespace


## Roles in RBAC

In Role-Based Access Control (RBAC), a Role is a set of permissions that define what actions (like read, write, update, delete) can be performed on specific resources within OpenEverest. In RBAC, roles are assigned to users, allowing them to interact with the resources according to the permissions defined by their roles.

!!! note
    It is recommended to prefix role names with **role:** to prevent confusion between role names and user names.

### Built-in role

In OpenEverest, the only predefined role is `role:admin`. A user with this role has unrestricted access to OpenEverest. However, the RBAC (Role-Based Access Control) configuration can define and allocate specific roles based on individual requirements and access privileges.

This built-in `role:admin` definition is equivalent to the following:

```sh
p, role:admin, namespaces, *, *
p, role:admin, pod-scheduling-policies, *, *
p, role:admin, database-engines, *, */*
p, role:admin, database-clusters, *, */*
p, role:admin, database-cluster-backups, *, */*
p, role:admin, database-cluster-restores, *, */*
p, role:admin, database-cluster-credentials, *, */*
p, role:admin, backup-storages, *, */*
p, role:admin, monitoring-instances, *, */*
p, role:admin, load-balancer-configs, *, *
```


## RBAC examples

In this section, we will explore some examples that demonstrate how to create policy definitions for the required roles.

??? Example "Examples"

    Let's dive into some role definitions for RBAC:
    
    === "Namespace admin role" 
    
        **Admin role for a single namepsace**

        Let's set up an admin role with unrestricted privileges to all the resources in a single namespace called `namespaceA`.

        ```sh
        p, role:namespaceAadmin, namespaces, *, namespaceA
        p, role:namespaceAadmin, database-engines, *, namespaceA/*
        p, role:namespaceAadmin, database-clusters, *, namespaceA/*
        p, role:namespaceAadmin, database-cluster-backups, *, namespaceA/*
        p, role:namespaceAadmin, database-cluster-restores, *, namespaceA/*
        p, role:namespaceAadmin, database-cluster-credentials, *, namespaceA/*
        p, role:namespaceAadmin, backup-storages, *, namespaceA/*
        p, role:namespaceAadmin, monitoring-instances, *, namespaceA/*
        p, role:namespaceAadmin, load-balancer-configs, *, *
        p, role:namespaceAadmin, pod-scheduling-policies, *, *
        ```

        ??? info "Let's dive into decoding this!"

            The `namespaceAadmin` role has the following privileges within the `namespaceA` namespace:

            - **namespace**: `Read` access to the `namespaceA`
            - **Database engines**: `Read` and `update` access
            - **Database clusters**: `All` access (read, create, update, delete)
            - **Database cluster backups**: `All` access (read, create, delete) 
            - **Database cluster rstores**: `All` access (read, create, update, delete) 
            - **Database clusters credentials**: `Read` acccess
            - **Backup storages**: `All` access (read, create, update, delete)
            - **Monitoring instances**: `All` access (read, create, update, delete)
            - **Load balancer configs**: `All` access (read, create, update, delete)
            - **Pod scheduling policies**: `All` access (read, create, update, delete)
            

    === "Read only role" 
    
        **1. Read only role without access to the database credentials**

        Let's set up a read only role with access to all resources in all namespaces with the **exception** of database credentials:

        ```sh
        p, role:readonly, namespaces, read, *
        p, role:readonly, database-engines, read, */*
        p, role:readonly, database-clusters, read, */*
        p, role:readonly, database-cluster-backups, read, */*
        p, role:readonly, database-cluster-restores, read, */*
        p, role:readonly, backup-storages, read, */*
        p, role:readonly, monitoring-instances, read, */*
        p, role:readonly, load-balancer-configs, read, *
        p, role:readonly, pod-scheduling-policies, read, *
        ```
        
        ??? info "Let's dive into decoding this!"

            The `readonly`role has the following privileges in all the namespaces:

            - **namespace**:  `Read` access to all the namespaces
            - **Database engines**: `Read` access
            - **Database clusters**: `Read` access
            - **Database cluster backups**: `Read` access
            - **Database cluster restores**: `Read` access
            - **Backup storages**: `Read` access
            - **Monitoring instances**: `Read` access
            - **Load balancer configs**: `Read` access
            - **Pod scheduling policies**: `Read` access
            
            
        **2. Read only role with access to the database credentials**

        Let's set up a read only role that has read-only access to all resources in all namespaces, **including** access to the database credentials.

        ```sh
        p, role:readonlywithcreds, namespaces, read, *
        p, role:readonlywithcreds, database-engines, read, */*
        p, role:readonlywithcreds, database-clusters, read, */*
        p, role:readonlywithcreds, database-cluster-backups, read, */*
        p, role:readonlywithcreds, database-cluster-restores, read, */*
        p, role:readonlywithcreds, backup-storages, read, */*
        p, role:readonlywithcreds, monitoring-instances, read, */*
        p, role:readonlywithcreds, database-cluster-credentials, read, */*
        p, role:readonlywithcreds, load-balancer-configs, read, *
        p, role:readonlywithcreds, pod-scheduling-policies, read, *
        ```

        ??? info "Let's dive into decoding this!"

            The `readonlywithcreds` role has the following privileges in all the namespaces:

            - **namespace**: `Read` access to all the namespaces
            - **Database engines**: `Read` access
            - **Database clusters**: `Read` access
            - **Database cluster backups**: `Read` access
            - **Database cluster restores**: `Read` access
            - **Backup storages**: `Read` access
            - **Monitoring instances**: `Read` access
            - **Database clusters credentials**: `Read` acccess
            - **Load balancer configs**: `Read` access
            - **Pod scheduling policies**: `Read` access


    === "Database admin role" 

        **1. Database admin role with read access to certain resources**

        Let's set up a role that has read only access to the `database-engines`, `backup-storages` and `monitoring-instances`. This means that users assigned to this role *can manage the databases* without restriction but *cannot manage the database Kubernetes operators' versions*. They also cannot create, update, or delete `backup-storages` and `monitoring-instances`.

        ```sh
        p, role:dbadmin, namespaces, *, *
        p, role:dbadmin, database-engines, read, */*
        p, role:dbadmin, database-clusters, *, */*
        p, role:dbadmin, database-cluster-backups, *, */*
        p, role:dbadmin, database-cluster-restores, *, */*
        p, role:dbadmin, database-cluster-credentials, *, */*
        p, role:dbadmin, backup-storages, read, */*
        p, role:dbadmin, monitoring-instances, read, */*
        ```

        ??? info "Let's dive into decoding this!"

            The `dbadmin` role has the following privileges in **all** the namespaces:

            - **namespace**: `Read` access in all the namespaces.
            - **Database engines**: `Read` access to all the database engines
            - **Database clusters**: `All` access (read, create, update, delete)
            - **Database cluster backups**: `All` access (read, create, delete)
            - **Database cluster restores**: `All` access (read, create, update, delete)
            - **Database clusters credentials**: `Read` acccess for **all** the databases       
            - **Backup storages**: `Read` access to all the backup storages
            - **Monitoring instances**: `Read` access to all the monitoring instances


        **2. Database admin role for a single database**

        Let's set up a role that has access to only a single database called `databaseA`:

        ```sh
        p, role:dbadminDatabaseA, namespaces, *, namespaceA
        p, role:dbadminDatabaseA, database-engines, read, namespaceA/*
        p, role:dbadminDatabaseA, database-clusters, *, namespaceA/databaseA
        p, role:dbadminDatabaseA, database-cluster-backups, *, namespaceA/databaseA
        p, role:dbadminDatabaseA, database-cluster-restores, *, namespaceA/databaseA
        p, role:dbadminDatabaseA, database-cluster-credentials, *, namespaceA/databaseA
        p, role:dbadminDatabaseA, backup-storages, read, namespaceA/*
        p, role:dbadminDatabaseA, monitoring-instances, read, namespaceA/*
        ```

        ??? info "Let's dive into decoding this!"

            This role has the following privileges:

            - **namespace**: `Read` access in the namespace `namespaceA`.
            - **Database engines**: `Read` access in the namespace `namespaceA`
            - **Database clusters**: `All` access (read, create, update, delete) in namespace `namespaceA` only for database `databaseA`
            - **Database cluster backups**: `All` access (read, create, delete) in the namespace `namespaceA`
            - **Database cluster restores**: `All` access (read, create, update, delete) in the namespace `namespaceA`
            - **Database clusters credentials**: `Read` acccess for **all** the databases       
            - **Backup storages**: `Read` access to all the backup storagesin the namespace `namespaceA`
            - **Monitoring instances**: `Read` access to all the monitoring instances in the namespace `namespaceA`


## More insights into backups and restore policies

Let's dive into different backup and restore policies for OpenEverest.

### Read only role for a single namespace

Let's start with a read only role for a **single** namespace:

```sh
p, role:exampleA, namespaces, read, namespaceA
p, role:exampleA, database-engines, read, namespaceA/*
p, role:exampleA, database-clusters, read, namespaceA/*
p, role:exampleA, database-cluster-backups, read, namespaceA/*
p, role:exampleA, database-cluster-restores, read, namespaceA/*
p, role:exampleA, backup-storages, read, namespaceA/*
p, role:exampleA, monitoring-instances, read, namespaceA/*
```

#### Permissions to create on-demand backups

In the policy mentioned [above](#read-only-role-for-a-single-namespace), just add permissions to create on-demand backups.

`p, role:exampleA, database-cluster-backups, create, namespaceA/*`


#### Permissions to manage backup schedules

In the policy mentioned [above](#read-only-role-for-a-single-namespace), just add permissions to manage backup schedules.Backup schedules are part of the database cluster spec, so we require update permissions for the database clusters:


```sh
p, role:exampleA, database-cluster-backups, create, namespaceA/*
p, role:exampleA, database-clusters, update, namespaceA/*
```

#### Permissions to restore a backup to an existing database

In the policy mentioned [above](#read-only-role-for-a-single-namespaceread-only-role-for-a-single-namespace), just add permissions to restore a backup to an existing database:

```sh
p, role:exampleA, database-cluster-restores, create, namespaceA/*
p, role:exampleA, database-cluster-credentials, read, namespaceA/*
```

!!! note
    You cannot restore the backup to an existing database without having read permissions to the database cluster credentials.


#### Permissions to restore a backup to a new database

In the policy mentioned [above](#read-only-role-for-a-single-namespace), just add permissions to restore a backup to a new database:

```sh
p, role:exampleA, database-cluster-restores, create, namespaceA/*
p, role:exampleA, database-cluster-credentials, read, namespaceA/*
p, role:exampleA, database-clusters, create, namespaceA/*
```


## Assigning roles to users

In order for roles to take effect, they need to be assigned to users. The syntax for this is as follows:

```sh
g, username, rolename
```

A new user in OpenEverest will initially have **no** permissions. To grant permissions, you must edit your RBAC configuration stored in the `everest-rbac` `ConfigMap` in the `everest-system` namespace:


```sh
kubectl edit cm everest-rbac -n everest-system
```

A text editor will open, and you can edit the `ConfigMap` as follows. You just have to add the new user and assign it the desired role.


```sh
apiVersion: v1
kind: ConfigMap
metadata:
  name: everest-rbac
data:
  policy.csv: |
    g, admin, role:admin
    g, <newuser>, role:admin
```

??? Example "Example"

    Let's assign the role `team-dev` to a new user named `john`:

    ```sh       
    p, role:team-dev, namespaces, read, dev
    p, role:team-dev, database-engines, read, dev/*
    p, role:team-dev, database-clusters, read, dev/*
    p, role:team-dev, database-clusters, update, dev/*
    p, role:team-dev, database-clusters, create, dev/*
    p, role:team-dev, database-clusters, delete, dev/*
    p, role:team-dev, database-cluster-credentials, read, dev/*
    g, john, role:team-dev
    ```

    The `team-dev` role has the following privileges **only** within the `dev` namespace:

    - **namespace**:  `Read` access to `dev`
    - **Database engines**: `Read` access
    - **Database clusters**: `Read` access
    - **Database clusters**: `Update` access
    - **Database clusters**: `Create` access
    - **Database clusters**: `Delete` access
    - **Database cluster credentials**: `Read` access

## Validate your RBAC policy

You can verify whether your Role-based access control (RBAC) policies are functioning correctly by executing the following command:

```sh
everestctl settings rbac validate --policy-file <file_path>
```

Where:

`policy-file` is an optional flag that takes the **policy file** path. If you do not specify the path to this file, it will look for the configuration file inside your existing OpenEverest installation, that is, under RBAC `ConfigMap`.

??? Example "Policy validation"

    The following example demonstrates that this is a valid policy.

    ```sh
    everestctl settings rbac validate --policy-file ./pkg/rbac/testdata/policy-1-good.csv
    ```
    Output:

    ```{.text .no-copy}
    ✓ Valid
    ```

    The following example demonstrates that this is an invalid policy.


    ```sh
    everestctl settings rbac validate --policy-file ./pkg/rbac/testdata/policy-bad.csv
    ```

    output:

    ```{.text .no-copy}
    × Invalid
    policy syntax error - unknown resource name 'non-existent-resource'
    ```

    You can also use the same command with the YAML file path containing the everest-rbac ConfigMap. For example:

    
    ```sh
    everestctl settings rbac validate --policy-file ./rbac-config.yaml
    ```

    


## Test your RBAC policy

You can verify if a role or individual (such as a group or a local user) has the necessary privileges to perform particular operations on designated resources. 

We have a straightforward command that can be used to test the RBAC (Role-Based Access Control) policies:


```sh
everestctl settings rbac can --policy-file <file_path>
```

Where:

`policy-file` is an optional flag that takes the **policy file** path. If you do not specify the path to this file, it will look for the configuration file inside your existing OpenEverest installation, that is, under RBAC `ConfigMap`.


??? Example "Test your policy"

    The following example tests whether a user, **Alice**, can create database clusters:

    ```sh
    everestctl settings rbac can alice create database-clusters '*' --policy-file ./pkg/rbac/testdata/policy-1-bad.csv
    ```
    Output

    ```{.text .no-copy}
    No
    ```


    The following example tests whether an **Admin** user can create database clusters:
    
    ```sh
    everestctl settings rbac can admin create database-clusters '*' --policy-file ./pkg/rbac/testdata/policy-1-good.csv
    ```

    Output:

    ```{.text .no-copy}
    Yes
    ```


## Breaking API changes for RBAC

Starting from OpenEverest v1.2.0, breaking changes are being implemented to the API for `monitoring-instances` and `backup-storages `resources. Explore further by checking out the section on [Breaking API changes](../api_rbac.md) for a deep dive into this topic.


















    
    
  






