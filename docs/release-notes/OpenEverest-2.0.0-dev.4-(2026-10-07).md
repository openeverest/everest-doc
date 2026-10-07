# What's new in OpenEverest 2.0.0 Developer Preview 4

!!! warning "Developer Preview — not for production"
    Not feature-complete; for testing and feedback only.

    **Install on a fresh cluster.** There is no upgrade path from Developer Preview 3, and none from v1 *yet*. Running v1 and v2 side by side in one cluster is not supported.

    **A v1 → v2 migration path is planned and ships with v2 General Availability.** It cannot exist during the preview because the v2 API is still changing and a migration tool needs a stable target. Keep running v1: it stays supported, and you will not be asked to rebuild what you are running by hand.

    **Breaking changes** to the `Instance` and `Provider` CRDs, the provider-runtime SDK and RBAC policies — see [Breaking changes](#breaking-changes-for-provider-developers-and-administrators). Nothing between previews gets a migration path, deliberately, so the API can be corrected before GA.

**New to OpenEverest?** Start with the [Quickstart Guide](https://openeverest.io/documentation/2.0.0-dev.4/quick-install.html), or [read the blog post](https://openeverest.io/blog/v2-developer-preview-release/) on the v2 architecture.

---

## Release highlights

### Instances say why their pods are not running

The runtime now fills `status.components` from each component's pods (`replicas`, `readyReplicas`) and sets two conditions: `PodsScheduled` turns `False` once a pod has waited more than a minute for a node, and `PodsReady` names the worst problem (`CrashLoopBackOff`, `ImagePullBackOff`, `CreateContainerConfigError`, `NotReady`). Messages have one line per component, for example *"engine: 1 of 3 pods cannot be scheduled: 0/2 nodes are available: 2 node(s) didn't match pod anti-affinity rules."*

The UI shows a warning banner and icon on affected instances, and `everestctl instance status` gains a `READY` column. It works for providers that label their pods with `Context.PodLabels()`.

### Pod placement in the UI, with defaults that hold

The creation wizard and the instance overview can now edit each component's affinity rules. A provider enables the editor from its UI schema with `widgetType: podSchedulingPolicy`.

The defaults are now explicit: omitting `affinity` or `topologySpreadConstraints` gets the provider's default, and an empty value (`{}` / `[]`) means none. The MongoDB and MySQL providers now place each data-bearing member on its own node, so a three-member cluster needs three schedulable nodes unless you set `affinity: {}`. A cluster that is too small shows up as `PodsScheduled=False` instead of quietly doubling up on one node.

### Optional sections in instance forms

Provider UI schemas can put optional settings behind an *Enable* switch with `groupType: toggleable`. While a section is off, its fields are hidden, not validated and not sent. A section starts on when the instance, preset or source backup already sets one of its fields; otherwise it starts off and the overview shows it as *Disabled*. Turning a section off while editing an instance removes its saved values. `groupType: bordered` draws the same card without the switch.

### Time-based backup retention

Schedules can now keep backups for a period instead of a count: `retention: {type: time, duration: 30d}` (`Nd`, `Nw` or `Nm`). The PostgreSQL provider supports it. The MongoDB and MySQL operators only support count retention, so those providers report `type: time` on the `BackupConfigured` condition. The UI still edits count retention only.

### Plugins that look like the core

Plugins can now match the core's look without depending on its internals. The host shares only React (through an import map) and its design tokens (`--everest-*` CSS variables). Each plugin bundles its own MUI, and the new `@openeverest/plugin-theme` package themes it from those tokens, including live dark mode: wrap the plugin UI in `<PluginThemeProvider cacheKey="my-plugin" nonce={api.cssNonce}>`. A plugin is now skipped if it fails `spec.compatibleHostVersions` (not enforced before) or the new `spec.compatibleUiContractVersions`.

---

## ⚠️⚠️⚠️ Breaking changes for provider developers and administrators

**1. `schedules[].retentionCopies` → `schedules[].retention`** on `Instance` and `InstancePreset`: `retentionCopies: 7` becomes `retention: {type: count, count: 7}`. After the CRD upgrade `retentionCopies` is pruned, so those schedules keep every backup until `retention` is set.

**2. `Context.Apply` uses server-side apply** (field manager `provider-<name>`, forced ownership) instead of Get + Update. Build the full desired object every time: fields you stop setting are removed, and fields you never set are left to the operator. Objects read with `Get` are rejected, and `obj` is no longer updated with the server's response. Empty non-pointer `omitempty` structs are dropped instead of sent as `{}`.

**3. `status.components` is computed by the runtime.** `ComponentStatus` is now `name`/`selector`/`replicas`/`readyReplicas` (was `podRefs`/`total`/`ready`/`state`), and `controller.Status.Components` and `controller.ComponentStatus` are removed. Call `c.PodLabels(component)` in `Sync` and put the labels on that component's pods; the provider needs `list` and `watch` on pods.

**4. `ComponentSpec.name` is removed;** the `spec.components` map key is the name. `GetComponentsOfType()` and `Context.ComponentsOfType()` return `map[string]ComponentSpec`.

**5. `default: true` → `defaultVersion`.** Version bundles and component versions no longer carry a `default` flag. Set `defaultVersion` at the top of the Provider spec and on each component type instead; the provider-sdk's `generate` emits it from `definition/versions.yaml` ([openeverest/provider-sdk#47](https://github.com/openeverest/provider-sdk/pull/47)). Without it, an Instance that omits `spec.version` gets no version bundle.

**6. `SchedulingPolicy.topologySpreadConstraints` is now a pointer** (`*[]corev1.TopologySpreadConstraint`), so unset and `[]` mean different things. Build a component's constraints with `controller.TopologySpreadConstraints(policy, c.PodLabels(component))`.

**7. RBAC.** `backups` and `restores` grants now match the owning instance (`<cluster>/<namespace>/<instance>`) instead of the backup's or restore's own name. Reading instance credentials (`.../instances/{instance}/connection`) needs the new `read-connection` action; `read` no longer covers it. Policies with unknown action names are rejected. Known issue: read-only users see a `403` error on the instance page until the UI checks `read-connection`.

---

## Changes

### Added

- **Pod status and conditions** on instances, shown in the UI and `everestctl` ([#3252](https://github.com/openeverest/openeverest/pull/3252) by @shivansh-source, [#3335](https://github.com/openeverest/openeverest/pull/3335), [#3312](https://github.com/openeverest/openeverest/pull/3312), [#3338](https://github.com/openeverest/openeverest/pull/3338)).
- **Pod scheduling policy editor** in the UI ([#3294](https://github.com/openeverest/openeverest/pull/3294)).
- **Toggleable form sections** and bordered groups in the ui-generator ([#3278](https://github.com/openeverest/openeverest/pull/3278)).
- `BackupImport` API and provider-runtime support, with no UI yet ([#3126](https://github.com/openeverest/openeverest/pull/3126), [#3138](https://github.com/openeverest/openeverest/pull/3138)). (by @chilagrow)
- **Time-based backup retention** ([#3222](https://github.com/openeverest/openeverest/pull/3222) by @AdityaPimpalkar).
- `controller.TopologySpreadConstraints` and documented scheduling defaults ([#3309](https://github.com/openeverest/openeverest/pull/3309), [#3311](https://github.com/openeverest/openeverest/pull/3311)); top-level `defaultVersion` ([#3264](https://github.com/openeverest/openeverest/pull/3264)); `read-connection` RBAC action ([#3288](https://github.com/openeverest/openeverest/pull/3288) by @VijetaPriya47).
- `@openeverest/plugin-theme` 0.1.0 ([#3076](https://github.com/openeverest/openeverest/pull/3076)) and `@openeverest/plugin-sdk` 0.4.0 ([#3283](https://github.com/openeverest/openeverest/pull/3283)).
- Nightly release e2e tests for PITR and restore to a new cluster ([#2925](https://github.com/openeverest/openeverest/pull/2925), [#3261](https://github.com/openeverest/openeverest/pull/3261)).
- Locci Cloud in `ADOPTERS.md` ([#3243](https://github.com/openeverest/openeverest/pull/3243) by @MikeTeddyOmondi).

### Changed & Improved

- **Breaking changes**, detailed in [Breaking changes](#breaking-changes-for-provider-developers-and-administrators) ([#3222](https://github.com/openeverest/openeverest/pull/3222), [#3240](https://github.com/openeverest/openeverest/pull/3240), [#3282](https://github.com/openeverest/openeverest/pull/3282), [#3335](https://github.com/openeverest/openeverest/pull/3335), [#3337](https://github.com/openeverest/openeverest/pull/3337), [#3264](https://github.com/openeverest/openeverest/pull/3264), [#3309](https://github.com/openeverest/openeverest/pull/3309), [#3204](https://github.com/openeverest/openeverest/pull/3204), [#3288](https://github.com/openeverest/openeverest/pull/3288)).
- `provider-runtime` no longer depends on any `github.com/percona/*` module ([#3285](https://github.com/openeverest/openeverest/pull/3285)).
- Go 1.27.1. A provider that bumps core needs a Go 1.27 toolchain to build, so pinned toolchains such as `golang:1.26` images or golangci-lint older than v2.13.0 must move too ([#3280](https://github.com/openeverest/openeverest/pull/3280)).
- CI moved to CNCF-hosted runners and kind, into a single pipeline with merge-queue support, and from MinIO to SeaweedFS ([#3205](https://github.com/openeverest/openeverest/pull/3205), [#3210](https://github.com/openeverest/openeverest/pull/3210), [#3211](https://github.com/openeverest/openeverest/pull/3211), [#3213](https://github.com/openeverest/openeverest/pull/3213), [#3216](https://github.com/openeverest/openeverest/pull/3216), [#3217](https://github.com/openeverest/openeverest/pull/3217), [#3218](https://github.com/openeverest/openeverest/pull/3218), [#3249](https://github.com/openeverest/openeverest/pull/3249), [#3259](https://github.com/openeverest/openeverest/pull/3259)).
- Contributor docs fixes ([#3199](https://github.com/openeverest/openeverest/pull/3199) by @bhuvan-somisetty, [#3202](https://github.com/openeverest/openeverest/pull/3202) by @Sarthak-Shreshtha01).

### Fixed

- **Security:** the restore endpoints enforced no RBAC ([#3204](https://github.com/openeverest/openeverest/pull/3204) by @VijetaPriya47); read-only roles could read instance credentials ([#3288](https://github.com/openeverest/openeverest/pull/3288) by @VijetaPriya47).
- OIDC requests had no response size limit or timeout ([#3219](https://github.com/openeverest/openeverest/pull/3219) by @alishair7071); an unresponsive PMM endpoint could hang monitoring reconciliation ([#3205](https://github.com/openeverest/openeverest/pull/3205)).
- A seeded Instance went back to `Restoring` after its source Backup was deleted ([#3257](https://github.com/openeverest/openeverest/pull/3257)).
- UI: switching topology kept the old topology's fields ([#3255](https://github.com/openeverest/openeverest/pull/3255)); overview cards jumped between columns ([#3316](https://github.com/openeverest/openeverest/pull/3316)); fields without `validation` were treated as required ([#3294](https://github.com/openeverest/openeverest/pull/3294)).
- `everestctl`: `upgrade` failed on the `everest-crds` release after the first upgrade ([#3241](https://github.com/openeverest/openeverest/pull/3241)); `namespaces list` ignored `--json` ([#3248](https://github.com/openeverest/openeverest/pull/3248) by @jhalak101205-cpu).
- CI fixes for fork PRs, go-consistent, `/assign` on edited comments and a flaky e2e step ([#3245](https://github.com/openeverest/openeverest/pull/3245), [#3258](https://github.com/openeverest/openeverest/pull/3258), [#3244](https://github.com/openeverest/openeverest/pull/3244), [#3308](https://github.com/openeverest/openeverest/pull/3308)).

---

## Installing OpenEverest 2.0.0 Developer Preview 4

v2 is installed with Helm; you need a running Kubernetes cluster and `helm`.

```bash
helm repo add openeverest https://openeverest.github.io/helm-charts/
helm repo update
helm install everest-core openeverest/openeverest \
  --devel --version "2.0.0-dev.4" \
  --namespace everest-system --create-namespace

# admin password
kubectl get secret everest-accounts -n everest-system \
  -o jsonpath='{.data.users\.yaml}' | base64 --decode | yq '.admin.passwordHash'

# UI
kubectl port-forward svc/everest 8080:8080 -n everest-system
```

`everestctl` is not published as a binary with this release: its `install`/`uninstall` commands still target v1 and cannot install v2, so shipping it would be misleading. To use the day-2 commands against an existing v2 installation, build it from source with `make build-cli` on the `v2.0.0-dev.4` tag.

Providers are released independently. Install the ones you need from their own repositories, picking a version built against `2.0.0-dev.4` — earlier provider releases target the previous `Instance` and `Provider` schemas and will not work correctly.

- [MongoDB](https://github.com/openeverest/provider-percona-server-mongodb) `0.4.0` · [MySQL](https://github.com/openeverest/provider-percona-xtradb-cluster) `0.3.0` · [PostgreSQL](https://github.com/openeverest/provider-percona-postgresql) `0.4.0`

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

@AdityaPimpalkar, @alishair7071, @bhuvan-somisetty, @jhalak101205-cpu, @MikeTeddyOmondi, @Sarthak-Shreshtha01, @shivansh-source, @VijetaPriya47

---

**Full Changelog**: https://github.com/openeverest/openeverest/compare/v2.0.0-dev.3...v2.0.0-dev.4
