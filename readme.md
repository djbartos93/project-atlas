# Project Atlas 🌐

**A Production-Grade Homelab ISP Simulation with Complete Infrastructure as Code**

> *Building a carrier-grade network simulation that routes all infrastructure through a BGP/MPLS core, with complete automation to rebuild everything in under an hour.*

---

## Note on AI use

This project leverages AI as a collaborative tool, but learning through hands-on implementation remains the primary goal. 

While I use AI to help refine documentation or "clean up" existing configurations for readability, all logic and core configuration files are authored manually. Any documents that have not yet been reviewed by myself will state that they have not been reviewed.  Most working or in progress documentatoin has been created with AI assistance, but has been reviewed by myself and includes the real values used in the lab (IP addresses, hostnames, etc.). 

Every line of code and piece of information has been personally reviewed and validated to ensure accuracy. This repository is about mastering real-world technologies, not letting a model do the work for me. In the case of documents that have not been reviewed, and will not be reviewed, theywill state that they have been created with AI and will not be edited by a human. 

## 📋 Project Overview

Project Atlas is an enterprise-grade homelab that simulates a complete Tier-2 ISP network. Unlike typical homelabs, **all infrastructure traffic routes through a simulated carrier network** built in EVE-NG, providing hands-on experience with:

- 🌐 BGP routing and policies
- 🔀 MPLS L3VPN and traffic engineering
- 🏗️ Carrier-grade network design
- 🌍 Multi-site connectivity simulation
- 🏢 Enterprise WAN architectures

## 🎯 Project Goals

1. **Resume Building** - Create portfolio-quality enterprise networking experience
2. **Skill Development** - Learn carrier/ISP networking, cloud-native technologies
3. **Automation Mastery** - Infrastructure as Code with Terraform + Ansible + GitOps
4. **Interview Prep** - Demonstrate real-world skills

## 🏗️ Architecture Summary

### Physical Layer
- **3x HP DL380 G8 Servers** (144 cores, 1.14TB RAM total)
- **Nested Proxmox VE** on VMware ESXi
- **TrueNAS** storage backend 
- **UniFi** home network infrastructure

### Technology Stack
- **Hypervisor**: Proxmox VE 8.x (nested on VMware)
- **Network Simulation**: EVE-NG Community Edition
- **Network Devices**: Cisco IOS-XRv/CSR1000v, Juniper vMX, Arista vEOS, VyOS
- **Kubernetes**: K3s with Cilium CNI
- **IaC**: Terraform + Ansible
- **GitOps**: FluxCD
- **Observability**: Prometheus + Grafana + Loki
- **Cloud**: Oracle Cloud (OCI), AWS, Azure, GCP


### Prerequisites
- 3x Proxmox VMs deployed on VMware
- EVE-NG installed and running
- Network device images loaded
- Git, Terraform, Ansible installed on workstation


## 🎓 Learning Resources

This project is designed as a learning journey. Key concepts covered:

### Networking
- BGP fundamentals and policy
- MPLS L3VPN architecture
- ISIS/OSPF interior gateway protocols
- Carrier network design patterns
- Multi-vendor interoperability

### Cloud & Containers
- Kubernetes architecture and operations
- Service mesh with Istio
- eBPF-based networking (Cilium)
- Multi-cloud hybrid architecture
- GitOps workflows

### Infrastructure as Code
- Terraform provider development
- Ansible role architecture
- CI/CD for infrastructure
- Secrets management best practices
- Testing infrastructure code


### Advanced BGP Concepts
* **Peering Relationships**
   - Settlement-free peering (Tier-1 ↔ Tier-1)
   - Paid transit (AS65000 → Tier-1)
   - IXP multilateral peering

* **Route Selection**
   - Local preference for inbound control
   - AS path prepending for traffic engineering
   - MED for multi-exit discrimination
   - Route preference: Peered routes > Transit routes

* **Multi-homing**
   - Active/active load balancing
   - Primary/backup scenarios
   - Failure detection and convergence

### IXP Operations
* **Route Server Benefits**
   - One BGP session = peer with many networks
   - Route filtering and policies
   - Simplified configuration

* **Peering Strategies**
   - When to use route server
   - When to establish bilateral sessions
   - Hybrid approaches

### MPLS L3VPN
* **Customer Isolation**
   - VRF per customer
   - Overlapping IP addresses
   - Route distinguishers (RD)
   - Route targets (RT)

* **MP-BGP**
   - VPNv4 address family
   - Route distribution between PEs
   - Label stack operation


## 🔐 Security Notes

- All secrets managed via Ansible Vault or HashiCorp Vault
- Network isolation between lab and production environments
- No credentials committed to Git
- TLS/encryption for all inter-cluster communication
- Regular security scanning with Trivy

## 🤝 Contributing

This is a personal learning project, but ideas and suggestions are welcome! If you're building something similar, feel free to use this as inspiration.

## 📝 License

This project is for educational purposes. Network device images must be sourced legally.

