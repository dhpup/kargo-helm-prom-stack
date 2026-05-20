# kargo-helm-prom-stack

A GitOps demo showing how to use [Kargo](https://kargo.io) to automate promotion of the [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) (Prometheus, Grafana, AlertManager) across multiple environments using Helm and ArgoCD.

## Overview

This repo implements a full GitOps promotion pipeline:

1. **Warehouses** poll for new versions of the Helm chart and container images (Grafana, Prometheus, AlertManager)
2. **Kargo stages** define environment progression: `mac1` → `mac2`
3. **Promotions** automatically update chart versions and image tags, render Helm manifests, and commit them to environment-specific Git branches
4. **ArgoCD** syncs the rendered manifests to their respective clusters

```
Warehouses ──► mac1 (auto-promote) ──► mac2 (auto-promote) ──► prod (manual)
               env/mac1 branch          env/mac2 branch
```

## Repository Structure

```
kargo-helm-prom-stack/
├── kargo-addons-bootstrap.yaml            # Bootstrap ArgoCD Application
├── generate-apps.sh                       # Script to regenerate argocd/ app manifests
├── .github/workflows/
│   └── generate-apps.yaml                 # CI workflow to auto-regenerate app manifests
├── appsets/
│   └── kube-prometheus-stack.yaml         # ApplicationSet — one app per environment
├── argocd/
│   ├── appproj.yaml                       # ArgoCD AppProject
│   ├── kargo-resources-app.yaml           # ArgoCD app for Kargo resources
│   └── kube-prometheus-stack.yaml         # Static ArgoCD apps for mac1/mac2
├── kargo-resources/
│   └── kube-prometheus-stack/
│       ├── project.yaml                   # Kargo Project + promotion policies
│       ├── warehouse.yaml                 # Warehouses watching chart & image versions
│       ├── stages.yaml                    # Stage definitions (mac1, mac2)
│       ├── promotiontask.yaml             # Promotion workflow steps
│       ├── promoteRole.yaml               # RBAC for the promotion ServiceAccount
│       └── analysisTemplates.yaml         # Prometheus-based verification query
└── addons/
    └── kube-prometheus-stack/
        ├── env/
        │   ├── mac1/
        │   │   ├── Chart.yaml             # Helm chart dependency (pinned version)
        │   │   └── values.yaml            # mac1 overrides
        │   └── mac2/
        │       ├── Chart.yaml
        │       └── values.yaml            # mac2 overrides
        └── extras/
            ├── kustomization.yaml         # Builds service monitors + dashboard ConfigMap
            ├── service-monitors.yaml      # ServiceMonitors for ArgoCD metrics
            └── akuity-dashboard.json      # Grafana dashboard for Akuity/ArgoCD metrics
```

## How It Works

### Warehouses

Four warehouses run on 5-minute polling intervals:

| Warehouse | Source | Registry |
|---|---|---|
| `kube-prometheus-stack` | Helm chart | prometheus-community |
| `grafana` | Container image | docker.io/grafana/grafana |
| `alertmanager` | Container image | quay.io/prometheus/alertmanager |
| `prometheus` | Container image | quay.io/prometheus/prometheus |

When a new version is detected, a **Freight** object is created containing the pinned artifact versions.

### Stages

- **mac1** — subscribes directly to all four warehouses; auto-promotion enabled
- **mac2** — subscribes to Freight sourced from `mac1` (downstream); auto-promotion enabled
- **prod** — manual approval required (not yet wired in this demo)

### Promotion Workflow

Each promotion runs the `default-promote` task, which:

1. Clones `main` (source) and the target environment branch (`env/mac1`, `env/mac2`) as output
2. Builds addons via Kustomize (service monitors + Grafana dashboard ConfigMap)
3. Updates the Helm chart version in `Chart.yaml`
4. Updates Grafana, AlertManager, and Prometheus image tags in `values.yaml`
5. Renders the Helm chart to Kubernetes manifests
6. Commits chart/image changes back to `main`
7. Commits rendered manifests to the environment branch
8. Pushes both commits and triggers an ArgoCD sync

### ArgoCD Integration

An **ApplicationSet** uses a Git directory generator over `addons/kube-prometheus-stack/env/*` to create one ArgoCD Application per environment. Each app:

- Targets its environment-specific branch (e.g., `env/mac1`)
- Deploys to the matching cluster (e.g., `mac1`)
- Uses auto-sync with pruning, self-heal, and ServerSideApply

A separate ArgoCD Application (`kargo-resources-app.yaml`) deploys the Kargo resources themselves from the `kargo-resources/` directory.

## Bootstrap

Apply the bootstrap manifest to your `in-cluster` ArgoCD instance to seed everything:

```bash
kubectl apply -f kargo-addons-bootstrap.yaml
```

This creates the root ArgoCD Application which manages `argocd/`, which in turn creates the Kargo resources and environment applications.

## Environment Configuration

Both environments expose Prometheus, Grafana, and AlertManager via localhost ingress:

| Component | mac1 | mac2 |
|---|---|---|
| Grafana | grafana-dev.localhost | grafana-test.localhost |
| Prometheus | prometheus-dev.localhost | prometheus-test.localhost |
| AlertManager | alertmanager-dev.localhost | alertmanager-test.localhost |

Grafana default credentials: `admin` / `admin`

## Addons

The `addons/kube-prometheus-stack/extras/` directory is built by Kustomize during each promotion and includes:

- **ServiceMonitors** — scrape ArgoCD repo server and application controller metrics from the `akuity` namespace
- **Grafana dashboard** — `akuity-dashboard.json` is bundled as a ConfigMap and auto-loaded by Grafana's sidecar

## RBAC

A `kargo-promote-non-prod` ServiceAccount is provisioned with minimal permissions to promote to `mac1` and `mac2` stages only. Production promotion requires elevated access.

## Requirements

- Kubernetes cluster(s) with ArgoCD and Kargo installed
- ArgoCD clusters registered as `mac1` and `mac2`
- Git repository accessible to both Kargo and ArgoCD
