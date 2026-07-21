# platform-config-apps

**Comparison variant of [`platform-config`](https://github.com/kanzalqalandri/platform-config): the same platform, but rendered as plain ArgoCD `Application`s instead of ApplicationSets.**

The repo root is a Helm chart. `templates/` walks the plane directories with
`.Files.Glob` and emits one `Application` per leaf — the same file tree, the
same app names, the same specs the ApplicationSets generate today. Each hub
runs ONE root app (`hubs/<region>/root-app.yaml`, path `.`, `region` value)
and self-manages everything from git. **NOT live** — nothing here is applied
to the clusters; `platform-config` remains the source of truth until a
decision is made.

```
Chart.yaml                    # the repo root IS the chart, so .Files.Glob sees the planes
values.yaml                   # region (required) + repoURL/revision for the $values ref
templates/
  tenants.yaml                # replaces the tenants ApplicationSet
  powergrader.yaml            # replaces the powergrader ApplicationSet
  addons.yaml                 # replaces cluster-addons matrix 1 (registry join, tier addons)
  addons-oneoffs.yaml         # replaces cluster-addons matrix 2 (one-off placement)
hubs/<region>/root-app.yaml   # per-hub bootstrap (unchanged role; path "." now)
clusters/  tenants/  addons/  powergrader/   # plane data — copied VERBATIM, zero changes
.github/workflows/render-check.yaml          # CI: helm template both regions on every push/PR
```

## Preview — the headline feature

```sh
helm template hub . --set region=dev    # exactly what the dev hub generates
helm template hub . --set region=prod   # exactly what the prod hub generates
```

No spike appsets, no applying anything to a cluster to find out what a change
does. CI renders both regions on every PR, so a malformed plane file is caught
before merge, and `helm template | diff` against main is a real plan output.

## What changed vs the ApplicationSets (and what didn't)

| | `platform-config` (appsets) | this repo (plain apps) |
|---|---|---|
| Template languages | 2 nested (Helm renders goTemplate: every ``{{ `{{...}}` }}`` backtick escape) | 1 (plain Helm/sprig) |
| Registry join | matrix generator + `pathParamPrefix` + templated child-2 paths | a nested `range` loop |
| One-offs | 2nd matrix + generator-level template (`project` CRD quirk) | a second small loop |
| Glob semantics | `*` greedily crosses `/` (caused the dest=`prod` incident) | `*` is one segment, `**` crosses |
| Preview a change | spike-appset dance on a live hub | `helm template` locally / in CI |
| Handover/offboard safety | finalizer re-add race (2026-07-18 cascade-delete incident); needs `preserveResourcesOnDeletion` | children are plain resources of the root app; orphan with `kubectl delete app hub-<region> --cascade=orphan` |
| Install | `kubectl apply --server-side` (appset CRD > annotation limit) | any apply; no appset CRD involved at all |
| Bad-file blast radius | breaks one generator | fails the whole root render — new deploys pause fleet-wide (nothing deleted); mitigated by the render-check CI gate |
| Scaling knobs | unchanged | unchanged — add a file: registry entry, tenant leaf, tier chart, placement file, `addonOverrides` pin |
| Self-service workflows | `platform-workflows` deploy/offboard/list | identical (same tree, same paths) |

Everything else is deliberately identical: app names (`<cluster>-<tenant>-<app>`,
`addon-<cluster>-<addon>`, `powergrader-<env>-<app>`), destinations, namespaces,
values ladders, `$values` multi-source refs, sync policies, version-pin `dig`.
Verified by render: dev = 9 apps, prod = 10 apps, matching the live hubs.

## Cutover (when/if this wins)

Identical app names ⇒ adoption, not redeployment:
1. On the hub: `kubectl -n argocd delete appset tenants powergrader cluster-addons --cascade=orphan`
   (or set `preserveResourcesOnDeletion: true` first — lesson of the 2026-07-18 incident).
2. Delete the old `hub-<region>` app with `--cascade=orphan`, then
   `kubectl apply -n argocd -f hubs/<region>/root-app.yaml` from THIS repo.
3. The root app adopts the existing Applications by name; nothing on the
   workload clusters moves.

Cluster onboarding contract is unchanged (registry file + cluster Secret in the
owning hub) — see `bootstrap/README.md`.
