# 08 — Troubleshooting Runbook

## Overview

This runbook addresses known issues observed in this cluster, starting with the high-restart pods flagged during the initial audit, followed by general debugging procedures for common failure scenarios.

---

## 🚨 High-Restart Pod Audit

These pods were flagged during initial documentation. Investigate each one.

| Pod | Namespace | Restarts | Priority |
|-----|-----------|----------|----------|
| `rancher-turtles-controller-manager` | `cattle-turtles-system` | 8241 | 🔴 High |
| `capi-controller-manager` | `cattle-capi-system` | 8161 | 🔴 High |
| `harbor-core` | `harbor` | 5662 | 🔴 High |
| `fleet-controller` | `cattle-fleet-system` | 1053 | 🟡 Medium |
| `fleet-agent` | `cattle-fleet-local-system` | 363 | 🟡 Medium |
| `harbor-jobservice` | `harbor` | 654 | 🟡 Medium |
| `gitjob` | `cattle-fleet-system` | 391 | 🟡 Medium |
| `helmops` | `cattle-fleet-system` | 347 | 🟡 Medium |
| `rancher` | `cattle-system` | 90 | 🟡 Medium |
| `harbor-database` | `harbor` | 4 | 🟢 Low |

---

## Issue 1 — Rancher Turtles + CAPI Controller (8000+ restarts)

### Symptoms
- `rancher-turtles-controller-manager` and `capi-controller-manager` both restarting constantly
- Pods are in `Running` state but restart count is very high

### Likely causes
1. **Leader election conflict** — two controllers fighting over the same lease
2. **CRD version mismatch** — CAPI CRDs installed by Rancher Turtles conflict with existing ones
3. **Webhook timeout** — controller can't reach its own admission webhook
4. **OOMKilled** — insufficient memory

### Diagnosis steps

```bash
# Step 1: Check exit reason
kubectl describe pod -n cattle-turtles-system \
  rancher-turtles-controller-manager-dbf5f754b-qthj2 \
  | grep -A5 "Last State"

# Step 2: Check for OOMKilled
kubectl get pod -n cattle-turtles-system \
  rancher-turtles-controller-manager-dbf5f754b-qthj2 \
  -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'

# Step 3: Read crash logs
kubectl logs -n cattle-turtles-system \
  rancher-turtles-controller-manager-dbf5f754b-qthj2 --previous --tail=100

# Step 4: Check CAPI controller similarly
kubectl logs -n cattle-capi-system \
  capi-controller-manager-85dcf57d4d-k741k --previous --tail=100

# Step 5: Check for CRD conflicts
kubectl get crds | grep cluster.x-k8s.io

# Step 6: Check leader election leases
kubectl get lease -n cattle-turtles-system
kubectl get lease -n cattle-capi-system
```

### Fixes

**If OOMKilled:** Increase memory limits
```bash
kubectl edit deployment -n cattle-turtles-system rancher-turtles-controller-manager
# Increase resources.limits.memory to e.g. 512Mi or 1Gi
```

**If CRD conflict:**
```bash
# List CAPI CRDs and check versions
kubectl get crds | grep cluster.x-k8s.io | awk '{print $1}' | \
  xargs -I{} kubectl get crd {} -o jsonpath='{.metadata.name}{"\t"}{.spec.versions[*].name}{"\n"}'
```
Then check the Rancher Turtles GitHub for your installed version and upgrade if needed.

**If webhook timeout:**
```bash
# Check if webhook endpoints are reachable
kubectl get endpoints -n cattle-turtles-system
kubectl get validatingwebhookconfigurations | grep turtles
```

---

## Issue 2 — Harbor Core (5662 restarts)

### Symptoms
- `harbor-core` restarting frequently
- Harbor UI may be slow, intermittently unavailable, or return 502 errors
- `harbor-jobservice` also restarting (dependent on core)

### Likely causes
1. **Database connection pool exhausted** — harbor-core can't connect to harbor-database
2. **OOMKilled** — harbor-core running out of memory
3. **Redis connection failure** — session/cache layer unavailable
4. **Secret mismatch** — Harbor internal secrets rotated or misconfigured

### Diagnosis steps

```bash
# Step 1: Check exit reason
kubectl describe pod -n harbor harbor-core-755b49b5f8-2gkp4 \
  | grep -A5 "Last State"

# Step 2: Read crash logs
kubectl logs -n harbor harbor-core-755b49b5f8-2gkp4 --previous --tail=150

# Step 3: Check database is reachable
kubectl exec -n harbor harbor-core-755b49b5f8-2gkp4 -- \
  nc -zv harbor-database 5432

# Step 4: Check database logs
kubectl logs -n harbor harbor-database-0 --tail=50

# Step 5: Check Redis
kubectl exec -n harbor harbor-core-755b49b5f8-2gkp4 -- \
  nc -zv harbor-redis-0 6379 2>/dev/null || \
  kubectl exec -n harbor harbor-core-755b49b5f8-2gkp4 -- \
  nc -zv redis 6379

# Step 6: Check resource usage
kubectl top pod -n harbor
```

### Fixes

**If database connection issues:**
```bash
# Restart database first, then core
kubectl rollout restart statefulset/harbor-database -n harbor
# Wait for database to be ready
kubectl wait --for=condition=ready pod/harbor-database-0 -n harbor --timeout=120s
# Then restart core
kubectl rollout restart deployment/harbor-core -n harbor
```

**If OOMKilled:**
```bash
kubectl edit deployment -n harbor harbor-core
# Increase resources.limits.memory
```

**Full Harbor restart (ordered):**
```bash
# Restart in dependency order
kubectl rollout restart statefulset/harbor-database -n harbor
sleep 30
kubectl rollout restart deployment/harbor-core -n harbor
sleep 15
kubectl rollout restart deployment/harbor-jobservice -n harbor
kubectl rollout restart deployment/harbor-registry -n harbor
kubectl rollout restart deployment/harbor-portal -n harbor
```

---

## Issue 3 — Fleet Controller / GitJob / HelmOps (300-1000+ restarts)

### Symptoms
- Fleet GitOps sync may be delayed or stuck
- `kubectl get gitrepo -A` shows repos in error state
- Bundle deployments not reconciling

### Likely causes
1. **GitHub rate limiting** — gitjob polling too frequently
2. **Webhook conflicts** — Fleet webhook unreachable
3. **Memory pressure** — fleet-controller OOMKilled on large bundle sets
4. **Rancher connection loss** — fleet-controller loses connection to Rancher API

### Diagnosis steps

```bash
# Step 1: Check fleet-controller logs
kubectl logs -n cattle-fleet-system \
  $(kubectl get pod -n cattle-fleet-system -l app=fleet-controller -o name) \
  --previous --tail=100

# Step 2: Check gitjob logs
kubectl logs -n cattle-fleet-system \
  $(kubectl get pod -n cattle-fleet-system -l app=gitjob -o name) \
  --previous --tail=100

# Step 3: Check GitRepo status
kubectl get gitrepo -A -o wide

# Step 4: Check for stuck bundles
kubectl get bundle -A | grep -v Active

# Step 5: Check fleet-agent connection
kubectl logs -n cattle-fleet-local-system \
  $(kubectl get pod -n cattle-fleet-local-system -l app=fleet-agent -o name) \
  --tail=50
```

### Fixes

**Force re-sync a stuck GitRepo:**
```bash
kubectl annotate gitrepo <name> -n fleet-default \
  fleet.cattle.io/force-sync=$(date +%s) --overwrite
```

**Restart all Fleet components in order:**
```bash
kubectl rollout restart deployment/fleet-controller -n cattle-fleet-system
kubectl rollout restart deployment/gitjob -n cattle-fleet-system
kubectl rollout restart deployment/helmops -n cattle-fleet-system
sleep 20
kubectl rollout restart deployment/fleet-agent -n cattle-fleet-local-system
```

---

## Issue 4 — Rancher Server (90 restarts)

### Symptoms
- Rancher UI intermittently unavailable
- `kubectl` operations via Rancher fail
- Fleet/CAPI operations slow to respond

### Diagnosis steps

```bash
# Current logs
kubectl logs -n cattle-system rancher-7cfb9c6d4b-r8tg4 --tail=100

# Previous crash logs
kubectl logs -n cattle-system rancher-7cfb9c6d4b-r8tg4 --previous --tail=100

# Check exit code
kubectl describe pod -n cattle-system rancher-7cfb9c6d4b-r8tg4 \
  | grep -A5 "Last State"

# Resource usage
kubectl top pod -n cattle-system
```

### Fix — increase Rancher memory limit

```bash
helm upgrade rancher rancher-stable/rancher \
  --namespace cattle-system \
  --reuse-values \
  --set resources.limits.memory=2Gi \
  --set resources.requests.memory=1Gi
```

---

## General Debugging Procedures

### Pod won't start / CrashLoopBackOff

```bash
# Full diagnosis sequence
kubectl get pod <name> -n <namespace>
kubectl describe pod <name> -n <namespace>      # events section is key
kubectl logs <name> -n <namespace> --previous   # last crash logs
kubectl logs <name> -n <namespace>              # current logs

# Check resource limits
kubectl get pod <name> -n <namespace> \
  -o jsonpath='{.spec.containers[0].resources}'
```

### Node not ready

```bash
# Check node status
kubectl get nodes
kubectl describe node <node-name>

# Check kubelet on the node (SSH via Tailscale)
ssh user@<NODE_TAILSCALE_IP>
sudo systemctl status kubelet
sudo journalctl -u kubelet -n 100 --no-pager

# Check containerd
sudo systemctl status containerd
sudo journalctl -u containerd -n 50 --no-pager

# Check disk space
df -h
```

### Service not reachable

```bash
# Check service exists and has endpoints
kubectl get svc <name> -n <namespace>
kubectl get endpoints <name> -n <namespace>

# Check if pods are ready
kubectl get pods -n <namespace> -l <selector>

# Test from inside the cluster
kubectl run -it --rm debug --image=busybox --restart=Never -- \
  wget -qO- http://<service-name>.<namespace>.svc.cluster.local:<port>
```

### DNS resolution failing

```bash
# Test DNS from inside a pod
kubectl run -it --rm dnstest --image=busybox --restart=Never -- \
  nslookup kubernetes.default

# Check CoreDNS
kubectl get pods -n kube-system | grep coredns
kubectl logs -n kube-system \
  $(kubectl get pod -n kube-system -l k8s-app=kube-dns -o name | head -1) --tail=50
```

### Ingress / Traefik not routing

```bash
# Check Traefik pod
kubectl get pods -n traefik
kubectl logs -n traefik $(kubectl get pod -n traefik -o name) --tail=100

# Check IngressRoutes
kubectl get ingressroute -A
kubectl describe ingressroute <name> -n <namespace>

# Check Traefik service and external IP
kubectl get svc -n traefik

# Test TLS cert
curl -v https://<subdomain>.ewtst.com 2>&1 | grep -E "subject|issuer|expire"
```

### PVC stuck in Pending

```bash
# Describe PVC for events
kubectl describe pvc <name> -n <namespace>

# Check local-path-provisioner logs
kubectl logs -n local-path-storage \
  $(kubectl get pod -n local-path-storage -o name) --tail=50

# Check node disk space
kubectl describe nodes | grep -A5 "Allocated resources"
```

### etcd unhealthy

```bash
# SSH into master node
ssh user@<MASTER_TAILSCALE_IP>

# Check etcd health
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health

# Check etcd disk usage (etcd is sensitive to slow disk)
sudo du -sh /var/lib/etcd/
```

---

## Quick Health Check Script

Save this as `health-check.sh` and run periodically from a Tailscale-connected machine:

```bash
#!/bin/bash
echo "=============================="
echo " Cluster Health Check"
echo " $(date)"
echo "=============================="

echo ""
echo "--- Nodes ---"
kubectl get nodes -o wide

echo ""
echo "--- Not-Running Pods ---"
kubectl get pods --all-namespaces --field-selector=status.phase!=Running \
  | grep -v Completed

echo ""
echo "--- High Restart Pods (>50) ---"
kubectl get pods --all-namespaces \
  -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{range .status.containerStatuses[*]}{.restartCount}{"\n"}{end}{end}' \
  | awk '$3 > 50 {print}' | sort -k3 -rn

echo ""
echo "--- PVC Status ---"
kubectl get pvc --all-namespaces

echo ""
echo "--- Certificate Status ---"
kubectl get certificates --all-namespaces

echo ""
echo "--- Fleet GitRepo Status ---"
kubectl get gitrepo --all-namespaces -o wide

echo ""
echo "--- Node Disk Usage (requires Tailscale SSH) ---"
echo "Run: df -h on each node"
```

```bash
chmod +x health-check.sh
./health-check.sh
```

---

## Useful One-Liners

```bash
# All pods not in Running or Completed state
kubectl get pods -A --field-selector=status.phase!=Running | grep -v Completed

# Top pods by restart count
kubectl get pods -A -o jsonpath='{range .items[*]}{.status.containerStatuses[0].restartCount}{"\t"}{.metadata.namespace}{"\t"}{.metadata.name}{"\n"}{end}' | sort -rn | head -20

# All pods on a specific node
kubectl get pods -A -o wide | grep <node-name>

# Events sorted by time (last 50)
kubectl get events -A --sort-by='.lastTimestamp' | tail -50

# Warning events only
kubectl get events -A --field-selector=type=Warning --sort-by='.lastTimestamp'

# Check all image versions running
kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{.spec.containers[0].image}{"\n"}{end}' | sort

# Force delete a stuck terminating pod
kubectl delete pod <name> -n <namespace> --grace-period=0 --force
```
