# Kargo POC — two pipeline options (side by side)

Same Warehouse, Project, Stages names, two ways to run the dev→uat→prod pipeline.
Apply **one** stages file (they define the same Stage names, so they're mutually exclusive
in a single project — to run both at once, duplicate into a second Project/namespace).

Shared (always apply):
- `00-project.yaml`        Kargo Project `reveal-ai-poc`
- `10-warehouse.yaml`      Warehouse: subscribes to the 3 component images (podinfo/nginx/redis)
- `30-analysistemplate.yaml`  `dev-smoke` health probe used by the control option

## Option A — Visibility (git-only)   `20-stages-visibility.yaml`
Kargo only bumps versions in git; each region's own Argo CD syncs independently.
Kargo's UI is the **dashboard of which version is on dev/uat/prod**. No cluster access,
no agents, scales to N regions with zero per-cluster config. Caveat: Kargo reports
"what it promoted," not independently-verified cluster health.

## Option B — Control   `20-stages-control.yaml`
Kargo also **drives the Argo CD sync** (`argocd-update`) and **gates on a real health
check** (`spec.verification` → `dev-smoke`). 
- `dev` works as-is (its Argo CD is on the hub/control-plane cluster).
- `uat`/`prod` use `spec.shard: uat|prod` and require a **Kargo agent** running in each
  of those clusters (the agent reconciles those stages and talks to the local Argo CD).
- Each target Argo CD `Application` must carry
  `kargo.akuity.io/authorized-stage: reveal-ai-poc:<stage>`.

Apply examples:
```
kubectl apply -f 00-project.yaml -f 10-warehouse.yaml -f 30-analysistemplate.yaml
kubectl apply -f 20-stages-visibility.yaml      # Option A
# or
kubectl apply -f 20-stages-control.yaml         # Option B (uat/prod need agents)
```
