# 01 — Cluster Architecture

## Overview

The cluster is a **bare-metal style Kubernetes setup** provisioned from scratch using `kubeadm` on two Contabo VPS instances. There is no managed control plane — every component runs as a pod or system service directly on the nodes.

---

## Nodes

| Role | Hostname | Provider | Plan | CPU | RAM | OS |
|------|----------|----------|------|-----|-----|----|
| Control Plane (Master) | `vmi3045386` | Contabo | VPS S | 4 vCPU | 8 GB | Ubuntu 24.04.3 LTS |
| Worker | *(your worker hostname)* | Contabo | VPS S | 4 vCPU | 8 GB | Ubuntu 24.04.3 LTS |

> **Note:** The master node in this setup also schedules workloads (taint removed or tolerated), effectively making it a combined control-plane + worker node.

You can confirm node names with:
```bash
kubectl get nodes -o wide
```

---

## Provisioning — kubeadm from Scratch

### Prerequisites (both nodes)

```bash
# Disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Load required kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

# Sysctl settings
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system
```

### Container Runtime — containerd

```bash
sudo apt-get update
sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
# Set SystemdCgroup = true in config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
```

### Install kubeadm, kubelet, kubectl

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
  https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### Initialize the Control Plane (Master only)

```bash
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=<TAILSCALE_IP_OF_MASTER>
```

After init, configure `kubectl`:
```bash
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Join the Worker Node

Run the `kubeadm join` command printed at the end of `kubeadm init` on the worker node:
```bash
sudo kubeadm join <MASTER_IP>:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

---

## CNI — Cilium

Cilium is used as the Container Network Interface (CNI). It provides:
- Pod-to-pod networking across nodes
- L7 network policy enforcement via Envoy
- eBPF-based observability

### Installation

```bash
# Install Cilium CLI
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
curl -L --remote-name-all \
  https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-amd64.tar.gz
sudo tar xzvf cilium-linux-amd64.tar.gz --directory /usr/local/bin

# Deploy Cilium
cilium install
```

### Pods running in `kube-system`

| Pod | Role |
|-----|------|
| `cilium-*` | Per-node Cilium agent (eBPF dataplane) |
| `cilium-envoy-*` | Per-node L7 proxy |
| `cilium-operator-*` | Cluster-wide controller |

### Verify

```bash
cilium status
kubectl get pods -n kube-system | grep cilium
```

---

## Control Plane Components

All running in `kube-system` namespace on the master node:

| Component | Pod Name | Role |
|-----------|----------|------|
| API Server | `kube-apiserver-vmi3045386` | Central API endpoint |
| etcd | `etcd-vmi3045386` | Cluster state store |
| Controller Manager | `kube-controller-manager-vmi3045386` | Reconciliation loops |
| Scheduler | `kube-scheduler-vmi3045386` | Pod placement decisions |
| kube-proxy | `kube-proxy-hwrfk`, `kube-proxy-k8zsj` | iptables / IPVS rules per node |
| CoreDNS | `coredns-76f75df574-*` (x2) | In-cluster DNS resolution |

---

## Storage

Local persistent storage is provided by the **Local Path Provisioner** running in the `local-path-storage` namespace. It provisions `hostPath`-based PVCs on whichever node the pod is scheduled on.

> ⚠️ Local path storage is **not replicated**. If a node goes down, PVCs on that node become unavailable. Consider this for stateful workloads like `db-0` (Nextcloud) and `harbor-database-0`.

---

## Accessing the Cluster

The API server and node SSH are **not publicly exposed**. Access requires being on the **Tailscale network**.

```bash
# From a Tailscale-enrolled machine:
kubectl get nodes

# Or SSH into a node:
ssh user@<TAILSCALE_IP_OF_NODE>
```

See [Networking](./02-networking.md) for full Tailscale setup details.
