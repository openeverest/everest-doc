# What's new in OpenEverest 2.0.0 Developer Preview 2

!!! warning "Developer Preview — not for production"
    OpenEverest 2.0.0 Developer Preview 2 is an early-look release and is **not feature-complete**. It is intended for testing and feedback only.

    There is **no supported upgrade path** from Developer Preview 1, and none from v1. Install Developer Preview 2 on a fresh cluster. v1 and v2 have fundamentally different architectures and data models, and running them side-by-side in the same cluster is not supported.

    This release contains **breaking changes to every CRD, to the HTTP API, and to the provider-runtime SDK**. Manifests, providers and plugins written against Developer Preview 1 will not work and must be updated — see [Breaking changes](#breaking-changes-for-provider-and-plugin-developers) for the full mapping. No migration path is provided at this stage of the preview. This is deliberate, so that the API can be corrected before General Availability.

**New to OpenEverest?** Get started with our [Quickstart Guide](https://openeverest.io/documentation/2.0.0-dev.2/quick-install.html).

[Read the blog post and watch the video](https://openeverest.io/blog/v2-developer-preview-release/) introducing the v2 architecture.

---

## 🌟 Release highlights

### `everestctl` becomes a real day-2 CLI for v2

Developer Preview 1 shipped v2 with a UI and an API, but no command-line story. Developer Preview 2 closes that gap: the whole v2 resource model is now drivable from `everestctl`, with consistent `--namespace/-n`, `--wait`, and JSON output across commands.

```bash
everestctl auth login --server https://everest.example.com
everestctl provider list
everestctl backup-class list
everestctl instance create --preset small-mongodb -n my-namespace --wait
everestctl instance status my-db -n my-namespace --watch
everestctl backup create --instance my-db --storage s3-backups -n my-namespace
everestctl restore create --instance my-db --backup my-db-20260812 -n my-namespace
everestctl backup-storage create --name s3-backups -n my-namespace
everestctl instance delete my-db -n my-namespace
```

New command groups: `auth`, `instance`, `backup`, `restore`, `backup-storage` (alias `bs`), `backup-class` (alias `bc`), `provider` (alias `prov`).

!!! note
    `everestctl` is **not shipped as a binary with this release**. `everestctl install` and `everestctl uninstall` still target v1 and cannot install v2, so publishing the binary would be misleading. To try the commands above against an existing v2 installation, build it from source:

    ```bash
    git clone --branch v2.0.0-dev.2 https://github.com/openeverest/openeverest.git
    cd openeverest
    make build-cli   # produces bin/everestctl
    ```

    Installation remains Helm-only for this preview.

### Point-in-time recovery, end to end

PITR is now a first-class part of the v2 model rather than a modifier on a backup reference.

- **Recovery window.** `Instance.status.backup.storages[].pitr` reports `earliestRestorableTime`, `latestRestorableTime`, `state` (`Available`/`Unavailable`), plus `reason` and `message`. The window is authoritative: providers truncate forward at a discontinuity and under-report rather than advertise a point that cannot be restored.
- **Point-in-time as a data source.** `dataSource.type` now discriminates between `Backup` and `PointInTime`. A point-in-time recovery no longer has to name a `Backup` CR it has nothing to do with.
- **Restore into a new database.** Because `dataSource` is the same shared type on `Instance.spec.dataSource` and `Restore.spec.dataSource`, you can seed a brand-new instance from a point in time on another instance's stream via `pointInTime.source.instanceRef`.

Providers opt in by implementing the new `controller.InstanceBackupStatusReporter` interface; the runtime writes the aggregated per-storage window to `instance.status.backup` after every successful `Sync`.

### Instance presets

`InstancePreset` is a new namespaced CRD that captures a ready-to-use instance shape — provider, version, topology, component sizing and storage class. Users pick a preset instead of filling in a form; power users still get the full parameter surface.

- New API: `GET /v1/clusters/{cluster}/instance-presets`, `.../{name}`, and `.../{name}/resolve` (which returns the fully materialised `Instance` spec a preset would produce, including the resolved default storage class).
- New CLI: `everestctl instance create --preset <name>`.

### Secrets, ConfigMaps, and schema-driven credentials

Providers can now declare, in their provider definition, the secrets and config maps they expect (`spec.secrets`, `spec.configMaps` on `Provider`), each with a parameters schema. The API server exposes managed CRUD for those objects and validates payloads against the declared schema before they reach Kubernetes:

- `POST|GET /v1/clusters/{cluster}/namespaces/{namespace}/secrets` and `GET|DELETE .../secrets/{name}`
- `POST|GET /v1/clusters/{cluster}/namespaces/{namespace}/config-maps` and `GET|DELETE .../config-maps/{name}`

The provider-runtime gained matching validation webhooks (`secret_validation.go`, `config_map_validation.go`).

### Auditable event stream with replay

`GET /v1/events` no longer silently drops events across a reconnect. Every event now carries a monotonic sequence and an epoch identifying the publishing process; `since` is `"<epoch>:<seq>"`, and an aged-out cursor or epoch mismatch returns `410 Gone`, which tells the client to resync rather than quietly lose data. Events also carry the acting user, laying the groundwork for auditing.

### Breaking changes for provider and plugin developers

The API surface was normalised in four sweeps. All of them are breaking, and all of them are intentional at preview stage.

**1. Cross-resource references are uniform.** Every reference is now an object (`{name: ...}`) drawn from `api/common/v1alpha1` — `ObjectRef`, `SecretRef`, `TypedObjectRef` — instead of a bare string or a `corev1.LocalObjectReference`.

| Resource | Before | After |
|---|---|---|
| `Instance.spec` | `provider: <string>` | `providerRef.name` |
| `Instance.spec.backup` | `classRef` (custom type) | `classRef` (`ObjectRef`) |
| `Instance.spec.backup.storages[]` | `name` + `storageRef` + `main` | `storageRef.name` only (key), `main` inferred from `pitr` |
| `Instance.status` | `connectionSecretRef` (`LocalObjectReference`) | `connectionSecretRef` (`*SecretRef`) |
| `Instance.status.components[]` | `pods` | `podRefs` |
| `Backup.spec` | `instanceName`, `backupClassName`, `storageName` | `instanceRef`, `classRef`, `storageRef` |
| `Backup.status` / `Restore.status` | `jobName`, `operatorBackupRef` (`TypedLocalObjectReference`) | `jobRef`, `operatorBackupRef` (`*TypedObjectRef`) |
| `Restore.spec` | `instanceName` | `instanceRef` |
| `BackupStorage.spec.s3` | `credentialsSecretName` | `credentialsSecretRef.name` |
| `MonitoringConfig.spec.pmm` | `credentialsSecretName` | `credentialsSecretRef.name` |

**2. `config` / `customSpec` / `global` are all `parameters`.** One name for provider-defined free-form configuration everywhere, and one envelope (`common.ParametersSchema`, i.e. `parametersSchema.openAPIV3Schema`) for declaring its schema.

| Before | After |
|---|---|
| `Instance.spec.global` | `Instance.spec.parameters` |
| `Instance.spec.topology.config` | `Instance.spec.topology.parameters` |
| `Instance.spec.components[].config` | `Instance.spec.components[].parameters` |
| `Backup.spec.config` / `Restore.spec.config` | `.spec.parameters` |
| `Provider` `globalConfigSchema` / `customSpecSchema` / `configSchema` | `parametersSchema` |
| `Provider` `secrets[].openAPIV3Schema` | `secrets[].parametersSchema` |
| `BackupClass` `config` / `restoreConfig` / `pitrConfigSchema` | `parametersSchema` / `restoreParametersSchema` / `pitrParametersSchema` |
| SDK `Context.DecodeConfig()` etc. | `DecodeParameters()`, `DecodeTopologyParameters()`, `DecodeComponentParameters()` |

`ParametersSchema.Validate` rejects a non-nil payload when no schema is declared, so components whose parameters are actually used **must** declare `parametersSchema` in `definition/provider.yaml`. Secret and config map definitions are the exception: an omitted `parametersSchema` there skips validation entirely rather than rejecting the payload.

**3. `dataSource` is a discriminated union.**

```yaml
# before
dataSource:
  backupName: my-backup
  pitr:
    type: date            # or "latest"
    date: "2026-08-12T10:00:00Z"

# after
dataSource:
  type: Backup            # or PointInTime
  backup:
    backupRef:
      name: my-backup
# ...or...
dataSource:
  type: PointInTime
  pointInTime:
    source:
      instanceRef: {name: source-db}   # optional; defaults to the target Instance
      storageRef:  {name: s3-backups}  # required, must have pitr.enabled=true
    recoveryTarget: date               # or "latest"
    date: "2026-08-12T10:00:00Z"       # RFC 3339 with an explicit UTC offset
```

`PITRType` is renamed `RecoveryTarget`. The flat `status.backup.storages[].latestRestorableTime` is replaced by the nested `pitr` block described above.

**4. `BackupClass` job specs are grouped.** `job` and `restoreJob` become `job.backup` and `job.restore`, mirroring the discriminated-union shape used by the other execution modes.

**Plugin API group renamed.** `plugin.openeverest.io/v1alpha1` is now `extensions.openeverest.io/v1alpha1`, and the `PluginInstallation` kind is now `InstalledExtension`. `Plugin.spec.kubePermissions` was removed — plugins ship their own RBAC in their Helm chart, and per-namespace plugin scoping was dropped.

**Typed metadata in generated clients.** CRD `metadata` is no longer an untyped map in the OpenAPI spec: a shared `ObjectMeta` schema is emitted and `$ref`'d by every kind, with `x-go-type` mapping the Go models onto `metav1.ObjectMeta`. Go and TypeScript API consumers get typed metadata and lose the string-map helpers.

**Provider upgrade catalog.** `Provider.spec.release` carries the provider's own `version` and `minUpgradableFrom`; `ComponentVersion` gained `deprecated` and `removedInVersion` to give users a deprecation runway.


---

## 📝 Changes

### Added

- **`everestctl` gained a full v2 command surface** — the whole v2 resource model is now drivable from the CLI, with consistent `--namespace/-n`, `--wait` and JSON output:

  | Group | Subcommands | PRs |
  |---|---|---|
  | `auth` | `login`, `logout` | [#2372](https://github.com/openeverest/openeverest/pull/2372), [#2443](https://github.com/openeverest/openeverest/pull/2443) |
  | `instance` | `create`, `list`, `status`, `delete` | [#2480](https://github.com/openeverest/openeverest/pull/2480), [#2563](https://github.com/openeverest/openeverest/pull/2563), [#2519](https://github.com/openeverest/openeverest/pull/2519), [#2762](https://github.com/openeverest/openeverest/pull/2762) |
  | `backup` | `create`, `list`, `delete` | [#2599](https://github.com/openeverest/openeverest/pull/2599), [#2597](https://github.com/openeverest/openeverest/pull/2597), [#2863](https://github.com/openeverest/openeverest/pull/2863) |
  | `restore` | `create`, `list`, `delete` | [#2656](https://github.com/openeverest/openeverest/pull/2656), [#2648](https://github.com/openeverest/openeverest/pull/2648), [#2847](https://github.com/openeverest/openeverest/pull/2847) |
  | `backup-storage` (`bs`) | `create`, `list`, `delete` | [#2671](https://github.com/openeverest/openeverest/pull/2671), [#2573](https://github.com/openeverest/openeverest/pull/2573), [#2765](https://github.com/openeverest/openeverest/pull/2765) |
  | `backup-class` (`bc`) | `list` | [#2565](https://github.com/openeverest/openeverest/pull/2565) |
  | `provider` (`prov`) | `list` | [#2564](https://github.com/openeverest/openeverest/pull/2564) |

  Cross-cutting flags: `instance create --preset` ([#2518](https://github.com/openeverest/openeverest/pull/2518)) and `--wait` ([#2600](https://github.com/openeverest/openeverest/pull/2600)), `instance status --watch/-w` ([#2519](https://github.com/openeverest/openeverest/pull/2519)), instance print columns ([#2594](https://github.com/openeverest/openeverest/pull/2594)).

- `InstancePreset` CRD, resolution API and UI support, with default storage class resolution ([#2407](https://github.com/openeverest/openeverest/pull/2407), [#2454](https://github.com/openeverest/openeverest/pull/2454)).
- Point-in-time recovery: validation and restore gating, per-storage recovery-window status, point-in-time data sources, and restore into a new database ([#2562](https://github.com/openeverest/openeverest/pull/2562), [#2848](https://github.com/openeverest/openeverest/pull/2848)).
- `controller.InstanceBackupStatusReporter` in provider-runtime, for providers to publish per-storage backup observability data ([#2848](https://github.com/openeverest/openeverest/pull/2848)).
- Secret management API with provider-declared schemas and runtime validation ([#2534](https://github.com/openeverest/openeverest/pull/2534)).
- ConfigMap API with provider-declared schemas and runtime validation ([#2541](https://github.com/openeverest/openeverest/pull/2541)).
- API auth token management with refresh tokens, and matching UI integration ([#2361](https://github.com/openeverest/openeverest/pull/2361), [#2362](https://github.com/openeverest/openeverest/pull/2362)).
- Audit events carrying the acting user ([#2415](https://github.com/openeverest/openeverest/pull/2415)).
- Replay buffer on `GET /v1/events` so reconnecting subscribers do not silently lose events ([#2591](https://github.com/openeverest/openeverest/pull/2591)).
- Provider upgrade catalog: `spec.release.version`, `spec.release.minUpgradableFrom`, and `deprecated` / `removedInVersion` on component versions ([#2604](https://github.com/openeverest/openeverest/pull/2604)).
- `affinity` on `Instance.spec.components[]` ([#2364](https://github.com/openeverest/openeverest/pull/2364)) and `service` / `loadBalancerService` configuration on components.
- Plugin hub in the v2 UI, with plugin-declared permissions ([#2367](https://github.com/openeverest/openeverest/pull/2367), [#2424](https://github.com/openeverest/openeverest/pull/2424)).
- Tiles view for creating instances in the UI ([#2453](https://github.com/openeverest/openeverest/pull/2453)).
- Backup/Restore reference validation at the API layer on create ([#2667](https://github.com/openeverest/openeverest/pull/2667)).
- `Instance` reconcile-request helper in provider-runtime ([#2515](https://github.com/openeverest/openeverest/pull/2515)).
- Finalizer on `BackupStorage` plus end-to-end tests ([#2327](https://github.com/openeverest/openeverest/pull/2327), [#2353](https://github.com/openeverest/openeverest/pull/2353)).
- RBAC fuzz tests ([#2581](https://github.com/openeverest/openeverest/pull/2581)), Dependabot coverage for v2 ([#2673](https://github.com/openeverest/openeverest/pull/2673)), CodeQL scanning of `release-2.0` and workflow files ([#2753](https://github.com/openeverest/openeverest/pull/2753)), and SBOM attestation for release artifacts ([#2858](https://github.com/openeverest/openeverest/pull/2858)).
- Self-serve issue triage and `/assign` process ([#2764](https://github.com/openeverest/openeverest/pull/2764), [#2796](https://github.com/openeverest/openeverest/pull/2796)).

### Changed & Improved

- **Breaking API changes.** Every CRD, the HTTP API and the provider-runtime SDK changed shape. Full before/after mapping in [Breaking changes](#breaking-changes-for-provider-and-plugin-developers):

  - Cross-resource references unified onto `common.ObjectRef` / `SecretRef` / `TypedObjectRef` ([#2570](https://github.com/openeverest/openeverest/pull/2570))
  - `config` / `customSpec` / `global` renamed to `parameters`; schemas wrapped in `common.ParametersSchema`, including secret definitions on `Provider.spec.secrets[]` ([#2583](https://github.com/openeverest/openeverest/pull/2583), [#2616](https://github.com/openeverest/openeverest/pull/2616))
  - `dataSource` restructured into a `Backup` / `PointInTime` discriminated union; `PITRType` → `RecoveryTarget`; flat `latestRestorableTime` → nested `pitr` status ([#2848](https://github.com/openeverest/openeverest/pull/2848))
  - `BackupClass` job specs grouped as `job.backup` / `job.restore` ([#2547](https://github.com/openeverest/openeverest/pull/2547))
  - Plugin API group renamed to `extensions.openeverest.io`; `PluginInstallation` → `InstalledExtension`; `spec.kubePermissions` removed ([#2367](https://github.com/openeverest/openeverest/pull/2367), [#2424](https://github.com/openeverest/openeverest/pull/2424))
  - CRD `metadata` typed as a shared `ObjectMeta` schema in the OpenAPI spec and generated clients ([#2652](https://github.com/openeverest/openeverest/pull/2652))

- **Breaking:** `POST /v1/auth/revoke` is now unauthenticated ([#2448](https://github.com/openeverest/openeverest/pull/2448)).
- Shared HTTP-poll classification extracted into `pkg/cli/wait`, so every `--wait`/`--watch` command behaves the same on transient errors ([#2808](https://github.com/openeverest/openeverest/pull/2808)).
- UI upgraded from MUI 5 to MUI 7 ([#2548](https://github.com/openeverest/openeverest/pull/2548), [#2550](https://github.com/openeverest/openeverest/pull/2550), [#2569](https://github.com/openeverest/openeverest/pull/2569)).
- Circular dependency removed from the UI and a strict ESLint rule added to prevent regressions ([#2417](https://github.com/openeverest/openeverest/pull/2417)).
- `golangci-lint` upgraded to v2, and CI now fails on unformatted Go code ([#2535](https://github.com/openeverest/openeverest/pull/2535), [#2507](https://github.com/openeverest/openeverest/pull/2507)).
- GitHub Actions pinned to full-length commit SHAs and workflow token permissions scoped down ([#2751](https://github.com/openeverest/openeverest/pull/2751), [#2752](https://github.com/openeverest/openeverest/pull/2752)).
- Tilt development environment reworked for core development, with `InstancePreset`, plugin-hub and PXC provider support ([#2483](https://github.com/openeverest/openeverest/pull/2483), [#2439](https://github.com/openeverest/openeverest/pull/2439), [#2508](https://github.com/openeverest/openeverest/pull/2508), [#2346](https://github.com/openeverest/openeverest/pull/2346)).

### Fixed

- provider-runtime reported `/readyz` as ready before the controller manager and its cache-backed client were usable ([#2611](https://github.com/openeverest/openeverest/pull/2611)).
- Provider status messages were silently dropped instead of surfacing on the `Instance` ([#2366](https://github.com/openeverest/openeverest/pull/2366)).
- Nil connection reference caused a panic when an instance had no connection secret yet ([#2590](https://github.com/openeverest/openeverest/pull/2590)).
- Init containers were counted incorrectly when computing instance resource totals ([#2719](https://github.com/openeverest/openeverest/pull/2719)).
- `MonitoringConfig` finalizers and secrets were not cleaned up reliably on delete ([#2408](https://github.com/openeverest/openeverest/pull/2408)).
- The server did not handle `SIGTERM`, so pods were killed instead of shutting down gracefully ([#2820](https://github.com/openeverest/openeverest/pull/2820)).
- OIDC well-known configuration fetch had no HTTP timeout ([#2725](https://github.com/openeverest/openeverest/pull/2725)).
- RBAC informer start errors were swallowed, and the ConfigMap adapter's Kubernetes calls had no timeout ([#2830](https://github.com/openeverest/openeverest/pull/2830), [#2822](https://github.com/openeverest/openeverest/pull/2822)).
- Nil pointer panic when the JWT private key PEM was invalid ([#2851](https://github.com/openeverest/openeverest/pull/2851)).
- Token blocklist update errors were not propagated ([#2769](https://github.com/openeverest/openeverest/pull/2769)).
- The arm64 server image shipped amd64 binaries ([#2606](https://github.com/openeverest/openeverest/pull/2606)).
- Helm chart dependency builds failed without a configured registry client ([#2607](https://github.com/openeverest/openeverest/pull/2607)).
- Editing a backup storage location and the empty state of scheduled backups both errored in the UI ([#2320](https://github.com/openeverest/openeverest/pull/2320), [#2319](https://github.com/openeverest/openeverest/pull/2319)).
- Database credentials cache was not scoped by namespace, and the RBAC route guard was not reactive ([#2173](https://github.com/openeverest/openeverest/pull/2173)).
- The upgrade dialog reopened after being dismissed ([#2228](https://github.com/openeverest/openeverest/pull/2228)).
- Multiline values overflowed the Database Summary preview ([#2090](https://github.com/openeverest/openeverest/pull/2090)).
- External links in the UI were missing `noreferrer`, and the login page lost spacing between the intro text and the community links ([#2857](https://github.com/openeverest/openeverest/pull/2857), [#2902](https://github.com/openeverest/openeverest/pull/2902)).
- `everestctl` flag and message fixes, including `-n` as a shorthand for `--namespace` and the `namesapce` typo ([#2856](https://github.com/openeverest/openeverest/pull/2856), [#2795](https://github.com/openeverest/openeverest/pull/2795), [#2444](https://github.com/openeverest/openeverest/pull/2444), [#2252](https://github.com/openeverest/openeverest/pull/2252), [#2196](https://github.com/openeverest/openeverest/pull/2196)).
- Documentation typos, broken links, stale extension CLI examples, the EKS z1d EBS volume limit, and the `providerRef.name` plugin filter reference ([#2458](https://github.com/openeverest/openeverest/pull/2458), [#2460](https://github.com/openeverest/openeverest/pull/2460), [#2622](https://github.com/openeverest/openeverest/pull/2622), [#2625](https://github.com/openeverest/openeverest/pull/2625), [#2778](https://github.com/openeverest/openeverest/pull/2778), [#2818](https://github.com/openeverest/openeverest/pull/2818), [#2624](https://github.com/openeverest/openeverest/pull/2624), [#2840](https://github.com/openeverest/openeverest/pull/2840)).

---

## 🚀 Installing OpenEverest 2.0.0 Developer Preview 2

OpenEverest v2 is installed via Helm. You will need a running Kubernetes cluster and `helm` installed. `everestctl install` still targets v1 and must not be used for v2.

Install the OpenEverest core:

```bash
helm repo add openeverest https://openeverest.github.io/helm-charts/
helm repo update
helm install everest-core openeverest/openeverest \
  --devel \
  --version "2.0.0-dev.2" \
  --namespace everest-system \
  --create-namespace
```

Retrieve the admin password:

```bash
kubectl get secret everest-accounts -n everest-system \
  -o jsonpath='{.data.users\.yaml}' | base64 --decode | yq '.admin.passwordHash'
```

Access the UI:

```bash
kubectl port-forward svc/everest 8080:8080 -n everest-system
```

Providers are released independently of the core. Install the ones you need from their own repositories, and pick a version built against `2.0.0-dev.2` — provider releases published before this one target the previous API and will not reconcile.

- [MongoDB provider](https://github.com/openeverest/provider-percona-server-mongodb)
- [MySQL provider](https://github.com/openeverest/provider-percona-xtradb-cluster)
- [PostgreSQL provider](https://github.com/openeverest/provider-percona-postgresql)

---

## v1 Support and End-of-Life Timeline

OpenEverest v1 continues to be supported while v2 matures. The transition timeline is as follows:

| Stage | Description |
|-------|-------------|
| **Now** | v2 Developer Preview — testing and feedback. |
| **v2 General Availability** | Expected in a few months. |
| **GA + 3 Months** | v1 enters Maintenance Mode. Security patches and critical bug fixes only; no new features. |
| **GA + 12 Months** | v1 reaches End of Life. No further releases. |

---

## Get involved

Try the Developer Preview, build a provider against the new API, and share your feedback. The project is open to contributions — provider plugins, generic plugins, and core improvements are all welcome.

- [Provider SDK](https://github.com/openeverest/provider-sdk) and the [provider development guide](https://github.com/openeverest/provider-sdk/blob/main/PROVIDER_DEVELOPMENT.md)
- [Generic Plugin Template](https://github.com/openeverest/generic-plugin-template)
- [Generic Plugins Spec](https://github.com/openeverest/specs/blob/main/specs/003-generic-plugins.md)
- [Blog post about the v2 Developer Preview](https://openeverest.io/blog/v2-developer-preview-release/)

---

**Full Changelog**: https://github.com/openeverest/openeverest/compare/v2.0.0-dev.1...v2.0.0-dev.2