# Telemetry on OpenEverest

Product telemetry fills in the gaps in our understanding of how you actually use OpenEverest, so we can build the best-in-class cloud-native database platform for the open-source community.

Participation in this anonymous program is optional, and you can opt out at any time if you prefer not to share any information. Read The Linux Foundation [privacy policy](https://lfprojects.org/policies/privacy-policy/) to learn more.

Telemetry is **enabled by default**.

## How telemetry is delivered

OpenEverest delivers telemetry through [Scarf](https://about.scarf.sh/), the CNCF-approved telemetry solution used by projects under the CNCF umbrella. Scarf is designed for open-source maintainers and is built to respect end-user privacy: no PII, no IP addresses retained for analytics, and a documented opt-out story.

The transport details are:

| Property | Value |
|---|---|
| Endpoint | `https://openeverest.gateway.scarf.sh/telemetry` (Scarf Gateway event collection) |
| Method | A single HTTP `POST` per reporting tick. **All data is encoded as URL query parameters** — there is no request body. |
| Cadence | One event every `TELEMETRY_INTERVAL` (default `24h`), after a 5-minute startup delay. |
| Failure mode | Telemetry never blocks or fails any user request. Transport errors are logged and forgotten. |

## Disable telemetry

Any one of the following turns telemetry off:

| Mechanism | When it applies |
|---|---|
| `everestctl install --disable-telemetry` | At install time. |
| `everestctl upgrade --disable-telemetry` | At upgrade time. |
| `DISABLE_TELEMETRY=true` env var on the `everest-server` pod | At runtime. |
| `TELEMETRY_URL=""` env var on the `everest-server` pod | At runtime — an empty URL short-circuits the reporter. |
| `DO_NOT_TRACK=1` env var on the `everest-server` pod | At runtime — honored by the underlying Scarf SDK. |

The first two options (`everestctl install/upgrade --disable-telemetry`) also propagate `telemetry=false` to the upstream database engine operators (PXC, PSMDB, PG), suppressing *their* independent telemetry as well.

To disable telemetry at runtime, patch the `everest-server` deployment:

```sh
kubectl -n everest-system patch deployment everest-server --type strategic -p 'spec:
  strategy:
    rollingUpdate:
      maxSurge: 0
      maxUnavailable: 1
    type: RollingUpdate
  template:
    spec:
      containers:
        - name: everest
          env:
          - name: DISABLE_TELEMETRY
            value: "true"'
```

## Enable telemetry

To re-enable telemetry at runtime:

```sh
kubectl -n everest-system patch deployment everest-server --type strategic -p 'spec:
  strategy:
    rollingUpdate:
      maxSurge: 0
      maxUnavailable: 1
    type: RollingUpdate
  template:
    spec:
      containers:
        - name: everest
          env:
          - name: DISABLE_TELEMETRY
            value: "false"'
```

## What is sent

Every event is a flat set of string key/value pairs. There is no nesting, no free-form text, and **no cluster, namespace, or user names**.

### Always-on properties

| Property | Example | Source |
|---|---|---|
| `event` | `heartbeat` | constant |
| `schema_version` | `1` | constant; bumps only on breaking schema changes |
| `everest_version` | `v1.7.0-3-abc1234` | `pkg/version.Version` (ldflags) |
| `build_channel` | `dev` \| `rc` \| `release` | `cmd/config.BuildChannel` (ldflags) |
| `instance_id` | `b3f9...d2a1` (random UUID) | ConfigMap-stored install UUID, generated once at install |

### Database fleet (probe: `db_clusters`)

| Property | Example | Meaning |
|---|---|---|
| `db_namespaces_count` | `3` | Number of namespaces watched by Everest |
| `db_clusters_total` | `12` | Total `DatabaseCluster` count across watched namespaces |
| `engine_pxc_count` | `4` | Per-engine cluster count |
| `engine_psmdb_count` | `5` | Per-engine cluster count |
| `engine_postgresql_count` | `3` | Per-engine cluster count |
| `engine_pxc_versions` | `8.0.32-24,8.0.36-28` | Sorted, deduped, comma-joined engine versions in use |
| `engine_psmdb_versions` | `6.0.9-7,7.0.5-3` | Same |
| `engine_postgresql_versions` | `15.4,16.1` | Same |
| `feat_pitr` | `true` | Any cluster has PITR enabled |
| `feat_scheduled_backups` | `false` | Any cluster has at least one backup schedule |
| `feat_external_access` | `true` | Any cluster exposes a non-internal proxy |
| `feat_data_import_in_use` | `false` | Any cluster was created from a `DataSource` |
| `feat_psp_in_use` | `true` | Any cluster references a `PodSchedulingPolicy` |

### Surrounding resources

| Property | Example | Meaning |
|---|---|---|
| `backup_storages_count` | `2` | Number of `BackupStorage` CRs |
| `monitoring_configs_count` | `1` | Number of `MonitoringConfig` CRs |
| `feat_monitoring_in_use` | `true` | Any monitoring config is present |
| `load_balancer_configs_count` | `0` | Number of `LoadBalancerConfig` CRs |
| `pod_scheduling_policies_count` | `1` | Number of `PodSchedulingPolicy` CRs |
| `split_horizon_dns_configs_count` | `0` | Number of `SplitHorizonDNSConfig` CRs |
| `data_importers_count` | `2` | Number of installed `DataImporter` CRs |
| `data_importers_installed` | `mongo-restore,pg-dump` | Sorted, comma-joined names of installed importers. These are well-known component names shipped with Everest, not user-chosen labels. |

### Authentication posture (probe: `auth`)

| Property | Example | Meaning |
|---|---|---|
| `auth_rbac_enabled` | `true` | Whether the RBAC ConfigMap has `enabled: true` |
| `auth_oidc_configured` | `false` | Whether an OIDC config exists (presence only — no provider, URL, or client ID) |

### Reserved for 2.0 (probe: `providers`)

| Property | Example | Meaning |
|---|---|---|
| `providers_installed` | *(empty in 1.x)* | Reserved name; in 2.0 will enumerate installed `Provider` CR names |

## What is explicitly NOT collected

- No cluster, namespace, secret, or user names
- No database, table, or schema contents
- No backup data, S3 paths, or storage credentials
- No IP addresses, hostnames, or DNS records
- No RBAC policy content (only the on/off bit)
- No OIDC configuration content (issuer, client ID, secrets — none of it)
- No request logs, query logs, or audit trails
- No machine fingerprints beyond the install UUID

## Verifying what is sent

To inspect exactly what your installation would send, you can either:
{.power-number}

1. Set the reporter's log level to `debug` — every send is logged.
2. Point the server at a local capture endpoint by setting `TELEMETRY_URL=http://localhost:9000/`, then run any HTTP echo server to print the query string.
