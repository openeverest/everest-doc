# What's new in OpenEverest 1.16.0

➡️ **New to OpenEverest?** Get started with our [Quickstart Guide](../quick-install.md).

---

## 🌟 Release highlights

### Separate CPU/Memory requests and limits for database engines and proxies

You can now configure resource *requests* and *limits* independently for both database engines and proxies, instead of having them locked to the same value. This gives you finer control over scheduling and resource efficiency — set conservative requests so pods schedule easily, while allowing bursts up to a higher limit. The new settings are available in the wizard's Resources step and in the API via the `DatabaseCluster` spec.

### Proxy configuration for every database engine

The `spec.proxy.config` field now works end-to-end across all supported engines:

- **PostgreSQL** — a free-form `pgbouncer.ini` snippet is parsed and applied to the pgBouncer configuration.
- **MongoDB (sharded)** — custom configuration is forwarded to the `mongos` routers.
- **MySQL** — proxy configuration continues to be applied to HAProxy/ProxySQL.

The web UI also exposes this in the Advanced Configuration step, with engine-specific labels (Proxy Configuration, PG Bouncer Configuration, Router Configuration).

### New database operator versions

OpenEverest now supports **Percona Operator for MySQL (PXC) 1.20.0** and **Percona Operator for PostgreSQL 3.0.0**, bringing in the latest upstream fixes and features for both engines.

---

## 📝 Changes

### Added

- [#2461](https://github.com/openeverest/openeverest/pull/2461), [operator#961](https://github.com/openeverest/openeverest-operator/pull/961): Separate CPU/Memory requests and limits for database engines and proxies ([#2360](https://github.com/openeverest/openeverest/issues/2360)).
- [#2108](https://github.com/openeverest/openeverest/pull/2108): Proxy configuration (`spec.proxy.config`) exposed in the Advanced Configuration step of the web UI, with engine-specific labels ([#2093](https://github.com/openeverest/openeverest/issues/2093)). Contributed by [@Denyme24](https://github.com/Denyme24).
- [operator#954](https://github.com/openeverest/openeverest-operator/pull/954): `spec.proxy.config` is now reconciled into the pgBouncer configuration for PostgreSQL clusters ([#2095](https://github.com/openeverest/openeverest/issues/2095)). Contributed by [@Heyy-Himanshuu](https://github.com/Heyy-Himanshuu).
- [operator#952](https://github.com/openeverest/openeverest-operator/pull/952): `spec.proxy.config` is now applied to `mongos` in MongoDB sharded mode ([#2094](https://github.com/openeverest/openeverest/issues/2094)). Contributed by [@areebahmeddd](https://github.com/areebahmeddd).
- [#2500](https://github.com/openeverest/openeverest/pull/2500), [operator#963](https://github.com/openeverest/openeverest-operator/pull/963): Support for Percona Operator for MySQL (PXC) 1.20.0.
- [#2503](https://github.com/openeverest/openeverest/pull/2503), [operator#962](https://github.com/openeverest/openeverest-operator/pull/962): Support for Percona Operator for PostgreSQL 3.0.0.

### Changed & Improved

- [#2098](https://github.com/openeverest/openeverest/pull/2098): Removed the Tech Preview label from the Shards section — MongoDB sharding is now generally available.
- [#2196](https://github.com/openeverest/openeverest/pull/2196): Improved the `everestctl namespaces` command description. Contributed by [@ayushHardeniya](https://github.com/ayushHardeniya).

### Fixed

- [#2525](https://github.com/openeverest/openeverest/pull/2525): Fixed SSO login not redirecting to the main interface after successful OIDC authentication, leaving users stuck on the `/login-callback` page ([#2306](https://github.com/openeverest/openeverest/issues/2306)).
- [operator#959](https://github.com/openeverest/openeverest-operator/pull/959): Fixed a race during PostgreSQL data import where, on slow storage, the recreated cluster could reuse leftover empty resources and come up ready without the imported data ([#2347](https://github.com/openeverest/openeverest/issues/2347)).
- [operator#950](https://github.com/openeverest/openeverest-operator/pull/950): Ensured PostgreSQL restore objects are deleted before database cleanup, preventing the PG operator from failing to remove restores once the cluster is gone.
- [#2177](https://github.com/openeverest/openeverest/pull/2177): Fixed annotations being deleted when restarting a database cluster from the UI ([#2175](https://github.com/openeverest/openeverest/issues/2175)).
- [#2088](https://github.com/openeverest/openeverest/pull/2088): `--disable-telemetry` is now correctly propagated to the main Everest Helm release during install and upgrade ([#2081](https://github.com/openeverest/openeverest/issues/2081)). Contributed by [@atharvamhaske](https://github.com/atharvamhaske).
- [#2086](https://github.com/openeverest/openeverest/pull/2086): Token invalidation no longer reports success when the blocklist secret update fails ([#2085](https://github.com/openeverest/openeverest/issues/2085)). Contributed by [@blenbot](https://github.com/blenbot).
- [#2173](https://github.com/openeverest/openeverest/pull/2173): Scoped the database credentials cache by namespace, so clusters with the same name in different namespaces no longer show each other's credentials. Contributed by [@ashnaaseth2325-oss](https://github.com/ashnaaseth2325-oss).
- [#2228](https://github.com/openeverest/openeverest/pull/2228): The Everest upgrade dialog no longer reopens after being dismissed. Contributed by [@ashnaaseth2325-oss](https://github.com/ashnaaseth2325-oss).
- [#2112](https://github.com/openeverest/openeverest/pull/2112): Fixed the auth interceptor being ejected from the wrong request pipeline on logout. Contributed by [@ashnaaseth2325-oss](https://github.com/ashnaaseth2325-oss).
- [#2166](https://github.com/openeverest/openeverest/pull/2166): Fixed a missing observer unsubscribe in `useNamespacePermissionsForResource`. Contributed by [@ashnaaseth2325-oss](https://github.com/ashnaaseth2325-oss).
- [#2052](https://github.com/openeverest/openeverest/pull/2052): Backup time labels are now hidden in the cluster overview when no backups exist ([#1733](https://github.com/openeverest/openeverest/issues/1733)). Contributed by [@StepanovPlaton](https://github.com/StepanovPlaton).
- [#2482](https://github.com/openeverest/openeverest/pull/2482): Added the missing `noreferrer` attribute to external links in the UI, hardening against reverse tabnabbing ([#2481](https://github.com/openeverest/openeverest/issues/2481)). Contributed by [@Pranav-IIITM](https://github.com/Pranav-IIITM).

---

## 🚀 Upgrade to OpenEverest 1.16.0

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

**Full Changelog**: https://github.com/openeverest/openeverest/compare/v1.15.2...v1.16.0
