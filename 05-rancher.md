# 05 — Rancher

## Overview

Rancher provides the cluster management UI, GitOps (Fleet), CAPI integration (Turtles), and the GitHub Actions runner provisioning. It runs in the `cattle-system` namespace and is accessible at `https://rancher.ewtst.com`.

---

## Architecture

```
rancher.ewtst.com
        │
        ▼
Traefik (TLS termination)
        │
        ▼
cattle-system/rancher (ClusterIP :443)
        │
        ├── cattle-fleet-system        (Fleet controller + gitjob + helmops)
        ├── cattle-fleet-local-system  (Fleet local agent)
        ├── cattle-capi-system         (Cluster API controller)
        └── cattle-turtles-system      (Rancher Turtles CAPI bridge)
```

---

## Installation

Rancher is installed via Helm into the `cattle-system` namespace.

### Prerequisites

- `kubectl` access to the cluster
- Helm 3 installed
- cert-manager already running (see [06-cert-manager.md](./06-cert-manager.md))

### Add Helm repo

```bash
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update
```

### Create namespace

```bash
kubectl create namespace cattle-system
```

### Install Rancher

```bash
helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --set hostname=rancher.ewtst.com \
  --set bootstrapPassword=<INITIAL_ADMIN_PASSWORD> \
  --set ingress.tls.source=letsEncrypt \
  --set letsEncrypt.email=your@email.com \
  --set letsEncrypt.ingress.class=traefik
```

> If you manage TLS externally via cert-manager + Traefik (recommended for this setup), use `--set ingress.tls.source=secret` and provide the secret manually.

### Wait for rollout

```bash
kubectl rollout status deployment/rancher -n cattle-system
kubectl get pods -n cattle-system
```

---

## Pods

| Pod | Role | Restarts |
|-----|------|----------|
| `rancher-7cfb9c6d4b-r8tg4` | Main Rancher server (UI + API) | 90 ⚠️ |
| `rancher-webhook-57fd6d9748-vrfxz` | Admission webhook — validates Rancher CRDs | 0 ✅ |

### Investigating Rancher restarts

```bash
# Check current logs
kubectl logs -n cattle-system rancher-7cfb9c6d4b-r8tg4 --tail=100

# Check previous crash logs
kubectl logs -n cattle-system rancher-7cfb9c6d4b-r8tg4 --previous

# Check resource usage
kubectl top pod -n cattle-system

# Describe for events
kubectl describe pod -n cattle-system rancher-7cfb9c6d4b-r8tg4
```

Common causes: OOM (increase memory limits), database connectivity, or webhook timeout loops.

---

## Fleet — GitOps

Fleet is Rancher's built-in GitOps engine. It watches Git repositories and continuously reconciles the cluster state to match.

**Rancher UI path:** `rancher.ewtst.com → Continuous Delivery`

### Components

| Pod | Namespace | Role | Restarts |
|-----|-----------|------|----------|
| `fleet-controller-d98b4d958-q8tv6` | `cattle-fleet-system` | Core controller; manages GitRepo → Bundle → BundleDeployment | 1053 ⚠️ |
| `gitjob-6d5dc59c4-dprj6` | `cattle-fleet-system` | Polls Git repositories for changes | 391 ⚠️ |
| `helmops-748d576d46-6sgmn` | `cattle-fleet-system` | Handles Helm-based bundle deployments | 347 ⚠️ |
| `fleet-agent-8d5bb9c7-4t7wg` | `cattle-fleet-local-system` | Applies bundles inside the local cluster | 363 ⚠️ |

### Adding a GitRepo via Rancher UI

1. Navigate to `rancher.ewtst.com → Continuous Delivery → Git Repos`
2. Click **Add Repository**
3. Fill in:
   - **Name:** your repo name
   - **Repository URL:** `https://github.com/<username>/<repo>`
   - **Branch:** `main`
   - **Paths:** subdirectory containing manifests (optional)
4. Add credentials if private repo (GitHub PAT)
5. Click **Create**

### Adding a GitRepo via kubectl

```yaml
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: my-app
  namespace: fleet-default
spec:
  repo: https://github.com/<username>/<repo>
  branch: main
  paths:
    - ./k8s
  targets:
    - name: local
      clusterSelector: {}
```

```bash
kubectl apply -f gitrepo.yaml
kubectl get gitrepo -n fleet-default
kubectl get bundle -n fleet-default
kubectl get bundledeployment -A
```

### Force a re-sync

```bash
kubectl annotate gitrepo <name> -n fleet-default \
  fleet.cattle.io/force-sync=$(date +%s) --overwrite
```

### Fleet troubleshooting

```bash
# Check GitRepo status
kubectl get gitrepo -A -o wide

# Check bundle status
kubectl get bundle -A

# Check bundle deployment errors
kubectl describe bundledeployment <name> -n <namespace>

# Fleet controller logs
kubectl logs -n cattle-fleet-system \
  $(kubectl get pod -n cattle-fleet-system -l app=fleet-controller -o name) --tail=100

# Gitjob logs
kubectl logs -n cattle-fleet-system \
  $(kubectl get pod -n cattle-fleet-system -l app=gitjob -o name) --tail=100
```

---

## Cluster API (CAPI) — Rancher Turtles

Rancher Turtles bridges Rancher with Cluster API, allowing cluster lifecycle management via CAPI providers.

### Components

| Pod | Namespace | Restarts |
|-----|-----------|----------|
| `capi-controller-manager-85dcf57d4d-k741k` | `cattle-capi-system` | 8161 ⚠️ |
| `rancher-turtles-controller-manager-dbf5f754b-qthj2` | `cattle-turtles-system` | 8241 ⚠️ |

> ⚠️ Both pods have very high restart counts. This is a known upstream instability in some Rancher Turtles versions. Steps to diagnose:

```bash
# Check CAPI controller logs
kubectl logs -n cattle-capi-system \
  capi-controller-manager-85dcf57d4d-k741k --tail=100

# Check Turtles logs
kubectl logs -n cattle-turtles-system \
  rancher-turtles-controller-manager-dbf5f754b-qthj2 --tail=100

# Check for CRD conflicts
kubectl get crds | grep cluster.x-k8s.io
```

---

## GitHub Actions Runner Provisioning

The self-hosted GitHub Actions runner is deployed inside the cluster via Rancher and registers itself with your personal GitHub account.

### Deployment (via Rancher UI)

1. In Rancher UI → select your cluster → **Workloads → Deployments**
2. Create a new Deployment in the `automation` namespace
3. Use image: `ghcr.io/actions/actions-runner:latest`
4. Set environment variables:

| Env Var | Value |
|---------|-------|
| `GITHUB_URL` | `https://github.com/<your-username>` |
| `RUNNER_TOKEN` | Registration token from GitHub → Settings → Actions → Runners → New Runner |
| `RUNNER_NAME` | e.g. `k8s-runner` |
| `RUNNER_WORKDIR` | `/tmp/runner` |

5. Mount an `emptyDir` volume at `/tmp/runner` for workspace

### Verify registration

```bash
# In GitHub: Settings → Actions → Runners
# Should show runner as "Idle" or "Active"

# In cluster:
kubectl get pods -n automation
kubectl logs -n automation <runner-pod-name>
```

### Runner token renewal

GitHub runner tokens expire. When the runner pod restarts with an auth error, generate a new token:
1. GitHub → Settings → Actions → Runners → find runner → click **...** → Re-register
2. Update the `RUNNER_TOKEN` environment variable in the Deployment

---

## Rancher Upgrade

```bash
helm repo update

helm upgrade rancher rancher-stable/rancher \
  --namespace cattle-system \
  --set hostname=rancher.ewtst.com \
  --set bootstrapPassword=<PASSWORD> \
  --set ingress.tls.source=letsEncrypt \
  --set letsEncrypt.email=your@email.com \
  --set letsEncrypt.ingress.class=traefik

kubectl rollout status deployment/rancher -n cattle-system
```

---

## Useful Commands

```bash
# Overall Rancher health
kubectl get pods -n cattle-system
kubectl get pods -n cattle-fleet-system
kubectl get pods -n cattle-fleet-local-system
kubectl get pods -n cattle-capi-system
kubectl get pods -n cattle-turtles-system

# Rancher version
kubectl get setting server-version -o jsonpath='{.value}' 2>/dev/null || \
  kubectl exec -n cattle-system deployment/rancher -- rancher --version

# Check all Fleet resources
kubectl get gitrepo,bundle,bundledeployment,clustergroup -A

# Check Rancher webhook
kubectl get validatingwebhookconfigurations | grep rancher
kubectl get mutatingwebhookconfigurations | grep rancher
```
