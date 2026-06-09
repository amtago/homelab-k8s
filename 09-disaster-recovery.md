# 09 — Disaster Recovery

## Overview

This document covers what to do when things go seriously wrong — from a single failed pod up to total loss of a node or the entire cluster. Given the single-master, local-storage architecture, recovery planning is critical.

---

## Risk Matrix

| Scenario | Data Loss Risk | Recovery Time | Complexity |
|----------|---------------|---------------|------------|
| Pod crash / restart | None | Seconds (auto) | None |
| Deployment rollback | None | Minutes | Low |
| Worker node lost | PVCs on that node | 30–60 min | Medium |
| Master node lost | etcd state | 1–4 hours | High |
| Both nodes lost | Everything | 4–8 hours | Very High |
| Accidental PVC deletion | All data in that PVC | Hours + backup restore | High |

---

## Scenario 1 — Pod Crash / Deployment Rollback

### Auto-recovery
Kubernetes restarts crashed pods automatically. No action needed unless it enters `CrashLoopBackOff`.

### Manual rollback to previous image

```bash
# View rollout history
kubectl rollout history deployment/<name> -n <namespace>

# Roll back to previous version
kubectl rollout undo deployment/<name> -n <namespace>

# Roll back to a specific revision
kubectl rollout undo deployment/<name> -n <namespace> --to-revision=3

# Monitor rollout
kubectl rollout status deployment/<name> -n <namespace>
```

---

## Scenario 2 — Worker Node Lost

If the Contabo worker VPS goes down or becomes permanently unavailable:

### Immediate steps

```bash
# Check node status
kubectl get nodes

# If node shows NotReady, cordon it to prevent new scheduling
kubectl cordon <worker-node-name>

# Force-delete pods stuck in Terminating on the dead node
kubectl get pods -A -o wide | grep <worker-node-name>
kubectl delete pod <pod-name> -n <namespace> --grace-period=0 --force
```

### If node is permanently lost

```bash
# Remove the node from the cluster
kubectl drain <worker-node-name> --ignore-daemonsets --delete-emptydir-data --force
kubectl delete node <worker-node-name>
```

### Restore lost PVCs

PVCs on the local disk of the lost node are gone unless you had backups. Restore procedure:

1. Restore data from most recent backup (see [06-storage.md](./06-storage.md))
2. Re-provision the node (see [01-cluster-architecture.md](./01-cluster-architecture.md))
3. Rejoin node to cluster
4. Recreate PVCs — local-path-provisioner will bind them to the new node
5. Restore data into the new volumes

```bash
# Example: restore Nextcloud DB from backup
kubectl cp nextcloud-db-backup-20240101.sql \
  nextcloud/db-0:/tmp/restore.sql

kubectl exec -n nextcloud db-0 -- \
  mysql -u root -p<PASSWORD> nextcloud < /tmp/restore.sql
```

### Provision a new worker node

```bash
# On new Contabo VPS — repeat all prerequisites from 01-cluster-architecture.md
# Then on master, generate a new join token:
kubeadm token create --print-join-command

# Run the output on the new worker node
sudo kubeadm join <MASTER_TAILSCALE_IP>:6443 \
  --token <NEW_TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

---

## Scenario 3 — Master Node Lost

This is the most serious single-node failure. The master holds etcd (cluster state), the API server, and the control plane.

### If master is temporarily unreachable

- Worker node continues running existing pods
- No new scheduling or configuration changes until master recovers
- Applications serving traffic remain up

```bash
# From Tailscale-connected machine, check if API server is reachable
kubectl cluster-info

# SSH into master via Tailscale
ssh user@<MASTER_TAILSCALE_IP>

# Check control plane components
sudo systemctl status kubelet
kubectl get pods -n kube-system
```

### If master is permanently lost — full rebuild

> ⚠️ This requires an etcd backup or you will lose all cluster state (deployments, secrets, configmaps, CRDs). Application data in PVCs on the **worker node** survives.

#### Step 1: Rebuild master node

Provision a new Contabo VPS with Ubuntu 24.04.3 LTS and repeat the setup from [01-cluster-architecture.md](./01-cluster-architecture.md):
- Install containerd
- Install kubeadm, kubelet, kubectl
- Enroll in Tailscale

#### Step 2a: Restore from etcd backup (if available)

```bash
# On new master node
sudo apt-get install -y etcd-client

# Restore etcd snapshot
sudo ETCDCTL_API=3 etcdctl snapshot restore /path/to/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-restore \
  --initial-cluster=<master>=https://<MASTER_IP>:2380 \
  --initial-advertise-peer-urls=https://<MASTER_IP>:2380 \
  --name=<master>

# Re-initialize control plane pointing at restored etcd
sudo kubeadm init --ignore-preflight-errors=DirAvailable--var-lib-etcd
```

#### Step 2b: Rebuild from scratch (no etcd backup)

```bash
# Re-initialize cluster
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=<NEW_MASTER_TAILSCALE_IP>

mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Reinstall Cilium
cilium install

# Rejoin worker node
kubeadm token create --print-join-command
# Run output on worker node
```

#### Step 3: Redeploy all workloads

After cluster rebuild without etcd, re-apply all manifests:

```bash
# If Fleet GitOps repos are intact, re-add them:
kubectl apply -f gitrepo.yaml

# Otherwise apply manifests manually:
kubectl apply -f ./k8s/

# Reinstall Helm releases:
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace --set crds.enabled=true

helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --set hostname=rancher.ewtst.com ...

helm install traefik traefik/traefik -n traefik ...
```

---

## Scenario 4 — Accidental PVC / Data Deletion

### Accidental PVC deletion

The `local-path` StorageClass has `reclaimPolicy: Delete`, meaning deleting the PVC also deletes the underlying hostPath data immediately.

**Before deleting any PVC**, always backup first:

```bash
# Backup Harbor DB
kubectl exec -n harbor harbor-database-0 -- \
  pg_dumpall -U postgres > harbor-db-$(date +%Y%m%d).sql

# Backup Nextcloud
kubectl exec -n nextcloud db-0 -- \
  mysqldump -u root -p<PASS> nextcloud > nc-db-$(date +%Y%m%d).sql
```

### Recovery after accidental deletion

1. Recreate the PVC (same name, same namespace)
2. Wait for pod to restart and mount new (empty) volume
3. Restore from backup file:

```bash
# Harbor database restore
kubectl cp harbor-db-20240101.sql harbor/harbor-database-0:/tmp/restore.sql
kubectl exec -n harbor harbor-database-0 -- \
  psql -U postgres -f /tmp/restore.sql

# Nextcloud database restore
kubectl cp nc-db-20240101.sql nextcloud/db-0:/tmp/restore.sql
kubectl exec -n nextcloud db-0 -- \
  mysql -u root -p<PASS> nextcloud < /tmp/restore.sql
```

---

## etcd Backup (Proactive)

Run this regularly from the master node. Add it to cron for automation.

```bash
# Manual etcd snapshot
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /tmp/etcd-backup-$(date +%Y%m%d-%H%M).db

# Verify the snapshot
sudo ETCDCTL_API=3 etcdctl snapshot status /tmp/etcd-backup-*.db --write-out=table
```

### Cron job on master node (daily at 2am)

```bash
sudo crontab -e

# Add:
0 2 * * * ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /var/backups/etcd/etcd-$(date +\%Y\%m\%d).db && \
  find /var/backups/etcd/ -name "*.db" -mtime +7 -delete
```

```bash
# Create backup directory
sudo mkdir -p /var/backups/etcd
```

### Copy etcd backups off-node (to S3 or local machine)

```bash
# From Tailscale-connected machine, pull latest snapshot
scp user@<MASTER_TAILSCALE_IP>:/var/backups/etcd/etcd-$(date +%Y%m%d).db ./

# Or push to S3
aws s3 cp /var/backups/etcd/etcd-$(date +%Y%m%d).db \
  s3://<YOUR_BUCKET>/etcd-backups/
```

---

## Cluster State Export (Lightweight Backup)

Even without etcd snapshots, exporting Kubernetes resources as YAML gives you a way to redeploy:

```bash
#!/bin/bash
# export-cluster-state.sh
OUTPUT_DIR=./cluster-export-$(date +%Y%m%d)
mkdir -p $OUTPUT_DIR

NAMESPACES=$(kubectl get ns -o jsonpath='{.items[*].metadata.name}')

for NS in $NAMESPACES; do
  mkdir -p $OUTPUT_DIR/$NS
  for RESOURCE in deployments statefulsets services configmaps ingresses ingressroutes certificates; do
    kubectl get $RESOURCE -n $NS -o yaml > $OUTPUT_DIR/$NS/$RESOURCE.yaml 2>/dev/null
  done
done

# Export CRDs
kubectl get crds -o yaml > $OUTPUT_DIR/crds.yaml

# Export cluster-scoped resources
kubectl get clusterroles,clusterrolebindings -o yaml > $OUTPUT_DIR/cluster-rbac.yaml
kubectl get storageclass -o yaml > $OUTPUT_DIR/storageclasses.yaml

echo "Cluster state exported to $OUTPUT_DIR"
tar czf cluster-export-$(date +%Y%m%d).tar.gz $OUTPUT_DIR
```

> ⚠️ This does **not** back up Secrets. Export secrets separately and encrypt them:
> ```bash
> kubectl get secrets -A -o yaml | gpg --symmetric > secrets-$(date +%Y%m%d).yaml.gpg
> ```

---

## Recovery Checklist

Use this after any significant recovery event:

```bash
# 1. All nodes ready?
kubectl get nodes

# 2. All system pods running?
kubectl get pods -n kube-system

# 3. Cilium healthy?
cilium status

# 4. Traefik running?
kubectl get pods -n traefik

# 5. cert-manager healthy?
kubectl get pods -n cert-manager
kubectl get certificates -A

# 6. Rancher accessible?
curl -sk https://rancher.ewtst.com | grep -i rancher

# 7. Harbor accessible?
curl -sk https://harbor.ewtst.com

# 8. Fleet syncing?
kubectl get gitrepo -A

# 9. All PVCs bound?
kubectl get pvc -A | grep -v Bound

# 10. No warning events?
kubectl get events -A --field-selector=type=Warning --sort-by='.lastTimestamp' | tail -20
```
