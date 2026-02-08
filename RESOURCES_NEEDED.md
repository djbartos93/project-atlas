# PROJECT ATLAS - RESOURCES & PREREQUISITES
## Complete Tool List with Download Links

**Last Updated:** 2026-02-08

---

## REQUIRED DOWNLOADS

### Core Infrastructure Software

| Software | Version | Download Link | Size | Purpose |
|----------|---------|---------------|------|---------|
| **Proxmox VE** | 8.x (latest) | https://www.proxmox.com/en/downloads/proxmox-virtual-environment/iso | ~1.2GB | Hypervisor |
| **EVE-NG Community** | Latest | https://www.eve-ng.net/index.php/download/ | ~3GB | Network simulation |
| **Terraform** | Latest stable | https://www.terraform.io/downloads | ~50MB | Infrastructure as Code |
| **Ansible** | Latest | https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html | Via package manager | Configuration management |
| **kubectl** | Latest | https://kubernetes.io/docs/tasks/tools/ | ~50MB | Kubernetes CLI |
| **Helm** | 3.x | https://helm.sh/docs/intro/install/ | ~15MB | K8s package manager |
| **FluxCD CLI** | Latest | https://fluxcd.io/flux/installation/ | ~20MB | GitOps CLI |
| **Git** | Latest | https://git-scm.com/downloads | Varies | Version control |

### Network Device Images

#### Option A: Cisco Platforms (Requires License)

| Platform | Exact Filename | Version | Download Source | Size |
|----------|----------------|---------|-----------------|------|
| **Cisco IOS-XRv** | `iosxrv-k9-demo-7.3.1.qcow2` | 7.3.1+ | Cisco Software Download / CML | ~1GB |
| **Cisco CSR1000v** | `csr1000v-universalk9.17.03.01.qcow2` | 17.3+ | Cisco Software Download / CML | ~1.5GB |
| **Cisco Nexus 9000v** | `nexus9300v.10.3.1.F.qcow2` | 10.3+ | Cisco Software Download | ~1.5GB |

**Download Links (Requires Account):**
- Cisco Software Download: https://software.cisco.com/
- Cisco DevNet: https://developer.cisco.com/
- Cisco CML: https://learningnetworkstore.cisco.com/cisco-modeling-labs-personal

#### Option B: Open Source / Free (Recommended Start)

| Platform | Filename | Download Link | Size |
|----------|----------|---------------|------|
| **VyOS** | `vyos-1.4-rolling-*.iso` | https://vyos.net/get/snapshots/ | ~500MB |
| **FRRouting** | Ubuntu 22.04 + FRR package | https://frrouting.org/ | ~4GB |
| **pfSense** | `pfSense-CE-2.7.0-RELEASE-amd64.iso` | https://www.pfsense.org/download/ | ~500MB |
| **OPNsense** | `OPNsense-*.img` | https://opnsense.org/download/ | ~500MB |

#### Option C: Other Vendor Platforms

| Platform | Filename | Download Link | Size | Account Required |
|----------|----------|---------------|------|------------------|
| **Juniper vMX** | `vmx-bundle-21.4R1.12.ova` | https://support.juniper.net/ | ~2GB | Yes (free) |
| **Arista vEOS** | `vEOS-lab-4.28.0F.vmdk` | https://www.arista.com/en/support/software-download | ~1GB | Yes (free) |
| **Fortinet FortiGate** | `FGT_VM64-*.qcow2` | https://support.fortinet.com/ | ~200MB | Yes |
| **MikroTik CHR** | `chr-*.img` | https://mikrotik.com/download | ~50MB | No |

### Utility Software

| Tool | Purpose | Download Link |
|------|---------|---------------|
| **WinSCP** | File transfer (Windows) | https://winscp.net/eng/download.php |
| **FileZilla** | File transfer (Cross-platform) | https://filezilla-project.org/download.php |
| **PuTTY** | SSH client (Windows) | https://www.putty.org/ |
| **Visual Studio Code** | Code editor | https://code.visualstudio.com/ |
| **Draw.io Desktop** | Network diagrams | https://github.com/jgraph/drawio-desktop/releases |

---

## ACCOUNTS TO CREATE

### Cloud Providers

| Service | Signup Link | Free Tier | Purpose |
|---------|-------------|-----------|---------|
| **Oracle Cloud** | https://cloud.oracle.com/free | ✅ 4x ARM (24GB!), Block storage | Primary cloud compute |
| **AWS** | https://aws.amazon.com/free | ✅ 12 months + always-free | Serverless, S3 |
| **Google Cloud** | https://cloud.google.com/free | ✅ $300 credit, always-free | BigQuery, GKE |
| **Azure** | https://azure.microsoft.com/en-us/free | ✅ $200 credit, 12 months | Hybrid identity, Arc |

### Development Tools

| Service | Signup Link | Free Tier | Purpose |
|---------|-------------|-----------|---------|
| **GitHub** | https://github.com/join | ✅ Unlimited public repos | Git hosting, CI/CD |
| **Docker Hub** | https://hub.docker.com/signup | ✅ Unlimited public images | Container registry |
| **GitLab** | https://gitlab.com/users/sign_up | ✅ Free tier | Alternative Git + CI/CD |

### Networking Tools

| Service | Signup Link | Free Tier | Purpose |
|---------|-------------|-----------|---------|
| **Tailscale** | https://login.tailscale.com/start | ✅ 20 devices | VPN mesh network |
| **ZeroTier** | https://www.zerotier.com/ | ✅ 25 devices | Alternative VPN mesh |

### Network Vendor Accounts (For Images)

| Vendor | Account Creation | Purpose |
|--------|------------------|---------|
| **Cisco DevNet** | https://developer.cisco.com/join/ | Free Cisco resources, some images |
| **Arista** | https://www.arista.com/en/login | Free vEOS download |
| **Juniper** | https://userregistration.juniper.net/ | Free vMX/vSRX access |

---

## TOOL INSTALLATION GUIDES

### Terraform Installation

**macOS (Homebrew):**
```bash
brew tap hashicorp/tap
brew install hashicorp/tap/terraform
terraform version
```

**Linux (Ubuntu/Debian):**
```bash
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform
```

**Windows (Chocolatey):**
```powershell
choco install terraform
```

### Ansible Installation

**macOS:**
```bash
brew install ansible
```

**Linux:**
```bash
sudo apt update
sudo apt install software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install ansible
```

**Python (any OS):**
```bash
python3 -m pip install ansible
```

### kubectl Installation

**macOS:**
```bash
brew install kubectl
```

**Linux:**
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

**Windows:**
```powershell
choco install kubernetes-cli
```

### Helm Installation

**Script (Linux/macOS):**
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

**Homebrew:**
```bash
brew install helm
```

### FluxCD CLI Installation

**macOS/Linux:**
```bash
curl -s https://fluxcd.io/install.sh | sudo bash
```

**Homebrew:**
```bash
brew install fluxcd/tap/flux
```

---

## LICENSING & COSTS

### Free Options (Total: $0)

| Component | License | Cost |
|-----------|---------|------|
| Proxmox VE | Community Edition | FREE |
| EVE-NG | Community Edition | FREE |
| VyOS | Open Source | FREE |
| FRRouting | Open Source | FREE |
| K3s | Apache 2.0 | FREE |
| All observability tools | Open Source | FREE |
| Oracle Cloud | Always Free Tier | FREE |
| Tailscale | Free tier | FREE (20 devices) |
| GitHub | Free tier | FREE |

**Total Monthly Cost:** $0

### Paid Options (Optional)

| Component | Cost | Benefits |
|-----------|------|----------|
| **Cisco CML Personal** | $199/year | All Cisco images legally + support |
| **EVE-NG Professional** | $90/year | Better performance, OVA format |
| **Cisco DevNet Sandbox** | FREE | Cloud-based, time-limited |
| **Visual Studio Subscription** | $539/year | Windows Server, SQL Server licenses |
| **VMware VMUG Advantage** | $200/year | VMware lab licenses (if not using Proxmox) |

### Recommended Budget Options

**Minimal (Pure Open Source):**
- Cost: $0/year
- Uses: VyOS, FRRouting, Proxmox, K3s
- Portfolio value: Good (shows cost-consciousness)

**Standard (Hybrid):**
- Cost: $90-199/year
- Uses: CML or EVE-NG Pro
- Portfolio value: Better (real vendor gear)

**Professional (Full Commercial):**
- Cost: $400-600/year
- Uses: CML + EVE-NG Pro + VS Subscription
- Portfolio value: Best (production tools)

---

## HARDWARE REQUIREMENTS

### System Requirements

While this project is designed to run on my homelab it can be scaled up or down based on your needs. Below are my specs that run this project.

**For This Project:**
- 3x Physical servers (HP DL380 G8)
- 144 cores total
- 1.14TB RAM total
- 10G networking 
- UniFi network gear 

**Per Proxmox Node:**
- 40 vCPUs
- 320GB RAM
- 100GB OS storage
- 4x network adapters

**EVE-NG VM:**
- 24 vCPUs
- 120GB RAM
- 200GB storage


### Storage Requirements

**Total Storage Needed:**
- Proxmox OS: 3x 100GB = 300GB
- EVE-NG: 200GB
- K8s VMs: 6x 50GB = 300GB
- Router images: ~10GB
- Container images: ~50GB
- Persistent volumes: ~200GB (grows over time)

**Total: ~1.2TB** 


## REFERENCE DOCUMENTATION

### Official Docs

| Resource | URL | Purpose |
|----------|-----|---------|
| Proxmox Wiki | https://pve.proxmox.com/wiki/ | Hypervisor reference |
| EVE-NG Docs | https://www.eve-ng.net/index.php/documentation/ | Network simulation |
| K3s Docs | https://docs.k3s.io/ | Lightweight Kubernetes |
| Terraform Docs | https://developer.hashicorp.com/terraform | IaC provisioning |
| Ansible Docs | https://docs.ansible.com/ | Configuration management |
| Cilium Docs | https://docs.cilium.io/ | eBPF networking |
| Prometheus Docs | https://prometheus.io/docs/ | Monitoring |
| Grafana Docs | https://grafana.com/docs/ | Visualization |

### Learning Resources

| Resource | URL | Topic |
|----------|-----|-------|
| **Cisco MPLS Fundamentals** | Book (Amazon) | MPLS theory/practice |
| **BGP Design and Implementation** | Cisco Press | BGP deep dive |
| **Kubernetes Patterns** | O'Reilly | K8s best practices |
| **Infrastructure as Code** | O'Reilly | IaC principles |
| **EVE-NG Cookbook** | Community forums | EVE-NG tips/tricks |

### Community Resources

| Resource | URL | Purpose |
|----------|-----|---------|
| r/homelab | https://reddit.com/r/homelab | Homelab community |
| r/networking | https://reddit.com/r/networking | Network engineering |
| EVE-NG Forum | https://www.eve-ng.net/index.php/community/forums/ | EVE-NG support |
| Proxmox Forum | https://forum.proxmox.com/ | Proxmox support |
| K8s Slack | https://kubernetes.slack.com/ | Kubernetes community |


## VERSION HISTORY

- v1.0 (2026-02-07): Initial resource compilation
- v1.1 (2026-02-07): Added detailed image requirements
- v1.2 (2026-02-07): Added installation guides
- v1.3 (2026-02-08): Cleaned up and updated original AI documentation

---

_Keep this document updated as new resources are discovered or tools updated._
