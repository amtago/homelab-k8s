# 10 — Nextcloud

## Overview

Nextcloud is a self-hosted file storage and collaboration platform running in the `nextcloud` namespace. It provides cloud storage, file sync, and sharing capabilities — a self-hosted alternative to Google Drive / Dropbox.

**Access:** `https://nextcloud.ewtst.com`

---

## Architecture

```
nextcloud.ewtst.com
        │
        ▼
Traefik (TLS via cert-manager)
        │
        ▼
nextcloud-0  (StatefulSet — PHP application + file storage)
        │
        ├── db-0          (MariaDB — metadata, users, shares)
        └── redis-7bf66994d7-8c446  (Redis — session cache, transactional locks)
```

---

## Pods

| Pod | Role | Age | Restarts |
|-----|------|-----|----------|
| `nextcloud-0` | Nextcloud PHP-FPM application + nginx | 109d | 0 ✅ |
| `db-0` | MariaDB database | 109d | 0 ✅ |
| `redis-7bf66994d7-8c446` | Redis cache | 106d | 0 ✅ |
| `nextcloud-cron-*` (x3) | Periodic background jobs (Completed — normal) | 14m / 9m / 4m | 0 ✅ |

All Nextcloud pods are healthy with zero restarts.

---

## Installation

Nextcloud is deployed as a StatefulSet, typically via a Helm chart or custom manifests.

### Helm install (Nextcloud community chart)

```bash
helm repo add nextcloud https://nextcloud.github.io/helm/
helm repo update

helm install nextcloud nextcloud/nextcloud \
  --namespace nextcloud \
  --create-namespace \
  --set nextcloud.host=nextcloud.ewtst.com \
  --set nextcloud.adminUser=admin \
  --set nextcloud.adminPassword=<ADMIN_PASSWORD> \
  --set mariadb.enabled=true \
  --set mariadb.auth.rootPassword=<DB_ROOT_PASS> \
  --set mariadb.auth.password=<DB_PASS> \
  --set redis.enabled=true \
  --set persistence.enabled=true \
  --set persistence.storageClass=local-path \
  --set persistence.size=50Gi
```

### Ingress / Traefik integration

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: nextcloud
  namespace: nextcloud
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`nextcloud.ewtst.com`)
      kind: Rule
      services:
        - name: nextcloud
          port: 8080
      middlewares:
        - name: nextcloud-redirectregex
  tls:
    secretName: nextcloud-tls
---
# Required middleware for Nextcloud CalDAV/CardDAV
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: nextcloud-redirectregex
  namespace: nextcloud
spec:
  redirectRegex:
    permanent: true
    regex: "https://(.*)/.well-known/(?:card|cal)dav"
    replacement: "https://${1}/remote.php/dav"
```

---

## Nextcloud `occ` — Admin CLI

The `occ` command is the primary tool for Nextcloud administration. All commands run inside the `nextcloud-0` pod.

```bash
# Base command
kubectl exec -n nextcloud nextcloud-0 -- php occ <command>

# Or open a shell
kubectl exec -it -n nextcloud nextcloud-0 -- bash
```

### Common occ commands

```bash
# Check overall status
kubectl exec -n nextcloud nextcloud-0 -- php occ status

# List all users
kubectl exec -n nextcloud nextcloud-0 -- php occ user:list

# Add a user
kubectl exec -n nextcloud nextcloud-0 -- php occ user:add <username>

# Reset a user's password
kubectl exec -n nextcloud nextcloud-0 -- php occ user:resetpassword <username>

# Check and repair the database
kubectl exec -n nextcloud nextcloud-0 -- php occ db:add-missing-indices
kubectl exec -n nextcloud nextcloud-0 -- php occ db:convert-filecache-bigint

# Run background jobs manually
kubectl exec -n nextcloud nextcloud-0 -- php occ background:cron

# Scan user files (after manual file uploads)
kubectl exec -n nextcloud nextcloud-0 -- php occ files:scan --all

# Check for updates
kubectl exec -n nextcloud nextcloud-0 -- php occ update:check

# List installed apps
kubectl exec -n nextcloud nextcloud-0 -- php occ app:list

# Enable / disable an app
kubectl exec -n nextcloud nextcloud-0 -- php occ app:enable <appname>
kubectl exec -n nextcloud nextcloud-0 -- php occ app:disable <appname>
```

---

## Background Jobs (Cron)

Nextcloud requires periodic background jobs for:
- Cleaning up temporary files
- Sending email notifications
- Indexing new files
- Expiring shares

The `nextcloud-cron-*` pods handle this. Three completed cron pods visible in the cluster output are normal — Kubernetes creates a new pod each run and the old ones linger briefly as `Completed`.

### Check cron is configured correctly

```bash
kubectl exec -n nextcloud nextcloud-0 -- php occ background:cron
# Should output: "cron" mode is already enabled
```

### Verify last cron run time (from Nextcloud admin panel)

Navigate to: `nextcloud.ewtst.com → Admin → Basic settings → Background jobs`
The last cron run time should be within the last 5–10 minutes.

---

## Maintenance Mode

Always enable maintenance mode before backups or upgrades to prevent data corruption.

```bash
# Enable
kubectl exec -n nextcloud nextcloud-0 -- php occ maintenance:mode --on

# Verify
kubectl exec -n nextcloud nextcloud-0 -- php occ status

# Disable
kubectl exec -n nextcloud nextcloud-0 -- php occ maintenance:mode --off
```

---

## Backup

### Full backup procedure

```bash
# Step 1: Enable maintenance mode
kubectl exec -n nextcloud nextcloud-0 -- php occ maintenance:mode --on

# Step 2: Backup the database
kubectl exec -n nextcloud db-0 -- \
  mysqldump -u root -p<ROOT_PASSWORD> --single-transaction nextcloud \
  > nextcloud-db-$(date +%Y%m%d-%H%M).sql

# Step 3: Backup Nextcloud data directory
kubectl exec -n nextcloud nextcloud-0 -- \
  tar czf - /var/www/html/data \
  > nextcloud-data-$(date +%Y%m%d-%H%M).tar.gz

# Step 4: Backup Nextcloud config
kubectl exec -n nextcloud nextcloud-0 -- \
  tar czf - /var/www/html/config \
  > nextcloud-config-$(date +%Y%m%d-%H%M).tar.gz

# Step 5: Disable maintenance mode
kubectl exec -n nextcloud nextcloud-0 -- php occ maintenance:mode --off

echo "Backup complete."
```

### Restore procedure

```bash
# Step 1: Enable maintenance mode
kubectl exec -n nextcloud nextcloud-0 -- php occ maintenance:mode --on

# Step 2: Restore database
kubectl cp nextcloud-db-20240101.sql nextcloud/db-0:/tmp/restore.sql
kubectl exec -n nextcloud db-0 -- \
  mysql -u root -p<ROOT_PASSWORD> -e "DROP DATABASE nextcloud; CREATE DATABASE nextcloud;"
kubectl exec -n nextcloud db-0 -- \
  mysql -u root -p<ROOT_PASSWORD> nextcloud < /tmp/restore.sql

# Step 3: Restore data
kubectl cp nextcloud-data-20240101.tar.gz nextcloud/nextcloud-0:/tmp/
kubectl exec -n nextcloud nextcloud-0 -- \
  tar xzf /tmp/nextcloud-data-20240101.tar.gz -C /

# Step 4: Fix permissions
kubectl exec -n nextcloud nextcloud-0 -- \
  chown -R www-data:www-data /var/www/html/data

# Step 5: Repair and re-scan
kubectl exec -n nextcloud nextcloud-0 -- php occ maintenance:repair
kubectl exec -n nextcloud nextcloud-0 -- php occ files:scan --all

# Step 6: Disable maintenance mode
kubectl exec -n nextcloud nextcloud-0 -- php occ maintenance:mode --off
```

---

## Upgrading Nextcloud

### Via Helm

```bash
helm repo update

# Check current version
helm list -n nextcloud

# Upgrade
helm upgrade nextcloud nextcloud/nextcloud \
  --namespace nextcloud \
  --reuse-values

# Monitor
kubectl rollout status statefulset/nextcloud -n nextcloud
```

### Post-upgrade steps

```bash
# Run upgrade finalization
kubectl exec -n nextcloud nextcloud-0 -- php occ upgrade

# Add any missing DB indices
kubectl exec -n nextcloud nextcloud-0 -- php occ db:add-missing-indices

# Convert integer fields (required after some upgrades)
kubectl exec -n nextcloud nextcloud-0 -- php occ db:convert-filecache-bigint

# Disable maintenance mode if still on
kubectl exec -n nextcloud nextcloud-0 -- php occ maintenance:mode --off
```

---

## Common Issues

### "Maintenance mode is enabled" after restart

```bash
kubectl exec -n nextcloud nextcloud-0 -- php occ maintenance:mode --off
```

### Files not showing after manual upload to PV

```bash
kubectl exec -n nextcloud nextcloud-0 -- php occ files:scan --all
```

### Database warnings in admin panel

```bash
# Fix missing indices
kubectl exec -n nextcloud nextcloud-0 -- php occ db:add-missing-indices

# Fix bigint conversion
kubectl exec -n nextcloud nextcloud-0 -- php occ db:convert-filecache-bigint
```

### Nextcloud behind reverse proxy — trusted proxies

If Nextcloud shows IP-related errors or CSRF issues, ensure Traefik's IP is trusted in `config.php`:

```bash
kubectl exec -it -n nextcloud nextcloud-0 -- bash
cat /var/www/html/config/config.php | grep trusted
```

Add if missing:
```php
'trusted_proxies' => ['10.0.0.0/8', '172.16.0.0/12'],
'overwriteprotocol' => 'https',
'overwrite.cli.url' => 'https://nextcloud.ewtst.com',
```

Or via occ:
```bash
kubectl exec -n nextcloud nextcloud-0 -- \
  php occ config:system:set trusted_proxies 0 --value="10.0.0.0/8"
kubectl exec -n nextcloud nextcloud-0 -- \
  php occ config:system:set overwriteprotocol --value="https"
```

### Redis connection error

```bash
# Check Redis is running
kubectl get pod -n nextcloud | grep redis

# Test connectivity from Nextcloud pod
kubectl exec -n nextcloud nextcloud-0 -- \
  nc -zv redis 6379

# Check Redis config in Nextcloud config.php
kubectl exec -n nextcloud nextcloud-0 -- \
  php occ config:system:get redis
```

---

## Useful Commands Summary

```bash
# Overall status
kubectl get pods -n nextcloud
kubectl exec -n nextcloud nextcloud-0 -- php occ status

# Database health
kubectl exec -n nextcloud db-0 -- \
  mysql -u root -p<PASS> -e "SHOW STATUS LIKE 'Uptime';"

# Redis health
kubectl exec -n nextcloud nextcloud-0 -- \
  redis-cli -h redis ping

# Check logs
kubectl logs -n nextcloud nextcloud-0 --tail=100
kubectl logs -n nextcloud db-0 --tail=50

# Check available disk space for Nextcloud data
kubectl exec -n nextcloud nextcloud-0 -- df -h /var/www/html/data
```
