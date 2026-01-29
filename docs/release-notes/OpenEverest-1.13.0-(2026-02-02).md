# What's new in OpenEverest 1.13.0

➡️ **New to OpenEverest?** Get started with our [Quickstart Guide](../quick-install.md).

??? info "Expand to unleash the key updates"

    ## 📋 Release summary

    |**#**|**Category**|**Description**|
    |---------|---------------------|---------|
    | **1.**|[Pod Logs Viewer](#pod-logs-viewer)|View real-time logs from database pods directly in the OpenEverest UI for easier troubleshooting and monitoring.|
    | **2.**|[Dynamic value injection in LoadBalancerConfig](#dynamic-value-injection-in-loadbalancerconfig)|Use Go templates in LoadBalancerConfig annotations to dynamically inject cluster-specific values.|
    | **3.**|[OpenEverest Rebranding](#openeverest-rebranding)|Full rebranding from Percona Everest to OpenEverest across all repos and documentation.|
    | **4.**|[Operator upgrades](#operator-updates)|Support for Percona XtraDB Cluster Operator v1.19.0.|
    | **5.**|[New features](#new-features)|Check out the new features introduced in OpenEverest 1.13.0|


## 🌟 Release highlights

=== "📜 Pod Logs Viewer"
    ### View pod logs directly in the UI
    OpenEverest now includes a built-in Pod Logs Viewer, allowing you to view real-time logs from database pods directly in the OpenEverest UI. This feature simplifies troubleshooting and monitoring by eliminating the need to use kubectl or other external tools to access pod logs.

    **Key capabilities:**

    - View logs from any pod in your database cluster
    - Filter logs by container
    - Stream logs in real-time
    - Search and navigate through log history
    - Copy logs for further analysis

    This feature significantly improves the operational experience, making it easier to diagnose issues and monitor database cluster health.

    📘 Related GitHub issue: [#1765](https://github.com/openeverest/openeverest/issues/1765)

=== "🔧 Dynamic value injection in LoadBalancerConfig"
    ### Go templating support for LoadBalancerConfig annotations
    LoadBalancerConfig now supports Go templating in annotation values, enabling dynamic value injection based on the target DatabaseCluster's properties. This powerful feature allows you to create reusable configurations that automatically adapt to each database cluster.

    **Example use case:**
    
    When integrating with ExternalDNS, you can now define a single LoadBalancerConfig that generates unique hostnames for each database cluster:

    ```yaml
    external-dns.alpha.kubernetes.io/hostname: "{{ .ObjectMeta.Namespace }}-{{ .ObjectMeta.Name }}.example.org"
    ```

    When applied to a DatabaseCluster named `my-postgres` in the `production` namespace, this template automatically resolves to:

    ```
    external-dns.alpha.kubernetes.io/hostname: "production-my-postgres.example.org"
    ```

    **Available template fields:**

    - `.ObjectMeta.Name` - Database cluster name
    - `.ObjectMeta.Namespace` - Namespace
    - `.Spec.Engine.Version` - Database engine version
    - Any other field from the DatabaseCluster custom resource

    This eliminates the need to create separate LoadBalancerConfigs for each database cluster, significantly reducing operational overhead.

    📘 For detailed instructions, refer to our [documentation](../networking/load_balancer_config.md).
    
    📘 Related GitHub issue: [#1817](https://github.com/openeverest/openeverest/issues/1817)
    Big thanks to [@Ankit152](https://github.com/Ankit152) for contributing this improvement!

=== "🎨 OpenEverest Rebranding"
    ### Complete rebranding to OpenEverest
    This release marks the official rebranding from Percona Everest to OpenEverest. All repos, documentation, and user interfaces have been updated to reflect the new branding, reinforcing our commitment to open-source principles and community-driven development.

    **What changed:**

    - Product name updated to OpenEverest across all interfaces
    - Documentation updated with new branding and URLs
    - UI refreshed with OpenEverest branding
    - Issue tracking migrated to GitHub Issues exclusively

=== "⚙️ Operators"
    ### Operator updates

    OpenEverest now supports:

    - Percona Operator for MySQL based on Percona XtraDB Cluster (PXC) [v1.19.0](https://docs.percona.com/percona-operator-for-mysql/pxc/ReleaseNotes/Kubernetes-Operator-for-PXC-RN1.19.0.html)

## New features

- [#1765](https://github.com/openeverest/openeverest/issues/1765): OpenEverest now includes a Pod Logs Viewer, allowing users to view real-time logs from database pods directly in the UI for easier troubleshooting and monitoring.

- [#1817](https://github.com/openeverest/openeverest/issues/1817): LoadBalancerConfig annotations now support Go templating for dynamic value injection based on DatabaseCluster properties, enabling reusable configurations that automatically adapt to each cluster. (Thanks [@Ankit152](https://github.com/Ankit152)!)

- OpenEverest now supports Percona XtraDB Cluster Operator v1.19.0.

## 🚀 Ready to Upgrade?

Upgrade to **OpenEverest 1.13.0** to access these new features and improvements.

📖 Explore our [Upgrade section](../upgrade/upgrade_with_helm.md) for the upgrade steps.
