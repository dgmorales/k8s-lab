# AGENTS.md

This file provides guidance to AI Agents when working with code in this repository.

## What this is

A lab/testbed for studying Kubernetes infrastructure techniques

- ClusterAPI for provisioning clusters,
- Cilium as CNI
- ArgoCD for GitOps management of cluster addons and applications
- Kargo, to complement ArgoCD with promotion and pipelines
- GitHub Actions Runner Controller
- ... among others

There is no application source code to build/lint/test — this repo is
almost entirely Kubernetes manifests, Helm values, and [Taskfile](https://taskfile.dev) task
definitions.

`setup/cilium` is a git submodule pointing at `cilium/cilium` upstream — it's vendored for its
`cilium-cli`/chart tooling, not a directory to edit.

## Commands

All operational commands are `task` targets (root `Taskfile.yml`, which includes the sub-Taskfiles
under `addons/`, `setup/`, and `demos/`). List everything with `task --list`; get details on any
task with `task --summary <name>`.

Bring-up / teardown of the whole lab:

- `task setup-tools` — installs `clusterctl`, `argocd`, `cilium` CLIs to `$HOME/bin` (or a path you pass as an arg)
- `task setup-lab` — full bring-up: creates the mgmt kind cluster, installs ArgoCD, bootstraps the addons and apps app-of-apps roots, initializes ClusterAPI, and creates the workload clusters
- `task destroy-clusters` — deletes all four kind clusters (mgmt, dev-0, prod-0, prod-1)
- `task port-forward` — port-forwards both ArgoCD and Cilium (Hubble) UIs at once

Individual steps (all namespaced by the sub-Taskfile they live in, e.g. `task clusterapi:init`):

- `task mgmt-create` — just the mgmt kind cluster (`setup/kind-config.yaml`)
- `task addons-init` — installs both the `addons-manager` and `apps-manager` app-of-apps roots into ArgoCD; requires `ARC_TOKEN` env var (a GitHub PAT for the Actions Runner Controller) and runs `kargo:init` as a dependency
- `task argocd:install` / `task argocd:port-forward` / `task argocd:get-password`
- `task kargo:init` / `task kargo:get-password` / `task kargo:port-forward` — Kargo's admin secret is generated out-of-band (bcrypt hash + signing key) rather than committed; `init` prints the plaintext password once, `get-password` re-fetches it from the `kargo-admin` secret
- `task clusterapi:init` — `clusterctl init` for the docker infra provider
- `task clusterapi:create-clusters` — applies the ClusterClass and the three workload `Cluster` manifests, waits for them, fetches kubeconfigs, and registers them with ArgoCD
- `task clusterapi:register-argocd-clusters` — (re-)registers the workload clusters with the mgmt ArgoCD, labelling each `type=worker` and the mgmt in-cluster entry `type=mgmt` (generators select on this label)
- `task clusterapi:local-kubeconfig` — rewrites the CAPD clusters' kubeconfigs to use the host-published load-balancer port, for use with `kubectl`/`k9s` from the host; re-run whenever cluster LB containers are recreated, since ports are reassigned
- `task clusterapi:render-cilium-manifest` — regenerates `setup/clusterapi/cilium-cni-configmap.yaml` from the Cilium Helm chart; run after bumping the chart version, and keep the pod CIDR in the `--set` flag in sync with the `Cluster` manifests' `clusterNetwork.pods.cidrBlocks`
- `task cilium:install` / `task cilium:install-ui`
- `task demo-cilium:test-conn` / `deathstar-up` / `deathstar-down` — Cilium network-policy demos

## Architecture

**Cluster topology**: one `kind` management cluster runs ArgoCD, Kargo, and the GitHub Actions
Runner Controller. Three workload clusters (`lab-dev-0`, `lab-prod-0`, `lab-prod-1`) are provisioned
via ClusterAPI's Docker infra provider (CAPD) from a shared `ClusterClass` (`quick-start`) defined in
`setup/clusterapi/`, with Cilium installed into them as CNI through a `ClusterResourceSet` (a
pre-rendered manifest, not `cilium install`, since CAPD clusters aren't reachable for direct CLI
install at creation time). All four clusters are registered in the mgmt cluster's ArgoCD, labelled
`type=mgmt` or `type=worker` — generators use this label to target "all workers" (or all clusters)
without naming clusters individually.

**Networking gotcha**: CAPD workload clusters' API servers are only reachable over the internal
`kind` Docker network, not from the host directly. Tasks that need to reach them from automation
(cluster registration) run a throwaway container attached to that network rather than calling
`argocd`/`kubectl` from the host; see `register-argocd-clusters` in `setup/clusterapi/Taskfile.yml`
for the pattern if extending it.

**GitOps layout** — two top-level app-of-apps roots (`addons/addons-manager.yaml`,
`apps/apps-manager.yaml`), both applied once by `task addons-init` and self-managing from then on:

- `addons/` — cluster infrastructure, split by target scope. `addons-manager` is a multi-source
  Application aggregating all three:
  - `addons/mgmt/apps/` — Applications that only make sense on the mgmt cluster (ArgoCD
    self-management, GitHub ARC, Kargo), each pointing at its raw manifests/values under
    `addons/mgmt/manifests/<addon>/`.
  - `addons/all/appsets/` — `ApplicationSet`s for addons that belong on every cluster (e.g.
    `cert-manager`), using a `clusters` generator with no selector so it fans out to mgmt and all
    workers.
  - `addons/workers/` — placeholder for addons that should only land on worker clusters (empty so
    far).
- `apps/` — workload applications. `apps-manager` points at `apps/appsets/`, which holds
  `ApplicationSet`s (e.g. `guestbook`) using a `clusters` generator selecting `type: worker`, so
  they land on every workload cluster and never on mgmt. Manifests live under
  `apps/manifests/<app>/`, with per-cluster/per-environment Helm values layered under
  `apps/values/<app>/{cluster,env}`.

**Secrets**: nothing sensitive is committed. The GitHub ARC controller secret and the Kargo admin
secret are created directly with `kubectl`/`openssl`/`htpasswd` by Task commands (`addons-init`,
`kargo:init`) and only referenced by name from the Application manifests.

Conventional commit scopes used in this repo (see `.vscode/settings.json`): `argocd`, `cilium`,
`clusterapi`, `addons`, `apps`.
