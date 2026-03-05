# Apps-of-Apps Development Template

**Archived snapshot** — see [ARCHIVED.md](ARCHIVED.md). Use as reference or copy into your repo.

Template for adding new apps under the existing ArgoCD apps-of-apps structure **without** secrets, params, or credentials. Use this for local/dev bootstrap; add Infisical/secrets/patches in real repos.

## Placeholders to replace before use

| Placeholder | Where | Example |
|-------------|--------|--------|
| `https://gitlab.labci.com/labci/ops/argocd-deployment/labci/apps-deployments.git` | All appsets, root-app-of-apps | Your Git repo URL for app manifests |
| `<tenant>` | _appset-tenant.yaml (in path and metadata.name) | `mytenant`, `uobkh`, `elw` |
| `<target-namespace>` | _appset-tenant.yaml (destination.namespace) | `mytenant`, `labci-uobkh-utrade` |

After copying `_appset-tenant.yaml` to e.g. `labci-dev-mytenant-appset.yml`, set `directories.path` to `apps/mytenant/*/overlays/dev` and `namespace` to `mytenant` so the sample apps deploy.

## Quick reference — namespaces (single-cluster)

| Component | Application | Namespace | App-of-apps |
|-----------|-------------|-----------|-------------|
| Grafana | ops-grafana | ops | dev |
| Prometheus dev | prometheus-dev | monitoring-dev | dev |
| Prometheus stg | prometheus-stg | monitoring-stg | stg |
| Fluent Bit dev | fluent-bit-dev | logging-dev | dev |
| Fluent Bit stg | fluent-bit-stg | logging-stg | stg |
| Sample apps (dev) | my-app-dev, … | mytenant | dev |
| Sample apps (stg) | my-app-stg, … | mytenant | stg |

## Structure (matches current layout)

```
template-dev/
├── README.md
├── root-app-of-apps.yaml          # Two app-of-apps: labci-dev-apps + labci-stg-apps
├── appsets/
│   ├── dev/
│   │   ├── _appset-tenant.yaml    # ApplicationSet template — copy per tenant
│   │   ├── fluent-bit-application.yaml   # Fluent Bit for dev (namespace logging-dev)
│   │   ├── grafana-application.yaml      # Grafana in ops cluster (namespace ops)
│   │   └── prometheus-application.yaml   # Prometheus for dev (namespace monitoring-dev)
│   └── stg/
│       ├── _appset-tenant.yaml
│       ├── fluent-bit-application.yaml   # Fluent Bit for stg (namespace logging-stg)
│       └── prometheus-application.yaml   # Prometheus for stg (namespace monitoring-stg)
├── values/
│   ├── fluent-bit/
│   │   ├── values.dev.yaml
│   │   └── values.stg.yaml
│   ├── grafana/
│   │   └── values.ops.yaml
│   └── prometheus/
│       ├── values.dev.yaml
│       └── values.stg.yaml
└── apps/
    └── mytenant/
        ├── my-app/                # Backend service (port 8080), dev + stg overlays
        ├── my-api/                # API (port 3000), dev + stg overlays
        ├── my-web/                # Web frontend (port 80), dev + stg overlays
        ├── my-worker/             # Background worker (no VirtualService), dev + stg
        └── my-cache/              # Redis cache (port 6379), dev + stg overlays
```

Each app has `base/` (deployment, service, hpa, configmap; + virtualservice for HTTP apps) and `overlays/dev/` and `overlays/stg/`.

## What’s included

- **Two app-of-apps**: `labci-dev-apps` syncs `appsets/dev` only; `labci-stg-apps` syncs `appsets/stg` only. Separate AppProjects for dev and stg child apps.
- **AppSets**: `appsets/dev/_appset-tenant.yaml` discovers `apps/<tenant>/*/overlays/dev`; `appsets/stg/_appset-tenant.yaml` discovers `apps/<tenant>/*/overlays/stg`. Copy and rename per tenant.
- **Sample apps**: `my-app`, `my-api`, `my-web`, `my-worker`, `my-cache` — each with base + `overlays/dev` and `overlays/stg`. No secrets/Infisical.
- **Logging (Fluent Bit)**: Same pattern as Prometheus — **managed from ops ArgoCD, one per env**. `fluent-bit-dev` in namespace `logging-dev`, `fluent-bit-stg` in `logging-stg`. Single cluster: both use `destination.server: https://kubernetes.default.svc`. Multi-cluster: set each Application’s `destination.server` to the target cluster API and add that cluster to the AppProject destinations.
- **Ops tools (Grafana / Prometheus)**: All managed from the same ops ArgoCD:
  - **Grafana**: One instance in the **ops cluster**, namespace `ops` — `appsets/dev/grafana-application.yaml` (synced by dev app-of-apps).
  - **Prometheus**: One per environment — `appsets/dev/prometheus-application.yaml` (namespace `monitoring-dev`) and `appsets/stg/prometheus-application.yaml` (namespace `monitoring-stg`). Grafana can use both as datasources.

## Before you apply

- [ ] Replace repo URL with your apps-deployments (or equivalent) Git URL in `root-app-of-apps.yaml` and in both `_appset-tenant.yaml` files if you copied them.
- [ ] Ensure ArgoCD has access to the repo and to Helm chart repos (fluent, grafana, prometheus-community).
- [ ] For multi-cluster: add each cluster to the relevant AppProject `destinations` and set each Application’s `destination.server` to the target cluster API URL.

## Usage

1. **Root**: Apply `root-app-of-apps.yaml` to create both app-of-apps (dev and stg).
2. **AppSet**: Copy `appsets/dev/_appset-tenant.yaml` to e.g. `labci-dev-mytenant-appset.yml` and `appsets/stg/_appset-tenant.yaml` to `labci-stg-mytenant-appset.yml`; replace `<tenant>` and `<target-namespace>`.
3. **New app**: Copy any `apps/mytenant/<app>/` to `apps/<your-tenant>/<your-app>/`; adjust names, image, ports, and overlay patches. Add stg overlay if missing.
4. **Fluent Bit**: One per env (`fluent-bit-dev` → `logging-dev`, `fluent-bit-stg` → `logging-stg`), managed from ops like Prometheus. For OpenSearch, create Secret `opensearch-credentials` in the app’s namespace (`logging-dev` / `logging-stg`) and set `envFrom` in the Application, or use `values/fluent-bit/values.dev.yaml` / `values.stg.yaml`. For **multi-cluster**, point each Application’s `destination.server` at the target cluster and register that cluster in the AppProject.
5. **Grafana**: Deployed with dev app-of-apps (appsets/dev). Set admin password via Secret or Helm values. Add Prometheus dev/stg as datasources (e.g. `http://prometheus-dev.monitoring-dev.svc.cluster.local`, `http://prometheus-stg.monitoring-stg.svc.cluster.local`).
6. **Prometheus**: Dev instance in `monitoring-dev`, stg in `monitoring-stg`; adjust scrape configs or ServiceMonitors per env as needed.

## Multi-cluster

For separate dev/stg clusters: (1) Register each cluster in ArgoCD (e.g. via `argocd cluster add`). (2) Add a destination entry per cluster in `labci-dev-apps-project` and `labci-stg-apps-project` (e.g. `server: https://dev-cluster.example.com`). (3) Set `spec.destination.server` in each Fluent Bit and Prometheus Application to the cluster that should run that component. Grafana stays on the ops cluster.
