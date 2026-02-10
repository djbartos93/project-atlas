# K3s Kubernetes Deployment on Proxmox

## NOT REVIEWED YET

## Overview

This guide walks through deploying a **production-grade K3s Kubernetes cluster** on your Proxmox environment. K3s is a lightweight, certified Kubernetes distribution perfect for homelabs - it uses 50% less resources than full K8s while maintaining full compatibility.

### Why K3s Instead of Full Kubernetes?

**K3s benefits**:
- Single binary (< 100MB vs multi-GB for K8s)
- Embedded SQLite or etcd (no separate etcd cluster needed)
- Lower resource usage (512MB RAM vs 2GB+ for K8s)
- Same API as full Kubernetes (all kubectl commands work)
- Production-ready (used by SUSE Rancher, edge computing)

**When to use full K8s**:
- Need specific cloud provider integrations
- Require specific storage backends
- Learning for specific K8s cert (CKA, CKAD)

### What You'll Deploy

```
Proxmox Cluster (3 hosts: pve1, pve2, pve3)
│
├── K3s Control Plane VMs (3x for HA)
│   ├── k3s-cp-1 (pve1): 2 vCPU, 4GB RAM
│   ├── k3s-cp-2 (pve2): 2 vCPU, 4GB RAM
│   └── k3s-cp-3 (pve3): 2 vCPU, 4GB RAM
│
└── K3s Worker VMs (3-6x for workloads)
    ├── k3s-worker-1 (pve1): 4 vCPU, 8GB RAM
    ├── k3s-worker-2 (pve2): 4 vCPU, 8GB RAM
    ├── k3s-worker-3 (pve3): 4 vCPU, 8GB RAM
    └── (optional) k3s-worker-4,5,6...

Total Resources: ~18 vCPU, ~36GB RAM for HA cluster
```

---

## Phase 1: Infrastructure Preparation

### Goal
Prepare Proxmox environment and networking for K8s deployment.

### Understanding Kubernetes Networking Requirements

**What K8s needs**:
- **Node network**: VMs talk to each other (192.168.100.0/24)
- **Pod network**: Pods get IPs (10.42.0.0/16 - K3s default)
- **Service network**: ClusterIPs for services (10.43.0.0/16 - K3s default)
- **LoadBalancer IPs**: External access (192.168.100.200-250 - MetalLB)

**Why these networks?**:
- Separation prevents conflicts
- Allows network policies
- Enables service mesh later

### Step 1: Network Planning

**Node IPs** (on your existing network):

| VM Name | IP Address | Proxmox Host | Role |
|---------|------------|--------------|------|
| k3s-cp-1 | 192.168.100.101 | pve1 | Control plane |
| k3s-cp-2 | 192.168.100.102 | pve2 | Control plane |
| k3s-cp-3 | 192.168.100.103 | pve3 | Control plane |
| k3s-worker-1 | 192.168.100.111 | pve1 | Worker |
| k3s-worker-2 | 192.168.100.112 | pve2 | Worker |
| k3s-worker-3 | 192.168.100.113 | pve3 | Worker |

**LoadBalancer IP range**: 192.168.100.200-250 (reserved for MetalLB)

**Why spread across Proxmox hosts?**:
- High availability (if one host fails, cluster continues)
- Resource distribution
- Avoid single point of failure

### Step 2: Create VM Template

**Why use a template?**: Creates VMs faster, ensures consistency

**Download Ubuntu cloud image**:

```bash
# On Proxmox host pve1
cd /var/lib/vz/template/iso/
wget https://cloud-images.ubuntu.com/releases/22.04/release/ubuntu-22.04-server-cloudimg-amd64.img

# Or use Rocky Linux for RHEL-like experience
# wget https://download.rockylinux.org/pub/rocky/9/images/x86_64/Rocky-9-GenericCloud-Base.latest.x86_64.qcow2
```

**Create template VM**:

```bash
# Create VM (ID 9000 for template)
qm create 9000 --name k3s-template --memory 4096 --cores 2 --net0 virtio,bridge=vmbr0

# Import cloud image as disk
qm importdisk 9000 ubuntu-22.04-server-cloudimg-amd64.img local-lvm

# Attach disk to VM
qm set 9000 --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-9000-disk-0

# Add cloud-init drive
qm set 9000 --ide2 local-lvm:cloudinit

# Set boot disk
qm set 9000 --boot c --bootdisk scsi0

# Add serial console
qm set 9000 --serial0 socket --vga serial0

# Enable QEMU agent
qm set 9000 --agent enabled=1

# Resize disk (20GB is enough for K8s node)
qm resize 9000 scsi0 20G

# Convert to template
qm template 9000
```

**Why cloud-init?**: Allows injecting SSH keys, network config without manual setup

### Step 3: Create Control Plane VMs

**Clone from template**:

```bash
# On pve1 - create k3s-cp-1
qm clone 9000 101 --name k3s-cp-1 --full

# Set cloud-init values
qm set 101 --ipconfig0 ip=192.168.100.101/24,gw=192.168.100.1
qm set 101 --nameserver 8.8.8.8
qm set 101 --sshkeys ~/.ssh/id_rsa.pub  # Your SSH key
qm set 101 --ciuser admin

# Adjust resources for control plane
qm set 101 --memory 4096 --cores 2

# Start VM
qm start 101
```

**Repeat for k3s-cp-2 and k3s-cp-3** on pve2 and pve3

**Alternative: Use Terraform** (automates this):

```hcl
# terraform/k3s-cluster.tf
resource "proxmox_vm_qemu" "k3s_control_plane" {
  count       = 3
  name        = "k3s-cp-${count.index + 1}"
  target_node = var.proxmox_nodes[count.index]  # Spread across hosts
  clone       = "k3s-template"

  cores   = 2
  memory  = 4096

  ipconfig0 = "ip=192.168.100.${101 + count.index}/24,gw=192.168.100.1"
  sshkeys   = file("~/.ssh/id_rsa.pub")
}
```

### Step 4: Create Worker VMs

**Same process**, but:
- More resources (4 vCPU, 8GB RAM)
- Different IP range (111+)

```bash
# On pve1 - create k3s-worker-1
qm clone 9000 111 --name k3s-worker-1 --full
qm set 111 --ipconfig0 ip=192.168.100.111/24,gw=192.168.100.1
qm set 111 --memory 8192 --cores 4
qm set 111 --sshkeys ~/.ssh/id_rsa.pub
qm set 111 --nameserver 8.8.8.8
qm start 111
```

**Repeat for workers 2-6** across your Proxmox hosts

---

## Phase 2: System Preparation

### Goal
Prepare all VMs for K3s installation.

### Step 1: SSH to All Nodes

**Verify SSH access**:

```bash
# From your laptop/workstation
ssh admin@192.168.100.101  # k3s-cp-1

# Or create SSH config (~/.ssh/config)
Host k3s-cp-*
  User admin
  IdentityFile ~/.ssh/id_rsa

Host k3s-worker-*
  User admin
  IdentityFile ~/.ssh/id_rsa
```

### Step 2: System Updates and Prerequisites

**Run on ALL nodes** (use parallel-ssh or Ansible):

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install prerequisites
sudo apt install -y curl open-iscsi nfs-common

# Disable swap (K8s requirement)
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Load kernel modules for K8s
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Sysctl settings for K8s
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system

# Verify
lsmod | grep br_netfilter
lsmod | grep overlay
sysctl net.ipv4.ip_forward  # Should be 1
```

**Why these settings?**:
- `br_netfilter`: Enables iptables for bridge traffic (K8s networking)
- `ip_forward`: Allows routing between pod network and outside
- `swap off`: K8s doesn't support swap (causes performance issues)

### Step 3: Set Hostnames

**On each node**:

```bash
# k3s-cp-1
sudo hostnamectl set-hostname k3s-cp-1

# Add to /etc/hosts (all nodes)
cat <<EOF | sudo tee -a /etc/hosts
192.168.100.101 k3s-cp-1
192.168.100.102 k3s-cp-2
192.168.100.103 k3s-cp-3
192.168.100.111 k3s-worker-1
192.168.100.112 k3s-worker-2
192.168.100.113 k3s-worker-3
EOF
```

---

## Phase 3: K3s Installation

### Goal
Deploy K3s in HA mode with embedded etcd.

### Understanding K3s HA Architecture

**Components**:
- **Control plane nodes**: Run K3s server with embedded etcd
- **Worker nodes**: Run K3s agent
- **Embedded etcd**: Quorum-based (needs 3+ servers for HA)
- **Kube-VIP**: Provides floating IP for API server

### Step 1: Install First Control Plane Node

**On k3s-cp-1**:

```bash
# Set environment variables
export K3S_TOKEN="your-super-secret-token-here"  # Generate with: openssl rand -base64 32
export K3S_KUBECONFIG_MODE="644"

# Install K3s in cluster mode
curl -sfL https://get.k3s.io | sh -s - server \
  --cluster-init \
  --tls-san=192.168.100.101 \
  --tls-san=192.168.100.100 \
  --disable traefik \
  --disable servicelb \
  --write-kubeconfig-mode=644

# Verify installation
sudo systemctl status k3s
sudo kubectl get nodes
```

**Key flags explained**:
- `--cluster-init`: Initialize embedded etcd cluster
- `--tls-san`: Add IPs to API server certificate (for HA IP later)
- `--disable traefik`: We'll install Nginx Ingress instead
- `--disable servicelb`: We'll use MetalLB instead

**Expected output**:
```
NAME       STATUS   ROLES                       AGE   VERSION
k3s-cp-1   Ready    control-plane,etcd,master   1m    v1.28.5+k3s1
```

### Step 2: Install Additional Control Plane Nodes

**On k3s-cp-2 and k3s-cp-3**:

```bash
# Use same token as cp-1
export K3S_TOKEN="your-super-secret-token-here"

# Install and join cluster
curl -sfL https://get.k3s.io | sh -s - server \
  --server https://192.168.100.101:6443 \
  --tls-san=192.168.100.102 \
  --tls-san=192.168.100.100 \
  --disable traefik \
  --disable servicelb

# Verify
sudo kubectl get nodes
```

**Expected**: All 3 control plane nodes in `Ready` state

**Check etcd cluster**:

```bash
sudo kubectl get nodes -o wide
sudo kubectl get pods -n kube-system | grep etcd
```

### Step 3: Install Worker Nodes

**On k3s-worker-1, k3s-worker-2, k3s-worker-3**:

```bash
# Use same token
export K3S_TOKEN="your-super-secret-token-here"
export K3S_URL="https://192.168.100.101:6443"

# Install K3s agent (worker)
curl -sfL https://get.k3s.io | K3S_URL=$K3S_URL K3S_TOKEN=$K3S_TOKEN sh -

# Verify
sudo systemctl status k3s-agent
```

**Check from control plane**:

```bash
# On k3s-cp-1
kubectl get nodes

# Expected output:
# NAME           STATUS   ROLES                       AGE     VERSION
# k3s-cp-1       Ready    control-plane,etcd,master   10m     v1.28.5+k3s1
# k3s-cp-2       Ready    control-plane,etcd,master   5m      v1.28.5+k3s1
# k3s-cp-3       Ready    control-plane,etcd,master   5m      v1.28.5+k3s1
# k3s-worker-1   Ready    <none>                      2m      v1.28.5+k3s1
# k3s-worker-2   Ready    <none>                      2m      v1.28.5+k3s1
# k3s-worker-3   Ready    <none>                      2m      v1.28.5+k3s1
```

---

## Phase 4: Cluster Access

### Goal
Set up kubectl access from your workstation.

### Step 1: Copy Kubeconfig

**From k3s-cp-1**:

```bash
# Copy kubeconfig to your laptop
scp admin@192.168.100.101:/etc/rancher/k3s/k3s.yaml ~/.kube/k3s-config

# Edit server IP (change 127.0.0.1 to actual IP)
sed -i 's/127.0.0.1/192.168.100.101/' ~/.kube/k3s-config

# Set KUBECONFIG
export KUBECONFIG=~/.kube/k3s-config

# Test
kubectl get nodes
```

### Step 2: Install kubectl (if not already)

```bash
# On macOS
brew install kubectl

# On Linux
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Verify
kubectl version --client
```

### Step 3: Install k9s (Optional But Recommended)

**k9s is a terminal UI for Kubernetes** - much easier than kubectl for daily use

```bash
# macOS
brew install k9s

# Linux
curl -sS https://webinstall.dev/k9s | bash

# Run it
k9s

# Keyboard shortcuts:
# 0: Show all namespaces
# :pods  → View pods
# :nodes → View nodes
# :deploy → View deployments
# / → Search
# d → Describe
# l → Logs
# s → Shell into pod
```

---

## Phase 5: Essential Add-ons

### Goal
Install critical cluster components.

### Step 1: MetalLB (LoadBalancer)

**Why?**: Bare-metal clusters don't have cloud LB (like AWS ELB). MetalLB provides LoadBalancer IPs.

```bash
# Install MetalLB
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.0/config/manifests/metallb-native.yaml

# Wait for pods to be ready
kubectl wait --namespace metallb-system \
  --for=condition=ready pod \
  --selector=app=metallb \
  --timeout=90s

# Create IP pool
cat <<EOF | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.100.200-192.168.100.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default
  namespace: metallb-system
spec:
  ipAddressPools:
  - default-pool
EOF
```

**Test MetalLB**:

```bash
# Create test service
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=LoadBalancer

# Check external IP (should be from MetalLB pool)
kubectl get svc nginx

# Expected:
# NAME    TYPE           CLUSTER-IP     EXTERNAL-IP       PORT(S)        AGE
# nginx   LoadBalancer   10.43.50.123   192.168.100.200   80:31234/TCP   10s

# Test from laptop
curl http://192.168.100.200  # Should show nginx welcome page

# Clean up test
kubectl delete svc nginx
kubectl delete deployment nginx
```

### Step 2: Nginx Ingress Controller

**Why?**: Ingress provides HTTP(S) routing (like reverse proxy for K8s)

```bash
# Install Nginx Ingress
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.0/deploy/static/provider/baremetal/deploy.yaml

# Check installation
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx

# Patch to use LoadBalancer (gets MetalLB IP)
kubectl patch svc ingress-nginx-controller -n ingress-nginx -p '{"spec": {"type": "LoadBalancer"}}'

# Verify external IP
kubectl get svc -n ingress-nginx ingress-nginx-controller

# Expected: EXTERNAL-IP should be from MetalLB pool
```

**Test Ingress**:

```bash
# Create test app
kubectl create deployment web --image=nginxdemos/hello
kubectl expose deployment web --port=80

# Create Ingress
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: web.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web
            port:
              number: 80
EOF

# Get Ingress IP
INGRESS_IP=$(kubectl get svc -n ingress-nginx ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# Test (add to /etc/hosts or use curl -H)
curl -H "Host: web.local" http://$INGRESS_IP

# Clean up
kubectl delete ingress web-ingress
kubectl delete svc web
kubectl delete deployment web
```

### Step 3: Cert-Manager (TLS Certificates)

**Why?**: Automates TLS cert issuance (Let's Encrypt)

```bash
# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.0/cert-manager.yaml

# Wait for ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/instance=cert-manager -n cert-manager --timeout=90s

# Create Let's Encrypt issuer (staging for testing)
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: your-email@example.com
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
    - http01:
        ingress:
          class: nginx
EOF

# Verify
kubectl get clusterissuer
```

---

## Verification and Testing

### Comprehensive Cluster Check

```bash
# 1. Node status
kubectl get nodes -o wide

# 2. System pods
kubectl get pods -A

# 3. Control plane health
kubectl get componentstatuses  # Deprecated but useful

# 4. Cluster info
kubectl cluster-info

# 5. Check etcd members
kubectl -n kube-system exec -it $(kubectl -n kube-system get pod -l component=etcd -o name | head -1) -- etcdctl member list

# 6. Check DNS
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup kubernetes.default

# 7. Resource usage
kubectl top nodes  # Requires metrics-server
```

---

## Troubleshooting

### Common Issues

**Issue 1: Nodes Not Ready**

```bash
# Check node status
kubectl describe node k3s-worker-1

# Check kubelet logs
ssh admin@192.168.100.111 "sudo journalctl -u k3s-agent -n 100"

# Common causes:
# - Networking issue (CNI not working)
# - Resource exhaustion
# - Kubelet crash loop
```

**Issue 2: Pods Stuck in Pending**

```bash
# Describe pod
kubectl describe pod <pod-name>

# Common causes:
# - No worker nodes available
# - Resource constraints (CPU/RAM)
# - PersistentVolume not bound
```

**Issue 3: MetalLB Not Assigning IPs**

```bash
# Check MetalLB pods
kubectl get pods -n metallb-system

# Check logs
kubectl logs -n metallb-system -l app=metallb

# Verify IP pool
kubectl get ipaddresspool -n metallb-system
```

---

## Best Practices

1. **Spread VMs across Proxmox hosts**: Ensures HA
2. **Use cloud-init**: Faster provisioning
3. **Pin K3s version**: Don't auto-upgrade (test first)
4. **Label nodes**: `kubectl label node k3s-worker-1 role=worker`
5. **Backup etcd**: Critical for cluster recovery
6. **Monitor resource usage**: Prevent node exhaustion
7. **Use namespaces**: Organize workloads
8. **Enable RBAC**: Security best practice

---

## Next Steps

1. Complete K3s cluster deployment
2. Install essential add-ons (MetalLB, Ingress, cert-manager)
3. Deploy first application
4. Set up monitoring (next guide)
5. Configure GitOps (next guide)
6. Implement backup strategy

---

*This guide provides a production-grade K3s cluster on Proxmox. The next guides will cover GitOps, observability, and multi-cluster setup.*

## VERSION HISTORY

- v1.0 (2026-02-09): Initial K3s deployment guide for Proxmox
