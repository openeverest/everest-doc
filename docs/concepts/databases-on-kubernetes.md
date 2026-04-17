# Running Databases on Kubernetes

Kubernetes was designed to orchestrate stateless processes. It knows when a pod is running or dead — but it knows nothing about replication lag, database deadlocks, or whether a replica has diverged from its primary. A pod can show `Running` while the engine is stuck replaying a corrupted Write-Ahead Log. Restarting it is not recovery; it is a gamble with your RTO.

This page covers the five areas you must get right when running production databases on Kubernetes, and explains how OpenEverest addresses each one.

## 1. Availability

### Kubernetes is data-blind

Kubernetes knows when a pod is running or has crashed. It does not know whether a database primary is accepting writes, how far a replica has fallen behind, or whether a WAL stream has stalled. For stateless applications, pod status is enough to make routing decisions. For databases, it is not.

This is where Database Operators close the gap. Instead of relying on Kubernetes to detect and react to a database failure, operators embed health intelligence directly into the platform: they monitor replication lag, track engine state, and orchestrate failover at the database layer — independently of pod scheduling.

### Engine-native replication

Databases can synchronize state across replicas in two fundamentally different ways:

- **Storage-level replication** — block storage volumes are mirrored at the infrastructure layer. The database engine is unaware of the topology.
- **Application-level replication** — the database engine itself streams changes between instances using its own replication protocol.

OpenEverest relies on application-level, engine-native replication. Each database engine ships with battle-tested replication capabilities built directly into the server process. Because each replica streams changes continuously and maintains its own dedicated `PersistentVolume`, it is always ready to be promoted — no volume re-attach, no data copy, no waiting for Kubernetes to reschedule a pod.

When a primary shows signs of failure, the operator promotes a healthy replica at the database layer. The application is re-routed to the new leader in seconds, well before pod rescheduling would even complete.

### Shared-nothing architecture

High availability is ultimately a physical problem. The guiding principle for production database topology on Kubernetes is **shared nothing**: every database instance in a cluster should run on a dedicated node, use its own storage volume, and reside in a separate failure domain. No two instances should share any part of the failure domain.

This means:

- **Separate nodes** — `podAntiAffinity` rules prevent the primary and replicas from landing on the same physical host. A single node failure takes out at most one member of the cluster.
- **Separate storage** — each instance uses its own `PersistentVolume`, not a shared volume. This is what makes engine-native failover fast: there is no volume to re-attach.
- **Separate failure domains** — on cloud providers, this means spreading instances across availability zones (AZs): independent data centers within a region with separate power, networking, and cooling. On-premises, the equivalent is separate physical racks or rooms with independent power feeds. `topologySpreadConstraints` distribute the cluster across a minimum of three such domains. If one goes dark, the remaining instances retain quorum and continue serving traffic.

!!! note "What OpenEverest does"
    Anti-affinity rules and topology spread constraints are applied automatically when you provision a database cluster. You do not need to write scheduling YAML manually.

---

## 2. Compute

### Why "Burstable" QoS is dangerous

In stateless workloads, setting a low memory request with a higher limit is common practice — it allows bin-packing pods onto nodes efficiently. For databases, this is a dangerous misconfiguration.

When memory requests are lower than limits, Kubernetes assigns the Pod a **Burstable** QoS class. Under node memory pressure, the Linux OOM killer targets Burstable processes before Guaranteed ones. Your database process — with its large in-memory caches — becomes an OOM kill candidate exactly when the host is under heavy load, which is also when you can least afford database downtime.

### Guaranteed QoS: requests == limits

The correct setting for any database pod is `requests == limits` for both CPU and memory. This signals to the Linux kernel that the process is critical, giving it the **Guaranteed** QoS class and protecting it from OOM evictions.

```yaml
resources:
  requests:
    memory: "8Gi"
    cpu: "2"
  limits:
    memory: "8Gi"
    cpu: "2"
```

### The 20% overhead rule

The memory value you configure should be ~20% higher than the database engine's internal cache size (for example, PostgreSQL's `shared_buffers`). This headroom is not waste — it is consumed by sidecar containers (backup agents, monitoring exporters), the OS page cache, and background operations like VACUUM. Getting this wrong produces random OOM kills during heavy index scans or maintenance windows.

!!! note "What OpenEverest does"
    OpenEverest sets Guaranteed QoS by default and automatically calculates the 20% memory overhead when you specify the database cache size during provisioning.

---

## 3. Storage

Storage is where most Kubernetes database problems originate. It is also the primary cost driver. Understanding the three storage tiers helps you match your workload requirements to the right choice.

### Tier 1 — S3: the foundation

Before choosing a disk type, establish your off-cluster backup stream. In Kubernetes, volumes are tied to the cluster's lifecycle. A region failure or a corrupted volume can make a "Persistent" Volume anything but persistent.

Following the 3-2-1 rule (3 copies, 2 media types, 1 off-site), OpenEverest streams WAL logs and snapshots to S3-compatible object storage continuously. This provides point-in-time recovery and region-level disaster protection regardless of which disk tier you choose for primary storage.

### Tier 2 — Network-attached storage (EBS/PD): the default

For most production workloads, network-attached block storage (AWS EBS, GCP Persistent Disk, Azure Disk) is the right choice. It is decoupled from the node lifecycle, straightforward to snapshot, and resilient by default.

The trade-off is latency and IOPS cost. As data volume and transaction rates grow, you will hit the network IOPS ceiling. At that point, the cost of Provisioned IOPS often exceeds the compute cost itself.

### Tier 3 — Local NVMe: when latency is non-negotiable

For high-frequency transaction processing or real-time analytics, local NVMe bypasses the network entirely. Expect 500k+ IOPS and sub-millisecond latency at 40-60% lower storage cost than equivalent Provisioned IOPS on network disks.

The trade-off is that a node failure means the data on that NVMe is gone. OpenEverest mitigates this at the application layer: synchronous replication across nodes running on separate NVMe devices means losing a node is a re-sync event, not a data loss event.

### Decision matrix

| Workload | Recommended storage |
|---|---|
| Development / test | EBS/PD (Standard) |
| General production | EBS/PD + S3 continuous backup |
| High-frequency transactions | Local NVMe + synchronous replication |
| Real-time analytics / AI | Local NVMe + S3 streams |

!!! note "What OpenEverest does"
    OpenEverest supports all three tiers through Kubernetes `StorageClass` selection. S3 backup streaming is configured at provisioning time and is independent of the primary storage choice.

---

## 4. Cost management

### The zombie database problem

Databases are typically always-on expenses. Dev and test databases run overnight, over weekends, and long after the feature branch they were created for has been merged - burning compute and IOPS with zero connections.

### Scale to zero

OpenEverest can integrate with KEDA (Kubernetes Event-Driven Autoscaling) to monitor connection activity. For non-production environments, KEDA scales the database to zero replicas after a configurable idle period. Compute and IOPS billing stops immediately. When a new connection arrives, the database wakes and resumes in seconds.

A typical dev database running 24/7 at ~$2,100/month can drop to ~$430/month when limited to active working hours. Across a fleet of 50 dev databases, the annual savings are significant.

### VPA rightsizing

Over-provisioning is equally expensive. Rather than guessing resource requirements upfront, use the Vertical Pod Autoscaler (VPA) in recommendation mode to observe actual usage patterns and suggest right-sized `requests` and `limits`.

Since Kubernetes 1.27, in-place resource updates allow VPA to resize pods without a restart, eliminating the last friction point in keeping resource allocations accurate.

!!! note "What OpenEverest does"
    OpenEverest can be integrated with KEDA for scale-to-zero in non-production namespaces. OpenEverest surfaces VPA recommendations through its monitoring integration so you can rightsize clusters without manual `kubectl` inspection.

---

## 5. Security

### Namespace isolation

Kubernetes namespaces provide logical boundaries between environments and teams. OpenEverest provisions each managed namespace with RBAC policies scoped to that namespace, ensuring teams can only access the databases they own. Resource quotas prevent any single namespace from consuming disproportionate cluster resources.

A typical layout:

| Namespace | Access level |
|---|---|
| `production` | Strict RBAC, limited to ops team |
| `staging` | Team-level access, quota-capped |
| `dev` | Developer self-service, auto-expiry |

### Default network isolation

By default, OpenEverest creates database services as `ClusterIP` — internal to the cluster, not reachable from outside. External exposure requires an explicit choice (LoadBalancer or NodePort) and is never the default. OpenEverest also applies `NetworkPolicy` rules that restrict ingress to the database pods to only the application namespaces that need it.

Avoid IP-based whitelisting: pod IPs change with every reschedule. Use `NetworkPolicy` with label selectors instead.

### Encrypted by default

There are two encryption layers to enforce:

**Client-to-database (TLS 1.3):** All connections from applications and users to the database proxy are encrypted. OpenEverest can integrate with `cert-manager` for automatic certificate issuance and rotation. Unencrypted connections are rejected by default.

**Replica-to-replica (SSL replication):** Traffic between primary and replicas is encrypted even within the same cluster. Lateral movement from a compromised pod should never yield plaintext replication streams.

#### Security checklist

- [ ] Databases in dedicated namespaces with RBAC per team
- [ ] `ClusterIP` service type by default
- [ ] `NetworkPolicy` restricts ingress to known application namespaces
- [ ] TLS 1.3 for all client connections
- [ ] Encrypted replication between cluster members
- [ ] Certificates managed and rotated automatically by `cert-manager` (when integrated)

!!! note "What OpenEverest does"
    Namespace isolation, ClusterIP services, NetworkPolicies, and TLS are configured automatically at provisioning time. `cert-manager` can be integrated to handle the full certificate lifecycle.
