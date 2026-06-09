# 04 — CI/CD Pipeline

## Overview

The CI/CD pipeline is fully automated from a `git push` to a live deployment on the cluster. It combines **GitHub Actions** (build), **Harbor** (registry), and **Drone CI** (deploy), with **Rancher Fleet** handling GitOps-based continuous sync.

---

## Pipeline Flow

```
Developer
   │
   │  git push
   ▼
GitHub (personal account)
   │
   │  Triggers GitHub Actions workflow
   ▼
GitHub Actions Runner
   │  (self-hosted runner provisioned inside the cluster)
   │
   ├── Build Docker image
   ├── Tag image (e.g. harbor.ewtst.com/project/app:sha-abc123)
   └── Push to Harbor registry
           │
           ▼
   Harbor (harbor.ewtst.com)
   In-cluster registry — stores the image
           │
           │  Drone detects new image / triggered by webhook
           ▼
   Drone CI (drone.ewtst.com)
   Kubernetes runner spins up pipeline pods
           │
           ├── Pull image from Harbor
           ├── Run tests / pre-deploy steps
           └── Apply Kubernetes manifests (kubectl apply / Helm upgrade)
                   │
                   ▼
           Kubernetes Cluster
           Application pod updated with new image
```

---

## 1. GitHub Actions — Build & Push

### Self-Hosted Runner

The GitHub Actions runner runs **inside the cluster**, provisioned and managed via Rancher. This means build jobs run as Kubernetes pods.

The runner is registered to your personal GitHub account and appears under:
`GitHub → Settings → Actions → Runners`

### Workflow file example

```yaml
# .github/workflows/build.yml
name: Build and Push

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: self-hosted  # uses the in-cluster runner

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Log in to Harbor
        uses: docker/login-action@v3
        with:
          registry: harbor.ewtst.com
          username: ${{ secrets.HARBOR_USERNAME }}
          password: ${{ secrets.HARBOR_PASSWORD }}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            harbor.ewtst.com/library/${{ github.event.repository.name }}:${{ github.sha }}
            harbor.ewtst.com/library/${{ github.event.repository.name }}:latest
```

### Required GitHub Secrets

| Secret | Description |
|--------|-------------|
| `HARBOR_USERNAME` | Harbor robot account or admin username |
| `HARBOR_PASSWORD` | Harbor robot account password / token |

---

## 2. Harbor — In-Cluster Registry

Harbor serves as the private container registry. All images built by GitHub Actions are pushed here before deployment.

**URL:** `https://harbor.ewtst.com`

### Key components

| Component | Role |
|-----------|------|
| `harbor-core` | API + auth + UI logic |
| `harbor-portal` | Web UI frontend |
| `harbor-registry` | OCI-compliant image storage |
| `harbor-database` | Metadata store (PostgreSQL) |
| `harbor-jobservice` | Background jobs: GC, replication, vulnerability scanning |

### Projects structure (recommended)

```
harbor.ewtst.com/
├── library/          # default public-ish project
│   ├── app-name:sha-abc123
│   └── app-name:latest
└── private/          # private project for sensitive images
```

### Robot accounts for CI

Create a Harbor robot account for GitHub Actions to avoid using admin credentials:

1. Harbor UI → Project → Robot Accounts → New Robot Account
2. Grant `push` + `pull` permissions
3. Copy the generated token → store as GitHub secret

### Pull secret for the cluster

```bash
kubectl create secret docker-registry harbor-pull-secret \
  --docker-server=harbor.ewtst.com \
  --docker-username=<robot-account> \
  --docker-password=<token> \
  --namespace=<target-namespace>
```

Reference it in pod specs:
```yaml
spec:
  imagePullSecrets:
    - name: harbor-pull-secret
```

---

## 3. Drone CI — Deploy

Drone CI handles the deployment step. The Kubernetes runner (`drone-runner-drone-runner-kube-*`) spins up ephemeral pipeline pods inside the cluster to execute steps.

**URL:** `https://drone.ewtst.com`

### Pods

| Pod | Role |
|-----|------|
| `drone-55cc94b6b6-hwss5` | Drone server — manages pipelines, UI, webhooks |
| `drone-runner-drone-runner-kube-55fbdcc47c-x927r` | Kubernetes runner — executes pipelines as pods |

### `.drone.yml` example

```yaml
kind: pipeline
type: kubernetes
name: deploy

trigger:
  event:
    - push
  branch:
    - main

steps:
  - name: deploy
    image: harbor.ewtst.com/library/kubectl:latest
    environment:
      KUBECONFIG:
        from_secret: kubeconfig
    commands:
      - kubectl set image deployment/my-app \
          app=harbor.ewtst.com/library/my-app:${DRONE_COMMIT_SHA} \
          -n default
      - kubectl rollout status deployment/my-app -n default
```

### Drone secrets

| Secret | Description |
|--------|-------------|
| `kubeconfig` | kubeconfig file for the cluster (Tailscale IP) |

Set secrets via Drone UI: `drone.ewtst.com → Repo → Settings → Secrets`

---

## 4. Rancher Fleet — GitOps Sync

Fleet provides continuous GitOps synchronization. When manifests in a Git repo change, Fleet automatically reconciles the cluster state.

### How Fleet is set up

Fleet watches a Git repository (your GitHub account) for Kubernetes manifests or Helm charts defined in `fleet.yaml` files.

**Rancher UI:** `rancher.ewtst.com → Continuous Delivery`

### Fleet repo structure example

```
my-fleet-repo/
├── fleet.yaml          # Fleet bundle config
├── apps/
│   ├── nextcloud/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   └── drone/
│       ├── deployment.yaml
│       └── service.yaml
```

### `fleet.yaml` example

```yaml
defaultNamespace: default

helm:
  releaseName: my-app
  chart: ./charts/my-app
  values:
    image:
      repository: harbor.ewtst.com/library/my-app
      tag: latest
```

### Fleet components

| Pod | Namespace | Role |
|-----|-----------|------|
| `fleet-controller` | `cattle-fleet-system` | Watches GitRepos, creates Bundles |
| `gitjob` | `cattle-fleet-system` | Polls Git for changes |
| `helmops` | `cattle-fleet-system` | Manages Helm bundle deployments |
| `fleet-agent` | `cattle-fleet-local-system` | Applies bundles to local cluster |

---

## 5. GitHub Actions Self-Hosted Runner — Setup

The runner is provisioned inside the cluster via Rancher. It connects back to GitHub over HTTPS (no inbound port required).

### Rancher provisioning steps

1. In Rancher UI → navigate to your cluster
2. Go to **Workloads** → add a new Deployment in the relevant namespace (e.g. `automation`)
3. Use the official GitHub Actions runner image:
   ```
   ghcr.io/actions/actions-runner:latest
   ```
4. Set environment variables:
   ```
   GITHUB_URL = https://github.com/<your-username>
   RUNNER_TOKEN = <registration token from GitHub>
   ```
5. The runner registers itself and appears in GitHub → Settings → Actions → Runners

### Current runner pod

```
automation/n8n-6b4c7b667d-6xjjv   1/1   Running   0   96d
```

> *(Update with actual runner pod name if different — the `n8n` pod may be a separate automation tool)*

---

## 6. Full Pipeline Trigger Summary

| Event | Triggers | Action |
|-------|----------|--------|
| `git push` to `main` | GitHub Actions | Build image, push to Harbor |
| Image pushed to Harbor | Drone webhook | Deploy to cluster |
| Manifest change in Fleet repo | Fleet gitjob | Reconcile cluster state |
| Manual trigger | Drone UI or `drone exec` | Re-run pipeline on demand |

---

## 7. Useful Commands

```bash
# Check GitHub runner pod logs
kubectl logs -n automation <runner-pod-name>

# Check Drone server logs
kubectl logs -n drone drone-55cc94b6b6-hwss5

# Check Drone runner logs
kubectl logs -n drone drone-runner-drone-runner-kube-55fbdcc47c-x927r

# Check Fleet sync status
kubectl get gitrepo -A
kubectl get bundle -A
kubectl get bundledeployment -A

# Force Fleet re-sync
kubectl annotate gitrepo <name> -n fleet-default \
  fleet.cattle.io/force-sync=$(date +%s) --overwrite

# Harbor: list images via CLI
curl -u admin:<password> \
  https://harbor.ewtst.com/v2/library/my-app/tags/list
```
