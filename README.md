# SecureFin GitOps Repository

## Overview

This repository is the GitOps source of truth for the **SecureFin** fintech platform. It contains Helm charts and ArgoCD ApplicationSet definitions for multi-environment deployments to a private AKS (Azure Kubernetes Service) cluster.

CI pipelines in the [secureFin-test](https://github.com/TashdidAlam/secureFin-test) repository build container images, push them to Azure Container Registry (ACR), and update the `image.tag` value in the appropriate `environments/<env>/values.yaml` file here. ArgoCD detects the change and automatically syncs the new version to the target AKS namespace.

## Structure

```
.
├── base/
│   └── securefin-app/          # Helm chart for the SecureFin application
│       ├── Chart.yaml
│       ├── values.yaml         # Default values (overridden per environment)
│       └── templates/
│           ├── _helpers.tpl
│           ├── namespace.yaml
│           ├── deployment.yaml
│           ├── service.yaml
│           ├── serviceaccount.yaml
│           ├── hpa.yaml
│           └── networkpolicy.yaml
├── environments/
│   ├── dev/values.yaml         # Dev environment overrides
│   ├── staging/values.yaml     # Staging environment overrides
│   └── production/values.yaml  # Production environment overrides
├── apps/
│   └── securefin-appset.yaml   # ArgoCD ApplicationSet definition
└── .github/
    └── workflows/
        └── validate.yml        # PR validation: helm lint & template dry-run
```

## Branching Strategy

| Branch       | Deploys to          | Purpose                              |
|--------------|---------------------|--------------------------------------|
| `dev`        | `dev` namespace     | Default branch; active development   |
| `staging`    | `staging` namespace | Pre-production validation            |
| `production` | `production` namespace | Live production workloads         |
| `dev-*`      | _(none)_            | Feature branches; PRs into `dev` only |

Feature branches (`dev-*`) are for pull requests only and are **never** deployed directly.

## How Deployments Work

1. A developer merges a PR into `dev` (or promotes to `staging`/`production`).
2. The **secureFin-test** CI builds a new container image and pushes it to ACR.
3. CI opens an automated PR (or direct commit) in this repo updating `environments/<env>/values.yaml` with the new `image.tag`.
4. ArgoCD detects the change via its automated sync policy and applies the updated Helm release to the corresponding AKS namespace.
5. ArgoCD's `selfHeal: true` ensures any out-of-band drift is automatically corrected.

```
secureFin-test CI
  └─ build & push image to ACR
  └─ update environments/<env>/values.yaml (image.tag)
        │
        ▼
secureFin-gitops (this repo)
  └─ ArgoCD ApplicationSet detects change
        │
        ▼
AKS Cluster
  └─ dev namespace       ← branch: dev
  └─ staging namespace   ← branch: staging
  └─ production namespace ← branch: production
```

## Local Development

### Prerequisites

- [Helm 3.x](https://helm.sh/docs/intro/install/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/) configured for your AKS cluster
- [ArgoCD CLI](https://argo-cd.readthedocs.io/en/stable/cli_installation/) (optional)

### Lint and validate the Helm chart locally

```bash
# Lint
helm lint base/securefin-app

# Dry-run template rendering for each environment
helm template securefin-dev base/securefin-app \
  --namespace dev \
  --values environments/dev/values.yaml

helm template securefin-staging base/securefin-app \
  --namespace staging \
  --values environments/staging/values.yaml

helm template securefin-production base/securefin-app \
  --namespace production \
  --values environments/production/values.yaml
```

### Apply the ArgoCD ApplicationSet

```bash
kubectl apply -f apps/securefin-appset.yaml -n argocd
```

> **Note:** Replace `<ACR_NAME>` in `base/securefin-app/values.yaml` and `<AZURE_CLIENT_ID_PLACEHOLDER>` in `base/securefin-app/templates/serviceaccount.yaml` with your actual Azure resource names before deploying.