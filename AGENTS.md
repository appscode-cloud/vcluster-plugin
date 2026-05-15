# AGENTS.md

This file provides guidance to coding agents (e.g. Claude Code, claude.ai/code) when working with code in this repository.

## Repository purpose

Go module `go.bytebuilders.dev/vcluster-plugin` — a [vCluster](https://github.com/loft-sh/vcluster) plugin that bridges between a virtual cluster and its host:

- **Syncer**: syncs `CAProviderClass` CRDs (from [`kubeops/csi-driver-cacerts`](https://github.com/kubeops/csi-driver-cacerts)) from the vcluster into the host cluster, so cert-manager CSI cacerts attached to vcluster workloads end up on the host where the CSI driver actually runs.
- **Pod hook**: mutates pods inside the vcluster on their way to the host (e.g. to attach the cacerts CSI volume).

Forked from [`loft-sh/vcluster-plugin-example`](https://github.com/loft-sh/vcluster-plugin-example). See https://www.vcluster.com/docs/v0.19/advanced-topics/plugins-overview for the vcluster plugin model.

The git remote is `appscode-cloud/vcluster-plugin`; the local folder is under `kubeops.dev/vcluster-plugin`. The **Go module is `go.bytebuilders.dev/vcluster-plugin`** — use that in imports.

The produced binary is the vcluster-plugin entry point (`go build .` builds it). There's no Makefile; the plugin ships via the `plugin.yaml` referenced in vcluster install.

## Architecture

- `main.go` — entry point at the module root. Wires the syncer + hooks through `github.com/loft-sh/vcluster-sdk/plugin`.
- `api/` — plugin configuration types (`PluginConfig`).
- `syncers/` — vcluster-side syncers; currently `NewCAProviderClassSyncer(ctx)` which projects `CAProviderClass` from vcluster → host.
- `hooks/` — vcluster hooks; currently `NewPodHook(cfg)` (and a commented-out `NewSecretHook()`).
- `manifests/` — installable manifests / examples.
- `plugin.yaml` — the **public vcluster plugin manifest** — the URL referenced in the README's `vcluster create -f https://github.com/appscode-cloud/vcluster-plugin/raw/master/plugin.yaml`. Edit carefully; this is the user contract.
- `Dockerfile` — plugin image.
- `devspace.yaml`, `devspace_start.sh` — DevSpace dev-loop config for fast iteration against a running vcluster.
- `vendor/` — checked-in deps.

Note: `loft-sh/vcluster-plugin-example` (the upstream this was forked from) is now **archived**. AppsCode owns the maintained version.

## Common commands

This repo has **no Makefile** — build with `go build` directly, ship via `plugin.yaml`.

- Build the plugin binary:

  ```
  go build -o bin/vcluster-plugin .
  ```

- Build the Docker image:

  ```
  docker build -t ghcr.io/appscode-cloud/vcluster-plugin:dev .
  ```

- Iterate against a running vcluster with DevSpace:

  ```
  devspace dev
  ```

- Install in a vcluster:

  ```
  # In host cluster: install csi-driver-cacerts first
  helm upgrade -i cert-manager-csi-driver-cacerts \
    oci://ghcr.io/appscode-charts/cert-manager-csi-driver-cacerts \
    --version v2024.10.17 -n cert-manager --create-namespace --wait

  vcluster create vcluster -n vcluster \
    -f https://github.com/appscode-cloud/vcluster-plugin/raw/master/plugin.yaml
  ```

Run a single Go test:

```
go test ./syncers/... -run TestName -v
```

## Conventions

- Module path is `go.bytebuilders.dev/vcluster-plugin` (vanity URL). Imports must use that; not `kubeops/vcluster-plugin` or `appscode-cloud/vcluster-plugin`.
- License: see `LICENSE`. Sign off commits (`git commit -s`).
- Vendor directory is checked in.
- Plugin entry points are registered in `main.go` via `plugin.MustRegister(...)`. New syncers/hooks belong in `syncers/` / `hooks/` and must be registered there.
- `plugin.yaml` is the public install manifest referenced by the README — when changing the binary's image or arg surface, update `plugin.yaml` in the same change.
- Track the vcluster API (`github.com/loft-sh/vcluster`, `github.com/loft-sh/vcluster-sdk`) carefully — plugin contract changes upstream propagate here.
