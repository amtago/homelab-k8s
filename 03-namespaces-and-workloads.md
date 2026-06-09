# 03 — Namespaces & Workloads

## Overview

All pods observed via `kubectl get pods --all-namespaces`. The cluster is organized into functional namespaces, each owning a distinct concern.

---

## Namespace Map

| Namespace | Purpose |
|-----------|---------|
| `kube-system` | Core Kubernetes + Cilium CNI |
| `cert-manager` | TLS certificate automation |
| `traefik` | Ingress controller |
| `cattle-system` | Rancher management UI |
| `cattle-fleet-system` | Fleet GitOps controller |
| `cattle-fleet-local-system` | Fleet local cluster agent |
| `cattle-capi-system` | Cluster API integration |
| `cattle-turtles-system` | Rancher Turtles (CAPI provider) |
| `harbor` | In-cluster container image registry |
| `drone` | Drone CI server + runner |
| `nextcloud` | Self-hosted file storage |
| `local-path-storage` | Local PVC provisioner |
| `fleet-default` | Fleet-managed workload namespace |
| `automation` | Custom automation workloads |
| `default` | Test / miscellaneous workloads |

---

## `kube-system` — Core Cluster

| Pod | Role |
|-----|------|
| `kube-apiserver-vmi3045386` | Kubernetes API server |
| `etcd-vmi3045386` | Cluster state (key-value store) |
| `kube-controller-manager-vmi3045386` | Core control loops |
| `kube-scheduler-vmi3045386` | Pod scheduling decisions |
| `kube-proxy-hwrfk` / `kube-proxy-k8zsj` | Per-node iptables rules (one per node) |
| `coredns-76f75df574-bs64b` / `...-l4b7r` | In-cluster DNS (2 replicas) |
| `cilium-725nv` / `cilium-rxp9k` | Cilium CNI agent (one per node) |
| `cilium-envoy-rlf8l` / `cilium-envoy-sglpt` | L7 proxy (one per node) |
| `cilium-operator-85d7b8db79-glqj8` | Cilium cluster controller |

---

## `cert-manager` — TLS Automation

| Pod | Role |
|-----|------|
| `cert-manager-67c98b89c8-cbztj` | Core controller; watches Certificate resources |
| `cert-manager-cainjector-5c5695d979-fzcb4` | Injects CA bundles into webhooks |
| `cert-manager-webhook-7f9f8648b9-6g85g` | Validates cert-manager API objects |

**Age:** 106 days — stable, no restarts.

---

## `traefik` — Ingress

| Pod | Role |
|-----|------|
| `traefik-c5898d9cc-qd57l` | Single ingress controller pod; all HTTP/HTTPS traffic routes through here |

**Age:** 109 days — stable, no restarts.

---

## `cattle-system` — Rancher

| Pod | Role |
|-----|------|
| `rancher-7cfb9c6d4b-r8tg4` | Rancher server (UI + API) — 90 restarts, last 9h ago |
| `rancher-webhook-57fd6d9748-vrfxz` | Validates Rancher CRD changes |

> ⚠️ `rancher` pod has **90 restarts**. Monitor with `kubectl logs -n cattle-system rancher-7cfb9c6d4b-r8tg4` to check for OOM or crash loops.

**Access:** `https://rancher.ewtst.com`

---

## `cattle-fleet-system` — Fleet GitOps

| Pod | Role |
|-----|------|
| `fleet-controller-d98b4d958-q8tv6` | Fleet controller (3/3 containers) — manages GitRepos and Bundles |
| `gitjob-6d5dc59c4-dprj6` | Polls Git repos for changes |
| `helmops-748d576d46-6sgmn` | Handles Helm-based Fleet bundles |

**Restarts:** fleet-controller has 1053 restarts (last 7h57m), gitjob 391 — these are relatively normal for Fleet in an active cluster but worth investigating if they become more frequent.

---

## `cattle-fleet-local-system` — Fleet Local Agent

| Pod | Role |
|-----|------|
| `fleet-agent-8d5bb9c7-4t7wg` | Agent running inside the local (managed) cluster to sync Fleet bundles |

---

## `cattle-capi-system` — Cluster API

| Pod | Role |
|-----|------|
| `capi-controller-manager-85dcf57d4d-k741k` | CAPI controller — manages cluster lifecycle objects |

**Restarts:** 8161 — high restart count, investigate with `kubectl describe pod -n cattle-capi-system`.

---

## `cattle-turtles-system` — Rancher Turtles

| Pod | Role |
|-----|------|
| `rancher-turtles-controller-manager-dbf5f754b-qthj2` | Bridges Rancher with CAPI providers; 8241 restarts |

> ⚠️ High restart count. This is a known upstream issue with Rancher Turtles in some versions — check Rancher Turtles release notes.

---

## `harbor` — Container Image Registry

| Pod | Role |
|-----|------|
| `harbor-core-755b49b5f8-2gkp4` | Core API + UI backend |
| `harbor-portal-5b9985797d-n24kc` | Web portal frontend |
| `harbor-registry-745dbcb84c-q99pj` | Image storage (2 containers: registry + registryctl) |
| `harbor-database-0` | PostgreSQL database for Harbor metadata |
| `harbor-jobservice-5764bfd47d-k6j42` | Async job processor (GC, replication, scan) |

**Access:** `https://harbor.ewtst.com`

> ⚠️ `harbor-core` has 5662 restarts (last 4h22m ago), `harbor-jobservice` has 654. These are high — likely caused by database connection timeouts or resource pressure. Check logs:
> ```bash
> kubectl logs -n harbor harbor-core-755b49b5f8-2gkp4
> kubectl logs -n harbor harbor-database-0
> ```

---

## `drone` — CI Runner

| Pod | Role |
|-----|------|
| `drone-55cc94b6b6-hwss5` | Drone CI server |
| `drone-runner-drone-runner-kube-55fbdcc47c-x927r` | Kubernetes runner — spins up pipeline pods |

**Access:** `https://drone.ewtst.com`

See [CI/CD Pipeline](./04-cicd-pipeline.md) for full workflow.

---

## `nextcloud` — Self-Hosted File Storage

| Pod | Role |
|-----|------|
| `nextcloud-0` | Nextcloud application (StatefulSet) |
| `db-0` | MariaDB/PostgreSQL database for Nextcloud |
| `redis-7bf66994d7-8c446` | Redis cache for Nextcloud sessions |
| `nextcloud-cron-*` (x3) | Periodic cron jobs (Completed — normal) |

**Access:** `https://nextcloud.ewtst.com`

---

## `local-path-storage` — Storage Provisioner

| Pod | Role |
|-----|------|
| `local-path-provisioner-94866968d-dh8nw` | Watches for PVC requests and creates hostPath volumes |

---

## `fleet-default` — Fleet Job Cleanup

| Pod | Status | Notes |
|-----|--------|-------|
| `rke2-machineconfig-cleanup-cronjob-*` (x3) | Completed | Periodic cleanup jobs run by Fleet; Completed status is expected |

---

## `automation` — Custom Automation

| Pod | Role |
|-----|------|
| `n8n-6b4c7b667d-6xjjv` | n8n workflow automation tool |

---

## `default` — Test Workloads

| Pod | Role |
|-----|------|
| `drone-test-app-7d5f95594c-967pr` | Test application deployed via Drone CI pipeline |

---

## Restart Count Summary & Health Notes

| Namespace | Pod | Restarts | Action Needed |
|-----------|-----|----------|---------------|
| `cattle-turtles-system` | rancher-turtles-controller-manager | 8241 | Investigate — check Turtles version |
| `cattle-capi-system` | capi-controller-manager | 8161 | Investigate — check CAPI logs |
| `harbor` | harbor-core | 5662 | Check DB connectivity + resource limits |
| `cattle-fleet-system` | fleet-controller | 1053 | Monitor — may be normal |
| `harbor` | harbor-jobservice | 654 | Monitor with harbor-core |
| `harbor` | harbor-database | 4 | Low — likely normal |
| `cattle-system` | rancher | 90 | Monitor for OOM |
| All others | — | 0 | ✅ Healthy |
