# 🖥️ Homelab Kubernetes Cluster — Documentation Wiki

> A self-hosted, production-grade Kubernetes cluster running on two Contabo VPS nodes, provisioned from scratch using `kubeadm`, secured via Tailscale, and exposed through Traefik + AWS Route 53.

---

## 📚 Wiki Index

| # | Document | Description |
|---|----------|-------------|
| 1 | [Cluster Architecture](./01-cluster-architecture.md) | Nodes, OS, specs, kubeadm setup, Cilium CNI |
| 2 | [Networking](./02-networking.md) | Tailscale, Traefik ingress, AWS Route 53, TLS via cert-manager |
| 3 | [Namespaces & Workloads](./03-namespaces-and-workloads.md) | All running namespaces, pods, and what each does |
| 4 | [CI/CD Pipeline](./04-cicd-pipeline.md) | GitHub Actions → Harbor → Drone → Cluster |
| 5 | [Rancher](./05-rancher.md) | Rancher install, Fleet GitOps, CAPI/Turtles, GitHub runner |
| 6 | [Storage](./06-storage.md) | Local Path Provisioner, stateful workloads, backup strategy |
| 7 | [cert-manager](./07-cert-manager.md) | TLS automation, ClusterIssuers, certificates, troubleshooting |
| 8 | [Troubleshooting Runbook](./08-troubleshooting-runbook.md) | High-restart pod diagnosis, general debug procedures, health check script |
| 9 | [Disaster Recovery](./09-disaster-recovery.md) | Node loss, etcd backup/restore, accidental deletion, recovery checklist |
| 10 | [Nextcloud](./10-nextcloud.md) | Architecture, occ CLI, backup/restore, upgrades, common issues |
| 11 | [Architecture Diagrams](./11-diagrams.md) | SVG visual diagrams — cluster overview, CI/CD, networking |

---

## 🗺️ High-Level Architecture

```
                        ┌─────────────────────────────┐
                        │        Developer (you)       │
                        │    git push → GitHub repo    │
                        └────────────┬────────────────┘
                                     │
                                     ▼
                        ┌─────────────────────────────┐
                        │      GitHub Actions          │
                        │  Build Docker image          │
                        │  Push to Harbor registry     │
                        └────────────┬────────────────┘
                                     │
                                     ▼
                        ┌─────────────────────────────┐
                        │   Harbor (in-cluster)        │
                        │   harbor.ewtst.com           │
                        └────────────┬────────────────┘
                                     │
                                     ▼
                        ┌─────────────────────────────┐
                        │   Drone CI (in-cluster)      │
                        │   Triggers deployment        │
                        └────────────┬────────────────┘
                                     │
                         ┌───────────▼────────────┐
                         │   Kubernetes Cluster    │
                         │                         │
                         │  ┌──────────────────┐  │
                         │  │  Master Node     │  │
                         │  │  Contabo VPS #1  │  │
                         │  │  4 vCPU / 8GB    │  │
                         │  └──────────────────┘  │
                         │  ┌──────────────────┐  │
                         │  │  Worker Node     │  │
                         │  │  Contabo VPS #2  │  │
                         │  │  4 vCPU / 8GB    │  │
                         │  └──────────────────┘  │
                         └───────────┬────────────┘
                                     │
                         ┌───────────▼────────────┐
                         │   Traefik Ingress        │
                         │   *.ewtst.com            │
                         │   AWS Route 53           │
                         └────────────────────────┘
```

---

![Cluster overview](./diagrams/cluster-overview.svg)

## 🔐 Access Model

- **Cluster nodes** are only reachable via **Tailscale VPN**
- `kubectl` must be run from a machine enrolled in the Tailscale network
- All public traffic enters via **Traefik** (exposed through Contabo public IPs)
- TLS is managed by **cert-manager** (Let's Encrypt)

---

## 🧰 Core Stack Summary

| Component | Tool | Notes |
|-----------|------|-------|
| Kubernetes | kubeadm | Provisioned from scratch |
| OS | Ubuntu 24.04.3 LTS | Both nodes |
| CNI | Cilium | With Envoy for L7 policy |
| Ingress | Traefik | All routes via single entrypoint |
| DNS | AWS Route 53 | `*.ewtst.com` |
| TLS | cert-manager | Let's Encrypt |
| Registry | Harbor | In-cluster, `harbor` namespace |
| CI | GitHub Actions | Build & push images |
| CD | Drone CI | Deploy to cluster |
| GitOps | Rancher Fleet | Continuous sync from Git |
| Cluster Manager | Rancher | UI + CAPI + Fleet |
| VPN | Tailscale | Secure node access |
| Storage | Local Path Provisioner | `local-path-storage` namespace |
| Cloud Storage (NC) | Nextcloud | Self-hosted, `nextcloud` namespace |
