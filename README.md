# gitops

What runs on the cluster, and which version of it.

ArgoCD watches this repo. The [lan-party
chart](https://github.com/MercuriusAalst/lan-party-helmchart) is one umbrella
release per environment — backend, frontend and the CloudNativePG database
together — so there is one `Application` per environment, not one per app.

```
core/                                    cluster-wide operators, pinned versions
  project.yaml                           AppProject: may create cluster-scoped resources
  cert-manager/application.yaml          wave -30
  external-secrets-operator/...          wave -30
  cloudnative-pg/application.yaml        wave -20
  barman-cloud/application.yaml          wave -10

workloads/lan-party/
  dev/project.yaml                       AppProject: namespaced resources only
  dev/application.yaml                   chart version  <- bot writes, on main
  dev/values.yaml                        image tags     <- bot writes, on main
  prd/project.yaml
  prd/application.yaml                   chart version  <- only via a merged PR
  prd/values.yaml                        image tags     <- only via a merged PR
```

## core

Four operators the workloads cannot run without:

| Application | Chart | Why |
|---|---|---|
| `cert-manager` | `cert-manager` v1.21.2 | issues the barman plugin's TLS certificates |
| `external-secrets-operator` | `external-secrets` 2.11.0 | serves every `ExternalSecret` the chart renders |
| `cloudnative-pg` | `cloudnative-pg` 0.29.1 | owns the `Cluster` CRD the chart renders |
| `barman-cloud` | `plugin-barman-cloud` 0.8.1 | WAL archiving and backups for that Cluster |

Sync waves order them: cert-manager and external-secrets first, then the
CloudNativePG operator, then the plugin that registers against it. The
workloads land in the default wave, after all four.

`barman-cloud` goes into `cnpg-system` on purpose — the operator only discovers
plugins in its own namespace. It is also why cert-manager is here at all: the
plugin chart will not start without it.

Versions are pinned. Renovate or a human bumps them; nothing bumps them
automatically.

## Projects

`core` may create cluster-scoped resources, because operators are CRDs,
webhooks and ClusterRoles by nature. `lan-party-dev` and `lan-party-prd` have
an empty `clusterResourceWhitelist` and a single allowed destination namespace,
so a chart change that starts creating cluster-scoped objects fails at the
project boundary instead of quietly gaining cluster-wide reach.

The chart itself comes from `oci://ghcr.io/mercuriusaalst/charts/lan-party`;
the values come from this repo, wired together by the `$values` ref in the
Application's second source.

## How a version gets here

Three repos announce releases over `repository_dispatch`:

| Sender | Event | Writes |
|---|---|---|
| `lan-party-backend` | `new-image` (`app: backend`) | `backend.image.tag` |
| `lan-party-frontend` | `new-image` (`app: frontend`) | `frontend.image.tag` |
| `lan-party-helmchart` | `new-chart` | `targetRevision` |

`.github/workflows/announce.yml` handles all three. Each one **commits the dev
bump straight to main** and **opens a PR for production**. Merging that PR is
the promotion — there is no separate promote workflow, and the PR diff is
exactly what dev has that production doesn't.

One branch per promotable thing (`promote/backend`, `promote/frontend`,
`promote/chart`), so three releases in a row update one PR rather than stacking
three, and backend can ship while frontend waits.

The bot only ever pushes to `dev/`. Production changes exclusively through a
merge — a path the workflow cannot reach, not a convention it politely follows.

## Registry access

Everything — both images and the chart — now lives on GHCR:

```
ghcr.io/mercuriusaalst/mercurius-backend    image
ghcr.io/mercuriusaalst/mercurius-frontend   image
ghcr.io/mercuriusaalst/charts/lan-party     chart
```

ArgoCD needs the chart registry registered once, from `infrastructure`, as a
repository Secret with `enableOCI: "true"`. **GHCR packages are private by
default even when the source repo is public**, which fails in two different
places: a private chart package makes the Application fail to resolve its
source, and a private image package gives you `ImagePullBackOff`. Either make
all three packages public once under the org's Packages settings, or give the
repository Secret a PAT with `read:packages` and set `imagePullSecrets` in the
values.

## Required secrets

| Secret | Where | Why |
|---|---|---|
| `GITOPS_DISPATCH_TOKEN` | the three sender repos | `GITHUB_TOKEN` cannot dispatch across repos |
| `GITOPS_PR_TOKEN` | this repo | PRs opened by `GITHUB_TOKEN` trigger no workflows |

Both are fine-grained PATs scoped to this repo: dispatch needs contents:write,
the PR token needs contents + pull-requests write.

## Editing by hand

Values files are yours — hosts, resources, storage sizes, per-env config. Only
the image tags and chart version belong to the bot. Promotion copies versions,
never whole files, so dev hostnames cannot reach production.

## Checking a change before pushing

```sh
helm template lan-party ../lan-party-helmchart --values workloads/lan-party/dev/values.yaml
```
