# 02 — Networking

## Overview

The cluster uses a layered networking model:

```
Internet
   │
   ▼
AWS Route 53  (*.ewtst.com DNS)
   │
   ▼
Contabo Public IP  (VPS node)
   │
   ▼
Traefik Ingress Controller  (in-cluster, traefik namespace)
   │
   ├──► rancher.ewtst.com  → Rancher (cattle-system)
   ├──► harbor.ewtst.com   → Harbor Registry (harbor)
   ├──► drone.ewtst.com    → Drone CI (drone)
   ├──► nextcloud.ewtst.com→ Nextcloud (nextcloud)
   └──► *.ewtst.com        → (other services)

Private / Admin Access
   │
   ▼
Tailscale VPN  (kubectl, SSH to nodes)
```

---

## 1. Tailscale — Private Cluster Access

Tailscale creates a WireGuard-based overlay network between your local machine and the cluster nodes. **No kubectl or SSH port is exposed to the public internet.**

### Setup on each node

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --authkey=<YOUR_TAILSCALE_AUTH_KEY>
```

### Usage

```bash
# Verify nodes are reachable
tailscale status

# kubectl works over Tailscale IP
export KUBECONFIG=~/.kube/config
kubectl get nodes
```

### Key points

- `kubeadm init` should advertise the **Tailscale IP** of the master as the API server address so `kubectl` routes through the VPN
- Add the Tailscale IP to the API server's `--apiserver-advertise-address` or update the kubeconfig server field after provisioning
- Both nodes must be enrolled in the same Tailscale tailnet

---

## 2. Traefik — Ingress Controller

Traefik runs in the `traefik` namespace and acts as the single entry point for all HTTP/HTTPS traffic into the cluster.

### Pod

```
traefik/traefik-c5898d9cc-qd57l   1/1   Running
```

### How it works

- Traefik watches for `Ingress` or `IngressRoute` resources across all namespaces
- When a request hits the Contabo public IP on port 80/443, Traefik routes it to the correct service based on the `Host` header
- TLS termination happens at Traefik using certificates issued by cert-manager

### Example IngressRoute (Rancher)

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: rancher
  namespace: cattle-system
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`rancher.ewtst.com`)
      kind: Rule
      services:
        - name: rancher
          port: 443
  tls:
    secretName: rancher-tls
```

### Known routes

| Subdomain | Namespace | Service |
|-----------|-----------|---------|
| `rancher.ewtst.com` | `cattle-system` | Rancher UI |
| `harbor.ewtst.com` | `harbor` | Harbor registry portal |
| `drone.ewtst.com` | `drone` | Drone CI |
| `nextcloud.ewtst.com` | `nextcloud` | Nextcloud |
| *(add others as needed)* | | |

---

## 3. AWS Route 53 — DNS

All subdomains under `ewtst.com` are managed in AWS Route 53.

### DNS record pattern

Each service gets an **A record** pointing to the Contabo VPS public IP:

```
rancher.ewtst.com   A   <CONTABO_VPS_PUBLIC_IP>   TTL 300
harbor.ewtst.com    A   <CONTABO_VPS_PUBLIC_IP>   TTL 300
drone.ewtst.com     A   <CONTABO_VPS_PUBLIC_IP>   TTL 300
nextcloud.ewtst.com A   <CONTABO_VPS_PUBLIC_IP>   TTL 300
```

> All records point to the **same IP** (the Traefik-facing node). Traefik handles the routing internally based on the `Host` header.

### Wildcard alternative

You can simplify this with a single wildcard record:
```
*.ewtst.com   A   <CONTABO_VPS_PUBLIC_IP>   TTL 300
```
Then Traefik routes each subdomain to the right service.

---

## 4. cert-manager — TLS Certificates

cert-manager runs in the `cert-manager` namespace and automates TLS certificate issuance from **Let's Encrypt**.

### Pods

```
cert-manager/cert-manager-67c98b89c8-cbztj
cert-manager/cert-manager-cainjector-5c5695d979-fzcb4
cert-manager/cert-manager-webhook-7f9f8648b9-6g85g
```

### ClusterIssuer (Let's Encrypt Production)

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your@email.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: traefik
```

### Certificate request example

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: rancher-tls
  namespace: cattle-system
spec:
  secretName: rancher-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - rancher.ewtst.com
```

### Verify certificates

```bash
kubectl get certificates --all-namespaces
kubectl get certificaterequests --all-namespaces
```

---

## 5. Network Flow Summary

```
Browser → rancher.ewtst.com
    │
    ▼ DNS lookup (Route 53)
    │  resolves to <CONTABO_PUBLIC_IP>
    │
    ▼ TCP :443
    │
    ▼ Traefik (traefik namespace)
    │  TLS terminated (cert-manager issued cert)
    │  Host: rancher.ewtst.com matched
    │
    ▼ ClusterIP Service: rancher (cattle-system:443)
    │
    ▼ Rancher Pod (cattle-system)
```

---

## 6. Firewall / Port Exposure

Only these ports should be open on the Contabo VPS public IP:

| Port | Protocol | Purpose |
|------|----------|---------|
| 80 | TCP | Traefik HTTP (redirect to HTTPS) |
| 443 | TCP | Traefik HTTPS |

All other ports (6443 API server, 22 SSH, etc.) should be **firewalled** and accessed only over Tailscale.

```bash
# Example UFW setup
sudo ufw default deny incoming
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow in on tailscale0
sudo ufw enable
```
