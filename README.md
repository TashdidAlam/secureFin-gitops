# SecureFin GitOps Repository

GitOps source of truth for the **SecureFin** fintech platform. Contains Helm charts and ArgoCD ApplicationSet for multi-environment deployments to a private AKS cluster.

## Structure

```
securefin-gitops/
├── apps/
│   └── securefin-app/              # Helm chart
│       ├── Chart.yaml
│       ├── values.yaml             # Default values
│       └── templates/
│           ├── _helpers.tpl
│           ├── namespace.yaml
│           ├── deployment.yaml
│           ├── service.yaml
│           ├── serviceaccount.yaml
│           ├── hpa.yaml
│           └── networkpolicy.yaml
├── environments/
│   ├── dev/values.yaml             # Dev overrides (image repo + tag, replicas, resources)
│   ├── staging/values.yaml         # Staging overrides
│   └── production/values.yaml     # Production overrides
├── applicationsets/
│   └── appset.yaml                 # ArgoCD ApplicationSet (3 envs)
└── .github/workflows/
    └── validate.yml                # PR validation: helm lint + template
```

## End-to-End Flow

```
securefin-app repo (developer pushes code)
  │
  ▼
GitHub Actions CI:
  1. Build Docker image
  2. Trivy scan (fail on CRITICAL)
  3. Push to ACR (tag: <branch>-<sha7>)
  4. Clone securefin-gitops → update image.tag → push
  │
  ▼
securefin-gitops (this repo — updated by CI)
  │
  ▼
ArgoCD (watches this repo):
  - Detects values.yaml change
  - Renders Helm chart with new image tag
  - Applies to AKS namespace
  │
  ▼
AKS Cluster:
  └─ dev namespace        ← branch: dev
  └─ staging namespace    ← branch: staging
  └─ production namespace ← branch: production
```

## ArgoCD Setup

### Install ArgoCD

The ArgoCD installation script lives in the **infra repo** (`secureFin-test/scripts/install-argocd.sh`).
ArgoCD is infrastructure — it runs on AKS, so it belongs with the Terraform code that provisions AKS.

```bash
# From the secureFin-test repo (infra)
chmod +x scripts/install-argocd.sh
./scripts/install-argocd.sh
```

### Access ArgoCD UI (private cluster — port-forward only)

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
# → https://localhost:8080
```

### Login

```bash
# Username: admin
# Password:
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

### Apply ApplicationSet

```bash
kubectl apply -f applicationsets/appset.yaml
```

This creates 3 ArgoCD Applications automatically — one per environment.

## Validation Commands

### Check ACR images
```bash
az acr repository show-tags --name acrsecurefindev --repository securefin-app -o table
```

### Check pods
```bash
kubectl get pods -n dev
kubectl get pods -n staging
kubectl get pods -n production
```

### Port-forward the app
```bash
kubectl port-forward svc/securefin-app -n dev 9090:80
# → http://localhost:9090       (frontend)
# → http://localhost:9090/api   (backend API)
# → http://localhost:9090/healthz
```

### Test the API
```bash
curl http://localhost:9090/healthz
curl http://localhost:9090/api
```

### Check ArgoCD sync status
```bash
kubectl get applications -n argocd
```

## Local Helm Validation

```bash
helm lint apps/securefin-app

helm template securefin-dev apps/securefin-app \
  --namespace dev \
  --values environments/dev/values.yaml

helm template securefin-staging apps/securefin-app \
  --namespace staging \
  --values environments/staging/values.yaml
```

## Branching Strategy

| Branch | Deploys to | Purpose |
|--------|-----------|---------|
| `dev` | `dev` namespace | Default branch; active development |
| `staging` | `staging` namespace | Pre-production validation |
| `production` | `production` namespace | Live production workloads |