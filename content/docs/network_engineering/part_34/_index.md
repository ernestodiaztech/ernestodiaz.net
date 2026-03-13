---
draft: true
title: '34 - Infrastructure-as-Code'
description: "Part 34 of my Ansible learning geared towards Network Engineering."
tags:
  - ansible
  - networking
categories:
  - automation
sidebar:
  exclude: true
weight: 1
---

{{< badge "Ansible" >}}
{{< badge content="VS Code" color="blue" >}}
{{< badge content="Linux" color="red" >}}

## Part 34: Infrastructure-as-Code — Provisioning VMs with Ansible

> *Everything built so far has been automation of network devices. This part turns that same discipline inward — the infrastructure running the automation gets automated too. Every VM in the stack is provisioned by a playbook, not clicked into existence through the Proxmox web UI. The result is that the entire lab environment — six VMs, all services, all configuration — can be rebuilt from a clean Proxmox host by running a single command. That's the IaC principle applied to infrastructure, and it's exactly what a senior network automation engineer is expected to understand.*

---

## 34.1 — Proxmox API Authentication

Before any Ansible module can talk to Proxmox, it needs credentials. There are two methods.

### Method comparison

```
Method 1: root@pam + password
  How it works: Pass the root username and password directly
  Pro:  Zero setup — works immediately
  Con:  Root password in vault (high blast radius if leaked)
        PVE API calls as root bypass all RBAC controls
        Audit log shows every action as root — no attribution
  Use when: Quick testing only, never in a persistent lab

Method 2: API token (what this guide uses)
  How it works: Create a dedicated PVE user + role + token
  Pro:  Token can be scoped to specific permissions
        Token can be revoked independently of any password
        Audit log shows actions attributed to the token name
        Token secret goes in vault — root password stays offline
  Con:  5 minutes of setup upfront
  Use when: Always — this is the correct approach
```

### Creating the API token (run on Proxmox host)

```bash
# SSH to Proxmox host
ssh root@172.16.0.1

# Create a dedicated PVE user for Ansible
pveum user add ansible-api@pve \
  --comment "Ansible automation API user" \
  --password "$(openssl rand -base64 24)"
# Password is set but irrelevant — token auth doesn't use it

# Create a role with the permissions Ansible needs for VM management
pveum role add AnsibleVMManager \
  --privs "VM.Allocate,VM.Clone,VM.Config.CDROM,VM.Config.CPU,\
VM.Config.Cloudinit,VM.Config.Disk,VM.Config.HWType,\
VM.Config.Memory,VM.Config.Network,VM.Config.Options,\
VM.Monitor,VM.PowerMgmt,VM.Snapshot,VM.Snapshot.Rollback,\
Datastore.AllocateSpace,Datastore.AllocateTemplate,\
Datastore.Audit,SDN.Use,Sys.Audit,Pool.Audit"

# Assign the role to the user at the root path (applies to all nodes)
pveum aclmod / \
  --user ansible-api@pve \
  --role AnsibleVMManager

# Create an API token for the user
# --privsep 0 means the token inherits the user's full permissions
pveum user token add ansible-api@pve ansible \
  --privsep 0 \
  --comment "Ansible provisioning token"

# Output will show the token secret — COPY IT NOW, it's shown only once:
# ┌──────────────┬──────────────────────────────────────┐
# │ key          │ value                                │
# ╞══════════════╪══════════════════════════════════════╡
# │ full-tokenid │ ansible-api@pve!ansible              │
# │ info         │ {"comment":"Ansible provisioning..."}│
# │ value        │ xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx │
# └──────────────┴──────────────────────────────────────┘

# Verify the token works
curl -s \
  -H "Authorization: PVEAPIToken=ansible-api@pve!ansible=<token-value>" \
  "https://172.16.0.1:8006/api2/json/version" \
  | python3 -m json.tool | grep release
```

### Store the token in Ansible Vault

```bash
# On the automation VM / control node
cd ~/projects/ansible-network

# Add Proxmox credentials to the vault
ansible-vault edit inventory/group_vars/all/vault.yml \
  --vault-id lab@.vault/lab.txt

# Add these lines:
vault_proxmox_api_user: "ansible-api@pve!ansible"
vault_proxmox_api_token: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
vault_proxmox_host: "172.16.0.1"
vault_proxmox_node: "pve"          # Proxmox node name — check with: pvesh get /nodes
vault_proxmox_template_id: 9000   # VM ID of the cloud-init template from Part 33
```

### Group vars reference

```bash
# Update infrastructure group vars to reference vault values
cat >> ~/projects/ansible-network/inventory/group_vars/infrastructure.yml << 'EOF'

# Proxmox API credentials (values from vault)
proxmox_api_user: "{{ vault_proxmox_api_user }}"
proxmox_api_token: "{{ vault_proxmox_api_token }}"
proxmox_node: "{{ vault_proxmox_node }}"
EOF
```

---

## 34.2 — Project Structure

```bash
# Create provisioning playbook directory
mkdir -p ~/projects/ansible-network/playbooks/infrastructure/provision
mkdir -p ~/projects/ansible-network/playbooks/infrastructure/lifecycle
mkdir -p ~/projects/ansible-network/playbooks/infrastructure/base

# Final structure for this part:
# playbooks/infrastructure/
# ├── provision/
# │   ├── network-lab.yml
# │   ├── netbox.yml
# │   ├── automation.yml
# │   ├── monitoring.yml
# │   ├── observability.yml
# │   ├── logging.yml
# │   └── all.yml              ← imports all six in sequence
# ├── lifecycle/
# │   ├── start.yml
# │   ├── stop.yml
# │   ├── snapshot.yml
# │   ├── restore.yml
# │   └── destroy.yml
# └── base/
#     ├── harden.yml           ← SSH, ufw, NTP, syslog on all VMs
#     └── configure_nics.yml   ← Second NIC (vmbr2) static IP config
```

### Install required Python library

```bash
# The community.general.proxmox module requires proxmoxer
source ~/ansible-venv/bin/activate
pip install proxmoxer requests --break-system-packages

# Verify
python3 -c "import proxmoxer; print('proxmoxer OK')"
```

---

## 34.3 — VM Definitions (Variables)

Each playbook uses a `vm` variable dict. Defining it consistently means the same tasks work for every VM — only the variables differ.

```yaml
# Standard VM variable structure used in all provisioning playbooks
vm:
  vmid: 101                          # Proxmox VM ID (unique)
  name: network-lab                  # VM hostname
  clone: 9000                        # Template VMID to clone from
  cores: 8                           # vCPU count
  memory: 24576                      # RAM in MB
  disk_size: 120                     # Boot disk size in GB
  storage: local-lvm                 # Proxmox storage pool
  ip0: "172.16.0.10/24"             # vmbr0 (management) IP/prefix
  gw0: "172.16.0.1"                 # vmbr0 gateway
  ip1: "192.168.100.10/24"          # vmbr2 (telemetry) IP/prefix
  vmbr_extra: vmbr2                  # Second bridge to attach
  nameserver: "8.8.8.8"
  searchdomain: "lab.local"
  tags: "lab,network"               # Proxmox tags (used by dynamic inventory)
```

---

## 34.4 — Shared Provisioning Tasks

Rather than repeating the same tasks in every playbook, a shared tasks file handles the common provisioning steps. Each individual playbook includes this file after setting its `vm` variables.

```bash
mkdir -p ~/projects/ansible-network/playbooks/infrastructure/provision/tasks

cat > ~/projects/ansible-network/playbooks/infrastructure/provision/tasks/provision_vm.yml << 'EOF'
---
# provision_vm.yml — shared tasks included by each VM playbook
# Requires: vm dict defined in the calling playbook vars

- name: "Proxmox | Check if VM {{ vm.vmid }} already exists"
  community.general.proxmox_kvm:
    api_host: "{{ proxmox_host }}"
    api_user: "{{ proxmox_api_user }}"
    api_token_id: "{{ proxmox_api_user.split('!')[1] }}"
    api_token_secret: "{{ proxmox_api_token }}"
    vmid: "{{ vm.vmid }}"
    state: current
  register: vm_status
  failed_when: false
  changed_when: false
  delegate_to: localhost

- name: "Proxmox | Clone template {{ vm.clone }} → VM {{ vm.vmid }} ({{ vm.name }})"
  community.general.proxmox_kvm:
    api_host: "{{ proxmox_host }}"
    api_user: "{{ proxmox_api_user }}"
    api_token_id: "{{ proxmox_api_user.split('!')[1] }}"
    api_token_secret: "{{ proxmox_api_token }}"
    node: "{{ proxmox_node }}"
    vmid: "{{ vm.vmid }}"
    clone: "{{ vm.clone }}"
    name: "{{ vm.name }}"
    full: true
    storage: "{{ vm.storage }}"
    timeout: 300
    state: present
  when: vm_status.status is not defined
  delegate_to: localhost
  register: clone_result

- name: "Proxmox | Configure VM resources"
  community.general.proxmox_kvm:
    api_host: "{{ proxmox_host }}"
    api_user: "{{ proxmox_api_user }}"
    api_token_id: "{{ proxmox_api_user.split('!')[1] }}"
    api_token_secret: "{{ proxmox_api_token }}"
    node: "{{ proxmox_node }}"
    vmid: "{{ vm.vmid }}"
    cores: "{{ vm.cores }}"
    memory: "{{ vm.memory }}"
    tags: "{{ vm.tags | default('lab') }}"
    update: true
  delegate_to: localhost

- name: "Proxmox | Resize boot disk to {{ vm.disk_size }}GB"
  community.general.proxmox_disk:
    api_host: "{{ proxmox_host }}"
    api_user: "{{ proxmox_api_user }}"
    api_token_id: "{{ proxmox_api_user.split('!')[1] }}"
    api_token_secret: "{{ proxmox_api_token }}"
    vmid: "{{ vm.vmid }}"
    disk: scsi0
    size: "{{ vm.disk_size }}G"
    state: resized
  delegate_to: localhost

- name: "Proxmox | Attach second NIC on {{ vm.vmbr_extra }}"
  community.general.proxmox_nic:
    api_host: "{{ proxmox_host }}"
    api_user: "{{ proxmox_api_user }}"
    api_token_id: "{{ proxmox_api_user.split('!')[1] }}"
    api_token_secret: "{{ proxmox_api_token }}"
    vmid: "{{ vm.vmid }}"
    interface: net1
    bridge: "{{ vm.vmbr_extra }}"
    model: virtio
  delegate_to: localhost

- name: "Proxmox | Configure cloud-init network and identity"
  community.general.proxmox_kvm:
    api_host: "{{ proxmox_host }}"
    api_user: "{{ proxmox_api_user }}"
    api_token_id: "{{ proxmox_api_user.split('!')[1] }}"
    api_token_secret: "{{ proxmox_api_token }}"
    node: "{{ proxmox_node }}"
    vmid: "{{ vm.vmid }}"
    ciuser: ansible
    cipassword: "{{ vault_ansible_device_password }}"
    sshkeys: "{{ lookup('file', '~/.ssh/id_ed25519.pub') }}"
    ipconfig:
      ipconfig0: "ip={{ vm.ip0 }},gw={{ vm.gw0 }}"
      ipconfig1: "ip={{ vm.ip1 }}"
    nameservers: "{{ vm.nameserver }}"
    searchdomains: "{{ vm.searchdomain }}"
    update: true
  delegate_to: localhost

- name: "Proxmox | Start VM {{ vm.vmid }} ({{ vm.name }})"
  community.general.proxmox_kvm:
    api_host: "{{ proxmox_host }}"
    api_user: "{{ proxmox_api_user }}"
    api_token_id: "{{ proxmox_api_user.split('!')[1] }}"
    api_token_secret: "{{ proxmox_api_token }}"
    vmid: "{{ vm.vmid }}"
    state: started
  delegate_to: localhost

- name: "Proxmox | Wait for SSH on {{ vm.ip0 | ipaddr('address') }}"
  ansible.builtin.wait_for:
    host: "{{ vm.ip0 | ipaddr('address') }}"
    port: 22
    delay: 10
    timeout: 180
    state: started
  delegate_to: localhost

- name: "Proxmox | Confirm VM is reachable via Ansible"
  ansible.builtin.ping:
  delegate_to: "{{ vm.name }}"
  register: ping_result

- name: "Proxmox | Report provisioning result"
  ansible.builtin.debug:
    msg:
      - "════════════════════════════════════════"
      - " VM provisioned successfully"
      - " Name:   {{ vm.name }}"
      - " VMID:   {{ vm.vmid }}"
      - " IP:     {{ vm.ip0 | ipaddr('address') }}"
      - " Telemetry: {{ vm.ip1 | ipaddr('address') }}"
      - "════════════════════════════════════════"
EOF
```

---

## 34.5 — Individual VM Provisioning Playbooks

### network-lab.yml

```bash
cat > ~/projects/ansible-network/playbooks/infrastructure/provision/network-lab.yml << 'EOF'
---
# Provisions the network-lab VM
# Role: Containerlab + vrnetlab network device simulation
# Usage: ansible-playbook playbooks/infrastructure/provision/network-lab.yml
#        --vault-id lab@.vault/lab.txt

- name: "Provision | network-lab VM"
  hosts: localhost
  gather_facts: false

  vars:
    vm:
      vmid: 101
      name: network-lab
      clone: "{{ vault_proxmox_template_id }}"
      cores: 8
      memory: 24576
      disk_size: 120
      storage: local-lvm
      ip0: "172.16.0.10/24"
      gw0: "172.16.0.1"
      ip1: "192.168.100.10/24"
      vmbr_extra: vmbr2
      nameserver: "8.8.8.8"
      searchdomain: "lab.local"
      tags: "lab,containerlab,network"
    proxmox_host: "{{ vault_proxmox_host }}"
    proxmox_node: "{{ vault_proxmox_node }}"
    proxmox_api_user: "{{ vault_proxmox_api_user }}"
    proxmox_api_token: "{{ vault_proxmox_api_token }}"

  tasks:
    - name: "Include shared provisioning tasks"
      ansible.builtin.include_tasks:
        file: tasks/provision_vm.yml

    # network-lab also needs vmbr1 for the lab data plane
    - name: "Proxmox | Attach vmbr1 (lab data plane) to network-lab"
      community.general.proxmox_nic:
        api_host: "{{ proxmox_host }}"
        api_user: "{{ proxmox_api_user }}"
        api_token_id: "{{ proxmox_api_user.split('!')[1] }}"
        api_token_secret: "{{ proxmox_api_token }}"
        vmid: "{{ vm.vmid }}"
        interface: net2
        bridge: vmbr1
        model: virtio
      delegate_to: localhost
EOF
```

### netbox.yml

```bash
cat > ~/projects/ansible-network/playbooks/infrastructure/provision/netbox.yml << 'EOF'
---
- name: "Provision | netbox VM"
  hosts: localhost
  gather_facts: false

  vars:
    vm:
      vmid: 102
      name: netbox
      clone: "{{ vault_proxmox_template_id }}"
      cores: 2
      memory: 4096
      disk_size: 40
      storage: local-lvm
      ip0: "172.16.0.20/24"
      gw0: "172.16.0.1"
      ip1: "192.168.100.20/24"
      vmbr_extra: vmbr2
      nameserver: "8.8.8.8"
      searchdomain: "lab.local"
      tags: "lab,netbox,source-of-truth"
    proxmox_host: "{{ vault_proxmox_host }}"
    proxmox_node: "{{ vault_proxmox_node }}"
    proxmox_api_user: "{{ vault_proxmox_api_user }}"
    proxmox_api_token: "{{ vault_proxmox_api_token }}"

  tasks:
    - name: "Include shared provisioning tasks"
      ansible.builtin.include_tasks:
        file: tasks/provision_vm.yml
EOF
```

### automation.yml

```bash
cat > ~/projects/ansible-network/playbooks/infrastructure/provision/automation.yml << 'EOF'
---
- name: "Provision | automation VM"
  hosts: localhost
  gather_facts: false

  vars:
    vm:
      vmid: 103
      name: automation
      clone: "{{ vault_proxmox_template_id }}"
      cores: 4
      memory: 8192
      disk_size: 60
      storage: local-lvm
      ip0: "172.16.0.30/24"
      gw0: "172.16.0.1"
      ip1: "192.168.100.30/24"
      vmbr_extra: vmbr2
      nameserver: "8.8.8.8"
      searchdomain: "lab.local"
      tags: "lab,automation,awx,gitea"
    proxmox_host: "{{ vault_proxmox_host }}"
    proxmox_node: "{{ vault_proxmox_node }}"
    proxmox_api_user: "{{ vault_proxmox_api_user }}"
    proxmox_api_token: "{{ vault_proxmox_api_token }}"

  tasks:
    - name: "Include shared provisioning tasks"
      ansible.builtin.include_tasks:
        file: tasks/provision_vm.yml
EOF
```

### monitoring.yml

```bash
cat > ~/projects/ansible-network/playbooks/infrastructure/provision/monitoring.yml << 'EOF'
---
- name: "Provision | monitoring VM"
  hosts: localhost
  gather_facts: false

  vars:
    vm:
      vmid: 104
      name: monitoring
      clone: "{{ vault_proxmox_template_id }}"
      cores: 2
      memory: 4096
      disk_size: 40
      storage: local-lvm
      ip0: "172.16.0.40/24"
      gw0: "172.16.0.1"
      ip1: "192.168.100.40/24"
      vmbr_extra: vmbr2
      nameserver: "8.8.8.8"
      searchdomain: "lab.local"
      tags: "lab,monitoring,zabbix"
    proxmox_host: "{{ vault_proxmox_host }}"
    proxmox_node: "{{ vault_proxmox_node }}"
    proxmox_api_user: "{{ vault_proxmox_api_user }}"
    proxmox_api_token: "{{ vault_proxmox_api_token }}"

  tasks:
    - name: "Include shared provisioning tasks"
      ansible.builtin.include_tasks:
        file: tasks/provision_vm.yml
EOF
```

### observability.yml

```bash
cat > ~/projects/ansible-network/playbooks/infrastructure/provision/observability.yml << 'EOF'
---
- name: "Provision | observability VM"
  hosts: localhost
  gather_facts: false

  vars:
    vm:
      vmid: 105
      name: observability
      clone: "{{ vault_proxmox_template_id }}"
      cores: 4
      memory: 8192
      disk_size: 80
      storage: local-lvm
      ip0: "172.16.0.50/24"
      gw0: "172.16.0.1"
      ip1: "192.168.100.50/24"
      vmbr_extra: vmbr2
      nameserver: "8.8.8.8"
      searchdomain: "lab.local"
      tags: "lab,observability,prometheus,grafana,loki"
    proxmox_host: "{{ vault_proxmox_host }}"
    proxmox_node: "{{ vault_proxmox_node }}"
    proxmox_api_user: "{{ vault_proxmox_api_user }}"
    proxmox_api_token: "{{ vault_proxmox_api_token }}"

  tasks:
    - name: "Include shared provisioning tasks"
      ansible.builtin.include_tasks:
        file: tasks/provision_vm.yml
EOF
```

### logging.yml

```bash
cat > ~/projects/ansible-network/playbooks/infrastructure/provision/logging.yml << 'EOF'
---
- name: "Provision | logging VM"
  hosts: localhost
  gather_facts: false

  vars:
    vm:
      vmid: 106
      name: logging
      clone: "{{ vault_proxmox_template_id }}"
      cores: 4
      memory: 12288
      disk_size: 100
      storage: local-lvm
      ip0: "172.16.0.60/24"
      gw0: "172.16.0.1"
      ip1: "192.168.100.60/24"
      vmbr_extra: vmbr2
      nameserver: "8.8.8.8"
      searchdomain: "lab.local"
      tags: "lab,logging,graylog,opensearch"
    proxmox_host: "{{ vault_proxmox_host }}"
    proxmox_node: "{{ vault_proxmox_node }}"
    proxmox_api_user: "{{ vault_proxmox_api_user }}"
    proxmox_api_token: "{{ vault_proxmox_api_token }}"

  tasks:
    - name: "Include shared provisioning tasks"
      ansible.builtin.include_tasks:
        file: tasks/provision_vm.yml
EOF
```

### all.yml — provision everything in sequence

```bash
cat > ~/projects/ansible-network/playbooks/infrastructure/provision/all.yml << 'EOF'
---
# Provisions all 6 infrastructure VMs in dependency order.
# Run this on a clean Proxmox host to rebuild the full stack from scratch.
# Usage: ansible-playbook playbooks/infrastructure/provision/all.yml
#        --vault-id lab@.vault/lab.txt

- name: "Provision all infrastructure VMs"
  ansible.builtin.import_playbook: network-lab.yml

- name: "Provision netbox VM"
  ansible.builtin.import_playbook: netbox.yml

- name: "Provision automation VM"
  ansible.builtin.import_playbook: automation.yml

- name: "Provision monitoring VM"
  ansible.builtin.import_playbook: monitoring.yml

- name: "Provision observability VM"
  ansible.builtin.import_playbook: observability.yml

- name: "Provision logging VM"
  ansible.builtin.import_playbook: logging.yml
EOF
```

---

## 34.6 — Running the Provisioning Playbooks

```bash
cd ~/projects/ansible-network

# Syntax check all provisioning playbooks
for pb in playbooks/infrastructure/provision/*.yml; do
  echo "→ $pb"
  ansible-playbook --syntax-check \
    -i inventory/hosts.yml \
    --vault-id lab@.vault/lab.txt \
    "$pb" || exit 1
done

# Provision a single VM (recommended approach — one at a time, review output)
ansible-playbook \
  playbooks/infrastructure/provision/network-lab.yml \
  -i inventory/hosts.yml \
  --vault-id lab@.vault/lab.txt \
  -v

# Provision all VMs in sequence
ansible-playbook \
  playbooks/infrastructure/provision/all.yml \
  -i inventory/hosts.yml \
  --vault-id lab@.vault/lab.txt

# Verify all VMs are reachable after provisioning
ansible linux_vms \
  -m ansible.builtin.ping \
  -i inventory/hosts.yml \
  --vault-id lab@.vault/lab.txt
```

### Understanding idempotency with Proxmox

```
Running a provisioning playbook twice is safe — this is what idempotent means:

First run (VM doesn't exist):
  Clone template → CHANGED
  Configure resources → CHANGED
  Resize disk → CHANGED
  Attach NICs → CHANGED
  Configure cloud-init → CHANGED
  Start VM → CHANGED
  Wait for SSH → OK
  Ping → OK

Second run (VM already exists, correct config):
  Clone template → SKIPPED  (vm_status check shows VM exists)
  Configure resources → OK  (already correct — no change needed)
  Resize disk → OK          (already correct size)
  Attach NICs → OK          (already attached)
  Configure cloud-init → OK (already correct)
  Start VM → OK             (already running)
  Wait for SSH → OK
  Ping → OK

If a VM is partially configured (e.g. crashed mid-provisioning):
  The playbook detects current state and applies only what's missing.
  This is the key advantage over shell scripts — no manual cleanup needed.
```

---

## 34.7 — VM Lifecycle Playbooks

### start.yml

```bash
cat > ~/projects/ansible-network/playbooks/infrastructure/lifecycle/start.yml << 'EOF'
---
# Start one or all infrastructure VMs
# Usage:
#   All VMs:    ansible-playbook lifecycle/start.yml
#   Single VM:  ansible-playbook lifecycle/start.yml -e "target_vmid=101"

- name: "Lifecycle | Start infrastructure VMs"
  hosts: localhost
  gather_facts: false

  vars:
    vm_ids:
      - { vmid: 101, name: network-lab }
      - { vmid: 102, name: netbox }
      - { vmid: 103, name: automation }
      - { vmid: 104, name: monitoring }
      - { vmid: 105, name: observability }
      - { vmid: 106, name: logging }
    target_vmid: "all"   # Override with -e "target_vmid=101" for single VM

  tasks:
    - name: "Start VM {{ item.name }} ({{ item.vmid }})"
      community.general.proxmox_kvm:
        api_host: "{{ vault_proxmox_host }}"
        api_user: "{{ vault_proxmox_api_user }}"
        api_token_id: "{{ vault_proxmox_api_user.split('!')[1] }}"
        api_token_secret: "{{ vault_proxmox_api_token }}"
        vmid: "{{ item.vmid }}"
        state: started
      loop: "{{ vm_ids }}"
      loop_control:
        label: "{{ item.name }}"
      when: target_vmid == 'all' or target_vmid | int == item.vmid
EOF
```

### stop.yml

```bash
cat > ~/projects/ansible-network/playbooks/infrastructure/lifecycle/stop.yml << 'EOF'
---
# Gracefully stop one or all infrastructure VMs
# Usage:
#   All VMs:    ansible-playbook lifecycle/stop.yml
#   Single VM:  ansible-playbook lifecycle/stop.yml -e "target_vmid=106"

- name: "Lifecycle | Stop infrastructure VMs"
  hosts: localhost
  gather_facts: false

  vars:
    # Stop in reverse dependency order — logging first, network-lab last
    vm_ids:
      - { vmid: 106, name: logging }
      - { vmid: 105, name: observability }
      - { vmid: 104, name: monitoring }
      - { vmid: 102, name: netbox }
      - { vmid: 103, name: automation }
      - { vmid: 101, name: network-lab }
    target_vmid: "all"

  tasks:
    - name: "Stop VM {{ item.name }} ({{ item.vmid }})"
      community.general.proxmox_kvm:
        api_host: "{{ vault_proxmox_host }}"
        api_user: "{{ vault_proxmox_api_user }}"
        api_token_id: "{{ vault_proxmox_api_user.split('!')[1] }}"
        api_token_secret: "{{ vault_proxmox_api_token }}"
        vmid: "{{ item.vmid }}"
        state: stopped
        force: false       # Graceful shutdown — sends ACPI signal, waits
        timeout: 60
      loop: "{{ vm_ids }}"
      loop_control:
        label: "{{ item.name }}"
      when: target_vmid == 'all' or target_vmid | int == item.vmid
EOF
```

### snapshot.yml

```bash
cat > ~/projects/ansible-network/playbooks/infrastructure/lifecycle/snapshot.yml << 'EOF'
---
# Create a named snapshot of one or all VMs
# Usage:
#   ansible-playbook lifecycle/snapshot.yml \
#     -e "snap_name=pre-zabbix-install snap_description='Before Zabbix Part 37'"
#   Single VM:
#   ansible-playbook lifecycle/snapshot.yml \
#     -e "target_vmid=102 snap_name=pre-netbox-upgrade"

- name: "Lifecycle | Snapshot infrastructure VMs"
  hosts: localhost
  gather_facts: false

  vars:
    vm_ids:
      - { vmid: 101, name: network-lab }
      - { vmid: 102, name: netbox }
      - { vmid: 103, name: automation }
      - { vmid: 104, name: monitoring }
      - { vmid: 105, name: observability }
      - { vmid: 106, name: logging }
    target_vmid: "all"
    snap_name: "snapshot-{{ lookup('pipe', 'date +%Y%m%d-%H%M%S') }}"
    snap_description: "Ansible automated snapshot"
    snap_vmstate: false    # true = include RAM state (slow but full restore point)

  tasks:
    - name: "Create snapshot '{{ snap_name }}' on VM {{ item.name }}"
      community.general.proxmox_snap:
        api_host: "{{ vault_proxmox_host }}"
        api_user: "{{ vault_proxmox_api_user }}"
        api_token_id: "{{ vault_proxmox_api_user.split('!')[1] }}"
        api_token_secret: "{{ vault_proxmox_api_token }}"
        vmid: "{{ item.vmid }}"
        snapname: "{{ snap_name }}"
        description: "{{ snap_description }}"
        vmstate: "{{ snap_vmstate }}"
        state: present
        timeout: 120
      loop: "{{ vm_ids }}"
      loop_control:
        label: "{{ item.name }}"
      when: target_vmid == 'all' or target_vmid | int == item.vmid

    - name: "List all snapshots on VM {{ item.name }}"
      community.general.proxmox_snap:
        api_host: "{{ vault_proxmox_host }}"
        api_user: "{{ vault_proxmox_api_user }}"
        api_token_id: "{{ vault_proxmox_api_user.split('!')[1] }}"
        api_token_secret: "{{ vault_proxmox_api_token }}"
        vmid: "{{ item.vmid }}"
        state: list
      loop: "{{ vm_ids }}"
      loop_control:
        label: "{{ item.name }}"
      when: target_vmid == 'all' or target_vmid | int == item.vmid
      register: snap_list

    - name: "Display snapshot list"
      ansible.builtin.debug:
        msg: "{{ item.snapshots | default([]) | map(attribute='name') | list }}"
      loop: "{{ snap_list.results }}"
      loop_control:
        label: "VM {{ item.item.name }}"
      when: item.snapshots is defined
EOF
```

### restore.yml

```bash
cat > ~/projects/ansible-network/playbooks/infrastructure/lifecycle/restore.yml << 'EOF'
---
# Roll back a VM to a named snapshot
# Usage:
#   ansible-playbook lifecycle/restore.yml \
#     -e "target_vmid=102 snap_name=pre-netbox-upgrade"
# CAUTION: This is destructive — all changes after the snapshot are lost.

- name: "Lifecycle | Restore VM from snapshot"
  hosts: localhost
  gather_facts: false

  vars:
    target_vmid: ~           # Required — must be set via -e
    snap_name: ~             # Required — must be set via -e
    vm_names:
      101: network-lab
      102: netbox
      103: automation
      104: monitoring
      105: observability
      106: logging

  pre_tasks:
    - name: "Gate | Verify required variables are set"
      ansible.builtin.assert:
        that:
          - target_vmid is not none
          - snap_name is not none
        fail_msg: >
          Both target_vmid and snap_name are required.
          Usage: -e "target_vmid=102 snap_name=pre-netbox-upgrade"

    - name: "Gate | Confirm restore operation"
      ansible.builtin.pause:
        prompt: >
          WARNING: About to roll back VM {{ target_vmid }}
          ({{ vm_names[target_vmid | int] | default('unknown') }})
          to snapshot '{{ snap_name }}'.
          All changes after this snapshot will be LOST.
          Press ENTER to continue or Ctrl+C to abort

  tasks:
    - name: "Stop VM {{ target_vmid }} before rollback"
      community.general.proxmox_kvm:
        api_host: "{{ vault_proxmox_host }}"
        api_user: "{{ vault_proxmox_api_user }}"
        api_token_id: "{{ vault_proxmox_api_user.split('!')[1] }}"
        api_token_secret: "{{ vault_proxmox_api_token }}"
        vmid: "{{ target_vmid }}"
        state: stopped
        force: true
        timeout: 30

    - name: "Restore snapshot '{{ snap_name }}' on VM {{ target_vmid }}"
      community.general.proxmox_snap:
        api_host: "{{ vault_proxmox_host }}"
        api_user: "{{ vault_proxmox_api_user }}"
        api_token_id: "{{ vault_proxmox_api_user.split('!')[1] }}"
        api_token_secret: "{{ vault_proxmox_api_token }}"
        vmid: "{{ target_vmid }}"
        snapname: "{{ snap_name }}"
        state: rollback
        timeout: 120

    - name: "Start VM {{ target_vmid }} after restore"
      community.general.proxmox_kvm:
        api_host: "{{ vault_proxmox_host }}"
        api_user: "{{ vault_proxmox_api_user }}"
        api_token_id: "{{ vault_proxmox_api_user.split('!')[1] }}"
        api_token_secret: "{{ vault_proxmox_api_token }}"
        vmid: "{{ target_vmid }}"
        state: started

    - name: "Wait for VM to be reachable after restore"
      ansible.builtin.wait_for:
        host: "{{ hostvars[vm_names[target_vmid | int]].ansible_host }}"
        port: 22
        timeout: 120
        state: started

    - name: "Confirm restore complete"
      ansible.builtin.debug:
        msg: >
          VM {{ target_vmid }} restored to '{{ snap_name }}' and
          is reachable. Verify services manually before proceeding.
EOF
```

### destroy.yml

```bash
cat > ~/projects/ansible-network/playbooks/infrastructure/lifecycle/destroy.yml << 'EOF'
---
# Permanently destroy one or all infrastructure VMs
# USE WITH EXTREME CAUTION — this is irreversible
# Usage:
#   Single VM (safest):
#   ansible-playbook lifecycle/destroy.yml \
#     -e "target_vmid=101 confirm_destroy=true"
#
#   All VMs (nuclear option — rebuild from scratch):
#   ansible-playbook lifecycle/destroy.yml \
#     -e "target_vmid=all confirm_destroy=true"

- name: "Lifecycle | Destroy infrastructure VMs"
  hosts: localhost
  gather_facts: false

  vars:
    vm_ids:
      - { vmid: 106, name: logging }
      - { vmid: 105, name: observability }
      - { vmid: 104, name: monitoring }
      - { vmid: 102, name: netbox }
      - { vmid: 103, name: automation }
      - { vmid: 101, name: network-lab }
    target_vmid: ~
    confirm_destroy: false

  pre_tasks:
    - name: "Gate | Require explicit confirmation"
      ansible.builtin.assert:
        that:
          - confirm_destroy | bool
          - target_vmid is not none
        fail_msg: >
          Destroy requires explicit confirmation.
          Pass -e "confirm_destroy=true target_vmid=<vmid|all>"
          This operation is IRREVERSIBLE.

    - name: "Gate | Final confirmation pause"
      ansible.builtin.pause:
        prompt: >
          IRREVERSIBLE: Destroying
          {{ 'ALL infrastructure VMs' if target_vmid == 'all'
             else 'VM ' + target_vmid | string }}.
          This cannot be undone. Press ENTER to proceed, Ctrl+C to abort.

  tasks:
    - name: "Stop VM {{ item.name }} before destroy"
      community.general.proxmox_kvm:
        api_host: "{{ vault_proxmox_host }}"
        api_user: "{{ vault_proxmox_api_user }}"
        api_token_id: "{{ vault_proxmox_api_user.split('!')[1] }}"
        api_token_secret: "{{ vault_proxmox_api_token }}"
        vmid: "{{ item.vmid }}"
        state: stopped
        force: true
        timeout: 30
      loop: "{{ vm_ids }}"
      loop_control:
        label: "{{ item.name }}"
      when: target_vmid == 'all' or target_vmid | int == item.vmid
      failed_when: false    # VM may already be stopped

    - name: "Destroy VM {{ item.name }} ({{ item.vmid }})"
      community.general.proxmox_kvm:
        api_host: "{{ vault_proxmox_host }}"
        api_user: "{{ vault_proxmox_api_user }}"
        api_token_id: "{{ vault_proxmox_api_user.split('!')[1] }}"
        api_token_secret: "{{ vault_proxmox_api_token }}"
        vmid: "{{ item.vmid }}"
        state: absent
        force: true
      loop: "{{ vm_ids }}"
      loop_control:
        label: "{{ item.name }}"
      when: target_vmid == 'all' or target_vmid | int == item.vmid
EOF
```

---

## 34.8 — Base Hardening Playbook

Once VMs are provisioned and reachable, this playbook configures them all consistently — SSH hardening, firewall, NTP, syslog, second NIC static IP, and hostname. This runs once after provisioning and is idempotent.

```bash
cat > ~/projects/ansible-network/playbooks/infrastructure/base/harden.yml << 'EOF'
---
# Base hardening for all infrastructure VMs
# Run after provisioning, before any role-specific installation
# Usage: ansible-playbook playbooks/infrastructure/base/harden.yml
#        -i inventory/hosts.yml --vault-id lab@.vault/lab.txt

- name: "Base | Harden all infrastructure VMs"
  hosts: linux_vms
  gather_facts: true
  become: true
  tags: [base, harden]

  tasks:

    # ── Packages ──────────────────────────────────────────────────────
    - name: "Packages | Update apt cache"
      ansible.builtin.apt:
        update_cache: true
        cache_valid_time: 3600

    - name: "Packages | Install base packages"
      ansible.builtin.apt:
        name:
          - curl
          - wget
          - git
          - vim
          - htop
          - net-tools
          - tcpdump
          - python3-pip
          - python3-venv
          - ufw
          - fail2ban
          - chrony          # NTP client
          - rsyslog         # Syslog client
          - unattended-upgrades
        state: present

    # ── Hostname ──────────────────────────────────────────────────────
    - name: "Hostname | Set hostname"
      ansible.builtin.hostname:
        name: "{{ inventory_hostname }}"

    - name: "Hostname | Update /etc/hosts"
      ansible.builtin.lineinfile:
        path: /etc/hosts
        regexp: "^127\\.0\\.1\\.1"
        line: "127.0.1.1 {{ inventory_hostname }}"

    # ── SSH hardening ──────────────────────────────────────────────────
    - name: "SSH | Harden sshd_config"
      ansible.builtin.lineinfile:
        path: /etc/ssh/sshd_config
        regexp: "{{ item.regexp }}"
        line: "{{ item.line }}"
        state: present
      loop:
        - { regexp: '^#?PermitRootLogin', line: 'PermitRootLogin no' }
        - { regexp: '^#?PasswordAuthentication', line: 'PasswordAuthentication no' }
        - { regexp: '^#?PubkeyAuthentication', line: 'PubkeyAuthentication yes' }
        - { regexp: '^#?X11Forwarding', line: 'X11Forwarding no' }
        - { regexp: '^#?MaxAuthTries', line: 'MaxAuthTries 3' }
        - { regexp: '^#?ClientAliveInterval', line: 'ClientAliveInterval 300' }
        - { regexp: '^#?ClientAliveCountMax', line: 'ClientAliveCountMax 2' }
        - { regexp: '^#?LoginGraceTime', line: 'LoginGraceTime 30' }
        - { regexp: '^#?AllowAgentForwarding', line: 'AllowAgentForwarding no' }
      notify: Restart SSH

    # ── Firewall (ufw) ─────────────────────────────────────────────────
    - name: "UFW | Set default deny inbound"
      community.general.ufw:
        default: deny
        direction: incoming

    - name: "UFW | Set default allow outbound"
      community.general.ufw:
        default: allow
        direction: outgoing

    - name: "UFW | Allow SSH from management network"
      community.general.ufw:
        rule: allow
        port: "22"
        proto: tcp
        src: "172.16.0.0/24"
        comment: "SSH from management network"

    - name: "UFW | Allow SSH from telemetry network"
      community.general.ufw:
        rule: allow
        port: "22"
        proto: tcp
        src: "192.168.100.0/24"
        comment: "SSH from telemetry network"

    - name: "UFW | Enable firewall"
      community.general.ufw:
        state: enabled

    # ── NTP ───────────────────────────────────────────────────────────
    - name: "NTP | Configure chrony servers"
      ansible.builtin.template:
        src: chrony.conf.j2
        dest: /etc/chrony/chrony.conf
        mode: '0644'
      notify: Restart chrony

    - name: "NTP | Ensure chrony is enabled and running"
      ansible.builtin.service:
        name: chrony
        state: started
        enabled: true

    # ── Syslog ────────────────────────────────────────────────────────
    - name: "Syslog | Forward logs to Graylog"
      ansible.builtin.lineinfile:
        path: /etc/rsyslog.conf
        line: "*.* @{{ syslog_server }}:{{ syslog_port }}"
        regexp: "^\\*\\.\\* @{{ syslog_server }}"
        state: present
      notify: Restart rsyslog

    - name: "Syslog | Ensure rsyslog is enabled"
      ansible.builtin.service:
        name: rsyslog
        state: started
        enabled: true

    # ── Second NIC (vmbr2 / telemetry) ────────────────────────────────
    - name: "NIC | Configure telemetry interface ({{ telemetry_interface }})"
      ansible.builtin.template:
        src: netplan-telemetry.yaml.j2
        dest: /etc/netplan/60-telemetry.yaml
        mode: '0600'
      notify: Apply netplan

    # ── Unattended upgrades ───────────────────────────────────────────
    - name: "Security | Enable unattended security upgrades"
      ansible.builtin.copy:
        content: |
          APT::Periodic::Update-Package-Lists "1";
          APT::Periodic::Unattended-Upgrade "1";
          APT::Periodic::AutocleanInterval "7";
        dest: /etc/apt/apt.conf.d/20auto-upgrades
        mode: '0644'

    # ── Fail2ban ──────────────────────────────────────────────────────
    - name: "Fail2ban | Enable and start"
      ansible.builtin.service:
        name: fail2ban
        state: started
        enabled: true

  handlers:
    - name: Restart SSH
      ansible.builtin.service:
        name: ssh
        state: restarted

    - name: Restart chrony
      ansible.builtin.service:
        name: chrony
        state: restarted

    - name: Restart rsyslog
      ansible.builtin.service:
        name: rsyslog
        state: restarted

    - name: Apply netplan
      ansible.builtin.command: netplan apply
      changed_when: true
EOF
```

### Chrony template

```bash
mkdir -p ~/projects/ansible-network/playbooks/infrastructure/base/templates

cat > ~/projects/ansible-network/playbooks/infrastructure/base/templates/chrony.conf.j2 << 'EOF'
# Managed by Ansible — do not edit manually
{% for server in ntp_servers %}
server {{ server }} iburst
{% endfor %}

driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
logdir /var/log/chrony
EOF
```

### Netplan template for second NIC

```bash
cat > ~/projects/ansible-network/playbooks/infrastructure/base/templates/netplan-telemetry.yaml.j2 << 'EOF'
# Managed by Ansible — do not edit manually
# Telemetry/services interface (vmbr2)
network:
  version: 2
  ethernets:
    {{ telemetry_interface }}:
      addresses:
        - {{ hostvars[inventory_hostname].ansible_host | regex_replace('^172\.16\.0\.', '192.168.100.') }}/24
      # No gateway — telemetry network is internal only
      nameservers:
        addresses: [8.8.8.8]
EOF
```

### Run the hardening playbook

```bash
# Run hardening on all VMs
ansible-playbook \
  playbooks/infrastructure/base/harden.yml \
  -i inventory/hosts.yml \
  --vault-id lab@.vault/lab.txt

# Verify second NIC is up on all VMs
ansible linux_vms \
  -i inventory/hosts.yml \
  -m ansible.builtin.command \
  -a "ip addr show {{ telemetry_interface }}" \
  --vault-id lab@.vault/lab.txt

# Verify syslog forwarding is configured
ansible linux_vms \
  -i inventory/hosts.yml \
  -m ansible.builtin.command \
  -a "grep '192.168.100.60' /etc/rsyslog.conf" \
  --vault-id lab@.vault/lab.txt

# Verify ufw is active
ansible linux_vms \
  -i inventory/hosts.yml \
  -m ansible.builtin.command \
  -a "ufw status" \
  --become \
  --vault-id lab@.vault/lab.txt
```

---

## 34.9 — Dynamic Inventory Plugin (Preview)

The `community.general.proxmox` inventory plugin queries the Proxmox API and returns VMs as inventory hosts, grouped by tags, pool, or node. It replaces the static `infrastructure` group entries in `hosts.yml`.

```bash
# Install the plugin dependency (already installed — proxmoxer)
pip show proxmoxer

# Create the dynamic inventory config file
cat > ~/projects/ansible-network/inventory/proxmox.yml << 'EOF'
---
# community.general.proxmox dynamic inventory plugin
# Queries Proxmox API and builds inventory from running VMs
# Usage: ansible-inventory -i inventory/proxmox.yml --graph

plugin: community.general.proxmox
url: "https://{{ lookup('env', 'PROXMOX_HOST') | default('172.16.0.1') }}:8006"
user: "{{ lookup('env', 'PROXMOX_USER') | default('ansible-api@pve!ansible') }}"
token_id: ansible
token_secret: "{{ lookup('env', 'PROXMOX_TOKEN') }}"
validate_certs: false

# Group VMs by their Proxmox tags
group_prefix: proxmox_tag_
# VMs tagged 'lab' appear in group proxmox_tag_lab
# VMs tagged 'monitoring' appear in proxmox_tag_monitoring
# etc.

# Only include running VMs
want_facts: true
filters:
  - status == 'running'

# Map Proxmox VM name to Ansible hostname
compose:
  ansible_host: proxmox_ipconfig0 | regex_search('ip=([^/,]+)', '\\1') | first

keyed_groups:
  - key: proxmox_tags
    prefix: proxmox_tag
    separator: _
EOF
```

### Using the dynamic inventory alongside the static one

```bash
# Test the dynamic inventory independently
export PROXMOX_TOKEN="your-token-secret"
ansible-inventory \
  -i inventory/proxmox.yml \
  --graph

# Use both inventories together — Ansible merges them
ansible-inventory \
  -i inventory/hosts.yml \
  -i inventory/proxmox.yml \
  --graph

# The static inventory remains the primary working inventory.
# The dynamic inventory is an additional source — useful for
# targeting VMs by tag without maintaining the static list:
ansible proxmox_tag_lab \
  -i inventory/proxmox.yml \
  -m ansible.builtin.ping
```

### ### ℹ️ Info: Why keep static inventory as default

> The dynamic inventory plugin requires the Proxmox API to be reachable and all VMs to be running. If you're provisioning VMs for the first time, or if a VM is stopped, the dynamic inventory won't include it. The static inventory always works regardless of VM state — it's the reliable fallback. Use the dynamic inventory for ad-hoc operations against running VMs; use the static inventory for provisioning and lifecycle management.

---

## 34.10 — Snapshot Strategy for the Lab

With the full lifecycle tooling in place, a consistent snapshot strategy means any mistake in Parts 35–42 can be rolled back cleanly.

```
Recommended snapshot points:

SNAP-01: "clean-provision"
  When:  Immediately after all 6 VMs are provisioned and hardened
  What:  Base Ubuntu, SSH keys, hardened config, second NIC — nothing else
  Command:
    ansible-playbook lifecycle/snapshot.yml \
      -e "snap_name=clean-provision \
          snap_description='Clean provision post-harden Part 34'"

SNAP-02: "pre-<service>-install"
  When:  Before each new service installation (Parts 35–41)
  What:  VM state just before a major change
  Examples:
    ansible-playbook lifecycle/snapshot.yml \
      -e "target_vmid=103 snap_name=pre-gitea \
          snap_description='Before Gitea install Part 35'"
    ansible-playbook lifecycle/snapshot.yml \
      -e "target_vmid=104 snap_name=pre-zabbix \
          snap_description='Before Zabbix install Part 37'"

SNAP-03: "post-<service>-install"
  When:  After a service is confirmed working
  What:  Known-good state to return to if later steps break this VM
  Examples:
    ansible-playbook lifecycle/snapshot.yml \
      -e "target_vmid=103 snap_name=post-gitea \
          snap_description='Gitea running and tested Part 35'"

Rolling back to a snapshot:
    ansible-playbook lifecycle/restore.yml \
      -e "target_vmid=103 snap_name=pre-gitea"
```

---

## 34.11 — Part 34 Checklist

```
[ ] Proxmox API token created:
    [ ] User ansible-api@pve created
    [ ] Role AnsibleVMManager created with correct privs
    [ ] ACL assigned at / for ansible-api@pve
    [ ] Token generated — secret stored in vault as vault_proxmox_api_token
    [ ] curl test confirms token works against /api2/json/version

[ ] proxmoxer installed in venv:
    pip show proxmoxer returns version info

[ ] Provisioning playbook structure created:
    [ ] playbooks/infrastructure/provision/tasks/provision_vm.yml
    [ ] playbooks/infrastructure/provision/network-lab.yml
    [ ] playbooks/infrastructure/provision/netbox.yml
    [ ] playbooks/infrastructure/provision/automation.yml
    [ ] playbooks/infrastructure/provision/monitoring.yml
    [ ] playbooks/infrastructure/provision/observability.yml
    [ ] playbooks/infrastructure/provision/logging.yml
    [ ] playbooks/infrastructure/provision/all.yml

[ ] All six VMs provisioned via playbooks (not via Proxmox UI):
    [ ] 101 network-lab  — 8c/24GB/120GB — running — SSH reachable
    [ ] 102 netbox       — 2c/4GB/40GB   — running — SSH reachable
    [ ] 103 automation   — 4c/8GB/60GB   — running — SSH reachable
    [ ] 104 monitoring   — 2c/4GB/40GB   — running — SSH reachable
    [ ] 105 observability — 4c/8GB/80GB  — running — SSH reachable
    [ ] 106 logging      — 4c/12GB/100GB — running — SSH reachable

[ ] Idempotency verified — running any provision playbook twice
    shows no unexpected CHANGED tasks on second run

[ ] Lifecycle playbooks created and tested:
    [ ] start.yml — starts VMs, tested with target_vmid=101
    [ ] stop.yml  — stops VMs gracefully
    [ ] snapshot.yml — creates named snapshot, lists snapshots
    [ ] restore.yml — rolls back to snapshot with confirmation gate
    [ ] destroy.yml — destroys VM with double confirmation gate
    [ ] destroy.yml tested with a test VM (not one of the 6) to
        confirm it works before needing it for real

[ ] Base hardening playbook run on all VMs:
    [ ] PasswordAuthentication no in sshd_config
    [ ] PermitRootLogin no in sshd_config
    [ ] ufw active — only port 22 from 172.16.0.0/24 allowed
    [ ] chrony running — NTP synchronised
    [ ] rsyslog forwarding to 192.168.100.60 (Graylog — not yet running)
    [ ] Second NIC (ens19) configured with 192.168.100.x address
    [ ] fail2ban running

[ ] Dynamic inventory plugin configured at inventory/proxmox.yml
    [ ] ansible-inventory -i inventory/proxmox.yml --graph returns VMs
    [ ] Static inventory remains default working inventory

[ ] Snapshot SNAP-01 "clean-provision" taken on all 6 VMs:
    ansible-playbook lifecycle/snapshot.yml \
      -e "snap_name=clean-provision"

[ ] All new playbook files committed to Git:
    git add playbooks/infrastructure/
    git commit -m "feat(infra): VM provisioning and lifecycle playbooks Part 34"
```

---

*Six VMs are provisioned, hardened, and snapshotted — entirely by Ansible, zero Proxmox UI clicks. The infrastructure is code: `all.yml` rebuilds the entire environment from a clean Proxmox host, `destroy.yml` tears it down, and `restore.yml` rolls back any VM to any known-good state. Every subsequent part in this series installs exactly one service per VM and snapshots before and after. If anything breaks, a single playbook command returns to the last known-good state.*

*Next up: **Part 35 — Git Server: Self-Hosted Gitea***
