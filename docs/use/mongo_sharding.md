# MongoDB sharding

[Sharding  :octicons-link-external-16:](https://docs.mongodb.com/manual/reference/glossary/#term-sharding) is used for horizontal database scaling. It distributes a database horizontally across multiple nodes or servers, known as **shards**. Each shard manages a portion of the data, forming a sharded cluster, which enables MongoDB to handle large datasets and high user concurrency effectively.


## Key components of MongoDB sharding

- **Shard:** Each shard has a subset of the data.
- **Routers:** The query router directs the client queries to the proper shard(s).

- **Config servers:** The configuration servers store the cluster's metadata and configuration settings.

## Why sharding?

Here are the key reasons for implementing sharding:

### Scalability

By increasing the number of shards in the cluster, you can scale your system horizontally. This enables you to manage larger datasets and handle higher traffic volumes, ensuring your system remains responsive and efficient despite growing demands.


### Improved performance

By splitting data across multiple servers, sharding reduces the load on any one server, which leads to faster query response times. Since each shard contains only a fraction of the total data, queries targeting a specific shard can be processed faster.


### Availability

If one shard (or server) fails, the remaining shards can continue to operate, enhancing the system's availability and fault tolerance. Data replication across replica sets ensures that no information is lost in the event of a failure.

## Create MongoDB sharded cluster

!!! warning
    - Once you have a sharded collection, ensure you take a new backup to avoid data inconsistency and potential restoration failures.

    - There are a few more limitations related to MongoDB sharding. See the [limitations](../reference/known_limitations.html#mongodb-sharding) section for details about these limitations.


To create a sharded cluster:
{.power-number}

1. From the OpenEverest homepage, click **Create Database** and select the **MongoDB** database. The **Basic information** page will be displayed.

2. On the **Basic Information** page, turn the **Sharded Cluster** toggle to **on**.

    ![!image](../images/everest_sharded_cluster.png)

3. Click **Continue**. The **Resources** page will be displayed.

4. On the **Resources** page, choose or enter the following details:

       * Number of Shards

       * On the **Nodes** panel, select the **Number of nodes** per shard and the corresponding **Resource size per node**.

       * On the **Routers** panel, select the **Number of routers** and the corresponding **Resource size per router**.

       * Number of configuration servers

    ![!image](../images/sharding_routers.png)


5. To check if sharding is enabled, go to the database view page and click on the specific database. Then, check the **Resources** panel to see if sharding is enabled.

    ![!image](../images/sharding_status.png)


### Obtain credentials for sharding collections

You need to obtain the credentials for a user with permissions to shard collections.

Here are instructions on obtaining permissions to shard a collection:

```sh
MONGODB_CLUSTER_ADMIN_USER=$(kubectl get secrets --namespace <NAMESPACE> everest-secrets-<CLUSTER_NAME> -o template='{{ "{{"}}.data.MONGODB_CLUSTER_ADMIN_USER | base64decode {{"}}"}}{{"{{"}}"\n"{{"}}"}}')
```

```sh
MONGODB_CLUSTER_ADMIN_PASSWORD=$(kubectl get secrets --namespace <NAMESPACE> everest-secrets-<CLUSTER_NAME> -o template='{{ "{{"}}.data.MONGODB_CLUSTER_ADMIN_PASSWORD | base64decode {{"}}"}}{{"{{"}}"\n"{{"}}"}}')
```

Once you establish a connection with this user, you will have the ability to execute the commands `sh.enableSharding()` and `sh.shardCollection()`. 

   - The `sh.enableSharding()` command allows you to enable sharding for a specific database. 
    
    - The `sh.shardCollection()` command is used to partition a collection across multiple shards, optimizing data distribution and query performance.


## Fine-tuning your sharding setup

Here are few recommendations to fine-tune your setup for MongoDB sharding.

- **Scalability**: To enhance your cluster's scalability, consider increasing the number of shards, based on your anticipated load.

- **Fault tolerance**: To ensure fault tolerance, MongoDB uses replica sets. This ensures that if one server fails, another replica can take over without impacting the cluster’s availability.

- **Redundancy**: It is recommended that the nodes be deployed in a MongoDB sharded cluster as replica sets of 3 or 5 members. This is important for maintaining high availability and ensuring the sharded cluster continues functioning even if some servers fail.

