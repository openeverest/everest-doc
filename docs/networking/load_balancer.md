# Load balancer configuration in OpenEverest

## Overview

Provisioning external access to Kubernetes clusters can be challenging, since cloud providers like AWS, GCP, and Azure each have their own annotations and configurations for load balancers. As a result, users often have to manually adjust settings for each environment, leading to a lack of a unified approach.

## Why use load balancer?

OpenEverest simplifies the process by enabling administrators to define reusable load balancer configurations. This includes cloud provider-specific settings that can be applied consistently across clusters, ensuring:

- Consistency across cloud and on-prem environments

- Reduced manual effort when provisioning external access

- Flexibility to support multiple cloud providers with a unified approach

- Dynamic value injection using Go templating for cluster-specific configurations



## Understanding some key terms

| **Term**                  | **Definition**                                                                                                                                            |
| ------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **ClusterIP** | In Kubernetes, a service type that exposes an application on a virtual IP address within the cluster. |
| **LoadBalancer**| In Kubernetes, a service type that exposes your application to the internet using a cloud provider’s load balancing infrastructure.|
| **Load balancer Config**  | A preset in OpenEverest that contains a set of key-value pairs representing annotations applied to the appropriate LoadBalancer. Supports Go templating for dynamic value injection based on the target DatabaseCluster.|
| **OpenEverest user** | A user of OpenEverest who manages database clusters. |
| **OpenEverest admin** | A user with full permissions to configure and maintain OpenEverest.|

## Use cases for load balancer configuration

### Standardized configurations

Reuse a predefined `LoadBalancerConfig` across multiple clusters to ensure consistent networking behavior. This is useful for organizations with strict compliance or networking policies.


### Quick environment provisioning

Enable Database-as-a-Service (DBaaS) users to quickly create clusters with external access in a single step, eliminating the need for platform teams to manually patch services.

### Dynamic cluster-specific configurations

Use Go templating in load balancer annotations to inject cluster-specific values automatically. This is particularly useful for integrations with tools like ExternalDNS, where each database cluster requires unique values (such as hostnames) derived from the cluster's metadata or specifications.


