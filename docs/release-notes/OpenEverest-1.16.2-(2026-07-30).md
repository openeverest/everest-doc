# What's new in OpenEverest 1.16.2

➡️ **New to OpenEverest?** Get started with our [Quickstart Guide](../quick-install.md).

> ⚠️ **Note**: This is a patch release containing a single fix for arm64 container images. There are no breaking changes and no migration steps.

---

## 🌟 Release highlights

### arm64 container images now actually run on arm64

The `arm64` variant of `ghcr.io/openeverest/openeverest` shipped an x86-64 `everest-api` binary, so the OpenEverest server crashed on arm64 hosts with `exec /everest-api: exec format error`. The build now honours the target architecture, and the release workflows verify it before publishing. The `everestctl` image and CLI binaries were never affected.

**Running on arm64?** Upgrade to 1.16.2 and make sure your nodes re-pull the image.

---

## 📝 Changes

### Fixed

- [#2605](https://github.com/openeverest/openeverest/pull/2605): The arm64 OpenEverest server image no longer ships an amd64 binary, fixing `exec /everest-api: exec format error` on arm64 clusters ([#2603](https://github.com/openeverest/openeverest/issues/2603)). Reported by [@auguster](https://github.com/auguster).

---

## 🚀 Upgrade to OpenEverest 1.16.2

### Using everestctl

```sh
everestctl upgrade
```

### Using Helm directly

Start with an update of Custom Resource Definitions (CRDs):
```sh
helm repo update
helm upgrade --install everest-crds \
    openeverest/everest-crds \
    --namespace everest-system \
    --take-ownership
```

Update OpenEverest itself:

```sh
helm upgrade everest openeverest/everest -n everest-system
```

---

**Full Changelog**: https://github.com/openeverest/openeverest/compare/v1.16.1...v1.16.2
