# Instance presets in OpenEverest

!!! info "Availability"
    Instance presets are available in OpenEverest 2.0.0 Developer Preview 3 and later. This is **phase 1**: one-click deployment from a preset. Editing a preset before deploying, and managing presets from the UI, arrive in a later release.

## Introduction

An **InstancePreset** is a reusable, ready-to-deploy configuration for an OpenEverest Instance. It captures a complete instance shape — provider, version, topology, component sizing, storage and scheduling — as a single Kubernetes object that users pick from a list instead of filling in every field by hand.

Presets turn instance creation into naming the instance and pressing **Create**.

## Why presets matter

Deploying an Instance from scratch means configuring each component individually: replicas, resources, storage class, backup, scheduling and more. This is repetitive, error-prone, and hard to keep consistent across a team.

Presets solve this by providing:
{.power-number}

1. **One-click deployment.** A curated configuration that is production-ready out of the box, so users do not have to know every field to deploy a sound instance.
2. **Consistency.** Everyone deploying from the same preset gets the same shape, which removes drift between environments and between team members.
3. **A safe default surface.** Administrators and provider authors encode good defaults once; users select rather than assemble. Power users still get the full parameter surface when they need it.

## How presets reach a cluster

Presets can be installed in a cluster in two ways, and both produce the same cluster-scoped `InstancePreset` resource:
{.power-number}

1. **Shipped with a provider.** A provider's Helm chart can bundle presets, so installing the provider also installs a set of ready-made configurations for it. This is the common path — users get sensible presets the moment a provider is available.
2. **Authored directly.** An administrator can create an `InstancePreset` custom resource by hand (or with `kubectl`) to capture an organization-specific configuration.

!!! info "Presets are cluster-scoped"
    `InstancePreset` is a cluster-scoped resource — it is not tied to a namespace, so a single preset can be used to create instances in any namespace. Namespace-specific values (such as monitoring configs and secrets) are left empty in the preset and resolved per namespace at creation time (see [Namespace default resolution](#namespace-default-resolution)).

## The `InstancePreset` resource

An `InstancePreset` mirrors the `Instance` spec: its `spec` is an `InstancePresetSpec`, which is the same structure as `Instance.spec`. This is why a preset can be applied to create an instance verbatim.

Because a preset is cluster-scoped, it must not reference namespace-scoped objects directly. The rule of thumb is:

- **Cluster-scoped values** (such as `storageClass`) **can** be set directly in the preset.
- **Namespace-scoped references** (such as secret and config map names) **must** be left empty and are resolved from annotated defaults in the target namespace when the preset is used.

### Creating a preset as a CR

Apply an `InstancePreset` like any other Kubernetes resource:

```yaml
apiVersion: core.openeverest.io/v1alpha1
kind: InstancePreset
metadata:
  name: mongodb-production
spec:
  providerRef:
    name: percona-server-mongodb
  version: "8.0.12"
  topology:
    type: replicaSet
  components:
    engine:
      type: mongod
      replicas: 3
      resources:
        limits:
          cpu: "2"
          memory: 8Gi
      storage:
        size: 100Gi
        storageClass: fast-ssd   # cluster-scoped — safe to set in a preset
```

```bash
kubectl apply -f mongodb-production-preset.yaml
```

The preset is now available cluster-wide and appears in the creation wizard for the `percona-server-mongodb` provider.

### Shipping presets with a provider

Provider authors bundle presets in the provider's Helm chart. Presets are declared in the chart's `values.yaml` under the `presets:` key, and each entry is rendered into an `InstancePreset` resource when the chart is installed.

```yaml
# charts/<provider-name>/values.yaml
presets:
  - name: standalone
    enabled: true
    spec:
      providerRef:
        name: my-provider
      version: "8.0.12"
      topology:
        type: standalone
      components:
        engine:
          replicas: 1
          resources:
            limits:
              cpu: "1"
              memory: 2Gi
          storage:
            size: 25Gi
```

Each entry has three fields:

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Preset name prefix, combined with the provider `shortName` to form the resource name |
| `enabled` | bool | Set to `false` to exclude the preset from the rendered chart |
| `spec` | object | The full `InstancePresetSpec` (provider, version, topology, components, …) |

The rendered resource is named `<shortName>-<preset.name>` (for example, `psmdb-standalone`). Installing the provider chart installs the presets alongside it; disabling an entry removes it on the next chart upgrade.

!!! info "For provider authors"
    See the [Provider Development Guide](https://github.com/openeverest/provider-sdk/blob/main/PROVIDER_DEVELOPMENT.md) for the full workflow of defining, rendering and testing presets shipped with a provider.

### Namespace default resolution

A preset leaves namespace-scoped references empty because it is cluster-scoped and does not know which namespace an instance will land in. When a preset is used to create an instance in a specific namespace, OpenEverest fills those empty references from resources in that namespace that are annotated as defaults.

- **Namespace-scoped resources** (Secret, ConfigMap, MonitoringConfig) use the annotation `openeverest.io/is-default-components-<component-name>: "true"`.
- **Cluster-scoped resources** (StorageClass) use the standard Kubernetes annotation `storageclass.kubernetes.io/is-default-class: "true"`.

For example, to mark a secret as the default for the `engine` component in the `prod` namespace:

```bash
kubectl annotate Secret db-credentials \
  openeverest.io/is-default-components-engine="true" \
  --namespace prod \
  --overwrite
```

Resolution behavior:

- Defaults for namespace-scoped resources are resolved per namespace; defaults for cluster-scoped resources are resolved cluster-wide.
- If several resources carry the default annotation, the most recently created one is used.
- If no default is found, the field stays empty and validation may fail when the field is required.

## Using presets in the UI

!!! warning "UI is pick-and-use only"
    From the UI, users can only select an existing preset and create an instance from it. Presets cannot be created, edited or deleted through the UI yet — they are authored as CRs or shipped with providers (see [How presets reach a cluster](#how-presets-reach-a-cluster)).

In the instance creation wizard, presets are the starting point:
{.power-number}

1. Open the creation wizard and choose a provider. The wizard shows a **preset picker** filtered to that provider.
2. Select a preset. OpenEverest resolves it for the target namespace — filling in namespace defaults and the cluster's default `StorageClass` — and populates the form. The topology is locked and the deeper configuration steps are disabled, since the preset already defines them.
3. Name the instance and press **Create**. The resolved configuration is submitted as-is, with an `openeverest.io/instance-preset` annotation recording which preset it came from.

The following demo shows the full flow of creating an instance from a preset:

<video controls width="100%" style="max-width: 900px;">
  <source src="../images/presets-demo.mp4" type="video/mp4">
  Your browser does not support the video tag. You can
  <a href="../images/presets-demo.mp4">download the video</a> instead.
</video>

## Limitations

Presets in this release are phase 1 and have the following limitations:

- Presets cannot be created from the UI. Users can only pick an existing preset and deploy from it; presets are authored as CRs or shipped with providers.
- Presets can be applied but not edited before deploying from the UI — the resolved configuration is submitted verbatim.
- Presets are scoped to a single provider and cannot be shared across providers.
- Creating an instance from scratch, without a preset, is unchanged and remains fully available.
