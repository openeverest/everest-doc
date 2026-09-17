# What's new in OpenEverest 2.0.0 Developer Preview 3

!!! warning "Developer Preview — not for production"
    Not feature-complete; for testing and feedback only.

    **Install on a fresh cluster.** There is no upgrade path from Developer Preview 2, and none from v1 *yet*. Running v1 and v2 side by side in one cluster is not supported.

    **A v1 → v2 migration path is planned and ships with v2 General Availability.** It cannot exist during the preview because the v2 API is still changing and a migration tool needs a stable target. Keep running v1: it stays supported, and you will not be asked to rebuild what you are running by hand.

    **Breaking changes** to the `Backup` and `Instance` CRDs, the HTTP API, the plugin API and the provider-runtime SDK — see [Breaking changes](#breaking-changes-for-provider-and-plugin-developers). Nothing between previews gets a migration path, deliberately, so the API can be corrected before GA.

**New to OpenEverest?** Start with the [Quickstart Guide](https://openeverest.io/documentation/2.0.0-dev.3/quick-install.html), or [read the blog post](https://openeverest.io/blog/v2-developer-preview-release/) on the v2 architecture.

---

## Release highlights

### One-click instances from a preset

Developer Preview 2 added `InstancePreset` — a captured instance shape: provider, version, topology, sizing, storage class — but the only way to use one was the API. It now has a UI, and creating an instance becomes naming it and pressing Create.

The creation wizard opens with a preset picker, filtered to the chosen provider. Picking one resolves it for the target namespace (filling in namespace defaults and the cluster's default `StorageClass`), populates the form, locks the topology and disables the deeper steps; **Create** lights up as soon as it resolves. The resolved spec is submitted verbatim with an `openeverest.io/instance-preset` provenance annotation, so nothing wizard-only leaks into the object. This is phase 1 — editing a preset before deploying and presets alongside plugins follow; creating from scratch is untouched.

See [Instance presets](../administer/instance_presets.md) for how to author presets, ship them with a provider, and use them in the UI.

### Provider upgrades that cannot surprise you

Upgrading a provider used to be an act of faith: the new operator might restart every pod, or refuse to manage an instance whose version its catalog had dropped, and you found out afterwards. Three layers now prevent that.

**Disruption is authorized, not assumed.** `Instance.spec.maintenance.autoApproveUpTo` is a standing tolerance on a scale of what the *application* observes — `NonDisruptive < RollingRestart < Downtime`. It defaults to `NonDisruptive`, so any restart-or-worse action is held on `status.pendingMaintenance`:

```yaml
status:
  pendingMaintenance:
    - description: "Restart the replica set one node at a time to adopt the new operator's pod template"
      severity: RollingRestart
      approvalToken: psmdb-0.3.0-converge-mydb
```

Copy the token into `spec.maintenance.approved` to authorize that one occurrence. It is matched literally, so a stale approval can never authorize a later action, and the pending list is rebuilt every reconcile so it can never go stale. An approved action that keeps failing trips a circuit breaker, so a crash-looping provider cannot repeatedly disrupt a running instance.

**Deprecated versions surface before the upgrade.** The read-only `ComponentVersionDeprecated` condition names the affected versions and the provider release that removes them — *"mongod 6.0.19-16 is deprecated and is removed in provider 0.3"* — without blocking anything.

**The upgrade is preflighted.** `provider-runtime/preflight` runs from a chart's Helm pre-upgrade hook and refuses an upgrade that would strand an Instance: a component version missing from the target catalog or past its `removedInVersion` blocks, a merely `deprecated` one warns, and an installed release below the target's `minUpgradableFrom` blocks. Providers add their own checks through the optional `controller.UpgradeProvider` interface.

### Change one field without rewriting the whole object

The Instance and BackupStorage APIs exposed `GET`, `PUT` and `DELETE` only, so changing one field meant read-modify-write — a lost-update window for every client, and a typed client older than the server silently erasing settings it did not model.

Both now accept `PATCH` with `application/merge-patch+json`: absent members keep their value, `null` removes one, and `metadata.resourceVersion` becomes a precondition that fails with `409`. Patches go out with `fieldValidation=Strict`, so a misspelt path is **rejected** rather than returning `200` having changed nothing. `everestctl instance update` and `everestctl backup-storage update` sit on the same endpoints — though `everestctl` is still **not shipped as a binary**, see [Installing](#installing-openeverest-200-developer-preview-3).

### Scheduling is more than affinity

`ComponentSpec.affinity` is replaced by a `schedulingPolicy` block carrying `schedulerName`, `nodeSelector`, `affinity`, `tolerations` and `topologySpreadConstraints`, defined once as `common.SchedulingPolicy`. Affinity alone was never the whole placement story — a component pinned to a tainted node pool needs tolerations, and zone spreading is normally `topologySpreadConstraints` — and the struct lives in `api/common/v1alpha1` because placement is not Instance-specific.

### A live preview for provider UI schemas

Writing a topology UI schema meant editing YAML, rebuilding, reinstalling the provider and clicking through the wizard to see the result. The playground at `/plugin-developer` makes it a split pane: paste the schema, watch the real `UIGenerator` render it, with syntax and structural errors reported inline on the offending key. The `ui-generator` also learned `uiType: toggle`, so a schema can describe a boolean switch without a custom React form.

---

## ⚠️⚠️⚠️ Breaking changes for provider and plugin developers

**1. `Backup.spec.instanceRef` moved into a discriminated `origin`.** A `Backup` no longer implies a live producer: `origin.type` is `Instance` or `External`, the latter naming data already in a `BackupStorage` with no operator object behind it. Only the schema and runtime plumbing land here — nothing yet discovers or creates `External` backups — but the shape change is immediate.

```yaml
# before
spec:
  instanceRef:
    name: my-instance

# after
spec:
  origin:
    type: Instance
    instanceRef:
      name: my-instance
```

After the CRD upgrade the API server prunes the unknown `spec.instanceRef`, existing `Backup` CRs have no `spec.origin` and every write to them is rejected. **Existing `Backup` CRs must be recreated; restores from them will not work until they are.** In the SDK, `controller.IndexBackupInstanceName` changed to `spec.origin.instanceRef.name` and `Context.BackupsForInstance()` follows. The reference providers are already updated.

**2. `ComponentSpec.affinity` → `ComponentSpec.schedulingPolicy.affinity`,** alongside the new `schedulerName`, `nodeSelector`, `tolerations` and `topologySpreadConstraints` siblings. `nodeName` is deliberately omitted: pinning every replica to one node is meaningless for a replicated workload, and `nodeSelector` on a unique label covers the real case.

**3. `POST`/`DELETE /v1/session` are removed.** Use `POST /v1/auth/token` with `grant_type=password` and read `access_token` (not `token`). The TTL is 15 minutes by default where `/v1/session` issued a 24-hour JWT, so long-running clients should renew via the `refresh_token` grant. `UserCredentials` is gone from the spec and every generated client.

**4. Backup storage `PATCH` changed content type and semantics.** It previously accepted `application/json` with a full document and applied a hand-rolled overlay under which nothing could be cleared and unrecognised members were dropped from a request that still answered `200`. It now requires `application/merge-patch+json` with a *partial* document and answers `415` otherwise; `status` and the `ownerReferences`/`finalizers`/`name`/`namespace` metadata members are rejected, as is a misspelt path.

**5. Plugin API paths are cluster-scoped.** Plugin discovery and context were hand-registered routes outside the standard chain; they now go through the same OpenAPI handler path as every other resource. Plugins hardcoding the old paths will break and must use `api.basePath` or relative paths.

| Before | After |
|---|---|
| `GET /v1/plugins` | `GET /v1/clusters/{cluster}/plugins` |
| `GET /v1/plugins/context` | `GET /v1/clusters/{cluster}/plugin-context` |
| `/v1/plugins/{name}/*` (bundle/proxy) | `/v1/clusters/{cluster}/plugins/{name}/*` |
| `POST /v1/plugins/{name}/instance-config` | `POST /v1/clusters/{cluster}/plugins/{name}/instance-config` |

**6. A `ProviderManaged` `BackupClass` must list exactly one supported provider.** Enforced by CEL; classes naming several are rejected.

**7. `Backup.status.state` and `Restore.status.state` are typed enums** (`Pending|Running|Succeeded|Failed|Error`, plus `Deleting` for backups). Generated clients expose constants instead of `*string`, and any other value is rejected.

**8. `provider-runtime/manifest` and `cmd/generate-provider-manifest` are removed,** superseded by the provider-SDK's own `provider-spec.yaml` generation.

**Worth adopting, not breaking.** `Instance.spec.userSecretRef` seeds an engine's bootstrap credentials from a Secret (immutable once set, since they only apply at creation). `controller.EffectiveVersionBundleName` exposes the runtime's bundle-selection order — `spec.version` → frozen `status.version` → the provider default — so a provider needing the bundle agrees with the runtime by construction.

---

## Changes

### Added

- **One-click instance creation from an `InstancePreset`** — provider-filtered preset picker in the creation wizard, resolved per namespace and submitted verbatim with a provenance annotation, with prefetching so the form never flashes a skeleton ([#3178](https://github.com/openeverest/openeverest/pull/3178), [#3183](https://github.com/openeverest/openeverest/pull/3183)).
- **Provider upgrade safety, end to end:**

  | Piece | What it adds | PRs |
  |---|---|---|
  | Maintenance API | `spec.maintenance.autoApproveUpTo` / `approved`, `status.pendingMaintenance`, `MaintenancePending` | [#3081](https://github.com/openeverest/openeverest/pull/3081) |
  | Runtime gating | `Context.RequestMaintenance()` and the post-`Sync` flush | [#3082](https://github.com/openeverest/openeverest/pull/3082) |
  | Failure breaker | repeatedly-failing approved actions are held, not retried forever | [#3101](https://github.com/openeverest/openeverest/pull/3101), [#3130](https://github.com/openeverest/openeverest/pull/3130) |
  | Preflight library | `PreflightUpgrade`, `CheckUpgradePath`, `HasBlockingIssues`, `UpgradeProvider` | [#2626](https://github.com/openeverest/openeverest/pull/2626) |
  | Hook runner | `preflight.Run()` for a chart's Helm pre-upgrade hook Job | [#3069](https://github.com/openeverest/openeverest/pull/3069) |
  | Deprecation notice | read-only `ComponentVersionDeprecated` condition | [#3070](https://github.com/openeverest/openeverest/pull/3070) |

- `Backup.spec.origin` discriminated union and `Instance.spec.userSecretRef` for seeding initial engine credentials ([#3063](https://github.com/openeverest/openeverest/pull/3063)). (by @chilagrow)
- `PATCH` with `application/merge-patch+json` on the Instance API ([#3055](https://github.com/openeverest/openeverest/pull/3055)), plus `everestctl instance update` (`--set`, `-f`, `--dry-run`) ([#3135](https://github.com/openeverest/openeverest/pull/3135)) and `everestctl backup-storage update` for S3 credential rotation and connection settings ([#3139](https://github.com/openeverest/openeverest/pull/3139)). (by @VijetaPriya47)
- `TopologyComponent.supportedFields` ([#3057](https://github.com/openeverest/openeverest/pull/3057)) and the `provider-runtime/conformance` checks that keep it honest ([#3053](https://github.com/openeverest/openeverest/pull/3053), [#3060](https://github.com/openeverest/openeverest/pull/3060)).
- `common.SchedulingPolicy` on components ([#3022](https://github.com/openeverest/openeverest/pull/3022)) and `controller.EffectiveVersionBundleName` ([#3054](https://github.com/openeverest/openeverest/pull/3054)).
- Live preview for topology UI schemas at `/plugin-developer` ([#2465](https://github.com/openeverest/openeverest/pull/2465) by @Denyme24) and `uiType: toggle` in the ui-generator ([#2288](https://github.com/openeverest/openeverest/pull/2288) by @StepanovPlaton).
- Typed `Backup`/`Restore` state enums across the CRDs and every generated client ([#2933](https://github.com/openeverest/openeverest/pull/2933)). (by @amh1k)
- Unit badges for Kubernetes resource quantities in the UI ([#3191](https://github.com/openeverest/openeverest/pull/3191)) and UI architecture documentation ([#3115](https://github.com/openeverest/openeverest/pull/3115)).
- CI checks that regenerated files are committed ([#3136](https://github.com/openeverest/openeverest/pull/3136) by @devanshu0x) and that the helm-charts pin is on the branch ([#2998](https://github.com/openeverest/openeverest/pull/2998)); multi-arch `helmtools` image on GHCR ([#2930](https://github.com/openeverest/openeverest/pull/2930)); PureCS in `ADOPTERS.md` ([#3166](https://github.com/openeverest/openeverest/pull/3166)).

### Changed & Improved

- **Breaking API changes**, fully mapped in [Breaking changes](#breaking-changes-for-provider-and-plugin-developers): `Backup.spec.origin` ([#3063](https://github.com/openeverest/openeverest/pull/3063)) · `ComponentSpec.schedulingPolicy` ([#3022](https://github.com/openeverest/openeverest/pull/3022)) · `/v1/session` removed ([#2377](https://github.com/openeverest/openeverest/pull/2377)) · backup storage `PATCH` semantics ([#3112](https://github.com/openeverest/openeverest/pull/3112)) · cluster-scoped plugin paths ([#3104](https://github.com/openeverest/openeverest/pull/3104)) · single-provider `ProviderManaged` classes ([#3063](https://github.com/openeverest/openeverest/pull/3063)) · typed state enums ([#2933](https://github.com/openeverest/openeverest/pull/2933)) · `provider-runtime/manifest` removed ([#3061](https://github.com/openeverest/openeverest/pull/3061)).
- Events are delivered in sequence order, so a replaying client cannot observe them out of order ([#3117](https://github.com/openeverest/openeverest/pull/3117)).
- Account passwords now require a minimum of 8 characters ([#3073](https://github.com/openeverest/openeverest/pull/3073)).
- `plugin-sdk` bumped to 0.2.0 ([#2845](https://github.com/openeverest/openeverest/pull/2845)); plugin-hub installation can be disabled for local development ([#2817](https://github.com/openeverest/openeverest/pull/2817)); stale generic-plugin docs removed ([#3125](https://github.com/openeverest/openeverest/pull/3125)).
- Database-specific wording removed from the UI — v2 manages instances, not only databases, and the interface is being generalised to match (phase 1) ([#3193](https://github.com/openeverest/openeverest/pull/3193)).
- `VMAgent` `remoteWrite` entries are deterministically ordered, so monitoring config no longer churns ([#3119](https://github.com/openeverest/openeverest/pull/3119)); the RBAC ConfigMap adapter no longer fetches the same ConfigMap twice per refresh ([#3165](https://github.com/openeverest/openeverest/pull/3165)).
- Tilt dev environment dropped `everest-operator`, documents the `br_netfilter` prerequisite and points at `openeverest/openeverest-operator` ([#3182](https://github.com/openeverest/openeverest/pull/3182), [#3181](https://github.com/openeverest/openeverest/pull/3181), [#3176](https://github.com/openeverest/openeverest/pull/3176) by @bhuvan-somisetty); branch-layout docs and CI follow the `main` ↔ `release-2.0` swap ([#2976](https://github.com/openeverest/openeverest/pull/2976), [#2977](https://github.com/openeverest/openeverest/pull/2977), [#2983](https://github.com/openeverest/openeverest/pull/2983), [#2985](https://github.com/openeverest/openeverest/pull/2985)).

### Fixed

- **Security:** RBAC was evaluated after request validation on the backup storage endpoints, letting an unauthorized caller drive a server-side request to an arbitrary endpoint (SSRF) ([#2937](https://github.com/openeverest/openeverest/pull/2937) by @KrishnaParihar1).
- RBAC: handler panics terminated the connection instead of returning a generic `500` ([#3200](https://github.com/openeverest/openeverest/pull/3200)); `ListNamespaces` evaluated against the wrong object ([#3198](https://github.com/openeverest/openeverest/pull/3198)); policy validation used one charset for every term position ([#3185](https://github.com/openeverest/openeverest/pull/3185)); an invalid policy refresh panicked ([#2791](https://github.com/openeverest/openeverest/pull/2791) by @RishiSingh13).
- Sessions and accounts: concurrent account updates raced on read-modify-write ([#2926](https://github.com/openeverest/openeverest/pull/2926) by @Harkirat1309); data race between `Allow` and `IncreaseTimeout` in the rate limiter ([#2881](https://github.com/openeverest/openeverest/pull/2881) by @Bhupesh-081) plus a hardened regression test ([#3010](https://github.com/openeverest/openeverest/pull/3010) by @Phran6ix); `DeleteSession` called `IncreaseTimeout` incorrectly ([#2800](https://github.com/openeverest/openeverest/pull/2800) by @ayushgupta-h).
- Server: plain and TLS listeners had no `ReadTimeout`/`IdleTimeout` ([#2949](https://github.com/openeverest/openeverest/pull/2949) by @alexsmolya); the event stream advertised v1 `Backup`/`Restore` kinds ([#2921](https://github.com/openeverest/openeverest/pull/2921) by @Harsh63870); a dropped error in `UpdateMonitoringConfig` was not wrapped ([#2945](https://github.com/openeverest/openeverest/pull/2945) by @HamidKhan1001); duplicate enum markers on `DeletionPolicy` ([#2913](https://github.com/openeverest/openeverest/pull/2913) by @Adii-45).
- Preflight: unpinned components, a nil target spec and a missing hook context ([#3129](https://github.com/openeverest/openeverest/pull/3129)); the deprecation condition reported removed versions as merely deprecated ([#3131](https://github.com/openeverest/openeverest/pull/3131)).
- `everestctl`: auth transport connections were shared where they should have been isolated ([#2903](https://github.com/openeverest/openeverest/pull/2903) by @luantaraschi); `update` with no arguments errored instead of showing help ([#2992](https://github.com/openeverest/openeverest/pull/2992) by @VijetaPriya47).
- UI: the provider version showed on the instance tile ([#3052](https://github.com/openeverest/openeverest/pull/3052) by @jadhavgaurav); select-field `helperText` was dropped by the ui-generator ([#3078](https://github.com/openeverest/openeverest/pull/3078) by @deveshxp4).
- CI: the dev build on `main` and the Dockerfile path in its workflow were broken ([#3162](https://github.com/openeverest/openeverest/pull/3162), [#3114](https://github.com/openeverest/openeverest/pull/3114)); CLI tests hung until timeout on a real terminal ([#3068](https://github.com/openeverest/openeverest/pull/3068), [#3110](https://github.com/openeverest/openeverest/pull/3110)); repository paths containing spaces broke the release tooling ([#2812](https://github.com/openeverest/openeverest/pull/2812) by @amh1k); issue-assignment automation was not idempotent ([#3002](https://github.com/openeverest/openeverest/pull/3002) by @moroshani) and a delegated `/assign` failure named neither the assignee nor the cause ([#3118](https://github.com/openeverest/openeverest/pull/3118) by @HamidKhan1001).

---

## Installing OpenEverest 2.0.0 Developer Preview 3

v2 is installed with Helm; you need a running Kubernetes cluster and `helm`.

```bash
helm repo add openeverest https://openeverest.github.io/helm-charts/
helm repo update
helm install everest-core openeverest/openeverest \
  --devel --version "2.0.0-dev.3" \
  --namespace everest-system --create-namespace

# admin password
kubectl get secret everest-accounts -n everest-system \
  -o jsonpath='{.data.users\.yaml}' | base64 --decode | yq '.admin.passwordHash'

# UI
kubectl port-forward svc/everest 8080:8080 -n everest-system
```

`everestctl` is not published as a binary with this release: its `install`/`uninstall` commands still target v1 and cannot install v2, so shipping it would be misleading. To use the day-2 commands against an existing v2 installation, build it from source with `make build-cli` on the `v2.0.0-dev.3` tag.

Providers are released independently. Install the ones you need from their own repositories, picking a version built against `2.0.0-dev.3` — earlier provider releases target the previous `Backup` and `Instance` schemas and will not reconcile.

- [MongoDB](https://github.com/openeverest/provider-percona-server-mongodb) · [MySQL](https://github.com/openeverest/provider-percona-xtradb-cluster) · [PostgreSQL](https://github.com/openeverest/provider-percona-postgresql)

---

## v1 support and end-of-life timeline

| Stage | Description |
|-------|-------------|
| **Now** | v2 Developer Preview — testing and feedback. v1 remains the released, supported version. |
| **v2 GA** | Expected in a few months, together with the supported v1 → v2 migration path. |
| **GA + 3 months** | v1 enters Maintenance Mode: security patches and critical fixes only. |
| **GA + 12 months** | v1 reaches End of Life. |

You do not need to plan a migration during the preview, and you will not be doing one by hand.

---

## Get involved

Build a provider against the new API and tell us where it hurts. Provider plugins, generic plugins and core improvements are all welcome.

- [Provider SDK](https://github.com/openeverest/provider-sdk) and the [provider development guide](https://github.com/openeverest/provider-sdk/blob/main/PROVIDER_DEVELOPMENT.md)
- [Generic Plugin Template](https://github.com/openeverest/generic-plugin-template) · [Generic Plugins Spec](https://github.com/openeverest/specs/blob/main/specs/003-generic-plugins.md) · [Provider Upgrades Spec](https://github.com/openeverest/specs/blob/main/specs/009-provider-upgrades.md)

---

## Thanks to our contributors

@Adii-45, @alexsmolya, @amh1k, @ayushgupta-h, @bhuvan-somisetty, @Bhupesh-081, @Denyme24, @devanshu0x, @deveshxp4, @HamidKhan1001, @Harkirat1309, @Harsh63870, @jadhavgaurav, @KrishnaParihar1, @luantaraschi, @moroshani, @Phran6ix, @RishiSingh13, @StepanovPlaton, @VijetaPriya47

`everestctl` and API work by @VijetaPriya47 continues through the [LFX Mentorship program 2026 Term 2](https://mentorship.lfx.linuxfoundation.org/project/42dff370-4958-4ec4-959c-4aaf6740698c).

---

**Full Changelog**: https://github.com/openeverest/openeverest/compare/v2.0.0-dev.2...v2.0.0-dev.3
