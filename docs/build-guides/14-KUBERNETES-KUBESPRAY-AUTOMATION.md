# Kubernetes Automation with Kubespray

## Overview

This guide walks through deploying the **same Kubernetes cluster** from guide 12, but using **kubespray** - an Ansible-based automation tool. After learning kubeadm (manual deployment), kubespray teaches you production-grade automation.

### Why Kubespray After kubeadm?

**kubeadm (guide 12)** taught you:
- ✅ Component architecture
- ✅ HA setup manually
- ✅ How K8s actually works
- ✅ Troubleshooting skills

**Kubespray teaches**:
- ✅ Infrastructure as Code (IaC)
- ✅ Repeatable deployments
- ✅ Configuration management
- ✅ Production automation
- ✅ Easy cluster updates/scaling

**Career relevance**:
- Interview: "How do you deploy K8s?" → "kubespray/Ansible" (not "I manually run kubeadm")
- Resume: "Automated K8s deployment with kubespray" ✓

### What is Kubespray?

**Kubespray** is a production-grade Kubernetes deployment tool:
- Uses Ansible under the hood
- Maintained by Kubernetes SIG
- Supports multiple CNI plugins, OS distributions
- Used by enterprises for on-prem K8s
- 15k+ GitHub stars

**How it works**:
1. You define inventory (which servers, what roles)
2. You set variables (K8s version, CNI, etc.)
3. You run playbook (`ansible-playbook cluster.yml`)
4. Kubespray deploys everything (kubeadm, CNI, add-ons)

**Think of it as**: Scripted version of guide 12

---

## Architecture - Same as kubeadm Guide

```
Proxmox VMs (same as guide 12):
├── k8s-cp-1,2,3 (control planes)
├── k8s-worker-1,2,3 (workers)
└── k8s-lb-1 (HAProxy)

Deployment differences:
kubeadm:   Manual commands on each node
kubespray: One command from ansible control machine
```

---

## Phase 1: Prerequisites

### Goal
Prepare control machine (your laptop) and target nodes.

### Step 1: Install Ansible (Control Machine)

**On your laptop/workstation**:

```bash
# macOS
brew install ansible

# Linux (Ubuntu)
sudo apt update
sudo apt install -y ansible python3-pip

# Verify
ansible --version  # Should be 2.15+
```

### Step 2: Install Python Dependencies

```bash
# Install kubespray dependencies
pip3 install jinja2 netaddr pbr jmespath ruamel.yaml cryptography
```

### Step 3: Prepare Target Nodes

**Ensure all VMs are deployed** (from guide 12 Phase 1):
- k8s-lb-1 (if using HAProxy)
- k8s-cp-1, k8s-cp-2, k8s-cp-3
- k8s-worker-1, k8s-worker-2, k8s-worker-3

**SSH access**:

```bash
# Test SSH to all nodes
for i in {100..103} {111..113}; do
  echo "Testing 192.168.100.$i..."
  ssh -o StrictHostKeyChecking=no ubuntu@192.168.100.$i "hostname"
done
```

**If SSH requires password**: Set up SSH keys

```bash
# Generate key (if you don't have one)
ssh-keygen -t rsa -b 4096

# Copy to all nodes
for i in {101..103} {111..113}; do
  ssh-copy-id ubuntu@192.168.100.$i
done
```

### Step 4: Clean VMs (If Reusing from kubeadm)

**If you already ran kubeadm**, reset nodes:

```bash
# On each node
sudo kubeadm reset -f
sudo rm -rf /etc/kubernetes /var/lib/kubelet /var/lib/etcd
sudo apt purge -y kubeadm kubelet kubectl
sudo apt autoremove -y

# Or script it
for i in {101..103} {111..113}; do
  ssh ubuntu@192.168.100.$i "sudo kubeadm reset -f && sudo rm -rf /etc/kubernetes /var/lib/kubelet /var/lib/etcd"
done
```

---

## Phase 2: Download and Configure Kubespray

### Goal
Get kubespray and configure for your cluster.

### Step 1: Clone Kubespray

```bash
# Clone specific version (don't use master - too bleeding edge)
cd ~/
git clone https://github.com/kubernetes-sigs/kubespray.git
cd kubespray
git checkout release-2.24  # Stable release

# Install Python requirements
pip3 install -r requirements.txt
```

### Step 2: Create Inventory

**Copy sample inventory**:

```bash
cp -r inventory/sample inventory/project-atlas
cd inventory/project-atlas
```

**Create `inventory.ini`**:

```ini
# inventory/project-atlas/inventory.ini

# Control plane nodes
[kube_control_plane]
k8s-cp-1 ansible_host=192.168.100.101 ip=192.168.100.101
k8s-cp-2 ansible_host=192.168.100.102 ip=192.168.100.102
k8s-cp-3 ansible_host=192.168.100.103 ip=192.168.100.103

# etcd nodes (usually same as control plane)
[etcd]
k8s-cp-1
k8s-cp-2
k8s-cp-3

# Worker nodes
[kube_node]
k8s-worker-1 ansible_host=192.168.100.111 ip=192.168.100.111
k8s-worker-2 ansible_host=192.168.100.112 ip=192.168.100.112
k8s-worker-3 ansible_host=192.168.100.113 ip=192.168.100.113

# All cluster members
[k8s_cluster:children]
kube_control_plane
kube_node
etcd

# Ansible connection settings
[all:vars]
ansible_user=ubuntu
ansible_become=yes
ansible_become_method=sudo
```

**Inventory structure explained**:
- `[kube_control_plane]`: Nodes running K8s control plane
- `[etcd]`: Nodes running etcd (usually same as control plane)
- `[kube_node]`: Worker nodes
- `[k8s_cluster:children]`: Combines all groups

### Step 3: Configure Cluster Variables

**Edit `inventory/project-atlas/group_vars/k8s_cluster/k8s-cluster.yml`**:

```yaml
# Kubernetes version
kube_version: v1.29.0

# Container runtime (containerd recommended)
container_manager: containerd

# Network plugin (Calico recommended)
kube_network_plugin: calico

# Service and pod networks
kube_service_addresses: 10.96.0.0/12
kube_pods_subnet: 10.244.0.0/16

# API server configuration
# Use HAProxy IP if you have load balancer
# Or use first control plane IP if no LB
kube_apiserver_ip: "{{ kube_service_addresses|ipaddr('net')|ipaddr(1)|ipaddr('address') }}"
# For HA with external LB, override:
# supplementary_addresses_in_ssl_keys: [192.168.100.100]

# Enable metrics server
metrics_server_enabled: true

# Enable local volume provisioner
local_volume_provisioner_enabled: true

# Helm deployment
helm_enabled: true
```

**Edit `inventory/project-atlas/group_vars/k8s_cluster/addons.yml`**:

```yaml
# MetalLB
metallb_enabled: true
metallb_speaker_enabled: true
metallb_version: v0.14.0
metallb_protocol: layer2
metallb_ip_range:
  - "192.168.100.200-192.168.100.250"

# Ingress nginx
ingress_nginx_enabled: true
ingress_nginx_host_network: false  # Use LoadBalancer type

# Cert-manager
cert_manager_enabled: true

# Dashboard (optional)
dashboard_enabled: true
```

### Step 4: Configure HA (If Using HAProxy)

**For external load balancer**, edit `group_vars/all/all.yml`:

```yaml
# Use external LB
loadbalancer_apiserver_localhost: false
loadbalancer_apiserver:
  address: 192.168.100.100  # HAProxy IP
  port: 6443
```

**Or use kube-vip** (no HAProxy needed):

```yaml
# Enable kube-vip instead of HAProxy
kube_vip_enabled: true
kube_vip_controlplane_enabled: true
kube_vip_address: 192.168.100.100
kube_vip_interface: eth0
```

---

## Phase 3: Deploy Cluster

### Goal
Run kubespray playbook to deploy K8s.

### Step 1: Verify Connectivity

```bash
# From kubespray directory
ansible all -i inventory/project-atlas/inventory.ini -m ping
```

**Expected output**:
```
k8s-cp-1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
k8s-cp-2 | SUCCESS => ...
```

**If ping fails**:
- Check SSH connectivity
- Verify `ansible_user` in inventory
- Check sudo permissions

### Step 2: Run Preflight Checks

```bash
# Dry run to check for issues
ansible-playbook -i inventory/project-atlas/inventory.ini \
  --become --become-user=root \
  cluster.yml --check
```

**Look for**:
- SSH errors
- Privilege escalation issues
- Missing Python modules

### Step 3: Deploy Cluster

**Full deployment**:

```bash
# Deploy cluster (takes 10-20 minutes)
ansible-playbook -i inventory/project-atlas/inventory.ini \
  --become --become-user=root \
  cluster.yml
```

**Watch output**:

```
PLAY [Prepare nodes] *******************************************************

TASK [Gathering Facts] *****************************************************
ok: [k8s-cp-1]
ok: [k8s-cp-2]
...

PLAY [Install container runtime] *******************************************

TASK [containerd : Install containerd] *************************************
changed: [k8s-cp-1]
...

PLAY [Bootstrap etcd cluster] **********************************************

TASK [etcd : Install etcd] *************************************************
changed: [k8s-cp-1]
...

PLAY [Install Kubernetes control plane] ************************************

TASK [kubeadm : Initialize first control plane] ****************************
changed: [k8s-cp-1]

TASK [kubeadm : Join additional control planes] ****************************
changed: [k8s-cp-2]
changed: [k8s-cp-3]

PLAY [Install CNI plugin] **************************************************

TASK [calico : Deploy Calico] **********************************************
changed: [k8s-cp-1]

PLAY [Join worker nodes] ***************************************************

TASK [kubeadm : Join workers] **********************************************
changed: [k8s-worker-1]
...

PLAY RECAP *****************************************************************
k8s-cp-1       : ok=567  changed=123  unreachable=0  failed=0
k8s-cp-2       : ok=489  changed=101  unreachable=0  failed=0
k8s-cp-3       : ok=489  changed=101  unreachable=0  failed=0
k8s-worker-1   : ok=345  changed=87   unreachable=0  failed=0
k8s-worker-2   : ok=345  changed=87   unreachable=0  failed=0
k8s-worker-3   : ok=345  changed=87   unreachable=0  failed=0
```

**Duration**: 10-20 minutes (first run), 5-10 minutes (subsequent runs)

### Step 4: Retrieve kubeconfig

**Copy kubeconfig from first control plane**:

```bash
# Kubespray stores it at /root/.kube/config
scp ubuntu@192.168.100.101:/etc/kubernetes/admin.conf ~/.kube/config-project-atlas

# Or use kubespray's fetch task
ansible-playbook -i inventory/project-atlas/inventory.ini \
  --become --become-user=root \
  playbooks/fetch-kubeconfig.yml

# Set KUBECONFIG
export KUBECONFIG=~/.kube/config-project-atlas

# Or merge with existing config
KUBECONFIG=~/.kube/config:~/.kube/config-project-atlas kubectl config view --flatten > ~/.kube/config-merged
mv ~/.kube/config-merged ~/.kube/config
```

### Step 5: Verify Cluster

```bash
kubectl get nodes

# Expected:
# NAME           STATUS   ROLES           AGE   VERSION
# k8s-cp-1       Ready    control-plane   5m    v1.29.0
# k8s-cp-2       Ready    control-plane   5m    v1.29.0
# k8s-cp-3       Ready    control-plane   5m    v1.29.0
# k8s-worker-1   Ready    worker          3m    v1.29.0
# k8s-worker-2   Ready    worker          3m    v1.29.0
# k8s-worker-3   Ready    worker          3m    v1.29.0

# Check add-ons
kubectl get pods -A
```

---

## Phase 4: Understanding What Kubespray Did

### Goal
Understand kubespray automation vs manual kubeadm.

### What Kubespray Automated

**1. System Preparation** (from guide 12 Phase 3):
```yaml
# Kubespray did this automatically:
- Disabled swap
- Loaded kernel modules
- Configured sysctl
- Installed containerd
- Installed kubeadm, kubelet, kubectl
```

**2. Control Plane Bootstrap** (from guide 12 Phase 4-6):
```yaml
- Generated kubeadm config
- Initialized first control plane
- Joined additional control planes
- Set up etcd cluster
```

**3. CNI Installation** (from guide 12 Phase 5):
```yaml
- Downloaded Calico manifest
- Applied to cluster
- Configured IP pools
```

**4. Worker Join** (from guide 12 Phase 7):
```yaml
- Joined all workers
- Labeled nodes appropriately
```

**5. Add-ons** (from guide 12 Phase 8):
```yaml
- MetalLB
- Nginx Ingress
- cert-manager
- Metrics server
```

### Files Kubespray Generated

**On control plane nodes**:

```bash
# Check what kubespray created
ssh ubuntu@192.168.100.101 ls -la /etc/kubernetes/

# You'll see:
# - admin.conf
# - controller-manager.conf
# - scheduler.conf
# - kubelet.conf
# - manifests/ (static pods)
# - pki/ (certificates)
```

**Same files as kubeadm** - kubespray uses kubeadm under the hood!

### View Kubespray Logs

**On control machine**:

```bash
# Kubespray creates detailed logs
ls -la ~/.ansible/

# Check specific task output
ansible-playbook -i inventory/project-atlas/inventory.ini cluster.yml -vvv
```

---

## Phase 5: Cluster Operations

### Goal
Learn day-2 operations with kubespray.

### Add Worker Node

**1. Update inventory**:

```ini
# inventory/project-atlas/inventory.ini

[kube_node]
k8s-worker-1 ansible_host=192.168.100.111 ip=192.168.100.111
k8s-worker-2 ansible_host=192.168.100.112 ip=192.168.100.112
k8s-worker-3 ansible_host=192.168.100.113 ip=192.168.100.113
k8s-worker-4 ansible_host=192.168.100.114 ip=192.168.100.114  # NEW
```

**2. Run scale playbook**:

```bash
ansible-playbook -i inventory/project-atlas/inventory.ini \
  --become --become-user=root \
  scale.yml --limit=k8s-worker-4
```

**3. Verify**:

```bash
kubectl get nodes
# Should show k8s-worker-4
```

### Remove Worker Node

**1. Drain node**:

```bash
kubectl drain k8s-worker-4 --ignore-daemonsets --delete-emptydir-data
```

**2. Remove from inventory**:

```bash
# Delete k8s-worker-4 line from inventory.ini
```

**3. Run remove-node playbook**:

```bash
ansible-playbook -i inventory/project-atlas/inventory.ini \
  --become --become-user=root \
  remove-node.yml --extra-vars "node=k8s-worker-4"
```

### Upgrade Kubernetes

**1. Update version in inventory**:

```yaml
# group_vars/k8s_cluster/k8s-cluster.yml
kube_version: v1.30.0  # Upgrade from v1.29.0
```

**2. Run upgrade playbook**:

```bash
ansible-playbook -i inventory/project-atlas/inventory.ini \
  --become --become-user=root \
  upgrade-cluster.yml
```

**Kubespray upgrades**:
- One control plane at a time
- Drains and upgrades workers
- Minimizes downtime

### Reset Cluster

**Destroy everything and start over**:

```bash
ansible-playbook -i inventory/project-atlas/inventory.ini \
  --become --become-user=root \
  reset.yml

# Confirm when prompted (THIS IS DESTRUCTIVE)
```

**Then redeploy**:

```bash
ansible-playbook -i inventory/project-atlas/inventory.ini \
  --become --become-user=root \
  cluster.yml
```

---

## Phase 6: Advanced Configuration

### Goal
Customize kubespray for specific needs.

### Enable Additional Add-ons

**Edit `group_vars/k8s_cluster/addons.yml`**:

```yaml
# Metrics server (for kubectl top)
metrics_server_enabled: true

# Kubernetes dashboard
dashboard_enabled: true

# Persistent volume provisioner
local_volume_provisioner_enabled: true
local_volume_provisioner_namespace: kube-system
local_volume_provisioner_storage_classes:
  local-storage:
    host_dir: /mnt/disks
    mount_dir: /mnt/disks

# ArgoCD
argocd_enabled: true
argocd_version: v2.10.0

# Registry (harbor)
registry_enabled: false  # Set true if needed
```

**Apply changes**:

```bash
ansible-playbook -i inventory/project-atlas/inventory.ini \
  --become --become-user=root \
  cluster.yml --tags=addons
```

### Change CNI Plugin

**Want to use Cilium instead of Calico?**

```yaml
# group_vars/k8s_cluster/k8s-cluster.yml
kube_network_plugin: cilium
```

**Redeploy** (requires cluster reset):

```bash
ansible-playbook -i inventory/project-atlas/inventory.ini reset.yml
ansible-playbook -i inventory/project-atlas/inventory.ini cluster.yml
```

### Configure etcd Backup

**Enable etcd snapshots**:

```yaml
# group_vars/etcd.yml
etcd_backup_enabled: true
etcd_backup_schedule: "0 2 * * *"  # Daily at 2 AM
etcd_backup_retention: 7  # Keep 7 days
etcd_backup_directory: /var/backups/etcd
```

**Verify backups**:

```bash
ssh ubuntu@192.168.100.101 "sudo ls -lh /var/backups/etcd/"
```

---

## Troubleshooting

### Common Issues

**Issue 1: Playbook Fails on Specific Task**

```bash
# Run with verbose output
ansible-playbook -i inventory/project-atlas/inventory.ini cluster.yml -vvv

# Focus on failed task
# Look for:
# - Network connectivity issues
# - Missing dependencies
# - Permission errors
```

**Issue 2: Nodes Not Joining**

**Check**:

```bash
# SSH to node
ssh ubuntu@192.168.100.111

# Check kubelet logs
sudo journalctl -u kubelet -f

# Check containerd
sudo systemctl status containerd

# Check network
ping 192.168.100.101  # Control plane
```

**Issue 3: Playbook Hangs**

**Common causes**:
- SSH timeout (increase in `ansible.cfg`)
- Slow network
- Under-resourced VMs

**Fix**:

```ini
# ansible.cfg
[defaults]
timeout = 60
host_key_checking = False
```

### Debug Mode

**Run single task**:

```bash
# Test connectivity
ansible all -i inventory/project-atlas/inventory.ini -m ping

# Check gathered facts
ansible all -i inventory/project-atlas/inventory.ini -m setup

# Run specific tag
ansible-playbook -i inventory/project-atlas/inventory.ini cluster.yml --tags=etcd
```

---

## Comparison: kubeadm vs kubespray

### What You Learned

| Task | kubeadm (Manual) | kubespray (Automated) |
|------|------------------|----------------------|
| Install containerd | SSH to each node, run apt | One playbook |
| Disable swap | SSH to each node, edit fstab | Automatic |
| Install kubeadm | SSH to each node, add repo | Automatic |
| Init control plane | Run kubeadm init | Automatic |
| Join nodes | Copy join command to each | Automatic |
| Install CNI | Apply manifest | Automatic |
| Install add-ons | Helm install each | Configure YAML |
| Scale cluster | Manual join | Update inventory + run scale.yml |
| Upgrade K8s | Manual drain/upgrade each | Run upgrade-cluster.yml |
| **Total time** | 2-3 hours (first time) | 15-20 minutes |
| **Repeatability** | Error-prone | Perfect |

### When to Use Each

**Use kubeadm when**:
- Learning K8s architecture
- Studying for CKA exam
- Debugging specific component
- One-off cluster

**Use kubespray when**:
- Production deployments
- Multiple clusters
- Automated CI/CD
- Enterprise environments

---

## Integration with Project Atlas

### Kubernetes + ISP Lab

**Now you have**:
- Network simulation (EVE-NG ISP lab)
- Compute platform (Kubernetes cluster)

**Next steps**:
1. Deploy apps on K8s
2. Expose via MetalLB (gets IP from 192.168.100.200-250)
3. Apps are reachable through ISP simulation
4. Cloud apps route through PastyNet → IXP → Atlas ISP → On-prem K8s

**Example**:

```bash
# Deploy app on K8s
kubectl create deployment web --image=nginx
kubectl expose deployment web --port=80 --type=LoadBalancer

# Gets MetalLB IP
kubectl get svc web
# NAME   TYPE           CLUSTER-IP     EXTERNAL-IP       PORT(S)
# web    LoadBalancer   10.96.50.123   192.168.100.200   80:31234/TCP

# Now 192.168.100.200 is routable through your ISP lab!
# Cloud → PastyNet → IXP → Atlas → 192.168.100.200
```

---

## Version Control Your Kubespray Config

### Git Repository

```bash
# Create repo for your kubespray configs
cd ~/
git init kubespray-project-atlas

# Copy inventory
cp -r kubespray/inventory/project-atlas kubespray-project-atlas/

# Add to git
cd kubespray-project-atlas
git add .
git commit -m "Initial kubespray config for Project Atlas"

# Push to GitHub
git remote add origin git@github.com:youruser/kubespray-project-atlas.git
git push -u origin main
```

**Now you can**:
- Track changes to cluster config
- Review before applying
- Rollback if needed
- Share with team

---

## Next Steps

1. ✅ Kubernetes deployed with kubespray
2. ✅ Understand automation vs manual
3. → Deploy observability stack (guide 13)
4. → Set up GitOps with ArgoCD (guide 15)
5. → Multi-cluster setup (on-prem + cloud)

---

*This guide taught you production Kubernetes automation. You now have both deep understanding (kubeadm) and automation skills (kubespray) - exactly what senior roles require.*

## VERSION HISTORY

- v1.0 (2026-02-09): Kubespray automation guide for Project Atlas K8s cluster
