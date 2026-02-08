# Terraform Learning Guide for Project Atlas

# This file has not yet been reviewed by a human!

This directory contains Infrastructure as Code (IaC) for provisioning all virtual infrastructure.

## 🎓 What is Terraform?

Terraform is a tool for **declaratively** defining infrastructure. You describe *what* you want, and Terraform figures out *how* to create it.

### Key Concepts

**1. Providers** - Plugins that let Terraform talk to APIs
   - We use: `telmate/proxmox` for Proxmox, `oracle/oci` for Oracle Cloud

**2. Resources** - Things you want to create
   - Example: `proxmox_vm_qemu` = a VM in Proxmox

**3. State** - Terraform's memory of what it created
   - Stored in `.tfstate` files (don't commit these!)
   - Tracks real infrastructure vs desired state

**4. Plan** - Preview of what will change
   - Run `terraform plan` to see what will happen
   - **Always review before applying!**

**5. Modules** - Reusable chunks of Terraform code
   - Like functions in programming
   - Located in `modules/` directory

## 📁 Directory Structure

```
terraform/
├── modules/              # Reusable modules (write once, use many times)
│   ├── proxmox-vm/      # Module to create a Proxmox VM
│   ├── eve-ng/          # Module to deploy EVE-NG
│   ├── k8s-node/        # Module for K8s node VMs
│   └── oci-instance/    # Module for OCI instances
├── vmware/              # VMware-specific configs (if needed)
├── proxmox/             # Root configs for Proxmox infrastructure
│   ├── main.tf          # Main configuration
│   ├── variables.tf     # Input variables
│   ├── outputs.tf       # Output values
│   ├── terraform.tfvars # Your actual values (DON'T COMMIT!)
│   └── eve-ng.tf        # EVE-NG specific config
├── oci/                 # Oracle Cloud configs
│   ├── main.tf
│   ├── variables.tf
│   └── k8s-cluster.tf
```

## 🚀 Getting Started

### Step 1: Install Terraform

```bash
# macOS
brew install terraform

# Linux
# Follow: https://developer.hashicorp.com/terraform/downloads

# Verify
terraform version
```

### Step 2: Learn Basic Syntax

Create a test directory outside this project:

```bash
mkdir ~/terraform-test && cd ~/terraform-test
```

Create `main.tf`:
```hcl
# Provider: tells Terraform what API to use
provider "local" {}

# Resource: what to create
resource "local_file" "hello" {
  filename = "${path.module}/hello.txt"
  content  = "Hello from Terraform!"
}

# Output: get information back
output "file_path" {
  value = local_file.hello.filename
}
```

Run it:
```bash
terraform init    # Downloads provider plugins
terraform plan    # Shows what will happen
terraform apply   # Actually does it
terraform destroy # Cleans up
```

### Step 3: Understand Proxmox Provider

The Proxmox provider needs:
- API endpoint (https://proxmox-host:8006/api2/json)
- API token or username/password
- TLS verification settings

Example minimal config:
```hcl
terraform {
  required_providers {
    proxmox = {
      source  = "telmate/proxmox"
      version = "~> 2.9"
    }
  }
}

provider "proxmox" {
  pm_api_url      = "https://192.168.100.11:8006/api2/json"
  pm_user         = "root@pam"
  pm_password     = var.proxmox_password  # Never hardcode!
  pm_tls_insecure = true  # Only for homelab!
}
```

## 🔧 Building Your First Module

**Module**: `proxmox-vm`

Purpose: Create a VM in Proxmox with consistent settings

### What You Need to Learn:

1. **Required vs Optional Arguments**
   - Read the provider docs: https://registry.terraform.io/providers/Telmate/proxmox/latest/docs
   - Understand what `proxmox_vm_qemu` needs
   - Decide what to make configurable

2. **Variables**
   - What should be different each time you use this module?
   - Examples: VM name, CPU count, RAM, IP address

3. **Outputs**
   - What information do you need back?
   - Examples: VM ID, IP address, SSH connection string

### Example Module Structure:

`modules/proxmox-vm/main.tf`:
```hcl
# This is the actual resource creation
resource "proxmox_vm_qemu" "vm" {
  name        = var.vm_name
  target_node = var.target_node

  # Use variables for everything that changes
  cores   = var.cpu_cores
  memory  = var.memory_mb

  # Network config
  network {
    model  = "virtio"
    bridge = var.network_bridge
    tag    = var.vlan_tag
  }

  # Disk config
  disk {
    slot    = 0
    size    = "${var.disk_size_gb}G"
    type    = "scsi"
    storage = var.storage_pool
  }
}
```

`modules/proxmox-vm/variables.tf`:
```hcl
# Define what can be customized
variable "vm_name" {
  description = "Name of the VM"
  type        = string
}

variable "cpu_cores" {
  description = "Number of CPU cores"
  type        = number
  default     = 2  # Sensible default
}

# ... more variables
```

`modules/proxmox-vm/outputs.tf`:
```hcl
# What information to return
output "vm_id" {
  description = "Proxmox VM ID"
  value       = proxmox_vm_qemu.vm.id
}

output "vm_ip" {
  description = "VM IP address"
  value       = proxmox_vm_qemu.vm.default_ipv4_address
}
```

### Using Your Module:

`proxmox/main.tf`:
```hcl
module "eve_ng" {
  source = "../modules/proxmox-vm"

  vm_name       = "eve-ng"
  target_node   = "proxmox-01"
  cpu_cores     = 16
  memory_mb     = 65536
  disk_size_gb  = 200
  network_bridge = "vmbr0"
  vlan_tag      = 100
  storage_pool  = "truenas-nfs"
}

# Access outputs
output "eve_ng_id" {
  value = module.eve_ng.vm_id
}
```

## 🔐 Managing Secrets

**NEVER commit secrets to Git!**

### Method 1: Environment Variables
```bash
export TF_VAR_proxmox_password="your-password"
terraform plan
```

### Method 2: terraform.tfvars (gitignored)
```hcl
# terraform.tfvars (this file is in .gitignore)
proxmox_password = "your-password"
proxmox_api_url  = "https://192.168.100.11:8006/api2/json"
```

### Method 3: terraform.tfvars.example (template)
```hcl
# terraform.tfvars.example (commit this as a template)
proxmox_password = "CHANGEME"
proxmox_api_url  = "https://PROXMOX-IP:8006/api2/json"
```

User copies to `terraform.tfvars` and fills in real values.

## 🧪 Testing Workflow

**Always follow this pattern:**

```bash
# 1. Initialize (first time, or after adding providers)
terraform init

# 2. Validate syntax
terraform validate

# 3. Preview changes
terraform plan

# 4. Review the plan carefully!
#    - What will be created?
#    - What will be modified?
#    - What will be destroyed?

# 5. Apply if plan looks good
terraform apply

# 6. Verify it worked
#    - Check Proxmox UI
#    - SSH to the VM
#    - Test connectivity

# 7. If something broke
terraform destroy  # Clean up
# Fix the code
# Try again
```

## 📖 Learning Resources

**Essential Reading:**
- Terraform Getting Started: https://developer.hashicorp.com/terraform/tutorials/aws-get-started
- Proxmox Provider Docs: https://registry.terraform.io/providers/Telmate/proxmox/latest/docs
- Terraform Best Practices: https://www.terraform-best-practices.com/

**Video Tutorials:**
- "Terraform in 15 Minutes" - YouTube
- "Terraform Course" by FreeCodeCamp
- "Proxmox + Terraform" tutorials

**Practice Projects:**
- Create 3 VMs with different configurations
- Use modules to avoid repetition
- Output a connection string for each VM
- Destroy and recreate to test idempotency

## ⚠️ Common Mistakes

**1. Not using variables**
```hcl
# Bad: hardcoded values
resource "proxmox_vm_qemu" "vm" {
  name = "my-vm"
  cores = 4
}

# Good: parameterized
resource "proxmox_vm_qemu" "vm" {
  name  = var.vm_name
  cores = var.cpu_cores
}
```

**2. Not planning before applying**
```bash
# Bad
terraform apply -auto-approve  # DANGEROUS!

# Good
terraform plan  # Review first!
terraform apply # Then confirm
```

**3. Committing secrets**
```bash
# Make sure .gitignore includes:
*.tfvars
*.tfstate
*.tfstate.*
```

**4. Not using modules**
- Don't copy-paste the same resource definition
- Create a module and reuse it

## 🎯 Your Learning Path

### Week 1: Basics
- [ ] Install Terraform
- [ ] Complete "Getting Started" tutorial
- [ ] Understand providers, resources, variables, outputs
- [ ] Create a simple local file resource

### Week 2: Proxmox Integration
- [ ] Set up Proxmox API access
- [ ] Create one VM manually with Terraform
- [ ] Destroy and recreate it
- [ ] Understand Terraform state

### Week 3: Modules
- [ ] Create `proxmox-vm` module
- [ ] Use module to create EVE-NG VM
- [ ] Use module to create K8s nodes
- [ ] Understand module inputs/outputs

### Week 4: Advanced
- [ ] Use `for_each` to create multiple VMs
- [ ] Use `depends_on` for ordering
- [ ] Implement remote state (if needed)
- [ ] Add OCI provider for cloud resources

## 🤔 Questions to Answer as You Learn

1. **Why use Terraform instead of clicking in Proxmox UI?**
   - Reproducibility: rebuild everything from code
   - Version control: track changes in Git
   - Documentation: code IS documentation
   - Automation: part of larger automation strategy

2. **What's the difference between `terraform plan` and `apply`?**
   - Plan: "here's what I would do"
   - Apply: "I'm actually doing it now"

3. **What is Terraform state and why does it matter?**
   - State tracks what Terraform created
   - Used to detect drift (manual changes)
   - Used to plan updates/deletions
   - Should be backed up!

4. **When should I use modules?**
   - When you're creating the same thing multiple times
   - When you want to reuse code
   - When you want to organize complex configs

5. **How do I update existing infrastructure?**
   - Modify the Terraform code
   - Run `terraform plan` to see what changes
   - Run `terraform apply` to make changes
   - Terraform will update, not recreate (usually)

---

Remember: **Start simple, build complexity gradually**. Don't try to automate everything at once. Get one thing working, understand it fully, then move to the next.

Good luck! 🚀
