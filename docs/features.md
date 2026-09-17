# Features

Some of the key features of OpenEverest are:

## Deployments

- **Multi-cloud deployments**: OpenEverest enables you to transcend the limitations of a single cloud provider. Deploy your databases in the Kubernetes cluster, to ensure redundancy, high availability, and data sovereignty.
- **Multiple open source database support**: OpenEverest supports multiple database technologies and can be hosted on any Kubernetes infrastructure, in the cloud or on-premises.
- **Private deployments and DIY**: OpenEverest provides complete control over your database environment, ensuring the privacy and security of your data. Private deployments enable you to customize every aspect of your database setup using a do-it-yourself approach.

## Database provisioning

* **[Horizontal and vertical scaling](use/scaling.md)**: OpenEverest enables you to adapt to your application's demands using horizontal scaling for multi-node deployments or vertical scaling for single-node setups. With OpenEverest, you have the power to manage resources and ensure your databases are performing at their best.
* **Database storage class support**: OpenEverest tailors your storage needs efficiently by leveraging its database storage class support. Furthermore, it allocates resources efficiently while maintaining optimal performance and cost.
* **[Disaster recovery capabilities](backups_and_restore/createBackups/CreateOnDemand.md)**: OpenEverest prioritizes data protection with a comprehensive disaster recovery suite. Create on-demand backups, seamlessly restore existing databases, or create new ones from backups to ensure that your data is protected.
* **[Advanced configuration options](use/db_engine_config.md)**: OpenEverest gives fine-grained control of your database environment with advanced configuration features. You can manage external access permissions and optimize your database engine configuration.

## Database management

* **[Resource allocation flexibility](use/scaling.md#vertical-scaling)**: You can handle changing workloads by adjusting your resource allocations on the fly. Whether you need more processing power or memory, OpenEverest's vertical scaling has you covered.

## Administrative

* **[Database monitoring](use/monitor_endpoints.md)**: Gain insights into performance metrics, query analysis, and other important functions through the observability tooling that each provider exposes. Providers integrate their own monitoring solution — for example, Percona providers support Percona Monitoring and Management (PMM).
