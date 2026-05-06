# What's new in OpenEverest 1.15.2

**New to OpenEverest?** Get started with our [Quickstart Guide](../quick-install.md).

---

## Release highlights

This is a patch release that fixes a critical bug where the `everestctl` Docker image was being built with version `0.0.0` instead of the actual release version. This caused pre-upgrade hook validation failures during Helm upgrades. The release also includes routine dependency updates.

---

## Changes

### Fixed

- [#2168](https://github.com/openeverest/openeverest/pull/2168): Fixed the `everestctl` Docker image being built with version `0.0.0` instead of the release version, which caused pre-upgrade hook validation failures during Helm upgrades. The fix adds the `VERSION` and `RELEASE_FULLCOMMIT` build arguments to the `everestctl` image build step in the release workflow.
- [#2145](https://github.com/openeverest/openeverest/pull/2145): Fixed API tests that broke due to a behavior change in the `echo-jwt` package. Version 4.3.0 of that package changed unauthenticated requests to return HTTP `400` instead of `401`; the subsequent dependency update ([#2142](https://github.com/openeverest/openeverest/pull/2142)) reverted that change back to `401`. The test assertions were updated to match the correct status code.
- [#2112](https://github.com/openeverest/openeverest/pull/2112): Fixed auth interceptor ejection from the correct request pipeline.

### Dependency updates

- Updated Go dependencies ([#2157](https://github.com/openeverest/openeverest/pull/2157), [#2142](https://github.com/openeverest/openeverest/pull/2142), [#2118](https://github.com/openeverest/openeverest/pull/2118))
- Updated API and CLI test dependencies ([#2146](https://github.com/openeverest/openeverest/pull/2146), [#2121](https://github.com/openeverest/openeverest/pull/2121), [#2120](https://github.com/openeverest/openeverest/pull/2120))
- Updated `oapi-codegen` to v2.7.0 ([#2163](https://github.com/openeverest/openeverest/pull/2163))
- Bumped `axios` from 1.8.2 to 1.12.2 in `/ui` ([#1651](https://github.com/openeverest/openeverest/pull/1651))
- Bumped `alpine` from 3.22 to 3.23 ([#1736](https://github.com/openeverest/openeverest/pull/1736), [#2050](https://github.com/openeverest/openeverest/pull/2050))
- Bumped `rancher/k3s` from v1.33.2-k3s1 to v1.35.4-k3s1 in `/dev` ([#2048](https://github.com/openeverest/openeverest/pull/2048), [#2125](https://github.com/openeverest/openeverest/pull/2125))
- Updated GitHub Actions dependencies:
    - `actions/checkout` 4.3.1 → 6.0.2
    - `actions/setup-node` 5.0.0 → 6.4.0
    - `actions/upload-artifact` 4.6.2 → 7.0.1
    - `actions/setup-python` 5.6.0 → 6.2.0
    - `actions/labeler` 5.0.0 → 6.0.1
    - `actions/configure-pages` 5.0.0 → 6.0.0
    - `actions/upload-pages-artifact` 4.0.0 → 5.0.0
    - `actions/attest-build-provenance` 2.4.0 → 4.1.0
    - `actions/add-to-project` 1.0.2 → 2.0.0
    - `docker/setup-buildx-action` 3.12.0 → 4.0.0
    - `github/codeql-action` 3.35.2 → 4.35.3
    - `helm/kind-action` 1.12.0 → 1.14.0
    - `pnpm/action-setup` 4.3.0 → 6.0.5
    - `softprops/action-gh-release` 2.6.2 → 3.0.0

---

## Upgrade to OpenEverest 1.15.2

### Using everestctl

```sh
everestctl upgrade
```

### Using Helm directly

```sh
helm repo update openeverest
helm upgrade everest openeverest/everest -n everest-system
```

---

**Full Changelog**: https://github.com/openeverest/openeverest/compare/v1.15.1...v1.15.2
