# 06 — Storage

## Overview

Storage in this cluster is handled by the **Local Path Provisioner**, which dynamically provisions `hostPath`-based PersistentVolumes on whichever node a pod is scheduled to. There is no distributed or replicated storage layer — all data lives on the local disk of the node.

---

## Storage Architecture

```
Pod requests PVC
      │
      ▼
StorageClass: local-path (default)
      │
      ▼
local-path-provisioner (local-path-storage namespace)
      │
      ▼
hostPath volume on the node's disk
e.g. /opt/local-path-provisioner/<pvc-name>/
```

---

## Local Path Provisioner

### Pod

```
local-path-storage / local-path-provisioner-94866968d-dh8nw   1/1   Running   0   106d
```

### StorageClass

```bash
kubectl get storageclass
# NAME         PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE
# local-path   rancher.io/local-path   Delete          WaitForFirstConsumer
```

- **ReclaimPolicy: Delete** — when a PVC is deleted, the underlying hostPath data is deleted too. Back up before deleting PVCs.
- **VolumeBindingMode: WaitForFirstConsumer** — the PV is not created until a pod is scheduled, ensuring the volume is on the same node as the pod.

### Installation (if re-deploying from scratch)

```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
```

### Set as default StorageClass

```bash
kubectl patch storageclass local-path \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

---

## Stateful Workloads Using Storage

### Summary

| Namespace | PVC / StatefulSet | Data |
|-----------|-------------------|------|
| `harbor` | `harbor-database-0` | Harbor PostgreSQL metadata |
| `nextcloud` | `db-0` | Nextcloud database |
| `nextcloud` | `nextcloud-0` | Nextcloud files + config |
| `nextcloud` | `redis-7bf66994d7-8c446` | Session cache (ephemeral — loss is safe) |

---

### Harbor Database (`harbor-database-0`)

Harbor uses an internal PostgreSQL instance to store:
- Image metadata (repositories, tags, manifests)
- User accounts and RBAC
- Replication and scan policies

```bash
# Check PVC
kubectl get pvc -n harbor

# Access the database (for debugging only)
kubectl exec -it -n harbor harbor-database-0 -- psql -U postgres

# Backup Harbor database
kubectl exec -n harbor harbor-database-0 -- \
  pg_dumpall -U postgres > harbor-db-backup-$(date +%Y%m%d).sql
```

> ⚠️ Harbor database is on the **local node disk**. If the node is lost without a backup, all image metadata (not the images themselves) will be lost. Regularly back up with the command above.

---

### Nextcloud (`nextcloud-0` + `db-0`)

Nextcloud stores user files and application data across two StatefulSets:

- `nextcloud-0` — the Nextcloud PHP application + user file storage
- `db-0` — the MariaDB database holding users, shares, and metadata

```bash
# Check PVCs
kubectl get pvc -n nextcloud

# Backup Nextcloud database
kubectl exec -n nextcloud db-0 -- \
  mysqldump -u root -p<PASSWORD> nextcloud > nextcloud-db-backup-$(date +%Y%m%d).sql

# Backup Nextcloud files
kubectl exec -n nextcloud nextcloud-0 -- \
  tar czf - /var/www/html/data > nextcloud-files-backup-$(date +%Y%m%d).tar.gz

# Put Nextcloud in maintenance mode before backup
kubectl exec -n nextcloud nextcloud-0 -- \
  php occ maintenance:mode --on
# ... run backup ...
kubectl exec -n nextcloud nextcloud-0 -- \
  php occ maintenance:mode --off
```

---

## PVC Management

### List all PVCs cluster-wide

```bash
kubectl get pvc --all-namespaces
```

### List all PVs

```bash
kubectl get pv
```

### Inspect a PV to find the host path

```bash
kubectl describe pv <pv-name>
# Look for: Source.Path — e.g. /opt/local-path-provisioner/pvc-abc123.../
```

### Find the physical location on a node

```bash
# SSH into the node (via Tailscale)
ssh user@<NODE_TAILSCALE_IP>

# List local-path volumes
ls /opt/local-path-provisioner/
```

---

## Backup Strategy

Since there is no distributed storage, backups must be done manually or via CronJobs.

### Recommended approach: Velero

Velero can back up PVCs and Kubernetes resources to an S3-compatible bucket (e.g. AWS S3).

```bash
# Install Velero CLI
brew install velero  # or download from GitHub releases

# Install Velero in cluster with AWS S3 backend
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.8.0 \
  --bucket <S3_BUCKET_NAME> \
  --secret-file ./credentials-velero \
  --backup-location-config region=<AWS_REGION> \
  --snapshot-location-config region=<AWS_REGION>

# Create a manual backup
velero backup create full-cluster-backup --include-namespaces '*'

# Schedule daily backups
velero schedule create daily-backup \
  --schedule="0 2 * * *" \
  --include-namespaces harbor,nextcloud,drone
```

### Minimal manual backup script

```bash
#!/bin/bash
# backup.sh — run from a Tailscale-connected machine
DATE=$(date +%Y%m%d-%H%M)
BACKUP_DIR=./backups/$DATE
mkdir -p $BACKUP_DIR

echo "Backing up Harbor DB..."
kubectl exec -n harbor harbor-database-0 -- \
  pg_dumpall -U postgres > $BACKUP_DIR/harbor-db.sql

echo "Backing up Nextcloud DB..."
kubectl exec -n nextcloud db-0 -- \
  mysqldump -u root -p<PASSWORD> nextcloud > $BACKUP_DIR/nextcloud-db.sql

echo "Exporting all Kubernetes resources..."
kubectl get all --all-namespaces -o yaml > $BACKUP_DIR/all-resources.yaml

echo "Done. Backup saved to $BACKUP_DIR"
```

---

## ⚠️ Important Limitations

| Limitation | Impact | Mitigation |
|------------|--------|------------|
| No replication | Node failure = data loss | Regular backups + Velero |
| ReclaimPolicy: Delete | Deleting PVC deletes data | Always backup before deleting |
| Single-node binding | Pod tied to node with its PV | Don't reschedule stateful pods across nodes |
| No snapshots | No point-in-time recovery by default | Use Velero or manual `pg_dump` |

---

## Disk Usage Monitoring

```bash
# Check disk usage on a node (SSH via Tailscale)
df -h

# Check local-path volume sizes
du -sh /opt/local-path-provisioner/*/

# Check PV capacity vs usage
kubectl get pv -o custom-columns=\
NAME:.metadata.name,\
CAPACITY:.spec.capacity.storage,\
STATUS:.status.phase,\
CLAIM:.spec.claimRef.name
```
