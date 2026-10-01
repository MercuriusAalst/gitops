# gitops

What runs on the cluster, and which version of it.

ArgoCD watches this repo. The [lan-party
chart](https://github.com/MercuriusAalst/lan-party-helmchart) is one umbrella
release per environment — backend, frontend and the CloudNativePG database
together — so there is one `Application` per environment, not one per app.

```
dev/application.yaml   chart version    <- bot writes, on main
dev/values.yaml        image tags       <- bot writes, on main
prd/application.yaml   chart version    <- only via a merged promotion PR
prd/values.yaml        image tags       <- only via a merged promotion PR
```

The chart itself comes from `oci://registry-1.docker.io/livingwooods/lan-party`;
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
helm template lan-party ../lan-party-helmchart --values dev/values.yaml
```
