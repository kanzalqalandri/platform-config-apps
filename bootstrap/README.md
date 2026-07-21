# bootstrap

**Regional hubs:** one ArgoCD per region, each managing its own region's clusters
— including the cluster it runs on. All hubs reconcile from this one repo; each
generates only its region's slice (`clusters/<region>/`).

Demo: **dev hub** = ArgoCD on `gitops-c1`, manages `gitops-c2`.
**prod hub** = ArgoCD on `gitops-c3`, manages `gitops-c3` itself.

## Standing up a hub

1. Install ArgoCD on the hub cluster (`kubectl apply -n argocd -f .../vX.Y.Z/manifests/install.yaml`).
2. Register the hub's clusters as cluster `Secret`s in ITS `argocd` namespace,
   named exactly like their registry dirs. For the hub's own cluster, the Secret
   points at `https://kubernetes.default.svc` (in-cluster) but still carries the
   registry name so `destination.name` routing stays uniform.
3. Apply the hub's root app ONCE: `kubectl apply -n argocd -f hubs/<region>/root-app.yaml`.
   From then on the hub self-manages: root app → `hubs/chart` (Helm, `region` value)
   → the 3 ApplicationSets, scoped to `clusters/<region>/`.

## Onboarding a cluster — TWO things must exist

1. A **registry file** `clusters/<region>/<cluster>/config.yaml` — the region
   path segment decides WHICH hub owns it; content (`cloud`, `role`) decides
   which add-ons it runs.
2. A **cluster Secret in the owning region's hub** (label
   `argocd.argoproj.io/secret-type: cluster`, name = registry dir name).

Registry file without Secret ⇒ apps stuck "cluster not found" on that hub.
Secret without registry file ⇒ the cluster gets nothing.
**Moving a cluster between regions** = `git mv` the registry dir + move the
Secret to the new region's hub (and any `addons/values/oneoffs/<region>/<cluster>/` placements).

The generators then auto-discover work per hub:
- `clusters/<region>/<cluster>/config.yaml` × `addons/charts/{fleet,roles/<role>,clouds/<cloud>}/`
  → one Application per (cluster, applicable addon).
- `addons/charts/oneoffs/<chart>` × `addons/values/oneoffs/<region>/<cluster>/<chart>/values.yaml`
  → one Application per placement.
- registry × `tenants/<cluster>/<tenant>/<app>/config.yaml` → tenant apps.
- registry × `powergrader/<cluster>/<env>/<app>/config.yaml` → powergrader apps.

Adding a tenant/add-on = add a file; removing one = delete it (the offboard
workflow does this, and ArgoCD cascades the cleanup).
