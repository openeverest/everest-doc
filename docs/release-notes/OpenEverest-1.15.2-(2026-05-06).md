# What's new in OpenEverest 1.15.2

**New to OpenEverest?** Get started with our [Quickstart Guide](../quick-install.md).

---

## Release highlights

This is a patch release that fixes a critical bug where the `everestctl` Docker image was being built with version `0.0.0` instead of the actual release version. This caused pre-upgrade hook validation failures during Helm upgrades. The release also includes routine dependency updates.

---

## Changes

### Fixed

- [#2168](https://github.com/openeverest/openeverest/pull/2168): Fixed the `everestctl` Docker image being built with version `0.0.0` instead of the release version, which caused pre-upgrade hook validation failures during Helm upgrades. The fix adds the `VERSION` and `RELEASE_FULLCOMMIT` build arguments to the `everestctl` image build step in the release workflow.

---

## Upgrade to OpenEverest 1.15.2

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

**Full Changelog**: https://github.com/openeverest/openeverest/compare/v1.15.1...v1.15.2
