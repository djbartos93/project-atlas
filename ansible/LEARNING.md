# Ansible Learning Guide for Project Atlas

This directory contains configuration management code for automating system configuration.

## 🎓 What is Ansible?

Ansible is a tool for **automating system configuration**. While Terraform provisions infrastructure (creates VMs), Ansible configures it (installs software, changes settings, etc.).

### Key Differences: Terraform vs Ansible

| Aspect | Terraform | Ansible |
|--------|-----------|---------|
| **Purpose** | Provision infrastructure | Configure systems |
| **State** | Maintains state file | Stateless (mostly) |
| **Language** | HCL (declarative) | YAML (procedural) |
| **Example Use** | Create a VM | Install packages on VM |
| **Idempotent** | Yes | Yes (when done right) |

**In this project:**
- Terraform creates the VMs
- Ansible configures them (updates, installs software, configures services)

## 🔑 Key Concepts

**1. Inventory** - List of hosts you manage
```ini
[proxmox_nodes]
proxmox-01 ansible_host=192.168.100.11
proxmox-02 ansible_host=192.168.100.12
proxmox-03 ansible_host=192.168.100.13

[eve_ng]
eve-ng ansible_host=192.168.100.50
```

**2. Playbooks** - What you want to do
```yaml
---
- name: Configure Proxmox nodes
  hosts: proxmox_nodes
  tasks:
    - name: Update apt cache
      apt:
        update_cache: yes
```

**3. Tasks** - Individual steps
- Each task uses a **module** (built-in functionality)
- Examples: `apt`, `yum`, `copy`, `template`, `service`

**4. Roles** - Organized collections of tasks
```
roles/proxmox-cluster/
├── tasks/      # What to do
├── handlers/   # Actions triggered by changes
├── templates/  # Jinja2 templates
├── files/      # Static files to copy
├── vars/       # Variables
└── defaults/   # Default variables
```

**5. Variables** - Data for your playbooks
- Can come from: inventory, playbooks, roles, command line

**6. Modules** - Pre-built functionality
- `apt`: Manage packages on Debian/Ubuntu
- `yum`: Manage packages on RHEL/CentOS
- `copy`: Copy files
- `template`: Deploy Jinja2 templates
- `service`: Manage systemd services
- Hundreds more: https://docs.ansible.com/ansible/latest/collections/

## 📁 Directory Structure

```
ansible/
├── inventory/           # Inventory files
│   ├── hosts.ini       # Main inventory (your servers)
│   └── group_vars/     # Variables for groups
│       ├── all.yml     # Variables for all hosts
│       ├── proxmox_nodes.yml
│       └── vault.yml   # Encrypted secrets
├── roles/              # Reusable roles
│   ├── proxmox-cluster/
│   ├── eve-ng/
│   ├── k3s-server/
│   ├── k3s-agent/
│   └── network-device/
├── proxmox-config.yml  # Playbook for Proxmox
├── eve-ng-config.yml   # Playbook for EVE-NG
├── k8s-cluster.yml     # Playbook for K8s
└── ansible.cfg         # Ansible configuration
```

## 🚀 Getting Started

### Step 1: Install Ansible

```bash
# macOS
brew install ansible

# Linux (Ubuntu/Debian)
sudo apt update
sudo apt install ansible

# Or via pip (any OS)
pip3 install ansible

# Verify
ansible --version
```

### Step 2: Test Basic Connectivity

Create a test inventory:
```bash
mkdir ~/ansible-test && cd ~/ansible-test
cat > inventory.ini <<EOF
[local]
localhost ansible_connection=local
EOF
```

Test ping:
```bash
ansible all -i inventory.ini -m ping
```

You should see:
```json
localhost | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

### Step 3: Your First Playbook

Create `test.yml`:
```yaml
---
- name: My first playbook
  hosts: local
  gather_facts: no  # Skip collecting system info (faster for test)

  tasks:
    - name: Print a message
      debug:
        msg: "Hello from Ansible!"

    - name: Create a file
      file:
        path: /tmp/ansible-test.txt
        state: touch

    - name: Show file was created
      stat:
        path: /tmp/ansible-test.txt
      register: file_info

    - name: Display file info
      debug:
        var: file_info
```

Run it:
```bash
ansible-playbook -i inventory.ini test.yml
```

## 🎯 Your Proxmox Inventory

Create `inventory/hosts.ini`:
```ini
# Proxmox Cluster Nodes
[proxmox_nodes]
proxmox-01 ansible_host=192.168.100.11 ansible_user=root
proxmox-02 ansible_host=192.168.100.12 ansible_user=root
proxmox-03 ansible_host=192.168.100.13 ansible_user=root

# EVE-NG Network Simulator
[eve_ng]
eve-ng ansible_host=192.168.100.50 ansible_user=root

# Kubernetes Nodes (once created)
[k8s_servers]
# k8s-server-01 ansible_host=192.168.100.101
# k8s-server-02 ansible_host=192.168.100.102
# k8s-server-03 ansible_host=192.168.100.103

[k8s_agents]
# k8s-agent-01 ansible_host=192.168.100.111
# k8s-agent-02 ansible_host=192.168.100.112
# k8s-agent-03 ansible_host=192.168.100.113

# Groups of groups
[k8s_cluster:children]
k8s_servers
k8s_agents

[all_vms:children]
proxmox_nodes
eve_ng
k8s_cluster
```

Test connectivity:
```bash
ansible proxmox_nodes -i inventory/hosts.ini -m ping
```

## 📝 Creating Your First Role

**Goal**: Create a role to configure Proxmox nodes

### Step 1: Generate Role Structure

```bash
cd ansible/roles
ansible-galaxy init proxmox-cluster
```

This creates:
```
proxmox-cluster/
├── defaults/       # Default variables
├── files/          # Static files
├── handlers/       # Event handlers
├── meta/           # Role metadata
├── tasks/          # Main tasks
├── templates/      # Jinja2 templates
├── tests/          # Test configs
└── vars/           # Variables
```

### Step 2: Define Tasks

Edit `roles/proxmox-cluster/tasks/main.yml`:
```yaml
---
# Main task file for proxmox-cluster role

- name: Update apt cache
  apt:
    update_cache: yes
    cache_valid_time: 3600  # Cache valid for 1 hour
  tags: ['packages']

- name: Install required packages
  apt:
    name:
      - sudo
      - curl
      - wget
      - vim
      - htop
      - net-tools
    state: present
  tags: ['packages']

- name: Remove Proxmox subscription nag
  lineinfile:
    path: /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js
    regexp: 'Ext.Msg.show'
    line: 'void({ //'
    backup: yes
  notify: Restart Proxmox web service
  tags: ['config']

- name: Configure TrueNAS NFS storage
  command: >
    pvesm add nfs {{ truenas_storage_name }}
    --server {{ truenas_ip }}
    --export {{ truenas_export }}
    --content {{ truenas_content_types }}
  register: storage_result
  failed_when: false  # Don't fail if already exists
  changed_when: storage_result.rc == 0
  tags: ['storage']

- name: Verify Proxmox cluster status
  command: pvecm status
  register: cluster_status
  changed_when: false  # Just checking, not changing
  tags: ['verify']

- name: Display cluster status
  debug:
    var: cluster_status.stdout_lines
  tags: ['verify']
```

### Step 3: Define Variables

Edit `roles/proxmox-cluster/defaults/main.yml`:
```yaml
---
# Default variables for proxmox-cluster role
# These can be overridden in inventory or playbook

truenas_storage_name: "truenas-nfs"
truenas_ip: "10.10.0.10"
truenas_export: "/mnt/tank/proxmox"
truenas_content_types: "images,iso,vztmpl,backup,snippets"
```

### Step 4: Create Handlers

Edit `roles/proxmox-cluster/handlers/main.yml`:
```yaml
---
# Handlers are tasks that run when notified

- name: Restart Proxmox web service
  systemd:
    name: pveproxy
    state: restarted
```

### Step 5: Create Playbook to Use Role

Create `proxmox-config.yml`:
```yaml
---
- name: Configure Proxmox Cluster
  hosts: proxmox_nodes
  become: yes  # Run as root

  roles:
    - proxmox-cluster

  tasks:
    - name: Check Proxmox version
      command: pveversion
      register: pve_version
      changed_when: false

    - name: Display Proxmox version
      debug:
        msg: "Proxmox version: {{ pve_version.stdout }}"
```

### Step 6: Run the Playbook

```bash
# Dry run (check mode) - doesn't make changes
ansible-playbook -i inventory/hosts.ini proxmox-config.yml --check

# Actually run it
ansible-playbook -i inventory/hosts.ini proxmox-config.yml

# Run only specific tags
ansible-playbook -i inventory/hosts.ini proxmox-config.yml --tags packages

# Run on specific hosts
ansible-playbook -i inventory/hosts.ini proxmox-config.yml --limit proxmox-01
```

## 🔐 Managing Secrets with Ansible Vault

**Never commit plaintext passwords!**

### Create Encrypted Variables File

```bash
# Create vault password file (don't commit this!)
echo "YourVaultPassword" > ~/.ansible-vault-pass

# Create encrypted variables file
ansible-vault create inventory/group_vars/vault.yml --vault-password-file ~/.ansible-vault-pass
```

In the editor that opens:
```yaml
---
proxmox_root_password: "your-secret-password"
truenas_api_key: "your-api-key"
oci_auth_token: "your-oci-token"
```

### Use Encrypted Variables

In your playbook:
```yaml
---
- name: Example using vault variables
  hosts: proxmox_nodes
  vars_files:
    - inventory/group_vars/vault.yml

  tasks:
    - name: Do something with secret
      debug:
        msg: "Root password is {{ proxmox_root_password }}"
      no_log: true  # Don't print secrets to console!
```

### Run with Vault

```bash
ansible-playbook -i inventory/hosts.ini playbook.yml --vault-password-file ~/.ansible-vault-pass
```

## 🧪 Testing and Development

### Ad-hoc Commands

Test things quickly:
```bash
# Check disk space on all nodes
ansible proxmox_nodes -i inventory/hosts.ini -m shell -a "df -h"

# Check uptime
ansible all -i inventory/hosts.ini -a "uptime"

# Install a package quickly
ansible eve_ng -i inventory/hosts.ini -m apt -a "name=python3-pip state=present" -b
```

### Check Mode (Dry Run)

Always test before making changes:
```bash
ansible-playbook -i inventory/hosts.ini playbook.yml --check --diff
```
- `--check`: Don't actually make changes
- `--diff`: Show what would change

### Limit to Specific Hosts

```bash
# Run on just one host
ansible-playbook -i inventory/hosts.ini playbook.yml --limit proxmox-01

# Run on multiple hosts
ansible-playbook -i inventory/hosts.ini playbook.yml --limit "proxmox-01,proxmox-02"
```

### Tags for Selective Running

```bash
# Run only tasks tagged 'packages'
ansible-playbook -i inventory/hosts.ini playbook.yml --tags packages

# Skip tasks tagged 'slow'
ansible-playbook -i inventory/hosts.ini playbook.yml --skip-tags slow
```

## 📖 Learning Resources

**Essential Reading:**
- Ansible Getting Started: https://docs.ansible.com/ansible/latest/getting_started/
- Ansible Best Practices: https://docs.ansible.com/ansible/latest/tips_tricks/ansible_tips_tricks.html
- "Ansible for DevOps" book by Jeff Geerling

**Video Tutorials:**
- "Ansible in 100 Seconds" - YouTube
- "Learn Ansible" by Learn Linux TV

**Practice:**
- Configure your Proxmox nodes (start simple!)
- Set up EVE-NG with Ansible
- Create reusable roles

## ⚠️ Common Mistakes

**1. Not using `become` when needed**
```yaml
# Bad: will fail if you need root permissions
- name: Install package
  apt:
    name: htop

# Good: elevate privileges
- name: Install package
  apt:
    name: htop
  become: yes
```

**2. Not making tasks idempotent**
```yaml
# Bad: always shows as "changed"
- name: Add line to file
  shell: echo "something" >> /etc/file.conf

# Good: idempotent
- name: Add line to file
  lineinfile:
    path: /etc/file.conf
    line: "something"
```

**3. Hardcoding values**
```yaml
# Bad
- name: Configure storage
  command: pvesm add nfs truenas-nfs --server 10.10.0.10

# Good: use variables
- name: Configure storage
  command: pvesm add nfs {{ storage_name }} --server {{ storage_ip }}
```

**4. Not handling errors**
```yaml
# Bad: fails if file doesn't exist
- name: Remove file
  file:
    path: /tmp/somefile
    state: absent

# Good: won't fail
- name: Remove file if exists
  file:
    path: /tmp/somefile
    state: absent
  ignore_errors: yes
```

## 🎯 Your Learning Path

### Week 1: Basics
- [ ] Install Ansible
- [ ] Create inventory for your lab
- [ ] Test connectivity with `ping` module
- [ ] Write a simple playbook with 3-5 tasks
- [ ] Run playbook with `--check` first

### Week 2: Roles
- [ ] Create `proxmox-cluster` role
- [ ] Add tasks one at a time, testing each
- [ ] Use variables for configuration
- [ ] Create handlers for service restarts

### Week 3: Advanced
- [ ] Create `eve-ng` role
- [ ] Use templates with Jinja2
- [ ] Set up Ansible Vault for secrets
- [ ] Create `k3s-server` and `k3s-agent` roles

### Week 4: Integration
- [ ] Combine Terraform and Ansible
- [ ] Terraform creates VMs
- [ ] Ansible configures them
- [ ] Test full automation workflow

## 🤔 Key Questions to Answer

**1. Why use Ansible instead of manual SSH commands?**
- Idempotency: can run multiple times safely
- Scale: configure many servers at once
- Documentation: code shows what you did
- Reusability: roles can be shared/reused

**2. What's the difference between a playbook and a role?**
- Playbook: specific list of tasks to run
- Role: organized, reusable collection of tasks
- Use playbooks to orchestrate, roles for organization

**3. When should I use `shell` vs built-in modules?**
- Prefer built-in modules when possible (more reliable, idempotent)
- Use `shell`/`command` only when no module exists
- Always check if a module exists first!

**4. How do I know if my tasks are idempotent?**
- Run the playbook twice
- Second run should show "changed=0"
- If always shows "changed", fix it!

---

**Remember**: Start with one simple task, test it, then add more. Don't try to automate everything at once!

Good luck! 🚀
