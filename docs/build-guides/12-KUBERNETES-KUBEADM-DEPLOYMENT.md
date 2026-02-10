# Kubernetes Deployment with kubeadm - Learning Guide

# In human review

## Note

All IP addresses and interface names are generic in this guide. You should replace them with your own IP addresses and interface names. See IP_ADDRESSING_PHILOSOPHY.md for more information.


## Overview

This guide walks through deploying a **production-grade Kubernetes cluster** using kubeadm on Proxmox. We're using kubeadm (not K3s) because this is a **learning environment** 


### Why kubeadm Specifically?

**kubeadm** is the official Kubernetes cluster bootstrapping tool:
- Created and maintained by Kubernetes project
- Used by CKA certification exam
- Foundation for automation tools (kubespray, kops)
- Teaches each component individually
- More manual = more learning

**After mastering kubeadm**: We'll migrate to kubespray (next guide) for automated deployments.

---

## Architecture Overview

### What You'll Build

```
Proxmox Cluster (3 hosts: pve1, pve2, pve3)
│
├── Control Plane Nodes (3x for HA)
│   ├── k8s-cp-1 (pve1): 4 vCPU, 8GB RAM
│   │   ├── etcd member
│   │   ├── kube-apiserver
│   │   ├── kube-controller-manager
│   │   ├── kube-scheduler
│   │   └── kubelet
│   ├── k8s-cp-2 (pve2): 4 vCPU, 8GB RAM
│   └── k8s-cp-3 (pve3): 4 vCPU, 8GB RAM
│
├── Worker Nodes (3-6x for workloads)
│   ├── k8s-worker-1 (pve1): 4 vCPU, 16GB RAM
│   ├── k8s-worker-2 (pve2): 4 vCPU, 16GB RAM
│   └── k8s-worker-3 (pve3): 4 vCPU, 16GB RAM
│
└── Load Balancer (1x for API HA)
    └── k8s-lb-1 (pve1): 1 vCPU, 2GB RAM
        └── HAProxy (192.168.100.100:6443 → 3 control planes)

Total Resources: ~25 vCPU, ~58GB RAM for HA cluster
```

### Component Architecture

**Control Plane Components** (run on k8s-cp-* nodes):
- **etcd**: Distributed key-value store (cluster state)
- **kube-apiserver**: REST API for all K8s operations
- **kube-controller-manager**: Reconciliation loops (deployments, services, etc.)
- **kube-scheduler**: Pod placement decisions

**Node Components** (run on all nodes):
- **kubelet**: Node agent (manages containers)
- **kube-proxy**: Network proxy (service routing)
- **Container runtime**: containerd (replaces Docker)

**Add-ons** (deployed after cluster creation):
- **CNI**: Calico (pod networking)
- **CoreDNS**: Cluster DNS
- **MetalLB**: LoadBalancer services
- **Nginx Ingress**: HTTP routing

---

## Phase 1: Infrastructure Preparation

### Goal
Prepare Proxmox VMs for Kubernetes deployment.

### Step 1: Network Planning

**IP Allocation**:

| VM Name | IP Address | Proxmox Host | Role | vCPU | RAM |
|---------|------------|--------------|------|------|-----|
| k8s-lb-1 | 192.168.100.100 | pve1 | Load balancer | 1 | 2GB |
| k8s-cp-1 | 192.168.100.101 | pve1 | Control plane | 4 | 8GB |
| k8s-cp-2 | 192.168.100.102 | pve2 | Control plane | 4 | 8GB |
| k8s-cp-3 | 192.168.100.103 | pve3 | Control plane | 4 | 8GB |
| k8s-worker-1 | 192.168.100.111 | pve1 | Worker | 4 | 16GB |
| k8s-worker-2 | 192.168.100.112 | pve2 | Worker | 4 | 16GB |
| k8s-worker-3 | 192.168.100.113 | pve3 | Worker | 4 | 16GB |

**Kubernetes Networks**:
- **Pod CIDR**: `10.244.0.0/16` (Calico default)
- **Service CIDR**: `10.96.0.0/12` (K8s default)
- **API Server VIP**: `192.168.100.100:6443` (HAProxy)
- **MetalLB Pool**: `192.168.100.200-250`

**Why spread across Proxmox hosts?**: HA - if one host fails, cluster continues

### Step 2: Create VM Template

**Download Ubuntu 22.04 cloud image**:

```bash
# On Proxmox host pve1
cd /var/lib/vz/template/iso/
wget https://cloud-images.ubuntu.com/releases/22.04/release/ubuntu-22.04-server-cloudimg-amd64.img
```

**Create template**:

```bash
# Create VM (ID 9000 for template)
qm create 9000 --name k8s-template --memory 8192 --cores 4 --net0 virtio,bridge=vmbr0

# Import cloud image
qm importdisk 9000 ubuntu-22.04-server-cloudimg-amd64.img local-lvm

# Attach disk
qm set 9000 --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-9000-disk-0

# Cloud-init drive
qm set 9000 --ide2 local-lvm:cloudinit

# Boot disk
qm set 9000 --boot c --bootdisk scsi0

# Serial console
qm set 9000 --serial0 socket --vga serial0

# QEMU agent
qm set 9000 --agent enabled=1

# Resize disk to 50GB (K8s needs more than 20GB)
qm resize 9000 scsi0 50G

# Convert to template
qm template 9000
```

**Why 50GB?**: K8s images, logs, and pod storage need space

### Step 3: Deploy VMs from Template

**Clone and configure** (example for k8s-cp-1):

```bash
# On pve1 - create control plane 1
qm clone 9000 101 --name k8s-cp-1 --full

# Configure cloud-init
qm set 101 --ipconfig0 ip=192.168.100.101/24,gw=192.168.100.1
qm set 101 --nameserver 8.8.8.8
qm set 101 --sshkeys ~/.ssh/id_rsa.pub
qm set 101 --ciuser ubuntu

# Set resources
qm set 101 --memory 8192 --cores 4

# Start
qm start 101
```

**Repeat for all VMs** (adjust IPs, memory/CPU per table above)

**Or use this script**:

```bash
#!/bin/bash
# deploy-k8s-vms.sh

VMS=(
  "100:k8s-lb-1:192.168.100.100:1:2048:pve1"
  "101:k8s-cp-1:192.168.100.101:4:8192:pve1"
  "102:k8s-cp-2:192.168.100.102:4:8192:pve2"
  "103:k8s-cp-3:192.168.100.103:4:8192:pve3"
  "111:k8s-worker-1:192.168.100.111:4:16384:pve1"
  "112:k8s-worker-2:192.168.100.112:4:16384:pve2"
  "113:k8s-worker-3:192.168.100.113:4:16384:pve3"
)

for vm in "${VMS[@]}"; do
  IFS=: read -r vmid name ip cores memory host <<< "$vm"

  # Clone on appropriate host
  ssh root@$host "qm clone 9000 $vmid --name $name --full"
  ssh root@$host "qm set $vmid --ipconfig0 ip=$ip/24,gw=192.168.100.1"
  ssh root@$host "qm set $vmid --nameserver 8.8.8.8"
  ssh root@$host "qm set $vmid --sshkeys ~/.ssh/id_rsa.pub"
  ssh root@$host "qm set $vmid --ciuser ubuntu"
  ssh root@$host "qm set $vmid --cores $cores --memory $memory"
  ssh root@$host "qm start $vmid"

  echo "Created $name on $host"
done
```

---

## Phase 2: Load Balancer Setup

### Goal
Deploy HAProxy to load balance API server requests across 3 control planes.

### Understanding HA Architecture

**Why load balancer?**:
- Control plane HA needs single endpoint (can't point to 3 IPs)
- HAProxy distributes API requests across healthy control planes
- If one control plane fails, HAProxy routes to others

**Alternative**: kube-vip (runs inside K8s), but HAProxy is easier to understand

### Step 1: Install HAProxy

**SSH to k8s-lb-1**:

```bash
ssh ubuntu@192.168.100.100

# Install HAProxy
sudo apt update
sudo apt install -y haproxy
```

### Step 2: Configure HAProxy

**Edit `/etc/haproxy/haproxy.cfg`**:

```bash
sudo tee /etc/haproxy/haproxy.cfg > /dev/null <<EOF
global
    log /dev/log local0
    log /dev/log local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin
    stats timeout 30s
    user haproxy
    group haproxy
    daemon

defaults
    log     global
    mode    tcp
    option  tcplog
    option  dontlognull
    timeout connect 5000
    timeout client  50000
    timeout server  50000

# Kubernetes API Server frontend
frontend k8s-api
    bind *:6443
    mode tcp
    option tcplog
    default_backend k8s-api-backend

# Kubernetes API Server backend
backend k8s-api-backend
    mode tcp
    option tcp-check
    balance roundrobin
    server k8s-cp-1 192.168.100.101:6443 check fall 3 rise 2
    server k8s-cp-2 192.168.100.102:6443 check fall 3 rise 2
    server k8s-cp-3 192.168.100.103:6443 check fall 3 rise 2

# Stats page (optional)
listen stats
    bind *:8080
    mode http
    stats enable
    stats uri /
    stats refresh 10s
    stats auth admin:changeme
EOF
```

**Key settings**:
- `bind *:6443`: Listen on standard K8s API port
- `balance roundrobin`: Distribute requests evenly
- `check fall 3 rise 2`: Health check (mark down after 3 fails, up after 2 success)

**Restart HAProxy**:

```bash
sudo systemctl restart haproxy
sudo systemctl enable haproxy
sudo systemctl status haproxy
```

**Verify**:

```bash
# Should show 3 backend servers (down initially, normal)
echo "show stat" | sudo socat stdio /run/haproxy/admin.sock | grep k8s-cp

# Stats page (optional)
curl http://192.168.100.100:8080
```

---

## Phase 3: System Preparation (All Nodes)

### Goal
Prepare all K8s nodes with prerequisites.

### Step 1: Update Systems

**Run on ALL nodes** (cp-1, cp-2, cp-3, worker-1, worker-2, worker-3):

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y apt-transport-https ca-certificates curl gpg
```

### Step 2: Disable Swap

**Kubernetes requires swap disabled**:

```bash
# Disable swap immediately
sudo swapoff -a

# Disable swap permanently
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Verify (should show no swap)
free -h
```

**Why?**: K8s memory management assumes no swap (performance reasons)

### Step 3: Load Kernel Modules

```bash
# Load modules now
sudo modprobe overlay
sudo modprobe br_netfilter

# Load modules on boot
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

**What these do**:
- `overlay`: OverlayFS for container storage
- `br_netfilter`: iptables can see bridged traffic (required for pod networking)

### Step 4: Configure Sysctl

```bash
# Configure networking
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply settings
sudo sysctl --system

# Verify
sysctl net.ipv4.ip_forward  # Should be 1
```

### Step 5: Install containerd

**Kubernetes uses containerd as container runtime** (Docker is deprecated):

```bash
# Install containerd
sudo apt install -y containerd

# Generate default config
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Enable SystemdCgroup (CRITICAL for K8s)
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# Restart containerd
sudo systemctl restart containerd
sudo systemctl enable containerd
sudo systemctl status containerd
```

**Why SystemdCgroup = true?**: K8s uses systemd, containerd must match

### Step 6: Install kubeadm, kubelet, kubectl

```bash
# Add Kubernetes apt repository
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Install K8s components
sudo apt update
sudo apt install -y kubelet kubeadm kubectl

# Hold packages (prevent auto-upgrade)
sudo apt-mark hold kubelet kubeadm kubectl

# Verify
kubeadm version
kubelet --version
kubectl version --client
```

**Why hold packages?**: K8s upgrades should be manual and tested

### Step 7: Configure kubelet

```bash
# Set node IP (CRITICAL for multi-NIC nodes)
echo "KUBELET_EXTRA_ARGS=--node-ip=$(hostname -I | awk '{print $1}')" | sudo tee /etc/default/kubelet

# Restart kubelet (will crash until joined to cluster - normal)
sudo systemctl restart kubelet
```

**Repeat all steps on ALL 6 nodes** (3 control planes + 3 workers)

---

## Phase 4: Initialize First Control Plane

### Goal
Bootstrap the first control plane node with kubeadm.

### Step 1: Create kubeadm Config

**On k8s-cp-1**, create `/tmp/kubeadm-config.yaml`:

```yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: v1.29.0
controlPlaneEndpoint: "192.168.100.100:6443"  # HAProxy VIP
networking:
  podSubnet: "10.244.0.0/16"  # Calico default
  serviceSubnet: "10.96.0.0/12"
apiServer:
  certSANs:
  - "192.168.100.100"  # HAProxy IP
  - "192.168.100.101"  # k8s-cp-1
  - "192.168.100.102"  # k8s-cp-2
  - "192.168.100.103"  # k8s-cp-3
  extraArgs:
    authorization-mode: "Node,RBAC"
etcd:
  local:
    dataDir: /var/lib/etcd
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 192.168.100.101  # This node's IP
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
  kubeletExtraArgs:
    node-ip: 192.168.100.101
```

**Key settings explained**:
- `controlPlaneEndpoint`: HAProxy IP (all nodes connect here)
- `certSANs`: API server certificate includes all control plane IPs
- `podSubnet`: Must match CNI plugin (Calico uses 10.244.0.0/16)
- `advertiseAddress`: This specific control plane's IP

### Step 2: Initialize Cluster

**On k8s-cp-1**:

```bash
sudo kubeadm init --config /tmp/kubeadm-config.yaml --upload-certs
```

**Watch output carefully!** You'll see:

```
[init] Using Kubernetes version: v1.29.0
[preflight] Running pre-flight checks
[certs] Generating certificates
[kubeconfig] Writing kubeconfig files
[control-plane] Creating static Pod manifests
[etcd] Creating static Pod manifest for local etcd
[wait-control-plane] Waiting for control plane to be ready
[apiclient] All control plane components are healthy
[upload-config] Storing configuration
[upload-certs] Storing certificates
[mark-control-plane] Marking node as control plane
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf"
[addons] Installing CoreDNS
[addons] Installing kube-proxy

Your Kubernetes control plane has initialized successfully!

To start using your cluster, run:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You can now join control plane nodes:

  kubeadm join 192.168.100.100:6443 --token abc123.xyz789 \
    --discovery-token-ca-cert-hash sha256:0123456789abcdef... \
    --control-plane --certificate-key fedcba9876543210...

You can join worker nodes:

  kubeadm join 192.168.100.100:6443 --token abc123.xyz789 \
    --discovery-token-ca-cert-hash sha256:0123456789abcdef...
```

**SAVE THESE COMMANDS!** You'll need them to join other nodes.

### Step 3: Configure kubectl

**On k8s-cp-1**:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Verify
kubectl get nodes
# NAME       STATUS     ROLES           AGE   VERSION
# k8s-cp-1   NotReady   control-plane   1m    v1.29.0
```

**Why NotReady?**: No CNI plugin yet (next step)

---

## Phase 5: Install CNI (Calico)

### Goal
Deploy Calico for pod networking.

### Understanding CNI

**CNI (Container Network Interface)** provides:
- IP addresses for pods
- Pod-to-pod communication (across nodes)
- Network policies (firewall rules)

**Why Calico?**:
- Most popular CNI (good for resume)
- Excellent network policy support
- BGP integration (matches your ISP lab!)
- Good documentation

**Alternatives**: Cilium (eBPF, more advanced), Flannel (simpler), Weave

### Step 1: Install Calico

**On k8s-cp-1**:

```bash
# Download Calico manifest
curl https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml -O

# Install Calico
kubectl apply -f calico.yaml

# Watch pods come up
kubectl get pods -n kube-system -w
```

**Wait 2-3 minutes**, then:

```bash
kubectl get nodes
# NAME       STATUS   ROLES           AGE   VERSION
# k8s-cp-1   Ready    control-plane   5m    v1.29.0
```

**Status changed to Ready!** ✓

### Step 2: Verify Calico

```bash
# Check Calico pods
kubectl get pods -n kube-system -l k8s-app=calico-node

# Should see 1 calico-node (will scale as you add nodes)

# Check calico-kube-controllers
kubectl get pods -n kube-system -l k8s-app=calico-kube-controllers
```

---

## Phase 6: Join Additional Control Planes

### Goal
Add k8s-cp-2 and k8s-cp-3 for HA.

### Step 1: Join k8s-cp-2

**On k8s-cp-2**, run the join command from kubeadm init output:

```bash
sudo kubeadm join 192.168.100.100:6443 --token abc123.xyz789 \
  --discovery-token-ca-cert-hash sha256:0123456789abcdef... \
  --control-plane --certificate-key fedcba9876543210...
```

**Watch output**:

```
[preflight] Running pre-flight checks
[preflight] Reading configuration from cluster
[download-certs] Downloading certificates
[certs] Generating certificates
[kubeconfig] Writing kubeconfig files
[control-plane-join] Starting control plane components
[etcd] Adding new etcd member
[mark-control-plane] Marking node as control plane
[kubelet-start] Starting kubelet

This node has joined the cluster and a new control plane instance was created.
```

**Configure kubectl on k8s-cp-2**:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

**Verify from k8s-cp-1**:

```bash
kubectl get nodes
# NAME       STATUS   ROLES           AGE   VERSION
# k8s-cp-1   Ready    control-plane   10m   v1.29.0
# k8s-cp-2   Ready    control-plane   2m    v1.29.0
```

### Step 2: Join k8s-cp-3

**Same process on k8s-cp-3**

**After joining, verify all 3**:

```bash
kubectl get nodes
# NAME       STATUS   ROLES           AGE   VERSION
# k8s-cp-1   Ready    control-plane   15m   v1.29.0
# k8s-cp-2   Ready    control-plane   7m    v1.29.0
# k8s-cp-3   Ready    control-plane   2m    v1.29.0
```

### Step 3: Verify etcd Cluster

```bash
# List etcd members
kubectl exec -n kube-system etcd-k8s-cp-1 -- etcdctl \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member list

# Should show 3 members (k8s-cp-1, k8s-cp-2, k8s-cp-3)
```

**Understanding etcd quorum**:
- 3 members = can tolerate 1 failure
- Need majority (2/3) for writes
- If 2 fail, cluster is read-only

---

## Phase 7: Join Worker Nodes

### Goal
Add worker nodes for running workloads.

### Step 1: Join Workers

**On k8s-worker-1, k8s-worker-2, k8s-worker-3**, run worker join command:

```bash
sudo kubeadm join 192.168.100.100:6443 --token abc123.xyz789 \
  --discovery-token-ca-cert-hash sha256:0123456789abcdef...
```

**Note**: No `--control-plane` flag (these are workers)

**Verify all nodes**:

```bash
kubectl get nodes
# NAME           STATUS   ROLES           AGE   VERSION
# k8s-cp-1       Ready    control-plane   20m   v1.29.0
# k8s-cp-2       Ready    control-plane   12m   v1.29.0
# k8s-cp-3       Ready    control-plane   7m    v1.29.0
# k8s-worker-1   Ready    <none>          2m    v1.29.0
# k8s-worker-2   Ready    <none>          2m    v1.29.0
# k8s-worker-3   Ready    <none>          2m    v1.29.0
```

**Label workers** (optional but nice):

```bash
kubectl label node k8s-worker-1 node-role.kubernetes.io/worker=worker
kubectl label node k8s-worker-2 node-role.kubernetes.io/worker=worker
kubectl label node k8s-worker-3 node-role.kubernetes.io/worker=worker

# Now shows ROLES properly
kubectl get nodes
# NAME           STATUS   ROLES           AGE   VERSION
# k8s-cp-1       Ready    control-plane   20m   v1.29.0
# k8s-cp-2       Ready    control-plane   12m   v1.29.0
# k8s-cp-3       Ready    control-plane   7m    v1.29.0
# k8s-worker-1   Ready    worker          2m    v1.29.0
# k8s-worker-2   Ready    worker          2m    v1.29.0
# k8s-worker-3   Ready    worker          2m    v1.29.0
```

---

## Phase 8: Essential Add-ons

### Goal
Install MetalLB, Ingress, and cert-manager.

### Step 1: Install MetalLB

**Why MetalLB?**: Provides LoadBalancer IPs in bare-metal environments

```bash
# Install MetalLB
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.0/config/manifests/metallb-native.yaml

# Wait for ready
kubectl wait --namespace metallb-system \
  --for=condition=ready pod \
  --selector=app=metallb \
  --timeout=90s

# Configure IP pool
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

**Test**:

```bash
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=LoadBalancer

# Check external IP
kubectl get svc nginx
# NAME    TYPE           CLUSTER-IP      EXTERNAL-IP       PORT(S)
# nginx   LoadBalancer   10.96.50.100    192.168.100.200   80:31234/TCP

curl http://192.168.100.200  # Should show nginx welcome

# Clean up
kubectl delete svc nginx
kubectl delete deployment nginx
```

### Step 2: Install Nginx Ingress

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.0/deploy/static/provider/baremetal/deploy.yaml

# Patch to use LoadBalancer (gets MetalLB IP)
kubectl patch svc ingress-nginx-controller -n ingress-nginx -p '{"spec": {"type": "LoadBalancer"}}'

# Verify
kubectl get svc -n ingress-nginx ingress-nginx-controller
```

### Step 3: Install cert-manager

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.0/cert-manager.yaml

# Wait for ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/instance=cert-manager -n cert-manager --timeout=90s
```

---

## Verification and Component Understanding

### Check Each Component

**1. API Server**:

```bash
# All 3 API servers should be healthy
kubectl get --raw='/healthz?verbose'

# Check which control plane you're hitting
kubectl cluster-info
```

**2. etcd**:

```bash
# etcd health
kubectl exec -n kube-system etcd-k8s-cp-1 -- etcdctl \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health --cluster

# Should show all 3 members healthy
```

**3. Scheduler & Controller Manager**:

```bash
kubectl get pods -n kube-system | grep -E 'kube-scheduler|kube-controller'

# Should see 3 of each (1 per control plane)
# Only 1 is active (leader election), others are standby
```

**4. Node Components**:

```bash
# Check kubelet status on any node
ssh ubuntu@192.168.100.111 "sudo systemctl status kubelet"

# Check kube-proxy
kubectl get pods -n kube-system -l k8s-app=kube-proxy
# Should see 6 (one per node)
```

**5. CNI**:

```bash
# Calico nodes
kubectl get pods -n kube-system -l k8s-app=calico-node
# Should see 6 (one per node)

# Check pod networking
kubectl run test --image=nginx
kubectl get pod test -o wide  # Should have IP from 10.244.0.0/16
kubectl delete pod test
```

---

## What You Learned vs K3s

**With kubeadm you learned**:

✅ **How Kubernetes actually works**:
- Separate components (API server, scheduler, controller-manager, etcd)
- Control plane vs data plane
- How components communicate

✅ **etcd cluster management**:
- Quorum concept (2/3 required)
- Leader election
- Member health checking

✅ **Certificate management**:
- PKI infrastructure (`/etc/kubernetes/pki/`)
- Certificate SANs for HA
- Certificate renewal

✅ **HA architecture**:
- Load balancer for API server
- Multiple control planes
- Failure scenarios

✅ **Troubleshooting**:
- Component-level logs (`kubectl logs -n kube-system`)
- Static pod manifests (`/etc/kubernetes/manifests/`)
- kubelet logs (`journalctl -u kubelet`)

**K3s would have hidden**:
- ❌ etcd (embedded SQLite or etcd you never see)
- ❌ Component separation (single binary)
- ❌ Certificate management (automatic)
- ❌ Real HA (simplified)

---

## Next Steps

1. ✅ Cluster deployed with kubeadm
2. ✅ Understand each component
3. → Deploy observability stack (guide 13)
4. → Migrate to kubespray automation (guide 14)
5. → Deploy multi-cluster (guide 15)

---

*This guide taught you production Kubernetes deployment. Next: Automate this with kubespray for repeatable deployments.*

## VERSION HISTORY

- v1.0 (2026-02-09): Full Kubernetes deployment with kubeadm (replaces K3s guide)
